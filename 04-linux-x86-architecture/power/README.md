# Saving Energy When Idle - Dynamic Power Management

> Ever wondered how your laptop lasts 10 hours on battery? How CPUs save power when idle? What happens when you suspend your system?

**You're about to find out!**

---

## What's This About?

This is **arch/x86/power/** - how x86 manages power consumption dynamically!

Here's where it fits:

```
arch/x86/
├── power/             ← Power management (THIS!)
│   ├── cpu.c          ← CPU state save/restore
│   ├── hibernate.c    ← Hibernation (S4)
│   └── suspend.c      ← Suspend to RAM (S3)
│
└── kernel/
    └── acpi/          ← ACPI power management
```

**This covers:**
- C-states (CPU idle/sleep states)
- P-states (Frequency/voltage scaling)
- Suspend to RAM (S3 sleep)
- Hibernation (S4 - suspend to disk)
- Wake-up and resume
- Device power management

---

# The Fundamental Problem

**Physical Reality:**

```
Desktop computer typical usage:

Active computation: 5% of time 
└── Compiling, gaming, rendering

Waiting for input: 95% of time 
└── Reading, typing, thinking

Without power management:
├── CPU: 100W continuous 
├── Active work: 100W × 5% = 5W useful
├── Idle waste: 100W × 95% = 95W wasted! 
└── Total: 100W consumed, only 5W useful!

Laptop with 50Wh battery:
├── 100W consumption
├── Runtime: 50Wh / 100W = 0.5 hours 
└── 30 minutes! Unusable! 

Desktop electricity cost:
├── 100W × 24 hours × 365 days = 876 kWh/year
├── At $0.10/kWh = $87.60/year
└── Plus cooling costs! 

Data center (10,000 servers):
├── 10,000 × 100W = 1 MW
├── Cost: $876,000/year 
└── Plus massive cooling! 
```

**CPUs waste power when idle!**

---

# Without Power Management

**Imagine no dynamic power control:**

```
Laptop scenario:

CPU always at full power:
├── Clock: 3.0 GHz continuous 
├── Voltage: 1.2V contiuous 
├── Power: 100W 
└── Heat: 100W 

Just browsing web:
├── CPU usage: 5%
├── 95% of cycles wasted! 
└── Power still 100W! 

Battery life:
├── 50Wh battery capacity
├── 100W consumption
├── Runtime: 0.5 hours 
└── Need to carry charger everywhere! 

Heat management:
├── 100W heat generation 
├── Need: Large heatsink
├── Need: Loud fan 
└── Laptop hot and noisy! 


Mobile device:

Phone CPU at 10W (no management):
├── Battery: 15Wh
├── Runtime: 1.5 hours 
└── Unusable! 


Suspend to RAM impossible:

Want to "sleep" laptop:
├── Keep RAM powered (okay)
├── But can't turn off CPU! 
├── Power: Still 100W! 
└── Battery dies in 30 minutes! 

No sleep mode! 
```

**Massive energy waste without dynamic management!**

---

# The Dynamic Power Solution

**Adapt power to workload:**

```
C-states (Idle power):
CPU not working?
├── C1: Halt (stop clock) → 50W 
├── C3: Deep sleep → 10W 
└── C6: Very deep sleep → 1W 

When interrupt arrives:
└── Wake up instantly! 
    Resume work! 


P-states (Active power):
Light workload?
├── Reduce frequency: 3.0 → 2.0 GHz
├── Reduce voltage: 1.2 → 1.0V
└── Power: 100W → 50W 

Performance: 67% (acceptable for light work)
Power: 50W (50% savings!)


Suspend to RAM (S3):
Not using laptop?
├── Save state to RAM 
├── Power off CPU, devices 
├── Keep only RAM powered
└── Power: 1-5W 

Resume:
└── 2-5 seconds to full operation 
    Feels instant! 


Result for laptop browsing:
├── Active (5%): 50W (P-state reduced)
├── Idle (95%): 2W (C-state sleep)
├── Average: (0.05 × 50) + (0.95 × 2) = 4.4W 
└── Battery life: 50Wh / 4.4W = 11 hours! 

22× better! 
```

---

# 1. C-States (CPU Idle States)

## The Sleep Depth Levels

**Different levels of CPU sleep:**

```
Think of human sleep:

C0 = Awake (eyes open) 
├── Executing instructions
├── Clock running: 3.0 GHz 
├── Voltage: 1.2V
└── Power: 100W

C1 = Drowsy (eyes closed) 
├── HLT instruction executed
├── Clock stopped 
├── Voltage: 1.2V (still full)
├── Wake time: < 1 microsecond
└── Power: 50W (50% savings!)

C3 = Light sleep 
├── Clock stopped 
├── Voltage reduced: 0.8V
├── Caches flushed! 
├── Wake time: ~100 microseconds
└── Power: 10W (90% savings!)

C6 = Deep sleep 
├── Clock stopped 
├── Core powered off! 
├── Everything lost (saved first)
├── Wake time: ~1 millisecond
└── Power: 1W (99% savings!)

Deeper = More savings, slower wake! 
```

## C1: Halt State

**The quickest sleep:**

```
CPU executes:
HLT  ; Halt until interrupt

Hardware behavior:
├── Clock distribution: Gated (stopped) 
├── Voltage: Unchanged (1.2V)
├── Caches: Valid 
├── TLB: Valid 
└── Registers: Valid 

Power savings:
├── No clock switching
├── No dynamic power 
└── Only leakage power (~50W)

Wake up:
Interrupt arrives → Resume instantly! 
Time: < 1 microsecond

Use case:
└── Brief pauses (waiting for I/O)
    Very short idle periods
```

## C6: Deep Power Down

**The deepest sleep:**

```
Entering C6:

Step 1: Flush caches
WBINVD  ; Write-back and invalidate
└── All dirty data → RAM 
    Caches empty 

Step 2: Save core state
├── All registers → Save area
├── TLB contents → Lost (okay)
└── Core state preserved 

Step 3: Request C6
MOV EAX, 0x20  ; C6 hint
MOV ECX, 1     ; Enable interrupts
MWAIT          ; Enter C6!

Hardware:
├── Stop clock 
├── Reduce voltage to minimum (0.6V)
├── Power off core! 
└── Power: ~1W 

CPU in deep sleep! 


Waking from C6:

Interrupt arrives (timer, keyboard, etc.)

Hardware:
├── Restore voltage 
├── Power on core 
├── Restore core state 
├── Start clock 
└── Jump to interrupt handler 

Time: ~1 millisecond

Cost of deep sleep:
├── Caches cold (all misses initially) 
├── TLB empty (all misses initially) 
└── Performance recovers gradually 
```

## C-State Selection

**Predicting idle duration:**

```
CPUidle governor (menu governor):

CPU becomes idle:
└── No runnable processes

Predict how long:
├── Look at history: Last 10 idle periods
├── Example: [5ms, 10ms, 15ms, 8ms, 12ms, ...]
├── Average: ~10ms
└── Predict next: ~10ms 

Choose C-state:
If predicted < 10μs: → C1 (fast wake)
If predicted < 100μs: → C1E
If predicted < 1ms: → C3
If predicted > 1ms: → C6 

Predicted 10ms → Enter C6! 

Enter C6:
└── Execute MWAIT instruction
    CPU sleeps in C6 

Interrupt arrives after 12ms:
├── Wake up! 
├── Handle interrupt 
└── Measure: Actual = 12ms

Update predictor:
├── Predicted: 10ms
├── Actual: 12ms
├── Error: 2ms
└── Learn for next time! 

Continuously improving! 
```

---

# 2. P-States (Performance States)

## Frequency and Voltage Scaling

**Different from C-states:**

```
C-states: CPU is IDLE (sleeping) 
P-states: CPU is ACTIVE (working) 

Think of car gears:
P0 = 5th gear (fast, high fuel) 
P1 = 4th gear
P2 = 3rd gear
P3 = 2nd gear
P4 = 1st gear (slow, low fuel) 
```

**P-state levels:**

```
P0 - Maximum Performance:
├── Frequency: 3.0 GHz 
├── Voltage: 1.2V
├── Power: 100W
└── Performance: 100%

P2 - Balanced:
├── Frequency: 2.0 GHz
├── Voltage: 1.0V
├── Power: 50W 
└── Performance: 67%

P4 - Power Saver:
├── Frequency: 800 MHz
├── Voltage: 0.8V
├── Power: 15W 
└── Performance: 27%

Choose based on workload! 
```

## Why P-States Work

**Power scaling:**

```
Power formula:
Power ∝ Voltage² × Frequency

Example calculation:
P0: 1.2V², 3.0 GHz
└── Power = 1.44 × 3.0 = 4.32 (relative)

P2: 1.0V², 2.0 GHz
└── Power = 1.0 × 2.0 = 2.0 (relative)

Savings: (4.32 - 2.0) / 4.32 = 54% 
Performance loss: (3.0 - 2.0) / 3.0 = 33%

Better efficiency!
└── 54% power saved for only 33% performance loss 
```

## P-State Governors

**Ondemand governor (common):**

```
Strategy: Dynamic based on load

Algorithm:
├── Measure CPU load every 10ms
└── Adjust frequency accordingly

Example timeline:
T=0ms: Load 90% → P0 (3.0 GHz) 
T=10ms: Load 95% → Stay P0
T=20ms: Load 20% → P2 (2.0 GHz)
T=30ms: Load 10% → P4 (800 MHz) 
T=40ms: Load 85% → P0 (3.0 GHz) 

Responsive to workload! 
Full power when needed! 
Low power when idle! 
```

**Schedutil governor (modern):**

```
Integrated with scheduler:

Scheduler knows:
├── How much work queued
├── How urgent it is
└── Deadline requirements

Directly sets frequency:
├── Much work → High frequency 
├── Little work → Low frequency 
└── Better than polling! 

Benefits:
├── Faster response 
├── More accurate 
└── Lower overhead 
```

## P-State Transition

**Changing frequency:**

```
Current state: P2 (2.0 GHz, 1.0V)
Governor decides: Need P0! (high load detected)

Step 1: Request change
MOV ECX, 0x199      ; IA32_PERF_CTL MSR
MOV EAX, 0x0000     ; P0 target
WRMSR               ; Write to MSR 

Step 2: CPU adjusts
Hardware PLL (Phase-Locked Loop):
├── Adjust frequency: 2.0 → 3.0 GHz 
├── Stabilize oscillator
└── Time: ~10-100 microseconds

Step 3: Voltage adjustment
Voltage regulator (on motherboard):
├── Increase: 1.0V → 1.2V 
├── PWM control adjustment
└── Time: ~100 microseconds

Step 4: Complete!
├── Frequency: 3.0 GHz 
├── Voltage: 1.2V 
└── Full performance! 

Transition: ~100 microseconds total
Transparent to software! 
```

---

# 3. Suspend to RAM (S3 Sleep)

## Pausing the Entire System

**What it is:**

```
Suspend to RAM (S3):
└── Save system state to RAM
    Power off CPU and devices
    Keep RAM powered 
    
Like: Pausing a video game 
├── Game state saved 
├── Can resume instantly 
└── Uses minimal power 

Power consumption:
├── Normal operation: 50W
└── S3 sleep: 1-5W 
    Just RAM refresh!
```

## Complete S3 Flow

**Entering suspend:**

```
User initiates:
echo mem > /sys/power/state

Kernel (arch/x86/power/):

Step 1: Freeze userspace
├── Send SIGSTOP to all processes
├── Pause everything
└── Only kernel running 

Step 2: Suspend devices
For each device (reverse order):
    device->suspend()
    
Network card:
├── Save MAC address
├── Power down PHY
└── Device suspended 

GPU:
├── Save framebuffer
├── Save registers
├── Power down
└── GPU off 

Disk:
├── Flush write cache
├── Spin down
└── Disk parked 

All devices suspended! 

Step 3: Disable non-boot CPUs
System has 4 CPUs:
├── Migrate all tasks to CPU 0
├── Stop CPUs 1, 2, 3
├── Power down CPUs 1, 2, 3
└── Only CPU 0 running! 

Step 4: Save CPU state
Save CPU 0 to memory:
├── CR0, CR3, CR4 (control registers)
├── GDT, IDT (descriptor tables)
├── General purpose registers (RAX, RBX, ...)
├── Segment registers
├── MSRs (model-specific registers)
└── FPU state

Structure saved to RAM:
struct saved_context {
    u64 cr0, cr3, cr4;
    u64 rax, rbx, rcx, rdx;
    struct desc_ptr gdt, idt;
    ...
};

All saved! 

Step 5: Create wakeup vector
Tell BIOS where to resume:
└── Set ACPI waking vector: 0x1000
    Points to wakeup code 

Step 6: Enter S3
Write to ACPI registers:
OUT 0x404, 0x0D  ; Enter S3

ACPI controller:
├── Power off CPU 
├── Power off chipset
├── Keep RAM powered! 
└── Enter self-refresh mode 

System in S3! 
Power: 1-5W (just RAM)
```

**Resuming from suspend:**

```
User presses power button:

Step 1: BIOS resumes
├── CPU powers on 
├── POST skipped (quick boot)
└── Real mode (16-bit) 

Step 2: Jump to wakeup vector
BIOS reads ACPI vector:
└── JMP 0x1000 (wakeup code)

Step 3: Wakeup code executes
arch/x86/realmode/rm/wakeup.S:

16-bit code (Real mode):
├── Enable A20 line
├── Load temp GDT
└── Enter Protected Mode 

32-bit code (Protected mode):
├── Load proper GDT
├── Enable paging 
└── Enable Long Mode 

64-bit code (Long mode):
└── Jump to kernel resume 

Step 4: Restore CPU state
Restore from saved_context:
├── CR0, CR3, CR4 
├── GDT, IDT 
├── All registers 
├── MSRs 
└── FPU state 

CPU state restored! 

Step 5: Re-enable other CPUs
├── Power on CPUs 1, 2, 3 
├── Initialize them
└── All CPUs running! 

Step 6: Resume devices
For each device (forward order):
    device->resume()

Network:
├── Restore MAC
├── Power up PHY
└── Link up! 

GPU:
├── Restore framebuffer
├── Restore registers
└── Display on! 

Disk:
├── Spin up
└── Ready! 

All devices operational! 

Step 7: Thaw userspace
├── Send SIGCONT to all processes
└── Resume execution 

System fully operational! 
Resume time: 2-5 seconds
Feels instant! 
```

---

# 4. Hibernation (S4)

## Suspend to Disk

**What it is:**

```
Hibernation (S4):
└── Save entire RAM to disk
    Power off everything
    Zero power! 

Like: Saving game and powering off console 
├── Complete save to disk 
├── Zero power consumption 
└── Slower resume (disk I/O) 

vs Suspend to RAM:
S3: Fast, uses power, lost if battery dies
S4: Slow, zero power, safe forever 
```

**Complete hibernation flow:**

```
Entering hibernation:

Step 1: Create snapshot
Calculate what to save:
├── Total RAM: 8 GB
├── Subtract: Free pages, cache
└── Need to save: ~2 GB 

Step 2: Freeze syste
├── Freeze userspace 
├── Suspend devices 
└── Disable non-boot CPUs 

Step 3: Create snapshot
Copy all used RAM:
└── Snapshot buffer: 2 GB 

Step 4: Resume temporarily
Need disk to write!
└── Resume disk controller 

Step 5: Write to swap
Write 2 GB to swap partition:
├── Block by block
├── Progress: ████████ 100%
└── All written! 

Step 6: Power off
ACPI: Enter S4
├── Power off CPU 
├── Power off RAM 
└── Everything off! 

Power: 0W 
Can stay off indefinitely! 


Resuming from hibernation:

Step 1: Normal boot
├── BIOS POST
├── Bootloader (GRUB)
└── Kernel starts

Step 2: Detect hibernation image
Kernel checks swap:
└── Hibernation signature found! 

Step 3: Load image from disk
Read 2 GB from swap:
├── Block by block
├── Progress: ████████ 100%
└── Image in RAM! 

Step 4: Restore memory
Copy snapshot to original locations:
└── Overwrite RAM with saved state 

Step 5: Jump to resume
Jump to saved instruction pointer:
└── Continue where hibernation started! 

Step 6: Resume system
├── Resume devices 
├── Thaw userspace 
└── System operational! 

Resume time: 10-30 seconds (disk I/O)
```

---

# 5. Physical Reality

## Clock Gating Hardware

**C1 halt implementation:**

```
CPU executes HLT:

Clock distribution network:

Before HLT:
├── Clock signal: ████████████ (running)
├── All gates switching
└── Dynamic power consumed

After HLT:
├── Clock gate closes 
├── Clock signal: ____________ (stopped)
├── Gates froze
└── No dynamic power! 

Physical:
└── Transistor gate blocking clock signal 
    Silicon implementation 

Power savings:
└── Only leakage power remains (~50%)
```

## Voltage Scaling

**P-state transition:**

```
P0 → P2 (3.0 GHz → 2.0 GHz):

Voltage regulator (VRM on motherboard):

Current: 1.2V output
Target: 1.0V

PWM (Pulse Width Modulation):
├── Adjust duty cycle
├── Change inductor current
├── Capacitors smooth voltage
└── Output: 1.2V → 1.0V 

Time: ~100 microseconds

CPU voltage changes:
└── Silicon receives lower voltage 
    Less power consumed! 

Power reduction:
Voltage²: (1.2)² → (1.0)² 
└── 1.44 → 1.00 = 31% savings! 
```

## RAM Self-Refresh

**S3 suspend RAM:**

```
DDR4 in S3 sleep:

Normal operation:
├── Memory controller sends refresh commands
├── External timing
└── Power: ~5W per module

Self-refresh mode:
├── Internal refresh controller activated 
├── RAM refreshes itself periodically
├── No external controller needed
└── Power: ~0.5W per module 

90% power savings! 

DRAM cells:
└── Capacitors need refresh (charge leaks)
    Self-refresh prevents data loss 
    
Data retained indefinitely! 
Can stay in S3 for days! 
```

---

# 6. Connections to Other Topics

## Connection to Context Switch

```
TOPIC 4: Context switching
└── Saves/restores process state

TOPIC 14: S3 suspend 
└── Saves/restores SYSTEM state!
    Like context switch for entire system
    
Same concept, bigger scale! 
```

## Connection to Interrupts

```
TOPIC 3: Interrupt handling
└── Wakes CPU from HLT

TOPIC 14: C-states 
└── CPU sleeps in C1-C6
    Interrupt wakes CPU! 
    
Interrupts enable power management! 
```

## Connection to Memory Init

```
TOPIC 10: Memory initialization
└── Sets up page tables

TOPIC 14: S3 resume 
└── Must restore page tables!
    CR3, GDT, IDT restored
    
Uses TOPIC 10 structures! 
```

## Connection to APIC

```
TOPIC 6: APIC
└── Interrupt delivery

TOPIC 14: Power management 
└── APIC timer wakes from C-states
    IPIs coordinate multi-CPU suspend
    
APIC critical for PM! 
```

---

# Summary

## What You've Mastered

**You now understand power management!**

```
- What it is: Dynamic power control
- Why needed: Battery life, cooling, cost
- C-states: Idle sleep (C1 → C6)
- P-states: Frequency/voltage scaling
- S3 suspend: RAM powered, quick resume
- S4 hibernation: Disk save, zero power
- Physical reality: Clock gating, voltage scaling
- Connections: Context switch, interrupts, memory, APIC
```

---

## Key Takeaways

**Power savings are massive:**

```
Without management:
└── 100W continuous, 30 min battery 

With management (laptop browsing):
├── Active 5%: 50W (P-state)
├── Idle 95%: 2W (C-state)
├── Average: 4.4W 
└── Battery: 11 hours! 

22× improvement! 
```

**Different states for different needs:**

```
C-states: CPU idle (sleeping) 
└── C1 → C6: Deeper sleep, more savings
    Wake time increases
    
P-states: CPU active (working) 
└── P0 → P4: Lower frequency, less power
    Performance decreases

S3: System suspended (quick resume)
└── 1-5W, 2-5 second wake

S4: System hibernated (zero power)
└── 0W, 10-30 second wake
```

**Automatic and transparent:**

```
CPUidle governor:
└── Predicts idle duration
    Chooses C-state
    Learns and improves 

P-state governor:
└── Monitors load
    Adjusts frequency
    Full power when needed 

User experience:
└── Transparent! 
    Full performance when needed
    Long battery when idle
```

**Resume preserves everything:**

```
S3 resume:
├── All RAM preserved 
├── All processes continue 
├── Network connections maintained 
└── Feels instant! 

S4 resume:
├── Complete state from disk 
├── Everything restored 
└── Safe even if unplugged! 
```

---

## The Big Picture

**Complete power management:**

```
Active high load:
└── P0 (3.0 GHz, 1.2V) → 100W 

Active light load:
└── P2 (2.0 GHz, 1.0V) → 50W 

Brief idle:
└── C1 (halt) → 50W 

Longer idle:
└── C6 (deep sleep) → 1W 

Suspend to RAM:
└── S3 → 2W, quick wake 

Hibernation:
└── S4 → 0W, safe forever 

Adapted to every scenario! 
```

**Typical laptop day:**

```
08:00: Wake from S3 (2 seconds)
08:00-12:00: Active work (P0/P2 + C-states)
12:00: Suspend to RAM (S3)
13:00: Wake from S3 (2 seconds)
13:00-17:00: Active work
17:00: Suspend to RAM (S3)
Next day: Resume or hibernate

Average power: ~5W
Battery life: 10+ hours 
```

---

## You're Ready!

With this knowledge, you can:
- Understand battery life variations
- Debug suspend/resume issues
- Optimize power consumption
- Choose appropriate power policies
- Understand laptop/server differences

> **Power management makes mobile computing possible - you now know how x86 saves energy when idle!**

---

**Energy efficiency through intelligence!** 
