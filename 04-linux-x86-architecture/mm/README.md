# Linux arch/x86/mm/ Internals

![Topics](https://img.shields.io/badge/Topics-06-000000?style=for-the-badge&logo=github&logoColor=white&labelColor=grey)
![Focus](https://img.shields.io/badge/Focus-x86%20Memory%20Management-EE4444?style=for-the-badge&logo=intel&logoColor=white&labelColor=grey)
![Status](https://img.shields.io/badge/Status-Complete-13AA52?style=for-the-badge&logo=buffer&logoColor=white&labelColor=grey)
![Level](https://img.shields.io/badge/Level-Hardware%20Deep%20Dive-1D96E8?style=for-the-badge&logo=opensourceinitiative&logoColor=white&labelColor=grey)

> Ever wondered how a virtual address becomes a physical one — in silicon? How the kernel boots with no memory manager yet? How a driver actually writes to a GPU register? Why page tables don't eat all your RAM?

**You're about to find out!**

---

## Purpose

The `arch/x86/mm/` directory contains the **core x86-specific memory management code** that makes virtual memory work on Intel and AMD processors. This is where CPU registers, page table entries, cache attributes, and MMU hardware are directly manipulated to provide the illusion of infinite, isolated, per-process memory.

> This repository explores the 6 fundamental mechanisms that power x86 memory management:

- **How page faults are caught** (CR2 register, 5-bit error codes, x86 MMU hardware)
- **How page tables are structured** (4-level hierarchy, 64-bit entries, on-demand allocation)
- **How translations are cached** (TLB hardware, INVLPG, multi-CPU shootdown)
- **How memory is bootstrapped at boot** (E820 map, initial page tables, paging enable)
- **How page properties change at runtime** (W^X security, cache modes, kernel hardening)
- **How drivers talk to hardware** (ioremap, uncacheable/write-combining mappings, PCIe)

This isn't a high-level overview. It's a **register-level deep dive** — MMU circuits, page table bits, TLB flush protocols, and nanosecond-by-nanosecond boot sequences.

---

## The Fundamental Problem

**Physical Reality:**

```
Generic kernel code (mm/, fs/, kernel/) doesn't know:
├── Which register holds the faulting address (CR2? FAR?)
├── How to structure page tables (x86 4-level? ARM? RISC-V?)
├── How to flush stale TLB entries (INVLPG? TLBI?)
├── How to detect physical RAM (E820? Device tree?)
└── How to map device registers into virtual space

But x86 hardware requires:
├── Specific registers (CR2, CR3, CR4, MSRs)
├── Specific page table format (4-level, 64-bit entries)
├── Specific instructions (INVLPG, WBINVD, MOV CR3)
├── Specific cache controls (PAT, PCD, PWT bits)
└── Exact MMU hardware protocols!

arch/x86/mm/ is the BRIDGE!
```

---

**Where arch/x86/mm/ Lives:**

```
linux/
│
├── arch/                    ← CPU architectures
│   └── x86/                 ← Intel/AMD x86-64
│       ├── kernel/          ← Core x86 kernel (syscalls, irq, process)
│       └── mm/              ← x86 memory management (THIS!)
│           ├── fault.c      ← Page fault entry (CR2, error codes)
│           ├── pgtable.c    ← 4-level page table operations
│           ├── tlb.c        ← TLB flush and shootdown
│           ├── init.c       ← Boot memory initialization
│           ├── pageattr.c   ← Runtime page attribute changes
│           └── ioremap.c    ← Device memory mapping
│
├── mm/                      ← Generic memory management (CPU-independent)
├── kernel/                  ← Generic kernel
└── ... other generic subsystems

> arch/x86/mm/ implements HOW memory works on x86 hardware!
> Generic mm/ defines WHAT memory management to do!
```

---

**The Problem:**

```
Same operation, different hardware:

Catch memory fault:
├── x86:    CR2 = fault address, 5-bit error code, exception #14
├── ARM:    FAR register, different error format, different exception
└── RISC-V: Different registers, different encoding entirely!

Walk page tables:
├── x86:    PML4 → PDPT → PD → PT (4 levels, 64-bit entries)
├── ARM:    Different table format, different levels
└── RISC-V: satp register, Sv39/Sv48 formats

Flush stale TLB entries:
├── x86:    INVLPG instruction, IPI via APIC
├── ARM:    TLBI instruction, different IPI mechanism
└── RISC-V: SFENCE.VMA instruction, PLIC-based IPIs

arch/x86/mm/ handles ALL x86-specific memory operations!
```

---

**Without arch/ separation:**
- Every CPU architecture would need a completely different kernel
- No memory management code reuse across platforms
- Maintenance nightmare across 30+ architectures

---

## The Solution: Architecture Abstraction

**The Brilliant Design:**

```
Separate WHAT from HOW:

Generic Kernel (mm/):
├── WHAT: "Handle this page fault"
├── WHAT: "Map virtual to physical"
├── WHAT: "Free this process's memory"
└── Calls architecture interfaces

arch/x86/mm/:
├── HOW on x86: Read CR2, parse error code bits
├── HOW on x86: Walk PML4→PDPT→PD→PT, check 64-bit entry
├── HOW on x86: INVLPG + APIC IPI shootdown
└── Implements interfaces for x86 MMU hardware

Result: Generic mm/ + x86 implementation = Working virtual memory!
```

**Example: Page Fault**

```
Program dereferences NULL pointer (or any bad address):
    ↓
arch/x86/mm/fault.c (x86 MMU raises exception #14):
    Hardware: MMU sets CR2 = fault address
    Hardware: CPU pushes 5-bit error code
    fault.c reads CR2 (x86 MOV RAX, CR2)
    fault.c parses error code (present? write? user? NX?)
    ↓
mm/memory.c (GENERIC fault handler):
    Receives: address + flags (architecture-independent)
    Decides: demand paging? COW? kill process?
    Fixes the fault
    ↓
arch/x86/mm/tlb.c (x86 TLB flush):
    invlpg(fault_address)
    If multi-CPU: send APIC IPI to remote cores
    ↓
Program resumes — never knew about fault!

x86 hardware interface: arch/x86/mm/
Generic fault logic:    mm/memory.c
```

---

## Repository Structure

### **[fault.c/](./fault.c/README.md)**
> The x86 entry point when memory access goes wrong — CR2 register, error codes, MMU hardware.

**Covers:**
- CR2 register — how the MMU hardware sets it automatically
- 5-bit error code format (Present, Write, User, Reserved, Instruction)
- All five fault triggers (unmapped, read-only, kernel page, NX, reserved bits)
- x86 security features: SMAP, SMEP, NX bit enforcement
- Two-layer design: x86 hardware → generic mm/memory.c
- Complete fault flow from MMU circuit to program resumption

**Why essential:** Every page fault — demand paging, COW, segfault — enters the kernel here first!

---

### **[pgtable.c/](./pgtable.c/README.md)**
> How x86 organizes the 4-level translation hierarchy — the foundation of virtual memory.

**Covers:**
- Why flat tables are impossible (32 PB per process!)
- 4-level hierarchy (PML4 → PDPT → PD → PT) with on-demand allocation
- 64-bit entry format — every bit dissected (Present, R/W, U/S, Accessed, Dirty, Global, NX)
- Complete address walk: virtual → physical in 5 memory reads
- Creating and destroying page tables (on demand and on exit)
- Huge pages (2 MB and 1 GB) for performance
- 5-level paging (57-bit, 128 PB address space, future CPUs)

**Why essential:** Page tables ARE virtual memory — understanding them unlocks everything!

---

### **[tlb.c/](./tlb.c/README.md)**
> How x86 makes address translation fast — hardware caching, flush protocols, multi-CPU shootdown.

**Covers:**
- Why TLB exists (without it: 500 cycles per memory access!)
- TLB as a Content-Addressable Memory — parallel search in 1 cycle
- The stale TLB problem — when page table changes must invalidate cache
- INVLPG instruction (flush one page, ~100 cycles)
- MOV CR3 (flush many pages, ~3,000 cycles — and when to choose each)
- Multi-CPU TLB shootdown via APIC IPI — the full coordination protocol
- PCID optimization — context IDs that survive context switches

**Why essential:** TLB is the reason virtual memory doesn't make your programs 20× slower!

---

### **[init.c/](./init.c/README.md)**
> How x86 bootstraps the entire memory system from nothing at boot.

**Covers:**
- The chicken-and-egg boot problem (need paging to run kernel, need kernel to set up paging)
- E820 BIOS memory map — detecting RAM vs reserved regions (APIC, VGA, ROM)
- Initial page tables compiled into the kernel binary — breaking the deadlock
- Identity mapping + high mapping — why both are needed for the critical transition
- Enabling paging and 64-bit Long Mode step-by-step (CR3, CR4, EFER, CR0)
- Direct map creation — all physical RAM accessible via a fixed virtual offset
- Memory zones (ZONE_DMA, ZONE_DMA32, ZONE_NORMAL)
- memblock bootstrap allocator → handoff to buddy allocator
- Complete boot timeline: T=0ms (power-on) → T=20ms (fully operational)

**Why essential:** Nothing in mm/ works until init.c runs — it builds the foundation from scratch!

---

### **[pageattr.c/](./pageattr.c/README.md)**
> How x86 changes page properties on the fly — security hardening and performance optimization.

**Covers:**
- Runtime permission changes: R/W ↔ read-only, Executable ↔ NX
- W^X enforcement — Writable XOR Executable, never both simultaneously
- Cache attribute changes: Write-Back (RAM), Uncacheable (registers), Write-Combining (frame buffers)
- PAT mechanism — 8 cache type combinations from 3 bits
- Kernel hardening — making kernel text read-only after boot
- System call table protection — blocking rootkit attacks
- Huge page splitting — when fine-grained control requires splitting 2 MB into 512 × 4 KB
- Multi-CPU synchronization — TLB shootdown after every attribute change

**Why essential:** W^X security and 25× graphics speedup both live here!

---

### **[ioremap.c/](./ioremap.c/README.md)**
> How x86 maps device registers and memory so drivers can talk to hardware.

**Covers:**
- Why device memory is different from RAM (GPU framebuffers, NIC registers, APIC)
- Why the direct map doesn't cover device memory
- ioremap() — uncacheable mapping for control registers (correct but slow)
- ioremap_wc() — write-combining for frame buffers (25× faster!)
- ioremap_cache() — cacheable, rare special case
- How ioremap allocates from the vmalloc virtual address area
- Page table entries for device memory — physical address + cache disable bits
- Complete PCIe transaction path: CPU write → MMU → memory controller → PCIe → device
- Driver lifecycle: request_mem_region → ioremap → readl/writel → iounmap → release

**Why essential:** Without ioremap, no driver can touch hardware — network, GPU, storage all depend on it!

---

## What Makes arch/x86/mm/ Special

### **Hardware Abstraction Layer**

```
arch/x86/mm/ is the translator:

Generic mm/ ←→ arch/x86/mm/ ←→ x86 MMU Hardware
(Portable C)    (CPU-specific)    (Physical silicon)

Example flow — process writes to unmapped address:

Program: *ptr = value;  (0x00601000 not mapped)
    ↓
x86 MMU hardware:
    CR2 ← 0x00601000 (automatic!)
    Exception #14 raised
    ↓
arch/x86/mm/fault.c:
    Read CR2 (MOV RAX, CR2)
    Parse error code bits
    ↓
mm/memory.c (GENERIC):
    Decide: demand paging → allocate page
    ↓
arch/x86/mm/pgtable.c:
    Build 64-bit PTE: 0x8000000012345007
    Write into 4-level hierarchy
    ↓
arch/x86/mm/tlb.c:
    invlpg(0x00601000)
    (Multi-CPU shootdown if needed)
    ↓
Program resumes — write succeeds!

arch-specific → generic → arch-specific → arch-specific
Hardware interactions ONLY in arch/x86/mm/!
```

### **Critical Path Performance**

```
Performance characteristics:

TLB hit:        ~1 cycle      (the common case!)
Page table walk: ~500 cycles  (TLB miss, ~5% of accesses)
Page fault:      ~10,000 cycles (rare, demand paging)
TLB shootdown:   ~5,000 cycles (multi-CPU sync)
ioremap():       one-time cost (driver init only)

Design principle: optimize ruthlessly for the hot path!
├── TLB exists to avoid page table walks
├── Global bit keeps kernel TLB across context switches
├── PCID avoids full TLB flush on context switch
└── INVLPG over MOV CR3 when only 1 page changes
```

---

## How to Use This Guide

**Read sequentially** — Each topic builds on previous concepts:
1. Start with **Page Tables** (understand the translation structure)
2. Move to **TLB Management** (understand why it's fast)
3. Study **Page Fault Handling** (understand how faults use both)
4. Learn **Memory Initialization** (understand how it all starts)
5. Explore **Page Attributes** (understand runtime security & performance)
6. Finish with **I/O Memory Mapping** (understand hardware driver access)

Each topic has its own comprehensive README with:
- Physical hardware reality
- Register-level and bit-level details
- Step-by-step execution flows
- Complete worked examples
- Performance numbers and trade-offs

**Prerequisites:**
- Basic x86 assembly (MOV, CR3, MSR)
- Understanding of CPU registers (RAX, CR0–CR4, RIP, RSP)
- Familiarity with privilege levels (Ring 0 / Ring 3)
- Computer architecture fundamentals

---

## Key Concepts You'll Master

### **Virtual → Physical Translation**
```
Every user memory access goes through:

CPU issues virtual address: 0x00007F1234567890
    ↓
Check TLB (1 cycle, 95%+ hit rate):
    Hit? → Physical address instantly 
    Miss? → Walk 4 page table levels
    ↓ (on TLB miss)
MMU reads from RAM 4 times:
    PML4[254] → PDPT[72] → PD[162] → PT[359]
    Extract physical: 0x12345000
    ↓
Add page offset: 0x12345000 + 0x890 = 0x12345890
    ↓
Access RAM at physical address 0x12345890 

All transparent to the program!
```

### **The x86 Page Fault Interface**
```
x86 hardware on every bad memory access:

1. MMU writes CR2 = fault address     (hardware)
2. MMU builds 5-bit error code        (hardware)
3. CPU raises exception #14           (hardware)
4. fault.c reads CR2                  (arch/x86/mm/)
5. fault.c parses error bits          (arch/x86/mm/)
6. mm/memory.c makes the decision     (generic mm/)
7. Page tables updated                (arch/x86/mm/)
8. TLB flushed                        (arch/x86/mm/)
9. Program resumes                    (transparent!)

Hardware enforces: No way to fake a page fault!
```

### **Boot Memory Deadlock — Solved**
```
The chicken-and-egg at boot:

Need paging ON to run kernel at 0xFFFFFFFF81000000
Need page tables to turn paging ON
Need memory allocator to build page tables
Need paging ON for memory allocator
    ↻ Circular!

Solution — three-stage bootstrap:

Stage 1: Page tables COMPILED into kernel binary
    └── No allocator needed — tables are just data!

Stage 2: Enable paging with those static tables
    └── Now running at high virtual addresses

Stage 3: Detect RAM (E820), build real tables,
         init zones, hand off to buddy allocator
    └── Normal operation!

From power-on to working memory in ~20ms!
```

### **W^X Security Policy**
```
Core security invariant:

A page is EITHER writable OR executable — never both!

Why: Writable + Executable = code injection attack possible

Enforcement via pageattr.c:
├── set_memory_rw() → automatically sets NX=1
└── set_memory_x()  → automatically clears R/W=1

Module loading sequence (secure):
1. Allocate: Writable + No-Execute (load the code)
2. set_memory_ro() + set_memory_x() 
3. Result: Read-Only + Executable (run the code safely)

Buffer overflow with NX stack:
├── Attacker writes shellcode to stack (writable) 
├── Jumps to stack address
├── CPU checks NX bit on instruction fetch
├── NX=1 on stack! PAGE FAULT! SIGSEGV! 
└── Attack blocked!
```

### **Device Memory Access Flow**
```
Driver wants to write to network card register at 0xF8000000:

/* Driver probe */
regs = ioremap(0xF8000000, 4096);
// Allocates virtual address from vmalloc area
// Creates page table entry with PCD=1, PWT=1 (uncacheable)
// Returns: virtual 0xFFFFC90000000000

writel(CMD_RESET, regs + CMD_REG);
// CPU: virtual 0xFFFFC90000000000
// MMU: → physical 0xF8000000 (uncacheable!)
// Memory controller: "Not RAM! Route to PCIe!"
// PCIe TLP packet transmitted
// Network card receives command
// Hardware resets!

iounmap(regs);
// Removes virtual mapping, frees vmalloc range
// TLB flushed

All hardware communication through ioremap!
```

---

## The Big Picture

**All six mechanisms work together:**

```
System boot:
arch/x86/mm/init.c (bootstrap from nothing)
    ↓
Normal operation:
│
├── Program accesses unmapped memory
│   ├── arch/x86/mm/fault.c    (catch via CR2 + error code)
│   ├── mm/memory.c            (decide: allocate page)
│   └── arch/x86/mm/pgtable.c  (build 4-level PTE)
│       └── arch/x86/mm/tlb.c  (flush stale entry)
│
├── Driver initializes hardware
│   └── arch/x86/mm/ioremap.c  (map device registers)
│
├── Kernel module loads
│   └── arch/x86/mm/pageattr.c (W^X: writable→executable)
│
└── Context switch between processes
    ├── MOV CR3 (switch page table root)
    └── arch/x86/mm/tlb.c (flush user TLB entries)

All x86 memory hardware interactions flow through arch/x86/mm/!
```

---

## Design Principles You'll Learn

**Two-Layer Design:**
- arch/x86/mm/ handles x86 hardware (CR2, INVLPG, PAT bits)
- Generic mm/ handles policy (COW, demand paging, OOM killing)
- Neither layer knows the other's internals

**Security at Hardware Level:**
- MMU enforces U/S bit — user code cannot touch kernel pages
- NX bit prevents executing data pages — stack shellcode blocked
- SMAP/SMEP prevent kernel from accidentally touching user memory
- W^X enforced in page table entries — never writable + executable

**Performance Where It Counts:**
- TLB hit: 1 cycle (99% of accesses)
- INVLPG instead of full flush when possible
- Global bit keeps kernel TLB across context switches
- Write-combining gives 25× speedup for frame buffers

**Portability Through Clear Interfaces:**
- Generic mm/ calls `handle_mm_fault()` — never touches CR2
- arch/x86/mm/ calls `__do_page_fault()` — never makes policy decisions
- Swapping to a new CPU architecture means rewriting arch/x86/mm/, nothing else

**Remember:**
> arch/x86/mm/ is not magic — it's precise manipulation of MMU hardware with careful software design!

---

## Context

This repository was built to understand **how Linux memory management actually works on x86 hardware** — not just what virtual memory does, but HOW x86 MMU silicon enforces security, how page table bits drive cache behavior, how the boot process solves a fundamental circular dependency, and how drivers reach hardware through the virtual address space.

Topics are organized by source file (fault.c, pgtable.c, tlb.c, init.c, pageattr.c, ioremap.c) with each providing both **technical depth** (register changes, bit formats, cycle counts) and **conceptual clarity** (why it works this way).

---

## What's Not Included

This repository does **not** cover:
- Generic memory management (`mm/` — CPU-independent policy like OOM, slab, buddy)
- Other x86 subsystems (`arch/x86/kernel/` — syscalls, IRQ, context switch)
- Device drivers (`drivers/` directory)
- Other CPU architectures (ARM, RISC-V, etc.)
- Kernel build system and configuration

Focus is exclusively on **x86-specific memory management** in `arch/x86/mm/`.

---

## You're Ready!

With this foundation, you can:
- Trace a page fault from CPU exception to program resume
- Read and decode a 64-bit x86 page table entry
- Understand why TLB shootdown is necessary on multi-core systems
- Debug kernel boot failures related to memory initialization
- Write a PCI device driver using ioremap correctly
- Understand how W^X protects against code injection attacks
- Contribute to Linux x86 memory management

> **The journey from understanding "virtual memory" to understanding "how x86 silicon implements it" starts here.**

**For detailed explanations, click into each topic's README. Each contains comprehensive coverage with register-level details and complete execution flows.**

---

### ⭐ Star this repository if you're ready to understand x86 memory at hardware level!

<div align="center">

<img src="https://readme-typing-svg.herokuapp.com?font=Fira+Code&size=18&pause=1000&color=60A5FA&center=true&vCenter=true&width=600&lines=From+virtual+address+to+physical;From+boot+nothing+to+working+memory;From+fault+to+transparent+resume;From+driver+write+to+PCIe+packet;Understanding+x86+MMU+at+silicon+level" alt="Typing SVG" />

</div>
