
# Linux drivers/ Device Subsystems

![Topics](https://img.shields.io/badge/Topics-08-000000?style=for-the-badge&logo=github&logoColor=white&labelColor=grey)
![Focus](https://img.shields.io/badge/Focus-Hardware%20Device%20Drivers-EE4444?style=for-the-badge&logo=linux&logoColor=white&labelColor=grey)
![Status](https://img.shields.io/badge/Status-Complete-13AA52?style=for-the-badge&logo=buffer&logoColor=white&labelColor=grey)
![Level](https://img.shields.io/badge/Level-Hardware%20Integration-1D96E8?style=for-the-badge&logo=opensourceinitiative&logoColor=white&labelColor=grey)

> Ever wondered how pressing a key reaches your application? How NVMe SSDs achieve millions of IOPS? How network packets flow from wire to socket? How USB devices enumerate themselves? How your display initializes at boot?

**You're about to find out!**

---

## Purpose

The `drivers/` directory contains the **hardware device subsystems** that make Linux interact with physical devices. This is where software meets hardware — where device registers are read, DMA transfers occur, interrupts are handled, and bytes flow between peripherals and applications.

> This repository explores the 8 fundamental subsystems that power Linux hardware interaction:

- **How device registration works** (device model, kobjects, uevents, sysfs)
- **How input devices work** (keyboards, mice, event codes, evdev)
- **How storage devices work** (block layer, bio, request queues, NVMe)
- **How network devices work** (net_device, sk_buff, NAPI, packet flow)
- **How USB devices work** (URB, enumeration, classes, hotplug)
- **How PCIe devices work** (BARs, MSI-X, DMA, configuration space)
- **How character devices work** (TTY, PTY, /dev/null, randomness)
- **How graphics devices work** (DRM, KMS, framebuffers, atomic commits)

This isn't high-level overview. It's **register-level deep dive** — DMA rings, interrupt vectors, hardware state machines, and microsecond-by-microsecond data flows.

---

## The Fundamental Problem

**Physical Reality:**

```
Diverse hardware devices everywhere:
├── Input: USB keyboard, touchpad, game controller
├── Storage: NVMe SSD, SATA HDD, USB flash drive
├── Network: Intel NIC, Realtek WiFi, Bluetooth
├── Display: Intel GPU, NVIDIA discrete GPU
├── USB: xHCI controller, hubs, endpoints
└── Thousands of different devices!

But applications need:
├── Simple interfaces (read/write, open/close)
├── Uniform behavior (same API for all keyboards)
├── Abstraction (no hardware details in apps)
└── One way to talk to all devices!

drivers/ is the BRIDGE!
```

---

**Where drivers/ Lives:**

```
linux/
│
├── drivers/                 ← Hardware device drivers (THIS!)
│   ├── base/                ← Core device infrastructure
│   │   ├── core.c           ← Device registration
│   │   ├── bus.c            ← Bus matching
│   │   └── dd.c             ← Driver binding
│   │
│   ├── input/               ← Input devices
│   │   ├── evdev.c          ← Event device interface
│   │   ├── keyboard/        ← Keyboard drivers
│   │   └── mouse/           ← Mouse drivers
│   │
│   ├── block/               ← Block devices
│   │   ├── blk-core.c       ← Block layer core
│   │   ├── blk-mq.c         ← Multi-queue block
│   │   └── elevator.c       ← I/O schedulers
│   │
│   ├── net/                 ← Network devices
│   │   ├── ethernet/        ← Ethernet drivers
│   │   │   ├── intel/       ← Intel NICs
│   │   │   └── realtek/     ← Realtek NICs
│   │   └── wireless/        ← WiFi drivers
│   │
│   ├── usb/                 ← USB subsystem
│   │   ├── core/            ← USB core
│   │   ├── host/            ← USB host controllers
│   │   └── storage/         ← USB storage
│   │
│   ├── pci/                 ← PCIe subsystem
│   │   ├── pci.c            ← PCIe core
│   │   ├── probe.c          ← Device discovery
│   │   └── msi.c            ← MSI-X interrupts
│   │
│   ├── char/                ← Character devices
│   │   ├── mem.c            ← /dev/null, /dev/zero
│   │   ├── random.c         ← /dev/random
│   │   └── tty/             ← TTY subsystem
│   │
│   └── gpu/                 ← Graphics devices
│       └── drm/             ← Direct Rendering Manager
│           ├── drm_drv.c    ← DRM core
│           ├── drm_gem.c    ← Graphics memory
│           └── i915/        ← Intel GPU
│
├── kernel/                  ← Generic kernel
├── mm/                      ← Memory management
└── fs/                      ← File systems

> drivers/ connects hardware to kernel!
> Applications see uniform file/device interfaces!
```

---

**The Problem:**

```
Same operation, different hardware:

Read data from storage:
├── NVMe: PCIe MMIO doorbell, DMA rings, MSI-X completion
├── SATA: AHCI FIS, NCQ commands, legacy interrupts
├── USB: URB submission, bulk transfers, hub routing
└── All DIFFERENT mechanisms!

Receive network packet:
├── Intel NIC: Multi-queue, RSS hashing, hardware offloads
├── Realtek: Single queue, software processing, basic features
├── WiFi: 802.11 frames, encryption, association state
└── Completely different protocols!

Handle input event:
├── USB keyboard: HID reports, endpoint polling, descriptors
├── PS/2 keyboard: Scan codes, i8042 controller, legacy IRQ
├── Touchscreen: Multitouch coordinates, pressure, gestures
└── Different data formats!

drivers/ abstracts ALL hardware differences!
```

---

**Without device drivers:**
- Applications must know every hardware variant
- No code reuse across devices
- Thousands of different interfaces
- Maintenance impossible

---

## The Solution: Layered Device Abstraction

**The Brilliant Design:**

```
Separate WHAT from HOW:

Applications:
├── WHAT: "Read file from disk"
├── WHAT: "Receive network packet"
├── WHAT: "Get keyboard event"
└── Use standard system calls (read/write/ioctl)

Subsystem Layer (block/net/input):
├── Common abstractions (bio, sk_buff, input_event)
├── Standard interfaces (request queues, NAPI, evdev)
├── Hardware-independent logic
└── Calls driver-specific operations

Hardware Driver Layer:
├── HOW on NVMe: PCIe doorbell, DMA, MSI-X
├── HOW on Intel NIC: Multi-queue, RSS, TSO
├── HOW on USB: URB, endpoints, enumeration
└── Implements hardware-specific operations

Result: Application + Subsystem + Driver = Working system!
```

**Example: Keyboard Press**

```
User presses 'A' key
    ↓
drivers/hid/usbhid/hid-core.c (USB HID driver):
    Hardware: USB interrupt endpoint polled
    URB completion receives HID report
    Parse: KEY_A pressed
    ↓
drivers/input/evdev.c (Input subsystem):
    Translate HID to Linux input event
    Event: type=EV_KEY, code=KEY_A, value=1
    Write to /dev/input/event0 ring buffer
    ↓
X server or Wayland (userspace):
    Read from /dev/input/event0
    Deliver to focused application
    ↓
Application receives key event

Entry: Hardware-specific (USB HID driver)
Middle: Generic (input subsystem)
Exit: Standard interface (/dev/input/eventX)
```

---

## Repository Structure

### **[00-foundation/](./00-foundation/README.md)**
> The device model — how Linux discovers, registers, and manages all devices.

**Covers:**
- Device model architecture (kobject, kset, ktype)
- Bus-device-driver matching
- sysfs representation (/sys/class/, /sys/bus/)
- Hotplug and uevents (device insertion/removal)
- Reference counting and lifecycle management
- cdev framework (character device registration)

**Why essential:** This is THE foundation — all other drivers build on this!

---

### **[01-input/](./01-input/README.md)**
> How keypresses and mouse movements reach applications.

**Covers:**
- Input subsystem architecture
- Event codes (KEY_A, REL_X, ABS_Y)
- evdev interface (/dev/input/eventX)
- Complete keyboard flow (USB interrupt → application)
- Input device registration
- Multitouch and absolute coordinates

**Why essential:** Understanding input is essential for interactive systems!

---

### **[02-block/](./02-block/README.md)**
> How storage devices serve read/write requests — the block layer.

**Covers:**
- Block layer architecture (bio, request, gendisk)
- Multi-queue block (blk-mq) design
- I/O schedulers (mq-deadline, BFQ, none)
- Complete NVMe flow (application read → DMA → completion)
- Request queue mechanics
- Performance optimization (merging, batching)

**Why essential:** Block layer is how ALL storage works — SSD, HDD, USB!

---

### **[03-net/](./03-net/README.md)**
> How network packets flow from wire to socket.

**Covers:**
- Network device abstraction (net_device, net_device_ops)
- Packet structure (sk_buff, zero-copy)
- NAPI polling (interrupt to poll-mode transition)
- Complete receive flow (NIC DMA → TCP stack → socket)
- Hardware offloads (checksum, TSO, GRO, RSS)
- Multi-queue networking

**Why essential:** Network drivers power the internet — understanding this is critical!

---

### **[04-usb/](./04-usb/README.md)**
> How USB devices connect and communicate — enumeration to data transfer.

**Covers:**
- USB protocol fundamentals (host/device/hub)
- URB lifecycle (USB Request Block)
- Complete enumeration (plug-in → /dev/sdb)
- USB classes (HID, Mass Storage, CDC)
- Transfer types (control, bulk, interrupt, isochronous)
- Hotplug mechanism

**Why essential:** USB connects keyboards, storage, network — everywhere!

---

### **[05-pci/](./05-pci/README.md)**
> How PCIe devices connect to the CPU — the backbone of modern systems.

**Covers:**
- PCIe topology and speeds (Gen 1-6, x1-x16)
- BDF addressing (Bus:Device.Function)
- Configuration space and BARs
- MSI-X interrupts (memory-write based)
- DMA bus mastering
- Complete NVMe initialization (cold boot → /dev/nvme0n1)

**Why essential:** PCIe is THE interconnect — NVMe, NICs, GPUs all use it!

---

### **[06-char/](./06-char/README.md)**
> Character devices — byte streams from /dev/null to terminals.

**Covers:**
- Character device fundamentals
- Simplest drivers (/dev/null in 4 lines)
- Linux randomness (entropy, ChaCha20 CSPRNG)
- TTY three-layer architecture (core, ldisc, driver)
- PTY master/slave (how SSH works)
- Complete keystroke flow (SSH remote terminal)

**Why essential:** TTY powers every terminal — understanding this unlocks shells and SSH!

---

### **[07-gpu/](./07-gpu/README.md)**
> Graphics and display — from framebuffer to monitor.

**Covers:**
- DRM/KMS architecture
- Display pipeline (Framebuffer → CRTC → Encoder → Connector)
- GEM memory management (allocation, dma-buf sharing)
- Command submission (OpenGL → GPU)
- Atomic modesetting (tear-free display)
- Complete boot flow (BIOS → Plymouth → desktop)

**Why essential:** GPU drivers power your display — final piece of the puzzle!

---

## What Makes drivers/ Special

### **Hardware Abstraction Layers**

```
drivers/ provides multiple abstraction levels:

Application ←→ Subsystem ←→ Driver ←→ Hardware
(File I/O)    (Block layer)  (NVMe)    (Physical SSD)

Example flow — disk read:
Application: read(fd, buf, 4096)
    ↓
VFS layer: vfs_read()
    ↓
File system: ext4_file_read_iter()
    ↓
Block layer (drivers/block/):
    Create bio (block I/O request)
    Submit to request queue
    I/O scheduler merges/reorders
    ↓
NVMe driver (drivers/nvme/host/):
    Convert bio to NVMe command
    Write to submission queue (DMA ring)
    Ring doorbell (PCIe MMIO write)
    ↓
Hardware: NVMe controller executes
    DMA reads command
    DMA writes data to buffer
    MSI-X interrupt signals completion
    ↓
NVMe driver: Process completion queue
    Mark bio complete
    ↓
Block layer: Wake waiting process
    ↓
Application: read() returns with data

Abstraction at every level!
Hardware details hidden from applications!
```

### **DMA and Zero-Copy Design**

```
Modern drivers minimize CPU involvement:

Old approach (programmed I/O):
├── CPU reads from device register
├── CPU writes to memory
├── Repeat for every byte
└── CPU busy entire time

Modern approach (DMA):
├── CPU tells device: "Read from address X, write to Y"
├── Device does everything via PCIe/DMA
├── CPU does other work
├── Device interrupts when done
└── CPU just handles completion

Network packet receive (zero-copy):
└── NIC DMA writes packet to RAM
    sk_buff POINTS to that memory (no copy!)
    Socket buffer POINTS to sk_buff (no copy!)
    Application reads via zero-copy interface
    
    0 CPU copies from wire to application!

Block I/O (zero-copy):
└── NVMe DMA writes directly to page cache
    Application mmap() gets direct access
    No kernel-to-user copy needed
    
Drivers enable zero-copy throughout the stack!
```

---

## How to Use This Guide

**Read sequentially** — Each subsystem builds on previous concepts:
1. Start with **Foundation** (understand device model)
2. Move to **Input** (simplest subsystem, clear data flow)
3. Study **Block** (understand I/O queuing and scheduling)
4. Learn **Network** (understand packet flow and NAPI)
5. Explore **USB** (understand enumeration and transfers)
6. Study **PCIe** (understand how devices connect to CPU)
7. Understand **Character** (terminals and byte streams)
8. Finish with **GPU** (understand display and rendering)

Each subsystem has its own comprehensive README with:
- Physical hardware reality
- Data structure deep-dives
- Step-by-step execution flows
- Complete examples (end-to-end)
- Performance considerations

**Prerequisites:**
- Basic C programming
- Understanding of pointers and structures
- Familiarity with system calls
- Computer architecture fundamentals

---

## Key Concepts You'll Master

### **Device Model Hierarchy**
```
Every device in Linux:

kobject (base object type)
    ↓
device (generic device)
    ↓
Specific device types:
├── pci_dev (PCIe devices)
├── usb_device (USB devices)
├── net_device (Network devices)
├── input_dev (Input devices)
└── gendisk (Block devices)

All integrated into sysfs:
/sys/devices/
/sys/bus/
/sys/class/

Uniform interface for all hardware!
```

### **Interrupt to Application Flow**
```
Hardware event delivery:

Device asserts interrupt (MSI-X write)
    ↓
APIC delivers to CPU
    ↓
Driver interrupt handler runs:
    - Read device status
    - Process completion
    - Schedule softirq/workqueue
    ↓
Softirq/workqueue context:
    - Process received data
    - Call subsystem layer
    ↓
Subsystem delivers to application:
    - Input: write to /dev/input/eventX
    - Network: sk_buff to socket
    - Block: bio completion, wake process
    ↓
Application receives data

Microsecond latencies from hardware to application!
```

### **DMA Ring Buffers**
```
Common pattern across all high-performance drivers:

Ring buffer (circular queue):
    ┌─────────────────────────────────┐
    │ Descriptor 0: addr, len, flags  │ ← Driver writes
    │ Descriptor 1: addr, len, flags  │
    │ Descriptor 2: addr, len, flags  │ ← Device reads
    │ Descriptor 3: addr, len, flags  │
    │ ...                             │
    └─────────────────────────────────┘
         ↑ TAIL (driver)  ↑ HEAD (device)

NVMe submission queue = DMA ring
Network TX ring = DMA ring
USB doesn't use rings (URB-based)

Pattern: Pre-allocate descriptors, DMA addresses
         Device and driver share circular buffer
         Doorbell register signals new work
```

### **Zero-Copy Packet Flow**
```
Network packet from wire to application:

NIC receives packet on wire
    ↓
DMA write to pre-allocated page (RX ring)
    ↓
Driver: sk_buff POINTS to that page (no copy!)
    ↓
Protocol stack: advances skb->data pointer
    - Ethernet header: skip 14 bytes
    - IP header: skip 20 bytes  
    - TCP header: skip 20 bytes
    - Now pointing at application data
    (Still no copy! Just pointer math!)
    ↓
Socket: sk_buff added to receive queue
    ↓
Application: read() copies from sk_buff to user buffer
    (First and ONLY copy: kernel to user)

One copy total (kernel→user mandatory)
vs. old approach: 3-4 copies!
```

---

## The Big Picture

**These subsystems work together:**

```
Complete system boot and operation:

Boot:
drivers/pci/ (discover PCIe devices)
    ↓
drivers/nvme/ + drivers/block/ (storage available)
    ↓
drivers/input/ (keyboard/mouse working)
    ↓
drivers/net/ (network active)
    ↓
drivers/gpu/ (display initialized)
    ↓
Normal operation:
│
├── User types → drivers/input/ → evdev → application
│
├── Network packet arrives → drivers/net/ → socket → application
│
├── Disk read → drivers/block/ + NVMe → page cache → application
│
├── USB device plugged → drivers/usb/ → class driver → /dev node
│
└── Display update → drivers/gpu/ → framebuffer → monitor

All data flows through drivers/!
```

---

## Design Principles You'll Learn

**Layered Abstraction:**
- Subsystem layer (block, net, input)
- Driver implements subsystem operations
- Hardware details hidden from applications

**DMA-First Design:**
- Minimize CPU involvement
- Device-initiated memory transfers
- Completion via interrupts
- CPU only handles control

**Zero-Copy Optimization:**
- Single buffer from hardware to user
- Pointer manipulation instead of copying
- mmap() for direct access where possible

**Multi-Queue Parallelism:**
- One queue per CPU core
- Parallel processing
- Lock-free where possible
- RSS/XPS for distribution

**Remember:**
> drivers/ are not magic — they're careful hardware manipulation with elegant software design!

---

## Context

This repository was built to understand **how Linux device drivers actually work** — not just what drivers do, but HOW hardware interfaces with software, HOW data flows from devices to applications, and HOW the kernel abstracts thousands of different hardware variants into uniform interfaces.

Topics are organized by subsystem with each providing both **hardware-level detail** (DMA rings, register access, interrupt handling) and **architectural clarity** (why it works this way, design patterns).

---

## What's Not Included

This repository does **not** cover:
- Specific device drivers (individual NIC drivers, specific GPU models)
- Platform-specific drivers (ARM SoC peripherals, embedded devices)
- Staging drivers (drivers/ staging/ directory)
- Driver development tools and testing
- Out-of-tree drivers

Focus is on **major device subsystems** in the drivers/ directory.

---

## You're Ready!

With this foundation, you can:
- Understand how any device driver works
- Debug hardware interaction issues
- Write new device drivers
- Optimize I/O performance
- Contribute to Linux driver development
- Navigate the drivers/ codebase confidently

> **The journey from understanding "devices work" to understanding "how drivers make devices work" starts here.**

**For detailed explanations, click into each subsystem's README. Each contains comprehensive coverage with hardware-level details and complete data flows.**

---

### ⭐ Star this repository if you're ready to understand Linux device drivers!

<div align="center">

<img src="https://readme-typing-svg.herokuapp.com?font=Fira+Code&size=18&pause=1000&color=60A5FA&center=true&vCenter=true&width=600&lines=From+keypress+to+application;From+disk+block+to+memory;From+network+packet+to+socket;From+USB+plug+to+device;Understanding+drivers+at+hardware+level" alt="Typing SVG" />

</div>
