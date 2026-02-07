# Changing Page Properties on the Fly - Runtime Attribute Control

> Ever wondered how kernel code becomes read-only after loading? How graphics drivers get fast frame buffer access? How W^X security works?

**You're about to find out!**

---

## What's This About?

This is **arch/x86/mm/pageattr.c** - how x86 changes page properties after they're already mapped!

Here's where it fits:

```
arch/x86/
└── mm/                    ← x86 memory management
    ├── fault.c            ← x86 page fault entry
    ├── pgtable.c          ← Page table operations
    ├── tlb.c              ← TLB management
    ├── init.c             ← Memory initialization
    ├── pageattr.c         ← Page attributes (THIS!)
    └── ioremap.c          ← I/O memory mapping
```

**This file handles:**
- Changing permissions (writable ↔ read-only, executable ↔ no-execute)
- Changing caching (normal ↔ uncacheable ↔ write-combining)
- Enforcing W^X security (writable XOR executable, never both!)
- Kernel code protection (make kernel read-only)
- Device memory optimization (special caching for graphics)

---

# The Fundamental Problem

**Physical Reality:**

```
Page properties need to change over time:

Scenario 1: Loading executable code
├── Initially: Need writable (to load code)
├── After load: Need executable (to run code)
└── But NOT both at once! (security!)

Scenario 2: Graphics frame buffer
├── Physical: 0xE0000000 (GPU memory)
├── Default: Uncacheable (safe but slow) 
└── Optimal: Write-combining (25× faster!) 

Scenario 3: Kernel hardening
├── After boot: Kernel code loaded
├── Security: Make all kernel code read-only 
└── Prevents: Kernel exploits from patching code 

All need runtime changes to already-mapped pages!
```

**Can't predict all needs at mapping time - must change dynamically!**

---

# Without Runtime Attribute Changes

**Imagine static-only attributes:**

```
Loading kernel module:

Step 1: Allocate memory
addr = vmalloc(module_size);

Set attributes at map time:
└── Writable?  (need to load code)
    Executable?  (need to run code)
    Both at once! 

Result: Security hole!
├── Module loaded into writable+executable memory 
├── Attacker overwrites module code 
└── Malicious code executes! 

W^X violated! Critical vulnerability! 


Graphics driver scenario:

Map GPU frame buffer:
Physical 0xE0000000 → Virtual 0xFFFFC90000000000

Default caching: Uncacheable (UC)
└── Safe for all devices 

Drawing 1920×1080 pixels:
for (i = 0; i < 2,073,600; i++)
    framebuffer[i] = color;

Each write with UC:
├── Goes directly to GPU 
├── No caching 
└── ~200 cycles per write 

Total: 2,073,600 × 200 = 414,720,000 cycles 
Time: ~0.4 seconds at 1 GHz 

Cannot change to write-combining! 
Graphics rendering glacially slow! 


Kernel protection scenario:

Kernel code sitting in memory:
├── Must be writable initially (to load) 
├── Cannot make read-only later 
└── Stays writable forever! 

Kernel exploit:
├── Finds vulnerability
├── Overwrites kernel code 
├── System compromised! 

No way to harden after boot! 
```

**Static attributes can't adapt to changing needs!**

---

# The Runtime Change Solution

**Dynamic attribute modification:**

```
Module loading (secure):
1. Allocate writable, non-executable:
   addr = vmalloc(size);
   Attributes: R/W=1, NX=1 

2. Load module code:
   memcpy(addr, module_binary, size);
   Can write because writable! 

3. Make read-only:
   set_memory_ro(addr, pages);
   Attributes: R/W=0, NX=1 

4. Make executable:
   set_memory_x(addr, pages);
   Attributes: R/W=0, NX=0 

Final: Read-only + Executable 
Never writable + executable! 
W^X enforced! Security maintained! 


Graphics optimization:
1. Map frame buffer (default UC):
   addr = ioremap(0xE0000000, size);
   Cache: Uncacheable 

2. Change to write-combining:
   set_memory_wc(addr, pages);
   Cache: Write-combining 

Drawing now:
├── Writes buffered in CPU 
├── Burst to GPU in chunks 
└── ~8 cycles per write! 

Total: 2,073,600 × 8 = 16,588,800 cycles 
Time: ~0.016 seconds 

25× faster! Graphics smooth! 


Kernel hardening:
1. After boot complete:
   All kernel code loaded 

2. Protect kernel text:
   set_memory_ro(kernel_text_start, text_pages);
   Kernel code now read-only! 

3. Protect system call table:
   set_memory_ro(sys_call_table, 1);
   Cannot be modified! 

Exploit attempts:
├── Try to overwrite kernel code
├── Page fault! Write to read-only! 
└── Attack prevented! 

Runtime hardening works! 
```

