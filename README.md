# Linux Kernel Development Notes

![Topics](https://img.shields.io/badge/Topics-08-000000?style=for-the-badge&logo=github&logoColor=white&labelColor=grey)
![Focus](https://img.shields.io/badge/Focus-Kernel%20Development-EE4444?style=for-the-badge&logo=target&logoColor=white&labelColor=grey)
![Status](https://img.shields.io/badge/Status-Working-13AA52?style=for-the-badge&logo=buffer&logoColor=white&labelColor=grey)
![License](https://img.shields.io/badge/License-GPL--2.0%20%7C%20CC--BY--SA--4.0-1D96E8?style=for-the-badge&logo=opensourceinitiative&logoColor=white&labelColor=grey)

> A technical deep-dive into kernel internals, device driver architecture, and the development process — built while learning kernel contribution workflows.

---

## Purpose

The Linux kernel is a complex system where hardware, software, and community processes intersect. This repository explores:

- **How the kernel actually works** (not just what it does)
- **Why design decisions were made** (monolithic vs microkernel, blocking I/O, etc.)
- **How hardware enforces isolation** (CPU rings, MMU, page tables)
- **Device driver fundamentals** (character devices, IOCTL, synchronization)
- **Development workflows** (building, testing, debugging, contributing)

This isn't a tutorial or course summary. It's a reference for understanding the **system** — the architecture, the mechanisms, and the reasoning behind kernel engineering.

---

## Repository Structure

### **[01-kernel-fundamentals/](./01-kernel-fundamentals)**
> Foundation concepts: What is a kernel? Why monolithic? How CPU rings enforce privilege separation.

**Covers:**
- Kernel vs Operating System
- CPU privilege rings (Ring 0 vs Ring 3)
- Protected instructions and hardware enforcement
- Monolithic vs Microkernel architecture

---

### **[02-kernel-architecture/](./02-kernel-architecture)**
> Core kernel subsystems: how processes run, how memory gets allocated, how hardware interrupts reach the CPU.

**Covers:**
- Process scheduler (how multitasking actually works)
- fork() and exit() (process lifecycle)
- System calls (user→kernel transitions)
- Hardware interrupts and deferred work

---

### **[03-memory-management/](./03-memory-management)**
> Memory management internals: from page faults to swap, how the kernel manages RAM.

**Covers:**
- Virtual memory and page fault handling
- Physical page allocation (buddy allocator)
- Page cache (why repeated file reads are fast)
- Swapping (what happens when RAM is full)
- kmalloc vs vmalloc (kernel memory allocation)

---

### **[04-linux-x86-architecture/](./04-linux-x86-architecture)** 
> x86-specific implementation: how Intel/AMD CPUs actually do syscalls, interrupts, and memory translation.

**Covers:**
- SYSCALL instruction (how system calls enter kernel)
- CPU exceptions and hardware interrupts
- Context switching (saving/restoring CPU state)
- x86 page tables (4-level translation)
- APIC (routing interrupts to multiple CPUs)
- TLB (caching address translations)


---

### **[05-device-drivers/](./05-device-drivers)** 🚧
> Writing kernel modules: character devices, talking to hardware, and not crashing the kernel.

**Covers:**
- Character device basics (open/read/write/ioctl)
- copy_to_user/copy_from_user (safe data transfer)
- Wait queues (blocking I/O without busy-waiting)
- Spinlocks and mutexes (preventing race conditions)

---

### **[06-filesystem-layer/](./06-filesystem-layer)** 🚧
> How files actually work: from open() to disk blocks, the layers between /dev and ext4.

**Covers:**
- VFS (why Linux supports multiple filesystems)
- Inodes and directory entries
- ext4 on-disk format (where your files live)
- devtmpfs (how /dev/sda appears automatically)

---

### **[07-network-stack/](./07-network-stack)** 🚧
> Network internals: from socket() to the wire, how packets move through the kernel.

**Covers:**
- Socket layer (the interface programs use)
- sk_buff (how kernel represents packets)
- Network device drivers (sending/receiving packets)
- Interrupt handling (NAPI polling)
---

## Context

This repository was built while studying kernel development fundamentals. The focus is on understanding **how the machine works** — the hardware mechanisms, kernel design patterns, and engineering trade-offs that make Linux possible.

Topics are organized by concept area rather than chronologically, with each section providing both **technical depth** (how it works) and **practical context** (why it matters).

---

## Learning Approach

> **Start with foundations, build incrementally:**

The content is designed to be explored sequentially for beginners, or referenced as needed for specific topics. Each README dives deep into a single concept area with clear explanations of both mechanisms and reasoning.

**Philosophy:**
- Understand the "why" before the "how"
- Learn hardware constraints that drive software design
- See the full picture: from CPU instructions to kernel APIs
- Build practical skills through real module development

---

## What's Not Included

This repository does **not** cover:
- Specific kernel subsystems in depth (networking, filesystems, etc.)
- Complete driver implementation tutorials
- Kernel configuration deep-dives
- Architecture-specific details (ARM, x86-64 internals, etc.)

Focus is on **foundational concepts** and **development workflows** that apply across subsystems and architectures.

---

## License

- Documentation released under CC-BY-SA-4.0.  
- Code examples (if any) under GPL-2.0 (kernel module standard).

---

## Author

> Built by **Jill Ravaliya** while learning kernel development workflows and device driver fundamentals.

**Focus Areas:**
- Kernel internals and hardware interfaces
- Device driver architecture
- Systems programming
- Upstream contribution process

**Currently Exploring:**
- Linux kernel module development
- Device driver design patterns
- Memory management internals
- Contributing to mainline kernel

---

## Connect With Me

I'm actively learning and building in the **systems programming** and **kernel development** space.

- **Email:** jillahir9999@gmail.com
- **LinkedIn:** [linkedin.com/in/jill-ravaliya-684a98264](https://linkedin.com/in/jill-ravaliya-684a98264)
- **GitHub:** [github.com/jillravaliya](https://github.com/jillravaliya)

**Open to:**
- Kernel development mentorship
- Systems programming collaboration
- Technical discussions on kernel internals
- Open source contribution guidance

---

### ⭐ Star this repository if you find it helpful for your kernel development journey!

---

> **For detailed explanations, navigate to individual topic folders. Each contains a comprehensive README covering its area.**


<div align="center">

<img src="https://readme-typing-svg.herokuapp.com?font=Fira+Code&size=18&pause=1000&color=60A5FA&center=true&vCenter=true&width=600&lines=Understanding+How+Linux+Works;From+Hardware+to+Kernel;Building+Real+Device+Drivers;Learning+in+Public" alt="Typing SVG" />

</div>
