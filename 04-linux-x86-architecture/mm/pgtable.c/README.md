# Organizing Memory Into 4-Level Hierarchies - x86 Translation

> Ever wondered how virtual addresses become physical addresses? How the CPU finds data in RAM? Why page tables don't consume all your memory?

**You're about to find out!**

---

## What's This About?

This is **arch/x86/mm/pgtable.c** - how x86 organizes the translation from virtual to physical addresses!

Here's where it fits:

```
arch/x86/
└── mm/                    ← x86 memory management
    ├── fault.c            ← x86 page fault entry
    ├── pgtable.c          ← Page table operations (THIS!)
    ├── tlb.c              ← TLB management
    ├── init.c             ← Memory initialization
    ├── pageattr.c         ← Page attributes
    └── ioremap.c          ← I/O memory mapping
```

**This file handles:**
- 4-level hierarchy structure (PML4→PDPT→PD→PT)
- 64-bit entry format (every bit matters!)
- Creating and destroying page tables
- Translation from virtual to physical addresses
- Huge page support (2 MB, 1 GB)

---

# The Fundamental Problem

**Physical Reality:**

```
Your Program Sees:
├── Virtual address space: 256 TB (48-bit addresses)
└── Every process has its own view

The Hardware Has:
├── RAM: Maybe 16 GB
└── Physical addresses need translation

Question: How does 0x00401000 (virtual) become 0x12345000 (physical)?
```

**The x86 MMU needs a lookup table to translate addresses!**

But here's the challenge:

```
Naive Approach (Flat Table):

Virtual space: 256 TB
Page size: 4 KB
Number of pages: 67 million pages
Entry size: 8 bytes

Table size per process: 32 PB (petabytes)! 
Your 16 GB RAM can't hold even ONE process table!
Completely impossible!
```

**The kernel needs EFFICIENT translation without consuming all RAM!**

---

# Without Hierarchical Page Tables

**Imagine only having a giant flat table:**

```
Firefox process needs translation table:

Flat table approach:
├── Entry for EVERY possible virtual page
├── 67 million entries × 8 bytes
├── = 512 MB per process 
└── Firefox uses only ~1000 pages (4 MB)!

Result: 99.8% wasted space! 

System with 100 processes:
└── 100 × 512 MB = 51 GB just for page tables! 
    More than your RAM!
```

**This is why flat tables don't work!**

---

# The 4-Level Solution

**x86 uses a hierarchical tree structure:**

```
Key Insight: Only allocate tables for USED memory!

Level 4: PML4 (1 table)
    ↓
Level 3: PDPT (allocate only if needed)
    ↓
Level 2: PD (allocate only if needed)
    ↓
Level 1: PT (allocate only if needed)
    ↓
Physical Page (4 KB)
```

**Memory savings:**

```
Firefox using 4 MB of memory (1000 pages):

With hierarchical tables:
├── PML4: 4 KB 
├── PDPT: 4 KB
├── PD: 4 KB 
├── PT: ~16 KB  (only 4 tables needed)
└── Total: ~28 KB of page tables

vs Flat table: 512 MB 

Savings: 18,000× smaller!
```

**Each level narrows down the address until we find the physical page!**

---

# 1. The 64-Bit Entry Format

## Every Entry is 8 Bytes of Packed Information

**The entry structure:**

```
Bit 63                                    Bit 0
  ↓                                          ↓
[NX][--][Physical Address][--][G][--][D][A][--][U/S][R/W][P]
  ↑      ↑                      ↑      ↑  ↑       ↑    ↑   ↑
 63    51-12                    8      6  5       2    1   0
```

### The Critical Bits

**Bit 0: Present (P)**

```
0 = Page not mapped → Page fault!
1 = Page exists → Translation continues

Example:
Entry = 0x0000000012345007
Bit 0 = 1 → Present 

Entry = 0x0000000012345006
Bit 0 = 0 → Not present, will fault! 
```

**Bit 1: Read/Write (R/W)**

```
0 = Read-only (code pages, COW pages)
1 = Writable (data pages, heap, stack)

Example uses:
Code section: R/W = 0 (prevent modification)
COW page: R/W = 0 (trigger copy on write)
Heap: R/W = 1 (allow writes)
```

**Bit 2: User/Supervisor (U/S)**

```
0 = Kernel only (Ring 0 access)
1 = User accessible (Ring 3 allowed)

Security check:
User program tries: 0xFFFFFFFF81000000 (kernel)
MMU sees: U/S = 0 (kernel only)
Current mode: Ring 3 (user) 
Result: PAGE FAULT! Protection violation!
```

**Bit 5: Accessed (A)**

