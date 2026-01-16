# 02-Kernel-Core-Functions

Deep dive into the `kernel/` directory - the brain of the Linux kernel. This document explores process management, system calls, time management, and interrupt handling at the hardware level.

---

## Table of Contents

1. [Overview](#overview)
2. [Process Termination](#1-process-termination-exitc)
3. [System Calls](#2-system-calls-sysc)
4. [Time Management](#3-time-management-time)
5. [Interrupt Handling](#4-interrupt-handling-irq)
6. [Workqueues](#5-workqueues-workqueuec)
7. [Summary](#summary)

---

## Overview

The `kernel/` directory contains the core functionality of Linux:

```
kernel/
├── sched/         ← Process scheduling (covered in Part 1)
├── fork.c         ← Process creation (covered in Part 1)
├── signal.c       ← Signal handling (covered in Part 1)
├── exit.c         ← Process termination
├── sys.c          ← System call infrastructure
├── time/          ← Time management
├── irq/           ← Interrupt handling
└── workqueue.c    ← Deferred work execution
```

> **This document covers:** Process termination, system calls, time management, interrupts, and workqueues - the mechanisms that make multitasking and hardware interaction possible.

---

## 1. Process Termination (exit.c)

### The Problem

When a program ends, what happens physically?

**Scenario:** Firefox running (PID 5000)
- Occupies 100 MB RAM
- Has open files (config, cache)
- Has network connections
- Scheduler giving it CPU time

User clicks X button → What happens to all these resources?

---

### How Programs Exit

#### Step 1: Program Calls exit()

**Normal termination:**

```c
int main() {
    // Do work
    return 0;  // Success
}

// When main() returns:
// 1. C library calls: exit(0)
// 2. exit() makes system call to kernel
// 3. Kernel's exit handler starts cleanup
```

**Forced termination:**

```
Program crashes:
    Segmentation fault → Kernel sends SIGSEGV
    Default handler → Calls exit(1)
    
User kills program:
    kill 5000 → Kernel sends SIGTERM
    Default handler → Calls exit(0)
```

All paths lead to exit!

---

#### Step 2: Close All Open Files

**Physical cleanup process:**

```
Program had open files:
    FD 0: stdin (terminal)
    FD 1: stdout (terminal)
    FD 2: stderr (terminal)
    FD 3: /home/user/document.txt
    FD 4: /tmp/cache.dat
    FD 5: socket (network connection)
```

**Exit handler walks through ALL file descriptors:**

```
For each open file:
    1. Flush buffers (write pending data to disk)
    2. Update file metadata (last modified time)
    3. Release locks (if file was locked)
    4. Close file descriptor
    5. Free kernel structures
```

> **Why this matters:**
> - Without proper cleanup, unsaved data is lost
> - File locks held forever
> - Potential disk corruption
> 
> Exit ensures data integrity and resource release.

---

#### Step 3: Free All Memory

**Memory regions owned by program:**

```
Virtual memory (program's view):
    0x00400000 - 0x00500000: Program code (1 MB)
    0x00600000 - 0x10000000: Heap (100 MB)
    0x7FFF0000 - 0x80000000: Stack (1 MB)

Physical memory (actual RAM):
    Physical pages backing these virtual addresses
```

**Exit handler process:**

```
1. Walk program's page table
2. For each virtual page:
    - Find physical page it maps to
    - Mark physical page as FREE
    - Remove page table entry
3. Free the page table itself
4. Free task_struct
```

**Memory reclaimed:**

```
BEFORE exit:
    Used RAM: 14 GB / 16 GB
    
AFTER exit (freed 100 MB):
    Used RAM: 13.9 GB / 16 GB
    
That 100 MB now available for other programs! ✅
```

---

#### Step 4: Notify Parent Process

**The parent-child relationship:**

```
Parent (bash, PID 1000) created child (firefox, PID 5000)

When child exits, parent might want to know:
    - Did child succeed? (exit code 0)
    - Did child fail? (exit code 1)
    - Did child crash? (exit code 139 = segfault)
```

**Physical notification mechanism:**

```
Child exits:
    1. Kernel sets child's state: ZOMBIE
       (Yes, really called zombie! 🧟)
    
    2. Kernel saves exit code in task_struct:
       task_struct[5000].exit_code = 0
    
    3. Kernel sends signal to parent:
       SIGCHLD → Parent PID 1000
    
    4. Child becomes zombie:
       - Most resources freed
       - But task_struct remains
       - Waiting for parent to collect exit code
```

**Why zombie state exists:**

```
Parent might want exit code!

Parent calls: wait() or waitpid()
    Kernel returns child's exit code
    Parent knows: "My child exited with code 0"
    
After parent collects exit code:
    Kernel fully removes child's task_struct
    Zombie gone! ✅

If parent never collects:
    Zombie stays forever! ❌
    (Wasting small amount of memory)
```

---

#### Step 5: Orphan Handling

**Special case: Parent dies first**

```
Scenario:
    Parent: bash (PID 1000)
    Child: firefox (PID 5000)
    
    User kills bash!
    bash exits → firefox becomes ORPHAN!
    
Question: Who's firefox's parent now?
```

**Re-parenting to init:**

```
When parent dies:
    Kernel finds all children
    Changes their PPID (parent PID)
    
    firefox: PPID was 1000 → Now PPID = 1
    
    PID 1 = init process (first process)
    init ADOPTS all orphans!
    
init's job:
    Call wait() periodically
    Collect exit codes from adopted children
    Prevent zombie accumulation ✅
```

---

#### Step 6: Final Removal

```
After all cleanup:

Kernel removes from scheduler:
    task_struct[5000] deleted
    No longer in scheduler's list
    
CPU never schedules this PID again!

Program completely gone! ✅
```

---

### Exit Codes

Exit code = Number program returns when it exits.

**Standard meanings:**

| Code | Meaning |
|------|---------|
| 0 | Success! Everything worked ✅ |
| 1 | General error ❌ |
| 2 | Misuse of command |
| 126 | Command cannot execute |
| 127 | Command not found |
| 128+N | Killed by signal N |
| 130 | Killed by SIGINT (CTRL+C) |
| 139 | Killed by SIGSEGV (segfault) |

**Example in bash:**

```bash
$ firefox
$ echo $?    # Show exit code of last command
0            # Firefox exited normally

$ kill -9 firefox
$ echo $?
137          # 128 + 9 (SIGKILL)
```

---

### Why Careful Exit Matters

**Database example:**

```
Database program:
    - Has uncommitted transaction in memory
    - Writing to disk
    - Holding locks
    
If killed abruptly (power loss):
    - Transaction lost ❌
    - Disk corruption ❌
    - Database broken ❌
    
If exits properly (SIGTERM):
    - Commits transaction ✅
    - Syncs disk ✅
    - Releases locks ✅
    - Database safe ✅
```

> **Best Practice:**
> - `kill` (SIGTERM): Polite "please exit"
> - `kill -9` (SIGKILL): Force kill (emergency only!)
> 
> Always try `kill` first! Only use `kill -9` if program is frozen.

---

## 2. System Calls (sys.c)

### The Problem

User programs need kernel services, but they can't access kernel directly.

**The separation:**

```
Ring 3 (User programs):
    ❌ Cannot access hardware
    ❌ Cannot access other program's memory
    ❌ Cannot do privileged operations
    
Ring 0 (Kernel):
    ✅ Can access hardware
    ✅ Can access all memory
    ✅ Can do anything

Question: How do user programs ASK kernel for help?
```

> **System calls are the bridge!**

---

### What Are System Calls?

System call = Controlled entry into kernel.

**Think of it like airport security:**

```
User space = Public area (can't board plane)
Kernel space = Secured area (can board plane)

System call = Going through security
    - Checked and authorized
    - Can't bring weapons (malicious code)
    - Can enter secured area
```

---

### How System Calls Work

#### Step 1: User Program Invokes System Call

```c
// Program wants to read file:
read(fd, buffer, size);

// What read() actually is:
//     NOT a normal function!
//     It's a WRAPPER that makes system call!
```

**Physical mechanism:**

```
read() function prepares:
    1. System call number (read = 0)
    2. Arguments (fd, buffer, size)
    3. Executes special CPU instruction: syscall
```

---

#### Step 2: CPU Switches to Kernel Mode

**The syscall instruction (CPU hardware feature):**

```
BEFORE syscall:
    CPL (Current Privilege Level) = 3 (user mode)
    RIP = 0x400500 (user program code)
    CS register = 0x0033 (user code segment)
    SS register = 0x002B (user stack segment)

CPU does (in hardware!):
    1. Check: Is syscall allowed? (yes, always)
    2. Save current state:
        - Save user RIP to RCX register
        - Save user RFLAGS to R11 register
    3. Switch privilege:
        - CPL = 0 (kernel mode)
        - CS = 0x0010 (kernel code segment)
        - SS = 0x0018 (kernel stack segment)
    4. Load kernel entry point:
        - RIP = 0x00001000 (from MSR register)
        
AFTER syscall:
    CPL = 0 (kernel mode!)
    RIP = 0x00001000 (kernel's syscall entry!)
    
CPU now executing KERNEL code!
```

> **Key insight:** CPU hardware changes privilege level! This is enforced by silicon, not software.

---

#### Step 3: Kernel Entry Point Receives Call

**Physical flow in kernel:**

```
CPU jumped to: 0x00001000 (kernel's entry point)

Entry point (entry_64.S) does:
    1. Save all user registers:
        Push RAX, RBX, RCX, RDX... onto kernel stack
        (Preserve user state!)
    
    2. Switch to kernel stack:
        User stack: 0x7FFF000 → Can't trust it!
        Kernel stack: 0x01FF000 → Safe kernel memory
    
    3. Look up system call number:
        RAX register = 0 (read syscall)
        
    4. Call appropriate handler:
        sys_call_table[0] → sys_read()
        Jump to sys_read() function!
```

---

#### Step 4: Kernel Executes Requested Operation

**sys_read() does the actual work:**

```c
sys_read(fd, buffer, size):
    
    1. Validate arguments:
        - Is fd valid? (check open file table)
        - Is buffer valid? (check page tables)
        - Is size reasonable? (not crazy big)
        
    2. Security check:
        - Does process have permission?
        - Is file readable?
        
    3. Perform operation:
        - If data in cache: Copy from cache
        - If not: Read from disk (via driver)
        - Copy data to user's buffer
        
    4. Return result:
        - RAX = number of bytes read
        - Or RAX = -1 (error)
```

---

#### Step 5: Return to User Space

**The sysret instruction (reverse of syscall):**

```
Kernel finished work!

Kernel executes: sysret instruction

CPU does (in hardware!):
    1. Restore user state:
        - RIP = RCX (saved user RIP)
        - RFLAGS = R11 (saved flags)
    2. Switch privilege:
        - CPL = 3 (back to user mode)
        - CS = 0x0033 (user code segment)
        - SS = 0x002B (user stack segment)
    3. Resume user program:
        - RIP = 0x400500 (where program was)

AFTER sysret:
    CPL = 3 (user mode again)
    RIP = 0x400500 (user program)
    RAX = 1024 (bytes read)
    
Program continues:
    "read() returned 1024, success!"
```

---

### The System Call Table

**How kernel knows which function to call:**

```c
// Kernel has table: sys_call_table[]
// Index = System call number
// Value = Function pointer

sys_call_table[0] = sys_read
sys_call_table[1] = sys_write
sys_call_table[2] = sys_open
sys_call_table[3] = sys_close
sys_call_table[4] = sys_stat
sys_call_table[5] = sys_fstat
...
sys_call_table[300+] = ... (300+ system calls!)

// When user calls read():
//     RAX = 0 (read's number)
//     Kernel looks up: sys_call_table[0]
//     Calls: sys_read()
```

---

### Common System Calls

**File operations:**
- `open()`, `close()`, `read()`, `write()`
- `lseek()`, `stat()`, `mkdir()`, `rmdir()`

**Process operations:**
- `fork()`, `exec()`, `exit()`, `wait()`
- `kill()`, `getpid()`, `getppid()`

**Memory operations:**
- `brk()`, `mmap()`, `munmap()`

**Network operations:**
- `socket()`, `bind()`, `listen()`, `accept()`
- `send()`, `recv()`, `connect()`

**Time operations:**
- `time()`, `gettimeofday()`, `sleep()`

**Total:** 300+ system calls!

---

### Why System Calls Are Slow

**The overhead:**

```
Normal function call:
    CALL instruction (~1 nanosecond)
    Function executes
    RET instruction (~1 nanosecond)
    Total: ~2 nanoseconds ✅

System call:
    syscall instruction (~100 nanoseconds)
    Save registers (~20 nanoseconds)
    Switch stacks (~10 nanoseconds)
    Validate arguments (~50 nanoseconds)
    Do work (~variable)
    Restore registers (~20 nanoseconds)
    sysret instruction (~100 nanoseconds)
    Total: ~300+ nanoseconds ❌
    
150× slower than function call!
```

**Why so slow?**

1. **Security checks:**
   - Validate every pointer
   - Check permissions
   - Verify arguments

2. **Privilege switching:**
   - Save/restore state
   - Change CPU mode
   - Switch stacks

> All necessary for protection! But costs performance.

---

### Optimization: vDSO

**Some system calls don't need kernel!**

**Example: gettimeofday()**

```
Old way:
    System call → Enter kernel
    Kernel reads clock
    Return to user
    300 nanoseconds ❌
    
New way (vDSO):
    Kernel maps read-only page to user space
    Page contains: Current time (updated by kernel)
    User reads directly: No system call!
    1 nanosecond ✅
    
300× faster!
```

**Other vDSO calls:**
- `gettimeofday()` - Read time
- `clock_gettime()` - High-res time
- `getcpu()` - Which CPU am I on?

All readable without system call!

---

## 3. Time Management (time/)

### The Problem

Kernel needs to track multiple types of time:

1. **Real-world time:** "What time is it?" (12:30:45 PM)
2. **Process CPU time:** "How long did this program run?"
3. **Sleep/timers:** "Wake me up in 5 seconds"
4. **Scheduler timing:** "10ms time slice expired!"

All different time concepts!

---

### Time Sources (Physical Hardware)

#### 1. RTC (Real-Time Clock)

```
Physical chip on motherboard:
    - Battery-powered (keeps time when PC off!)
    - Low resolution (1 second granularity)
    - Used for: Wall clock time
    
Location: CMOS chip (same as BIOS settings)

Kernel reads RTC at boot:
    Read: "2024-01-16 10:30:00"
    Kernel now knows wall clock time!
```

---

#### 2. PIT (Programmable Interval Timer)

```
8254 timer chip:
    - Generates interrupts at fixed rate
    - Programmable frequency (100 Hz typical)
    - Used for: Scheduler ticks
    
Every 10ms (100 Hz):
    Timer fires interrupt
    Kernel updates time
    Scheduler runs
```

---

#### 3. TSC (Time Stamp Counter)

```
CPU internal counter:
    - Increments every CPU cycle
    - 3 GHz CPU = 3 billion ticks/second
    - Very high resolution!
    - Used for: Precise timing
    
Read TSC:
    Special CPU instruction: RDTSC
    Returns: Number of cycles since boot
    
Convert to time:
    Cycles / CPU_frequency = Time in seconds
```

---

#### 4. HPET (High Precision Event Timer)

```
Modern timer chip:
    - Higher resolution than PIT
    - Multiple independent timers
    - Used for: High-res timing
    
Resolution: 10 MHz (100 nanosecond precision)
Much better than PIT!
```

---

### How Kernel Maintains Time

**The jiffies counter:**

```c
// Kernel has global variable: jiffies

// Every timer interrupt (10ms):
jiffies++

// Boot time: jiffies = 0
// After 1 second: jiffies = 100 (100 ticks @ 100 Hz)
// After 1 hour: jiffies = 360,000
// After 1 day: jiffies = 8,640,000

// Current time = Boot time + (jiffies / HZ)
```

**Example calculation:**

```
Boot time: January 16, 2024 08:00:00
jiffies: 360,000 (1 hour worth)
HZ: 100 (100 ticks per second)

Current time = 08:00:00 + (360,000 / 100 seconds)
             = 08:00:00 + 3,600 seconds
             = 09:00:00
             
Current time: January 16, 2024 09:00:00 ✅
```

---

### Process CPU Time Tracking

**Every process has CPU time counters:**

```c
task_struct has fields:
    utime: User mode time (nanoseconds)
    stime: System mode time (nanoseconds)
    
Every timer tick (10ms):
    If process in user mode:
        process->utime += 10,000,000 ns
    If process in kernel mode (system call):
        process->stime += 10,000,000 ns
```

**The time command uses this:**

```bash
$ time firefox

real    2m30.500s    # Wall clock time (stopwatch)
user    1m45.200s    # CPU time in user mode
sys     0m12.800s    # CPU time in kernel mode

# Notice: user + sys < real
# Why? Process was WAITING (I/O, sleeping)!
```

---

### Sleep and Timers

**How sleep() works:**

```c
// Program calls:
sleep(5);  // Sleep 5 seconds
```

**Physical mechanism:**

```
1. System call to kernel

2. Kernel calculates wake time:
    Current jiffies: 100,000
    Sleep duration: 5 seconds = 500 jiffies
    Wake time: 100,500 jiffies
    
3. Kernel changes process state:
    task_struct[PID].state = TASK_INTERRUPTIBLE
    task_struct[PID].wake_time = 100,500
    
4. Kernel adds to timer list:
    "Wake PID 5000 at jiffies 100,500"
    
5. Process removed from scheduler:
    Not eligible to run!
    
6. Every timer tick:
    Kernel checks: jiffies == 100,500?
    NO → Keep sleeping
    YES → Wake process!
        task_struct[5000].state = TASK_RUNNING
        Add back to scheduler
        
7. Process resumes execution!
```

---

**Why INTERRUPTIBLE?**

```
TASK_INTERRUPTIBLE:
    Sleep can be interrupted by signal
    CTRL+C wakes process early
    
TASK_UNINTERRUPTIBLE:
    Sleep CANNOT be interrupted
    Must wait full duration
    Used for critical I/O
```

---

### High-Resolution Timers

**For precise timing needs:**

```
Old way (jiffies):
    Resolution: 10ms (HZ = 100)
    sleep(0.001) → Actually sleeps 10ms! ❌
    
New way (hrtimer):
    Resolution: nanoseconds!
    Uses TSC or HPET
    sleep(0.001) → Sleeps 1ms! ✅
```

**Used for:**
- Multimedia (audio/video)
- Real-time applications
- Precise measurements

---

## 4. Interrupt Handling (irq/)

### The Problem

Hardware needs attention NOW!

> **Interrupts = Hardware yelling at CPU**

**Scenario: Network packet arrives**

```
Option 1 (Polling):
    CPU constantly checks: "Packet arrived yet?"
    Check, check, check, check...
    Wastes CPU time! ❌
    
Option 2 (Interrupts):
    Network card sends signal: "PACKET HERE!"
    CPU immediately responds
    Efficient! ✅
```

---

### What Are Interrupts?

Interrupt = Electrical signal to CPU.

**Physical hardware:**

```
Network card has pin → CPU's INTR pin
    
When packet arrives:
    Network card sets pin HIGH (voltage)
    CPU detects voltage change
    CPU MUST respond (hardware requirement!)
```

---

### How Interrupts Work

#### Step 1: Hardware Generates Interrupt

```
Network card receives packet:

1. Packet enters network card
2. Card stores packet in its memory
3. Card sets interrupt pin HIGH
4. Electrical signal travels to CPU
```

---

#### Step 2: CPU Detects Interrupt

```
CPU executing normal code:
    Fetch instruction from 0x400000
    Decode: ADD
    Execute: Add registers
    
DURING execution:
    CPU checks interrupt pin (every cycle!)
    Detects: Pin is HIGH!
    
CPU immediately:
    1. Finishes current instruction
    2. Stops normal execution
    3. Prepares to handle interrupt
```

---

#### Step 3: CPU Identifies Interrupt Source

```
CPU asks: "Which device interrupted?"

Interrupt controller (PIC/APIC) responds:
    "Interrupt number 50" (network card's number)

CPU looks up in IDT (Interrupt Descriptor Table):
    IDT[50] = 0x00002000 (handler address)
    
CPU knows: Jump to 0x00002000 to handle this!
```

---

#### Step 4: Save Current State

```
Before handling interrupt:

CPU saves on stack:
    - RIP (where program was)
    - RFLAGS (CPU flags)
    - CS (code segment)
    - Error code (if applicable)
    
Why? To RESUME program later!

If interrupted program was in user mode:
    Also save: RSP (user stack), SS (stack segment)
    Switch to kernel stack!
```

---

#### Step 5: Execute Interrupt Handler

```
CPU jumps to: 0x00002000 (network handler)

Handler does:

1. Acknowledge interrupt:
    Tell interrupt controller: "I got it!"
    Clear interrupt pin
    
2. Read data from device:
    Network card has packet
    Copy packet to kernel memory
    
3. Process data:
    Pass packet to network stack
    TCP/IP processing
    
4. Wake up waiting process:
    If process was waiting for network data
    Change state: TASK_RUNNING
```

---

#### Step 6: Return from Interrupt

```
Handler finishes!

CPU executes: IRET (interrupt return)

CPU restores:
    - Pop RIP (resume address)
    - Pop RFLAGS
    - Pop CS
    - If was user mode: Pop RSP, SS
    
CPU continues:
    Exactly where it was!
    Like nothing happened!
    (Except interrupt was handled)
```

---

### Interrupt Types

#### 1. Hardware Interrupts

```
External devices:
    IRQ 0: Timer (every 10ms)
    IRQ 1: Keyboard (key pressed)
    IRQ 3: Serial port
    IRQ 4: Serial port
    IRQ 6: Floppy disk
    IRQ 8: RTC (real-time clock)
    IRQ 12: Mouse
    IRQ 14: Primary IDE (hard disk)
    IRQ 15: Secondary IDE
    
Modern: 24+ IRQ lines (PCI devices)
```

---

#### 2. Software Interrupts

```
Triggered by CPU instructions:
    INT 0x80: Old Linux system call
    SYSCALL: Modern system call
    INT 3: Debugger breakpoint
```

---

#### 3. Exceptions

```
CPU-generated:
    Divide by zero
    Invalid opcode
    Page fault
    General protection fault
    Double fault
    
These are ERROR conditions!
```

---

### Interrupt Priorities

**Not all interrupts are equal!**

```
High priority (must handle fast!):
    Timer: Keeps system time accurate
    Network: Packets arriving fast
    
Low priority (can wait):
    Keyboard: Human types slowly
    
If multiple interrupts arrive:
    Handle highest priority first!
    Others wait in queue
```

---

### Interrupt Latency

**Time from interrupt to handling:**

```
Interrupt fires → Handler runs:
    
Latency = Time waiting to run

Low latency system:
    < 10 microseconds
    Real-time systems
    
High latency system:
    > 1000 microseconds
    Desktop systems OK
```

**Why latency matters:**

```
Audio playback:
    Need samples every 10ms
    High latency → Audio skips! ❌
    
Keyboard:
    Human types ~100ms between keys
    High latency OK ✅
```

---

### Top Half vs Bottom Half

**Problem: Interrupt handler must be FAST!**

```
If handler takes too long:
    Other interrupts wait!
    System becomes unresponsive! ❌
```

**Solution: Split work!**

#### Top Half (in interrupt handler)

> **MUST BE FAST!** (<1 microsecond)

```
Do minimum:
    1. Acknowledge interrupt
    2. Read data from hardware
    3. Schedule bottom half
    4. Return IMMEDIATELY!
```

#### Bottom Half (workqueue)

> **Can take time** (milliseconds OK)

```
Do processing:
    1. Process packet data
    2. Complex calculations
    3. Call other kernel functions
    4. Wake up processes
    
Runs later, when CPU available!
```

---

## 5. Workqueues (workqueue.c)

### The Problem

Sometimes kernel needs to do work, but not NOW.

**Scenario:**

```
Interrupt handler:
    Got network packet
    Can't process now (too slow!)
    Need to defer...
    
Solution: Workqueues!
```

---

### What Are Workqueues?

Workqueue = List of tasks to do later.

**Think of it like a TODO list:**

```
Interrupt handler:
    "Add to TODO: Process network packet"
    Returns immediately
    
Later, when CPU free:
    Worker thread: "Let me check TODO list"
    "Ah, process network packet!"
    Does the work
```

---

### How Workqueues Work

#### Step 1: Queue Work

```c
// Interrupt handler:
Create work item:
    work->function = process_packet
    work->data = packet_address
    
Add to queue:
    workqueue_add(work)
    
Return from interrupt!
```

---

#### Step 2: Worker Thread Processes

```
Kernel has worker threads:
    Always running in background
    State: TASK_INTERRUPTIBLE (sleeping)
    
When work queued:
    Wake worker thread!
    thread->state = TASK_RUNNING
    
Worker thread:
    1. Get work from queue
    2. Call: work->function(work->data)
    3. Function processes packet
    4. Remove from queue
    5. Go back to sleep if queue empty
```

---

### Why Workqueues?

**Advantages:**

- ✅ Interrupts stay fast (top half only)
- ✅ Complex work done in process context
- ✅ Can sleep (wait for locks, I/O)
- ✅ Can be scheduled (not blocking interrupts)

**Without workqueues:**

```
Interrupt handler does everything:
    Process packet (10 milliseconds)
    While processing:
        Other interrupts wait! ❌
        Timer interrupt delayed! ❌
        System time wrong! ❌
        Mouse/keyboard frozen! ❌
```

---

### Types of Deferred Work

#### 1. Softirqs (fastest)

```
Very time-critical:
    Network packet processing
    Block device completion
    
Runs: In interrupt context
Can't sleep!
Very fast!
```

---

#### 2. Tasklets (medium)

```
Time-sensitive:
    Hardware-specific processing
    
Runs: In interrupt context
Can't sleep!
Medium speed
```

---

#### 3. Workqueues (slowest but flexible)

```
Can take time:
    File operations
    Memory allocation
    Complex processing
    
Runs: In process context
CAN sleep! ✅
Most flexible!
```

---

## Summary

### Complete kernel/ Overview

This section covered 8 core topics:

| Component | Location | Purpose |
|-----------|----------|---------|
| **Scheduler** | `sched/` | Timer interrupts, context switching, fair CPU sharing |
| **Fork** | `fork.c` | Process creation, copy-on-write, parent-child relationship |
| **Signals** | `signal.c` | Asynchronous notifications (CTRL+C, kill, etc.) |
| **Exit** | `exit.c` | Process termination, resource cleanup, zombie/orphan handling |
| **System Calls** | `sys.c` | User→Kernel bridge, privilege switching, 300+ syscalls |
| **Time** | `time/` | Wall clock, process CPU time, sleep/timers, high-
