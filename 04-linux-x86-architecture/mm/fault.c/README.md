# Catching Invalid Memory Access - The x86 Entry Point

> Ever wondered how your CPU catches invalid memory access? How writing to read-only memory triggers copy-on-write? How the kernel knows WHICH address caused the fault?

**You're about to find out!**

---

## What's This About?

This is **arch/x86/mm/fault.c** - the x86-specific entry point when memory access goes wrong!

Here's where it fits:

```
arch/x86/
└── mm/                    ← x86 memory management
    ├── fault.c            ← x86 page fault entry (THIS!)
    ├── pgtable.c          ← Page table operations
    ├── tlb.c              ← TLB management
    ├── init.c             ← Memory initialization
    ├── pageattr.c         ← Page attributes
    └── ioremap.c          ← I/O memory mapping
```

**This file handles:**
- Reading x86-specific registers (CR2)
- Parsing x86 error codes
- First response to page faults
- Calling generic handler (mm/memory.c)
- x86-specific fault cases

---

## The Fundamental Problem

**Physical Reality:**

```
Your System:
├── x86 CPU with MMU (Memory Management Unit)
├── Page tables in RAM
└── Programs accessing memory constantly

Question: When memory access fails, what happens?
```

**The x86 CPU detects the problem and generates Exception #14 (Page Fault)**

But here's the challenge:

```
Different CPUs handle faults DIFFERENTLY:

x86:
├── Sets CR2 register = fault address
├── Pushes error code with 5 bits
└── Generates exception #14

ARM:
├── Sets FAR register = fault address (different name!)
├── Different error format
└── Different exception number

PowerPC:
├── Completely different mechanism
├── Different registers
└── Different error format
```

**The kernel needs a BRIDGE between x86 hardware and generic code!**

---

## Without x86-Specific Handler

**Imagine only having generic mm/memory.c:**

```
Page fault occurs on x86 CPU:

x86 Hardware does:
├── Sets CR2 = 0x12345000 (fault address)
├── Pushes error code = 0x07
└── Generates exception #14

Generic handler (mm/memory.c):
└── "What address faulted?"
       Doesn't know about CR2 register!
       Doesn't know x86 error code format!
       Can't get the information!

Result: Can't handle the fault! 
System crashes! 
```

**This is why we need arch/x86/mm/fault.c!**

---

## The Two-Layer Solution

**Layer 1: x86-Specific (fault.c)**

```
Knows x86 hardware:
├── Read CR2 register (x86 instruction!)
├── Parse x86 error code (5-bit format)
├── Extract: address, write/read, user/kernel
└── Translate to generic format
```

**Layer 2: Generic (mm/memory.c)**

```
Receives generic info:
├── Fault address
├── Access type (read/write)
├── Mode (user/kernel)
└── Makes decision: demand paging, COW, kill process

Works on ALL CPUs! 
```

**Together they handle ANY page fault on x86!**

---

## 1. The x86 Hardware Interface

### CR2 Register (Control Register 2)

**What is CR2?**

> Special 64-bit register INSIDE the x86 CPU that holds the faulting virtual address!

**Physical reality:**

```
x86 CPU silicon contains:
├── General registers (RAX, RBX, RCX...)
├── Control registers (CR0, CR1, CR2, CR3...)
└── CR2 = Page fault address register

Built into CPU hardware!
64 flip-flops storing the address!
```

**How CR2 gets set:**

```
Program accesses: 0x12345678

MMU (hardware) detects fault
    ↓
MMU writes to CR2: CR2 = 0x12345678
    ↓ (Automatic! No software involved!)
Exception generated
    ↓
Handler can now read CR2 
```

**Reading CR2:**

```
Assembly instruction:
└── MOV RAX, CR2    ; Copy CR2 to RAX

C code in kernel:
unsigned long address;
address = read_cr2();  // Special function

Now: address = 0x12345678 
```

**Why CR2 is special:**