```
CPU sets automatically on ANY access:
0 → CPU changes to 1 on first access

Kernel uses for LRU:
├── Clear all A bits
├── Wait some time
├── Check which pages have A=1
└── Pages with A=0 haven't been used → swap candidates!
```

**Bit 6: Dirty (D)**

```
CPU sets automatically on WRITE:
0 → CPU changes to 1 on first write

Kernel uses for swap/cache:
├── D=0: Clean page (discard safely)
└── D=1: Dirty page (must write to disk first!)
```

**Bit 8: Global (G)**

```
0 = Flush on CR3 change (user pages)
1 = Keep in TLB on CR3 change (kernel pages)

Performance optimization:
Kernel pages: G=1
├── Same kernel in all processes
├── Don't flush kernel TLB on context switch
└── Kernel access stays fast! 
```

**Bit 63: No Execute (NX)**

```
0 = Executable (code pages)
1 = No execute (data pages)

Security feature:
Stack: NX=1 → Can't execute shellcode! 
Heap: NX=1 → Can't execute injected code! 
Code: NX=0 → Can execute instructions 
```

**Bits 51-12: Physical Address**

```
40 bits for physical address:

Entry = 0x0000000012345007
Bits 51-12 = 0x12345
Physical address = 0x12345 << 12 = 0x12345000

Why shift left 12?
Pages are 4 KB aligned (2^12)
Lower 12 bits always zero
Don't need to store them! 
```

---

# 2. The 4-Level Walk

## How MMU Translates Addresses

**Virtual address breakdown:**

```
48-bit address: 0x00007F1234567890

Split into 5 parts:
├── Bits 47-39 (9 bits): PML4 index = 254
├── Bits 38-30 (9 bits): PDPT index = 72
├── Bits 29-21 (9 bits): PD index = 162
├── Bits 20-12 (9 bits): PT index = 359
└── Bits 11-0 (12 bits): Page offset = 2192
```

**Complete translation flow:**

### Step 1: Start with CR3

```
CPU reads CR3 register:
CR3 = 0x02000000 (PML4 physical address)

This is the root of the tree! 
```

### Step 2: Access PML4

```
PML4 index: 254

Calculate entry address:
├── PML4 base: 0x02000000
├── Index: 254
├── Entry address = 0x02000000 + (254 × 8)
└── Entry address = 0x020007F0

MMU reads from RAM:
Read 8 bytes from 0x020007F0
Value: 0x0000000003000007

Parse entry:
├── P bit = 1 → Present 
└── Physical address = 0x03000000 (PDPT location)
```

### Step 3: Access PDPT

```
PDPT index: 72

Calculate entry address:
├── PDPT base: 0x03000000 (from PML4 entry)
├── Index: 72
└── Entry address = 0x03000000 + (72 × 8) = 0x03000240

MMU reads from RAM:
Value: 0x0000000004000007

Parse entry:
├── P bit = 1 → Present 
└── Physical address = 0x04000000 (PD location)
```

### Step 4: Access PD

```
PD index: 162

Calculate entry address:
├── PD base: 0x04000000
├── Index: 162
└── Entry address = 0x04000000 + (162 × 8) = 0x04000510

MMU reads from RAM:
Value: 0x0000000005000007

Parse entry:
├── P bit = 1 → Present 
├── PS bit = 0 → Not 2MB page, continue 
└── Physical address = 0x05000000 (PT location)
```

### Step 5: Access PT

```
PT index: 359

Calculate entry address:
├── PT base: 0x05000000
├── Index: 359
└── Entry address = 0x05000000 + (359 × 8) = 0x05000B38

MMU reads from RAM:
Value: 0x0000000012345007

Parse entry:
├── P bit = 1 → Present 
├── R/W bit = 1 → Writable 
├── U/S bit = 1 → User accessible 
├── NX bit = 0 → Executable 
└── Physical address = 0x12345000 (PAGE location!)
```

### Step 6: Add Offset

```
Page base: 0x12345000
Offset: 0x890 (from virtual address bits 11-0)

Final physical address:
└── 0x12345000 + 0x890 = 0x12345890 

TRANSLATION COMPLETE!
Virtual 0x00007F1234567890 → Physical 0x12345890 
```

**Total memory accesses: 5 reads!**

```
This is why TLB exists:
├── Without TLB: 5 memory reads per access 
└── With TLB: 0 memory reads (cached!) 
```

---

# 3. Creating Page Tables

## Building the Hierarchy On Demand

**Scenario: Program calls malloc(4096)**

```
Program: malloc(4096)

Kernel (mm/mmap.c):
├── Create VMA (virtual memory area) 
├── Don't allocate physical page yet! (demand paging)
└── Don't create page tables yet!

Program: Accesses the memory

Result: Page fault! (no page table entry exists)
```

