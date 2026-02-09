# From Power Button to Running Kernel - The Boot Journey

> Ever wondered what happens when you press the power button? How the CPU goes from off to running a 64-bit kernel? Why it takes several seconds to boot?

**You're about to find out!**

---

## What's This About?

This is **arch/x86/boot/** and **arch/x86/kernel/head_64.S** - the complete journey from power-on to kernel!

Here's where it fits:

```
arch/x86/
├── boot/              ← Boot code (THIS!)
│   ├── header.S       ← 32-bit entry point
│   ├── compressed/    ← Kernel decompression
│   └── setup.c        ← Boot setup
│
└── kernel/
    ├── head_64.S      ← 64-bit transition (THIS!)
    └── head64.c       ← First C code
```

**This covers:**
- Power-on reset sequence
- BIOS/UEFI initialization (POST, memory detection)
- Bootloader operation (GRUB loading kernel)
- Mode transitions (16-bit → 32-bit → 64-bit)
- Enabling virtual memory (paging activation)
- Kernel initialization (becoming operational)

---

# The Fundamental Problem

**Physical Reality:**

```
When you press the power button:

CPU state:
├── Powered off (no electricity)
├── No memory of previous state
├── No idea what to do
└── Must start from SOMEWHERE 

But where?
├── No operating system loaded yet 
├── No kernel in memory yet 
├── No virtual memory yet 
└── Chicken and egg problem! 

Historical constraint:
├── Original IBM PC: Intel 8086 (1981)
├── Modern CPU: Must still be compatible! 
└── Must start in 16-bit Real Mode 
    Even though we want 64-bit! 

Can't skip to 64-bit:
├── Need to set up page tables first
├── Need to set up descriptors first
├── Need to enable features step-by-step
└── Must follow the sequence! 
```

**How does a CPU with no memory know what to do first?**

---

# Without a Boot Sequence

**Imagine no structured boot process:**

```
Power button pressed:

CPU powers on:
└── Execute instruction at... where? 
    No program counter set!
    No memory initialized!
    Random garbage!

Try to run 64-bit kernel immediately:
├── 64-bit mode enabled? NO 
├── Paging enabled? NO 
├── Kernel loaded? NO 
└── Instant crash! 

Try to use RAM:
├── RAM timing configured? NO 
├── Memory controller initialized? NO 
└── Random errors! 

Try to access disk:
├── Disk controller initialized? NO 
├── Can't load kernel! 
└── System stuck! 

Try to enable paging immediately:
├── Page tables exist? NO 
├── CR3 points where? Random! 
└── Triple fault! CPU resets! 

Complete chaos without order! 
```

**Need structured, ordered initialization!**

---

# The Gradual Transition Solution

**Power-on to 64-bit kernel in stages:**

```
Stage 1: Hardware Reset (0.1ms)
├── CPU powers on in 16-bit Real Mode 
├── Hardwired: First instruction at 0xFFFFFFF0 (ROM)
└── BIOS/UEFI starts executing 

Stage 2: BIOS/UEFI (0-5 seconds)
├── Initialize hardware (memory, devices)
├── Detect RAM amount (build E820 map)
├── Test hardware (POST)
└── Load bootloader from disk 

Stage 3: Bootloader (5.0-5.1 seconds)
├── Switch: 16-bit Real Mode → 32-bit Protected Mode 
├── Load kernel from disk into RAM
└── Jump to kernel entry point 

Stage 4: Kernel Boot (5.1-5.5 seconds)
├── Switch: 32-bit Protected → 64-bit Long Mode 
├── Enable paging (virtual memory) 
├── Initialize memory management
└── Start scheduler and userspace 

System operational! 

Each stage enables the next:
├── Can't load kernel without BIOS initializing disk
├── Can't run 64-bit without proper setup
└── Ordered progression breaks the deadlock! 
```

---

# 1. CPU Modes

## Real Mode (16-bit - 1978)

**Original 8086 operating mode:**

```
Why it exists:
└── IBM PC (1981) used Intel 8086
    All software written for 8086
    BIOS expects Real Mode
    Modern CPUs must start compatible! 

Characteristics:

Address calculation:
├── Segment:Offset notation
└── Physical = (Segment × 16) + Offset

Example:
CS:IP = 0xF000:0xFFF0
Physical = (0xF000 × 16) + 0xFFF0
Physical = 0xFFFF0 

Address space:
└── 20-bit addresses (1 MB total)
    0x00000 - 0xFFFFF

Registers:
└── 16-bit: AX, BX, CX, DX, SI, DI, BP, SP
    Segments: CS, DS, ES, SS

Memory protection:
└── NONE! 
    Any code can access anywhere
    Crash one program = crash system

Virtual memory:
└── NONE! 
    Physical addresses only
```

**What runs in Real Mode:**

```
- BIOS/UEFI (initially)
- First part of bootloader
- DOS (if anyone still runs it)

Then: Transition to Protected Mode! 
```

## Protected Mode (32-bit - 1985)

**Enhanced mode with protection:**

```
What it adds:

Address space:
└── 32-bit addresses (4 GB total)
    0x00000000 - 0xFFFFFFFF 

Registers:
└── 32-bit: EAX, EBX, ECX, EDX, ESI, EDI, EBP, ESP

Memory protection:
└── YES! 
    4 privilege levels (Ring 0-3)
    Segment limits and permissions

Virtual memory:
└── Optional paging
    Can enable if needed

Segmentation:
└── Uses descriptor tables
    GDT (Global Descriptor Table)
```

**GDT (Global Descriptor Table):**

```
Table of segment descriptors:

Entry 0: Null (required)
Entry 1: Code segment
├── Base: 0x00000000
├── Limit: 0xFFFFFFFF (4 GB)
└── Permissions: Executable, Ring 0

Entry 2: Data segment
├── Base: 0x00000000
├── Limit: 0xFFFFFFFF (4 GB)
└── Permissions: Writable, Ring 0

CPU uses selector (e.g., CS=0x08) to index GDT:
└── Selector 0x08 = Entry 1 = Code segment 
```

**What runs in Protected Mode:**

```
- GRUB bootloader
- Early kernel setup (32-bit portion)

Then: Transition to Long Mode! 
```

## Long Mode (64-bit - 2003)

**Modern 64-bit mode:**

```
What it adds:

Address space:
└── 48-bit virtual (current CPUs)
    256 TB addressable
    57-bit with 5-level paging (newest CPUs)

Registers:
├── 64-bit: RAX, RBX, RCX, RDX, RSI, RDI, RBP, RSP
└── Additional: R8-R15 (8 more!) 

Memory protection:
└── YES! 
    Ring 0 (kernel) / Ring 3 (user)

Virtual memory:
└── MANDATORY! 
    Paging always enabled
    Can't disable!

Segmentation:
└── Mostly disabled 
    Flat memory model
    CS/DS still exist but simplified
```

**What runs in Long Mode:**

```
- 64-bit Linux kernel
- All modern operating systems
- Your applications
- EVERYTHING!
```

---

# 2. The Complete Boot Flow

## Stage 1: Power-On Reset (T=0-1ms)

**Hardware initialization:**

```
T=0ms: Power button pressed

Power supply:
├── Stabilizes voltages (3.3V, 5V, 12V)
├── Asserts "Power Good" signal
└── Motherboard releases reset 

CPU receives power:
├── Internal reset sequence 
├── All flip-flops reset
└── Enters known state 

CPU reset state:
├── Mode: Real Mode (16-bit) 
├── CS = 0xF000 (segment)
├── IP = 0xFFF0 (offset)
└── CS.base = 0xFFFF0000 (hidden!)

First instruction address:
Physical = CS.base + IP
Physical = 0xFFFF0000 + 0xFFF0
Physical = 0xFFFFFFF0 

This is: RESET VECTOR! 
```

**First instruction fetch:**

```
CPU fetches from 0xFFFFFFF0:

Motherboard address decoder:
└── 0xFFFFFFF0 → Routes to BIOS ROM 

BIOS ROM chip:
└── Contains: JMP FAR 0xF000:0xE05B
    Jump to BIOS entry point 

BIOS starts executing! 
```

## Stage 2: BIOS POST (T=1ms-5s)

**Power-On Self-Test:**

```
BIOS performs hardware checks:

Test 1: CPU
├── Register test
├── Basic math operations
└── CPU functional 

Test 2: Memory
├── Write test patterns
├── Read and verify
├── Detect amount: 8 GB 
└── Find bad regions (if any)

Test 3: Video
├── Initialize GPU
├── Display POST screen
└── Can show errors 

Test 4: Storage
├── Detect hard drives: 1 found 
├── Detect CD/DVD drives
└── Storage ready 

Test 5: Input
├── Keyboard controller 
├── Mouse controller 
└── Can interact with BIOS

All pass: Continue 
Any fail: Beep codes! 
```

**E820 memory map creation:**

```
BIOS scans physical address space:

0x00000000 - 0x0009FFFF: RAM (640 KB) 
0x000A0000 - 0x000FFFFF: Reserved (VGA)
0x00100000 - 0xDFFFFFFF: RAM (3.5 GB) 
0xE0000000 - 0xEFFFFFFF: PCIe devices
0xFEC00000 - 0xFEC00FFF: I/O APIC 
0xFEE00000 - 0xFEE00FFF: Local APIC 
0x100000000 - 0x1FFFFFFFF: RAM (4 GB) 

Store map at: 0x00002D00 

Kernel will read this in TOPIC 10! 
```

**Device initialization:**

```
BIOS sets up hardware:

PCI bus scan:
├── Find all PCI devices
├── Assign memory regions (BARs)
└── Example: Network card at 0xF8000000 

Interrupt controllers:
├── 8259 PIC (legacy)
└── Or APIC (modern) 

Timers:
└── PIT (Programmable Interval Timer) 

Keyboard ready 
```

## Stage 3: Bootloader Load (T=5s)

**Finding and loading GRUB:**

```
BIOS boot device search:

Try order:
1. Hard drive 
2. CD/DVD
3. USB
4. Network (PXE)

Read hard drive sector 0:
├── Master Boot Record (MBR)
├── 512 bytes at LBA 0
└── Last 2 bytes: 0x55AA? (boot signature)

Load MBR to RAM:
├── Read 512 bytes from disk
├── Copy to: Physical 0x7C00
└── MBR loaded! 

Execute MBR:
└── JMP 0x0000:0x7C00 
    Bootloader stage 1 running!
```

**GRUB stage 1 (MBR):**

```
Limited space (446 bytes):

Task: Load GRUB stage 2
├── Read GRUB from partition
├── Load to: 0x8000 (example)
└── JMP 0x0000:0x8000 

GRUB main code now running!
```

**GRUB stage 2:**

```
Still in Real Mode initially!

Switch to Protected Mode:

Step 1: Create GDT
Entry 0: Null descriptor
Entry 1: Code segment (4 GB, Ring 0)
Entry 2: Data segment (4 GB, Ring 0)

Step 2: Load GDT
LGDT [gdt_descriptor] 

Step 3: Enable Protected Mode
MOV EAX, CR0
OR EAX, 0x01        ; Set PE bit
MOV CR0, EAX        

Step 4: Far jump
JMP 0x08:protected_mode_entry
└── CS=0x08 (code segment selector)

Now in Protected Mode! 
32-bit code! 
```

**Loading the kernel:**

```
GRUB in Protected Mode (32-bit):

Read GRUB config:
└── /boot/grub/grub.cfg
    linux /boot/vmlinuz-6.1.0 root=/dev/sda1
    initrd /boot/initrd.img-6.1.0

Load kernel image:
├── Read /boot/vmlinuz-6.1.0
├── Parse ELF headers
├── Size: ~10 M
└── Load to: Physical 0x01000000 

Load initrd:
├── Read /boot/initrd.img
└── Load to: Physical 0x02000000 

Prepare boot parameters:
├── E820 map location
├── Command line
├── Initrd location and size
└── Structure ready 

Jump to kernel:
JMP 0x01000000 

Kernel takes over! 
```

## Stage 4: Kernel Boot (T=5.1s)

**32-bit kernel entry:**

```
arch/x86/boot/header.S:

CPU state:
├── Mode: Protected Mode (32-bit)
├── Paging: DISABLED 
└── Physical addresses only

Kernel at:
└── Physical: 0x01000000

Tasks:
├── Decompress kernel (if compressed)
├── Set up page tables
└── Prepare for Long Mode 
```

**Setting up paging:**

```
Page tables compiled into kernel:
(From TOPIC 10!)

init_pml4:  PML4 table
init_pdpt:  PDPT table
init_pd:    PD table

Two mappings:
1. Identity: Virtual 0x00000000 = Physical 0x00000000
2. High: Virtual 0xFFFFFFFF80000000 = Physical 0x00000000

Point CR3:
MOV EAX, init_pml4
MOV CR3, EAX 

Page tables ready! 
```

**Enabling Long Mode:**

```
arch/x86/boot/compressed/head_64.S:

Step 1: Enable PAE
MOV EAX, CR4
OR EAX, 0x20        ; PAE bit
MOV CR4, EAX 

Step 2: Set Long Mode Enable
MOV ECX, 0xC0000080 ; EFER MSR
RDMSR
OR EAX, 0x100       ; LME bit
WRMSR 

Step 3: Enable Paging
MOV EAX, CR0
OR EAX, 0x80000000  ; PG bit
MOV CR0, EAX 

- PAGING ENABLED! 
- LONG MODE ACTIVE! 

Step 4: Load 64-bit GDT
LGDT [gdt64_descriptor] 

Step 5: Far jump to 64-bit code
JMP 0x10:long_mode_entry 

Now in 64-bit mode! 
```

**Jump to high kernel:**

```
Currently executing:
└── Physical 0x01002000 (identity mapped)

Need high kernel addresses:

LEA RAX, [high_kernel_entry]
; RAX = 0xFFFFFFFF81000000

JMP RAX 

Now at:
├── Virtual: 0xFFFFFFFF81000000
└── Physical: 0x01000000 (via high map)

Remove identity mapping:
└── PML4[0] = 0 
    Only high kernel remains 
```

**First C code:**

```
arch/x86/kernel/head64.c → x86_64_start_kernel():

First C function! 

Initialize:
├── Copy boot parameters 
├── Early console (early_printk) 
└── Parse command line 

Call main kernel:
start_kernel(); 
```

## Stage 5: Main Kernel Init (T=5.1-5.5s)

**Massive initialization:**

```
init/main.c → start_kernel():

Memory (TOPIC 10):
├── Parse E820 map 
├── Initialize memblock 
├── Create proper page tables 
├── Initialize buddy allocator 
└── Initialize slab allocators 

Architecture (x86):
├── Detect CPU (TOPIC 5) 
├── Initialize APIC (TOPIC 6)
└── Set up IDT (TOPIC 2) 

Scheduler (TOPIC 4):
├── Initialize scheduler 
└── Create init process (PID 1) 

Devices:
├── Initialize timers 
├── Initialize interrupts 
└── Probe devices 

Filesystem:
├── Mount initrd 
└── Execute /init 

System operational! 
```

## Stage 6: Userspace (T=5.5s+)

**Init process starts:**

```
Kernel starts /sbin/init (or systemd):

Init (PID 1):
├── Mount real root filesystem 
├── Start system services
├── Start getty (login) 
└── System ready for login! 

Boot complete! 
```

---

# 3. Mode Transitions

## Real → Protected Mode

**The first transition:**

```
Before:
├── Real Mode (16-bit)
├── Segment:Offset addressing
└── 1 MB address space

Transition code:

1. Create GDT in memory:
   gdt:
       Entry 0: Null
       Entry 1: Code (0-4GB, Ring 0)
       Entry 2: Data (0-4GB, Ring 0)

2. Load GDT:
   LGDT [gdt_descriptor] 

3. Enable Protected Mode:
   MOV EAX, CR0
   OR EAX, 0x01        ; PE bit
   MOV CR0, EAX 

4. Far jump (flush pipeline):
   JMP 0x08:prot_mode  ; CS=0x08
   
After:
├── Protected Mode (32-bit) 
├── GDT-based addressing 
└── 4 GB address space 
```

## Protected → Long Mode

**The second transition:**

```
Before:
├── Protected Mode (32-bit)
├── Paging disabled
└── 4 GB address space

Transition code:

1. Create page tables:
   PML4, PDPT, PD ready 

2. Load CR3:
   MOV EAX, pml4
   MOV CR3, EAX 

3. Enable PAE:
   MOV EAX, CR4
   OR EAX, 0x20
   MOV CR4, EAX 

4. Enable Long Mode:
   MOV ECX, 0xC0000080  ; EFER
   RDMSR
   OR EAX, 0x100        ; LME
   WRMSR 

5. Enable Paging:
   MOV EAX, CR0
   OR EAX, 0x80000000   ; PG
   MOV CR0, EAX 
   
   Long Mode activates! 

6. Load 64-bit GDT:
   LGDT [gdt64] 

7. Far jump to 64-bit:
   JMP 0x10:long_mode 

After:
├── Long Mode (64-bit) 
├── Paging enabled (mandatory) 
├── 256 TB address space 
└── R8-R15 registers available 
```

---

# 4. Physical Reality

## Reset Vector Hardware

**What really happens:**

```
Power-on:

CPU reset circuit:
├── All flip-flops reset 
├── Control registers cleared
└── Known state achieved 

Hardwired state:
├── CS.base = 0xFFFF0000 (fixed in silicon!)
├── IP = 0xFFF0 (fixed in silicon!)
└── First fetch = 0xFFFFFFF0 

Motherboard routing:
├── Address 0xFFFFFFF0 detected
├── Chipset routes to BIOS ROM chip 
└── Not RAM! 

BIOS ROM:
├── Flash memory chip
├── Contains firmware
└── Returns instruction bytes 

Electrical signals:
└── ROM → CPU data bus 
    Instruction executed 
```

## Mode Switching Hardware

**Protected Mode enable:**

```
MOV CR0, EAX instruction:

CPU microcode:
├── Write to CR0 flip-flops 
├── PE bit: 0 → 1 
└── Mode switch signal 

Internal CPU changes:
├── Segmentation unit: Activates 
├── GDT lookup logic: Enables 
├── Protection checks: Enables 
└── Address calculation: Changes 

Silicon state changed! 
```

**Long Mode + Paging:**

```
MOV CR0, EAX (PG bit set):

CPU microcode:
├── Activate MMU 
├── TLB: Clears 
├── Page walker: Activates 
└── Long Mode: Activates 

Next instruction:
├── Virtual address used
├── MMU translates 
└── Paging working! 

Hardware transformation! 
```

---

# 5. Connections to Other Topics

## Connection to Memory Init

```
TOPIC 10: Memory initialization
└── Creates proper page tables
    Parses E820 map
    Sets up direct mapping

TOPIC 13: Provides foundation 
└── BIOS creates E820 map
    Bootloader loads kernel
    Initial page tables exist

Boot enables memory init! 
```

## Connection to CPU Detection

```
TOPIC 5: CPU detection (CPUID)
└── Detects features

TOPIC 13: Must boot first! 
└── Can't detect before CPU on!
    Long Mode needed for full CPUID

Boot enables detection! 
```

## Connection to Interrupts

```
TOPICS 2 & 3: Interrupt handling
└── IDT, handlers

TOPIC 13: Sets up initial IDT 
└── Creates initial handlers
    Enables interrupts

Boot enables interrupt system! 
```

## Connection to Context Switching

```
TOPIC 4: Context switching
└── Requires Long Mode, paging, scheduler

TOPIC 13: Provides all! 
└── Enables Long Mode 
    Enables paging 
    Initializes scheduler 

Boot enables multitasking! 
```

---

# Summary

## What You've Mastered

**You now understand the boot sequence!**

```
- What it is: Power-on to kernel journey
- Why gradual: Compatibility + technical requirements
- Three modes: Real, Protected, Long
- Complete flow: BIOS → Bootloader → Kernel
- Mode transitions: How to switch (CR0, CR4, EFER)
- Physical reality: Reset vector, ROM, hardware
- Connections: Enables all other kernel features
```

---

## Key Takeaways

**Historical compatibility:**

```
Why start in Real Mode?
└── IBM PC (1981) used 8086
    BIOS expects Real Mode
    Modern CPUs must be compatible 

Even though:
└── We want 64-bit immediately!
    Can't skip the sequence! 
```

**Gradual transitions:**

```
Can't jump to 64-bit directly:
├── Need page tables first
├── Need GDT setup first
├── Need features enabled in order
└── Must follow sequence! 

The path:
Real (16-bit) → Protected (32-bit) → Long (64-bit)
```

**Each stage enables the next:**

```
BIOS:
└── Initializes hardware
    Creates E820 map
    Loads bootloader 

Bootloader:
└── Switches to Protected Mode
    Loads kernel
    Jumps to kernel 

Kernel:
└── Switches to Long Mode
    Enables paging
    Becomes operational 
```

**Reset vector is hardwired:**

```
CPU always starts at 0xFFFFFFF0:
├── Fixed in silicon!
├── Points to BIOS ROM
└── First instruction fetch 

Motherboard routes:
└── 0xFFFFFFF0 → BIOS chip 
```

---

## The Big Picture

**Complete timeline:**

```
T=0ms: Power button
└── CPU powers on in Real Mode

T=0-5s: BIOS POST
└── Test hardware, build E820, init devices

T=5s: Bootloader (GRUB)
└── Switch to Protected Mode
    Load kernel from disk

T=5.1s: Kernel entry
└── Switch to Long Mode
    Enable paging
    64-bit active!

T=5.1-5.5s: Kernel init
└── Memory, interrupts, scheduler
    All subsystems ready

T=5.5s+: Userspace
└── Init process, services, login
    System operational! 

From power-on to login in ~5-10 seconds! 
```

**Mode transitions:**

```
Real Mode (16-bit):
├── BIOS compatibility
├── 1 MB address space
└── No protection

↓ (Set CR0.PE, load GDT)

Protected Mode (32-bit):
├── 4 GB address space
├── Memory protection
└── Bootloader runs here

↓ (Set CR4.PAE, EFER.LME, CR0.PG)

Long Mode (64-bit):
├── 256 TB address space
├── Mandatory paging
└── Kernel runs here! 

Each transition adds capabilities! 
```

---

## You're Ready!

With this knowledge, you can:
- Understand complete boot process
- Debug boot failures
- Write bootloader code
- Understand why boot takes time
- Trace kernel initialization

> **The boot sequence is the foundation - you now know how x86 goes from power button to running kernel!**

---

**The journey from off to on!** 