```
Read-only from software!
├── Can READ: MOV RAX, CR2 
└── Cannot WRITE: MOV CR2, RAX (would fault!)

Only MMU hardware can write it!
```

---

### Error Code (5-Bit Format)

**What is the error code?**

> 32-bit value pushed on stack by CPU, but only lower 5 bits used!

**Physical mechanism:**

```
When page fault happens:
1. CPU determines WHY it faulted
2. CPU builds 5-bit error code
3. CPU pushes onto stack
4. Handler reads from stack
```

**Error Code Format:**

```
Bit 31                                           Bit 0
  ↓                                                ↓
[Reserved (all 0)] ... [I/D][RSVD][U/S][W/R][P]
                         ↑     ↑    ↑    ↑   ↑
                        Bit 4  3    2    1   0
```

**Bit 0 (P - Present):**

```
P = 0: Page NOT present
└── Page table entry Present bit = 0
    Page not mapped in memory
    Need to allocate or load from disk

P = 1: Page IS present (protection violation)
└── Page exists but access not allowed
    Permission problem!

Example:
├── Error code = 0x00 → Bit 0 = 0 → Not present 
└── Error code = 0x01 → Bit 0 = 1 → Present 
```

**Bit 1 (W/R - Write/Read):**

```
W/R = 0: Read access (or instruction fetch)
└── Program was reading memory

W/R = 1: Write access
└── Program was writing to memory

Example:
└── Error code = 0x02 → Bit 1 = 1 → Write 

Important for Copy-On-Write:
└── Write to COW page → W/R = 1
    Kernel knows: Need to copy! 
```

**Bit 2 (U/S - User/Supervisor):**

```
U/S = 0: Supervisor mode (kernel, Ring 0)
└── Fault happened in kernel code

U/S = 1: User mode (Ring 3)
└── Fault happened in user program

Example:
└── Error code = 0x04 → Bit 2 = 1 → User mode 

Security check:
└── User accessing kernel memory?
    Check this bit! 
```

**Bit 3 (RSVD - Reserved):**

```
RSVD = 0: No reserved bit violation
└── Page table entry was valid

RSVD = 1: Reserved bit was set
└── Page table corrupted!
    Bits 51-62 in entry had 1
    Should NEVER happen! 

This indicates:
├── Hardware error
├── Kernel bug
└── Usually: PANIC! 
```

**Bit 4 (I/D - Instruction/Data):**

```
I/D = 0: Data access
└── Reading/writing data

I/D = 1: Instruction fetch
└── CPU trying to execute from address
    RIP pointing to faulting address

Example:
└── Error code = 0x10 → Bit 4 = 1 → Instruction 

Important for NX (No Execute):
└── If I/D = 1 and page has NX bit:
    Trying to execute data!
    Security violation!
```

**Common Error Code Values:**

```
0x00 (00000): Kernel read, not present
└── Demand paging or kernel bug

0x04 (00100): User read, not present
└── Demand paging (normal!) 

0x06 (00110): User write, not present
└── Demand paging or invalid access

0x07 (00111): User write, present
└── Copy-On-Write! 

0x15 (10101): User instruction, present + NX
└── Execute from NX page! Security violation! 
```

---

## 2. When Page Faults Happen

### Trigger Events (x86 MMU Detection)

**The x86 MMU watches EVERY memory access!**

### TRIGGER 1: Access to Unmapped Page

```
Program accesses: 0x12345000

x86 MMU:
└── "Let me check page table"
    Walk: PML4 → PDPT → PD → PT
    Look for entry for 0x12345000
    
    Entry NOT PRESENT! 
    Present bit (bit 0) = 0
    
    "PAGE FAULT!" 
    ├── Generate exception #14
    ├── Set CR2 = 0x12345000
    └── Error code bit 0 = 0 (not present)
```

### TRIGGER 2: Write to Read-Only Page

