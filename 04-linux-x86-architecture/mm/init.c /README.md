# Bootstrapping Virtual Memory From Nothing - Boot Initialization

> Ever wondered how the kernel sets up memory when it first starts? How paging gets enabled from scratch? Where the first page tables come from?

**You're about to find out!**

---

## What's This About?

This is **arch/x86/mm/init.c** - how x86 bootstraps the entire memory system at boot!

Here's where it fits:

```
arch/x86/
└── mm/                    ← x86 memory management
    ├── fault.c            ← x86 page fault entry
    ├── pgtable.c          ← Page table operations
    ├── tlb.c              ← TLB management
    ├── init.c             ← Memory initialization (THIS!)
    ├── pageattr.c         ← Page attributes
    └── ioremap.c          ← I/O memory mapping
```

**This file handles:**
- Detecting how much RAM exists (E820 memory map)
- Creating initial page tables (before anything else works)
- Enabling paging and 64-bit mode (critical transition)
- Setting up memory zones (DMA, Normal)
- Bootstrapping the buddy allocator (before it can allocate!)

---

# The Fundamental Problem

**Physical Reality:**

```
When the computer powers on:

CPU state:
├── Real mode (16-bit!) 
├── No paging enabled 
├── No virtual memory 
└── Physical addresses only

Memory state:
├── RAM exists but kernel doesn't know how much
├── No page tables yet
├── No memory allocator yet
└── Nothing initialized!

Kernel wants to run:
├── Code at: 0xFFFFFFFF81000000 (high virtual address)
├── Needs: Virtual memory enabled
├── Needs: Page tables created
└── Needs: Memory allocator working

But to create these things:
├── Need memory to store page tables 
├── Need allocator to get memory 
├── Need page tables to enable allocator 
└── Chicken and egg! 
```

**Can't use virtual memory until it's set up, but need virtual memory to set it up!**

---

# Without Memory Initialization

**Imagine trying to run the kernel without init.c:**

```
Bootloader loads kernel at physical 0x01000000:
└── Kernel code sitting in RAM 

Bootloader jumps to kernel entry point:
JMP 0x01000000
└── Start executing 

First kernel instruction:
CALL 0xFFFFFFFF81234567  // High virtual address

CPU tries to execute:
├── Virtual address: 0xFFFFFFFF81234567
├── Paging enabled? NO 
├── CPU treats as physical address
└── Physical 0xFFFFFFFF81234567 doesn't exist! 

Result: Triple fault! CPU resets! 

Even if we enable paging:
├── CR3 = ??? (no page tables exist!) 
├── Walk page tables → Garbage data 
└── Wrong translations! Crash! 

Kernel needs memory allocator:
page = alloc_page();
├── Buddy allocator initialized? NO 
├── Free lists created? NO 
├── Zone structures exist? NO 
└── Can't allocate! 

Everything depends on initialization! 
```

**Nothing works without bootstrap!**

---

# The Bootstrap Solution

**Three-stage initialization to break the deadlock:**

```
Stage 1: Minimal (compiled-in page tables)
├── Static page tables compiled into kernel image 
├── Just enough to enable paging
├── Identity map low memory (1:1 mapping)
├── High map kernel (0xFFFF... → physical)
└── Get into 64-bit mode 

Stage 2: Memory detection (find the RAM)
├── Read E820 map from BIOS 
├── Know exactly how much RAM exists
├── Know which regions are usable
└── Build proper page tables for all RAM 

Stage 3: Full setup (transfer to real allocators)
├── Create memory zones 
├── Initialize page structures
├── Transfer to buddy allocator
└── Normal operation ready! 

Each stage enables the next! 
```

---

# 1. Memory Detection

## The E820 Memory Map

**BIOS tells us about RAM:**