**Building the chain:**

### Step 1: Allocate Physical Page

```
Kernel allocates:
page = alloc_page(GFP_KERNEL)
Physical address: 0x12345000 
```

### Step 2: Check if PT Exists

```
Virtual address: 0x00601000
Indexes: PML4=0, PDPT=0, PD=3, PT=1

Check PD[3]:
└── Read entry
    P bit = 0  (not present!)

PT doesn't exist! Need to create it!
```

### Step 3: Allocate PT

```
arch/x86/mm/pgtable.c:

pt_page = alloc_page(GFP_KERNEL)
Physical address: 0x05000000

Zero the table:
memset(0x05000000, 0, 4096)
All 512 entries = 0 (not present)

PT created! 
```

### Step 4: Link PT into PD

```
Update PD[3] entry:
├── Physical address: 0x05000000
├── Present: 1
├── Writable: 1
└── User: 1

Entry value: 0x0000000005000007

Write to PD:
*pd_entry = 0x0000000005000007 

PT now linked into hierarchy! 
```

### Step 5: Create PT Entry

```
Update PT[1] entry:
├── Physical address: 0x12345000 (the actual page)
├── Present: 1
├── Writable: 1
├── User: 1
└── NX: 1 (heap, no execute!)

Entry value: 0x8000000012345007

Write to PT:
*pt_entry = 0x8000000012345007 

Page now mapped! 
```

### Step 6: Flush TLB

```
invlpg(0x00601000)
└── Invalidate old TLB entry
    Next access will use new translation 
```

---

# 4. Destroying Page Tables

## Cleanup When Process Exits

**Scenario: Process terminates**

```
Process calls: exit(0)

Kernel must:
├── Free all physical pages
└── Free all page table pages
```

**Walking the hierarchy:**

```
For each PML4 entry (if present):
    For each PDPT entry (if present):
        For each PD entry (if present):
            For each PT entry (if present):
                └── Free physical page 
            └── Free PT page 
        └── Free PD page
    └── Free PDPT page 
└── Free PML4 page 
```

**Memory reclaimed:**

```
Process was using:
├── 1000 pages × 4 KB = 4 MB (data)
├── Page tables: ~28 KB
└── Total: ~4.03 MB

After cleanup:
└── All 4.03 MB returned to system! 
```

---

# 5. Huge Pages (2 MB and 1 GB)

## Skipping Levels for Performance

**Normal 4 KB pages:**

```
Translation path:
PML4 → PDPT → PD → PT → 4 KB page
(4 levels, 4 memory reads)
```

**2 MB huge pages:**

```
Translation path:
PML4 → PDPT → PD → 2 MB page (skip PT!)
(3 levels, 3 memory reads)

PD entry with PS=1:
Bit 7 (Page Size) = 1
Maps entire 2 MB directly! 
```

**Benefits:**

```
TLB efficiency:
├── 512 × 4 KB pages = 2 MB (uses 512 TLB entries)
└── 1 × 2 MB page = 2 MB (uses 1 TLB entry!) 

Less page table memory:
├── Don't need PT level
└── Saves 4 KB per 2 MB region

Faster translation:
└── 3 memory reads instead of 4 
```

**1 GB huge pages:**

```
Translation path:
PML4 → PDPT → 1 GB page (skip PD and PT!)
(2 levels, 2 memory reads)

PDPT entry with PS=1:
Maps entire 1 GB directly! 

Used for:
├── Supercomputers
├── Large databases
└── In-memory computing
```

---

# 6. 5-Level Paging (Future CPUs)

## Extending the Address Space

**Current 4-level:**

```
Address space: 256 TB (48 bits)

Enough for now, but:
├── Supercomputers hitting limits
├── Future applications need more
└── In-memory databases growing
```

**New 5-level:**

```
PML5 → PML4 → PDPT → PD → PT → Page

Address space: 128 PB (57 bits) 
512× larger!

Available on:
├── Intel Ice Lake+
└── Future AMD CPUs

Enable with:
CR4.LA57 = 1
```

---

# 7. Physical Reality

## What Actually Exists in Hardware

**Page tables in RAM:**

```
Physical DRAM cells:

Address 0x02000000:
[0x0000000003000007]  ← PML4[0]
[0x0000000000000000]  ← PML4[1] (not present)
[0x0000000000000000]  ← PML4[2] (not present)
...
[0x0000000003001007]  ← PML4[254]

Real hardware:
├── Each cell = capacitor (1 bit)
├── 64 cells = 1 entry (8 bytes)
└── 512 entries = 1 table (4 KB page)
```

