# Driver Core Mechanics

> Ever wondered how `read()` works on both a keyboard AND a disk? How your app uses the same `open()` call for a file AND a hardware device? Why `/dev/` is full of things that look like files but talk to hardware? How data safely crosses from kernel memory into your program?

**You're about to discover the bridge between hardware and userspace!**

---

## Where Are We?

> We're inside **04-device-drivers/00-foundation/driver-core/** - the foundation every Linux driver is built on!

Here's where we are in this series:

```
04-device-drivers/
├── 00-foundation/        <- device registration in the kernel
│   └── driver-core/      <- hardware exposed as files    (THIS!)
├── 01-input/             <- input events to userspace
├── 02-block/             <- storage read and write
├── 03-net/               <- packet send and receive
├── 04-usb/               <- USB device communication
├── 05-pci/               <- PCIe device discovery
├── 06-char/              <- character device data flow
└── 07-gpu/               <- display and rendering pipeline
```



**This section explains:**
- How ANY driver connects to the VFS layer
- Major/minor number system
- The /dev/ filesystem
- Complete read/write/ioctl flow
- Safe data transfer across kernel/user boundary
- Memory rules in drivers

---

## What You'll Learn

- **The core problem** - Why hardware needs a unified interface
- **Major/minor numbers** - The license plate system for devices
- **cdev structure** - The linker between numbers and operations
- **file_operations** - The contract every driver must fulfill
- **Complete VFS flow** - Exactly what happens from `open()` to hardware
- **copy_to/from_user** - The only safe way to move data
- **/dev/ filesystem** - How device files appear in userspace
- **Real examples** - `/dev/null` and `/dev/zero` traced completely

**Pure fundamentals!** Master this before touching any specific driver type.

---

# The Problem: Hardware Is Chaos

> **Why a unified interface had to exist**

## Without Driver Mechanics

Imagine the Linux kernel had NO unified driver system. Every hardware type would invent its own API:

```
GPU driver:     gpu_open(), gpu_render(), gpu_close()
Network driver: net_open(), net_send(), net_close()
Disk driver:    disk_open(), disk_read(), disk_close()
USB driver:     usb_open(), usb_transfer(), usb_close()
```

Every application would need to know which device it's talking to and call the right vendor-specific function. Standard Unix tools like `cat`, `dd`, `head`, `tail` would be useless on hardware. Developers would write different code for every device type they support.

This is total chaos. 

---

## The Unix Solution: Everything Is a File!

Linux says: **every device looks like a file.** One API rules them all:

```
GPU:      fd = open("/dev/dri/card0", O_RDWR)
Disk:     fd = open("/dev/sda", O_RDONLY)
Terminal: fd = open("/dev/tty", O_RDWR)
Random:   fd = open("/dev/random", O_RDONLY)

All use the same: open  read/write  close
```

Now `cat /dev/random` works. `dd if=/dev/zero of=file` works. Any app can talk to any device using the same three system calls it already knows.

**Driver core mechanics is the machinery that makes this possible.** 

---

# Major and Minor Numbers

> **The license plate system - every device gets two numbers**

## What They Mean

Every device in Linux is identified by exactly two numbers:

```
Major number = WHAT TYPE of device (which driver handles it)
Minor number = WHICH SPECIFIC device of that type
```

Look at real examples:

```
/dev/sda        8:0      major 8 = SCSI disk, minor 0 = first disk
/dev/sda1       8:1      same driver, minor 1 = first partition
/dev/sdb        8:16     same driver, minor 16 = second disk
/dev/tty0       4:0      major 4 = TTY driver, minor 0 = first terminal
/dev/tty1       4:1      same driver, second terminal
/dev/null       1:3      major 1 = mem devices, minor 3 = null
/dev/zero       1:5      same driver, minor 5 = zero
/dev/dri/card0  226:0    major 226 = GPU driver, minor 0 = first GPU
```

