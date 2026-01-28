# APIC: Modern Interrupt Controller

> Ever wondered how interrupts reach the right CPU in multi-core systems? How the kernel timer works independently on each core? How one CPU can interrupt another CPU? Why old systems had only 16 IRQ lines but modern ones have 256?

**You're about to discover the interrupt routing system!**

---

## What's Inside This Guide?

> We're exploring **arch/x86/kernel/apic/** - The INTERRUPT ROUTER for multi-CPU systems!

Here's what routes interrupts:

```
arch/x86/kernel/
└── apic/                  ← APIC subsystem
    ├── apic.c             ← Local APIC operations
    ├── io_apic.c          ← I/O APIC configuration
    ├── vector.c           ← Interrupt vector management
    └── ipi.c              ← Inter-processor interrupts
```

**This code controls:**
- Routing device interrupts to CPUs
- Per-CPU timer (scheduler heartbeat)
- CPU-to-CPU communication (IPIs)
- Interrupt priorities and masking
- Load balancing interrupts across cores
- Modern interrupt handling (vs old PIC)

---

## What You'll Learn

- **Old PIC problems** - Why 8259 PIC was inadequate
- **APIC architecture** - I/O APIC + Local APIC
- **Interrupt routing** - Device to CPU delivery
- **Complete flow** - Network packet interrupt example
- **Local APIC timer** - Per-CPU timing
- **IPIs** - CPU talking to CPU
- **x2APIC** - Next generation improvements

**Pure routing!** Understand how interrupts find CPUs before writing drivers.

---

# The Old System's Problems

> **Why we needed something better than PIC**

## The 8259 PIC (Programmable Interrupt Controller)

**1970s technology:**

```
8259 PIC chip limitations:
├── Only 16 IRQ lines (IRQ 0-15)
├── Supports ONE CPU only 
├── Fixed priorities
├── Edge-triggered only
└── 1970s design!

IRQ assignments (fixed):
├── IRQ 0:  Timer
├── IRQ 1:  Keyboard
├── IRQ 3:  Serial port
├── IRQ 6:  Floppy disk
├── IRQ 14: Primary IDE
└── ... only 16 total!

Modern systems have 50+ devices! 
```

---

## Problems with Old PIC

### **Problem 1: Not Enough Interrupts**

```
Only 16 IRQ lines available:

Modern PC has:
├── Network card
├── Sound card
├── Multiple USB controllers
├── SATA controllers
├── Graphics card
├── WiFi
├── Bluetooth
└── 30+ more devices!

Must share IRQs:
└── IRQ 11: Network + Sound + USB 

When interrupt arrives:
└── Which device? Must ask all! Slow!
```

### **Problem 2: Single CPU Only**

```
Old PIC connects to ONE CPU:

[Devices] → [PIC] → CPU 0 only!

Modern system: 16 CPU cores
Result: ALL interrupts go to CPU 0! 

CPU 0: Overloaded handling interrupts 
CPU 1-15: Idle, wasted! 

Terrible load balancing!
```

### **Problem 3: Fixed Priorities**

```
PIC hardcoded priorities:
├── IRQ 0 (Timer) = Highest
├── IRQ 1 (Keyboard) = High
└── IRQ 15 (IDE) = Lowest

Cannot change!
What if network more important? Too bad! 
```

---

## The Solution: APIC

**Modern interrupt controller:**

```
APIC advantages:
-  256 interrupt vectors (vs 16!)
-  Supports multiple CPUs (up to 255+)
-  Flexible routing (any interrupt → any CPU)
-  Programmable priorities
-  Level-triggered (can't lose interrupts)
-  Per-CPU timer built-in
-  CPU-to-CPU interrupts (IPIs)
-  Much better for modern systems!
```

---

# APIC Architecture

> **Two components working together**

## The Two Parts

```
APIC = Two separate pieces:

1. I/O APIC (one per system)
   └── On motherboard
       Routes device interrupts

2. Local APIC (one per CPU core)
   └── Inside each CPU
       Delivers to that CPU
```

---

## 1. Local APIC (Inside Each CPU)

**One per CPU core:**

```
4-core system:
├── CPU 0: Has Local APIC 0
├── CPU 1: Has Local APIC 1
├── CPU 2: Has Local APIC 2
└── CPU 3: Has Local APIC 3

Each Local APIC can:
-  Receive interrupts from I/O APIC
-  Has own timer (per-CPU!)
-  Send interrupts to other CPUs (IPIs)
-  Control interrupt priorities
-  Mask/unmask interrupts
-  Deliver to its CPU core

Physical location:
└── Built into CPU silicon! 
```

---

## 2. I/O APIC (On Motherboard)

**Central interrupt router:**