```
Program writes to: 0x00400000 (code section)

x86 MMU:
└── "Check page table for 0x00400000"
    Entry found: Present = 1 
    
    Check Write bit (bit 1):
    Write bit = 0 (read-only) 
    
    "Write to read-only! PAGE FAULT!" 
    ├── Generate exception #14
    ├── Set CR2 = 0x00400000
    ├── Error code bit 0 = 1 (present)
    └── Error code bit 1 = 1 (write)
```

### TRIGGER 3: User Access to Kernel Page

```
User program (Ring 3): Read from 0xFFFFFFFF80000000

x86 MMU:
└── "Check page table"
    Entry found: Present = 1 
    
    Check User bit (bit 2):
    User bit = 0 (kernel only) 
    Current mode: Ring 3 (user) 
    
    "User accessing kernel! PAGE FAULT!" 
    ├── Generate exception #14
    ├── Set CR2 = 0xFFFFFFFF80000000
    └── Error code bit 2 = 1 (user mode)
```

### TRIGGER 4: Execute from No-Execute Page

```
Program jumps to: 0x7FFFF000 (stack)

x86 MMU:
└── "Check page table"
    Entry found: Present = 1 
    
    Check NX bit (bit 63):
    NX bit = 1 (no execute) 
    Access type: Instruction fetch 
    
    "Execute from NX! PAGE FAULT!" 
    ├── Generate exception #14
    ├── Set CR2 = 0x7FFFF000
    └── Error code bit 4 = 1 (instruction)
```

### TRIGGER 5: Reserved Bit Set

```
x86 MMU walking page table:
└── "Reading entry"
    Entry = 0x0000000012345E67
    
    Check reserved bits (bits 51-62):
    Some reserved bit = 1 
    
    "Reserved bit! PAGE FAULT!" 
    ├── Generate exception #14
    └── Error code bit 3 = 1 (reserved)
```

---

## What x86 MMU Does (Hardware Actions)

### STEP 1: Set CR2 Register

```
Hardware writes fault address:

CR2 = 0x12345000

Physical action:
└── MMU circuit writes to CR2 flip-flops
    All 64 bits updated
    1 clock cycle! 
```

### STEP 2: Build Error Code

```
Combinational logic determines:

Bit 0 = page_present ? 1 : 0
Bit 1 = access_is_write ? 1 : 0
Bit 2 = cpu_in_user_mode ? 1 : 0
Bit 3 = reserved_bits_set ? 1 : 0
Bit 4 = instruction_fetch ? 1 : 0

Result: error_code = 0x07
└── (Present, Write, User, No RSVD, Data)

All in hardware! Instant! 
```

### STEP 3: Save State on Stack

```
x86 CPU automatically pushes:

Stack (kernel):
├── SS (if privilege change)
├── RSP (if privilege change)
├── RFLAGS
├── CS
├── RIP
└── Error Code ← PAGE FAULT SPECIFIC!

Complete state saved! 
```

### STEP 4: Look Up Handler

```
IDT (Interrupt Descriptor Table):

Entry 14 = Page Fault Handler
├── Segment: Kernel code
├── Offset: 0xFFFFFFFF81234567
└── DPL: 0 (kernel only)

CPU reads handler address 
```

### STEP 5: Switch to Ring 0

```
If in Ring 3 (user):
├── Switch to Ring 0 (kernel)
├── Change to kernel stack
└── CS = Kernel code segment

Now in kernel mode! 
```

### STEP 6: Jump to Handler

```
CPU sets:
└── RIP = 0xFFFFFFFF81234567

This is: arch/x86/kernel/entry_64.S
Which calls: do_page_fault() in fault.c

Now in x86 page fault handler! 
```

---

## 3. Complete Page Fault Flow

**Let me trace: User writing to Copy-On-Write page**

### Initial State

```
Program: Firefox (PID 1234)
State: Running (Ring 3)
Instruction: MOV [0x00601000], EAX
Value: EAX = 0x42424242

Page table for 0x00601000:
├── Present: 1 (mapped)
├── Writable: 0 (read-only) ← COW!
├── User: 1 (accessible)
└── Physical: 0x12345000
```

