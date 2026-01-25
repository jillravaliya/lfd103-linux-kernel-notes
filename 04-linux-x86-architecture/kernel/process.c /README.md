# Context Switching: The Multitasking Illusion

> Ever wondered how your computer runs hundreds of programs with only 4 CPU cores? How Firefox, Chrome, and your music player all seem to run simultaneously? How the CPU remembers exactly where each program was when it switches back?

**You're about to discover the magic trick behind multitasking!**

---

## What's Inside This Guide?

> We're exploring **arch/x86/kernel/process.c** - The PROGRAM SWITCHER that makes multitasking possible!

Here's what handles program switching:

```
arch/x86/
└── kernel/                ← CPU-specific kernel code
    ├── entry_*.S          ← System call entry
    ├── traps.c            ← Exception handlers
    ├── irq.c              ← Hardware interrupts
    ├── process.c          ← Context switching (THIS!)
    ├── signal.c           ← Signal handling setup
    └── time.c             ← Timer setup
```

**This code controls:**
- Saving complete program state (all registers)
- Switching between program contexts
- Loading new program state
- Memory view switching (CR3 register)
- FPU state preservation
- Stack switching (user and kernel)
- The illusion that programs never stop
- Making 200 programs share 4 CPUs

---

## What You'll Learn

Each switching mechanism is explained with:

- **What is context** - Complete program state
- **When switches happen** - Timer, I/O, sleep, termination
- **What gets saved** - Every register, stack, memory view
- **Complete switch flow** - Firefox to Chrome, step-by-step
- **Physical transformations** - What actually changes in CPU
- **Performance costs** - Cache pollution, TLB flushing
- **Optimization tricks** - CPU affinity, lazy FPU

**Pure mechanism!** Understand the multitasking magic before optimizing it.

---

# The Multitasking Problem

> **How 200 programs share 4 CPU cores**

## The Fundamental Challenge

**Physical Reality:**

```
Your Computer Right Now:
├── CPU Cores: 4 (4 execution units)
├── Running Programs: 200+ processes
└── Problem: 200 programs, 4 cores!

Question: How do all programs run "simultaneously"?
```

---

## Without Context Switching

**Single-tasking nightmare:**

```
System boots:
├── Load Program A (Firefox)
└── Firefox starts executing...

Firefox runs FOREVER!
├── No way to run Chrome
├── No way to run Terminal
├── No way to run anything else
└── System stuck on Firefox! 

Like MS-DOS (1980s):
├── One program at a time
├── Want to run different program?
└── Exit current one first!

Modern computing IMPOSSIBLE! 
```

---

## The Solution: Context Switching

**Think of it like juggling tasks:**

```
You at your desk:
├── Reading book (Task A)
├── Phone rings! 
├── Put bookmark in book (save state)
├── Answer phone (Task B)
├── Phone call ends
├── Find bookmark (restore state)
└── Continue reading! 

CPU with programs:
├── Running Firefox (Program A)
├── Timer interrupt! 
├── Save Firefox state (all registers, memory view)
├── Run Chrome (Program B)
├── Timer interrupt again! 
├── Save Chrome state
├── Restore Firefox state
└── Firefox continues! 
```

**With context switching:**

```
T = 0ms:   Firefox running (10ms time slice)
T = 10ms:  SWITCH! → Chrome running
T = 20ms:  SWITCH! → Terminal running
T = 30ms:  SWITCH! → Music player running
T = 40ms:  SWITCH! → Back to Firefox!

All programs share the CPU!
Each program thinks it's running continuously!
Each program has NO IDEA it was paused!

This is MULTITASKING! 
```

---

## Why This Works

**The speed illusion:**

```
Time slice: 10ms per program
Human perception: ~100ms minimum

Program gets 10ms every 40ms:
├── 10ms running / 40ms total
├── 25% CPU time
└── But human can't tell it's switching!

To humans:
└── All programs appear to run simultaneously! 

To CPU:
└── Rapidly switching between programs! 

The illusion is PERFECT! 
```

---

# What is "Context"?

## Complete Program State

**Everything CPU needs to run a program:**

```
CONTEXT = Complete CPU state for one program

Includes:
├── All 16 general-purpose registers
├── Instruction pointer (where program is)
├── Stack pointer (function call state)
├── Page table pointer (memory view)
├── CPU flags (condition codes)
├── FPU state (floating point)
├── Segment registers (FS, GS)
└── EVERYTHING needed to resume!

Lose even ONE piece = Program crashes! 
Save EVERYTHING = Program never knows it stopped! 
```

---

## The 16 General-Purpose Registers