---

# 1. Permission Changes

## Read-Only ↔ Read-Write

**Making pages read-only:**

```
set_memory_ro(unsigned long addr, int numpages);

What it does:
For each page:
1. Walk page tables (PML4→PDPT→PD→PT)
2. Find page table entry
3. Clear R/W bit (bit 1):
   Entry &= ~0x02
   R/W = 0 (read-only) 
4. Write back entry
5. Flush TLB: invlpg(addr)

Result: Page now read-only! 
```

**Example: Protecting kernel module**

```
Module loaded at: 0xFFFFC90000000000
Size: 1 MB (256 pages)

Initial state:
Entry = 0x0000000012345007
└── R/W = 1 (writable)

Call: set_memory_ro(0xFFFFC90000000000, 256);

For each page:
├── Find entry
├── Clear bit 1: 0x007 → 0x005
└── New entry: 0x0000000012345005

Result:
└── R/W = 0 (read-only) 

Attempt to modify:
module_code[0] = 0x90;  // Write instruction

CPU detects
├── Write to R/W=0 page! 
├── Page fault! Error code = 0x03 (write protection)
└── SIGSEGV sent! 

Module cannot modify itself! 
```

## No-Execute ↔ Executable

**The NX bit (bit 63):**

```
set_memory_nx(unsigned long addr, int numpages);
set_memory_x(unsigned long addr, int numpages);

What NX does:
├── NX=1: Cannot execute instructions 
└── NX=0: Can execute 

For each page:
1. Find page table entry
2. Set/clear bit 63:
   NX: Entry |= (1ULL << 63)
   X:  Entry &= ~(1ULL << 63)
3. Write back
4. Flush TLB
```

**Example: Data page protection**

```
Heap allocation:
ptr = malloc(4096);
Virtual: 0x00601000

Initial attributes (modern Linux):
├── R/W = 1 (writable) 
└── NX = 1 (no execute) 

Attacker writes shellcode:
memcpy(ptr, shellcode, 100);
└── Succeeds (page writable) 

Attacker tries to execute:
((void(*)())ptr)();  // Jump to shellcode

CPU detects:
├── Instruction fetch from NX=1 page! 
├── Page fault! Error code = 0x11 (instruction fetch, NX)
└── SIGSEGV sent! 

Shellcode prevented! 
Security works! 
```

## W^X Policy Enforcement

**Write XOR Execute - Never Both:**

```
The rule:
Page can be EITHER:
├── Writable (for data) 
└── Executable (for code) 

But NEVER both simultaneously! 

Why?
Writable + Executable = Code injection possible! 
└── Attacker writes malicious code + runs it! 

Enforcement in pageattr.c:

set_memory_rw(addr, pages):
├── Set R/W = 1 (writable)
├── Automatically set NX = 1 (no execute) 
└── Cannot be executable!

set_memory_x(addr, pages):
├── Clear NX = 0 (executable)
├── Automatically clear R/W = 0 (read-only) 
└── Cannot be writable!

Result: W^X automatically enforced! 
```

**Real-world impact:**

```
Without W^X
Stack: Writable + Executable 
├── Buffer overflow writes shellcode 
├── Return address points to stack 
└── Shellcode executes! 
    Classic stack smashing! 

With W^X:
Stack: Writable + No-Execute 
├── Buffer overflow writes shellcode  (can still write)
├── Return address points to stack 
├── Try to execute: Page fault! 
└── Attack prevented! 
```