---

### Timeline: Complete Fault Handling

#### T = 0ns: CPU Executes Write

```
Firefox executes:
└── MOV [0x00601000], EAX

CPU:
└── "Write EAX to 0x00601000"
```

#### T = 1ns: MMU Translates

```
MMU:
└── "Translate 0x00601000"
    
    Read CR3: 0x02000000
    Walk tables: PML4→PDPT→PD→PT
    
    Entry: 0x0000000012345027
    ├── Physical: 0x12345000 
    ├── Present: 1
    └── Writable: 0 
```

#### T = 2ns: MMU Detects Violation

```
MMU:
└── Access: WRITE
    Page writable: NO 
    
    PROTECTION VIOLATION! 
```

#### T = 3ns: Start Fault Sequence

```
CPU hardware:

ACTION 1: Set CR2
└── CR2 = 0x00601000 

ACTION 2: Build error code
└── error_code = 0x07 (Present, Write, User)

ACTION 3: Look up IDT[14]
└── Handler = 0xFFFFFFFF81200000

ACTION 4: Save state
└── Push SS, RSP, RFLAGS, CS, RIP, Error Code

ACTION 5: Switch to Ring 0
└── CPL = 0, kernel stack active

ACTION 6: Jump to handler
└── RIP = 0xFFFFFFFF81200000
```

#### T = 100ns: Entry Code (entry_64.S)

```
Assembly:
├── PUSH all registers (RAX...R15)
├── Create struct pt_regs
├── MOV RDI, RSP (first arg = regs)
├── Get error code (second arg)
└── CALL do_page_fault

Now in: arch/x86/mm/fault.c! 
```

#### T = 200ns: do_page_fault() Runs

```
STEP 1: Read CR2
└── address = read_cr2()
    address = 0x00601000 

STEP 2: Parse error code
└── error_code = 0x07
    present = 1 
    write = 1 
    user = 1 

STEP 3: Get current task
└── task = Firefox (PID 1234)
    mm = task->mm

STEP 4: Validate address
└── 0x00601000 in user space? Yes 
    Address canonical? Yes 

STEP 5: x86 checks
└── SMAP? Not applicable 
    SMEP? Not applicable 
    Reserved bits? No (bit 3 = 0) 
```

#### T = 300ns: Call Generic Handler

```
fault.c calls:
└── handle_mm_fault(mm, 0x00601000, FAULT_FLAG_WRITE)

Entering: mm/memory.c (generic!) 

Generic handler:
├── Find VMA (0x00600000-0x00700000)
├── Check permissions (VM_WRITE allowed)
├── Examine page table (Present but read-only)
└── Detect: COPY-ON-WRITE! 

Handle COW:
├── Allocate new page: 0x98765000
├── Copy old→new: 4096 bytes
├── Update page table: 0x98765000, Writable
└── Decrement old page ref count
```

#### T = 10,000ns: Update x86 Page Table

```
arch/x86/mm/pgtable.c:
└── set_pte(entry, new_value)

New entry:
├── Physical: 0x98765000
├── Present: 1
├── Writable: 1 
└── User: 1

Write to page table:
└── *pte = 0x0000000098765007 
```

#### T = 10,100ns: Flush TLB

```
arch/x86/mm/tlb.c:
└── invlpg(0x00601000)

Assembly:
└── INVLPG [0x00601000]

TLB entry invalidated! 
Next access will use new entry!
```

#### T = 10,200ns: Return Path

```
mm/memory.c returns: VM_FAULT_MAJOR
fault.c returns to entry_64.S

Assembly:
├── Restore all registers
├── Execute: IRET

IRET pops:
├── Error code
├── RIP = 0x00400500 (original instruction!)
├── CS, RFLAGS
└── RSP, SS (back to user stack)

CPU switches:
├── Ring 0 → Ring 3 
└── Kernel stack → User stack 
```

