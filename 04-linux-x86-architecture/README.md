# Linux arch/x86/ Internals

![Directories](https://img.shields.io/badge/Directories-07-000000?style=for-the-badge&logo=github&logoColor=white&labelColor=grey)
![Focus](https://img.shields.io/badge/Focus-x86%20Architecture-EE4444?style=for-the-badge&logo=intel&logoColor=white&labelColor=grey)
![Status](https://img.shields.io/badge/Status-Working-13AA52?style=for-the-badge&logo=buffer&logoColor=white&labelColor=grey)
![Level](https://img.shields.io/badge/Level-Hardware%20In%20Depth-1D96E8?style=for-the-badge&logo=opensourceinitiative&logoColor=white&labelColor=grey)

> Ever wondered how the SYSCALL instruction actually works? How x86 CPUs handle page faults? Why Linux can run on 30+ different CPU architectures without rewriting everything?

**You're about to find out!**

---

## Purpose

The Linux kernel runs on everything from phones to supercomputers, but each CPU architecture is **fundamentally different**. The `arch/` directory is the bridge between hardware and the generic kernel — where CPU-specific assembly meets portable C code.

> This repository explores **arch/x86/** — the Intel/AMD implementation that powers most desktops and servers:

- **How x86 hardware actually works** (SYSCALL, IDT, APIC, CR3 registers)
- **Why arch/ exists** (CPU differences, abstraction layers, portability)
- **How generic kernel code stays portable** (interfaces, function pointers, abstraction)
- **Physical hardware mechanisms** (Ring 0/3 transitions, interrupt routing, MMU operations)
- **Complete execution flows** (from user program to kernel and back)

This isn't high-level overview. It's **hardware-level deep dive** — register changes, CPU modes, electrical signals, and nanosecond-by-nanosecond execution flows.

---

## The Fundamental Problem

**Physical Reality:**

```
Linux supports 30+ CPU architectures:
├── x86 (Intel, AMD) - Your laptop/desktop
├── ARM (phones, Raspberry Pi, servers)
├── ARM64 (newer phones, Apple M1/M2)
├── PowerPC (old Macs, embedded systems)
├── MIPS (routers, embedded devices)
├── RISC-V (new open architecture)
├── s390 (IBM mainframes)
└── ... 25+ more architectures

Each CPU is COMPLETELY DIFFERENT!
```

---

**Where arch/x86/ Lives in the Kernel:**

```
linux/
│
├── arch/                    ← CPU ARCHITECTURES (33 architectures!)
│   ├── x86/                ← Intel/AMD (what you're using!)
│   │   ├── kernel/         ← x86 core kernel code
│   │   │   ├── entry_64.S  ← System call entry (ASSEMBLY!)
│   │   │   ├── traps.c     ← CPU exceptions (page fault, etc.)
│   │   │   ├── irq.c       ← Interrupt handling
│   │   │   ├── process.c   ← Context switching, fork
│   │   │   ├── signal.c    ← x86 signal handling
│   │   │   ├── cpu/        ← CPU detection, features
│   │   │   ├── apic.c      ← Advanced interrupt controller
│   │   │   └── smp.c       ← Multi-core CPU support
│   │   ├── mm/             ← x86 memory management
│   │   │   ├── fault.c     ← Page fault handler (x86 specific)
│   │   │   ├── init.c      ← Memory initialization
│   │   │   ├── ioremap.c   ← I/O memory mapping
│   │   │   └── pgtable.c   ← Page table operations
│   │   ├── boot/           ← Boot code
│   │   │   ├── compressed/ ← Kernel decompression
│   │   │   └── setup.c     ← Early boot setup
│   │   ├── lib/            ← x86-specific library functions
│   │   └── include/        ← x86 headers
│   ├── arm/                ← ARM (phones, Raspberry Pi)
│   ├── arm64/              ← 64-bit ARM
│   ├── powerpc/            ← PowerPC
│   ├── mips/               ← MIPS
│   └── ... (30+ more!)
│
├── kernel/                  ← GENERIC kernel (works on ALL CPUs)
├── mm/                      ← GENERIC memory management
├── fs/                      ← Filesystems
├── net/                     ← Networking
└── drivers/                 ← Device drivers

> This repository explores arch/x86/ — the Intel/AMD-specific code!
```
---

**The Problem:**

```
x86 CPU:
├── Instructions: MOV RAX, 5 / SYSCALL
├── Registers: RAX, RBX, CR3, RIP
├── System calls: SYSCALL instruction
├── Page tables: 4-level format
└── Interrupts: IDT + APIC

ARM CPU:
├── Instructions: MOV R0, #5 / SVC
├── Registers: R0-R15, PC, CPSR
├── System calls: SVC instruction
├── Page tables: Completely different format
└── Interrupts: Vector table + GIC

NOTHING IS COMPATIBLE!
```
---

**Without arch/:**
- Need 30+ separate kernels (kernel-x86, kernel-arm, kernel-powerpc...)
- Maintenance nightmare
- Bug fixes must be replicated 30+ times
- No code reuse

---

## The Solution: Architecture Abstraction

**The Brilliant Design:**

```
Separate WHAT from HOW:

Generic Kernel (kernel/, mm/, fs/, net/):
├── WHAT to do: "Switch processes"
├── WHAT to do: "Handle page fault"  
├── WHAT to do: "Read system time"
└── Calls INTERFACES (doesn't know CPU details!)

Architecture Code (arch/x86/, arch/arm/):
├── HOW to do it on x86: Use SYSCALL, save RAX/RBX
├── HOW to do it on ARM: Use SVC, save R0-R15
└── Implements SAME INTERFACES, different hardware!

Result: ONE generic kernel runs on ALL CPUs!
```

**Example: Process Context Switch**

```
Generic scheduler (kernel/sched/core.c):
    "Need to switch from Firefox to Chrome"
    Calls: context_switch(prev, next)
    Doesn't know: CPU type, registers, instructions
    ↓
On x86 (arch/x86/kernel/process.c):
    Save x86 registers: RAX, RBX, RCX, RDX...
    Switch page table: Load CR3 register
    Restore x86 registers for next process
    ↓
On ARM (arch/arm/kernel/process.c):
    Save ARM registers: R0-R15
    Switch page table: Load TTBR0 register
    Restore ARM registers for next process

SAME INTERFACE, DIFFERENT IMPLEMENTATION!
Generic kernel works on BOTH architectures!
```

---

## Repository Structure

### **[kernel/](./kernel)**
> The heart of x86 — system calls, exceptions, interrupts, and process management.

**Covers:**
- System call entry (SYSCALL instruction, Ring 3→Ring 0 transition)
- Exception handling (page faults, divide-by-zero, IDT mechanism)
- Interrupt handling (hardware interrupts, APIC, top-half/bottom-half)
- Process context switching (register save/restore, CR3 switching)
- CPU detection and features
- Modern APIC interrupt controller

**Why essential:** This is WHERE kernel execution happens on x86 hardware!

---

### **[mm/](./mm)**
> x86-specific memory management — page faults, page tables, and TLB operations.

**Covers:**
- x86 page fault handling (CR2 register, error codes)
- x86 page table format (4-level paging structure)
- TLB operations (INVLPG instruction, flushing)
- Memory initialization and I/O mapping

**Why essential:** Completes understanding of how x86 MMU actually works!

---

### **[entry/](./entry)**
> Modern system call entry optimizations and security features.

**Covers:**
- Modern entry code architecture
- Performance optimizations
- KPTI (Kernel Page Table Isolation) security
- Entry/exit trampolines

**Why essential:** See how modern kernels optimize for Spectre/Meltdown!

---

### **[boot/](./boot)**
> From power-on to kernel — BIOS/UEFI interaction and early initialization.

**Covers:**
- Very early boot code (before kernel runs!)
- BIOS vs UEFI boot process
- Kernel decompression
- CPU mode transitions (Real → Protected → Long mode)
- Initial page table setup

**Why essential:** Understand what happens BEFORE kernel starts!

---

### **[power/](./power)**
> CPU power management — suspend, resume, and power state transitions.

**Covers:**
- CPU suspend/resume mechanisms
- Complete state save/restore
- ACPI power states
- Hardware power control

**Why essential:** Understand laptop suspend/resume at hardware level!

---

## What Makes arch/x86/ Special

### **Hardware Abstraction Layer**

```
arch/ is the BRIDGE between:

Generic Kernel ←→ arch/x86/ ←→ x86 Hardware
(Portable C)      (CPU-specific)  (Physical CPU)

Example flow — reading a file:
User program: read(fd, buf, size)
    ↓
arch/x86/kernel/entry_64.S (x86 SYSCALL entry)
    Save x86 registers, switch to kernel stack
    ↓
kernel/sys.c (GENERIC system call dispatcher)
    Look up system call table, call sys_read()
    ↓
fs/read_write.c (GENERIC file operations)
    VFS operations, call filesystem
    ↓
mm/filemap.c (GENERIC page cache check)
    IF PAGE FAULT...
    ↓
arch/x86/mm/fault.c (x86 page fault handler)
    Read CR2 register (faulting address)
    x86-specific fault handling
    ↓
mm/memory.c (GENERIC fault resolution)
    Allocate page, update mappings
    ↓
arch/x86/mm/pgtable.c (x86 page table update)
    Update x86 4-level page table
    INVLPG instruction (flush TLB)
    ↓
arch/x86/kernel/entry_64.S (x86 SYSRET exit)
    Restore x86 registers, SYSRET to user
    ↓
User program continues

Flow: Generic ↔ arch-specific ↔ Generic ↔ arch-specific
arch/x86/ handles ALL hardware-specific operations!
```

### **Same Structure Across All Architectures**

```
arch/x86/          arch/arm/          arch/riscv/
├── kernel/        ├── kernel/        ├── kernel/
├── mm/            ├── mm/            ├── mm/
├── boot/          ├── boot/          ├── boot/
├── lib/           ├── lib/           ├── lib/
└── include/       └── include/       └── include/

SAME DIRECTORY STRUCTURE!
Different implementations inside!
Makes kernel maintainable across 30+ architectures!
```

---

## What You'll Learn

Each mechanism is explained with:

- **Physical hardware reality** — What actually happens on the CPU
- **Register-level details** — RIP, RAX, CR3, RFLAGS, and more
- **Step-by-step execution flows** — Nanosecond-by-nanosecond timelines
- **Design trade-offs** — Why x86 works this way
- **Complete examples** — Real scenarios from start to finish

**No code dumps, pure theory!** Build a complete mental model of x86 hardware before studying implementation.

---

## How to Use This Guide

**Read sequentially** — Each topic builds on previous concepts:
1. Start with **kernel/** (understand system calls, exceptions, interrupts)
2. Move to **mm/** (see how x86 memory management works)
3. Continue through remaining directories
4. Refer back while studying actual kernel source code

**Prerequisites:**
- Basic computer architecture (CPU, registers, memory)
- Understanding of C programming
- Familiarity with Linux user-space concepts
- Optional: Assembly language basics (helpful but not required)

---

## Key Concepts You'll Master

### **CPU Privilege Levels (Rings)**
```
Ring 0 (Kernel mode):
├── Can execute privileged instructions
├── Can access all memory
├── Can modify CPU control registers
└── Full hardware control

Ring 3 (User mode):
├── Cannot execute privileged instructions
├── Limited memory access (own process only)
├── Cannot modify control registers
└── Must ask kernel for hardware access

arch/x86/kernel/entry_64.S handles Ring 3 ↔ Ring 0 transitions!
```

### **System Call Mechanism**
```
User program executes: SYSCALL instruction
    ↓ (Hardware automatically!)
CPU switches to Ring 0
CPU reads MSR_LSTAR register (kernel entry point)
CPU jumps to arch/x86/kernel/entry_64.S
    ↓
Kernel validates, processes request
    ↓ (Hardware automatically!)
CPU executes: SYSRET instruction
CPU switches back to Ring 3
User program continues

arch/x86/ implements the entry/exit points!
```

### **Interrupt Routing**
```
Hardware device (network card) needs attention:
Device sends electrical signal to IOAPIC
    ↓
IOAPIC routes interrupt to specific CPU's LAPIC
    ↓
LAPIC signals CPU core
    ↓
CPU looks up handler in IDT (Interrupt Descriptor Table)
    ↓
CPU jumps to arch/x86/kernel/irq.c handler
    ↓
Handler acknowledges interrupt, processes data
    ↓
Returns to interrupted program

arch/x86/ manages IDT and APIC configuration!
```

### **Page Fault Flow**
```
Program accesses memory address 0x20000000:
MMU checks page table entry
    ↓
Entry says: NOT PRESENT!
    ↓
CPU generates Page Fault exception
CPU saves faulting address in CR2 register
    ↓
CPU jumps to arch/x86/mm/fault.c
    ↓
Handler reads CR2, checks if valid access
If valid: Allocate page, update page table, retry
If invalid: Send SIGSEGV signal (segmentation fault)

arch/x86/mm/ handles all x86-specific page fault logic!
```

---

## The Big Picture

**These directories work together:**

```
Power button pressed
    ↓
arch/x86/boot/ (BIOS → bootloader → kernel decompression)
    ↓
arch/x86/kernel/setup.c (CPU detection, initialization)
    ↓
Generic kernel initialization (kernel/init/)
    ↓
System running:
├── User program makes system call
│   └── arch/x86/kernel/entry_64.S handles entry
├── Hardware interrupt arrives  
│   └── arch/x86/kernel/irq.c handles interrupt
├── Page fault occurs
│   └── arch/x86/mm/fault.c handles fault
└── Process switch needed
    └── arch/x86/kernel/process.c performs context switch

arch/x86/ is the FOUNDATION everything runs on!
```

---

## Design Principles You'll Learn

**Abstraction Without Performance Loss:**
- Generic code calls interfaces
- Interfaces compile to direct function calls (no overhead!)
- CPU-specific code uses inline assembly for zero-cost abstractions

**Hardware-Enforced Security:**
- CPU rings enforce privilege separation
- MMU enforces memory isolation  
- IDT/APIC prevent interrupt hijacking
- MSRs protect kernel entry points

**Portability Through Standardization:**
- Same directory structure across all architectures
- Same interface signatures
- Different implementations, same behavior

**Performance Where It Matters:**
- Critical paths in assembly (entry/exit, context switch)
- Bulk operations optimized (memcpy, checksums)
- Everything else in C for maintainability

**Remember:**
> arch/ is not magic — it's hardware manipulation with careful software design!

---

## Context

This repository was built to understand **how Linux actually runs on x86 hardware** — not just what the kernel does, but HOW x86 CPUs enforce isolation, handle interrupts, manage memory, and execute kernel code.

Topics are organized by functional area (kernel operations, memory management, boot, etc.) with each section providing both **technical depth** (register-level details) and **conceptual clarity** (why it works this way).

---

## What's Not Included

This repository does **not** cover:
- Other CPU architectures (ARM, RISC-V, PowerPC) in detail
- Generic kernel subsystems (those are in kernel/, mm/, fs/, etc.)
- Device drivers (those are in drivers/)
- Networking or filesystem internals
- Kernel configuration and build system

Focus is on **x86 architecture-specific code** that makes generic kernel run on Intel/AMD CPUs.

---

## You're Ready!

With this foundation, you can:
- Understand how system calls actually work (hardware level)
- Debug low-level kernel issues (register dumps, page faults)
- Write architecture-aware kernel code
- Contribute to arch/x86/ development
- Port concepts to other architectures

> **The journey from understanding "the kernel" to understanding "how x86 runs the kernel" starts here.**

**For detailed explanations, navigate to individual directories. Each contains a comprehensive README covering its area in complete depth.**

---

### ⭐ Star this repository if you're ready to understand x86 hardware!

<div align="center">

<img src="https://readme-typing-svg.herokuapp.com?font=Fira+Code&size=18&pause=1000&color=60A5FA&center=true&vCenter=true&width=600&lines=From+SYSCALL+to+SYSRET;From+Page+Fault+to+Resolution;From+Hardware+Interrupt+to+Handler;Understanding+x86+at+Register+Level" alt="Typing SVG" />

</div>