```
RAX = Accumulator (results, return values)
RBX = Base (general purpose)
RCX = Counter (loops, shift counts)
RDX = Data (general purpose)
RSI = Source Index (string operations, syscall arg 2)
RDI = Destination Index (string ops, syscall arg 1)
RBP = Base Pointer (stack frames)
RSP = Stack Pointer (top of stack) ← CRITICAL!
R8-R15 = Additional 64-bit registers

Example Firefox state:
├── RAX = 42 (some calculation)
├── RBX = 0x00500000 (pointer to data)
├── RCX = 1000 (loop counter)
├── RDX = 0 (unused)
├── RSI = 0x00600000 (source buffer)
├── RDI = 0x00700000 (dest buffer)
├── RBP = 0x7FFF0FF0 (stack frame)
├── RSP = 0x7FFF0FE0 (stack top)
└── R8-R15 = (various values)

ALL must be saved! 
```

---

## The Critical Pointers

### **RIP (Instruction Pointer)**

```
RIP = Where the program is executing

Example:
└── RIP = 0x00400500 (middle of some function)

When resuming:
└── CPU starts executing from THIS address!

Lose RIP = Don't know where to continue! 
Save RIP = Resume exactly where left off! 
```

### **RSP (Stack Pointer)**

```
RSP = Top of the stack

Stack contains:
├── Function call return addresses
├── Local variables
├── Function parameters
└── Saved registers

Example:
└── RSP = 0x7FFF0FE0

Lose RSP = Lose all function calls! 
Save RSP = Perfect call stack preserved! 
```

### **CR3 (Page Table Pointer)**

```
CR3 = Control Register 3 (page table base)

Contains:
└── Physical address of process's page table

Example:
├── Firefox: CR3 = 0x02000000 (Firefox's page table)
└── Chrome: CR3 = 0x03000000 (Chrome's page table)

CR3 determines ENTIRE memory view!

Switch CR3:
└── Entire virtual memory mapping changes!
    Virtual 0x00400000 now points to different physical memory!

This is how processes are ISOLATED! 
```

---

## FPU State (Floating Point)

```
FPU registers:
├── ST0-ST7 (x87 floating point stack)
├── XMM0-XMM15 (SSE 128-bit vectors)
└── YMM0-YMM15 (AVX 256-bit vectors)

Used for:
├── Floating point calculations (3.14159...)
├── Vector operations (SIMD)
├── Graphics processing
└── Multimedia codecs

Program doing math:
└── Relies on these values!

Must save if program uses FPU! 
```

---

## CPU Flags (RFLAGS)

```
RFLAGS register contains status flags:

CF (Carry Flag) - Arithmetic carry/borrow
ZF (Zero Flag) - Result was zero
SF (Sign Flag) - Result was negative
OF (Overflow Flag) - Signed overflow
IF (Interrupt Flag) - Interrupts enabled
DF (Direction Flag) - String operation direction

Program's conditional branches depend on these:

    CMP RAX, RBX    // Compare RAX and RBX
    JE  equal       // Jump if ZF=1 (equal)
    
If ZF not preserved:
└── Jump happens wrong! Program breaks! 

Must save RFLAGS! 
```

---

# When Does Context Switching Happen?

## The Five Triggers

### **1. Timer Interrupt (Most Common!)**

```
Every 10ms:
├── Timer chip fires interrupt
├── CPU handles timer interrupt
└── Timer handler calls scheduler

Scheduler checks:
├── Current process used 10ms (full time slice)
└── Time to switch! 

Forced preemption:
├── Process MUST yield CPU
├── No choice!
└── Ensures fairness! 

Timeline:
T = 0ms:   Firefox running
T = 10ms:  TIMER! Switch to Chrome
T = 20ms:  TIMER! Switch to Terminal
T = 30ms:  TIMER! Switch back to Firefox

Forever! This is preemptive multitasking! 
```

---

### **2. Process Voluntarily Sleeps**

```
Process calls: sleep(5)  // Sleep 5 seconds

Kernel logic:
├── Mark process: TASK_INTERRUPTIBLE
├── Add to timer wait queue
├── Will wake in 5 seconds
└── Process gives up CPU voluntarily

Scheduler:
├── "This process sleeping"
├── "Run someone else!"
└── Context switch to different process! 

5 seconds later:
├── Timer expires
├── Mark process: TASK_RUNNING
├── Add back to run queue
└── Eventually scheduled again! 

This is cooperative! Process chose to sleep!
```

---

### **3. Waiting for I/O (Blocking)**

```
Process calls: read(fd, buffer, size)

Kernel checks:
├── Data in page cache? NO
├── Must read from disk (takes milliseconds!)
└── Process CANNOT continue without data

Kernel logic:
├── Mark process: TASK_UNINTERRUPTIBLE
├── Submit disk I/O request
└── Process blocks (waiting for disk)

Scheduler:
├── "This process blocked"
├── "CPU wasted if we wait"
└── Context switch to different process! 

When disk completes:
├── Disk controller sends interrupt
├── Kernel marks process: TASK_RUNNING
├── Add to run queue
└── Eventually scheduled again! 

Efficiency! CPU does useful work instead of waiting! 
```

---

### **4. Process Terminates**