#### T = 10,300ns: Retry Write

```
Firefox at: RIP = 0x00400500
Instruction: MOV [0x00601000], EAX
(SAME instruction!)

CPU executes AGAIN:
└── "Write EAX to 0x00601000"

MMU translates:
├── TLB miss (we flushed!)
├── Walk page table
├── Find: 0x0000000098765007
├── Physical: 0x98765000 (NEW!)
└── Writable: 1 

MMU check:
└── Write to writable? YES 
    No fault! 

Write succeeds:
└── Physical 0x98765000: Write 0x42424242 
    MMU sets Dirty bit 

SUCCESS! 
```

---

## Final State

```
Virtual 0x00601000:
BEFORE: → Physical 0x12345000 (shared, read-only)
AFTER:  → Physical 0x98765000 (private, writable) 

Firefox:
└── Has own copy 
    Can write freely 

Child process (if any):
└── Still has: 0x00601000 → 0x12345000
    Original unchanged 

Copy-On-Write complete! 
Total time: ~10 microseconds! 
```

---

## 4. x86-Specific Fault Cases

### Case 1: Kernel Page Fault

**Kernel code faults - different from user!**

```
Scenario: NULL pointer in kernel

kernel_function() {
    char *ptr = NULL;
    *ptr = 5;  ← Fault!
}

x86 CPU:
├── Access: 0x00000000
├── Mode: Ring 0 (kernel)
└── Error code: U/S bit = 0

fault.c detects:
└── "Kernel fault!"
    
    Is address valid?
    └── NULL? No! 
    
    Check exception table (fixup):
    └── No fixup found 
    
    KERNEL PANIC! 
    ├── Print Oops message
    ├── Show stack trace
    └── Halt or continue
```

**Why different?**

```
User fault: Kill process (send SIGSEGV) 
Kernel fault: Can't kill kernel! Must panic! 
```

---

### Case 2: SMAP Violation

**SMAP = Supervisor Mode Access Prevention**

> x86 feature preventing kernel from accidentally accessing user memory!

```
x86 feature (Broadwell+):
└── Prevents kernel bugs from becoming exploits

Scenario: Kernel bug

User buffer: 0x00601000
Kernel code:
    char *user_ptr = 0x00601000;
    *user_ptr = 'A';  ← Direct access!

x86 CPU with SMAP:
├── Mode: Ring 0 (kernel)
├── Access: 0x00601000 (user page!)
├── SMAP enabled in CR4
└── Page User bit = 1

Check:
└── Kernel accessing user page? YES! 
    SMAP violation! 

fault.c:
└── "SMAP violation!"
    "Kernel bug!"
    OOPS! Kernel panic! 
```

**Correct way:**

```
Use copy_from_user():

copy_from_user(kernel_buf, user_ptr, size);

This function:
├── Temporarily disables SMAP (STAC instruction)
├── Copies data
├── Re-enables SMAP (CLAC instruction)
└── Safe!

Wrong way:
└── memcpy(kernel_buf, user_ptr, size);
    SMAP violation! 
```

**Why SMAP?**

```
Security!
└── Prevents kernel exploits
    Attacker can't trick kernel into using
    user-controlled pointers 
```

---

### Case 3: SMEP Violation

**SMEP = Supervisor Mode Execution Prevention**

> x86 feature preventing kernel from executing user memory!

```
Scenario: Exploit attempt

Attacker:
├── Controls user page: 0x00601000
├── Puts shellcode there
└── Tricks kernel into jumping there

Attack:
    kernel_ptr = 0x00601000;
    kernel_ptr();  ← Execute user code!

x86 CPU with SMEP:
├── Mode: Ring 0 (kernel)
├── Instruction fetch from: 0x00601000
├── SMEP enabled in CR4
└── Page User bit = 1

Check:
└── Kernel executing user page? YES! 
    SMEP violation! 

fault.c:
└── "SMEP violation!"
    "Kernel exploit attempt!"
    OOPS! Panic! 
```