The kernel keeps an internal table: `major_map[8] = SCSI disk driver`. When you open `/dev/sda`, the kernel looks up major 8, finds the right driver, and uses minor 0 to tell it "first disk, first partition."

---

## How They're Stored

Both numbers fit into a single 32-bit value called `dev_t`:

```
Bits 31-20 = Major (12 bits  up to 4096 different device types)
Bits 19-0  = Minor (20 bits  up to 1 million specific devices)

Macros to work with it:
    MAJOR(dev)           pull out the major number
    MINOR(dev)           pull out the minor number
    MKDEV(major, minor)  combine them into one dev_t
```

---

## Static vs Dynamic Registration

Drivers need to claim a major number. There are two ways:

**Old way (static):** Driver picks a specific number and hopes nobody else took it. Conflicts happen.

**Modern way (dynamic):** Driver asks the kernel to assign any free major number. No conflicts, no coordination needed. This is always preferred today.

---

# The cdev Structure

> **The glue between a device number and what the driver can do**

## What It Is

When a driver registers itself, the kernel needs to know: "when someone opens device number 8:0, which functions should I call?" The `cdev` structure holds that answer.

Think of cdev as a connector:

```
Device number (8:0)
        
      cdev
        
 file_operations   actual driver functions
```

It links "this major:minor pair" to "these open/read/write/close implementations."

---

## Registration: 4 Steps

Every character driver does this exact sequence at startup:

```
Step 1: Reserve a device number
        Ask kernel: "give me a major:minor I can use"

Step 2: Allocate a cdev
        Create the connector object

Step 3: Initialize the cdev
        Tell it: "use THESE file_operations functions"

Step 4: Add cdev to kernel
        Tell kernel: "number X:Y  use this cdev"
```

After step 4, the kernel knows exactly what to do when someone opens that device. The connection is live. 

---

# file_operations: The Driver Contract

> **A function table that every driver must fill in**

## What It Is

`file_operations` is a struct full of function pointers. VFS (the Virtual File System layer) calls these functions. Your driver implements them. Think of it like an interface or contract:

```
VFS says: "I need open, read, write, ioctl, close"
Driver says: "Here are MY implementations of those"
```

The key functions:

```
open()     Called when app opens the device
            Initialize hardware, set up state

read()     Called when app reads from device
            Get data from hardware, send to app

write()    Called when app writes to device
            Take data from app, send to hardware

ioctl()    Called for special device commands
            Anything that doesn't fit read/write

release()  Called when app closes the device
            Cleanup, free resources
```

A driver doesn't have to implement all of them. If `read` is NULL, the kernel returns an error automatically. But most real drivers implement at least open, read, write, and release.

---

## Why release() Matters

What happens if an app crashes while a device is open?

Without `release()`: device stays locked, resources leaked, other apps can't use it forever.

With `release()`: kernel detects the crash, walks the process's open file list, and calls `release()` on every open device. Driver cleans up. Everything resets automatically. 

This is why resource management in drivers is safe - the kernel guarantees cleanup.

---

# The /dev/ Filesystem

> **Device files - they look like files, but they're not**

## What You See

```
ls -l /dev/

c  null         1,  3    character device, major 1, minor 3
c  zero         1,  5    character device
c  random       1,  8
b  sda          8,  0    block device, major 8, minor 0
b  sda1         8,  1
c  tty0         4,  0
c  dri/card0  226,  0
```

The `c` means character device. The `b` means block device. Those two numbers after the owner are major and minor - not file size like a regular file.

---

## Character vs Block

**Character devices** talk to hardware byte-by-byte, with no buffering. You get exactly what the hardware gives you, when it gives it. Examples: keyboard, mouse, terminals, `/dev/null`, `/dev/random`.

**Block devices** talk to hardware in chunks (512 or 4096 bytes at a time) and go through the kernel's page cache. Reads and writes can be buffered, reordered, and optimized. Examples: hard disks, SSDs.

---

## How /dev/ Files Get Created