```
One I/O APIC chip on motherboard:

What it does:
-  Receives interrupts from devices (24 inputs)
-  Has routing table (where to send each)
-  Sends to appropriate Local APIC
-  Can distribute load across CPUs
-  Programmable configuration

Physical location:
└── Chip on motherboard 
```

---

## How They Work Together

**Simple flow:**

```
Device (Keyboard) generates interrupt
    ↓
I/O APIC receives signal (IRQ 1)
    ↓
I/O APIC checks routing table:
    "IRQ 1 → Send to CPU 0, Vector 33"
    ↓
I/O APIC sends message to Local APIC 0
    ↓
Local APIC 0 receives message
    ↓
Local APIC 0 signals CPU Core 0
    ↓
CPU 0 handles interrupt (Vector 33)
    ↓
Done! 

Each device interrupt can go to different CPU!
Load balanced! 
```

---

# Complete Interrupt Flow Example

> **Network packet arrival - step by step**

## Initial State

```
System: 4 CPU cores
Network card: Connected to I/O APIC input 11

Current CPU loads:
├── CPU 0: Busy (running Firefox)
├── CPU 1: Busy (running Chrome)
├── CPU 2: Idle (waiting)
└── CPU 3: Busy (running compiler)
```

---

## Step 1: Device Generates Interrupt

```
Network card receives packet:
├── Packet data stored in card buffer
└── Card asserts IRQ 11 line

Physical: Pin goes HIGH (voltage) 
I/O APIC detects voltage change
```

---

## Step 2: I/O APIC Routes Interrupt

```
I/O APIC looks up entry 11 in routing table:

Entry 11 configuration:
├── Vector: 43 (kernel interrupt number)
├── Delivery: Lowest Priority (send to least busy CPU)
├── Destination: All CPUs (choose dynamically)
└── Trigger: Level (reliable)

I/O APIC checks CPU priorities:
├── CPU 0: Priority 5 (busy)
├── CPU 1: Priority 4 (busy)
├── CPU 2: Priority 1 (idle) ← Lowest! 
└── CPU 3: Priority 6 (busy)

Decision: Send to CPU 2! 
```

---

## Step 3: I/O APIC Sends to Local APIC

```
I/O APIC sends message:
├── Destination: Local APIC 2
├── Vector: 43
└── Via APIC bus (hardware communication)

Local APIC 2 receives message 
```

---

## Step 4: Local APIC Delivers to CPU

```
Local APIC 2 checks:
├── Interrupts enabled? Yes 
├── Vector 43 masked? No 
├── Priority acceptable? Yes 
└── Ready to deliver!

Local APIC 2 signals CPU Core 2:
└── Set interrupt pending bit

CPU 2 detects interrupt at next instruction boundary
```

---

## Step 5: CPU Handles Interrupt

```
CPU 2 enters interrupt handler:
├── Save state on kernel stack
├── Switch to Ring 0
├── Jump to handler for Vector 43
└── Handler executes (network code)

Handler tasks:
├── Acknowledge to Local APIC (send EOI)
├── Read packet from network card
├── Process packet data
└── Return from interrupt

IRET: Back to what CPU 2 was doing
```

---

## Step 6: System Continues

```
CPU 2 resumes normal work
Network interrupt handled! 
I/O APIC ready for next interrupt
Local APIC ready for next interrupt

Total time: ~1-5 microseconds! 
```

---

# I/O APIC Configuration

## Routing Table Entries

**Each IRQ line has configuration:**

```
Entry structure (64 bits):

Bits 0-7:   Vector (which interrupt number)
Bits 8-10:  Delivery Mode
            000 = Fixed (specific CPU)
            001 = Lowest Priority (least busy CPU)
            111 = ExtINT (legacy PIC mode)
Bit 11:     Destination Mode
            0 = Physical (use CPU ID)
            1 = Logical (use group)
Bit 15:     Trigger Mode
            0 = Edge-triggered
            1 = Level-triggered
Bit 16:     Mask
            0 = Enabled
            1 = Disabled
Bits 56-63: Destination (which CPU(s))
```

---

## Example Configurations

**Keyboard (IRQ 1):**

```
Entry 1:
├── Vector: 33
├── Delivery: Fixed (always same CPU)
├── Destination: CPU 0
├── Trigger: Edge
└── Enabled 

Result: Keyboard always goes to CPU 0
```

**Network (IRQ 11):**

```
Entry 11:
├── Vector: 43
├── Delivery: Lowest Priority (load balance)
├── Destination: All CPUs
├── Trigger: Level
└── Enabled 

Result: Network goes to least busy CPU! 
```

---

# Local APIC Timer

> **Per-CPU timer - no sharing needed!**

## Why Per-CPU Timers?

**Old system:**