```
Process calls: exit(0)

Kernel logic:
├── Process finished
├── No longer needs CPU
├── Mark: TASK_DEAD
└── Remove from run queue

Scheduler:
├── "This process dead"
├── "Need different process"
└── Context switch! 

Dead process:
├── Resources cleaned up later
├── Never scheduled again
└── Removed from system! 
```

---

### **5. Higher Priority Process Ready**

```
High-priority process wakes up:
├── Example: Real-time audio processing
└── Must run NOW! 

Scheduler checks:
├── New process priority: 120
├── Current process priority: 100
└── 120 > 100? PREEMPT! 

Immediate context switch:
├── Current process interrupted
├── High-priority runs immediately
└── Ensures responsiveness! 

Use cases:
├── Real-time audio (can't skip!)
├── Video playback (smooth frames)
└── Interactive tasks (low latency)
```

---

# Complete Context Switch Flow

> **Let's trace Firefox → Chrome switch in COMPLETE DETAIL!**

## INITIAL STATE: Firefox Running

```
Process: Firefox (PID 5000)
Mode: Ring 3 (user mode)

Currently executing:
    ADD RAX, RBX  // Some Firefox calculation
    
CPU State:
├── RIP: 0x00400500 (Firefox code)
├── RSP: 0x7FFF0000 (Firefox user stack)
├── RAX: 42 (Firefox value)
├── RBX: 10 (Firefox value)
├── RCX, RDX, RSI, RDI, RBP, R8-R15: (Firefox values)
├── CR3: 0x02000000 (Firefox page table)
└── RFLAGS: (Firefox flags)

task_struct[5000] (Firefox metadata):
├── state: TASK_RUNNING (currently on CPU)
├── cpu_time_used: 1500ms
├── time_slice_remaining: 0ms ← EXPIRED!
└── priority: 120

Firefox has used its time slice! Time to switch!
```

---

## STEP 1: Timer Interrupt Fires

```
T = 10ms: Timer chip sends interrupt!

Hardware automatic response:
├── LAPIC signals CPU
├── CPU finishes ADD RAX, RBX (current instruction)
├── CPU checks: Interrupt pending? YES!
└── CPU begins interrupt handling! 

Save Firefox state on kernel stack:
├── Push SS (user stack segment: 0x002B)
├── Push RSP (user stack: 0x7FFF0000)
├── Push RFLAGS (Firefox flags)
├── Push CS (Ring 3 segment: 0x0033)
└── Push RIP (next instruction: 0x00400503)

Switch to kernel mode:
├── CS = 0x0010 (Ring 0!)
├── RSP = Firefox's kernel stack
└── Jump to timer interrupt handler

Now in kernel, Ring 0! 
```

---

## STEP 2: Timer Handler Runs

```
Now in: arch/x86/kernel/irq.c → timer_interrupt

Handler tasks:

STEP 2a: Save All Registers
Push to kernel stack:
└── RAX, RBX, RCX, RDX, RSI, RDI, RBP, R8-R15

All Firefox registers now on kernel stack! 

STEP 2b: Acknowledge Timer
└── Send EOI to LAPIC (clear interrupt)

STEP 2c: Update System Time
└── jiffies++ (global tick counter)

STEP 2d: Update Process Accounting
├── task_struct[5000].cpu_time_used += 10ms
├── task_struct[5000].time_slice_remaining = 0
└── Time slice EXPIRED! 

STEP 2e: Call Scheduler
└── schedule() ← Enter scheduler!
```

---

## STEP 3: Scheduler Picks Next Process

```
Now in: kernel/sched/core.c → schedule()

STEP 3a: Current Process State
Firefox (PID 5000):
├── state: Still TASK_RUNNING (runnable)
├── But not currently ON CPU
└── Will get another turn later!

STEP 3b: Examine Run Queue
All TASK_RUNNING processes:
├── PID 5000: Firefox (cpu_time: 1510ms)
├── PID 6000: Chrome (cpu_time: 1200ms) ← Least time!
├── PID 7000: Terminal (cpu_time: 1400ms)
└── PID 8000: Music (cpu_time: 1350ms)

STEP 3c: CFS Algorithm Chooses
CFS (Completely Fair Scheduler):
├── Pick process with LEAST cpu_time
├── Winner: Chrome (PID 6000) 
└── Give it CPU time to "catch up"!

STEP 3d: Prepare Switch
├── prev = task_struct[5000] (Firefox)
├── next = task_struct[6000] (Chrome)
└── Call: context_switch(prev, next) → THE SWITCH!
```

---

## STEP 4: Context Switch Happens!

```
Now in: arch/x86/kernel/process.c → context_switch()

This is THE CRITICAL FUNCTION! 

Two main operations:
A. Switch memory context (page tables)
B. Switch CPU context (registers, stack)
```

---

### **PART A: Switch Memory Context**