**Old way:** Admin manually runs `mknod /dev/mydevice c 240 0`. Static, doesn't handle hot-plug.

**Modern way (udev):** Driver tells the kernel "I have a device called mydevice at 240:0." Kernel sends a message to udev (a userspace daemon). udev creates the `/dev/` entry, sets permissions, creates symlinks - all automatically. Works for hot-plug (USB plug/unplug, etc.) 

---

# Complete VFS  Driver Flow

> **Tracing every step from your app to the hardware**

## open() - Establishing the Connection

Your app calls `open("/dev/mydevice", O_RDWR)`. Here's every step:

```
App calls open()
    
Syscall: crosses into kernel (Ring 3  Ring 0)
    
VFS: resolves the path "/dev/mydevice"  finds inode
    
VFS: checks your permissions
    
VFS: sees this inode is a character device (not a regular file)
    
VFS: reads the device number from the inode (e.g. 240:0)
    
VFS: looks up cdev_map[240][0]  finds the registered cdev
    
VFS: sets file->f_op = cdev->ops (NOW file knows which driver to call)
    
VFS: calls driver's open() function
    
Driver: initializes device, returns success
    
VFS: assigns file descriptor (fd=3), stores in process table
    
Returns to app: fd = 3 
```

From this point forward, every `read(3, ...)` or `write(3, ...)` goes straight to your driver through those stored function pointers.

---

## read() - Data Flows Up

```
App calls read(fd, buf, 100)
    
Kernel: looks up fd=3  finds the file struct
    
Kernel: calls file->f_op->read()   your driver's read function
    
Driver: reads data from hardware
    
Driver: uses copy_to_user() to send data to app's buffer
    
Driver: returns number of bytes read
    
App: gets data 
```

---

## write() - Data Flows Down

```
App calls write(fd, data, 9)
    
Kernel: calls file->f_op->write()   your driver's write function
    
Driver: uses copy_from_user() to safely receive app's data
    
Driver: sends data to hardware
    
Driver: returns number of bytes written
    
App: knows how many bytes were accepted 
```

---

## ioctl() - Special Commands

Some devices need commands that don't fit into "read data" or "write data." A temperature sensor might need "start calibration." A disk might need "get geometry." A GPU might need "allocate buffer."

```
App calls ioctl(fd, COMMAND, argument)
    
Kernel: calls file->f_op->unlocked_ioctl()
    
Driver: looks at COMMAND, performs action
    
Driver: may use copy_to/from_user for the argument
    
Returns result 
```

ioctl is the escape hatch - anything device-specific that doesn't fit the normal read/write model goes here.

---

## close() - Guaranteed Cleanup

```
App calls close(fd)
    
Kernel: removes fd from process table
    
Kernel: decrements file reference count
    
If count reaches 0:
    Kernel calls file->f_op->release()
    Driver: frees resources, resets hardware
    Kernel: frees the file struct 

fd is now free for reuse 
```

Even if the app crashes, the kernel does this cleanup automatically.

---

# copy_to_user / copy_from_user

> **The ONLY safe way to move data across the kernel/user boundary**

## Why You Can't Just Use memcpy

This is one of the most important rules in driver writing. When your driver has data in kernel memory and needs to send it to a user buffer, you **cannot** just copy it directly.

**Reason 1: Different address spaces.** The user's buffer address (like `0x00601000`) is a virtual address. In kernel context, that address maps to completely different physical memory. Direct copy = wrong data or crash.

**Reason 2: Security.** A malicious app could pass a kernel address as its buffer. Direct copy would leak kernel memory to userspace.

**Reason 3: Page faults.** The user's page might not be in RAM yet. Direct copy would crash the kernel. The kernel needs to handle this gracefully.

---

## What copy_to_user Does

`copy_to_user(user_buffer, kernel_data, size)` handles all three problems:

```
1. Validates the user address
   Is it actually in user address range? Not a kernel address?

2. Checks permissions
   Does a valid VMA (Virtual Memory Area) exist for this address?
   Is the app allowed to write there?

3. Handles page faults gracefully
   If the page isn't in RAM, triggers allocation and retries

4. Performs the copy safely

5. Returns how many bytes it couldn't copy
   0 = full success, anything else = partial failure
```

---

## What copy_from_user Does

The exact mirror - for when your driver receives data from an app:

`copy_from_user(kernel_buffer, user_data, size)`

Same validation, same page fault handling, same safety guarantees - just in the opposite direction.

**The golden rule:** kernel  userspace = `copy_to_user`. Userspace  kernel = `copy_from_user`. Never `memcpy` across this boundary. Ever.

---

## For Single Values: put_user / get_user

When you only need one integer or pointer (common in ioctl), there are optimized shortcuts:

```
get_user(value, user_pointer)   read one value from user safely
put_user(value, user_pointer)   write one value to user safely
```

Same safety, less overhead than the full copy functions.

---

# Real Examples

> **Two of the simplest drivers in the kernel - fully explained**

## /dev/null - The Black Hole

`/dev/null` is registered at major 1, minor 3. Its entire implementation is almost comically simple:

**Read:** Always returns 0 bytes. Immediately. No data ever comes out. This signals EOF to whatever is reading.

**Write:** Always claims success. Accepts any amount of data, discards it silently, returns the byte count as if it was stored.

This is why `command > /dev/null` works - the shell opens /dev/null, writes all output into it, and the driver says "yes, I got it all" while throwing it away.

---

## /dev/zero - The Infinite Zeros

`/dev/zero` is registered at major 1, minor 5. Nearly as simple, but different:

**Read:** Never returns EOF. Always fills the user's buffer with zero bytes (0x00). However much you ask for, you get that many zeros. Forever.

**Write:** Same as /dev/null - silently discarded.

This is why `dd if=/dev/zero of=bigfile bs=1M count=100` creates a 100MB file full of zeros. The read_zero function just clears the user's buffer and returns the count. No actual data is stored anywhere - it's generated on demand.

---

# Physical Reality

## What Actually Happens in Hardware

When your app calls `read(fd, buf, 100)`, here's what the hardware is actually doing:

The CPU executes a SYSCALL instruction, which causes a hardware-level privilege switch from Ring 3 to Ring 0. The CPU saves the current instruction pointer, switches stacks, and jumps to the kernel's syscall handler - all in silicon.

The kernel walks a chain of pointers in RAM: current process  file descriptor table  file struct  file_operations  driver read function. Each dereference is a memory access.

Inside `copy_to_user`, the MMU (Memory Management Unit, a hardware component inside the CPU) translates the user's virtual address to a physical address, checking permission bits in the page table. If the page isn't present, hardware raises a page fault exception, the kernel allocates RAM and maps it, then the copy resumes.

Finally, SYSRET executes - another hardware-level privilege switch back to Ring 3. The app sees its buffer filled with data.

Total time: typically 1-10 microseconds. All of it real hardware doing real work.

---

## The Process File Descriptor Table

Every Linux process has a table in kernel memory:

```
current->files->fd[]

fd[0] = stdin
fd[1] = stdout
fd[2] = stderr
fd[3] = /dev/mydevice   your opened device
fd[4] = NULL
...
```

This array lives in RAM as part of the process's `task_struct`. Each entry is a pointer to a `struct file`, which holds the `file_operations` pointer to your driver. When you call `read(3, ...)`, the kernel indexes this array with 3, finds your device's file struct, and calls your read function.

---

# Connections to Other Topics

## System Calls (01-kernel-fundamentals)

```
open/read/write/ioctl/close are all syscalls.
The SYSCALL instruction, Ring 3  Ring 0 transition,
and the return path are exactly what you studied there.
Driver core mechanics lives on top of that foundation.
```

## VFS Layer (02-kernel-architecture)