**SMEP prevents:**

```
Return-to-user attacks 
Kernel exploits 
Major security improvement! 
```

---

### Case 4: Reserved Bits

**Reserved bits must be zero!**

```
Page table entry bits 51-62:
└── Reserved (must be 0)
    If 1 → Violation!

Scenario: Corruption

Entry for 0x00601000:
└── Bit 52 = 1  (should be 0)

MMU:
└── "Reserved bit set!" 
    Error code bit 3 = 1

fault.c:
└── "Reserved bit violation!"
    This should NEVER happen! 
    
Causes:
├── Hardware error (RAM corruption)
├── Kernel bug
└── Attack attempt

Usually: KERNEL PANIC! 
```

---

### Case 5: NX Bit (No Execute)

**NX = Bit 63 in page table entry**

```
Scenario: Buffer overflow attack

Stack page: 0x7FFFF000
├── Contains injected shellcode
└── Page table: NX bit = 1

Attack: Return address overwritten
└── Function returns to: 0x7FFFF000

x86 CPU:
├── Instruction fetch from 0x7FFFF000
├── Page NX bit = 1 
└── Instruction fetch from NX page! 

Error code:
└── I/D bit = 1 (instruction fetch)

fault.c:
└── "Execute from NX page!"
    User mode? Yes (U/S = 1)
    
    Send SIGSEGV! 
    Process killed! 
    Attack prevented! 
```

**NX prevents:**

```
Stack execution attacks 
Heap execution attacks 
Buffer overflow exploits 
```

---

## 5. Physical Hardware Reality

### Inside x86 MMU Silicon

**The MMU is actual circuitry in the CPU!**

```
x86 CPU die contains:

MMU block:
├── Page table walker (state machine)
├── TLB (cache, 64-512 entries)
├── Permission checker (comparators)
└── CR2 register (64 flip-flops)
```

**On every memory access:**

```
1. Check TLB
   ├── Associative lookup
   ├── Fast! (1 cycle)
   └── Hit? Use translation

2. Walk page tables (if TLB miss)
   ├── Read CR3 (PML4 base)
   ├── Access RAM (4 memory reads)
   ├── State machine controls
   └── Takes ~100 cycles

3. Check permissions
   ├── Parallel comparators
   ├── Compare: access type vs entry bits
   └── All in 1 cycle! 

4. Allow or fault
   ├── Allowed: Complete access 
   └── Denied: Generate exception 
```

---

## CR2 Register Hardware

```
Physical implementation:

CR2 = 64 D flip-flops
├── Input: From MMU fault logic
├── Output: To register read port
└── Control: Write enable from MMU

On page fault:
└── MMU asserts write_enable
    Fault address loaded
    1 clock cycle! 
```

---

## Error Code Generation

```
Combinational logic circuit:

Inputs:
├── page_table_entry[63:0]
├── access_type[1:0] (read/write/execute)
├── cpu_mode (ring level)
└── reserved_check_result

Logic gates compute:
├── error_code[0] = entry.present
├── error_code[1] = (access == WRITE)
├── error_code[2] = (cpu_mode == RING_3)
├── error_code[3] = reserved_violation
└── error_code[4] = (access == EXEC)

Output: 5-bit error code
All combinational! Instant! 
```

---

## Page Fault Signal Path

```
Physical signal flow:

MMU fault detector
    ↓ (wire)
Exception logic
    ↓ (wire)
Interrupt controller (internal)
    ↓ (wire)
Exception #14 raised
    ↓ (wire)
IDT lookup hardware
    ↓ (wire)
Pipeline flush
    ↓ (wire)
RIP loaded with handler address
    ↓
Handler executes

All in nanoseconds! 
```

---

## 6. Connections to Other Topics

### Connection to Exceptions

**Page fault IS exception #14!**