```
STEP 4A: Load New Page Table

Current memory view:
└── CR3 = 0x02000000 (Firefox page table)

Need Chrome's memory view:
└── CR3 = 0x03000000 (Chrome page table)

Execute ONE instruction:
    MOV CR3, 0x03000000

BOOM! Entire memory view changed! 

What this means:

Virtual address 0x00400000:
├── Before: Points to Firefox code
└── After: Points to Chrome code! 

Virtual address 0x7FFF0000:
├── Before: Points to Firefox stack
└── After: Points to Chrome stack! 

Every virtual address now maps to different physical memory!
Complete isolation between processes! 

Side effect:
└── TLB flushed! (all cached translations invalid)
    Next memory accesses slower (TLB misses)
    But necessary for correctness! 
```

---

### **PART B: Switch CPU Context**

```
STEP 4B1: Save Firefox Stack Pointer

Firefox's kernel stack currently active:
└── Contains all saved registers

Save the stack pointer:
task_struct[5000].kernel_stack_pointer = RSP

Now Firefox's state is completely saved!
Can restore it anytime! 

STEP 4B2: Load Chrome Stack Pointer

Get Chrome's saved stack:
RSP = task_struct[6000].kernel_stack_pointer

We're now on Chrome's kernel stack! 

Chrome's stack contains:
└── Chrome's registers (from when it was switched out before)

STEP 4B3: Save/Restore FPU State

Save Firefox FPU:
├── Execute: FXSAVE instruction
└── Store to: task_struct[5000].fpu_state

Load Chrome FPU:
├── Execute: FXRSTOR instruction
└── Load from: task_struct[6000].fpu_state

FPU state switched! 

STEP 4B4: Switch FS/GS Registers

FS/GS used for thread-local storage:
├── Save Firefox's FS/GS values
└── Load Chrome's FS/GS values

Complete context switched! 
```

---

## STEP 5: The Magic Return

```
context_switch() returns:

But wait... WHERE does it return to?

MAGIC TRICK! 

Because we switched stacks:
├── We're now on Chrome's kernel stack
├── Chrome's return address is on this stack
└── RET instruction pops Chrome's return address!

Returns to Chrome's code path!

Not Firefox's path anymore!
We've "teleported" into Chrome's execution! 
```

---

## STEP 6: Unwind Back Through Chrome

```
Return from context_switch():
└── Back in schedule() function
    But now executing Chrome's code path!

Return from schedule():
└── Back to timer interrupt handler
    But Chrome's handler path!

Now in timer handler, but Chrome's context! 
```

---

## STEP 7: Restore Chrome Registers

```
Timer handler cleanup:

Pop Chrome's registers from kernel stack:
└── R15, R14, R13, R12, R11, R10, R9, R8
    RBP, RDI, RSI, RDX, RCX, RBX, RAX

Chrome's register values now in CPU! 

Example:
├── RAX = 100 (Chrome's value, not Firefox's 42!)
├── RBX = 20 (Chrome's value, not Firefox's 10!)
└── All Chrome's values! 
```

---

## STEP 8: Return to User Mode

```
Handler executes: IRET (Interrupt Return)

CPU hardware automatically:
├── Pop RIP (Chrome's instruction: 0x00500600)
├── Pop CS (0x0033 - Ring 3)
├── Pop RFLAGS (Chrome's flags)
├── Pop RSP (Chrome's user stack: 0x7FFE0000)
└── Pop SS (user stack segment)

Switch to Ring 3! 
Jump to Chrome's RIP! 
```

---

## STEP 9: Chrome Resumes Execution!

```
CPU State Now:
├── RIP: 0x00500600 (Chrome code!)
├── RSP: 0x7FFE0000 (Chrome user stack!)
├── RAX: 100 (Chrome value)
├── RBX: 20 (Chrome value)
├── All registers: Chrome values!
├── CR3: 0x03000000 (Chrome page table!)
└── Memory view: Chrome's memory!

Mode: Ring 3 (user mode)

Chrome continues executing:
    MOV [RDI], RAX  // Chrome's next instruction
    
Chrome has NO IDEA it was paused!
├── Registers exactly as it left them
├── Stack exactly as it left it
├── Memory exactly as it left it
└── Thinks it's been running continuously!

Perfect illusion! 
```

---

## STEP 10: What Happened to Firefox?

```
Firefox is now READY but NOT RUNNING:

task_struct[5000] (Firefox):
├── state: TASK_RUNNING (can run, but not currently)
├── In scheduler run queue (waiting for turn)
├── All context saved:
│   ├── All registers saved on kernel stack
│   ├── Stack pointer saved (RSP)
│   ├── Page table remembered (CR3 = 0x02000000)
│   ├── FPU state saved
│   └── Everything preserved!
└── Next Firefox turn: ~30ms later

When Firefox's turn comes:
├── Reverse process happens
├── Load Firefox's CR3 (memory view)
├── Load Firefox's RSP (switch stack)
├── Pop Firefox's registers
├── IRET to Firefox
└── Firefox resumes from RIP = 0x00400503!

Firefox will continue EXACTLY where it left off!
Never knew it stopped! 
```

