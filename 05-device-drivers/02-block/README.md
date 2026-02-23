# Every Byte to Disk - The Storage Abstraction Layer

> Ever wondered how writing a file actually reaches your SSD? How Linux handles both spinning hard drives and blazing-fast NVMe? How thousands of small writes become efficient large transfers?

**You're about to find out!**

---

## What's This About?

This is **block/** - the unified interface between filesystems and storage hardware!

Here's where it fits in the driver ecosystem:

```
drivers/
04-device-drivers/
├── 00-foundation/      ← Device registration in kernel
├── 01-input/           ← Input events to userspace
├── 02-block/           ← Storage read and write
│   ├── Block layer     ← Request queuing and scheduling
│   ├── bio structure   ← Fundamental I/O unit
│   └── Drivers         ← NVMe, SATA, SCSI
├── 03-net/             ← Packet send and receive
├── 04-usb/             ← USB device communication
├── 05-pci/             ← PCIe device discovery
├── 06-char/            ← Character device data flow
└── 07-gpu/             ← Display and rendering pipeline
```

**Location in code:**

```
block/
├── bio.c              ← Block I/O structure
├── blk-core.c         ← Core block layer
├── blk-mq.c           ← Multi-queue (modern!)
├── blk-mq-sched.c     ← Scheduler interface
├── elevator.c         ← I/O scheduler framework
├── mq-deadline.c      ← Deadline scheduler
├── bfq-iosched.c      ← Budget Fair Queueing
└── kyber-iosched.c    ← Kyber for NVMe

drivers/
├── nvme/              ← NVMe SSD driver
├── ata/               ← SATA/IDE (libata)
└── scsi/              ← SCSI subsystem
```

**This covers:**
- Block I/O abstraction (bio structure)
- Request queuing (single vs multi-queue)
- I/O scheduling (none, deadline, BFQ, Kyber)
- Page cache integration (reads and writes)
- Storage technologies (HDD, SATA SSD, NVMe)
- DMA and completion

---

# The Fundamental Problem

**Physical Reality:**

```
Your system has different storage devices:

HDD (Hard Disk Drive):
├── Spinning magnetic platters (5400-15000 RPM)
├── Mechanical head movement (seek time: 3-15ms)
├── Sequential reads: fast (100-200 MB/s)
├── Random reads: slow (100-200 IOPS)
└── Optimal: Sort requests by physical location

SATA SSD:
├── NAND flash, no moving parts
├── Random reads: fast (50,000-100,000 IOPS)
├── SATA interface limit: 550 MB/s max
├── Command queue: 32 maximum
└── Optimal: Parallel requests

NVMe SSD:
├── NAND flash on PCIe interface
├── Direct CPU-to-storage (no controller overhead)
├── Random reads: very fast (500,000-1,000,000 IOPS)
├── PCIe bandwidth: 3,000-7,000 MB/s
├── Command queues: 65,535 maximum
└── Optimal: Flood with requests
```

**The problem:**

```
Filesystems need to write data:

ext4 wants to write block 1234
btrfs wants to write block 5678
XFS wants to write block 9012

Each filesystem must handle:
├── HDD: Sort by location, batch requests
├── SATA SSD: Queue up to 32 requests
├── NVMe: Use multiple hardware queues
└── Every device type needs different strategy

Without abstraction:
├── ext4 implements HDD logic
├── ext4 implements SATA logic
├── ext4 implements NVMe logic
├── btrfs reimplements all of above
├── XFS reimplements all of above
└── Massive code duplication

Can't have every filesystem know every device
```

**How do you provide ONE interface for ALL storage devices?**

---

# Without a Block Layer

**Imagine no storage abstraction:**

```
ext4 filesystem writes data directly:

For HDD:
Must implement:
├── SATA protocol (ATA commands, registers)
├── Request sorting by cylinder/head/sector
├── Elevator algorithm for head movement
├── NCQ (Native Command Queuing)
└── Error recovery

For NVMe:
Must implement:
├── NVMe protocol (submission/completion queues)
├── PCIe register access
├── Multiple queue management
├── MSI-X interrupt handling
└── Different error recovery

For SCSI RAID:
Must implement:
├── SCSI protocol
├── RAID-specific commands
└── Yet another interface

Every filesystem reimplements for every device
Thousands of lines of duplicated code
Impossible to maintain


Performance disaster:

Process A writes 4KB to file A (sector 1000)
Process B writes 4KB to file B (sector 5000)
Process C writes 4KB to file C (sector 2000)

Without optimization:
├── Seek to sector 1000 (10ms seek time)
├── Write 4KB
├── Seek to sector 5000 (10ms seek time)
├── Write 4KB
├── Seek to sector 2000 (10ms seek time)
└── Write 4KB

Total: 30ms just seeking
Could be optimized to: 12ms (sorted order)

No central place to optimize
Each filesystem does its own thing


Request merging impossible:

Process writes file byte by byte:
├── write(fd, "H", 1) → Submit to disk
├── write(fd, "e", 1) → Submit to disk
├── write(fd, "l", 1) → Submit to disk
├── write(fd, "l", 1) → Submit to disk
└── write(fd, "o", 1) → Submit to disk

5 disk operations for 5 bytes
Should be 1 disk operation for 4KB block

No merging layer = terrible performance
```

**Chaos without unified storage layer!**

---

# The Three-Layer Solution

**Unified storage abstraction:**

```
Layer 1: FILESYSTEMS (what to write)
├── ext4: "Write data to block 1234"
├── btrfs: "Write data to block 5678"
└── XFS: "Write data to block 9012"

All use same block layer API
No device-specific knowledge


Layer 2: BLOCK LAYER (optimization)
├── Receives requests from filesystems
├── Merges adjacent requests
├── Schedules optimal order (I/O scheduler)
├── Batches for efficiency
└── Routes to appropriate driver

One place for optimization
Hardware-independent algorithms


Layer 3: DEVICE DRIVERS (how to write)
├── NVMe driver: NVMe protocol, PCIe, interrupts
├── SATA driver: SATA protocol, registers, DMA
└── SCSI driver: SCSI protocol, commands

Each knows its hardware intimately
No filesystem knowledge needed


Result:
ext4 write → Block layer (merge, schedule) → NVMe driver → Disk
btrfs write → Same block layer → SATA driver → Disk
XFS write → Same block layer → SCSI driver → Disk

One interface, any filesystem, any device
```

---

# 1. The bio Structure

## Fundamental Unit of Block I/O

**What is a bio?**

```c
struct bio {
    struct bio *bi_next;           // Next bio (for chaining)
    struct block_device *bi_bdev;  // Target device (/dev/sda, /dev/nvme0n1)
    unsigned int bi_opf;           // Operation: READ, WRITE, FLUSH, DISCARD
    sector_t bi_sector;            // Starting sector on disk
    unsigned int bi_size;          // Total bytes to transfer
    
    unsigned short bi_vcnt;        // Number of segments
    struct bio_vec bi_io_vec[];    // Array of page segments
    
    bio_end_io_t *bi_end_io;      // Callback when complete
    void *bi_private;              // Data for callback
    
    blk_status_t bi_status;        // Result (success or error)
};

One bio = One I/O operation to disk
Can span multiple memory pages
```

**bi_io_vec - Scatter/Gather:**

```
bio can span non-contiguous pages:

Writing 12KB of data:
├── Page 1: Physical 0x10000, bytes 0-4095
├── Page 2: Physical 0x20000, bytes 4096-8191
└── Page 3: Physical 0x30000, bytes 8192-12287

bio structure:
bi_vcnt = 3
bi_io_vec[0] = { page=0x10000, offset=0, len=4096 }
bi_io_vec[1] = { page=0x20000, offset=0, len=4096 }
bi_io_vec[2] = { page=0x30000, offset=0, len=4096 }

DMA engine:
├── Reads from page 1 → writes to disk
├── Reads from page 2 → writes to disk
└── Reads from page 3 → writes to disk

No need to copy to contiguous buffer
Scatter/gather DMA saves memory copy
```

**Example bio for write:**

```
Writing "hello world" to sector 1000:

bio {
    bi_bdev = /dev/sda
    bi_opf = REQ_OP_WRITE
    bi_sector = 1000
    bi_size = 4096 (full block)
    bi_vcnt = 1
    bi_io_vec[0] = {
        page = physical address of page cache page
        offset = 0
        len = 4096
    }
    bi_end_io = end_bio_callback
}

submit_bio(bio);
```

---

# 2. Request Queue and Requests

## Organizing Work for Devices

**struct request:**

```c
struct request {
    struct request_queue *q;       // Which queue owns this
    struct bio *bio;               // First bio
    struct bio *biotail;           // Last bio (for merging)
    
    sector_t __sector;             // Starting sector
    unsigned int __data_len;       // Total bytes
    
    unsigned int cmd_flags;        // READ/WRITE/SYNC flags
    
    struct gendisk *rq_disk;       // Which disk
    
    u64 start_time_ns;             // When queued (latency tracking)
};

One request = One or more merged bios
Sent as atomic unit to driver
```

**struct request_queue:**

```c
struct request_queue {
    struct list_head queue_head;   // List of pending requests
    
    make_request_fn_t *make_request_fn;  // How to submit
    request_fn_t *request_fn;            // Old interface
    
    // Multi-queue (modern):
    struct blk_mq_ops *mq_ops;
    struct blk_mq_ctx *queue_ctx;        // Per-CPU queues
    struct blk_mq_hw_ctx **queue_hw_ctx; // Hardware queues
    
    // Device limits:
    struct queue_limits limits;
        unsigned int max_sectors;         // Max per request
        unsigned short max_segments;      // Max scatter/gather
        unsigned int logical_block_size;  // 512 or 4096
        unsigned int physical_block_size; // Actual hardware
    
    // Scheduler:
    struct elevator_queue *elevator;
    
    // Statistics:
    unsigned long in_flight;               // Active requests
};

One queue per block device
Central coordination point
```

**Request merging:**

```
Two bios submitted:

bio 1: Write sectors 1000-1007 (4KB)
bio 2: Write sectors 1008-1015 (4KB)

Adjacent sectors detected:
└── Merge into single request: sectors 1000-1015 (8KB)

Result:
├── One disk operation instead of two
├── Better throughput
└── Lower overhead

Merging happens automatically in block layer
```

---

# 3. Single Queue vs Multi-Queue

## Evolution of Block Layer Architecture

**Old architecture (single queue):**

```
All CPUs share ONE request queue:

CPU 0 submits write ──┐
CPU 1 submits read  ──┤→ SPINLOCK → Request Queue → Driver → Disk
CPU 2 submits write ──┘

Problems:
├── Lock contention (all CPUs fight for lock)
├── Bottleneck at queue lock
├── Can't scale to many CPUs
└── Wastes fast NVMe capabilities

For HDD: Not a problem (HDD is bottleneck)
└── HDD: 100-200 IOPS, lock overhead minimal

For NVMe: Major problem
└── NVMe: 1,000,000 IOPS, lock is bottleneck
```

**Modern architecture (blk-mq):**

```
Multi-queue block layer:

CPU 0 → Software Queue 0 ──┐
CPU 1 → Software Queue 1 ──┤→ Hardware Queue 0 ──┐
CPU 2 → Software Queue 2 ──┤→ Hardware Queue 1 ──┤→ NVMe SSD
CPU 3 → Software Queue 3 ──┘→ Hardware Queue 2 ──┘
                            → Hardware Queue 3 ──┘

Software queues:
├── One per CPU (no contention)
├── CPU 0 uses only its queue
└── CPU 1 uses only its queue

Hardware queues:
├── Match device capabilities
├── HDD: 1 hardware queue
├── SATA SSD: 1-2 hardware queues
└── NVMe: Up to 65,535 hardware queues

Benefits:
├── No lock contention
├── Scales to many CPUs
├── Full NVMe performance
└── 1,000,000 IOPS achievable
```

**blk_mq_ops:**

```c
struct blk_mq_ops {
    queue_rq_fn *queue_rq;         // Submit request to hardware
    commit_rqs_fn *commit_rqs;     // Commit batch
    
    init_hctx_fn *init_hctx;       // Init hardware queue
    exit_hctx_fn *exit_hctx;       // Cleanup
    
    complete_fn *complete;         // Completion callback
    timeout_fn *timeout;           // Request timeout
    poll_fn *poll;                 // Polling mode (no IRQ)
};

Driver implements these operations
Block layer calls them
```

---

# 4. I/O Schedulers

## Deciding Request Order

**What schedulers do:**

```
Three goals:
1. Maximize throughput (more bytes per second)
2. Minimize latency (faster individual requests)
3. Ensure fairness (no process starved)

Schedulers decide:
├── Which request to send next?
├── Should we merge adjacent requests?
└── Should we wait for more requests?
```

**none (for NVMe):**

```
Strategy: Submit everything immediately, no scheduling

Algorithm:
└── Request arrives → send to driver immediately

Why for NVMe?
├── NVMe latency: 100 microseconds
├── Scheduler overhead: 50 microseconds
└── Scheduler = 50% overhead (too much)

NVMe has internal scheduler:
└── Let hardware decide order

Best for:
├── NVMe SSDs
├── Low-latency devices
└── Random I/O workloads (databases)

Configuration:
echo none > /sys/block/nvme0n1/queue/scheduler
```

**mq-deadline:**

```
Strategy: Ensure requests complete before deadline

For each request, assign deadline:
├── Read: 500ms maximum wait
└── Write: 5000ms maximum wait

Algorithm:
1. Sort requests by sector (efficient)
2. Check: Any request past deadline?
3. If yes: Process that request NOW (priority)
4. If no: Process in sector order

Why shorter read deadline?
├── User often waiting for read
└── Writes can be buffered

Example scenario:
Request queue:
├── Write sector 1000 (deadline in 1s)
├── Write sector 2000 (deadline in 0.1s) → Expired!
└── Read sector 1500 (deadline in 0.4s)

Order: Write 2000, Read 1500, Write 1000

Best for:
├── SATA SSDs
├── Mixed read/write workloads
└── Latency-sensitive applications
```

**BFQ (Budget Fair Queueing):**

```
Strategy: Fairness between processes + low latency

Each process gets budget (time slice for I/O):

Interactive process (terminal):
├── Small budget
├── But served quickly
└── Low latency feel

Background process (backup):
├── Large budget
├── High throughput
└── Doesn't starve interactive

Real scenario:

WITHOUT BFQ:
rsync copying 100GB:
└── Hammers disk with requests
    Your text editor saves:
    └── Waits behind rsync queue
        Takes 10 seconds (bad UX)

WITH BFQ:
rsync copying 100GB:
└── Gets budget, runs efficiently
Your text editor saves:
└── Gets small budget but served FIRST
    Saves in 100ms (you don't notice rsync)

Best for:
├── Desktop systems (interactive + background mix)
├── HDDs (seeks expensive, fairness matters)
└── Latency-sensitive users

Default on many desktop Linux systems
```

**Kyber:**

```
Strategy: Lightweight scheduling for fast devices

Two priority classes:
├── Synchronous (reads, sync writes)
└── Asynchronous (buffered writes)

Token-based queue limiting:
└── Prevent async from starving sync

Best for:
├── Fast SSDs
├── Multi-queue devices
└── When need scheduling but device is fast

Overhead lower than deadline/BFQ
```

---

# 5. Page Cache Integration

## Block Layer Meets Memory Management

**Read path:**

```
Application: read(fd, buffer, 4096);

VFS checks page cache:

Cache HIT:
├── Page already in memory
├── copy_to_user(buffer, cached_page, 4096)
├── Return to user
└── NO disk access (fast)

Cache MISS:
├── Page not in cache
├── Need to read from disk

Steps:
1. Allocate page in page cache (mm/page_alloc.c)
2. Determine disk location (filesystem logic)
3. Create bio:
   └── bi_bdev = /dev/sda
       bi_opf = REQ_OP_READ
       bi_sector = 8000
       bi_io_vec[0] = { new page, 0, 4096 }
       bi_end_io = read_completion_callback
4. submit_bio(bio)
5. Block layer → scheduler → driver → disk
6. DMA fills page
7. Interrupt fires
8. Completion callback: mark page uptodate
9. Wake waiting process
10. copy_to_user(buffer, cached_page, 4096)
11. Return to user

Page now in cache for future reads
```

**Write path:**

```
Application: write(fd, buffer, 4096);

VFS writes to page cache:

Steps:
1. Find/allocate page in cache
2. copy_from_user(cached_page, buffer, 4096)
3. Mark page DIRTY
4. Return to user immediately (fast)

User doesn't wait for disk

Later (writeback):
pdflush/writeback kernel threads:
└── "Too many dirty pages" or "page dirty for 30s"
    
    For each dirty page:
    1. Determine disk location
    2. Create bio:
       └── bi_bdev = /dev/sda
           bi_opf = REQ_OP_WRITE
           bi_sector = 8000
           bi_io_vec[0] = { dirty page, 0, 4096 }
           bi_end_io = write_completion_callback
    3. submit_bio(bio)
    4. Block layer → scheduler → driver → disk
    5. DMA reads from page
    6. Interrupt fires
    7. Completion callback: mark page clean

Why delayed?
├── Multiple writes to same page → one disk write
├── Chance to merge with nearby writes
└── Better performance

Risk: Data loss if power fails before writeback
Solution: fsync() or O_SYNC forces immediate write
```

**Direct I/O:**

```
Bypass page cache:

open(file, O_DIRECT);

read/write goes directly to disk:
├── Application buffer → bio → block layer → disk
├── No page cache involved
└── DMA directly from/to user pages

Why use Direct I/O?
1. Databases have their own cache:
   └── Don't want kernel cache + DB cache (double caching)
2. Large files that won't be reread:
   └── Video streaming: cache pointless
3. Control over I/O timing:
   └── Database can control exact write timing

Trade-off:
├── Bypass cache benefits (no double caching)
└── Lose cache benefits (must read from disk every time)
```

---

# 6. Complete Write Flow

## From write() to Disk

**Scenario: echo "hello" > /tmp/test.txt**

**Step 1: System call**

```
Shell executes: write(fd, "hello\n", 6)

Kernel: sys_write() → vfs_write() → ext4_write()
```

**Step 2: Filesystem processes**

```
ext4_file_write():

File is new, need disk space:
└── Allocate block: block 100000

Find page in page cache:
└── New file, no page yet
    Allocate page: page_alloc() → physical 0x45678000

Copy from user:
└── copy_from_user(page + 0, "hello\n", 6)

Mark page dirty:
└── SetPageDirty(page)
    Page needs writeback

Update inode metadata:
└── File size: 0 → 6 bytes
    Modification time updated

Return to user:
└── write() returns 6 (bytes written)

User process continues
Data NOT on disk yet (still in cache)
```

**Step 3: Writeback triggered**

```
Writeback daemon or fsync():

Scan dirty pages:
└── Find page for /tmp/test.txt

Map file offset to disk block:
└── File offset 0 → disk block 100000
    Block 100000 → sector 800000 (sector = block × 8 for 4K)
```

**Step 4: Create bio**

```
bio = bio_alloc(GFP_NOIO, 1);

Fill bio structure:
bi_bdev = /dev/nvme0n1
bi_opf = REQ_OP_WRITE
bi_sector = 800000
bi_size = 4096
bi_vcnt = 1
bi_io_vec[0] = {
    .bv_page = page (0x45678000)
    .bv_offset = 0
    .bv_len = 4096
}
bi_end_io = ext4_end_bio_write

submit_bio(bio);
```

**Step 5: Block layer**

```
submit_bio() → blk_mq_make_request()

Get current CPU's software queue:
└── CPU 2 running → use software queue 2

Try to merge:
└── Check for adjacent requests
    None found (new file)

Create request:
└── request = blk_mq_alloc_request()
    request->__sector = 800000
    request->__data_len = 4096
    request->bio = bio
    request->start_time = now (latency tracking)

Choose hardware queue:
└── CPU 2 → hardware queue 2 (for NVMe)

Run scheduler:
└── mq-deadline: add to queue, check deadlines
    No expired deadlines
    Ready to dispatch

Dispatch to driver:
└── nvme_queue_rq(hctx, bd)
```

**Step 6: NVMe driver**

```
nvme_queue_rq():

Map bio pages for DMA:
└── dma_map_page(page) → DMA address 0x45678000

Create NVMe command (64 bytes):
opcode = NVM_CMD_WRITE (0x01)
nsid = 1 (namespace ID)
slba = 800000 (starting logical block)
length = 7 (8 blocks - 1, zero-indexed)
prp1 = 0x45678000 (physical page for DMA)

Write to submission queue:
└── sq_cmds[tail] = nvme_cmd
    tail = (tail + 1) % queue_depth

Ring doorbell (tell controller):
└── writel(tail, doorbell_register)
    This is ioremap from TOPIC 12
    PCIe write to device
```

**Step 7: NVMe controller**

```
NVMe hardware (on SSD):

Read submission queue:
└── New command at tail position

DMA from host memory:
└── PCIe DMA read from physical 0x45678000
    4096 bytes transferred via PCIe
    Data now in SSD buffer

Write to NAND flash:
└── Find appropriate NAND page
    Program NAND cells (100-500 microseconds)
    Data persisted on flash

Complete command:
└── Write to completion queue
    Status = success
    Command ID included

Post interrupt:
└── MSI-X interrupt to CPU 2 (TOPIC 6)
```

**Step 8: Completion**

```
CPU 2 receives MSI-X interrupt:

nvme_irq():
└── Read completion queue
    Extract command ID, status
    status = success

    Ring completion doorbell:
    └── Tell controller we processed

    Complete request:
    └── blk_mq_complete_request(request)

bio completion callback:
└── bi_end_io(bio, status)
    ext4_end_bio_write()
    
    Mark page clean:
    └── ClearPageDirty(page)
    
    Wake waiters:
    └── Any process waiting on fsync()
        wake_up()

fsync() returns:
└── Data now safely on disk

Total time: 100-500 microseconds (NVMe)
```

---

# 7. Storage Technologies

## Different Hardware Characteristics

**HDD (Hard Disk Drive):**

```
Physical mechanism:
├── Rotating magnetic platters (5400-15000 RPM)
├── Read/write heads on actuator arm
└── Moving parts (mechanical)

Performance:
├── Sequential read: 100-200 MB/s
├── Random IOPS: 100-200
├── Seek time: 3-15ms (head movement)
└── Rotational latency: 2-5ms

Optimal access pattern:
└── Sequential access (minimize seeking)
    Scheduler sorts requests by location
    Elevator algorithm reduces head movement

Use case:
└── Large sequential data (backups, video)
    Cheap per gigabyte
    Avoid for random I/O workloads
```

**SATA SSD:**

```
Physical:
├── NAND flash chips (no moving parts)
└── SATA controller interface

Interface limits:
├── SATA III: 6 Gbps = 550 MB/s max
├── Command queue: 32 maximum
└── Legacy protocol designed for HDDs

Performance:
├── Sequential: 500-550 MB/s
├── Random IOPS: 50,000-100,000
└── Latency: 50-100 microseconds

Bottleneck:
└── SATA interface, not NAND
    Flash chips can go faster
    Interface holds them back

Best for:
└── Upgrade from HDD
    Boot drives
    General desktop use
```

**NVMe SSD:**

```
Physical:
├── NAND flash chips
├── NVMe controller
└── PCIe interface (direct CPU connection)

Interface advantages:
├── PCIe Gen 4 x4: 8 GB/s bandwidth
├── 65,535 command queues possible
├── No legacy protocol overhead
└── Designed for flash from ground up

Performance:
├── Sequential: 3,000-7,000 MB/s
├── Random IOPS: 500,000-1,000,000
├── Latency: 100-200 microseconds
└── Scales with more CPU cores

Why so fast?
├── PCIe direct to CPU (no SATA controller)
├── Multiple hardware queues (65,535 vs 1)
├── blk-mq architecture matches hardware
└── Modern protocol, no legacy baggage

Device naming:
└── /dev/nvme0 (controller)
    /dev/nvme0n1 (namespace = disk)
    /dev/nvme0n1p1 (partition)

Best for:
└── High-performance workloads
    Databases
    Virtual machines
    Anything I/O intensive
```

**Comparison:**

```
Operation         | HDD    | SATA SSD | NVMe SSD
--------------------------------------------------
Random IOPS       | 200    | 75,000   | 600,000
Sequential (MB/s) | 150    | 550      | 5,000
Latency          | 10ms   | 80μs     | 120μs
Queue depth      | 1      | 32       | 65,535
Seek penalty     | Yes    | No       | No
Price/GB         | Low    | Medium   | High
```

---

# 8. TRIM/DISCARD

## Maintaining SSD Performance

**The NAND problem:**

```
NAND flash limitation:
└── Can't overwrite directly
    Must erase before write
    Erase operates on large blocks (128KB-512KB)

Without TRIM:

File deleted:
├── Filesystem marks sectors free
├── But doesn't tell SSD
└── SSD still thinks data valid

Later write to "freed" sector:
├── SSD must erase old data first
├── Erase takes time (1-3ms)
├── Write amplification (erase 128KB to write 4KB)
└── Performance degrades over time
```

**TRIM solution:**

```
File deleted:
└── Filesystem marks sectors free
    Send DISCARD command to SSD:
    └── bio with REQ_OP_DISCARD
        bi_sector = 800000
        length = 8000 sectors

SSD receives DISCARD:
└── Mark those blocks as invalid
    Can pre-erase during idle
    Ready for fast writes later

Next write to that location:
└── Already erased
    Fast write (no erase needed)

Benefits:
├── Maintains performance over time
├── Extends SSD lifespan
└── Reduces write amplification

Enable TRIM:
└── mount -o discard (continuous)
    Or: fstrim /mount/point (periodic)
```

---

# 9. Partitions

## Dividing Disks

**What are partitions?**

```
Physical disk: /dev/sda (whole disk)

Partition table (GPT or MBR):
└── Defines regions of disk

Example layout:
sda (1TB total, 1,953,525,168 sectors):
├── sda1: sectors 2048-2099199 (1GB, /boot)
├── sda2: sectors 2099200-4196351 (1GB, swap)
└── sda3: sectors 4196352-1953525134 (remaining, /)

gendisk structure:
disk_name = "sda"
capacity = 1953525168
part[1] = { start=2048, length=2097152 }
part[2] = { start=2099200, length=2097152 }
part[3] = { start=4196352, length=1949328783 }
```

**Partition translation:**

```
Write to /dev/sda3:

User requests: sector 100 of sda3

Kernel adds partition offset:
└── sda3 starts at sector 4196352
    Actual disk sector = 4196352 + 100 = 4196452

bio created:
└── bi_sector = 4196452 (absolute sector on disk)

Partition isolation:
└── Can't write outside partition boundary
    Kernel enforces limits
    Prevents accidental data corruption
```

---

# 10. Physical Reality

## What Happens in Hardware

**DMA transfer:**

```
Write operation with DMA:

bio page: physical address 0x45678000

NVMe driver:
└── Map for DMA: dma_map_page()
    IOMMU validates: page is accessible
    Returns DMA address (might be same or different)

NVMe command:
└── PRP1 = 0x45678000 (physical address)

NVMe controller:
└── Issues PCIe DMA read
    Memory controller fetches from 0x45678000
    4096 bytes transferred via PCIe
    CPU is FREE (not involved in copy)

DMA benefits:
├── CPU can do other work
├── Direct memory-to-device transfer
└── Faster than CPU copy

Completion:
└── MSI-X interrupt notifies CPU
    Only involves CPU at start and end
```

**PCIe transaction:**

```
Doorbell ring (submit command):

CPU: writel(tail, doorbell)
└── ioremap from TOPIC 12

Physical path:
├── CPU write to memory-mapped I/O address
├── Memory controller: route to PCIe
├── PCIe TLP (Transaction Layer Packet)
├── Travels to NVMe controller
└── NVMe sees doorbell ring

Time: Nanoseconds

DMA read:
├── NVMe issues PCIe read request
├── Memory controller responds
├── Data flows over PCIe lanes
└── Full 4KB transferred in microseconds

PCIe Gen 4 x4: 8 GB/s bandwidth
Enough for even fastest SSDs
```

**Sectors vs blocks:**

```
Physical SSD internals:

NAND page: 4KB-16KB (smallest writable unit)
NAND block: 128KB-512KB (smallest erasable unit)
└── Confusing: NAND "block" != filesystem block

OS view:

Logical sector: 512 bytes (traditional)
Physical sector: 4096 bytes (modern native)
Filesystem block: 4096 bytes (typical)

Block layer uses 512-byte sectors internally:
└── Even on 4K native drives
    Sectors 0-7 = first physical sector (4096 bytes)
    
Why 512-byte granularity?
└── Backwards compatibility
    Some old code assumes 512
    Kernel handles conversion
```

---

# 11. Connections to Other Topics

## Block Layer in Context

**Connection to Page Cache:**

```
mm/filemap.c: Page cache management

Block layer integration:
├── Cache miss → block layer reads from disk
├── Dirty pages → block layer writes to disk
└── Direct I/O → block layer without cache

They work together:
└── Cache hit: no block layer (fast)
    Cache miss: block layer fills cache
    Writeback: block layer from dirty pages

Performance:
└── Page cache reduces disk access by 80-90%
    Block layer handles remaining 10-20%
```

**Connection to ioremap:**

```
TOPIC 12: ioremap - map device memory

NVMe driver needs registers:
└── regs = ioremap(pci_bar0, size)
    Use registers to control NVMe

Doorbell writes:
└── writel(value, regs + doorbell_offset)
    This IS ioremap in action

Block drivers use ioremap extensively
```

**Connection to DMA:**

```
Memory zones (TOPIC 10):

ZONE_DMA: 0-16MB (old ISA devices)
ZONE_NORMAL: 16MB-end

bio pages must be DMA-accessible:
└── Most modern devices: any memory OK
    Old devices: must use ZONE_DMA

DMA mapping:
└── dma_map_page() from mm/ subsystem
    IOMMU translates if needed
    Ensures DMA safety
```

**Connection to Interrupts:**

```
TOPICS 3, 6: IRQ handling, MSI-X

NVMe completion:
└── MSI-X interrupt (TOPIC 6)
    APIC routes to CPU
    nvme_irq() handler runs
    
No interrupts = no completion notification
Must use polling (less efficient)
```

**Connection to Device Model:**

```
TOPIC 15: Bus/Device/Driver

gendisk contains struct device:
└── Device model integration

sysfs exposure:
/sys/block/nvme0n1/
├── queue/
│   ├── scheduler (read/write)
│   ├── nr_requests
│   ├── max_sectors_kb
│   └── iostats
├── size (sectors)
├── dev (major:minor)
└── partition info

Hotplug via device model:
└── udev creates /dev/ entries
    Automatic on device plug
```

---

# Summary

## What You've Mastered

**You now understand the block layer:**

```
What it is: Unified storage abstraction layer
Why needed: Hardware diversity, optimization, filesystem independence
bio structure: Fundamental I/O unit (sectors + pages)
Request queue: Per-device work organization
blk-mq: Modern multi-queue architecture (no lock contention)
I/O schedulers: none (NVMe), deadline (SATA), BFQ (fair), Kyber (fast)
Page cache integration: Reads fill cache, writes go through cache
Complete flow: write() → cache → writeback → bio → block layer → driver → disk
Storage types: HDD (slow, sequential), SATA (good), NVMe (best)
TRIM: Maintaining SSD performance
Physical reality: DMA, PCIe, interrupt completion
```

---

## Key Takeaways

**Unified abstraction:**

```
Many storage devices:
├── HDD: mechanical, slow random access
├── SATA SSD: flash, SATA interface limit
└── NVMe SSD: flash, PCIe direct connection

All look same to filesystem:
└── submit_bio(bio)
    Block layer handles differences
    Filesystem code is device-independent
```

**blk-mq architecture:**

```
Old: Single queue (lock contention)
└── Bottleneck for fast devices

Modern: Multi-queue
├── Per-CPU software queues (no contention)
├── Multiple hardware queues (match device)
└── Scales to 1,000,000 IOPS

NVMe can fully utilize fast hardware
```

**I/O schedulers optimize:**

```
HDD: Sort by location (minimize seeking)
└── BFQ scheduler: fairness between processes

SSD: Less critical, but still useful
└── mq-deadline: guarantee latency bounds

NVMe: Skip scheduler overhead
└── none: submit immediately
```

**Page cache crucial:**

```
Read: Check cache first
└── Hit: instant (no disk)
    Miss: bio to disk, fill cache

Write: Write to cache
└── Mark dirty, return immediately
    Writeback later (batched)

Direct I/O: Bypass cache
└── Databases control their own caching
```

**Performance difference:**

```
HDD:
└── 100-200 IOPS
    10ms latency
    Sequential friendly

SATA SSD:
└── 75,000 IOPS
    80μs latency
    SATA interface limited

NVMe SSD:
└── 600,000 IOPS
    120μs latency
    PCIe bandwidth, many queues
    Modern design, maximum performance
```

---

## The Big Picture

**Complete storage stack:**

```
Application
    write(fd, data, size)
        │
        ↓
VFS layer
    file → inode → operations
        │
        ↓
Filesystem (ext4, btrfs, XFS)
    logical block → physical sector
        │
        ↓
Page Cache (mm/filemap.c)
    cache hit: return
    cache miss or write: continue
        │
        ↓
Block Layer (THIS TOPIC)
    bio creation
    request merging
    I/O scheduling
    blk-mq queuing
        │
        ↓
Device Driver (nvme, sd, etc)
    Hardware commands
    DMA setup
    Doorbell ring
        │
        ↓
Hardware (NVMe SSD)
    DMA transfer
    NAND flash write
    Completion interrupt
        │
        ↓
Interrupt (MSI-X)
    nvme_irq()
    completion callback
    bi_end_io()
        │
        ↓
Page marked clean
Data on disk
```

---

## You're Ready!

With this knowledge, you can:
- Understand how writes reach disk
- Choose appropriate I/O scheduler
- Optimize for different storage types
- Debug block I/O performance
- Understand NVMe advantages
- Build storage drivers

> **The block layer is the unified interface to storage - you now know how Linux makes every storage device speak one language while maintaining maximum performance.**

---

**The abstraction that makes storage universal.**