---

# 2. Cache Attribute Changes

## The Four Cache Types

**Write-Back (WB) - Normal RAM:**

```
Default for regular memory:

Reads: Cached in L1/L2/L3 
Writes: Go to cache first 
└── Written to RAM later (on eviction)

Performance:
├── Multiple writes to same location
├── Only last write goes to RAM 
└── Fastest! ~4 cycles per access 

Example:
counter = 0;
for (i = 0; i < 1000; i++)
    counter++;

With WB:
├── First write: RAM + cache (~100 cycles)
├── Next 999: Cache only (~4 cycles each)
├── Final writeback: RAM (~100 cycles)
└── Total: ~4,100 cycles 

Without cache: 1000 × 100 = 100,000 cycles 
24× faster! 
```

**Uncacheable (UC) - Device Registers:**

```
For memory-mapped device registers:

Reads: Directly from device 
Writes: Directly to device 
No caching! 

Why needed?
Device register at 0xF8000000 (network card status):

With caching (WRONG!):
├── Read register → Cached value 
├── Device changes state
├── Read again → Stale cached value! 
└── Wrong status! Device malfunction! 

With UC (CORRECT!):
├── Read register → From device 
├── Device changes state
├── Read again → From device 
└── Current status! Always correct! 

Cost: ~200 cycles per access 
But necessary for correctness! 
```

**Write-Combining (WC) - Frame Buffers:**

```
Perfect for graphics memory:

Reads: Uncached (from device)
Writes: Buffered then burst to device 

How it works:
Write pixel[0]: → WC buffer
Write pixel[1]: → WC buffer
Write pixel[2]: → WC buffer
...
Write pixel[63]: → WC buffer (full!)
└── Burst all 64 writes to GPU! 

Performance vs UC:
UC: 64 × 200 cycles = 12,800 cycles 
WC: 1 burst × 500 cycles = 500 cycles 

25× faster! 

Perfect for:
├── Graphics frame buffers 
├── PCIe device memory 
└── Sequential large writes 
```

**Write-Through (WT) - Less Common:**

```
Hybrid approach:

Reads: Cached 
Writes: Go to cache AND memory 

Use case: Shared memory with hardware
├── Software needs fast access (cache)
└── Hardware needs to see writes immediately

Slower than WB, faster than UC ⚖️
```

## PAT (Page Attribute Table)

**Extending cache control:**

```
Problem: Only 2 bits in page table entry
├── PCD (bit 4) = Page Cache Disable
└── PWT (bit 3) = Page Write-Through

2 bits = 4 combinations:
├── 00: Write-back
├── 01: Write-through
├── 10: Uncacheable
└── 11: Uncacheable

Missing: Write-combining! 

Solution: PAT adds third bit!
└── PAT bit (bit 7 in page table)

3 bits = 8 combinations 
```

**PAT MSR (Model-Specific Register):**

```
PAT MSR (0x277) defines mapping:

Index 0 (PAT=0, PCD=0, PWT=0): WB
Index 1 (PAT=0, PCD=0, PWT=1): WT
Index 2 (PAT=0, PCD=1, PWT=0): UC-
Index 3 (PAT=0, PCD=1, PWT=1): UC
Index 4 (PAT=1, PCD=0, PWT=0): WB
Index 5 (PAT=1, PCD=0, PWT=1): WT
Index 6 (PAT=1, PCD=1, PWT=0): UC-
Index 7 (PAT=1, PCD=1, PWT=1): WC 

Write-combining available! 
```

**Changing cache attributes:**

```
set_memory_wc(addr, pages):

For each page:
1. Find page table entry
2. Set PAT/PCD/PWT bits for WC (index 7):
   ├── Set PAT (bit 7) = 1
   ├── Set PCD (bit 4) = 1
   └── Set PWT (bit 3) = 1
3. Write entry
4. Flush TLB
5. CRITICAL: Flush CPU caches! 
   └── WBINVD instruction

Why flush caches?
├── Page was WB (cached)
├── Now changed to WC (uncacheable)
├── Old cached data still in L1/L2/L3 
└── Must invalidate! 
```