```
Before kernel runs, BIOS/UEFI:
├── Probes memory controllers
├── Tests RAM chips
├── Detects total RAM amount
└── Creates memory map 

E820 function (BIOS INT 0x15, AX=0xE820):
└── Returns list of memory regions

Example E820 map for 8 GB system:

Region 0:
├── Start: 0x00000000
├── Size: 640 KB
└── Type: 1 (Usable RAM) 

Region 1:
├── Start: 0x000A0000
├── Size: 384 KB
└── Type: 2 (Reserved - VGA memory) 

Region 2:
├── Start: 0x00100000
├── Size: 3.5 GB
└── Type: 1 (Usable RAM) 

Region 3:
├── Start: 0xFEC00000
├── Size: 4 KB
└── Type: 2 (Reserved - I/O APIC!) 

Region 4:
├── Start: 0xFEE00000
├── Size: 4 KB
└── Type: 2 (Reserved - Local APIC!) 

Region 5:
├── Start: 0x100000000 (above 4 GB)
├── Size: 4.5 GB
└── Type: 1 (Usable RAM) 

Total usable: ~8 GB 
```

## Why Reserved Regions Matter

**Some physical addresses aren't RAM:**

```
Physical 0xFEE00000:
├── Not RAM! 
├── Local APIC registers (from TOPIC 6!)
└── Writing here = controlling interrupts

If kernel tried to use as RAM:
├── Corruption of interrupt controller 
├── System crashes 
└── E820 prevents this! 

Physical 0xFF000000:
├── Not RAM! 
├── Flash ROM (BIOS code)
└── Read-only!

If kernel tried to write:
├── Write ignored or causes fault 
└── E820 protects us! 
```

**E820 map is ground truth about system memory!**

## Parsing the Map

```
Kernel reads E820 from bootloader:
├── Bootloader stored it at fixed location
└── Kernel parses each entry

For each region:
    if (type == E820_RAM):
        - Usable! Add to total
        - Remember this region
    else if (type == E820_RESERVED):
        - Don't touch!
        - Never allocate from here

Results:
├── Total RAM: 8 GB
├── Usable regions list: [0-640KB, 1MB-3.5GB, 4GB-8.5GB]
└── Reserved regions list: [VGA, APIC, BIOS, ...]

Now we know what memory exists! 
```

---

# 2. Initial Page Tables

## The Deadlock Breaker

**Compiled into kernel image:**

```
Problem: Need paging to run kernel
        Need page tables for paging
        Need memory allocator for page tables
        Need paging for allocator
        ↻ Circular dependency!

Solution: Build page tables AT COMPILE TIME! 

Kernel source (arch/x86/kernel/head_64.S):

.section .init.data
.align 4096

init_pml4:
    .quad init_pdpt + 0x003    # Entry 0: Low mapping
    .fill 510, 8, 0            # Entries 1-510: Empty
    .quad init_pdpt + 0x003    # Entry 511: High mapping

init_pdpt:
    .quad init_pd + 0x003      # Entry 0
    .fill 511, 8, 0

init_pd:
    .quad 0x0000000 + 0x083    # 0-2MB (huge page)
    .quad 0x0200000 + 0x083    # 2-4MB
    .quad 0x0400000 + 0x083    # 4-6MB
    ...
    (512 entries total, mapping 1 GB)

These are just DATA in the kernel binary! 
When kernel loaded, page tables already in RAM! 
```

## Two Critical Mappings

**Identity mapping:**

```
Virtual 0x00000000 → Physical 0x00000000
Virtual 0x00200000 → Physical 0x00200000
Virtual 0x00400000 → Physical 0x00400000
...

Why needed?
├── Early boot code runs at physical addresses
├── When paging enables, next instruction fetches
├── If address changes, CPU crashes! 
└── Identity mapping keeps addresses same 

Example:
Before paging:
├── Executing at physical 0x00001000
└── Next instruction at 0x00001004

Enable paging:
├── Now using virtual addresses
├── Instruction fetch from virtual 0x00001004
├── Translates to physical 0x00001004 
└── Same location! Continue executing! 
```

**High mapping:**

```
Virtual 0xFFFFFFFF80000000 → Physical 0x00000000
Virtual 0xFFFFFFFF80200000 → Physical 0x00200000
Virtual 0xFFFFFFFF80400000 → Physical 0x00400000
...

Why needed?
├── Kernel compiled to run at high addresses
├── All function calls expect high addresses
├── All data references expect high addresses
└── Must map them to actual physical RAM! 

Example:
Kernel code calls function:
CALL 0xFFFFFFFF81234567

Translation:
├── Virtual: 0xFFFFFFFF81234567
├── Via high mapping → Physical: 0x01234567 
└── Function code is actually there! 

Both mappings point to SAME physical RAM:
├── Low: 0x00000000 → Physical 0x00000000
├── High: 0xFFFFFFFF80000000 → Physical 0x00000000
└── Two ways to access same memory! 
```