---

# Physical Transformations

## What Actually Changed in Hardware

### **CPU Registers**

```
BEFORE (Firefox):
├── RAX: 42
├── RBX: 10
├── RIP: 0x00400500
├── RSP: 0x7FFF0000
└── All Firefox values

AFTER (Chrome):
├── RAX: 100
├── RBX: 20
├── RIP: 0x00500600
├── RSP: 0x7FFE0000
└── All Chrome values

Complete register swap! Every register changed! 
```

---

### **Memory View (CR3 Register)**

```
BEFORE (CR3 = 0x02000000 - Firefox page table):

Virtual 0x00400000 → Physical 0x10000000 (Firefox code)
Virtual 0x7FFF0000 → Physical 0x10100000 (Firefox stack)

AFTER (CR3 = 0x03000000 - Chrome page table):

Virtual 0x00400000 → Physical 0x20000000 (Chrome code!)
Virtual 0x7FFF0000 → Physical 0x20100000 (Chrome stack!)

Same virtual addresses!
But now point to completely different physical memory!

This is how process isolation works! 
```

---

### **Stack Switching**

```
TWO stack switches happen:

1. Kernel Stack:
   ├── Before: Firefox's kernel stack
   └── After: Chrome's kernel stack

2. User Stack:
   ├── Before: Firefox's user stack (0x7FFF0000)
   └── After: Chrome's user stack (0x7FFE0000)

Each process has:
├── One kernel stack (for kernel mode)
└── One user stack (for user mode)

Both must be switched! 
```

---

### **TLB Flush**

```
TLB = Translation Lookaside Buffer

Caches virtual→physical translations:
├── Before switch: Full of Firefox translations
├── CR3 changes: TLB flushed! (all invalid)
└── After switch: Empty TLB

Chrome's first memory accesses:
├── TLB miss! (translation not cached)
├── Walk page table (slow!)
└── Cache new translation

Over time:
└── TLB fills with Chrome translations

This is overhead cost of switching! 
```

---

# Context Switch Costs

## Direct Costs

```
Typical context switch time: 1-5 microseconds

Breakdown:
├── Save registers: ~100 nanoseconds
├── Switch CR3 (MOV instruction): ~50 nanoseconds
├── TLB flush (automatic): ~500 nanoseconds
├── FPU save/restore: ~200 nanoseconds
├── Cache effects: ~1-2 microseconds
└── Miscellaneous overhead: ~500 nanoseconds

Total: ~2-3 microseconds typical 

Fast! But adds up with frequent switching!
```

---

## Indirect Costs (The Hidden Killers)

### **Cache Pollution**

```
L1/L2/L3 caches full of Firefox data:
├── Firefox code in instruction cache
├── Firefox data in data cache
└── Fast access! (nanoseconds)

Switch to Chrome:
├── Chrome accesses its data
├── Cache miss! (not in cache)
├── Load from RAM (slow! ~100 nanoseconds)
└── Evicts Firefox data from cache

Over time:
├── Firefox data completely evicted
└── Chrome data fills caches

Switch back to Firefox:
├── Firefox data not in cache anymore!
├── Cache misses! 
└── Performance degraded!

Performance impact:
└── 5-20% slower after context switch! 

Frequent switching = constant cache thrashing!
```

---

### **TLB Thrashing**

```
TLB holds ~100-1000 translations:

Firefox running:
├── TLB fills with Firefox translations
└── High TLB hit rate (fast!)

Context switch to Chrome:
├── CR3 changes
├── TLB flushed! (all translations invalid)
└── TLB now empty!

Chrome running:
├── Every memory access: TLB miss initially
├── Walk page table (4-5 memory accesses!)
└── Slow! 

Over time:
└── TLB fills with Chrome translations

Context switch back to Firefox:
└── TLB flushed again! Repeat! 

Frequent switching:
└── TLB never gets warm
    Constant misses
    Performance degraded! 
```

---

### **Branch Predictor Pollution**

```
CPU has branch prediction:
├── Predicts if/else, loops
└── 95%+ accuracy when trained

Firefox running:
└── Branch predictor learns Firefox patterns

Switch to Chrome:
├── Chrome has different patterns!
├── Predictions wrong! 
└── Pipeline stalls (mispredictions)

Performance impact:
└── ~1-5% slower after switch

Adds to overhead! 
```

---

## Total Impact

```
Direct cost: 2-3 microseconds
Indirect costs:
├── Cache misses: 5-20% slower
├── TLB misses: 2-10% slower
└── Branch mispredictions: 1-5% slower

Total performance impact:
└── 10-30% slower for first milliseconds after switch!

This is why:
└── Minimize context switches when possible!
    Keep programs on same CPU (affinity)
    Batch work to reduce switching! 
```

---

# Optimizations