---

# 3. Complete Example: Graphics Frame Buffer

## Optimizing GPU Memory Access

**Initial state:**

```
GPU card installed:
├── Frame buffer: Physical 0xE0000000
├── Size: 16 MB (4096 pages)
└── Type: PCI device memory (MMIO)

Driver loads:
addr = ioremap(0xE0000000, 16 MB);
└── Virtual: 0xFFFFC90000000000

Default mapping (ioremap):
└── Cache: UC (uncacheable) 
    Safe default for devices! 
```

**Step 1: Measure performance with UC**

```
Drawing screen (1920×1080 pixels):

for (i = 0; i < 2,073,600; i++)
    framebuffer[i] = 0xFF0000;  // Red

Each write with UC:
├── CPU → Memory controller → PCIe → GPU
└── ~200 cycles 

Total: 2,073,600 × 200 = 414,720,000 cycles
At 1 GHz: ~0.4 seconds 
Frame rate: 2.5 FPS

Unacceptably slow! 
```

**Step 2: Change to write-combining**

```
Driver calls:
set_memory_wc(0xFFFFC90000000000, 4096);

For each of 4096 pages:
├── Walk to page table entry
├── Read current entry:
│   Entry = 0x00000000E000001B
│   Cache bits: PAT=0, PCD=1, PWT=1 (UC)
│
├── Modify for WC:
│   Set PAT (bit 7) = 1
│   Entry = 0x00000000E000009B
│   Cache bits: PAT=1, PCD=1, PWT=1 (WC) 
│
├── Write entry atomically
├── Flush TLB: invlpg(addr)
└── Continue to next page...

After all pages:
└── WBINVD (flush all CPU caches) 
```

**Step 3: Multi-CPU TLB shootdown**

```
System has 4 CPUs:

CPU 0 (running driver):
├── Modified all 4096 page table entries 
├── Flushed local TLB 
└── Now send IPI to CPUs 1, 2, 3 

CPUs 1, 2, 3 receive IPI:
├── Interrupt handler executes
├── Each flushes range:
│   for (page = start; page < end; page += 4KB)
│       invlpg(page);
│
└── Acknowledge to CPU 0 

CPU 0 waits for all ACKs:
└── All CPUs synchronized! 
```

**Step 4: Measure new performance**

```
Drawing same screen with WC:

for (i = 0; i < 2,073,600; i++)
    framebuffer[i] = 0xFF0000;

CPU with WC:
├── Writes buffered (64 writes per burst)
├── PCIe burst transfers 
└── ~8 cycles per write! 

Total: 2,073,600 × 8 = 16,588,800 cycles
At 1 GHz: ~0.016 seconds 
Frame rate: 60+ FPS 

25× faster! 
Smooth graphics! 
```

---

# 4. Kernel Code Protection

## Hardening After Boot

**Making kernel read-only:**

```
After boot completes:

Kernel memory layout:
├── .text (code): 10 MB at 0xFFFFFFFF81000000
├── .rodata (constants): 2 MB
└── .data (variables): 5 MB

Initially all writable (needed for loading) 

Hardening:
set_memory_ro(kernel_text_start, text_pages);
set_memory_ro(kernel_rodata_start, rodata_pages);

Result:
├── .text: Read-only 
├── .rodata: Read-only 
└── .data: Still writable 

Protects against:
├── Kernel rootkits 
├── Kernel exploits 
└── Code injection 
```

**System call table protection:**

```
sys_call_table:
├── Array of function pointers
├── sys_call_table[1] = sys_write
├── sys_call_table[2] = sys_open
└── etc.

Classic rootkit attack:
sys_call_table[1] = malicious_write;
└── Intercepts all writes! 

Protection:
set_memory_ro(sys_call_table, 1);

Now read-only! 

Attack attempt:
sys_call_table[1] = malicious_write;
├── Write to read-only page! 
├── Page fault! 
└── Kernel panic! Attack failed! 
```