---

# 3. The Critical Transition

## Enabling Paging and 64-bit Mode

**The moment everything changes:**

```
Early boot code (still physical addressing):

Step 1: Load page table base
    MOV RAX, init_pml4      # Physical address of PML4
    MOV CR3, RAX            # Tell CPU where tables are 

Step 2: Enable PAE (Physical Address Extension)
    MOV RAX, CR4
    OR RAX, 0x20            # Set PAE bit
    MOV CR4, RAX            

Step 3: Enable Long Mode (64-bit)
    MOV ECX, 0xC0000080     # EFER MSR
    RDMSR
    OR EAX, 0x100           # Set LME (Long Mode Enable)
    WRMSR                   

Step 4: Enable Paging
    MOV RAX, CR0
    OR RAX, 0x80000000      # Set PG (Paging) bit
    MOV CR0, RAX            

      PAGING NOW ENABLED! 

Step 5: Jump to high kernel
    JMP 0xFFFFFFFF81000000  

    Now executing at high virtual addresses! 
    64-bit mode active! 
    Virtual memory working! 
```

**Before and after:**

```
Before (T = 10ms):
├── Mode: Protected mode (32-bit)
├── Paging: Disabled 
├── Addresses: Physical only
├── Running at: Physical 0x00001000
└── CR3: Undefined

Execute: MOV CR0, RAX (enable paging)

After (T = 10.001ms)
├── Mode: Long mode (64-bit) 
├── Paging: Enabled 
├── Addresses: Virtual
├── Running at: Virtual 0xFFFFFFFF81001000 
└── CR3: Points to init_pml4 

Everything changed in one instruction! 
```

---

# 4. Building Real Page Tables

## Beyond the Bootstrap

**Initial tables are minimal:**

```
Bootstrap page tables:
├── Coverage: Only first 1 GB mapped 
├── Type: Only 2 MB huge pages
├── Permissions: Basic (no fine control)
└── Good enough to boot 

But system has 8 GB RAM!
├── Most of RAM unmapped! 
├── Can't access it yet! 
└── Need proper mappings! 
```

**Creating the direct map:**

```
Goal: Map ALL physical RAM to kernel virtual space

Direct mapping scheme:
Physical 0x00000000 → Virtual 0xFFFF888000000000
Physical 0x00100000 → Virtual 0xFFFF888000100000
Physical 0x01000000 → Virtual 0xFFFF888001000000
...
Physical 0x21FFFFFFF → Virtual 0xFFFF88821FFFFFFF

Fixed offset: 0xFFFF888000000000
Simple arithmetic: virtual = physical + offset 

Why?
├── Kernel needs to access ANY physical page
├── Without direct map: Must create temporary mappings 
├── With direct map: Just add offset! 
└── Convenience and speed! 
```

**Building process:**

```
init_mem_mapping():

For each 2 MB of RAM (0 to 8 GB):
    physical_addr = current_position
    virtual_addr = 0xFFFF888000000000 + physical_addr
    
    Create page table entry:
    ├── Virtual: virtual_addr
    ├── Physical: physical_addr
    ├── Present: 1
    ├── Writable: 1
    ├── NX: 1 (no execute for data)
    └── Huge page: 1 (2 MB)
    
    current_position += 2 MB
    
All 8 GB now mapped! 

Any physical address instantly accessible:
├── Physical 0x50000000
├── Virtual = 0xFFFF888000000000 + 0x50000000
└── Virtual 0xFFFF888050000000 
```

**Removing identity map:**

```
After jump to high kernel:
├── Running at: 0xFFFFFFFF8...
├── Don't need identity map anymore
└── Remove it for security! 

Update PML4:
PML4[0] = 0  (was identity map, now removed)
PML4[511] = still points to high mapping 

Identity mapping gone! 
Low virtual addresses now invalid:
├── Access 0x00000000 → Page fault! 
└── Security: Can't access low memory by accident 
```

