# Linux Memory Management Internals (mm/)

> Ever wondered how your 8 GB computer runs programs using 50 GB of memory? How Firefox's crash doesn't corrupt Chrome's data? How the kernel decides which program to kill when RAM runs out?

**You're about to find out!**

---

## What's Inside This Repository?

> This repository explores the **mm/** directory - the **BRAIN of Memory Management** where virtual becomes physical!

Here's what we cover:

```
mm/
├── memory.c        ← Page fault handling (THE HEART!)
├── mmap.c          ← Memory mapping (files → memory)
├── page_alloc.c    ← Physical RAM management
├── slab.c/slub.c   ← Fast object caching
├── filemap.c       ← File page cache
├── swap.c          ← Disk overflow (when RAM full)
├── vmscan.c        ← Page reclaiming (freeing memory)
├── vmalloc.c       ← Large kernel allocations
├── readahead.c     ← Predictive file loading
├── mprotect.c      ← Permission enforcement
├── kmalloc.c       ← Kernel's malloc()
└── oom_kill.c      ← Last resort killer
```

**This directory controls:**
- Virtual memory illusion (private address spaces)
- Physical RAM allocation (the actual bytes)
- Page fault handling (demand loading)
- File caching (speed up disk access)
- Memory protection (security enforcement)
- Swapping (use disk as RAM overflow)
- Out-of-memory handling (who dies when RAM exhausted)
- Kernel memory allocation (kernel's own needs)

---

## What You'll Learn

Each file explained with:

- **What it does** - Clear purpose and responsibility
- **Why it exists** - The problems it solves
- **How it works** - Physical mechanisms and workflows
- **Important details** - Critical concepts that make it tick
- **Real examples** - Concrete scenarios you can visualize

**No code, pure theory!** Build a solid mental model before diving into implementation.

---

## How to Use This Guide

**Read sequentially** - Each file builds understanding:
1. Start with memory.c (understand virtual memory)
2. Continue through all 12 files
3. Finish with the complete Firefox journey (all files working together)
4. Refer back as needed while studying kernel source

**Prerequisites:**
- Basic understanding of CPU, RAM, and disk
- Familiarity with the kernel/ directory concepts (processes, interrupts)
- Curiosity about how computers actually work!

---

# FILE 1: memory.c

> **The heart of virtual memory - where FAKE addresses become REAL**

## What is memory.c?

**THE CORE of the memory management system!**

```
memory.c = ~5,000 lines controlling:
├── Page fault handling (MOST IMPORTANT!)
├── Virtual memory operations
├── Page table management
├── Copy-on-write mechanism
├── Memory mapping operations
└── Address space manipulation
```

**This file makes EVERYTHING work!**

---

## The Fundamental Problem

**Physical Reality (1980s computers):**

```
Computer has 4 MB RAM:
    Physical addresses: 0x000000 - 0x3FFFFF

Load Program A:
    Compiler says: "Code starts at 0x000000"
    Load at: 0x000000 

Load Program B:
    Compiler says: "Code starts at 0x000000" ← SAME ADDRESS!
    
COLLISION! 
```

**Every program wants the same addresses!**

---

## The Virtual Memory Solution

**The revolutionary idea:**

> Give each program FAKE addresses that map to DIFFERENT physical locations!

```
Program A thinks:
    "I have addresses 0x00000000 - 0xFFFFFFFF"
    "I own ALL memory!"

Program B thinks:
    "I have addresses 0x00000000 - 0xFFFFFFFF"
    "I own ALL memory!"

Both see SAME virtual addresses!
But MMU maps to DIFFERENT physical addresses!

MAGIC! 
```

---

## How Virtual Memory Works

**The translation layer:**

```
PROGRAM A:
    Uses virtual: 0x00400000
    ↓ MMU translates
    Maps to physical: 0x10000000
    
PROGRAM B:
    Uses virtual: 0x00400000 ← SAME VIRTUAL!
    ↓ MMU translates (different page table)
    Maps to physical: 0x20000000 ← DIFFERENT PHYSICAL!
    
No collision! Both programs coexist! 
```

**Key Components:**

```
Virtual Address = What programs SEE
Physical Address = Where data ACTUALLY lives
MMU = Hardware translator (in CPU)
Page Table = Translation dictionary
CR3 Register = Points to page table
```

---

## Virtual Address Space Layout

**What each program sees (64-bit Linux):**

```
0x0000000000000000 - 0x0000000000400000: NULL guard
    ↓ Empty (catch NULL pointer bugs!)
    
0x0000000000400000 - 0x0000000000600000: Code (.text)
    ↓ Your program's instructions
    
0x0000000000600000 - 0x0000000000800000: Data (.data, .bss)
    ↓ Global variables
    
0x0000000000800000 - 0x0000100000000000: HEAP
    ↓ malloc() allocates here, grows UP ↑
    
    [Huge gap - unused virtual space]
    
0x00007FFF00000000 - 0x00007FFFFFFFFFFF: STACK
    ↓ Function calls, grows DOWN ↓
    
0xFFFF800000000000 - 0xFFFFFFFFFFFFFFFF: KERNEL
    ↓ Kernel code and data
```

**Why this layout?**

```
NULL guard:
    Catch *ptr = 5 when ptr = NULL
    Access 0x0 → Segmentation fault! 
    
Heap grows up, Stack grows down:
    Gap between them
    If they meet → Stack overflow detected!
    
Kernel at top:
    Always mapped (convenient!)
    Protected from user access
```

---

## Page Faults - THE CRITICAL MECHANISM

**What is a page fault?**

> Virtual address not in page table → MMU generates PAGE FAULT!

**When they happen:**

```
REASON 1: Demand paging (load on access)
    Program has 100 MB executable
    Don't load all at start! Too slow!
    Load pages as accessed 

REASON 2: Swapped out (RAM full)
    Page moved to disk
    Access it → Load back from swap
    
REASON 3: Copy-on-write (after fork)
    Parent and child share pages
    Write attempt → Copy page first
    
REASON 4: Invalid access (ERROR!)
    NULL pointer dereference
    Write to read-only page
    Kill program! 
```

---

## Page Fault Handler Flow

**Step-by-step execution:**

```
STEP 1: CPU detects page fault
    Program: MOV RAX, [0x00500000]
    MMU: Page not present!
    CPU: Save address in CR2 register
    CPU: Jump to kernel page fault handler

STEP 2: Kernel identifies fault type
    Read CR2: Address 0x00500000
    Check process's VMAs (memory regions)
    Found: Code region (valid access)
    Reason: Page not loaded yet

STEP 3: Load page from disk
    File: /usr/bin/firefox
    Offset: Calculated from address
    Allocate physical page
    Read 4 KB from file
    
STEP 4: Update page table
    Virtual 0x00500000 → Physical 0x50000000
    Mark: Present, Read-only, Executable
    
STEP 5: Return to program
    CPU retries instruction
    MMU translates successfully
    Instruction executes! 
```

---

## Page Tables - The Translation Map

**Multi-level structure (4 levels on x86-64):**

```
Why NOT simple array?
    64-bit address space: 128 TB!
    Page size: 4 KB
    Need: 34 BILLION entries!
    Each entry: 8 bytes
    Total: 274 GB per process! 
    
Solution: 4-level hierarchical tables!
    Only create tables for USED memory
    Sparse address space = Small tables 
```

**How translation works:**

```
Virtual address: 0x00400000

Parse bits:
    Bits 39-47: Level 4 index (PML4)
    Bits 30-38: Level 3 index (PDPT)
    Bits 21-29: Level 2 index (PD)
    Bits 12-20: Level 1 index (PT)
    Bits 0-11:  Offset (4096 bytes)

Walk page tables:
    CR3 → PML4 → PDPT → PD → PT → Physical page
    
5 memory accesses just to translate ONE address!
```

---

## TLB - The Speed Secret

**The problem:**

> 5 memory accesses per translation = TOO SLOW!

**The solution: TLB (Translation Lookaside Buffer)**

```
TLB = Small cache in CPU
    Caches recent translations
    Size: 64-1024 entries
    Speed: 1 CPU cycle (~0.3 nanoseconds)

TLB HIT (99% of time):
    Virtual → Check TLB → Physical address
    Total: 1 cycle 
    
TLB MISS (1% of time):
    Walk page tables (5 memory reads)
    Add to TLB for next time
    Total: 200 cycles 
```

**Why context switches are expensive:**

```
When switching processes:
    Different page tables!
    Must FLUSH TLB (clear all entries)
    Next accesses: All TLB misses!
    Performance hit!
```

---

## Important Details

### Detail 1: Zero Page Optimization

**The problem:**
```
Programs need lots of zero-filled memory:
    BSS section (uninitialized globals)
    malloc() allocations
    Stack pages
```

**The solution:**
```
ONE physical zero page (filled with zeros)
ALL zero virtual pages map to it (read-only!)

On write:
    Page fault!
    Allocate real page
    Fill with zeros
    Update mapping
    
Saves THOUSANDS of pages! 
```

### Detail 2: Copy-on-Write (COW)

**After fork():**
```
Don't copy all memory!
    Parent and child SHARE pages
    Mark all pages READ-ONLY
    
On write attempt:
    Page fault!
    Copy page NOW
    Each gets private copy
    
Saves time and memory! 
```

### Detail 3: Transparent Huge Pages

**Normal vs Huge:**
```
Normal page: 4 KB
Huge page: 2 MB (512× bigger!)

Advantage:
    TLB entry covers 2 MB instead of 4 KB
    512× better TLB efficiency!
    10-30% performance improvement! 
    
Kernel uses automatically when possible!
```

---

# FILE 2: mmap.c

> **Maps files into memory - the bridge between disk and RAM**

## What is mmap.c?

**Creates virtual memory regions!**

```
mmap.c handles:
├── Loading executables into memory
├── Memory-mapped files (files as memory)
├── Anonymous memory (malloc for large sizes)
├── Shared memory between processes
└── Virtual Memory Area (VMA) management
```

---

## The Problem

**Traditional file access:**

```
read() system call:
    1. User calls read(fd, buffer, size)
    2. Kernel reads from disk → Kernel buffer
    3. Kernel copies to user buffer
    
Two copies! Slow! 
```

**mmap() solution:**

```
mmap() maps file directly:
    1. Create virtual memory mapping
    2. Don't load anything yet!
    3. User accesses memory
    4. Page fault!
    5. Kernel loads page from file
    6. Direct access! One copy! 
```

---

## How mmap Works

**Creating a mapping:**

```
Program: ptr = mmap(NULL, 10MB, PROT_READ, MAP_PRIVATE, fd, 0)

Kernel creates VMA (Virtual Memory Area):
    Start: 0x10000000 (kernel picks address)
    End: 0x10A00000 (10 MB)
    File: /home/user/data.bin
    Offset: 0
    Permissions: Read-only
    Flags: PRIVATE (copy-on-write)
    
Returns: 0x10000000

Program now has pointer!
But NOTHING loaded yet! (demand paging)
```

---

## VMA Structure

**What is a VMA?**

> VMA = One contiguous region in virtual address space

**Each process has many VMAs:**

```
Firefox VMAs:
    VMA 1: Code (0x400000-0x600000, read+exec)
    VMA 2: Data (0x600000-0x800000, read+write)
    VMA 3: Heap (0x800000-0x1000000, read+write)
    VMA 4: libssl.so (0x40000000-0x40200000, read+exec)
    VMA 5: Stack (0x7FFF0000-0x80000000, read+write)
```

**On page fault:**
```
Kernel searches VMAs:
    "Which VMA contains faulting address?"
    Determines how to handle based on VMA type
    
Fast lookup: Red-black tree (O(log n))
```

---

## MAP_SHARED vs MAP_PRIVATE

**MAP_SHARED:**
```
Multiple processes see SAME physical pages!

Use cases:
    - Shared libraries (libc.so shared by all)
    - Shared memory IPC
    - Memory-mapped databases
    
Process A writes:
    Process B sees change immediately! 
```

**MAP_PRIVATE:**
```
Each process gets own COPY (copy-on-write)

Use cases:
    - Loading executables
    - Normal malloc (large sizes)
    
Process A writes:
    Copy page, A gets private copy
    Process B still sees original 
```

---

## Important Details

### Detail 1: VMA Merging

**Adjacent compatible VMAs merge:**
```
Before:
    VMA1: 0x1000-0x2000 (read+write, anonymous)
    VMA2: 0x2000-0x3000 (read+write, anonymous)
    
After:
    VMA: 0x1000-0x3000 (merged!)
    
Reduces VMA count = Faster lookups! 
```

### Detail 2: Address Space Layout Randomization (ASLR)

**Security feature:**
```
Without ASLR:
    Code always at: 0x00400000
    Heap always at: 0x00800000
    Predictable! Easy to exploit! 
    
With ASLR:
    Code at: 0x56400000 (random!)
    Heap at: 0x55800000 (random!)
    Unpredictable! Hard to exploit! 
```

---

# FILE 3: page_alloc.c

> **Manages physical RAM pages - the banker of memory**

## What is page_alloc.c?

**THE manager of physical memory!**

```
page_alloc.c controls:
├── All physical page allocation/freeing
├── Buddy allocator algorithm
├── Memory zones (DMA, Normal, High)
├── Watermarks (min, low, high)
└── Emergency reserves
```

---

## The Page Concept

**Physical memory divided into pages:**

```
Page size: 4 KB (4096 bytes)

16 GB RAM:
    16 GB / 4 KB = 4,194,304 pages
    
Each page is a unit:
    Page 0: 0x00000000 - 0x00000FFF
    Page 1: 0x00001000 - 0x00001FFF
    Page 2: 0x00002000 - 0x00002FFF
    ...
```

---

## Buddy Allocator Algorithm

**The problem:**

> How to allocate pages efficiently? Avoid fragmentation!

**The solution: Buddy system**

**Pages organized in power-of-2 groups:**

```
Free lists:
    Order 0: 1 page (4 KB)
    Order 1: 2 pages (8 KB)
    Order 2: 4 pages (16 KB)
    Order 3: 8 pages (32 KB)
    ...
    Order 10: 1024 pages (4 MB)
```

**Allocation example (need 3 pages):**

```
STEP 1: Round up
    3 pages → 4 pages (order 2)
    
STEP 2: Check order 2 list
    Empty! 
    
STEP 3: Go to higher order
    Order 3 has 8-page block 
    
STEP 4: Split block
    8 pages → 4 + 4
    Return first 4
    Put second 4 on order 2 list
```

**Freeing example (free 4-page block):**

```
STEP 1: Find buddy
    Block at 0x10000 freed
    Buddy at 0x14000
    
STEP 2: Buddy free too?
    YES! 
    
STEP 3: Merge
    0x10000 + 0x14000 = 8-page block
    
STEP 4: Repeat
    Check if 8-page buddy free
    Merge again if possible
    
Self-organizing! Reduces fragmentation! 
```

---

## Memory Zones

**RAM divided by hardware constraints:**

```
ZONE_DMA (0-16 MB):
    For old devices (ISA cards)
    Can only access first 16 MB
    
ZONE_NORMAL (16 MB - 896 MB on 32-bit):
    Normal kernel memory
    Directly mapped
    
ZONE_HIGHMEM (> 896 MB on 32-bit):
    Not directly mapped
    Needs special handling
    
(64-bit: Mostly just DMA and Normal)
```

**Why zones matter:**

```
Some devices limited:
    Old floppy: Only first 16 MB
    Some network cards: Only first 4 GB
    
Must allocate from correct zone! 
```

---

## Watermarks

**Three levels per zone:**

```
pages_high: 10,000 pages (safe)
pages_low:  5,000 pages (warning)
pages_min:  2,500 pages (critical)
```

**What they control:**

```
Free > high:
    Everything fine! 
    Normal allocation
    
Free < low:
    Warning! 
    Wake up kswapd (background reclaim)
    Still allow allocation
    
Free < min:
    Critical! 
    Direct reclaim (allocator waits)
    Only emergency allocation allowed
```

---

## Important Details

### Detail 1: GFP Flags

**Tell page_alloc HOW to allocate:**

```
GFP_KERNEL:
    Normal allocation
    Can sleep (wait for memory)
    Used by most kernel code
    
GFP_ATOMIC:
    Cannot sleep!
    Must succeed immediately
    Used in interrupts
    Uses emergency reserves
    
GFP_DMA:
    Must allocate from DMA zone
    For devices needing low memory
```

### Detail 2: Page Migration

**Moving pages in physical RAM:**

```
Why needed:
    - Memory hotplug (removing RAM module)
    - Defragmentation (create huge page)
    - NUMA optimization (move to local node)
    
How it works:
    1. Allocate new destination page
    2. Copy content
    3. Update all page tables (reverse mapping!)
    4. Free old page
    5. Flush TLB everywhere
    
Transparent to programs! 
```

---

# FILE 4: slub.c

> **Fast object caching - kernel's high-speed allocator**

## What is slub.c?

**Object-oriented memory allocator!**

```
SLUB manages:
├── Kernel object caches (task_struct, inodes, etc.)
├── Fast allocation (no page allocation needed)
├── Per-CPU caches (lock-free!)
└── Slab structures
```

---

## The Problem

**Kernel allocates/frees objects constantly:**

```
task_struct:
    fork() → allocate
    exit() → free
    fork() → allocate again
    Pattern: allocate, use, free, repeat!
    
Using page_alloc directly:
    Every fork(): Allocate page (slow)
    Use tiny part (10 KB of 4 KB page)
    Free page (wasteful)
    
Inefficient! 
```

---

## The SLUB Solution

**Pre-allocate many objects:**

```
Boot time:
    Allocate pages
    Divide into objects
    Create cache
    
Allocation time:
    Just grab object from cache!
    Fast! No page allocation! 
```

**Example: task_struct cache**

```
Object size: 10 KB
Allocate: 3 pages (12 KB)
Objects per slab: 1 (barely fits)

Slab structure:
    [task_struct #1] [used]
    Free list → Empty
    
On fork():
    Check slab: No free objects
    Allocate new slab
    Return object
    Very fast! 
```

---

## Slab States

**Three states:**

```
FULL slab:
    All objects allocated
    Not scanned for allocation
    
PARTIAL slab:
    Some allocated, some free
    Active! Used for allocations 
    
EMPTY slab:
    All objects free
    Can return pages to page_alloc!
```

---

## Per-CPU Caches

**The scalability secret:**

```
Problem:
    Multiple CPUs allocating simultaneously
    Shared cache needs lock
    Lock contention = Slow! 
    
Solution:
    Each CPU has OWN cache!
    
CPU 0: Own free list
CPU 1: Own free list
CPU 2: Own free list

No locking! Fast! 
```

**How it works:**

```
CPU 0 allocates task_struct:
    1. Check CPU 0's free list
    2. Object available? Take it! (no lock!)
    3. Return to caller
    
CPU 0's list empty?
    1. Lock shared slab (briefly)
    2. Get batch of 16 objects
    3. Unlock
    4. Next 15 allocations: No lock! 
```

---

## Important Details

### Detail 1: Object Alignment

**Cache line alignment:**
```
Cache line: 64 bytes

Unaligned object at 0x1003:
    Spans 2 cache lines
    2 cache fetches needed 
    
Aligned object at 0x1000:
    Fits in 1 cache line
    1 cache fetch 
    
2× faster! 
```

### Detail 2: Slab Coloring

**Reduce cache conflicts:**
```
Without coloring:
    Slab 1: Objects at 0x0, 0x40, 0x80...
    Slab 2: Objects at 0x0, 0x40, 0x80...
    Same cache line offsets! Conflicts! 
    
With coloring:
    Slab 1: Objects at 0x00, 0x40, 0x80...
    Slab 2: Objects at 0x10, 0x50, 0x90...
    Different offsets! Better cache use! 
```

---

# FILE 5: filemap.c

> **The page cache - makes disk access fast**

## What is filemap.c?

**Caches file contents in RAM!**

```
filemap.c manages:
├── Page cache (file contents in memory)
├── Dirty pages (modified in RAM)
├── Write-back (syncing to disk)
├── Shared file mappings
└── Read/write operations
```

---

## The Page Cache

**What it is:**

> RAM used to cache file contents - avoid repeated disk reads!

**Structure:**

```
Hash table: (inode, offset) → Physical page

File /usr/bin/firefox:
    Offset 0: Page 500
    Offset 4096: Page 501
    Offset 8192: Page 502
    ...
```

**Why it matters:**

```
Without cache:
    Firefox starts: Read 50 MB (500ms)
    Firefox exits
    Firefox starts: Read 50 MB again (500ms)
    
With cache:
    First start: Read, cache in RAM (500ms)
    Exit: Cache stays!
    Second start: Read from cache (5ms) 
    
100× faster!
```

---

## Dirty Pages

**What they are:**

```
Clean page:
    Matches disk
    Can drop anytime 
    
Dirty page:
    Modified in RAM
    Must write to disk first! 
```

**Write-back process:**

```
Program writes to file:
    write(fd, data, size)
    
Kernel marks page dirty:
    Don't write immediately!
    Batch writes for efficiency
    
Write-back daemon runs:
    Every 30 seconds OR
    When too many dirty pages
    
    Writes all dirty pages to disk
    Marks clean
```

**Why batching?**

```
Without:
    1000 writes = 1000 disk writes 
    
With batching:
    1000 writes to same page = 1 disk write! 
```

---

## Read vs Write Paths

**Read path:**

```
read(fd, buffer, 4096):

STEP 1: Check page cache
    Key: (file, offset)
    
STEP 2a: CACHE HIT! 
    Copy from cache to buffer
    Return (microseconds)
    
STEP 2b: CACHE MISS 
    Allocate page
    Read from disk (10ms)
    Add to cache
    Copy to buffer
```

**Write path:**

```
write(fd, data, 4096):

STEP 1: Check page cache
    
STEP 2a: Page in cache 
    Copy data to cache
    Mark dirty
    Return (microseconds)
    
STEP 2b: Not in cache
    Read page first! (can't lose data)
    Modify
    Mark dirty
    Return
```

---

## Important Details

### Detail 1: Shared File Mappings

**Multiple processes, one cache:**

```
Process A: mmap(file.txt, MAP_SHARED)
Process B: mmap(file.txt, MAP_SHARED)

Both map SAME physical pages!

Process A writes:
    Process B sees immediately! 
    
Memory sharing:
    File 10 MB
    Without sharing: 20 MB RAM 
    With sharing: 10 MB RAM 
```

### Detail 2: Cache Size

**Dynamic sizing:**

```
Use ALL free RAM for cache!

16 GB system:
    Kernel: 1 GB
    Programs: 8 GB
    Free: 7 GB → ALL used for cache!
    
When program needs memory:
    Evict cache pages
    Give to program
    
Cache grows and shrinks automatically! 
```

---

# FILE 6: swap.c

> **Disk overflow - when RAM isn't enough**

## What is swap.c?

**Manages overflow to disk!**

```
swap.c handles:
├── Swap space management
├── Swapping pages out (RAM → Disk)
├── Swapping pages in (Disk → RAM)
├── Swap cache (recently swapped pages)
└── Swap readahead
```

---

## The Problem

**Physical RAM limit:**

```
System has: 8 GB RAM
Programs want: 12 GB

Without swap:
    Hit 8 GB → OOM killer! 
    Kill programs!
    
With swap:
    Use disk as overflow
    12 GB virtual = 8 GB RAM + 4 GB swap 
```

---

## Swap Space

**Where pages go:**

```
Swap space: /swapfile (4 GB)

Divided into slots:
    Slot size: 4 KB (same as page)
    Total slots: 1,048,576
    
Bitmap tracks usage:
    Bit 0: 1 (used)
    Bit 1: 0 (free)
    Bit 2: 1 (used)
    ...
```

---

## Swap-out Process

**Moving page to disk:**

```
vmscan.c picks victim: Chrome's page at virtual 0x30000000
Physical page: 1500

STEP 1: Allocate swap slot
    Find free slot: 2500
    Mark as used
    
STEP 2: Write to swap
    Write physical page 1500 → swap slot 2500
    I/O operation (10-50ms)
    
STEP 3: Update page table
    Before: Virtual 0x30000000 → Physical 1500 (Present=1)
    After: Virtual 0x30000000 → Swap 2500 (Present=0)
    
STEP 4: Free physical page
    Page 1500 back to page_alloc
    Available for reuse! 
```

---

## Swap-in Process

**Loading page from disk:**

```
Chrome accesses swapped page:
    MOV RAX, [0x30000000]
    MMU: Present=0! PAGE FAULT!
    
STEP 1: Detect swap fault
    Page table: Present=0, Swap offset=2500
    "This is swap fault!"
    
STEP 2: Allocate physical page
    Get free page: 3000
    
STEP 3: Read from swap
    Read swap slot 2500 → page 3000
    I/O operation (slow!)
    
STEP 4: Update page table
    Virtual 0x30000000 → Physical 3000 (Present=1)
    
STEP 5: Free swap slot
    Slot 2500 available again
    
STEP 6: Resume Chrome
    Instruction retries
    Success! 
```

---

## Important Details

### Detail 1: Swap Cache

**Optimization for frequently swapped pages:**

```
Page swapped out:
    Keep swap slot allocated
    
Page swapped in:
    Swap slot still valid!
    
Need to swap out again:
    Already on disk!
    No write needed! 
    Just update page table
    
Saves disk writes!
```

### Detail 2: Swap Readahead

**Predict sequential access:**

```
Access swap slot 100:
    Also read slots 101, 102, 103...
    
Sequential pattern:
    All loaded together
    Faster than individual reads! 
```

---

# FILE 7: vmscan.c

> **The memory reclaimer - frees pages when needed**

## What is vmscan.c?

**Finds pages to evict!**

```
vmscan.c controls:
├── LRU lists (Least Recently Used)
├── kswapd daemon (background reclaim)
├── Direct reclaim (emergency)
├── Page scanning algorithms
└── Victim selection
```

---

## LRU Lists

**Track page usage:**

```
Two lists per zone:

Active list:
    Recently accessed pages
    HOT pages - don't evict!
    
Inactive list:
    Not recently accessed
    COLD pages - evict from here!
```

**How pages move:**

```
New page allocated:
    → Inactive list (tail)
    
Page accessed:
    → Active list (head)
    
Page not accessed (long time):
    Active → Inactive
    
Page in inactive, accessed again:
    → Active (second chance!)
```

---

## kswapd Daemon

**Background memory reclaimer:**

```
kswapd = Kernel thread
    Always running (sleeping most of time)
    Wakes when memory low
```

**Trigger points:**

```
Free pages > high watermark (10,000):
    kswapd sleeps (nothing to do)
    
Free pages < low watermark (5,000):
    kswapd wakes up! 
    Start reclaiming
    Goal: Get back above high
    
Free pages < min watermark (2,500):
    Emergency! 
    Direct reclaim!
```

**What kswapd does:**

```
Wake up:
    1. Scan inactive LRU list
    2. Check if page accessed recently
    3. Not accessed? Evict!
    4. Accessed? Move to active (second chance)
    5. Continue until enough freed
    6. Go back to sleep
```

---

## Direct Reclaim

**Emergency reclaim:**

> When kswapd can't keep up, process does the work itself!

```
Firefox allocates 100 MB:
    Free pages: 2,000 (below min 2,500) 
    
page_alloc.c:
    "Cannot allocate! Must reclaim!"
    
Firefox BLOCKS:
    Stops running
    Kernel reclaims using Firefox's time
    
Reclaim pages:
    Scan LRU
    Evict pages
    Write dirty pages
    Swap anonymous pages
    
Freed enough:
    Firefox resumes
    Allocation succeeds 
```

**Why this matters:**

```
kswapd proactive:
    Runs in background
    System stays responsive 
    
Direct reclaim reactive:
    Process waits (latency!)
    But prevents crash 
```

---

## Page Reference Tracking

**Hardware-assisted aging:**

```
Page table entry has "Referenced" bit:
    Set by CPU when page accessed
    Cleared by kernel periodically
```

**How kernel uses it:**

```
Every 10 seconds:
    1. Kernel clears all referenced bits
    2. Wait 10 seconds
    3. Check which bits set again
    
Referenced bit SET:
    Page accessed in last 10 seconds
    HOT page! Keep in active list 
    
Referenced bit CLEAR:
    Not accessed in 10 second
    COLD page! Move to inactive 
```

**Example timeline:**

```
T=0:  Clear referenced bit (bit=0)
T=5:  Program accesses page
      CPU sets bit=1 (automatic!)
T=10: Kernel checks: bit=1 → HOT! 
      Move to active list
      Clear bit again
T=20: Kernel checks: bit=0 → COLD! 
      Move to inactive list
```

---

## Important Details

### Detail 1: Reclaim Priority

**Aggressive scanning control:**

```
Priority levels: 0-12
    12 = Gentle (scan 1/4096 of pages)
    0 = Aggressive (scan ALL pages)

Start gentle:
    Priority 12: Try to reclaim
    Not enough? Increase to 11
    Still not enough? Increase to 10
    Continue...
    
Last resort:
    Priority 0: Scan everything! 
```

### Detail 2: LRU Approximation

**Why approximation?**

```
True LRU = Track every access
    Impossible! (too much overhead)
    
Linux LRU = Approximation
    Sample page accesses
    Referenced bit check
    Good enough! 
```

---

# FILE 8: vmalloc.c

> **Large kernel allocations - virtual contiguity without physical**

## What is vmalloc.c?

**Kernel's large memory allocator!**

```
vmalloc.c provides:
├── Large allocations (> 128 KB)
├── Virtually contiguous memory
├── Physically scattered pages
└── Guard pages (debug)
```

---

## The Problem

**Kernel needs 10 MB contiguous:**

```
System running for days:
    Physical RAM fragmented!
    
Free pages: 100,000 (plenty!)
But: Scattered everywhere
No 10 MB contiguous block! 

kmalloc fails! 
```

---

## The vmalloc Solution

**Virtual contiguity, physical scatter:**

```
Virtual address space IS contiguous:
    0xFFFF800010000000 - 0xFFFF800010A00000 (10 MB)
    Kernel sees contiguous memory! 
    
Physical pages scattered:
    Virtual 0xFFFF800010000000 → Physical page 1000
    Virtual 0xFFFF800010001000 → Physical page 5000
    Virtual 0xFFFF800010002000 → Physical page 2000
    Different pages! Scattered!
    
MMU translates, kernel doesn't care! 
```

---

## vmalloc vs kmalloc

**When to use which?**

**kmalloc:**
```
Physically contiguous

Advantages:
    Fast (better cache locality)
    Can use for DMA
    Less TLB pressure
    
Disadvantages:
    Limited size
    Can fail (fragmentation)
    
Use for:
    - Small allocations (< 128 KB)
    - DMA buffers
    - Performance-critical
```

**vmalloc:**
```
Virtually contiguous, physically scattered

Advantages:
    Large allocations possible
    Always succeeds (enough total memory)
    No fragmentation worries
    
Disadvantages:
    Slower (TLB thrashing)
    Can't use for DMA
    More overhead
    
Use for:
    - Large allocations (> 128 KB)
    - Module loading
    - Non-critical buffers
```

---

## vmalloc Address Space

**Kernel virtual memory layout:**

```
0xFFFF800000000000 - 0xFFFF87FFFFFFFFFF:
    Direct mapping (1:1 physical)
    
0xFFFF880000000000 - 0xFFFFC7FFFFFFFFFF:
    vmalloc area ← Allocations here!
    Size: ~128 GB
    
0xFFFFC80000000000 - 0xFFFFFFFFFFFFFFFF:
    Other kernel areas
```

---

## Important Details

### Detail 1: Guard Pages

**Buffer overflow detection:**

```
Allocation layout:
    Alloc 1: 0xFFFF880010000000-0xFFFF880010100000 (1 MB)
    GUARD:   0xFFFF880010100000-0xFFFF880010101000 (4 KB) ← UNMAPPED!
    Alloc 2: 0xFFFF880010101000-0xFFFF880010201000 (1 MB)
    
Code bug writes past allocation 1:
    Tries to write guard page
    PAGE FAULT! 
    
Immediate crash with clear error! 
Easy to debug!
```

### Detail 2: Lazy Unmapping

**Optimization for vfree():**

```
Problem:
    vfree() must unmap pages
    Must flush TLB on ALL CPUs
    Expensive! (100+ microseconds) 
    
Solution:
    Mark area as free (fast)
    Don't unmap immediately
    Batch many unmappings
    One TLB flush for all! 
    
vfree() returns quickly!
```

---

# FILE 9: readahead.c

> **Predictive loading - reads before you ask**

## What is readahead.c?

**Predicts future file accesses!**

```
readahead.c provides:
├── Sequential access detection
├── Dynamic window sizing
├── Asynchronous loading
├── Thrashing detection
└── Application hints (fadvise)
```

---

## The Problem

**Sequential file access:**

```
Program reads file sequentially:
    Read offset 0 (4 KB) → Wait for disk
    Read offset 4096 → Wait for disk
    Read offset 8192 → Wait for disk
    ...
    
Constant waiting! 

Each read: 10ms
Reading 100 MB: 25,600 reads × 10ms = 256 seconds! 
```

---

## The Readahead Solution

**Predict and pre-load:**

```
Program reads offset 0:
    Kernel thinks: "Probably sequential!"
    Load pages 0-15 (64 KB)
    
By time program reads offset 4096:
    Already in cache! 
    No waiting!
    
Continue predicting:
    Stay ahead of program! 
```

---

## Dynamic Window Sizing

**Adaptive window size:**

```
Start small, grow if sequential:

First read:
    Window: 4 pages (16 KB)
    
Second sequential read:
    Window: 8 pages (32 KB)
    Pattern confirmed! 
    
Third sequential read:
    Window: 16 pages (64 KB)
    Growing! 
    
Max window:
    128 KB (32 pages)
```

**Why start small?**

```
Random access pattern:
    Readahead wastes I/O!
    Small window = Less waste
    
Sequential pattern emerges:
    Grow window
    Maximize benefit! 
```

---

## Async vs Sync Readahead

**Synchronous readahead:**

```
read() at offset 0:
    Cache miss!
    Trigger readahead (0-15)
    WAIT for page 0 
    Return to program
    
Pages 1-15 load in background
```

**Asynchronous readahead:**

```
read() at offset 0:
    HIT! (from previous readahead)
    Return immediately 
    
Detect: Approaching window end
Start next readahead (16-31) in background

By time program reaches offset 16:
    Already loaded! 
    No blocking ever!
```

---

## Readahead Page Marking

**The trigger mechanism:**

```
Readahead pages 0-15:
    Mark page 12 with PG_readahead flag
    
When program accesses page 12:
    Kernel sees flag
    Triggers next readahead (16-31)
    
Why page 12, not 15?
    Need time to load next window!
    While reading 12-15, load 16-31
    No blocking! 
```

---

## Important Details

### Detail 1: Thrashing Detection

**When readahead hurts:**

```
File: 500 MB
RAM: 50 MB
Readahead: 128 KB

Reading sequentially:
    Load 128 KB → Cache full
    Evict old pages → Load next 128 KB
    Evict pages just read! 
    Thrashing!
    
Detection:
    Track readahead hit rate
    Hit rate < 25%? Reduce window! 
    Hit rate > 50%? Increase window! 
```

### Detail 2: Forced Readahead

**Application hints:**

```
posix_fadvise(fd, offset, len, POSIX_FADV_WILLNEED);

Kernel:
    "Application knows future access!"
    Start aggressive readahead NOW
    Don't wait for pattern
    
Use case:
    Video player: "Need next 10 MB"
    Database: "SELECT * FROM table" (sequential scan)
    
By time application needs it: In cache! 
```

---

# FILE 10: mprotect.c

> **Permission enforcer - controls read/write/execute**

## What is mprotect.c?

**Changes memory permissions!**

```
mprotect.c handles:
├── Changing page permissions
├── W^X enforcement (Write XOR Execute)
├── Security policies
├── JIT compilation support
└── TLB invalidation
```

---

## Protection Flags

**Available permissions:**

```
PROT_NONE:  No access (guard pages)
PROT_READ:  Can read
PROT_WRITE: Can write
PROT_EXEC:  Can execute

Combinations:
    Code: PROT_READ | PROT_EXEC
    Data: PROT_READ | PROT_WRITE
    Guard: PROT_NONE
```

---

## W^X Security Policy

**Write XOR Execute:**

> Page can be writable OR executable, NEVER both!

**Why this matters:**

```
Attack scenario:
    1. Exploit buffer overflow
    2. Inject malicious code
    3. Jump to buffer and execute
    
Without W^X:
    Buffer: Writable AND executable 
    Attack succeeds!
    
With W^X:
    Buffer: Writable but NOT executable 
    Jump to buffer: PAGE FAULT! 
    Attack prevented! 
```

---

## Changing Permissions

**Runtime permission changes:**

**Use case: JIT compilation**

```
JavaScript engine generates code:

STEP 1: Allocate writable
    mmap(PROT_READ | PROT_WRITE)
    
STEP 2: Generate machine code
    Write compiled code to memory
    
STEP 3: Make executable
    mprotect(PROT_READ | PROT_EXEC)
    Can't write anymore! 
    
STEP 4: Execute
    Jump to generated code
    Fast execution! 
```

**Why two steps?**

```
W^X enforcement:
    Can't have writable + executable
    Must choose!
    
Generate: Need write
Execute: Need execute
Solution: Switch permissions! 
```

---

## How mprotect Works

**Step-by-step:**

```
Process calls: mprotect(0x40000000, 2MB, PROT_READ|PROT_EXEC)

STEP 1: Find VMA
    Lookup address 0x40000000
    Found: libssl.so mapping
    
STEP 2: Update VMA permissions
    Old: Read + Write
    New: Read + Execute
    
STEP 3: Update page table entries
    Walk all pages in range (512 pages)
    For each entry:
        Old flags: Present, User, Writable
        New flags: Present, User, Executable
    
STEP 4: Flush TLB
    TLB has old permissions cached!
    Must invalidate:
        invlpg instruction (per page)
        Or full TLB flush
    
STEP 5: Return
    Next access uses new permissions! 
```

---

## Important Details

### Detail 1: TLB Flush Requirement

**The caching problem:**

```
Changed page table:
    Old: Writable
    New: Read-only
    
But TLB still has old entry:
    TLB: Writable
    Page table: Read-only
    Inconsistency! 
    
Solution: Flush TLB
    Clear cached entries
    Next access: Reload from page table
    Sees new permissions! 
```

### Detail 2: Multi-CPU TLB Flush

**Global invalidation:**

```
Must flush on ALL CPUs:
    CPU 0 changed page table
    CPU 1-3 have stale TLB entries!
    
Send IPI (Inter-Processor Interrupt):
    Interrupt all CPUs
    Each CPU flushes TLB
    Wait for completion
    
Expensive but necessary! 
```

---

# FILE 11: kmalloc.c

> **Kernel's malloc - small fast allocations**

## What is kmalloc.c?

**Kernel's general-purpose allocator!**

```
kmalloc.c provides:
├── Small object allocation
├── Size class rounding
├── NUMA awareness
├── GFP flag handling
└── Uses SLUB underneath
```

---

## Size Classes

**Power-of-2 rounding:**

```
kmalloc rounds up:
    Request 100 bytes → Get 128 bytes
    Request 250 bytes → Get 256 bytes
    Request 1000 bytes → Get 1024 bytes
    
Standard sizes:
    32, 64, 128, 256, 512, 1024, 2048, 4096...
```

**Why power of 2?**

```
Simplifies allocation:
    Quick size calculation (bit operations)
    Efficient slab layout
    
Reduces fragmentation:
    Standard sizes reused
    Predictable patterns 
```

---

## How kmalloc Works

**Under the hood:**

```
kmalloc(100, GFP_KERNEL):

STEP 1: Round size
    100 → 128 bytes
    
STEP 2: Select cache
    Use SLUB cache for 128-byte objects
    
STEP 3: Allocate from SLUB
    Check per-CPU cache
    Grab object
    Return pointer 
    
Fast! No page allocation!
```

---

## GFP Flags

**Critical allocation flags:**

**GFP_KERNEL:**
```
Normal allocation
Can sleep (wait for memory)
Can trigger reclaim
Used by most kernel code

Example:
    File system allocating buffer
    Can wait if needed 
```

**GFP_ATOMIC:**
```
Cannot sleep!
Must succeed immediately
Uses emergency reserves

Example:
    Network interrupt handler
    Cannot wait! 
    Must get memory now! 
```

**GFP_DMA:**
```
Must allocate from DMA zone
For devices limited to low memory

Example:
    Old floppy driver
    Can only access first 16 MB
```

---

## NUMA Awareness

**kmalloc_node:**

```
NUMA system:
    Node 0: CPU 0-7, RAM 0-8 GB
    Node 1: CPU 8-15, RAM 8-16 GB
    
Local access: 50ns 
Remote access: 150ns 
    
kmalloc_node(size, flags, node):
    Allocate from specific node's memory
    
CPU 0 allocates:
    Automatically uses Node 0
    Local access! Fast! 
```

---

## Important Details

### Detail 1: NULL Safety

**Safe to free NULL:**

```
kfree(NULL);
    Does nothing, returns safely 
    
Common pattern:
    char *ptr = NULL;
    if (condition)
        ptr = kmalloc(100);
    ...
    kfree(ptr); // Safe even if NULL! 
```

### Detail 2: Debug Features

**CONFIG_SLUB_DEBUG:**

```
Red zones:
    Extra bytes around object
    Filled with 0xBB pattern
    Detect buffer overflows! 
    
Poisoning:
    Fill freed objects with 0x6B
    Detect use-after-free! 
    
Tracking:
    Record allocation stack traces
    Debug memory leaks! 
```

---

# FILE 12: oom_kill.c

> **Last resort - when memory exhausted**

## What is oom_kill.c?

**The emergency killer!**

```
oom_kill.c handles:
├── Out-of-memory detection
├── Process scoring (who to kill)
├── Victim selection
├── SIGKILL sending
└── OOM reaper (fast cleanup)
```

---

## When OOM Killer Activates

**The crisis:**

```
System state:
    Free pages: 1,000 (very low!)
    All reclaim failed
    Direct reclaim failed
    No swap space left
    
New allocation request:
    Need: 10,000 pages
    Available: 1,000 pages
    Impossible! 
    
OOM killer:
    "Must kill something to free memory!"
```

---

## Scoring Algorithm

**How to choose victim:**

```
For each process:

Base score = Memory usage (MB)
    
Adjustments:
    Root process: -300 (protect)
    High priority (nice < 0): -100
    Long-running: -50
    Recently started: +100
    User setting (oom_score_adj): -1000 to +1000
    
Final score = Base + adjustments

Highest score = Victim! 
```

---

## Example Scoring

**Real scenario:**

```
Firefox:
    Memory: 2000 MB
    Adjustments: 0
    Final score: 2000
    
Chrome:
    Memory: 3000 MB
    Adjustments: 0
    Final score: 3000 ← Highest! 
    
systemd (PID 1):
    Memory: 10 MB
    Root: -300
    oom_score_adj: -1000
    Final score: 10 - 300 - 1000 = -1290 
    Protected!
    
Database:
    Memory: 1500 MB
    oom_score_adj: -900 (user protected)
    Final score: 1500 - 900 = 600 
    Less likely to die
```

---

## OOM Kill Process

**Step-by-step execution:**

```
STEP 1: Calculate all scores
    Scan all processes
    Compute scores
    
STEP 2: Sort by score
    Highest first
    
STEP 3: Check if killable
    Not PID 1 (init)
    Not kernel thread
    Not already exiting
    
STEP 4: Select highest killable
    Chrome (score 3000)
    
STEP 5: Send SIGKILL
    signal 9 (cannot be caught!)
    Process terminates immediately 
    
STEP 6: Log message
    dmesg: "Out of memory: Killed process 5234 (chrome)"
```

---

## OOM Reaper

**Fast memory reclaim:**

**The problem:**

```
Process killed:
    Still holds memory!
    Must wait for process cleanup
    Cleanup might be slow! 
```

**The solution: OOM Reaper**

```
Kernel thread that:
    1. Detects killed process
    2. Immediately unmaps memory:
        - Walk page tables
        - Free all pages
        - Don't wait for process
    3. Memory available NOW! 
    
Process cleanup happens later
But memory freed immediately!
```

---

## User Control

**Protecting processes:**

```
/proc/[pid]/oom_score_adj:

Protect critical database:
    echo -900 > /proc/$(pidof postgres)/oom_score_adj
    Less likely to be killed! 
    
Sacrifice browser tabs:
    echo 500 > /proc/$(pidof chrome)/oom_score_adj
    More likely to be killed! 
    
Never kill:
    echo -1000 > /proc/$(pidof critical)/oom_score_adj
    Immune to OOM killer! 
```

---

## Important Details

### Detail 1: OOM Killer Message

**What you see in dmesg:**

```
Out of memory: Kill process 5234 (chrome) score 3000 or sacrifice child
Killed process 5234 (chrome) total-vm:3145728kB, anon-rss:2097152kB, file-rss:1048576kB
oom_reaper: reaped process 5234 (chrome), now anon-rss:0kB

Information:
    - Process name (chrome)
    - PID (5234)
    - Score (3000)
    - Memory breakdown:
        total-vm: Total virtual
        anon-rss: Anonymous (malloc)
        file-rss: File-backed
```

### Detail 2: OOM Prevention

**Better than killing:**

```
Monitoring:
    Watch /proc/meminfo
    Free memory dropping? 
    
Prevention:
    Add swap space
    Increase RAM
    Limit process memory (cgroups)
    Tune watermarks
    
OOM killer = Last resort! 
```

---

# THE COMPLETE FIREFOX JOURNEY

> **Now see ALL 12 files working together in ONE continuous flow!**

## The Setup

**Physical reality:**

```
Your Computer:
├── CPU: 4 cores with MMU
├── RAM: 16 GB physical
├── Disk: 512 GB SSD
├── Swap: 4 GB swap file
└── Running: bash shell (PID 1000)

Firefox:
├── Location: /usr/bin/firefox
├── Size: 100 MB
└── Sections: Code (50 MB), Data (10 MB)
```

---

## Phase 1: STARTING FIREFOX

### T = 0.000s: User Double-Clicks Icon

```
Desktop detects click
Launches: /usr/bin/firefox
bash shell (PID 1000) handles it
```

---

### T = 0.001s: Fork - KMALLOC.C + SLUB.C

**bash calls fork():**

```
Kernel needs task_struct (10 KB object)

FILE: kmalloc.c
    kmalloc(sizeof(task_struct), GFP_KERNEL)
    Routes to SLUB allocator
    
FILE: slub.c
    task_struct cache has pre-allocated objects
    Slab 1: [used][used][used][FREE] ← Grab this!
    Return address: 0x02000500
    
Child task_struct allocated! 
Fast! No page allocation needed!
```

---

### T = 0.002s: Create Page Table - PAGE_ALLOC.C

**Kernel needs page table:**

```
4-level page table needs 4 pages:
    PML4: 1 page
    PDPT: 1 page
    PD: 1 page
    PT: 1 page
    
FILE: page_alloc.c
    alloc_pages(GFP_KERNEL, order=0, count=4)
    
    Buddy allocator finds:
        Page 100 → PML4 (physical 0x00064000)
        Page 101 → PDPT (physical 0x00065000)
        Page 102 → PD (physical 0x00066000)
        Page 103 → PT (physical 0x00067000)
    
    Marks pages as USED
    Returns physical addresses 
    
Page tables ready!
```

---

### T = 0.003s: exec() - MMAP.C

**Child calls execve("/usr/bin/firefox"):**

```
Replace bash with Firefox!

FILE: mmap.c
    Creates Virtual Memory Areas (VMAs):
    
    VMA 1: Code section
        Virtual: 0x00400000 - 0x03600000 (50 MB)
        File: /usr/bin/firefox, offset 0
        Permissions: Read + Execute
        Flags: FILE_BACKED, DEMAND_PAGED
        
    VMA 2: Data section
        Virtual: 0x03600000 - 0x04200000 (15 MB)
        File: /usr/bin/firefox, offset 50 MB
        Permissions: Read + Write
        
    VMA 3: Stack
        Virtual: 0x7FFF0000 - 0x80000000 (16 MB)
        File: None (anonymous)
        Permissions: Read + Write
        
IMPORTANT: Don't load pages yet!
    Just create mappings
    Load on-demand 
    
Firefox address space set up!
```

---

### T = 0.004s: First Instruction - MEMORY.C + FILEMAP.C

**CPU jumps to Firefox entry:**

```
RIP = 0x00400000
CR3 = 0x00064000 (PML4 address)

CPU fetches instruction:
    Virtual 0x00400000
    ↓
MMU walks page tables:
    PML4[0] → PDPT → PD → PT
    PT[0]: NOT PRESENT! 
    ↓
PAGE FAULT! CR2 = 0x00400000
```

**FILE: memory.c - Page Fault Handler**

```
STEP 1: Identify fault
    CR2 = 0x00400000
    Check VMAs: Code section (file-backed)
    Valid access! 
    
STEP 2: Determine action
    Type: FILE_BACKED
    Need to load from file
    Call filemap.c!
```

**FILE: filemap.c - Page Cache**

```
STEP 1: Check cache
    Key: (firefox inode, offset 0)
    Cache: MISS! Not loaded yet 
    
STEP 2: Need disk I/O
    Read /usr/bin/firefox, offset 0, 4 KB
    
STEP 3: Allocate physical page
    Call page_alloc.c:
        alloc_pages(GFP_KERNEL, 1)
        Returns: Page 200 (physical 0x000C8000)
        
STEP 4: Read from disk
    Block I/O: 4 KB → physical 0x000C8000
    Disk operation complete (10ms)
    
STEP 5: Add to cache
    Cache[(firefox, 0)] = Page 200
    
STEP 6: Return to memory.c
    Page ready at 0x000C8000! 
```

**Back to memory.c:**

```
STEP 3: Update page table
    PT[0] = Physical 0x000C8 | Present | User | Read-only
    
STEP 4: Return to Firefox
    CPU retries instruction
    MMU translates: 0x00400000 → 0x000C800
    Instruction fetches successfully! 

Firefox's first instruction executed!
```

---

### T = 0.005s: More Code Pages - READAHEAD.C

**More page faults as Firefox runs:**

```
Instruction at 0x00400010: PAGE FAULT!
Instruction at 0x00400020: PAGE FAULT!
Instruction at 0x00400030: PAGE FAULT!

This is slow! Each fault = 10ms disk I/O 
```

**FILE: readahead.c - Predictive Loading**

```
Detects sequential pattern:
    Accessed 0, then 0x1000, then 0x2000...
    Sequential! 
    
Triggers readahead:
    Window: 128 KB (32 pages)
    Start background loading:
        Offset 4 KB, 8 KB, 12 KB... 128 KB
        
For each page:
    1. page_alloc.c: Allocate physical page
    2. Block I/O: Read from file
    3. filemap.c: Add to page cache
    
All done BEFORE Firefox accesses! 

Next fault at 0x00401000:
    filemap.c: Cache HIT! 
    Instant return! No disk I/O!
    
Readahead hides disk latency! 
```

---

## Phase 2: FIREFOX ALLOCATES MEMORY

### T = 0.010s: malloc(1 MB) - MMAP.C + PAGE_ALLOC.C

**Firefox code:**

```
char *buffer = malloc(1024 * 1024); // 1 MB
```

**C library (libc):**

```
Large allocation (> 128 KB):
    Use mmap instead of heap!
    
Calls: mmap(NULL, 1MB, PROT_READ|PROT_WRITE, MAP_ANONYMOUS, ...)
```

**FILE: mmap.c - Anonymous Mapping**

```
STEP 1: Find free virtual space
    Process memory map:
        0x00400000-0x04200000: Code+Data (used)
        0x04200000-0x7FFF0000: FREE! 
        
    Pick address: 0x10000000
    
STEP 2: Create VMA
    Virtual: 0x10000000 - 0x10100000 (1 MB)
    Type: ANONYMOUS (no file)
    Permissions: Read + Write
    Flags: DEMAND_PAGED
    
STEP 3: Don't allocate pages yet!
    Mark page table entries: NOT PRESENT
    
STEP 4: Return address
    Return 0x10000000 to libc
    malloc() returns to Firefox 
    
Firefox has pointer, but no physical memory yet!
```

---

### T = 0.011s: Write to Buffer - MEMORY.C + PAGE_ALLOC.C

**Firefox writes:**

```
buffer[0] = 'A'; // First write

CPU: MOV [0x10000000], 'A'
MMU: Page NOT PRESENT!
PAGE FAULT! CR2 = 0x10000000
```

**FILE: memory.c**

```
STEP 1: Check VMA
    Address 0x10000000: In anonymous VMA 
    Permissions: Read+Write 
    Valid access!
    
STEP 2: This is anonymous page
    No file backing
    Need fresh zero-filled page
```

**FILE: page_alloc.c**

```
alloc_pages_zeroed(GFP_KERNEL, 1):
    
    Find free page: Page 300 (physical 0x0012C000)
    
    Zero the page:
        memset(0x0012C000, 0, 4096)
        Security! Previous data erased! 
        
    Mark as USED
    Return Page 300
```

**Back to memory.c:**

```
STEP 3: Update page table
    PT entry for 0x10000000:
        Physical: 0x0012C
        Flags: Present | User | Writable
        
STEP 4: Return
    CPU retries MOV instruction
    Write succeeds! 
    
Firefox's buffer now has physical memory!
```

---

## Phase 3: FIREFOX LOADS WEBPAGE

### T = 1.000s: User Browses to Website

**Firefox downloads 5 MB of data:**

```
HTML, CSS, JavaScript, Images
Needs memory for all of it!
```

**Same flow as before:**

```
malloc(5 MB):
    mmap.c creates 5 MB anonymous VMA
    Virtual: 0x20000000 - 0x20500000
    
On write:
    Page faults
    page_alloc.c allocates pages (1,280 pages)
    memory.c updates page tables
    
Physical RAM usage:
    Before: 1,999,996 free pages
    After: 1,998,716 free pages
    Still plenty! 
```

---

## Phase 4: MEMORY PRESSURE

### T = 60.000s: System Running Many Programs

**Memory state:**

```
Programs running:
    Firefox: 500 MB
    Chrome: 600 MB
    LibreOffice: 400 MB
    Background processes: many
    
Total RAM: 16 GB
Used: 14 GB
Free: 2 GB 

Getting low!
```

---

### T = 65.000s: User Opens Another Tab

**Firefox needs 100 MB more:**

**FILE: page_alloc.c**

```
alloc_pages request: 25,600 pages (100 MB)

Current free: 100,000 pages (~400 MB)
After allocation: 74,400 pages (~300 MB) 

page_alloc.c:
    "Free pages dropping below low watermark!"
    Wake up kswapd! 
```

**FILE: vmscan.c - kswapd Daemon**

```
kswapd wakes up:
    "Memory pressure detected!"
    "Need to free pages!"
    
STEP 1: Scan LRU lists
    Look at inactive list
    Find pages not recently accessed
    
STEP 2: Classify pages
    Clean file pages: 2,000 pages
    Dirty file pages: 1,000 pages
    Anonymous pages: 2,000 pages
    
STEP 3: Select victims
    Start with clean file pages (easiest)
    Pick 5,000 pages total to evict
```

---

### T = 65.001s: Evicting Pages - FILEMAP.C + SWAP.C

**VICTIM 1: Clean File Page**

```
Page from old LibreOffice document:
    File: /home/user/document.odt
    Offset: 16384
    State: CLEAN (not modified)
```

**FILE: filemap.c**

```
STEP 1: Remove from page cache
    Delete cache[(document.odt, 16384)]
    
STEP 2: Remove from page table
    LibreOffice's PT: Mark NOT PRESENT
    
STEP 3: Free physical page
    page_alloc.c: Page back to free list 
    
No disk write needed! Fast! 

If LibreOffice accesses laterz
    Page fault → Re-read from file 
```

**VICTIM 2: Dirty File Page**

```
Firefox cache file:
    File: ~/.cache/firefox/data
    Offset: 8192
    State: DIRTY (modified, not saved)
```

**FILE: filemap.c - Write-back**

```
Can't just drop it! Would lose data! 

STEP 1: Mark for writeback
    Set PG_writeback flag
    
STEP 2: Write to disk
    Submit I/O: Write 4 KB to file
    Wait for completion (10ms)
    
STEP 3: Mark clean
    Clear PG_dirty
    Set PG_clean
    
STEP 4: Now can evict
    Remove from cache
    Free physical page 
```

**VICTIM 3: Anonymous Page**

```
Chrome memory:
    Virtual: 0x30000000
    Type: Anonymous (malloc'd)
    No file backing!
```

**FILE: swap.c - Swapping Out**

```
Where to save this? 
No file! Must use swap!

STEP 1: Allocate swap slot
    Swap bitmap: Find free slot
    Slot 2500: FREE 
    Mark as USED
    
STEP 2: Write to swap
    Write physical page → /swapfile offset (2500 × 4096)
    Disk write (10-50ms)
    
STEP 3: Update page table
    Chrome's PT entry:
        Before: Virtual 0x30000000 → Physical 1500 (Present=1)
        After: Virtual 0x30000000 → Swap 2500 (Present=0)
        
STEP 4: Free physical page
    page_alloc.c: Page 1500 freed 
    
Physical page reclaimed!
```

**After reclaiming 5,000 pages:**

```
Free pages: 74,400 → 79,400 (~320 MB) 
Breathing room restored!
```

---

## Phase 5: KERNEL NEEDS MEMORY

### T = 70.000s: Network Packet Arrives

**Network card receives packet:**

```
Size: 1500 bytes
Driver needs buffer
```

**FILE: kmalloc.c**

```
Driver: kmalloc(1500, GFP_ATOMIC)

GFP_ATOMIC = Cannot sleep! (in interrupt!)

kmalloc logic:
    Size: 1500 → Round to 2048 bytes
    Use SLUB cache for 2 KB objects
```

**FILE: slub.c**

```
Per-CPU cache (CPU receiving interrupt):
    Check free list
    Object available! 
    Return address: 0x05000000
    
Driver copies packet to buffer
Processes network data
    
After processing:
    kfree(0x05000000)
    Object returns to cache 
    
Fast! No page allocation needed!
```

---

### T = 70.001s: Large Kernel Allocation - VMALLOC.C

**Kernel subsystem needs 10 MB:**

```
Too large for kmalloc!
Can't guarantee physically contiguous!
```

**FILE: vmalloc.c**

```
vmalloc(10 MB):

STEP 1: Allocate virtual space
    Kernel vmalloc area: 0xFFFF880000000000...
    Find free region: 0xFFFF880010000000
    
STEP 2: Allocate physical pages
    10 MB = 2,560 pages
    
    Call page_alloc.c for each:
        Page 1000, Page 3500, Page 8900... (scattered!)
        NOT contiguous in physical RAM!
        
STEP 3: Create mappings
    Virtual 0xFFFF880010000000 → Physical page 1000
    Virtual 0xFFFF880010001000 → Physical page 3500
    Virtual 0xFFFF880010002000 → Physical page 8900
    ...
    
STEP 4: Return virtual address
    Return: 0xFFFF880010000000
    
Kernel sees contiguous memory!
MMU handles scattered pages! 
```

---

## Phase 6: MEMORY PROTECTION

### T = 80.000s: Firefox Loads Shared Library

**Loading libssl.so:**

```
Shared library with code
Must be executable but NOT writable!
```

**FILE: mmap.c - Initial Load**

```
mmap(libssl.so, PROT_READ|PROT_WRITE):
    
    Mapped at: 0x40000000 - 0x40200000 (2 MB)
    Initial: Writable (to load code)
```

**After loading code:**

**FILE: mprotect.c - Change Permissions**

```
Firefox calls:
    mprotect(0x40000000, 2MB, PROT_READ|PROT_EXEC)
    
STEP 1: Find VMA
    Address 0x40000000: libssl.so mapping 
    
STEP 2: Update VMA
    Old permissions: Read + Write
    New permissions: Read + Execute
    
STEP 3: Update page tables
    Walk all 512 pages:
        Old flags: Present | User | Writable
        New flags: Present | User | Executable
        
STEP 4: Flush TLB
    invlpg for each page
    Or full TLB flush
    Clear cached translations!
    
STEP 5: Return
    Next access uses new permissions 
```

**W^X enforcement:**

```
If Firefox tries to WRITE to libssl.so code:
    PAGE FAULT!
    memory.c: "Write to read-only page!"
    Send SIGSEGV
    Firefox crashes 
    
Security enforced! 
```

---

## Phase 7: OUT OF MEMORY CRISIS

### T = 100.000s: System CRITICALLY Low

**Memory state:**

```
Free pages: 5,000 (~20 MB) 
Watermarks:
    min: 2,500 (critical!)
    low: 5,000 (warning!)
    high: 10,000 (safe)
    
Currently AT low watermark!
```

**User opens Blender (needs 500 MB):**

```
Blender: malloc(500 MB)
Needs: 128,000 pages
Available: 5,000 pages

IMPOSSIBLE! 
```

---

**FILE: page_alloc.c - Allocation Fails**

```
alloc_pages request: 128,000 pages

Free pages: 5,000
Request: 128,000

page_alloc.c:
    "Cannot allocate! Emergency!"
    Trigger direct reclaim!
```

**FILE: vmscan.c - Aggressive Reclaim**

```
Direct reclaim (blocking Blender):

Scan ALL pages aggressively:
    Priority 0 (most aggressive)
    Evict everything possible!
    
Reclaimed:
    Clean file pages: 20,000
    Dirty file pages: 15,000 (write first)
    Anonymous pages: 15,000 (swap out)
    
Total freed: 50,000 pages (~200 MB)

Free pages: 5,000 → 55,000 
```

**But STILL not enough for Blender!**

```
Need: 128,000 pages
Have: 55,000 pages
Still short: 73,000 pages! 
```

---

**FILE: oom_kill.c - Last Resort**

```
OOM killer activates:
    "System out of memory!"
    "Must kill something!"
    
STEP 1: Score all processes
    
    Firefox:
        Memory: 500 MB
        Score: 500
        
    Chrome:
        Memory: 600 MB
        Score: 600 ← HIGHEST!
        
    LibreOffice:
        Memory: 400 MB
        Score: 400
        
    systemd (PID 1):
        Memory: 10 MB
        oom_score_adj: -1000
        Score: -990 (protected!)
        
STEP 2: Choose victim
    Highest score: Chrome (600)
    "Chrome uses most memory, kill it!"
    
STEP 3: Send SIGKILL
    kill -9 Chrome
    Chrome terminates immediately! 
    
STEP 4: OOM Reaper frees memory
    Walk Chrome's page tables
    Free all pages immediately
    Don't wait for cleanup
    
    Freed: 600 MB → 153,600 pages! 
    
STEP 5: Memory now available
    Free pages: 55,000 → 208,600 (~830 MB) 
    
STEP 6: Retry Blender allocation
    Now enough memory!
    Blender succeeds! 
```

**System message:**

```
dmesg:
    Out of memory: Killed process 3000 (chrome)
    total-vm:614400kB, anon-rss:524288kB
    oom_reaper: reaped process 3000
```

**User sees:**

```
Chrome window disappears!
Blender opens successfully!
System saved (but Chrome died) 
```

---

# COMPLETE SUMMARY

## What We Covered

**All 12 files working together:**

1. **memory.c** - Handled ALL page faults (demand paging, COW, swap-in)
2. **mmap.c** - Created VM mappings (exec, malloc, libraries)
3. **page_alloc.c** - Allocated physical pages (page tables, anonymous pages)
4. **slub.c** - Fast kernel allocations (task_struct, network buffers)
5. **filemap.c** - Cached files in memory (Firefox executable, documents)
6. **swap.c** - Swapped pages to disk (anonymous pages when RAM full)
7. **vmscan.c** - Found pages to evict (LRU algorithm, kswapd)
8. **vmalloc.c** - Large kernel allocations (scattered physical pages)
9. **readahead.c** - Predicted file access (loaded pages in advance)
10. **mprotect.c** - Changed permissions (W^X security)
11. **kmalloc.c** - Small kernel allocations (network packets)
12. **oom_kill.c** - Last resort (killed Chrome to save system)

---

## The Big Picture

**These 12 mechanisms work together:**

```
User clicks Firefox
    ↓
fork() creates child (kmalloc + SLUB)
    ↓
exec() loads executable (mmap + memory.c)
    ↓
First instruction (memory.c page fault + filemap.c loads from disk)
    ↓
Sequential reads (readahead.c predicts and pre-loads)
    ↓
malloc() allocates memory (mmap creates VMA + page_alloc on write)
    ↓
Memory fills up (vmscan.c + swap.c free pages)
    ↓
Library loading (mprotect.c enforces W^X)
    ↓
Network packets (kmalloc + SLUB fast allocation)
    ↓
Large kernel needs (vmalloc.c scattered allocation)
    ↓
Out of memory (oom_kill.c saves system)
```

**Everything connects! Understanding these fundamentals makes the entire mm/ subsystem comprehensible.**

---

## Design Principles You've Mastered

**The philosophy behind Linux memory management:**

- **Laziness is good** - Demand paging: Don't allocate until actually needed
- **Caching is everything** - Page cache: Keep frequently used data in RAM
- **Fairness matters** - LRU: Give all pages equal chance based on usage
- **Virtual is powerful** - Isolation: Each process sees its own world
- **Physical is precious** - Buddy allocator: Minimize fragmentation
- **Speed needs caching** - TLB & SLUB: Cache translations and objects
- **Security is paramount** - W^X & mprotect: Prevent code injection
- **Overflow gracefully** - Swap & OOM: Degrade before crashing

**Remember:**
> Memory management is not magic - it's clever hardware manipulation with brilliant software design!

---

## The Complete Memory Hierarchy

**Now you understand every layer:**

```
SPEED & SIZE (what you learned):

CPU Registers (RAX, RBX)
    ↓ Fastest, smallest
    
L1/L2/L3 Cache
    ↓ Hardware managed
    
TLB (Translation Cache)
    ↓ FILE: memory.c manages
    ↓ 99% hit rate crucial!
    
Main Memory (RAM)
    ↓ FILE: page_alloc.c manages
    ↓ Buddy allocator, zones, watermarks
    ↓
    ↓ FILE: filemap.c (page cache)
    ↓ FILE: slub.c (object cache)
    ↓ FILE: mmap.c (virtual mappings)
    ↓ FILE: memory.c (page faults)
    
Swap Space
    ↓ FILE: swap.c manages
    ↓ FILE: vmscan.c decides what to swap
    ↓ Disk is slow but extends capacity
    
Disk Storage
    ↓ Slowest, largest
    ↓ FILE: readahead.c optimizes access
```

**You understand how data flows through every layer!**

---

## You're Ready! 

Go forth and:
- Debug mysterious OOM kills
- Optimize memory-intensive applications  
- Understand kernel panics
- Read mm/ patches with confidence
- Contribute to Linux development
- Teach others about memory management

**The journey from confused about NULL guards to understanding Linux memory management is complete!**

> **Welcome to the world of kernel memory management!**