```
ONE timer chip (8254 PIT):
└── Interrupts ONE CPU
    That CPU must tell others
    Complicated! 

New system:
Each CPU has OWN timer! 
├── CPU 0: Local APIC timer
├── CPU 1: Local APIC timer
├── CPU 2: Local APIC timer
└── CPU 3: Local APIC timer

All fire independently!
No coordination needed! 
```

---

## How Local Timer Works

**Configuration:**

```
Each Local APIC has timer registers:

Initial Count: Starting value
Divide Config: Frequency divider
Current Count: Counts down (read-only)
LVT Timer: Configuration (vector, mode)

Operation:
├── Write initial count → Timer starts
├── Counts down to 0
├── Fires interrupt!
└── Can be periodic or one-shot
```

---

## Timer Setup Example

**10ms timer tick:**

```
CPU frequency: 3 GHz
Divide by: 16
Effective rate: 187.5 MHz

For 10ms:
└── Cycles needed: 187.5M × 0.01 = 1,875,000

Configuration:
├── Divide Config = 16
├── Initial Count = 1,875,000
├── LVT Timer = Vector 32, Periodic
└── Timer fires every 10ms! 

This drives the scheduler! 
Each CPU switches processes independently!
```

---

# Inter-Processor Interrupts (IPIs)

> **CPUs talking to each other**

## What Are IPIs?

**One CPU interrupting another:**

```
IPI = Inter-Processor Interrupt
    = CPU sending interrupt to another CPU

Example:
CPU 0 needs CPU 2 to do something
    ↓
CPU 0 sends IPI to CPU 2
    ↓
CPU 2 receives interrupt
    ↓
CPU 2 handles request
    ↓
Done! 

Like tapping someone on the shoulder!
```

---

## Why IPIs Are Needed

### **Use Case 1: TLB Shootdown**

```
Problem:
CPU 0 changes page table
Other CPUs have stale TLB entries! 

Solution:
CPU 0 changes page table
    ↓
CPU 0 sends IPI to CPU 1, 2, 3
    ↓
CPU 1, 2, 3 receive IPI
    ↓
CPU 1, 2, 3 flush their TLBs
    ↓
All CPUs synchronized! 
```

### **Use Case 2: Scheduler Wake-Up**

```
CPU 0: Process becomes TASK_RUNNING
       Process last ran on CPU 2
       CPU 2 currently idle

CPU 0 sends IPI to CPU 2: "Wake up! Work available!"
    ↓
CPU 2 receives IPI
    ↓
CPU 2 runs scheduler
    ↓
CPU 2 picks up waiting process
    ↓
Process running on CPU 2! 
```

### **Use Case 3: System-Wide Function Call**

```
Need all CPUs to update configuration:

CPU 0: smp_call_function_many(all_cpus, update_func)
    ↓
Sends IPI to CPU 1, 2, 3
    ↓
All CPUs receive IPI
    ↓
All CPUs execute update_func()
    ↓
All CPUs send acknowledgment
    ↓
CPU 0 continues
    ↓
All CPUs updated! 
```

---

## How to Send IPI

**Use Local APIC ICR register:**

```
ICR = Interrupt Command Register (2 registers)

ICR High: Destination CPU ID
ICR Low: Vector, delivery mode, etc.

Example: CPU 0 sends IPI to CPU 2

1. Write destination:
   ICR_High = 2 (CPU 2's APIC ID)

2. Write command:
   ICR_Low = Vector 253, Fixed delivery

3. APIC sends automatically! 

CPU 2 receives Vector 253 interrupt!
```

---

# x2APIC: Next Generation

> **Improved APIC for modern systems**

## What Is x2APIC?

**Extended xAPIC:**

```
Improvements:
-  Supports MORE CPUs (up to billions!)
-  MSR-based access (faster!)
-  Better IPI performance
-  Larger APIC IDs (32-bit vs 8-bit)
-  Self-IPI register (optimization)

Old xAPIC:
└── Memory-mapped at 0xFEE00000
    Read/write to memory address

New x2APIC:
└── MSR registers (RDMSR/WRMSR)
    CPU instructions, faster! 
```

---

## x2APIC Benefits

**Performance:**

```
Memory-mapped xAPIC:
├── Access via memory load/store
├── Goes through cache
└── ~10-20 cycles

MSR-based x2APIC:
├── Access via RDMSR/WRMSR
├── Direct CPU register
└── ~5-10 cycles

2× faster! 
```

**Scalability:**

```
Old xAPIC:
└── 8-bit APIC ID = 255 CPUs max

New x2APIC:
└── 32-bit APIC ID = 4 billion CPUs!

Future-proof! 
```

---

## Detection and Enabling

**Check for x2APIC:**