---

# 5. Memory Zones

## Hardware Constraints Drive Design

**Not all RAM is equal:**

```
Ancient ISA devices (very old):
├── DMA limited to 24-bit addresses
├── Can only access: 0x00000000 - 0x00FFFFFF (16 MB)
└── Need: ZONE_DMA

Newer devices with 32-bit DMA:
├── DMA limited to 32-bit addresses
├── Can only access: 0x00000000 - 0xFFFFFFFF (4 GB)
└── Need: ZONE_DMA32

Modern devices and CPU:
├── Can access all 64-bit addresses
└── Use: ZONE_NORMAL
```

**Creating zones:**

```
Based on E820 map (8 GB system):

ZONE_DMA:
├── Physical: 0x00000000 - 0x01000000 (16 MB)
├── For: Very old ISA devices
├── Pages: 4,096
└── Buddy free lists: (will be filled) 

ZONE_DMA32:
├── Physical: 0x01000000 - 0x100000000 (~4 GB)
├── For: 32-bit DMA devices
├── Pages: ~1,000,000
└── Buddy free lists: (will be filled) 

ZONE_NORMAL:
├── Physical: 0x100000000 - 0x21FFFFFFF (~4 GB above 4GB)
├── For: Normal allocations
├── Pages: ~1,000,000
└── Buddy free lists: (will be filled) 

Total: ~2,000,000 pages (8 GB / 4 KB) 
```

**Zone selection:**

```
Driver requests DMA memory:
dma_alloc_coherent(..., GFP_DMA);
└── Allocate from ZONE_DMA only 
    Device can access it! 

Normal kernel allocation:
kmalloc(..., GFP_KERNEL);
└── Try ZONE_NORMAL first 
    Fall back to ZONE_DMA32 if needed 
    Preserve scarce ZONE_DMA 
```

---

# 6. Bootstrap Allocator

## Before Buddy Can Work

**The chicken-and-egg of allocation:**

```
Buddy allocator needs:
├── Free page lists → Requires memory! 
├── Zone structures → Requires memory! 
├── Page structures (struct page[]) → Requires 128 MB! 
└── Can't allocate without allocator! 

Solution: Simple bootstrap allocator! 
```

**memblock allocator:**

```
Ultra-simple design:
├── Track: "Memory from X to Y is used"
├── Allocate: "Find next unused region"
└── No freeing! (one-way only)

Example usage:

memblock_alloc(4096):
    Current position: 0x03000000
    Mark 0x03000000-0x03001000 as used
    Return: 0x03000000 
    New position: 0x03001000

memblock_alloc(8192):
    Current position: 0x03001000
    Mark 0x03001000-0x03003000 as used
    Return: 0x03001000 
    New position: 0x03003000

Simple forward allocation! 
Good enough for boot! 
```

**Building buddy with memblock:**

```
Using memblock to create buddy:

Step 1: Allocate page structures
    struct page *pages;
    pages = memblock_alloc(128 MB);
    └── 2,000,000 pages × 64 bytes each 

Step 2: Allocate zone structures
    zones = memblock_alloc(sizeof(zones));
    └── Zone metadata 

Step 3: Allocate free list headers
    free_lists = memblock_alloc(sizeof(lists));
    └── Buddy list heads 

Now buddy has its data structures! 
```

---

# 7. Transfer to Buddy

## The Final Handoff

**Transitioning allocators:**

```
After buddy structures created:

Step 1: Find all free memblock memory
    For each memblock region:
        if (not reserved):
            This is free memory! 

Step 2: Give each free page to buddy
    For each free 4 KB page:
        __free_pages(page, 0);
        ├── Adds page to buddy free lists
        └── Now buddy can allocate it! 

Example:
    memblock region: [0x04000000 - 0x08000000] (64 MB)
    
    Not reserved! All free!
    
    Give to buddy:
    for (addr = 0x04000000; addr < 0x08000000; addr += 4096):
        page = page_for_addr(addr)
        __free_pages(page, 0) 
    
    Buddy now has 64 MB more! 

Step 3: Disable memblock
    memblock_active = false 
    
From now on:
├── All allocations through buddy 
├── memblock no longer used
└── Normal operation! 
```