## 1. Avoid Unnecessary Switches

```
Scheduler intelligence:

Check before switching:
├── Are there other TASK_RUNNING processes?
├── NO? Don't switch! Current process continues!
└── YES? Pick next process and switch

Example:
├── Only Firefox running (others sleeping/blocked)
├── Timer interrupt arrives
├── Scheduler: "No one else ready!"
└── Firefox continues! No switch! 

Saves overhead when system not busy!
```

---

## 2. CPU Affinity

```
Try to run process on SAME CPU:

Process ran on CPU 0 last time:
├── Its data in CPU 0's caches!
├── Its translations in CPU 0's TLB!
└── Branch predictor trained!

Scheduler tries:
├── Schedule on CPU 0 again if possible
└── Caches stay warm! 

Only move to different CPU when:
├── Load balancing required
└── CPU 0 too busy

This is "soft affinity" - prefer same CPU!

User can set "hard affinity":
└── Force process to specific CPUs
    taskset command: taskset -c 0,1 ./program
```

---

## 3. Lazy FPU Switching

```
Don't save/restore FPU if not used!

Track FPU usage:
└── Has this process used FPU?

Integer-only process:
├── Never uses FPU registers
├── Skip FPU save/restore!
└── Faster switch! 

When process first uses FPU:
├── CPU generates exception
├── Kernel: "Ah, using FPU now!"
├── Save previous process's FPU state
└── Mark: FPU in use

Next switch:
└── Now must save/restore FPU

Optimization for integer-heavy workloads!
```

---

## 4. Process Control Block Optimization

```
task_struct organized for cache efficiency:

Hot fields (accessed every switch):
├── state (RUNNING, SLEEPING, etc.)
├── priority
├── cpu_time_used
└── kernel_stack_pointer

Pack together in same cache line!
└── One cache fetch gets all! 

Cold fields (rarely accessed):
├── parent_pid
├── uid/gid
├── umask
└── executable path

Separate cache lines!
└── Don't pollute cache with unused data!

Minimizes cache misses during switch! 
```

---

# Connections to Other Mechanisms

## Timer Interrupts (Topic 3)

```
Timer interrupt triggers context switch!

Flow:
Timer fires (every 10ms)
    ↓
Timer interrupt handler
    ↓
Update accounting
    ↓
Call scheduler
    ↓
Scheduler picks next process
    ↓
Context switch! 

Without timer interrupts:
└── No preemptive multitasking possible!
    First process runs forever!

Timer + Context switch = Multitasking! 
```

---

## Memory Management (mm/)

```
CR3 register connects to page tables:

Context switch changes CR3:
    ↓
New page table loaded
    ↓
Entire virtual memory view changes!

This connects to:
├── mm/memory.c (page table walking)
├── mm/page_alloc.c (allocating page tables)
└── mm/mmap.c (virtual memory areas)

Memory isolation enforced by CR3 switch! 
```

---

## Scheduler (kernel/sched/)

```
Scheduler decides WHO runs:
└── Which process should run next?

Context switch does HOW:
└── Actually performs the switch

Division of labor:
├── Scheduler: Policy (who, when, priority)
└── Context switch: Mechanism (save, load, switch)

Both needed for multitasking! 
```

---

## Signal Delivery (kernel/signal.c)

```
After context switch, check for signals:

Context switch to process
    ↓
About to return to user mode
    ↓
Check: Any pending signals?
    ↓
YES? Deliver signal first!
    ↓
Then return to user mode

Signals delivered at context switch boundary!
Safe point for signal handling! 
```

---

# Complete Context Switch Summary

## What Is Context Switching?

**The multitasking mechanism:**

```
Context Switch =
├── Save complete state of current program
├── Load complete state of next program
├── CPU now runs different program
└── Creates illusion of simultaneous execution
```

---

## Why Does It Exist?

**Three critical purposes:**

```
1. MULTITASKING
   ├── Multiple programs share CPU
   ├── 200 programs on 4 cores
   └── All appear to run simultaneously

2. EFFICIENCY
   ├── Don't waste CPU on waiting programs
   ├── Blocked on I/O? Switch to someone else!
   └── Maximize CPU utilization

3. FAIRNESS
   ├── Every program gets CPU time
   ├── No program starves
   └── Time slices ensure equity
```

---

## When Does It Happen?

**Five trigger conditions:**

```
1. Timer Interrupt (forced, every 10ms)
   └── Preemptive multitasking

2. Process Sleeps (voluntary)
   └── sleep(), usleep(), nanosleep()

3. Waiting for I/O (blocked)
   └── read(), write(), network, disk

4. Process Terminates (done)
   └── exit(), return from main()

5. Higher Priority Ready (preemption)
   └── Real-time process wakes up
```

---

## What Gets Saved?

**Complete CPU state:**