**Live patching (temporary unprotect):**

```
Security update needs to patch kernel:

1. Temporarily unprotect:
   set_memory_rw(patch_addr, 1);
   └── R/W = 1 (writable) 

2. Apply patch:
   memcpy(patch_addr, new_code, size);
   └── Code patched! 

3. Re-protect:
   set_memory_ro(patch_addr, 1);
   └── R/W = 0 (read-only) 

4. Flush instruction cache:
   sync_core();
   └── CPU sees new code 

Minimal exposure window! 
Patch applied securely! 
```

---

# 5. Huge Page Splitting

## When Fine Control Needed

**The problem:**

```
2 MB huge page:
├── Virtual: 0x00200000 - 0x003FFFFF
├── Physical: 0x10000000 - 0x101FFFFF
└── Single page table entry (in PD)

Want to protect just first 4 KB:
└── Address: 0x00200000 - 0x00200FFF

Problem:
├── Huge page is atomic! 
├── Can't change part of it! 
└── All-or-nothing! 
```

**The solution - Split:**

```
split_large_page():

1. Allocate new PT (page table):
   pt = alloc_page();
   └── 4 KB for 512 entries 

2. Fill PT with 512 × 4 KB entries:
   for (i = 0; i < 512; i++):
       pt[i] = (0x10000000 + i×4KB) | attrs
   
   PT[0] = 0x10000000 | attrs
   PT[1] = 0x10001000 | attrs
   ...
   PT[511] = 0x101FF000 | attrs

3. Update PD entry:
   Old: 0x10000083 (huge page, PS=1)
   New: PT_phys | 0x003 (points to PT, PS=0) 

4. Flush TLB for entire range:
   for (addr = start; addr < end; addr += 4KB)
       invlpg(addr);

Huge page now split! 
```

**After split:**

```
Can now change individual 4 KB pages:

PT[0]: Change to read-only 
└── First 4 KB protecte

PT[1-511]: Leave as-is 
└── Rest unchanged

Precise control! 
```

**Cost of splitting:**

```
Memory overhead:
├── New PT: 4 KB 
└── Per huge page split

TLB overhead:
├── Was: 1 TLB entry (2 MB)
├── Now: Up to 512 entries (4 KB each) 
└── TLB pressure increased!

Performance:
├── Slight slowdown from TLB misses 
└── Trade-off: Flexibility vs Performance 
```

---

# 6. Physical Reality

## What Happens in Hardware

**Page table modification:**

```
Page table entry in RAM:
Address: 0x03700000 (physical)
Value: 0x00000000E000001B (UC)

Atomic change:
LOCK CMPXCHG [0x03700000], new_value

CPU:
├── Lock memory bus 
├── Read-Modify-Write 
└── New value: 0x00000000E000009B (WC) 

Real DRAM cells changed! 
```

**TLB invalidation:**

```
INVLPG instruction:

CPU executes:
1. Search TLB (CAM lookup):
   ├── Compare VPN against all entries
   └── Parallel hardware! 

2. Find matching entry

3. Clear valid bit:
   ├── Flip-flop: 1 → 0
   └── Single transistor state change 

Time: ~100 cycles
Physical: One bit flip in silicon! 
```

**Cache flush:**

```
WBINVD instruction:

CPU hardware:
1. Scan ALL cache lines:
   ├── L1 (32 KB)
   ├── L2 (256 KB)
   └── L3 (8 MB)
   
2. For each dirty line:
   ├── Write to memory controller 
   └── Memory controller → RAM 
   
3. Invalidate all:
   └── Clear valid bits 

Time: Microseconds! 
Expensive operation! 
Physical: Millions of bit flips! 
```

**Multi-CPU IPI:**

```
Physical path:

CPU 0 writes to Local APIC:
└── Electrical signal in CPU die 

Local APIC → APIC bus:
└── Electrical signal on motherboard 

APIC bus → I/O APIC:
└── Copper traces, nanoseconds 

I/O APIC → CPU 1 Local APIC:
└── Routing via APIC bus 

CPU 1 receives interrupt:
└── Hardware interrupt line 

All physical! Electricity! 
```

