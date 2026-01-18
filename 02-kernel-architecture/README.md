# Linux Kernel Internals 

> Ever wondered how your computer runs hundreds of programs simultaneously? How pressing CTRL+C actually stops a program? How the kernel knows what time it is? 

**You're about to find out!**

---

## What's Inside This Repository?

> This repository explores the **kernel/** directory - the **HEART of Linux** where all the magic happens! 

Here's what we cover:

```
kernel/
├── sched/          ← Process scheduler (WHO runs WHEN)
├── fork.c          ← Creating new processes  
├── exit.c          ← Process termination
├── signal.c        ← Signals (kill, CTRL+C, etc.)
├── sys.c           ← System call implementations
├── time/           ← Timers and clocks
├── irq/            ← Interrupt handling
└── workqueue.c     ← Deferred work
```

**This directory controls:**
- When programs run (scheduler)
- Creating new programs (fork)
- Communication between programs (signals)
- Time management (timers)
- Hardware events (interrupts)
- Deferred work execution (workqueues)
- Clean program exits (termination)
- User-kernel bridge (system calls)

---

## What You'll Learn

Each mechanism is explained with:

- **Physical hardware interactions** - What actually happens on the CPU
- **Register-level details** - RIP, RAX, CR3, and more
- **Step-by-step execution flows** - From start to finish
- **Design trade-offs** - Why Linux does things this way
- **Real-world examples** - Concrete scenarios you can relate to

**No code, pure theory!** Build a solid mental model before diving into implementation.

---

## How to Use This Guide

**Read sequentially** - Each topic builds on previous concepts:
1. Start with the Scheduler (understand CPU sharing)
2. Move to Process Creation (how programs are born)
3. Continue through all 8 mechanisms
4. Refer back as needed while studying kernel source code

**Prerequisites:**
- Basic C programming
- Computer architecture fundamentals (CPU, RAM, registers)
- Linux user-space familiarity

---

# 1. Process Scheduler

> **How 200+ programs share 4 CPU cores fairly**

## The Fundamental Problem

**Physical Reality:**

```
Your Computer:
├── CPU: 4 cores (4 execution units)
└── Programs Running: 200+ processes
```

**Question:** Who runs when?

---

## Without Scheduler

```
First 4 programs load:
├── Program A → CPU 0
├── Program B → CPU 1
├── Program C → CPU 2
└── Program D → CPU 3

They run FOREVER!

Programs E, F, G... Z: Never execute! 
System frozen on first 4 programs!
```

---

## The Timer Interrupt Solution

### **Physical Hardware Setup**

```
Motherboard Components:
└── PIT (Programmable Interval Timer)
    ├── Hardware: 8254 chip or APIC timer
    ├── Function: Generates electrical signal every 10ms
    └── Output: Signal → CPU interrupt pin
```

### **What Happens Physically**

```
T = 0ms: Program A running on CPU 0
    ├── Fetch from address 0x400000
    ├── Decode: MOV instruction
    └── Execute: Move data to register
    
T = 10ms: Timer chip sends INTERRUPT!
    ↓
    Physical interrupt pin gets voltage
    ↓
    CPU MUST respond (hardware requirement)
    ↓
    CPU STOPS Program A immediately
    ↓
    CPU saves Program A's state
    ↓
    CPU jumps to kernel's interrupt handler
    ↓
    Now KERNEL runs (scheduler code)
```

> **Key Insight:** Timer interrupt is PHYSICAL HARDWARE forcing CPU to stop current program!

---

## Scheduler Workflow

### **STEP 1: Save Current Program State**

```
Program A was running, CPU had:
├── RIP register: 0x400000 (instruction pointer)
├── RAX register: 5 (some value)
├── RBX register: 3 (some value)
├── RSP register: 0x7FFF000 (stack pointer)
└── ... (all other registers)

Scheduler saves ALL registers into task_struct
(Think of task_struct like a SAVE FILE in video game!)
```

**Physical Location in RAM:**

```
Address 0x02000000: Program A's task_struct
    ├── Saved RIP: 0x400000
    ├── Saved RAX: 5
    ├── Saved RBX: 3
    ├── Saved RSP: 0x7FFF000
    ├── State: RUNNING
    ├── CPU time used: 100ms
    └── Priority: 120
    
Address 0x02001000: Program B's task_struct  
    ├── Saved RIP: 0x500000
    ├── Saved RAX: 10
    ├── State: READY (waiting to run)
    ├── CPU time used: 50ms
    └── Priority: 120
```

---

### **STEP 2: Decide Next Program (CFS Algorithm)**

**Goal:** Give each program FAIR share of CPU time

```
Scheduler examines all programs:
├── Program A: Used 100ms CPU time
├── Program B: Used 50ms CPU time
├── Program C: Used 75ms CPU time
└── Program D: Used 60ms CPU time

CFS Logic: Pick program with LEAST CPU time
→ Program B (only used 50ms)
→ Give it a turn to "catch up"
```

**Why Fair?**

```
Over time, CPU usage balances:
├── Program A: 100ms → 110ms → 120ms...
├── Program B: 50ms → 60ms → 70ms... ← Catching up!
├── Program C: 75ms → 85ms → 95ms...
└── Program D: 60ms → 70ms → 80ms...

Eventually all programs have similar total time!
Nobody starves! 
```

---

### **STEP 3: Context Switch to Next Program**

**Physical Steps:**

```
1. Load Program B's page table:
   └── CR3 register = 0x00600000 (Program B's page table)
       (MMU now translates using Program B's mappings!)

2. Restore Program B's registers:
   ├── RIP ← 0x500000 (where Program B was)
   ├── RAX ← 10
   ├── RBX ← 7
   └── RSP ← 0x7FFE000

3. Jump to Program B:
   └── CPU's RIP = 0x500000
       CPU starts fetching from this address
       Program B continues executing!
```

> Program B thinks: "I never stopped!"  
> (But it was paused for 10ms!)

---

### **STEP 4: Repeat Forever**

```
Timeline:
T = 0ms:     Program A runs (10ms)
T = 10ms:    Timer interrupt → Switch to B
T = 20ms:    Timer interrupt → Switch to C  
T = 30ms:    Timer interrupt → Switch to D
T = 40ms:    Timer interrupt → Back to A
T = 50ms:    Timer interrupt → B again
...forever...

Each program gets:
├── 10ms time slice
├── Then must wait
└── Then gets another 10ms
```

**Over 1 second:**
- Each program gets ~100 time slices (10ms × 100 = 1 second)
- Feels like running continuously! 

---

## Why 10 Milliseconds?

### **The Trade-off Analysis**

**SHORTER time slices (1ms):**
```
More responsive (programs switch faster)
More overhead (spend time switching, not working)

Example:
├── Context switch takes: 0.1ms
├── Useful work: 1ms - 0.1ms = 0.9ms
└── Overhead: 10%! 
```

**LONGER time slices (100ms):**
```
Less overhead (more time working)
Less responsive (wait longer for turn)

Example: Mouse click
├── Program handling mouse is waiting
├── Might wait up to 100ms for its turn
└── Feels laggy! 
```

**GOLDILOCKS (10ms):**
```
Responsive enough (max 10ms wait)
Low overhead (~1%)
Just right! 
```

---

## Priority System

**Not all programs are equal!**

### **Real-time Audio Player**
```
├── Must process sound every 10ms
├── Can't wait 100ms (sound would skip!)
└── Needs HIGH priority
```

### **Background File Indexer**
```
├── Searching your files
├── Not urgent
└── Can use LOW priority
```

### **How Priority Works**

```
Normal priority: Gets 10ms every 40ms
└── 25% CPU time

High priority: Gets 10ms every 20ms  
└── 50% CPU time (more turns!)

Low priority: Gets 10ms every 100ms
└── 10% CPU time (fewer turns)
```

> System stays responsive!  
> Important stuff runs more!

---

# 2. Process Creation (fork)

> **How new programs are born in Linux**


## The Problem

**When you double-click Firefox icon:**

```
Questions:
├── How does Firefox program start running?
├── Where does its code come from?
├── How does it get memory?
└── How does scheduler know about it?
```

---

## The Fork Mechanism

**Why called "fork"?**
> Process SPLITS like a tree branch! 
> One process becomes TWO!

---

## Physical Process Flow

### **STEP 1: Parent Program Calls fork()**

**Scenario:** bash shell wants to run Firefox

```
bash shell running:
├── PID: 1000 (process ID)
├── Memory: 0x80000000 - 0x90000000
└── Code: bash executable
    
bash executes: fork() system call
```

---

### **STEP 2: Kernel Creates Copy**

**Physical Duplication:**

```
BEFORE fork():
RAM:
└── 0x80000000: bash code & data
    
AFTER fork():
RAM:
├── 0x80000000: bash code & data (parent)
└── 0xA0000000: bash code & data (child) ← COPIED!

Two IDENTICAL processes now exist!
├── Both have SAME code
└── Both have SAME memory contents
```

---

## Copy-on-Write Optimization

**Problem:** Full copying is wasteful!

**Modern Optimization:**

```
Don't actually copy yet!
Both processes SHARE same physical memory!

Page tables point to SAME physical pages:
├── Parent's page table: Virtual 0x80000000 → Physical 0x10000000
└── Child's page table:  Virtual 0x80000000 → Physical 0x10000000
                                              ↑ SAME!

Mark pages as READ-ONLY
```

**When either tries to WRITE:**

```
1. MMU generates Page Fault
2. Kernel copies the page NOW
3. Update page tables
4. Both can write independently

Copy happens ONLY when needed! 
Saves memory and time!
```

---

### **STEP 3: Give Child Unique Identity**

**Child process gets:**

```
├── NEW PID (e.g., 2000)
├── Own task_struct
├── Own register state
├── Points to parent (PPID = 1000)
└── Starts with SAME code as parent
```

---

### **STEP 4: Return from fork() - The Magic Trick!**

> **fork() returns TWICE!** 

```
In parent process:
└── fork() returns child's PID (2000)
    
In child process:
└── fork() returns 0
```

**Why?** So processes can tell who they are!

```
Parent knows: "I got PID 2000, that's my child"
Child knows: "I got 0, I'm the child"
```

**Physical Mechanism:**

```
When fork completes, TWO task_structs exist:

task_struct[1000] (parent):
├── RIP: 0x400500 (after fork call)
├── RAX: 2000 (child's PID)
└── State: RUNNING
    
task_struct[2000] (child):
├── RIP: 0x400500 (SAME instruction!)
├── RAX: 0 (return value for child)
└── State: READY (waiting to run)

Next scheduler tick:
└── Child gets its turn!
    Continues from SAME point as parent!
    But sees different return value!
```

---

### **STEP 5: Child Runs Different Program (exec)**

**Problem:** After fork, child still running bash code!  
**Solution:** Child calls exec("firefox")

```
exec() does:
├── 1. Throw away current code (bash)
├── 2. Load new code (firefox) from disk
├── 3. Setup new memory
└── 4. Jump to firefox's main()
    
Now child running Firefox! 
Parent (bash) still running!
```

---

## Complete Example: Starting Firefox

```
T = 0: bash shell running (PID 1000)
       └── User types: firefox
    
T = 1: bash calls fork()
       └── Kernel creates child (PID 2000)
           Copies memory (copy-on-write)
    
T = 2: fork() returns
       ├── Parent (bash): Gets return value 2000
       └── Child (bash copy): Gets return value 0
    
T = 3: Child checks return value
       └── Sees 0 → "I'm the child!"
           Calls exec("firefox")
    
T = 4: exec replaces child's memory
       ├── Throws away bash code
       ├── Loads firefox from /usr/bin/firefox
       └── Sets up firefox memory
    
T = 5: Child now running Firefox!
       ├── PID 2000 (was bash, now firefox)
       └── Parent still bash (PID 1000)
    
T = 6: Scheduler gives both turns
       ├── bash still in terminal
       └── firefox opens in new window
       
Both running simultaneously! 
```

---

## Why This Design?

### **Historical Reason**

```
Early Unix (1970s):
├── Fork was simple to implement!
└── Copy parent = Easy!
```

### **Modern Optimization**

```
Copy-on-write = Fast!
└── No wasted copying!
```

### **Alternative Approaches**

```
Windows: CreateProcess (different API)
Linux: fork + exec works well! 
```

---

# 3. Signals

> **Asynchronous notifications between kernel and processes**


## The Problem

**Scenario:** You press CTRL+C in terminal

```
Terminal running program (PID 5000)
    
Question: How does kernel tell program to stop?
    
├── Can't just kill it! (might have unsaved data!)
└── Need to ASK program to exit gracefully!
```

**Signals are the solution!**

---

## What Are Signals?

**Signals = Asynchronous notifications**

> Think of signal like: Tapping someone on shoulder!

```
Program running normally:
├── Executing its code
└── Doing its work
    
Kernel sends signal:
└── "Hey! User pressed CTRL+C!"
    
Program stops what it's doing:
├── Handles the signal
└── Then continues (or exits)
```

---

## How Signals Work

### **STEP 1: Signal is Sent**

**Example:** User presses CTRL+C

**Physical Events:**

```
1. Keyboard sends scancode to keyboard controller
2. Controller generates interrupt
3. Kernel's keyboard driver handles interrupt
4. Driver sees: CTRL+C pressed
5. Driver looks up: Terminal PID = 5000
6. Kernel marks: "Send SIGINT to PID 5000"
```

**Where Signal is Stored:**

```
Program's task_struct has field: pending_signals

Before CTRL+C:
└── task_struct[5000].pending_signals = 0000000000000000
    (binary, each bit = one signal)

After CTRL+C:
└── task_struct[5000].pending_signals = 0000000000000010
                                                       ↑
                                        Bit 2 = SIGINT (signal 2)

Signal is "pending" (waiting to be delivered)!
```

---

### **STEP 2: Scheduler Delivers Signal**

**Next time Program 5000 gets CPU:**

```
Scheduler checks: "Any pending signals?"
└── Sees: SIGINT pending!
    
Before returning to program:
└── Scheduler interrupts the program
    Forces program to handle signal FIRST!
```

**Physical Delivery:**

```
Program 5000 was running:
└── RIP: 0x400000 (middle of some function)
    
Scheduler changes RIP:
├── RIP: 0x300000 (signal handler function!)
└── Also saves old RIP: 0x400000 (to return later)
    
Program starts executing signal handler:
├── Not where it was!
└── Forced to handle signal!
```

---

### **STEP 3: Program Handles Signal**

**Default Handlers:**

```
Each signal has default action:

├── SIGINT (CTRL+C): Terminate program
├── SIGTERM: Terminate program
├── SIGKILL: Force terminate (can't catch!)
├── SIGSTOP: Pause program
├── SIGCONT: Resume program
└── SIGSEGV: Segmentation fault (program crash)
```

**Custom Handlers:**

```
Program can say: "When I get SIGINT, do THIS instead!"

Example: Text editor
├── Normal: SIGINT kills program
└── Custom: SIGINT → "Save file first, then exit"
    
Program registers handler:
└── "When SIGINT arrives, call my_function()"
    
Kernel remembers:
└── task_struct[5000].signal_handlers[SIGINT] = 0x300000
                                                 ↑
                                        Address of my_function
```

---

### **STEP 4: Return from Handler**

**After signal handler finishes:**

```
Program returns to where it was!

Kernel restores:
└── RIP: 0x400000 (original location)
    
Program continues:
└── Like nothing happened!
    (Except it handled the signal)
```

---

## Special Signals

### **SIGKILL (Signal 9) - The Unkillable**

```
Cannot be caught!
Cannot be ignored!
ALWAYS kills program!

Why?
├── What if program hangs in infinite loop?
├── Custom handler won't execute! (loop never ends)
└── SIGKILL goes around the program:
    ├── Kernel forcibly terminates
    ├── No handler involved
    └── Program dies immediately! 

Usage: kill -9 PID
```

---

### **SIGSEGV (Segmentation Fault)**

**Sent when program accesses invalid memory!**

```
Example:
├── Program tries: *ptr = 5;
├── ptr = NULL (invalid!)
└── Flow:
    ├── MMU generates Page Fault
    ├── Kernel sees: Invalid access!
    ├── Kernel sends: SIGSEGV to program
    └── Default action: Core dump + terminate

Program can catch SIGSEGV:
├── Debug the error
├── Log information
└── Exit gracefully
```

---

## Why Signals Exist

### **Coordinated Shutdown**

```
Without signals:
├── Need to kill program → Just terminate it
└── Problem: Unsaved data lost! 
    
With signals:
├── Send SIGTERM → Program saves data
└── Program exits cleanly 
```

### **Inter-process Communication**

```
Parent process → Child process:
└── Send signal to communicate!
    
Example: Web server + worker
├── Worker taking too long?
└── Server sends: SIGALRM (timeout signal)
    Worker responds: Speed up or exit
```

---

## Common Signals Reference

| Signal | Number | Default Action | Can Catch? |
|--------|--------|----------------|------------|
| SIGHUP | 1 | Terminate | Yes |
| SIGINT | 2 | Terminate | Yes |
| SIGQUIT | 3 | Core dump | Yes |
| SIGKILL | 9 | Terminate | **NO** |
| SIGSEGV | 11 | Core dump | Yes |
| SIGTERM | 15 | Terminate | Yes |
| SIGSTOP | 19 | Pause | **NO** |
| SIGCONT | 18 | Resume | Yes |

---

# 4. Process Termination

> **How programs end and clean up resources**

## The Problem

**Firefox running (PID 5000):**

```
├── Occupies RAM (100 MB)
├── Has open files (config, cache)
├── Has network connections (downloading)
└── Scheduler giving it CPU time

User clicks X button (close Firefox)
    
Questions:
├── How is memory freed?
├── What about open files?
├── Network connections?
└── Does it just disappear?
```

**The exit mechanism handles ALL cleanup!**

---

## How Programs Exit

### **STEP 1: Program Calls exit()**

**Normal program flow:**

```
Program's main() function:
├── 1. Do work
├── 2. Finish work
└── 3. Return 0 (success)
    
When main() returns:
├── C library calls: exit(0)
├── exit() makes system call to kernel
└── Kernel's exit handler starts cleanup!
```

**Or: Forced Exit**

```
Program crashes:
├── Segmentation fault → Kernel sends SIGSEGV
└── Default handler → Calls exit(1)
    
User kills program:
├── kill 5000 → Kernel sends SIGTERM
└── Default handler → Calls exit(0)
    
All paths lead to exit!
```

---

### **STEP 2: Close All Open Files**

**Physical Cleanup:**

```
Program had open files:

task_struct[5000].open_files:
├── FD 0: stdin (terminal)
├── FD 1: stdout (terminal)
├── FD 2: stderr (terminal)
├── FD 3: /home/user/document.txt
├── FD 4: /tmp/cache.dat
└── FD 5: socket (network connection)
```

**Exit handler walks through ALL file descriptors:**

```
For each open file:
├── 1. Flush buffers (write pending data to disk)
├── 2. Update file metadata (last modified time)
├── 3. Release locks (if file was locked)
├── 4. Close file descriptor
└── 5. Free kernel structures
```

**Why Important?**

```
If program just vanished:
├── Unsaved data lost!
├── File locks held forever! 
└── Disk corruption possible!
    
Exit ensures:
├── Data written to disk 
├── Locks released 
└── File system consistent 
```

---

### **STEP 3: Free All Memory**

**Physical Memory Release:**

```
Program owned memory regions:

Virtual memory (program's view):
├── 0x00400000 - 0x00500000: Program code (1 MB)
├── 0x00600000 - 0x10000000: Heap (100 MB)
└── 0x7FFF0000 - 0x80000000: Stack (1 MB)

Physical memory (actual RAM):
└── Physical pages backing these virtual addresses
```

**Exit handler:**

```
1. Walk program's page table
2. For each virtual page:
   ├── Find physical page it maps to
   ├── Mark physical page as FREE
   └── Remove page table entry
3. Free the page table itself
4. Free task_struct
```

**Memory reclaimed:**

```
BEFORE exit:
└── Used RAM: 14 GB / 16 GB
    
AFTER exit (freed 100 MB):
└── Used RAM: 13.9 GB / 16 GB
    
That 100 MB now available for other programs!
```

---

### **STEP 4: Notify Parent Process**

**The parent-child relationship:**

```
Remember fork?
└── Parent (bash, PID 1000) created child (firefox, PID 5000)
    
When child exits:
└── Parent might want to know!
    ├── Did child succeed? (exit code 0)
    ├── Did child fail? (exit code 1)
    └── Did child crash? (exit code 139 = segfault)
```

**Physical notification:**

```
Child exits:
1. Kernel sets child's state: ZOMBIE
   (Yes, really called zombie!)

2. Kernel saves exit code in task_struct:
   └── task_struct[5000].exit_code = 0

3. Kernel sends signal to parent:
   └── SIGCHLD → Parent PID 1000

4. Child becomes zombie:
   ├── Most resources freed
   ├── But task_struct remains
   └── Waiting for parent to collect exit code
```

**Why zombie state?**

```
Parent might want exit code!

Parent calls: wait() or waitpid()
└── Kernel returns child's exit code
    Parent knows: "My child exited with code 0"
    
After parent collects exit code:
└── Kernel fully removes child's task_struct
    Zombie gone! 

If parent never collects:
└── Zombie stays forever! 
    (Wasting small amount of memory)
```

---

### **STEP 5: Orphan Handling**

**Special case: Parent dies first!**

```
Scenario:
├── Parent: bash (PID 1000)
└── Child: firefox (PID 5000)
    
User kills bash!
└── bash exits → firefox becomes ORPHAN!
    
Question: Who's firefox's parent now?
```

**Re-parenting to init:**

```
When parent dies:
├── Kernel finds all children
└── Changes their PPID (parent PID)
    
firefox: PPID was 1000 → Now PPID = 1
    
PID 1 = init process (first process)
└── init ADOPTS all orphans!
    
init's job:
├── Call wait() periodically
├── Collect exit codes from adopted children
└── Prevent zombie accumulation 
```

---

### **STEP 6: Final Removal**

**After all cleanup:**

```
Kernel removes from scheduler:
├── task_struct[5000] deleted
└── No longer in scheduler's list
    
CPU never schedules this PID again!

Program completely gone! 
```

---

## Exit Codes

**Exit code = Number program returns when it exits**

**Standard meanings:**

```
├── 0: Success! Everything worked 
├── 1: General error 
├── 2: Misuse of command
├── 126: Command cannot execute
├── 127: Command not found
└── 128+N: Killed by signal N
    ├── 130: Killed by SIGINT (CTRL+C)
    └── 139: Killed by SIGSEGV (segfault)
```

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

## Why Careful Exit Matters

**Database example:**

```
Database program:
├── Has uncommitted transaction in memory
├── Writing to disk
└── Holding locks
    
If killed abruptly (power loss):
├── Transaction lost 
├── Disk corruption 
└── Database broken 
    
If exits properly (SIGTERM):
├── Commits transaction 
├── Syncs disk 
├── Releases locks 
└── Database safe 
```

**This is why:**

```
kill (SIGTERM): Polite "please exit"
kill -9 (SIGKILL): Force kill (emergency only!)

Always try kill first!
Only use kill -9 if program frozen!
```

---

# 5. System Calls

> **The bridge between user programs and kernel**

## The Problem

**Remember the separation:**

```
Ring 3 (User programs):
├── Cannot access hardware
├── Cannot access other program's memory
└── Cannot do privileged operations
    
Ring 0 (Kernel):
├── Can access hardware
├── Can access all memory
└── Can do anything

Question: How do user programs ASK kernel for help?
```

**System calls are the bridge!**

---

## What Are System Calls?

**System call = Controlled entry into kernel**

> Think of it like: Airport security checkpoint!

```
User space = Public area (can't board plane)
Kernel space = Secured area (can board plane)

System call = Going through security
├── Checked and authorized
├── Can't bring weapons (malicious code)
└── Can enter secured area
```

---

## How System Calls Work

### **STEP 1: User Program Invokes System Call**

**Program wants to read file:**

```
Program code calls: read(fd, buffer, size)

What read() actually is:
├── NOT a normal function!
└── It's a WRAPPER that makes system call!
```

**Physical mechanism:**

```
read() function prepares:
├── 1. System call number (read = 0)
├── 2. Arguments (fd, buffer, size)
└── 3. Executes special CPU instruction: syscall
```

---

### **STEP 2: CPU Switches to Kernel Mode**

**The syscall instruction (CPU hardware feature):**

```
BEFORE syscall:
├── CPL (Current Privilege Level) = 3 (user mode)
├── RIP = 0x400500 (user program code)
├── CS register = 0x0033 (user code segment)
└── SS register = 0x002B (user stack segment)

CPU does (in hardware!):
1. Check: Is syscall allowed? (yes, always)
2. Save current state:
   ├── Save user RIP to RCX register
   └── Save user RFLAGS to R11 register
3. Switch privilege:
   ├── CPL = 0 (kernel mode)
   ├── CS = 0x0010 (kernel code segment)
   └── SS = 0x0018 (kernel stack segment)
4. Load kernel entry point:
   └── RIP = 0x00001000 (from MSR register)
        
AFTER syscall:
├── CPL = 0 (kernel mode!)
└── RIP = 0x00001000 (kernel's syscall entry!)
    
CPU now executing KERNEL code!
Hardware enforced switch! 
```

> **Key insight:** CPU hardware changes privilege level!

---

### **STEP 3: Kernel Entry Point Receives Call**

**Physical flow in kernel:**

```
CPU jumped to: 0x00001000 (kernel's entry point)

Entry point does:
1. Save all user registers:
   └── Push RAX, RBX, RCX, RDX... onto kernel stack
       (Preserve user state!)

2. Switch to kernel stack:
   ├── User stack: 0x7FFF000 → Can't trust it!
   └── Kernel stack: 0x01FF000 → Safe kernel memory

3. Look up system call number:
   └── RAX register = 0 (read syscall)

4. Call appropriate handler:
   └── sys_call_table[0] → sys_read()
       Jump to sys_read() function!
```

---

### **STEP 4: Kernel Executes Requested Operation**

**sys_read() does the actual work:**

```
sys_read(fd, buffer, size):
    
1. Validate arguments:
   ├── Is fd valid? (check open file table)
   ├── Is buffer valid? (check page tables)
   └── Is size reasonable? (not crazy big)
        
2. Security check:
   ├── Does process have permission?
   └── Is file readable?
        
3. Perform operation:
   ├── If data in cache: Copy from cache
   ├── If not: Read from disk (via driver)
   └── Copy data to user's buffer
        
4. Return result:
   ├── RAX = number of bytes read
   └── Or RAX = -1 (error)
```

---

### **STEP 5: Return to User Space**

**The sysret instruction (reverse of syscall):**

```
Kernel finished work!

Kernel executes: sysret instruction

CPU does (in hardware!):
1. Restore user state:
   ├── RIP = RCX (saved user RIP)
   └── RFLAGS = R11 (saved flags)
2. Switch privilege:
   ├── CPL = 3 (back to user mode)
   ├── CS = 0x0033 (user code segment)
   └── SS = 0x002B (user stack segment)
3. Resume user program:
   └── RIP = 0x400500 (where program was)

AFTER sysret:
├── CPL = 3 (user mode again)
├── RIP = 0x400500 (user program)
└── RAX = 1024 (bytes read)
    
Program continues:
└── "read() returned 1024, success!"
```

---

## The System Call Table

**How kernel knows which function to call:**

```
Kernel has table: sys_call_table[]

Index = System call number
Value = Function pointer

Example table (simplified):
├── sys_call_table[0] = sys_read
├── sys_call_table[1] = sys_write
├── sys_call_table[2] = sys_open
├── sys_call_table[3] = sys_close
├── sys_call_table[4] = sys_stat
├── sys_call_table[5] = sys_fstat
└── ...
    sys_call_table[300+] = ... (300+ system calls!)

When user calls read():
├── RAX = 0 (read's number)
├── Kernel looks up: sys_call_table[0]
└── Calls: sys_read()
```

---

## Common System Calls

**File operations:**
```
├── open(), close(), read(), write()
└── lseek(), stat(), mkdir(), rmdir()
```

**Process operations:**
```
├── fork(), exec(), exit(), wait()
└── kill(), getpid(), getppid()
```

**Memory operations:**
```
└── brk(), mmap(), munmap()
```

**Network operations:**
```
├── socket(), bind(), listen(), accept()
└── send(), recv(), connect()
```

**Time operations:**
```
└── time(), gettimeofday(), sleep()
```

> **... 300+ total system calls!**

---

## Why System Calls Are Slow

**The overhead:**

```
Normal function call:
├── CALL instruction (~1 nanosecond)
├── Function executes
├── RET instruction (~1 nanosecond)
└── Total: ~2 nanoseconds 

System call:
├── syscall instruction (~100 nanoseconds)
├── Save registers (~20 nanoseconds)
├── Switch stacks (~10 nanoseconds)
├── Validate arguments (~50 nanoseconds)
├── Do work (~variable)
├── Restore registers (~20 nanoseconds)
├── sysret instruction (~100 nanoseconds)
└── Total: ~300+ nanoseconds 
    
150× slower than function call!
```

**Why so slow?**

```
Security checks!
├── Validate every pointer
├── Check permissions
└── Verify arguments
    
Privilege switching!
├── Save/restore state
├── Change CPU mode
└── Switch stacks
    
All necessary for protection! 
But costs performance! 
```

---

## Optimization: vDSO

**Some system calls don't need kernel!**

```
Example: gettimeofday()
└── Returns current time
    
Old way:
├── System call → Enter kernel
├── Kernel reads clock
├── Return to user
└── 300 nanoseconds 
    
New way (vDSO):
├── Kernel maps read-only page to user space
├── Page contains: Current time (updated by kernel)
├── User reads directly: No system call!
└── 1 nanosecond 
    
300× faster!
```

**Other vDSO calls:**
```
├── gettimeofday() - Read time
├── clock_gettime() - High-res time
└── getcpu() - Which CPU am I on?

All readable without system call!
```

---

# 6. Time Management

> **How the kernel tracks and manages time**

## The Problem

**Kernel needs to track time for multiple purposes:**

```
1. Real-world time:
   └── "What time is it?" (12:30:45 PM)
    
2. Process CPU time:
   └── "How long did this program run?"
    
3. Sleep/timers:
   └── "Wake me up in 5 seconds"
    
4. Scheduler timing:
   └── "10ms time slice expired!"
    
All different time concepts!
```

---

## Time Sources (Physical Hardware)

### **1. RTC (Real-Time Clock)**

```
Physical chip on motherboard:
├── Battery-powered (keeps time when PC off!)
├── Low resolution (1 second granularity)
└── Used for: Wall clock time
    
Location: CMOS chip (same as BIOS settings)

Kernel reads RTC at boot:
├── Read: "2024-01-16 10:30:00"
└── Kernel now knows wall clock time!
```

---

### **2. PIT (Programmable Interval Timer)**

```
8254 timer chip:
├── Generates interrupts at fixed rate
├── Programmable frequency (100 Hz typical)
└── Used for: Scheduler ticks
    
Every 10ms (100 Hz):
├── Timer fires interrupt
├── Kernel updates time
└── Scheduler runs
```

---

### **3. TSC (Time Stamp Counter)**

```
CPU internal counter:
├── Increments every CPU cycle
├── 3 GHz CPU = 3 billion ticks/second
├── Very high resolution!
└── Used for: Precise timing
    
Read TSC:
├── Special CPU instruction: RDTSC
└── Returns: Number of cycles since boot
    
Convert to time:
└── Cycles / CPU_frequency = Time in seconds
```

---

### **4. HPET (High Precision Event Timer)**

```
Modern timer chip:
├── Higher resolution than PIT
├── Multiple independent timers
└── Used for: High-res timing
    
Resolution: 10 MHz (100 nanosecond precision)
Much better than PIT!
```

---

## How Kernel Maintains Time

### **The jiffies counter:**

```
Kernel has global variable: jiffies

Every timer interrupt (10ms):
└── jiffies++
    
Boot time: jiffies = 0
After 1 second: jiffies = 100 (100 ticks @ 100 Hz)
After 1 hour: jiffies = 360,000
After 1 day: jiffies = 8,640,000

Current time = Boot time + (jiffies / HZ)
```

**Example:**

```
Boot time: January 16, 2024 08:00:00
jiffies: 360,000 (1 hour worth)
HZ: 100 (100 ticks per second)

Current time = 08:00:00 + (360,000 / 100 seconds)
             = 08:00:00 + 3,600 seconds
             = 09:00:00
             
Current time: January 16, 2024 09:00:00
```

---

## Process CPU Time Tracking

**Every process has CPU time counters:**

```
task_struct has fields:
├── utime: User mode time (nanoseconds)
└── stime: System mode time (nanoseconds)
    
Every timer tick (10ms):
├── If process in user mode:
│   └── process->utime += 10,000,000 ns
└── If process in kernel mode (system call):
    └── process->stime += 10,000,000 ns
```

**The time command uses this:**

```bash
$ time firefox

real    2m30.500s    ← Wall clock time (stopwatch)
user    1m45.200s    ← CPU time in user mode
sys     0m12.800s    ← CPU time in kernel mode

Notice: user + sys < real
Why? Process was WAITING (I/O, sleeping)!
```

---

## Sleep and Timers

### **How sleep() works:**

**Program calls: sleep(5)  // Sleep 5 seconds**

```
Physical mechanism:

1. System call to kernel

2. Kernel calculates wake time:
   ├── Current jiffies: 100,000
   ├── Sleep duration: 5 seconds = 500 jiffies
   └── Wake time: 100,500 jiffies
    
3. Kernel changes process state:
   ├── task_struct[PID].state = TASK_INTERRUPTIBLE
   └── task_struct[PID].wake_time = 100,500
    
4. Kernel adds to timer list:
   └── "Wake PID 5000 at jiffies 100,500"
    
5. Process removed from scheduler:
   └── Not eligible to run!
    
6. Every timer tick:
   ├── Kernel checks: jiffies == 100,500?
   ├── NO → Keep sleeping
   └── YES → Wake process!
       ├── task_struct[5000].state = TASK_RUNNING
       └── Add back to scheduler
        
7. Process resumes execution!
```

---

### **Why INTERRUPTIBLE?**

```
TASK_INTERRUPTIBLE:
├── Sleep can be interrupted by signal
└── CTRL+C wakes process early
    
TASK_UNINTERRUPTIBLE:
├── Sleep CANNOT be interrupted
├── Must wait full duration
└── Used for critical I/O
```

---

## High-Resolution Timers

**For precise timing needs:**

```
Old way (jiffies):
├── Resolution: 10ms (HZ = 100)
└── sleep(0.001) → Actually sleeps 10ms! 
    
New way (hrtimer):
├── Resolution: nanoseconds!
├── Uses TSC or HPET
└── sleep(0.001) → Sleeps 1ms! 

Used for:
├── Multimedia (audio/video)
├── Real-time applications
└── Precise measurements
```

---

# 7. Interrupt Handling

> **How hardware gets the kernel's attention**


## The Problem

**Hardware needs attention NOW!**

```
Scenario: Network packet arrives

Option 1 (Polling):
├── CPU constantly checks: "Packet arrived yet?"
├── Check, check, check, check...
└── Wastes CPU time! 
    
Option 2 (Interrupts):
├── Network card sends signal: "PACKET HERE!"
├── CPU immediately responds
└── Efficient! 
```

---

## What Are Interrupts?

**Interrupt = Electrical signal to CPU**

```
Physical hardware:

Network card has pin → CPU's INTR pin
    
When packet arrives:
├── Network card sets pin HIGH (voltage)
├── CPU detects voltage change
└── CPU MUST respond (hardware requirement!)
```

---

## How Interrupts Work

### **STEP 1: Hardware Generates Interrupt**

**Network card receives packet:**

```
1. Packet enters network card
2. Card stores packet in its memory
3. Card sets interrupt pin HIGH
4. Electrical signal travels to CPU
```

---

### **STEP 2: CPU Detects Interrupt**

```
CPU executing normal code:
├── Fetch instruction from 0x400000
├── Decode: ADD
└── Execute: Add registers
    
DURING execution:
├── CPU checks interrupt pin (every cycle!)
└── Detects: Pin is HIGH!
    
CPU immediately:
├── 1. Finishes current instruction
├── 2. Stops normal execution
└── 3. Prepares to handle interrupt
```

---

### **STEP 3: CPU Identifies Interrupt Source**

**The interrupt vector:**

```
CPU asks: "Which device interrupted?"

Interrupt controller (PIC/APIC) responds:
└── "Interrupt number 50" (network card's number)

CPU looks up in IDT (Interrupt Descriptor Table):
└── IDT[50] = 0x00002000 (handler address)
    
CPU knows: Jump to 0x00002000 to handle this!
```

---

### **STEP 4: Save Current State**

**Before handling interrupt:**

```
CPU saves on stack:
├── RIP (where program was)
├── RFLAGS (CPU flags)
├── CS (code segment)
└── Error code (if applicable)
    
Why? To RESUME program later!

If interrupted program was in user mode:
├── Also save: RSP (user stack), SS (stack segment)
└── Switch to kernel stack!
```

---

### **STEP 5: Execute Interrupt Handler**

**CPU jumps to: 0x00002000 (network handler)**

```
Handler does:

1. Acknowledge interrupt:
   ├── Tell interrupt controller: "I got it!"
   └── Clear interrupt pin
    
2. Read data from device:
   ├── Network card has packet
   └── Copy packet to kernel memory
    
3. Process data:
   ├── Pass packet to network stack
   └── TCP/IP processing
    
4. Wake up waiting process:
   ├── If process was waiting for network data
   └── Change state: TASK_RUNNING
```

---

### **STEP 6: Return from Interrupt**

**Handler finishes!**

```
CPU executes: IRET (interrupt return)

CPU restores:
├── Pop RIP (resume address)
├── Pop RFLAGS
├── Pop CS
└── If was user mode: Pop RSP, SS
    
CPU continues:
├── Exactly where it was!
└── Like nothing happened!
    (Except interrupt was handled)
```

---

## Interrupt Types

### **1. Hardware Interrupts**

```
External devices:
├── IRQ 0: Timer (every 10ms)
├── IRQ 1: Keyboard (key pressed)
├── IRQ 3: Serial port
├── IRQ 4: Serial port
├── IRQ 6: Floppy disk
├── IRQ 8: RTC (real-time clock)
├── IRQ 12: Mouse
├── IRQ 14: Primary IDE (hard disk)
├── IRQ 15: Secondary IDE
└── Modern: 24+ IRQ lines (PCI devices)
```

---

### **2. Software Interrupts**

```
Triggered by CPU instructions:
├── INT 0x80: Old Linux system call
├── SYSCALL: Modern system call
└── INT 3: Debugger breakpoint
```

---

### **3. Exceptions**

```
CPU-generated:
├── Divide by zero
├── Invalid opcode
├── Page fault
├── General protection fault
└── Double fault
    
These are ERROR conditions!
```

---

## Interrupt Priorities

**Not all interrupts equal!**

```
High priority (must handle fast!):
├── Timer: Keeps system time accurate
└── Network: Packets arriving fast
    
Low priority (can wait):
└── Keyboard: Human types slowly
    
If multiple interrupts arrive:
├── Handle highest priority first!
└── Others wait in queue
```

---

## Interrupt Latency

**Time from interrupt to handling:**

```
Interrupt fires → Handler runs:
    
Latency = Time waiting to run

Low latency system:
├── < 10 microseconds
└── Real-time systems
    
High latency system:
├── > 1000 microseconds
└── Desktop systems OK
```

**Why latency matters:**

```
Audio playback:
├── Need samples every 10ms
└── High latency → Audio skips! 
    
Keyboard:
├── Human types ~100ms between keys
└── High latency OK 
```

---

## Top Half vs Bottom Half

**Problem: Interrupt handler must be FAST!**

```
If handler takes too long:
├── Other interrupts wait!
└── System becomes unresponsive!
```

**Solution: Split work!**

### **Top Half (in interrupt handler)**

```
MUST BE FAST! (<1 microsecond)

Do minimum:
├── 1. Acknowledge interrupt
├── 2. Read data from hardware
├── 3. Schedule bottom half
└── 4. Return IMMEDIATELY!
```

### **Bottom Half (workqueue)**

```
Can take time (milliseconds OK)

Do processing:
├── 1. Process packet data
├── 2. Complex calculations
├── 3. Call other kernel functions
└── 4. Wake up processes
    
Runs later, when CPU available!
```

---

# 8. Workqueues

> **Deferred work execution in the kernel**


## The Problem

**Sometimes kernel needs to do work, but not NOW:**

```
Interrupt handler:
├── Got network packet
├── Can't process now (too slow!)
└── Need to defer...
    
Solution: Workqueues!
```

---

## What Are Workqueues?

**Workqueue = List of tasks to do later**

> Think of it like: TODO list!

```
Interrupt handler:
├── "Add to TODO: Process network packet"
└── Returns immediately
    
Later, when CPU free:
├── Worker thread: "Let me check TODO list"
├── "Ah, process network packet!"
└── Does the work
```

---

## How Workqueues Work

### **STEP 1: Queue Work**

**Interrupt handler:**

```
Create work item:
├── work->function = process_packet
└── work->data = packet_address
        
Add to queue:
└── workqueue_add(work)
        
Return from interrupt!
```

---

### **STEP 2: Worker Thread Processes**

**Kernel has worker threads:**

```
Always running in background:
└── State: TASK_INTERRUPTIBLE (sleeping)
    
When work queued:
├── Wake worker thread!
└── thread->state = TASK_RUNNING
    
Worker thread:
├── 1. Get work from queue
├── 2. Call: work->function(work->data)
├── 3. Function processes packet
├── 4. Remove from queue
└── 5. Go back to sleep if queue empty
```

---

## Why Workqueues?

**Advantages:**

```
Interrupts stay fast (top half only)
Complex work done in process context
Can sleep (wait for locks, I/O)
Can be scheduled (not blocking interrupts)
```

**Without workqueues:**

```
Interrupt handler does everything:
├── Process packet (10 milliseconds)
└── While processing:
    ├── Other interrupts wait! 
    ├── Timer interrupt delayed! 
    ├── System time wrong! 
    └── Mouse/keyboard frozen! 
```

---

## Types of Deferred Work

### **1. Softirqs (fastest)**

```
Very time-critical:
├── Network packet processing
└── Block device completion
    
Runs: In interrupt context
Can't sleep!
Very fast!
```

---

### **2. Tasklets (medium)**

```
Time-sensitive:
└── Hardware-specific processing
    
Runs: In interrupt context
Can't sleep!
Medium speed
```

---

### **3. Workqueues (slowest but flexible)**

```
Can take time:
├── File operations
├── Memory allocation
└── Complex processing
    
Runs: In process context
CAN sleep! 
Most flexible!
```

---

## Complete Example: Network Packet

**Full flow from hardware to application:**

```
T = 0ms: Packet arrives at network card
    └── Card generates interrupt

T = 0.001ms: CPU handles interrupt (Top Half)
    ├── Acknowledge interrupt
    ├── Copy packet from card to memory
    ├── Queue work item: "Process this packet"
    └── Return from interrupt (FAST!)

T = 0.002ms: Scheduler gives turn to worker thread

T = 0.003ms: Worker processes packet (Bottom Half)
    ├── Parse Ethernet header
    ├── Parse IP header
    ├── Parse TCP header
    ├── Find socket waiting for this data
    ├── Copy data to socket buffer
    └── Wake up application process

T = 0.010ms: Scheduler gives turn to application

T = 0.011ms: Application reads data
    └── read() returns with packet data!

Total: 11 milliseconds from packet to app 
```

---

## Summary Table

| Mechanism | Context | Can Sleep? | Speed | Use Case |
|-----------|---------|------------|-------|----------|
| **Hardware IRQ** | Interrupt | NO | Fastest | Hardware acknowledgment |
| **Softirq** | Interrupt | NO | Very fast | Network RX/TX |
| **Tasklet** | Interrupt | NO | Fast | Device-specific |
| **Workqueue** | Process | YES | Flexible | Complex processing |

---

# Complete Summary

## What You've Mastered

You now understand the **8 fundamental mechanisms** that make Linux tick:

```
Scheduler      → Fair CPU time sharing with 10ms precision
Fork           → Efficient process creation with copy-on-write
Signals        → Asynchronous process communication
Exit           → Clean resource cleanup and zombie handling
System Calls   → Secure user-kernel transitions
Time           → Multiple time sources and high-res timers
Interrupts     → Fast hardware event handling
Workqueues     → Flexible deferred work execution
```

---

## The Big Picture

**These 8 mechanisms work together:**

```
User clicks Firefox icon
    ↓
Shell fork()s new process (Process Creation)
    ↓
Child exec()s Firefox binary (System Call)
    ↓
Scheduler gives Firefox CPU time (Scheduler)
    ↓
Firefox runs, handles SIGTERM gracefully (Signals)
    ↓
Network card interrupts for packets (Interrupts)
    ↓
Work deferred to process context (Workqueues)
    ↓
Firefox uses timers for animations (Time Management)
    ↓
User closes Firefox, resources freed (Process Termination)
```

**Everything connects!** Understanding these fundamentals makes the entire kernel comprehensible.

---

## Key Takeaways

**Design Principles You've Learned:**

- **Fairness** - Every process deserves CPU time
- **Efficiency** - Minimize overhead, maximize throughput  
- **Safety** - Protect kernel from malicious code
- **Responsiveness** - Keep the system interactive
- **Cleanup** - Always free resources properly
- **Isolation** - Processes can't harm each other
- **Performance** - Optimize hot paths aggressively

**Remember:**
> The kernel is not magic - it's hardware manipulation with careful software design! 

---

## You're Ready!

With this foundation, you can:
- Understand kernel panic messages
- Debug performance issues
- Write robust system software
- Contribute to open source
- Build your own OS components

> **The journey from user-space programmer to kernel developer starts here.**