```
REGISTERS:
├── 16 general-purpose: RAX, RBX, RCX, RDX, RSI, RDI, RBP, RSP, R8-R15
├── RIP (instruction pointer)
├── RFLAGS (CPU flags)
└── Segment registers (CS, DS, ES, FS, GS, SS)

MEMORY CONTEXT:
└── CR3 (page table pointer - entire memory view!)

FPU STATE:
├── ST0-ST7 (x87 floating point)
├── XMM0-XMM15 (SSE)
└── YMM0-YMM15 (AVX)

STACKS:
├── User stack pointer
└── Kernel stack pointer

EVERYTHING needed to resume! 
```

---

## Complete Flow Summary

**Firefox to Chrome:**

```
Firefox running (Ring 3)
    ↓
Timer interrupt fires
    ↓
Save Firefox state on kernel stack (hardware)
    ↓
Switch to Ring 0 (kernel mode)
    ↓
Timer handler: Save all registers
    ↓
Call scheduler
    ↓
Scheduler picks Chrome (CFS algorithm)
    ↓
Context switch function:
    ├── Save Firefox kernel stack pointer
    ├── Switch CR3 (Firefox → Chrome page table)
    ├── Load Chrome kernel stack pointer
    ├── Save/restore FPU state
    └── Switch FS/GS registers
    ↓
Return from context_switch()
    (Now on Chrome's stack! Magic!)
    ↓
Return from scheduler (Chrome's path)
    ↓
Return from timer handler (Chrome's path)
    ↓
Restore Chrome's registers
    ↓
IRET: Return to Chrome (Ring 3)
    ↓
Chrome resumes execution!

Firefox saved, ready for next turn!
```

---

## Physical Reality

**What actually changes:**

```
CPU REGISTERS:
├── All 16 general-purpose registers: New values
├── RIP: Different code location
├── RSP: Different stack
└── RFLAGS: Different flags

CR3 REGISTER:
├── Before: 0x02000000 (Firefox page table)
├── After: 0x03000000 (Chrome page table)
└── Entire memory view transformed!

MEMORY VIEW:
├── Virtual 0x00400000: Now points to different physical memory
├── Virtual 0x7FFF0000: Different stack in physical RAM
└── Complete isolation between processes!

STACKS:
├── Kernel stack: Switched (different stack in kernel)
├── User stack: Switched (different stack in user space)
└── Two stacks per process!

CACHES:
├── TLB: Flushed (all translations invalid)
├── L1/L2/L3: Polluted (new process data)
└── Branch predictor: Retrained
```

---

## Performance Characteristics

**Costs and overhead:**

```
DIRECT COSTS:
├── Switch time: 1-5 microseconds
├── Save registers: ~100 ns
├── CR3 switch: ~50 ns
├── TLB flush: ~500 ns
├── FPU save/restore: ~200 ns
└── Overhead: ~2-3 microseconds total

INDIRECT COSTS:
├── Cache misses: 5-20% performance loss
├── TLB misses: 2-10% performance loss
├── Branch mispredictions: 1-5% performance loss
└── Total: 10-30% slower initially after switch

FREQUENCY:
├── Every 10ms for each process
├── 100 switches/second per process
├── With 200 processes: 20,000 switches/second!
└── Overhead adds up! 
```

---

## Optimization Strategies

**Minimizing overhead:**

```
1. Avoid Unnecessary Switches
   └── Don't switch if no one else ready

2. CPU Affinity
   └── Keep process on same CPU (warm caches)

3. Lazy FPU Switching
   └── Skip FPU if not used

4. Cache-Optimized Data Structures
   └── Pack hot fields together

5. Batch Work
   └── Reduce switching frequency

6. Process Priorities
   └── Important tasks get more CPU time
```

---

## Key Design Principles

**Why context switching works:**

```
1. COMPLETE STATE PRESERVATION
   ├── Save EVERYTHING
   ├── Perfect restoration
   └── Programs never know they stopped

2. HARDWARE SUPPORT
   ├── CR3 switch in one instruction
   ├── FXSAVE/FXRSTOR for FPU
   └── Efficient state management

3. SEPARATION OF STACKS
   ├── Kernel stack (trusted)
   ├── User stack (per-process)
   └── Clean isolation

4. MEMORY ISOLATION
   ├── CR3 switch changes entire view
   ├── Processes can't see each other
   └── Security enforced in hardware

5. SCHEDULER INTEGRATION
   ├── Scheduler decides WHO
   ├── Context switch does HOW
   └── Clean separation of concerns
```

---

# You've Mastered Context Switching!

## What You Now Understand

**The complete multitasking picture:**

```
 Context concept
   └── Complete program state in CPU

 Switch triggers
   └── Timer, sleep, I/O, termination, priority

 What gets saved
   └── All registers, CR3, FPU, stacks, everything!

 Complete switch flow
   └── Firefox → Chrome, every step traced

 Physical transformations
   └── Registers, memory view, stacks all change

 Performance costs
   └── Direct and indirect overhead

 Optimizations
   └── Affinity, lazy FPU, cache optimization

 Design brilliance
   └── Why this architecture enables modern computing
```