```
CPUID function 1, ECX bit 21:
└── If set: x2APIC available! 

Enable x2APIC:
└── Set bit in IA32_APIC_BASE MSR
    x2APIC mode activated!

Modern CPUs (2010+):
└── Nearly all have x2APIC 
```

---

# Physical Hardware Reality

## What Actually Exists

**Local APIC:**

```
Physical reality:
└── Silicon inside CPU chip
    Part of each core

Contains:
├── Registers (flip-flops)
├── Priority logic
├── Timer counter
└── Routing circuits

Memory-mapped window:
└── CPU maps APIC to 0xFEE00000
    Reads/writes go to APIC hardware!
```

**I/O APIC:**

```
Physical reality:
└── Separate chip on motherboard
    Connected to chipset

Contains:
├── 24 input pins (physical wires)
├── Routing table (registers)
├── Priority logic
└── APIC bus interface

Devices physically connect:
└── IRQ line = actual wire! 
```

---

# Connections to Other Topics

## Interrupts (Topic 3)

```
Topic 3: Explained interrupt mechanism
Topic 6: Explained HOW they're routed!

Complete picture:
Device → I/O APIC → Local APIC → CPU

APIC is the routing hardware! 
```

## Context Switching (Topic 4)

```
Local APIC timer:
├── Fires every 10ms
├── Generates timer interrupt
└── Timer interrupt → Scheduler → Context switch!

APIC timer enables multitasking! 
```

## CPU Detection (Topic 5)

```
CPUID detects APIC:
├── Is APIC present?
└── Is x2APIC supported?

Boot code uses this:
├── If APIC: Use APIC mode 
└── If no APIC: Fall back to PIC 
```

## Memory Management (mm/)

```
TLB shootdown IPIs:
├── Page table changes
├── Send IPI to other CPUs
└── Force TLB flush

APIC IPIs ensure memory consistency! 
```

---

# Complete APIC Summary

## What Is APIC?

```
APIC = Advanced Programmable Interrupt Controller
├── I/O APIC: Routes device interrupts
└── Local APIC: Per-CPU delivery + timer + IPIs
```

## Why Does It Exist?

```
Old PIC problems:
├── Only 16 IRQs 
├── Single CPU only 
└── Fixed priorities 

APIC solutions:
├── 256 vectors 
├── Multi-CPU support 
└── Flexible routing 
```

## How Does It Work?

```
Device interrupt
    ↓
I/O APIC receives
    ↓
Checks routing table
    ↓
Sends to Local APIC
    ↓
Local APIC delivers to CPU
    ↓
Handler runs
    ↓
Acknowledge (EOI)
    ↓
Done!
```

## Key Features

```
-  256 interrupt vectors
-  Multi-CPU routing
-  Per-CPU timer
-  IPIs (CPU-to-CPU)
-  Programmable priorities
-  Load balancing
-  x2APIC (modern systems)
```

## Physical Reality

```
Local APIC: Inside CPU silicon
I/O APIC: Chip on motherboard
APIC bus: Physical wires
Hardware routing! 
```

---

## What You Now Understand

```
-  APIC architecture (I/O + Local)
-  Interrupt routing to CPUs
-  Per-CPU timer mechanism
-  IPIs (CPU-to-CPU communication)
-  x2APIC improvements
-  Physical hardware reality
-  Complete interrupt flow
```

---

## The Big Reveal

**When network packet arrives:**

```
One packet triggers:
├── Device assertion (voltage change)
├── I/O APIC detection
├── Routing table lookup
├── CPU load balancing decision
├── APIC bus message
├── Local APIC reception
├── Priority checking
├── CPU core signaling
├── Interrupt handler execution
├── EOI acknowledgment
└── Perfect delivery to right CPU!

All in microseconds! 

That's the magic of APIC:
└── Intelligent routing
    Load balancing
    Multi-CPU coordination
    Modern interrupt handling! 
```

---

## Your New Superpowers

```
-  Understand interrupt distribution
-  Debug CPU load imbalance
-  Analyze IPI performance
-  Configure interrupt affinity
-  Monitor interrupt statistics
-  Appreciate multi-core design
```

---

## A Final Thought

**Every interrupt you receive:**

```
Keyboard press
Mouse move
Network packet
Disk completion
USB event
```

**Remember:**

```
It's not random
    ↓
APIC routes it intelligently
    ↓
Checks which CPU is least busy
    ↓
Delivers to optimal core
    ↓
Load balanced automatically
    ↓
Multi-core efficiency
    ↓
That's modern interrupt handling! 
```

---

We've completed **ALL 6 topics** of **arch/x86/kernel/**!

```
-  System Call Entry
-  Exception Handling
-  Interrupt Handling
-  Context Switching
-  CPU Detection
-  APIC Controller

arch/x86/kernel/ = 100% MASTERED! 
```

**You now understand the hardware layer of Linux!**