```
From Exception Handling (TOPIC 2):
├── How CPU handles exceptions
├── IDT entry 14
└── Exception delivery mechanism

This topic (TOPIC 7):
├── WHAT exception #14 does
├── x86-specific details
└── CR2 and error code

Together:
└── Exception mechanism delivers to
    Page fault handler 
```

---

### Connection to mm/memory.c

**Two-layer design:**

```
arch/x86/mm/fault.c:
├── Reads CR2 (x86 hardware)
├── Parses error code (x86 format)
└── Extracts: address, flags

    ↓ (calls)

mm/memory.c (generic):
├── Receives: address, flags
├── Decides: COW, demand paging, kill
└── Platform independent! 
```

---

### Connection to Page Tables (TOPIC 8)

**fault.c needs to read page tables!**

```
fault.c:
└── "Why did this fault?"
    Check page table entry
    Examine: Present? Writable? User?

Entry format = x86-specific!
└── TOPIC 8 will explain! 
```

---

### Connection to TLB (TOPIC 9)

**After fixing fault, must flush TLB!**

```
fault.c flow:
├── Handle fault
├── mm/memory.c allocates page
├── Update page table
└── Flush TLB! 

Calls: arch/x86/mm/tlb.c
└── TOPIC 9 explains HOW! 
```

---

### Connection to Context Switch (TOPIC 4)

**CR3 affects page faults!**

```
Context switch:
├── Old process: CR3 = 0x02000000
└── New process: CR3 = 0x03000000

Page fault in new process:
├── Uses NEW page tables (CR3)
├── Different memory view!
└── fault.c walks current page tables 
```

---

## Summary

### What You've Mastered

**You now understand x86 page fault handling!**

```
- What it is: x86-specific entry point for memory faults
- Why needed: Bridge between x86 hardware and generic kernel
- CR2 register: Holds fault address, set by MMU hardware
- Error code: 5 bits explaining WHY fault happened
- Complete flow: From MMU detection to fault resolution
- x86 cases: SMAP, SMEP, NX, reserved bits
- Physical reality: MMU silicon, registers, logic gates
- Connections: Exceptions, mm/, page tables, TLB
```

---

### Key Takeaways

**The x86 Hardware Interface:**

```
CR2 Register:
├── 64-bit register in CPU
├── Automatically set by MMU
├── Read with: read_cr2()
└── Contains: Faulting virtual address

Error Code:
├── 5 bits of information
├── Bit 0: Present/not present
├── Bit 1: Write/read
├── Bit 2: User/kernel
├── Bit 3: Reserved violation
└── Bit 4: Instruction/data
```

**The Two-Layer Design:**

```
Layer 1 (fault.c):
└── x86-specific hardware interface
    Reads CR2, parses error code

Layer 2 (mm/memory.c):
└── Generic fault handling
    Demand paging, COW, decisions
```

**x86 Security Features:**

```
SMAP: Prevents kernel accessing user memory
SMEP: Prevents kernel executing user code
NX bit: Prevents executing data pages
All protect against exploits! 
```

---

### The Big Picture

**When you write to memory:**

```
Program: *ptr = value;
    ↓
MMU translates address
    ↓
Checks permissions
    ↓
If violation: PAGE FAULT!
    ├── MMU sets CR2 = address
    ├── MMU builds error code
    ├── CPU generates exception #14
    ├── fault.c reads CR2 & error code
    ├── Calls mm/memory.c
    ├── Fixes fault (COW, allocate, etc.)
    ├── Updates page table
    ├── Flushes TLB
    └── Returns to program

Program: "Write succeeded!" 
└── Never knew about fault! 
```

**All in ~10 microseconds!** 

---

### You're Ready!

With this knowledge, you can:
- Debug segmentation faults
- Understand kernel panics
- Trace memory issues
- Build memory management systems
- Understand security features

> **The x86 page fault handler is the gatekeeper of memory safety - you now know how it works at the hardware level!**

---

**Next up: Page Tables**

Ready to learn how x86 organizes memory into 4-level hierarchies? 