**Before and after:**

```
Before transfer (T = 14ms):
├── memblock: Has all free memory
├── Buddy: Empty free lists 
└── Allocations via memblock

Execute: Transfer all free memory

After transfer (T = 15ms):
├── memblock: Disabled 
├── Buddy: Has ~7 GB free memory 
└── Allocations via buddy 

Memory management fully operational! 
```

---

# 8. Complete Boot Sequence

## Power-On to Ready

**The full timeline:**

```
T=0ms: Power button pressed
├── CPU powers on
├── RAM powers on
└── CPU starts at reset vector

T=0-10ms: BIOS/UEFI runs
├── Test hardware
├── Probe memory controllers
├── Detect: 8 GB RAM installed 
├── Create E820 map 
└── Store at: 0x00002D00

T=10ms: Bootloader (GRUB) runs
├── Load kernel from disk
├── Place at: Physical 0x01000000 
├── Load initrd
└── Jump to: 0x01000000

T=11ms: Kernel entry point
├── Mode: Protected (32-bit)
├── Paging: Disabled 
└── Running at: Physical addresses

T=11.1ms: Enable paging
├── Load CR3: init_pml4 
├── Enable PAE 
├── Enable Long Mode 
├── Enable Paging 
└── Jump to: 0xFFFFFFFF81000000

T=11.2ms: Now in high kernel
├── Mode: Long (64-bit) 
├── Paging: Enabled 
└── Virtual memory working 

T=11.3ms: Parse E820 map
├── Read from: 0x00002D00
├── Total RAM: 8 GB 
├── Usable regions: Known 
└── Reserved regions: Known 

T=12ms: Create real page tables
├── Map all 8 GB 
├── Direct map created 
├── Remove identity map 
└── Full page tables ready 

T=13ms: Initialize zones
├── ZONE_DMA: 16 MB 
├── ZONE_DMA32: ~4 GB 
├── ZONE_NORMAL: ~4 GB 
└── Zones ready 

T=14ms: Create page structures
├── Allocate: 128 MB via memblock
├── Initialize: 2,000,000 structs
└── Page tracking ready 

T=15ms: Transfer to buddy
├── Give all free pages to buddy 
├── Buddy has ~7 GB free 
├── Disable memblock 
└── Buddy allocator operational! 

T=20ms: Final initialization
├── Slab allocators 
├── Vmalloc 
├── Page cache 
└── Everything ready! 

T=50ms+: Normal operation
├── Can allocate memory 
├── Can create processes 
├── Can load drivers 
└── Kernel fully operational! 
```

---

# 9. Physical Reality

## What Actually Happens in Hardware

**Memory detection:**

```
Memory controller chip on motherboard:
├── Knows how many DIMM slots filled
├── Knows capacity of each DIMM
├── Knows total RAM: 8 GB

BIOS queries memory controller:
├── Read configuration registers
├── Get memory layout
└── Create E820 map 

Real hardware detection! 
```

**Page tables in RAM:**

```
Initial page tables (init_pml4):
├── Compiled into kernel binary (just data)
├── Loaded at: Physical 0x01500000 (example)
└── Just DRAM cells holding values! 

Structure in RAM:
Address 0x01500000:  [PML4 entry 0]
Address 0x01500008:  [PML4 entry 1]
Address 0x01500010:  [PML4 entry 2]
...

When CR3 loaded:
├── CR3 = 0x01500000
├── CPU's MMU now reads from this address
└── Hardware table walker active! 
```

**Mode transitions:**

```
Protected → Long Mode:

Set EFER.LME bit:
└── CPU internal state changes (silicon)

Set CR0.PG bit:
└── MMU circuit enables 

CPU now in different mode:
├── 64-bit instruction decoding
├── Extended registers available
├── Virtual addressing active
└── All in hardware! Silicon state! 
```

---

# 10. Connections to Other Topics

## How Init Enables Everything

**Connection to Page Tables:**