```
VFS is the layer ABOVE drivers.
It provides the unified file abstraction.
Drivers implement file_operations.
VFS calls those implementations.
They work together - VFS handles the abstraction,
drivers handle the hardware reality.
```

## Memory Management (03-memory-management)

```
copy_to_user can trigger page faults.
Page fault  kernel allocates physical page.
MMU updates the page table.
TLB gets the new mapping.
Copy resumes.

Every copy across the kernel/user boundary
potentially touches the mm/ subsystem.
```

## Linux x86 Architecture (04-linux-x86-architecture)

```
The SYSCALL/SYSRET instructions are x86-specific.
Ring 3  Ring 0 privilege switching is x86-specific.
MMU address translation is x86-specific.

Driver core mechanics uses all of that
every time open/read/write is called.
```

---

# Complete Driver Core Summary

## What Is It?

```
Driver Core Mechanics = The foundation that makes
hardware look like files to userspace.

Five components:
├── file_operations     the driver function table
├── cdev                links device numbers to operations
├── Major/minor         identifies every device
├── copy_to/from_user   safe data transfer
└── /dev/ entries       visible device files
```

## Why Does It Exist?

```
Problem                    Solution
─────────────────────────────────────────────────
Different API per device  Unified file interface
Unsafe kernel/user copy   copy_to/from_user
No device identification  Major/minor numbers
Resource leaks on crash   release() auto-cleanup
Manual /dev/ management   udev automation
```

## The Full Flow

```
App: open("/dev/mydevice")
    
VFS resolves path  finds inode
    
VFS reads device number from inode
    
VFS looks up cdev  finds file_operations
    
VFS calls driver open()
    
App gets file descriptor

App: read(fd, buf, size)
    
VFS calls driver read()
    
Driver: hardware  kernel buffer  copy_to_user  buf
    
App gets data

App: close(fd)
    
VFS calls driver release()
    
Driver cleans up, hardware reset
    
Resources freed 
```

---

## What You Now Understand

```
 Why unified interface exists (everything is a file!)
 Major/minor numbers (device type + specific instance)
 cdev (the device number  driver function linker)
 file_operations (the VFS/driver contract)
 VFS flow (complete path: app  syscall  VFS  driver)
 copy_to/from_user (the ONLY safe transfer method)
 /dev/ filesystem (how entries appear and what they mean)
 /dev/null and /dev/zero (real drivers, fully traced)
 Physical reality (CPU privilege switch, MMU, RAM)
 Connections to every previous section
```

---

## The Big Reveal

**When you type `cat /dev/null`:**

```
cat opens "/dev/null"
    
VFS finds inode  reads major 1, minor 3
    
VFS looks up cdev_map[1][3]  finds null driver
    
Calls null's open()  success
    
cat calls read(fd, buf, 4096)
    
Calls null's read()  returns 0 (EOF immediately!)
    
cat sees EOF  prints nothing  exits
    
Kernel calls null's release()  no-op cleanup
    
fd freed 

Total lines in null's driver: about 10.
Total machinery underneath it: the entire driver core.

That's the magic:
└── Simple driver interface on top
    Complex unified machinery underneath
    Hardware looks like files
    Unix philosophy in action! 
```

---

## A Final Thought

**Every time you use a device in Linux:**

```
cat /dev/random
dd if=/dev/zero of=file
echo "hello" > /dev/tty
read -r line < /dev/stdin
```

**Remember:**

```
It's not magic
    
file_operations routes the call
    
cdev connects the device number to the driver
    
copy_to/from_user crosses the boundary safely
    
VFS makes hardware look like files
    
The kernel guarantees cleanup on crash
    
That's driver core mechanics! 
```

---

We've completed **driver-core - 00-foundation is done!**

```
04-device-drivers/
├── 00-foundation/    done
│   ├── device-model/ done
│   └── driver-core/  done
└── 01-input/         next
```

**Every driver you study from here builds on exactly this foundation.**
