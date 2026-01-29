# Linux arch/x86/kernel/ Internals

![Topics](https://img.shields.io/badge/Topics-06-000000?style=for-the-badge&logo=github&logoColor=white&labelColor=grey)
![Focus](https://img.shields.io/badge/Focus-Core%20Kernel%20Operations-EE4444?style=for-the-badge&logo=intel&logoColor=white&labelColor=grey)
![Status](https://img.shields.io/badge/Status-Complete-13AA52?style=for-the-badge&logo=buffer&logoColor=white&labelColor=grey)
![Level](https://img.shields.io/badge/Level-Hardware%20Deep%20Dive-1D96E8?style=for-the-badge&logo=opensourceinitiative&logoColor=white&labelColor=grey)

> Ever wondered how the SYSCALL instruction actually enters the kernel? How x86 CPUs route interrupts to specific cores? Why context switches require saving dozens of registers? How CPUs detect their own features?

**You're about to find out!**

---

## Purpose

The `arch/x86/kernel/` directory contains the **core x86-specific kernel operations** that make Linux run on Intel and AMD processors. This is where hardware meets software — where CPU instructions, registers, and privilege levels are manipulated to provide system calls, handle exceptions, route interrupts, and switch between processes.

> This repository explores the 6 fundamental mechanisms that power x86 kernel execution:

- **How system calls work** (SYSCALL instruction, Ring 3→0 transition, entry points)
- **How exceptions are handled** (page faults, divide-by-zero, IDT mechanism)
- **How interrupts reach CPUs** (APIC routing, vector tables, acknowledgment)
- **How processes switch** (register save/restore, CR3 switching, FPU state)
- **How CPUs identify themselves** (CPUID instruction, feature detection, bug mitigation)
- **How modern interrupt controllers work** (I/O APIC, Local APIC, IPIs, x2APIC)

This isn't high-level overview. It's **register-level deep dive** — CPU modes, privilege transitions, hardware state changes, and nanosecond-by-nanosecond execution flows.

---

## The Fundamental Problem

**Physical Reality:**

```
Generic kernel code (kernel/, mm/, fs/) doesn't know:
├── How to enter kernel mode (SYSCALL? SVC? TRAP?)
├── How to handle hardware interrupts (APIC? GIC? PIC?)
├── How to switch processes (which registers to save?)
├── How to detect CPU features (CPUID? Device tree?)
└── Hardware-specific details!

But x86 hardware requires:
├── Specific instructions (SYSCALL, SYSRET, IRET)
├── Specific registers (RAX, RBX, CR3, MSRs)
├── Specific privilege levels (Ring 0, Ring 3)
├── Specific interrupt mechanisms (IDT, APIC)
└── Exact hardware protocols!

arch/x86/kernel/ is the BRIDGE!
```

---

**Where arch/x86/kernel/ Lives:**

```
linux/
│
├── arch/                    ← CPU architectures
│   └── x86/                 ← Intel/AMD x86-64
│       ├── kernel/          ← Core x86 kernel code (THIS!)
│       │   ├── entry_64.S   ← System call entry (SYSCALL)
│       │   ├── traps.c      ← Exception handlers (page faults, etc.)
│       │   ├── irq.c        ← Interrupt handling
│       │   ├── process.c    ← Context switching, process management
│       │   ├── cpu/         ← CPU detection and features
│       │   │   ├── common.c ← Generic CPU detection
│       │   │   ├── intel.c  ← Intel-specific quirks
│       │   │   ├── amd.c    ← AMD-specific quirks
│       │   │   └── bugs.c   ← Vulnerability detection
│       │   ├── apic/        ← Advanced interrupt controller
│       │   │   ├── apic.c   ← Local APIC operations
│       │   │   ├── io_apic.c← I/O APIC routing
│       │   │   └── ipi.c    ← Inter-processor interrupts
│       │   ├── signal.c     ← x86 signal delivery
│       │   └── time.c       ← x86 timer setup
│       ├── mm/              ← x86 memory management
│       ├── boot/            ← Boot code
│       ├── entry/           ← Modern entry optimizations
│       └── include/         ← x86 headers
│
├── kernel/                  ← Generic kernel (CPU-independent)
├── mm/                      ← Generic memory management
└── ... other generic subsystems

> arch/x86/kernel/ implements HOW on x86 hardware!
> Generic kernel/ defines WHAT to do!
```

---

**The Problem:**

```
Same operation, different hardware:

Enter kernel mode:
├── x86:     SYSCALL instruction + MSR_LSTAR
├── ARM:     SVC instruction + vector table
├── RISC-V:  ECALL instruction + stvec register
└── All DIFFERENT mechanisms!

Route interrupt to CPU:
├── x86:     IDT + APIC (I/O APIC + Local APIC)
├── ARM:     Vector table + GIC
├── RISC-V:  Interrupt vector + PLIC
└── Completely different hardware!

Switch processes:
├── x86:     Save RAX-R15, switch CR3, restore registers
├── ARM:     Save R0-R15, switch TTBR0, restore registers
├── RISC-V:  Save x0-x31, switch satp, restore registers
└── Different register sets, different page table registers!

arch/x86/kernel/ handles ALL x86-specific operations!
```

---

**Without arch/ separation:**
- Need to rewrite kernel for each CPU architecture
- No code reuse across platforms
- 30+ separate kernel implementations
- Maintenance nightmare

---

## The Solution: Architecture Abstraction

**The Brilliant Design:**

```
Separate WHAT from HOW:

Generic Kernel (kernel/):
├── WHAT: "Handle system call"
├── WHAT: "Switch to next process"
├── WHAT: "Deliver signal to process"
└── Calls architecture interfaces

arch/x86/kernel/:
├── HOW on x86: Use SYSCALL, save RAX/RBX/RCX
├── HOW on x86: Switch CR3, save x86 registers
├── HOW on x86: Set up x86 signal stack frame
└── Implements interfaces for x86 hardware

Result: Generic kernel + x86 implementation = Working system!
```

**Example: System Call**

```
User program: read(fd, buffer, size)
    ↓
arch/x86/kernel/entry_64.S (x86 SYSCALL entry):
    Hardware: SYSCALL instruction executed
    Save x86 registers (RAX, RBX, RCX, etc.)
    Switch to kernel stack
    ↓
kernel/sys.c (GENERIC dispatcher):
    Look up system call number
    Call sys_read() function
    ↓
fs/read_write.c (GENERIC file operations):
    Perform file read
    Return result
    ↓
arch/x86/kernel/entry_64.S (x86 SYSRET exit):
    Restore x86 registers
    Hardware: SYSRET instruction executed
    Back to user mode
    ↓
User program continues

Entry/exit: x86-specific (arch/x86/kernel/)
Middle processing: Generic (kernel/, fs/)
```

---

## Repository Structure

### [entry_64.S/](./entry_64.S.md)
> The gateway from user space to kernel — where SYSCALL happens.

**Covers:**
- SYSCALL instruction mechanism
- Ring 3 → Ring 0 privilege transition
- Register saving and kernel stack setup
- System call table dispatch
- SYSRET return to user space
- Complete syscall flow with register changes

**Why essential:** This is THE entry point — understanding this unlocks everything!

---

### [traps.c/](./traps.c.md)
> How x86 CPUs handle errors — page faults, divide-by-zero, invalid opcodes.

**Covers:**
- What exceptions are (CPU-detected errors)
- IDT (Interrupt Descriptor Table) mechanism
- Exception types: Faults, Traps, Aborts
- Page fault flow (CR2 register, error codes)
- Divide-by-zero and segmentation faults
- Signal delivery from exceptions

**Why essential:** Exceptions are how CPUs report problems — page faults drive virtual memory!

---

### [irq.c/](./irq.c.md)
> How hardware devices get the CPU's attention — keyboard, network, disk, timer.

**Covers:**
- Hardware interrupts vs exceptions
- APIC architecture (I/O APIC + Local APIC)
- Interrupt routing to CPUs
- IDT vector assignment
- Top-half/bottom-half design
- Timer interrupts (scheduler heartbeat)

**Why essential:** Interrupts are how hardware communicates — timer interrupt drives multitasking!

---

### [process.c/](./process.c.md)
> How x86 CPUs switch between programs — the multitasking mechanism.

**Covers:**
- What context switching is (complete state swap)
- When switches happen (timer, I/O wait, sleep)
- What gets saved (16 registers, CR3, FPU, flags)
- Complete switch flow (Firefox → Chrome)
- CR3 switching (memory view change)
- Performance costs (cache pollution, TLB flush)

**Why essential:** Context switching creates the illusion of simultaneous execution!

---

### [cpu/](./cpu/README.md)
> How Linux discovers what CPU it's running on — features, bugs, capabilities.

**Covers:**
- CPUID instruction (hardware query API)
- Vendor detection (Intel, AMD, VIA)
- Feature detection (SSE, AVX, AES, virtualization)
- Cache size discovery
- Core and thread counting
- Bug detection (Spectre, Meltdown, mitigations)

**Why essential:** One kernel must adapt to thousands of different CPUs!

---

### [apic/](./apic/README.md)
> Modern interrupt controller — routing interrupts to multi-core CPUs.

**Covers:**
- Why old PIC was inadequate (16 IRQs, single CPU)
- APIC architecture (I/O APIC + Local APIC per core)
- Interrupt routing and load balancing
- Local APIC timer (per-CPU scheduler tick)
- IPIs (Inter-Processor Interrupts, CPU-to-CPU)
- x2APIC (next generation)

**Why essential:** APIC is how modern multi-core systems distribute interrupt load!

---

## What Makes arch/x86/kernel/ Special

### **Hardware Abstraction Layer**

```
arch/x86/kernel/ is the translator:

Generic Kernel ←→ arch/x86/kernel/ ←→ x86 Hardware
(Portable C)      (CPU-specific)       (Physical CPU)

Example flow — timer interrupt:
Timer chip fires interrupt
    ↓
arch/x86/kernel/irq.c (x86 interrupt entry)
    Hardware: CPU saves state, switches to kernel
    Read interrupt vector from LAPIC
    Acknowledge interrupt (EOI)
    ↓
kernel/timer.c (GENERIC timer handling)
    Update jiffies, check timer expiry
    ↓
kernel/sched/core.c (GENERIC scheduler)
    Current process time slice expired?
    Pick next process to run
    ↓
arch/x86/kernel/process.c (x86 context switch)
    Save current x86 registers
    Switch CR3 (page table pointer)
    Load next process's x86 registers
    ↓
arch/x86/kernel/entry_64.S (x86 return)
    Hardware: IRET instruction
    CPU restores state, switches to user
    ↓
New process resumes execution

arch-specific → generic → generic → arch-specific
Hardware interactions ONLY in arch/x86/kernel/!
```

### **Critical Path Performance**

```
Performance-critical paths in assembly:

entry_64.S:
├── System call entry (SYSCALL)
├── System call exit (SYSRET)
├── Exception entry (IDT handlers)
├── Interrupt entry (IRQ handlers)
└── All in assembly for zero-overhead!

Everything else in C:
├── CPU detection (runs once at boot)
├── APIC configuration (setup phase)
├── Process switching logic (not critical)
└── Maintainability over raw speed

Balance: Performance where it matters, clarity everywhere else
```

---

## How to Use This Guide

**Read sequentially** — Each topic builds on previous concepts:
1. Start with **System Call Entry** (understand kernel entry)
2. Move to **Exception Handling** (understand CPU error reporting)
3. Study **Interrupt Handling** (understand hardware communication)
4. Learn **Context Switching** (understand multitasking)
5. Explore **CPU Detection** (understand hardware discovery)
6. Finish with **APIC** (understand modern interrupt routing)

Each topic has its own comprehensive README with:
- Physical hardware reality
- Register-level details
- Step-by-step execution flows
- Complete examples
- Performance considerations

**Prerequisites:**
- Basic x86 assembly (MOV, CALL, RET)
- Understanding of CPU registers (RAX, RBX, RIP, RSP)
- Familiarity with privilege levels (Ring 0/3)
- Computer architecture fundamentals

---

## Key Concepts You'll Master

### **CPU Privilege Levels**
```
Ring 0 (Kernel mode):
├── Can execute all instructions
├── Can access all memory
├── Can modify control registers (CR0, CR3, CR4)
└── Full hardware control

Ring 3 (User mode):
├── Restricted instruction set
├── Limited memory access (own process only)
├── Cannot modify control registers
└── Must use SYSCALL to request kernel services

arch/x86/kernel/entry_64.S manages Ring 3 ↔ Ring 0 transitions!
```

### **System Call Mechanism**
```
Hardware-enforced entry:

User: SYSCALL instruction
    ↓ (CPU automatically!)
CPU reads MSR_LSTAR (kernel entry address)
CPU switches to Ring 0
CPU jumps to entry_64.S
    ↓
Kernel processes request
    ↓ (CPU automatically!)
Kernel: SYSRET instruction
CPU switches to Ring 3
CPU returns to user code

Hardware guarantees: No user code can fake kernel entry!
```

### **Interrupt Routing**
```
Modern multi-core interrupt distribution:

Network packet arrives
    ↓
Device asserts IRQ 11
    ↓
I/O APIC receives signal
I/O APIC checks routing table: "Send to least busy CPU"
    ↓
I/O APIC sends message to Local APIC on CPU 2
    ↓
Local APIC signals CPU 2 core
    ↓
CPU 2 jumps to IDT vector 43 handler
    ↓
arch/x86/kernel/irq.c processes interrupt
    ↓
Handler acknowledges (EOI to LAPIC)
    ↓
Returns to interrupted program

Load balanced across cores automatically!
```

### **Context Switch Mechanism**
```
Complete process state swap:

Timer interrupt fires
    ↓
arch/x86/kernel/irq.c handles timer
    ↓
kernel/sched/core.c picks next process
    ↓
arch/x86/kernel/process.c context_switch():
    Save Firefox state:
    ├── All 16 general-purpose registers
    ├── RIP (instruction pointer)
    ├── RSP (stack pointer)
    ├── RFLAGS (CPU flags)
    └── FPU/SSE state
    
    Switch memory view:
    └── Load Chrome's CR3 (page table pointer)
    
    Load Chrome state:
    ├── Restore all Chrome's registers
    └── Load Chrome's stack
    ↓
Chrome resumes execution

Firefox frozen in time, Chrome thinks it never stopped!
```

---

## The Big Picture

**These mechanisms work together:**

```
System startup:
arch/x86/kernel/cpu/ (detect CPU features)
    ↓
arch/x86/kernel/apic/ (configure interrupt routing)
    ↓
Normal operation:
│
├── User makes system call
│   └── arch/x86/kernel/entry_64.S (SYSCALL entry)
│
├── Hardware interrupt arrives
│   └── arch/x86/kernel/irq.c (interrupt handling)
│
├── CPU detects error (page fault, divide-by-zero)
│   └── arch/x86/kernel/traps.c (exception handling)
│
└── Timer interrupt triggers process switch
    └── arch/x86/kernel/process.c (context switching)

All kernel execution flows through arch/x86/kernel/!
```

---

## Design Principles You'll Learn

**Hardware-Enforced Security:**
- CPU rings prevent user code from kernel privileges
- MSR registers protect kernel entry points
- IDT protects exception/interrupt handlers
- CR3 isolation prevents cross-process memory access

**Zero-Cost Abstractions:**
- Generic kernel calls compile to direct jumps
- No function pointer overhead in critical paths
- Assembly for hot paths, C for clarity elsewhere

**Portability Through Interfaces:**
- Same function signatures across all architectures
- Generic kernel doesn't know CPU details
- Architecture code doesn't know high-level policy

**Performance Where It Matters:**
- SYSCALL/SYSRET: ~100 nanoseconds
- Context switch: ~2-3 microseconds
- Interrupt handling: ~1-5 microseconds
- Optimized for common case

**Remember:**
> arch/x86/kernel/ is not magic — it's precise hardware manipulation with careful software design!

---

## Context

This repository was built to understand **how Linux kernel operations actually work on x86 hardware** — not just what the kernel does, but HOW x86 CPUs enforce security, route interrupts, switch processes, and execute kernel code at the hardware level.

Topics are organized by mechanism (syscalls, exceptions, interrupts, etc.) with each providing both **technical depth** (register changes, instruction flows) and **conceptual clarity** (why it works this way).

---

## What's Not Included

This repository does **not** cover:
- Generic kernel code (kernel/, mm/, fs/ — those are architecture-independent)
- Other x86 subsystems (mm/, boot/, entry/ — those have their own READMEs)
- Device drivers (drivers/ directory)
- Other CPU architectures (ARM, RISC-V, etc.)
- Kernel configuration and build system

Focus is on **core x86 kernel operations** in arch/x86/kernel/ directory.

---

## You're Ready!

With this foundation, you can:
- Understand how system calls actually enter the kernel
- Debug page faults and exceptions
- Analyze interrupt distribution across cores
- Optimize context switch performance
- Write architecture-aware kernel code
- Contribute to Linux x86 development

> **The journey from understanding "the kernel" to understanding "how x86 executes the kernel" starts here.**

**For detailed explanations, click into each topic's README. Each contains comprehensive coverage with register-level details and complete execution flows.**

---

### ⭐ Star this repository if you're ready to understand x86 kernel execution!

<div align="center">

<img src="https://readme-typing-svg.herokuapp.com?font=Fira+Code&size=18&pause=1000&color=60A5FA&center=true&vCenter=true&width=600&lines=From+SYSCALL+to+kernel;From+exception+to+handler;From+interrupt+to+CPU;From+process+to+process;Understanding+x86+at+hardware+level" alt="Typing SVG" />

</div>