```
TOPIC 8: Page table format (4-level, 64-bit entries)
TOPIC 10: Creates those tables 

init.c:
├── Allocates PML4, PDPT, PD, PT
├── Fills in entries
├── Points CR3 to them
└── Enables what TOPIC 8 describes! 
```

**Connection to Page Faults:**

```
TOPIC 7: Page fault handling
TOPIC 10: Must run first! 

Can't have page faults without page tables:
├── init.c creates page tables 
├── init.c enables paging 
└── Then page faults can occur 
```

**Connection to TLB:**

```
TOPIC 9: TLB caching
TOPIC 10: First to fill TLB 

After enable paging:
├── TLB starts empty
├── First memory accesses fill it
└── Gradually warms up 
```

**Connection to Buddy Allocator:**

```
mm/page_alloc.c: Buddy allocator
TOPIC 10: Bootstraps it! 

init.c provides:
├── Memory detection (how much RAM)
├── Memory zones (DMA, Normal)
├── Page structures (tracking)
└── Initial free memory
```

**Connection to CPU Detection:**

```
TOPIC 5: CPU feature detection
TOPIC 10: Uses features! 

Check capabilities:
├── PAE supported?  Enable it
├── Long mode supported?  Use it
├── Huge pages supported?  Use them
└── Detection guides initialization! 
```

---

# Summary

## What You've Mastered

**You now understand memory initialization!**

```
- What it is: Bootstrap from nothing to working memory
- Why needed: Chicken-and-egg problem solver
- E820 detection: Finding RAM via BIOS
- Initial page tables: Compiled into kernel
- Critical transition: Enabling paging and 64-bit
- Direct mapping: All RAM accessible
- Memory zones: DMA, DMA32, Normal
- memblock: Bootstrap allocator
- Transfer to buddy: Final handoff
- Complete sequence: Power-on to operational
```

---

## Key Takeaways

**The bootstrap problem:**

```
Need memory to set up memory management!

Deadlocks:
├── Page tables need memory
├── Memory needs allocator
├── Allocator needs page tables
└── Circular! 

Solution:
├── Compile page tables into kernel 
├── Use simple memblock allocator 
├── Transfer to real allocator 
└── Three stages break deadlock! 
```

**The E820 map:**

```
BIOS provides ground truth:

Maps physical address space:
├── RAM regions (usable) 
├── Reserved regions (APIC, VGA, BIOS) 
└── Total: 8 GB in example

Kernel must respect this!
├── Don't allocate from reserved 
└── Only use RAM regions 
```

**The critical transition:**

```
One instruction changes everything:

Before: MOV CR0, RAX (enable paging)
├── Physical addressing
├── 32-bit mode
└── Limited

After: (paging enabled)
├── Virtual addressing 
├── 64-bit mode 
├── Full capability 

Identity + high mappings make it work! 
```

**The direct map:**

```
Simple and powerful:

Physical X → Virtual (0xFFFF888000000000 + X)

Benefits:
├── Any physical page instantly accessible
├── No temporary mappings needed
├── Just arithmetic! 
└── Kernel convenience! 
```

---

## The Big Picture

**Boot sequence:**

```
Power-on (T=0):
└── Nothing initialized

BIOS (T=0-10ms):
└── Detect RAM, create E820

Bootloader (T=10-11ms):
└── Load kernel into RAM

Enable paging (T=11ms):
└── Critical transition 

Parse E820 (T=11-12ms):
└── Know what memory exists

Create page tables (T=12-13ms):
└── Map all RAM

Initialize zones (T=13-14ms):
└── DMA, DMA32, Normal

Create page tracking (T=14-15ms):
└── struct page for each

Transfer to buddy (T=15ms):
└── Real allocator ready

Normal operation (T=20ms+):
└── Everything works! 

From nothing to working memory in 20ms! 
```

---

## You're Ready!

With this knowledge, you can:
- Understand kernel boot process
- Debug early boot issues
- Add custom memory initialization
- Understand memory detection
- Work with bootloaders

> **Memory initialization is the foundation - you now know how x86 bootstraps virtual memory from scratch!**

---

**Next up: I/O Memory Mapping**

Ready to learn how to map device registers for driver development? 