**MMU hardware walker:**

```
MMU circuit in CPU silicon:

Components:
├── Address splitter (combinational logic)
│   └── Extracts indexes from virtual address
│
├── Table walker (state machine)
│   └── Reads entries from RAM sequentially
│
├── Permission checker (comparators)
│   └── Checks R/W, U/S, NX bits in parallel
│
└── TLB (cache)
    └── Stores recent translations
```

**CR3 register:**

```
Physical implementation:
├── 64 D flip-flops (one per bit)
├── Special register file (not general purpose)
└── Hardware write-only (software can only read)

On CR3 write (MOV CR3, RAX):
1. Update CR3 register 
2. Flush most TLB entries 
3. Invalidate internal caches 

All automatic! Hardware magic! 
```

---

# 8. Connections to Other Topics

## How Page Tables Fit In

**Connection to Page Faults:**

```
fault.c (arch/x86/mm/fault.c):
└── "Why did fault happen?"
    Check page table entry:
    ├── P=0? Not present (demand paging)
    ├── R/W=0? Read-only (COW)
    └── U/S=0? Kernel-only (protection)

Page fault handler needs to understand
the 64-bit entry format! 
```

**Connection to TLB:**

```
Page tables are SLOW:
├── 4-5 memory reads per translation 

TLB (Translation Lookaside Buffer):
├── Caches recent translations
├── Virtual → Physical mapping
└── 0 memory reads on hit! 

TLB is why you don't notice
the 4-level walk! 
```

**Connection to Context Switch:**

```
Context switch changes CR3:
├── Old process: CR3 = 0x02000000
└── New process: CR3 = 0x03000000

Effect:
├── Entire memory view changes!
├── Different PML4 → Different translations
└── Process isolation! 
```

**Connection to mm/memory.c:**

```
Generic code (mm/memory.c):
└── "Map virtual 0x00601000 to physical 0x12345000"
    Calls ↓

x86 code (arch/x86/mm/pgtable.c):
├── Create entry: 0x8000000012345007
├── Format: 64-bit x86 format
└── Write to PT

Collaboration between generic
and architecture-specific! 
```

---

# Summary

## What You've Mastered

**You now understand x86 page tables!**

```
- What they are: 4-level hierarchical translation structure
- Why hierarchical: Memory savings (18,000× smaller!)
- 64-bit entry: Every bit has meaning and purpose
- How MMU walks: 4-level lookup from virtual to physical
- Creating tables: Bottom-up on-demand allocation
- Destroying tables: Top-down cleanup and reclamation
- Huge pages: 2 MB and 1 GB for performance
- 5-level paging: Future-proofing for 128 PB
- Physical reality: DRAM cells, MMU circuits, CR3 register
- Connections: Fault handling, TLB, context switch, mm/
```

---

## Key Takeaways

**The Hierarchical Design:**

```
Problem: Flat table = 32 PB per process 
Solution: 4-level hierarchy = ~40 KB per process 

Only allocate tables for USED memory!
Sparse address space handled efficiently!
```

**The 64-Bit Entry:**

```
Not just an address!

Each entry packs:
├── Physical address (40 bits)
├── Present flag
├── Read/write permission
├── User/kernel separation
├── Accessed tracking
├── Dirty tracking
├── Global hint
├── No-execute protection
└── More!
```

**The Translation Process:**

```
5 memory reads total:
1. PML4 entry
2. PDPT entry
3. PD entry
4. PT entry
5. Actual data

This is why TLB matters:
├── Without: 5× slower 
└── With: Instant! 
```

---

## The Big Picture

**When a program accesses memory:**

```
Program: *ptr = 42;  (virtual address 0x00601000)
    ↓
MMU extracts indexes:
├── PML4 index: 0
├── PDPT index: 0
├── PD index: 3
├── PT index: 1
└── Offset: 0
    ↓
MMU walks 4 levels:
PML4[0] → PDPT[0] → PD[3] → PT[1]
    ↓
Finds physical page: 0x12345000
    ↓
Adds offset: 0x12345000 + 0 = 0x12345000
    ↓
Writes to RAM: Physical address 0x12345000 = 42 
    ↓
Program: "Write succeeded!"
    (Never knew about translation!)
```

**All in ~100 nanoseconds!** 

---

## You're Ready!

With this knowledge, you can:
- Understand virtual memory systems
- Debug memory mapping issues
- Optimize memory-intensive applications
- Build memory management systems
- Use huge pages effectively

> **The 4-level page table is the foundation of virtual memory - you now know how x86 makes it work!**

---

**Next up: TLB Management**

Ready to learn how x86 caches translations to make this fast? 