---

# 7. Connections to Other Topics

## How Attribute Changes Fit In

**Connection to Page Tables:**

```
TOPIC 8: 64-bit page table entry format
└── Bits: R/W, U/S, NX, PAT, PCD, PWT

TOPIC 11: Modifies those bits! 
└── Changes R/W (permission)
    Changes NX (execute)
    Changes PAT/PCD/PWT (caching)

pageattr.c manipulates TOPIC 8 structures! 
```

**Connection to TLB:**

```
TOPIC 9: TLB caches translations + attributes
└── Entries include R/W, NX, cache type

TOPIC 11: Changes attributes
└── MUST flush TLB! 
    Uses INVLPG (TOPIC 9)
    Uses TLB shootdown (TOPIC 9)

Can't change without TLB flush! 
```

**Connection to Page Faults:**

```
TOPIC 7: Page fault handling
└── Detects permission violations

TOPIC 11: Creates those protections! 
└── Makes pages read-only
    Makes pages NX
    Violations → Page fault! 

Fault handler enforces attribute restrictions! 
```

**Connection to Memory Init:**

```
TOPIC 10: Sets initial attributes
└── Kernel text, data, etc.

TOPIC 11: Refines after boot 
└── Hardens kernel code
    Adjusts as needed

Runtime refinement of boot settings! 
```

**Connection to ioremap:**

```
TOPIC 12 (coming): Maps device memory

TOPIC 11: Sets device cache attributes! 
└── Makes MMIO uncacheable
    Makes frame buffers WC

ioremap + pageattr work together! 
```

---

# Summary

## What You've Mastered

**You now understand page attribute changes!**

```
- What they are: Runtime property modifications
- Why needed: Security, performance, devices
- Permission changes: R/W, U/S, NX bits
- W^X enforcement: Writable XOR executable
- Cache attributes: WB, WT, UC, WC
- PAT mechanism: 8 cache type combinations
- Complete flow: Graphics WC example
- Huge page splitting: Fine-grained control
- Kernel hardening: Protecting code
- Physical reality: RAM, TLB, caches, IPIs
```

---

## Key Takeaways

**Dynamic needs require dynamic control:**

```
Can't predict all needs at map time:
├── Code loading: Writable → Executable
├── Graphics: UC → WC (25× faster!)
├── Security: Writable → Read-only
└── Must change after mapping! 
```

**W^X security:**

```
Critical security policy:
└── Page = Writable XOR Executable

Never both! 
├── Prevents code injection
├── Enforced automatically
└── Modern security essential! 
```

**Cache attributes matter:**

```
Wrong caching = Wrong behavior:

Device memory with WB:
└── Cached stale data! 

Graphics with UC:
└── 25× slower! 

Correct attributes essential! 
```

**TLB and cache flushing:**

```
After attribute change:
1. Flush TLB (stale translations) 
2. Flush caches if cache type changed 
3. Multi-CPU coordination (shootdown) 

Without flushing:
└── Stale state! Incorrect behavior! 
```

---

## The Big Picture

**Attribute lifecycle:**

```
Page mapping:
├── Create: Initial attributes 
├── Use: Normal operatio
├── Change: Runtime modification 
└── Use: With new attributes 

Example: Module loading
1. Map: Writable, NX
2. Load: Copy code
3. Change: Read-only, Executable 
4. Run: Secure execution 
```

**Performance impact:**

```
Cache attribute changes:

Graphics frame buffer:
├── UC: 0.4 seconds per frame 
└── WC: 0.016 seconds per frame 
    25× speedup! 

Device registers:
├── Must be UC (correctness)
└── Performance cost acceptable 
```

---

## You're Ready!

With this knowledge, you can:
- Implement W^X security
- Optimize graphics performance
- Protect kernel code
- Handle device memory correctly
- Debug attribute-related issues

> **Page attribute changes enable both security and performance - you now know how x86 makes pages adaptable!**

---

**Next up: I/O Memory Mapping**

Ready to learn how to map device memory for driver development? 