---

## The Big Reveal

**Remember clicking between Firefox and Chrome?**

```
You click Chrome icon in taskbar
```

**Now you see the UNIVERSE behind it:**

```
One click triggers:
├── Window manager requests Chrome activation
├── Kernel checks: Chrome process exists
├── Scheduler: Boost Chrome priority temporarily
├── Next timer interrupt (≤10ms wait)
├── Timer handler calls scheduler
├── Scheduler picks Chrome (high priority)
├── Context switch begins:
│   ├── Save current process registers (all 16+)
│   ├── Save stack pointer (kernel and user)
│   ├── Save FPU state (128+ bytes)
│   ├── Save FS/GS registers
│   ├── Switch CR3 (entire memory view!)
│   ├── Flush TLB (translation cache)
│   ├── Load Chrome's page table
│   ├── Load Chrome's stack pointer
│   ├── Load Chrome's FPU state
│   ├── Load Chrome's registers
│   └── Return to Chrome's code!
├── Chrome resumes exactly where it left off
├── Redraws window to foreground
└── You see Chrome window appear!

All in ~3 microseconds! 

That's the magic of context switching:
└── Seamless switching between programs
    Perfect state preservation
    Illusion of continuous execution! 
```

---

## Your New Superpowers

**With this knowledge, you can now:**

```
 Understand system performance
   └── Why frequent context switches slow things down!

 Optimize applications
   └── Minimize blocking, batch work, reduce switches!

 Debug mysterious slowdowns
   └── Check context switch rate: vmstat, perf!

 Design better systems
   └── Consider switch overhead in architecture!

 Analyze CPU usage
   └── Understand %sys time, scheduling delays!

 Configure process priority
   └── nice, renice, CPU affinity with taskset!

 Appreciate the design
   └── Multitasking is a carefully orchestrated dance!
```

---

## The Deeper Truth

**Context switching taught you more than multitasking:**

```
You learned:

 The illusion of simultaneity
   └── How sequential execution appears parallel

 Complete state management
   └── What "state" really means in computing

 Performance trade-offs
   └── Overhead vs. responsiveness

 Resource sharing
   └── Fair allocation of limited resources

 Hardware-software cooperation
   └── CR3, FXSAVE, IRET work together

 Separation of concerns
   └── Scheduler (policy) vs. switcher (mechanism)

These patterns appear everywhere:
└── Virtual machines, containers, green threads
    Context switching is fundamental! 
```

---

## The Path Forward

**This is your foundation:**

```
You've conquered: Context Switching
                 └── The MULTITASKING ENGINE

Next frontiers await:

 Process Scheduler (kernel/sched/)
   └── WHO runs, WHEN, and WHY!

 Process Creation (kernel/fork.c)
   └── How new processes are born!

 Memory Management (mm/)
   └── What CR3 points to, how page tables work!

 System Calls (arch/x86/entry/)
   └── How user programs request kernel services!

Each builds on context switching!
The switch is your foundation!
```

---

## A Final Thought

**Every time your computer feels responsive:**

```
Playing music
Browsing web
Compiling code
Downloading files
All at the same time
```

**Remember:**

```
It's not magic
    ↓
It's context switching
    ↓
Happening thousands of times per second
    ↓
Save Firefox
    ↓
Load Chrome
    ↓
Save Chrome
    ↓
Load Music player
    ↓
Save Music player
    ↓
Load Firefox
    ↓
Forever, flawlessly
    ↓
Each program thinks it owns the CPU
    ↓
Each program has no idea others exist
    ↓
Perfect isolation
    ↓
Perfect preservation
    ↓
Perfect illusion
    ↓
That's the beauty of context switching! 
```

**Modern computing is built on this foundation!**

---

## The Context Switch Code in Perspective

**arch/x86/kernel/process.c:**

```
~200 lines of switching code
+
Scheduler integration
+
Architecture-specific optimizations

But these 200 lines:
├── Execute thousands of times per second
├── Enable all multitasking
├── Make modern computing possible
├── Preserve perfect program state
├── Isolate processes completely
├── Balance load across CPUs
└── Create the illusion of simultaneity

Some of the most critical code
    in the operating system! 
```

---

## Journey Complete!

```
         FIREFOX RUNNING
              ↓
      [TIMER INTERRUPT]
              ↓
        SAVE FIREFOX STATE
              ↓
         ═════════════
        ║  SCHEDULER  ║
         ═════════════
              ↓
       PICK NEXT PROCESS
              ↓
     CONTEXT SWITCH! ← You understand THIS now!
              ↓
        LOAD CHROME STATE
              ↓
         CHROME RUNNING
```
---

> *"A context switch is not an interruption - it's how hundreds of programs share one CPU and none of them ever know."*

**You now understand how the CPU switches between programs. Context switching has no secrets left!**
