# The Kernel's Fastest Cache - Directory Entry Cache

> Ever wondered why opening the same file 1000 times doesn't trigger 1000 disk reads? How Linux handles millions of file lookups per second? Why your system stays fast even with deep directory hierarchies?

**You're about to find out!**

---

## What's This About?

This is **fs/dcache.c** - the dentry cache that makes Linux file access lightning fast!

Here's where it fits in the VFS ecosystem:

```
fs/
├── vfs-core/
│   ├── namei.c             ← Path lookup
│   ├── dcache.c            ← Dentry cache (THIS!)
│   │   ├── Hash table      ← O(1) lookup
│   │   ├── RCU lockless    ← No lock contention
│   │   ├── Negative cache  ← Cache "not found"
│   │   ├── LRU reclaim     ← Memory pressure handling
│   │   └── Dentry tree     ← Filesystem in memory
│   ├── inode.c             ← File metadata
│   ├── file.c              ← Open file state
│   └── file_table.c        ← File descriptor table
├── io/                     ← Read/write operations
└── ext4/                   ← Real filesystem
```

**Location in code:**

```
fs/
├── dcache.c                ← Dentry cache (3500+ lines)
│   ├── __d_lookup_rcu()    ← RCU lockless lookup
│   ├── d_lookup()          ← Locked fallback
│   ├── d_add()             ← Add to cache
│   ├── d_instantiate()     ← Attach inode
│   ├── dget()/dput()       ← Reference counting
│   ├── shrink_dcache_sb()  ← Memory reclaim
│   └── d_alloc()           ← Allocate dentry
│
include/linux/
└── dcache.h                ← Key structures

Entry from namei.c:
└── Every component lookup hits dcache first
    __d_lookup_rcu() called thousands of times per second
```

**This covers:**
- In-memory filename to inode cache
- Global hash table with RCU lockless access
- Negative caching (cache "file not found")
- LRU-based memory reclaim
- Dentry tree (filesystem hierarchy in memory)
- Hard link handling via alias lists
- Complete cache hit/miss flows

---

# The Fundamental Problem

**Physical Reality:**

```
Common file accesses on busy system:
├── /lib/x86_64-linux-gnu/libc.so.6    (every process!)
├── /usr/lib/locale/locale-archive      (every C program)
├── /etc/ld.so.cache                    (every dynamic link)
├── /proc/self/maps                     (memory queries)
└── Accessed THOUSANDS of times per second

Each open() requires path lookup:
    "/etc/passwd"
    └── Find "/" → find "etc" → find "passwd"
        Each step needs directory search!

Without caching:
    Every lookup = disk read
    /lib/libc.so.6 accessed 1000x/sec
    = 1000 disk reads/second
    = 5 seconds of disk I/O (HDD)
    = System unusable!
```

**Directory search complexity:**

```
ext4 directory with 10,000 files:
    Linear search: O(n) = 10,000 comparisons
    HTree (hash tree): O(log n) but still disk I/O
    
Every string comparison:
    Read directory block from disk
    Search entries
    Compare name strings
    
For /usr/lib/x86_64-linux-gnu/:
    Thousands of files
    Every access = expensive search
    Without cache = performance disaster
```

---

# Without Dentry Cache

**Imagine no dcache.c:**

```
Every open() needs full path walk:
├── Read / from disk
├── Search / for "etc"
├── Read /etc from disk
├── Search /etc for "passwd"
├── Read /etc/passwd inode
└── Finally open file!

Multiple accesses to same file:
    open("/etc/passwd")  → 3 disk reads
    open("/etc/passwd")  → 3 disk reads (again!)
    open("/etc/passwd")  → 3 disk reads (again!)
    
    No memory of previous lookups!

Tab completion disaster:
    User types: ls xyz<TAB>
    Shell checks:
        /usr/bin/xyz?        → disk read → not found
        /usr/local/bin/xyz?  → disk read → not found
        /bin/xyz?            → disk read → not found
        
    Each completion = dozens of disk reads!

System load:
    1000 processes/second start
    Each needs /lib/libc.so.6
    = 5000+ disk reads/second
    = Disk saturated!
    = System crawls!
```

**Complete chaos without caching!**

---

# The Layered Solution

**dcache.c provides:**

```
Layer 1: HASH TABLE
In-memory hash table:
└── Maps (parent, name) → dentry
    O(1) average lookup time
    RCU lockless access (no contention)
    Global table shared by all filesystems

Layer 2: NEGATIVE CACHING
Cache "file not found":
└── stat("/nonexistent")  → disk read (once)
    stat("/nonexistent")  → cache hit! (instant)
    Tab completion doesn't thrash disk
    PATH searches don't kill performance

Layer 3: DENTRY TREE
Filesystem hierarchy in memory:
└── Parent/child relationships
    Sibling links
    Full directory tree navigable
    No disk access for tree walks

Layer 4: LRU RECLAIM
Intelligent memory management:
└── Unused dentries → LRU list
    Memory pressure → free from tail
    Per-superblock LRU for fast umount
    Keeps hot entries, discards cold

Layer 5: REFERENCE COUNTING
Lockref optimization:
└── Combined lock + refcount in 64 bits
    Single atomic operation
    No separate lock for refcount increment
    Massive scalability win

Result:
└── 99%+ cache hit rate
    ~15ns per lookup (memory speed)
    Millions of lookups per second
    Zero disk access for hot paths
```

---

# 1. Key Data Structures

## struct dentry

**The directory entry:**

```c
struct dentry {
    // RCU lockless access
    seqcount_spinlock_t d_seq;      // Sequence for RCU
    
    // Hash table linkage
    struct hlist_bl_node d_hash;    // In hash bucket
    
    // Tree structure
    struct dentry *d_parent;        // Parent dentry
    struct qstr d_name;             // Name (hash + string)
    struct inode *d_inode;          // The inode (NULL = negative)
    
    // Inline short name optimization
    unsigned char d_iname[DNAME_INLINE_LEN];  // 36 bytes
    
    // Reference counting (combined lock + count!)
    struct lockref d_lockref;
    
    // Operations
    const struct dentry_operations *d_op;
    struct super_block *d_sb;       // Which filesystem
    
    // LRU for reclaim
    struct list_head d_lru;         // LRU list linkage
    
    // Children
    struct list_head d_child;       // In parent's d_subdirs
    struct list_head d_subdirs;     // Our children
    
    // Aliases (hard links)
    struct hlist_node d_alias;      // In inode->i_dentry
    
    // Flags and metadata
    unsigned int d_flags;           // DCACHE_* flags
    unsigned long d_time;           // Revalidation time
    void *d_fsdata;                 // Filesystem private
};

Size: ~192 bytes per dentry

Usage:
    Millions per system
    200-500MB typical memory usage
    Worth it! Makes system 1000× faster
```

## d_name - Inline Optimization

**Smart name storage:**

```
struct qstr d_name:
    union {
        struct {
            u32 hash;       // Precomputed hash
            u32 len;        // Length
        };
        u64 hash_len;       // Atomic 64-bit compare
    };
    const unsigned char *name;  // The string

Short names (≤ 36 chars):
    name points to d_iname[]
    Stored INSIDE dentry struct
    No extra allocation needed
    Better cache locality
    
    Example:
        "passwd" → stored in d_iname
        "libc.so.6" → stored in d_iname
        
Long names (> 36 chars):
    name points to kmalloc'd memory
    Separate allocation
    
    Example:
        "very_long_filename_that_exceeds_36_chars.txt"
        → separate allocation

Statistics:
    99% of filenames < 36 chars
    Inline storage = huge win!
```

## lockref - The Clever Trick

**Atomic lock + refcount:**

```c
struct lockref {
    union {
        aligned_u64 lock_count;     // 64-bit atomic
        struct {
            spinlock_t lock;        // 32 bits
            int count;              // 32 bits
        };
    };
};

The optimization:
    On x86-64: CMPXCHG can operate on 64 bits
    
    dget() (increment refcount):
        Old way:
            spin_lock(&dentry->d_lock)
            dentry->d_count++
            spin_unlock(&dentry->d_lock)
            
        New way (lockref):
            cmpxchg64(&dentry->d_lockref.lock_count,
                     old_value, new_value)
            
            Single atomic operation!
            No lock acquisition!
            
Performance:
    Common case: no lock needed
    Rare case: fall back to locked
    
    On 64-core system:
        Before lockref: severe contention
        After lockref: linear scaling
```

## dentry_operations

**Filesystem callbacks:**

```c
struct dentry_operations {
    // Still valid?
    int (*d_revalidate)(struct dentry *, unsigned int);
    
    // Compute hash
    int (*d_hash)(const struct dentry *, struct qstr *);
    
    // Compare names
    int (*d_compare)(const struct dentry *,
                     unsigned int, const char *,
                     const struct qstr *);
    
    // Should delete?
    int (*d_delete)(const struct dentry *);
    
    // Being freed
    void (*d_release)(struct dentry *);
    
    // Inode being released
    void (*d_iput)(struct dentry *, struct inode *);
    
    // Generate name
    char *(*d_dname)(struct dentry *, char *, int);
};

Use cases:
    d_revalidate: NFS checks if remote file changed
    d_hash/d_compare: Case-insensitive filesystems
    d_dname: Pipes, sockets (generate "pipe:[12345]")
```

---

# 2. The Hash Table

## Global Hash Table

**Core lookup structure:**

```
dentry_hashtable[] - allocated at boot

Size scales with RAM:
    512MB:  128K buckets
    4GB:    512K buckets
    16GB:   2M buckets
    
Each bucket:
    hlist_bl_head (hash list with bit-lock)
    
Structure:
    bucket[0]:  dentry_A → dentry_B → NULL
    bucket[1]:  NULL
    bucket[2]:  dentry_C → NULL
    bucket[3]:  dentry_D → dentry_E → NULL
    ...
    bucket[N]:  ...

Hash key computation:
    hash = hash_32(parent_dentry_address ^ name_hash,
                   d_hash_shift)
    
Why parent + name?
    "passwd" can exist in:
        /etc/passwd
        /home/passwd
        /tmp/passwd
        
    Same name, different parents!
    Must hash both for uniqueness
```

## __d_lookup_rcu() - Lockless Fast Path

**The most-called function:**

```
__d_lookup_rcu(parent, name, seqp):

Purpose:
    Find dentry for (parent, name) without locks
    
Flow:
    Compute hash:
        hash = name->hash
        
    Find bucket:
        bucket = dentry_hashtable[hash & d_hash_mask]
        
    Walk bucket chain (RCU - no locks!):
        
        hlist_bl_for_each_entry_rcu(dentry, bucket):
            
            Quick checks (cheap comparisons first):
                
                Hash + length match?
                if (dentry->d_name.hash_len != name->hash_len)
                    continue
                    
                Same parent?
                if (dentry->d_parent != parent)
                    continue
                    
            Passed quick checks - read sequence:
                seq = read_seqcount_begin(&dentry->d_seq)
                
            String compare (only if above match):
                if (memcmp(dentry->d_name.name,
                          name->name, len) != 0)
                    continue
                    
            Found it!
                *seqp = seq
                return dentry
                
    Not found:
        return NULL

Performance:
    Common case (short chain):
        0-2 entries per bucket
        Single iteration
        ~10-20ns total
        
    Hash collision:
        Multiple entries
        Each: hash compare, parent compare
        Rare string compare
        ~30-50ns total
        
    Zero locks taken!
    Why RCU is critical for scalability
```

## d_lookup() - Locked Fallback

**When RCU fails:**

```
d_lookup(parent, name):

When used:
    RCU seqcount retry
    Need to hold reference (dget)
    Modify dentry state
    
Flow:
    rcu_read_lock()
    
    dentry = __d_lookup(parent, name)
    
    if (dentry)
        lockref_get(&dentry->d_lockref)
        
    rcu_read_unlock()
    
    return dentry

Performance:
    Takes RCU read lock (cheap)
    May take dentry lock (rare)
    ~50-100ns
    
    Still fast! Just not as fast as pure RCU
```

---

# 3. Dentry States

## Four States

**State machine:**

```
STATE 1: USED (active)
    d_lockref.count > 0
    d_inode != NULL
    In use by:
        - namei walking path
        - Open files (file->f_path.dentry)
        - Current working directory
        - Mount point
        
    NOT in LRU list
    Cannot be reclaimed
    
    Example:
        /etc/passwd open by 5 processes
        d_lockref.count = 5
        Stays in memory until all close

STATE 2: UNUSED (cached)
    d_lockref.count == 0
    d_inode != NULL
    
    In LRU list (can be reclaimed)
    Still in hash table (still findable)
    
    This is the "hot cache"!
    
    Example:
        /lib/libc.so.6 was accessed
        All processes closed it
        But kept in cache for next access
        
STATE 3: NEGATIVE (cached miss)
    d_lockref.count == 0
    d_inode == NULL
    
    File doesn't exist!
    But we cache this fact
    
    In LRU list
    In hash table
    
    Example:
        stat("/home/user/.vimrc_nonexistent")
        → ENOENT
        
        Negative dentry created!
        Next stat() → instant ENOENT
        No disk access!
        
STATE 4: KILLED (dying)
    Removed from hash table
    d_flags: DCACHE_UNHASHED
    Waiting for refcount → 0
    
    Example:
        unlink("/tmp/file")
        while someone has it open
        
        Dentry unhashed (unfindable)
        But not freed (file still open)
        Freed when last close()
```

## State Transitions

**The lifecycle:**

```
        d_alloc()
            ↓
        NEGATIVE
            ↓ d_instantiate(inode)
         USED
            ↓ dput() (count → 0)
        UNUSED ←──────┐
            ↓          │
    Memory pressure    │ dget() (reused)
    or d_delete()      │
            ↓          │
         KILLED        │
            ↓          │
     dentry_free() ────┘
            ↓
          freed

Example flow - creating file:
    1. d_alloc() → NEGATIVE
    2. vfs_create() → USED (count > 0)
    3. close() → UNUSED (count = 0, in LRU)
    4. open() again → USED (cache hit!)
    5. close() → UNUSED
    6. Memory pressure → KILLED → freed
```

---

# 4. LRU and Memory Reclaim

## The LRU List

**Least Recently Used tracking:**

```
Global per-superblock LRU:
    sb->s_dentry_lru
    
Structure:
    [MRU end] ←─── recently used ───→ [LRU end]
    
    recent_dentry ← ... ← old_dentry

When dentry becomes UNUSED:
    dput() decrements count to 0
    
    Add to MRU end:
        list_add(&dentry->d_lru, &sb->s_dentry_lru)
        
When dentry reused:
    dget() increments count
    
    Remove from LRU:
        list_del_init(&dentry->d_lru)
        
    Now USED again!

Memory pressure:
    Scan from LRU end (oldest first)
    Free dentries that haven't been used
```

## shrink_dcache_sb()

**Superblock-specific reclaim:**

```
shrink_dcache_sb(sb):

Called when:
    umount filesystem
    Memory pressure
    Explicit drop_caches
    
Flow:
    Walk sb->s_dentry_lru:
        
        For each dentry:
            
            Still referenced?
            if (dentry->d_lockref.count > 0)
                Move to MRU end
                Continue
                
            Has children?
            if (!list_empty(&dentry->d_subdirs))
                Can't free parent before children
                Continue
                
            Remove from LRU:
                d_lru_del(dentry)
                
            Unhash:
                d_drop(dentry)
                Remove from hash table
                
            Kill:
                dentry_kill(dentry)
                
                If count still 0:
                    Call d_op->d_release()
                    Call d_op->d_iput()
                    Free memory

On umount:
    shrink_dcache_sb() must succeed
    All dentries for filesystem freed
    Otherwise: umount fails (EBUSY)
```

## Per-Superblock LRU

**Why per-filesystem LRU:**

```
Without per-sb LRU:
    Single global LRU
    All dentries mixed together
    
    umount /mnt/usb:
        Must scan ENTIRE LRU
        Find all dentries for /mnt/usb
        Very expensive!
        
With per-sb LRU:
    Each filesystem has own LRU
    
    umount /mnt/usb:
        shrink_dcache_sb(/mnt/usb)
        Scan only /mnt/usb dentries
        Fast!
        
Memory reclaim:
    Can target specific filesystems
    Balance between filesystems
    Fair reclaim
```

---

# 5. Dentry Tree - Filesystem in Memory

## Tree Structure

**Hierarchy representation:**

```
In-memory dentry tree:

        root (/)
        │
        ├─ etc/
        │  ├─ passwd (inode 1234)
        │  ├─ shadow (inode 1235)
        │  └─ hosts (inode 1236)
        │
        ├─ usr/
        │  └─ bin/
        │     ├─ bash (inode 5678)
        │     └─ ls (inode 5679)
        │
        └─ home/
           └─ user/
              └─ .bashrc (inode 9012)

Each dentry maintains:
    d_parent → parent dentry
    d_subdirs → list of children
    d_child → sibling linkage
    d_inode → the inode

Navigation:
    Parent: dentry->d_parent
    Children: list_for_each(dentry->d_subdirs)
    Inode: dentry->d_inode

This IS the VFS tree!
    Not on disk (disk has blocks)
    Reconstructed in memory
    Cached for fast access
```

## Hard Links and d_alias

**Multiple paths to same inode:**

```
Hard link scenario:
    /home/user/file     → inode 1234
    /home/user/link     → inode 1234 (same!)
    
Two dentries, one inode!

inode structure:
    inode->i_dentry: [dentry_A, dentry_B]
    
    Alias list of all dentries for this inode

Each dentry:
    d_alias: linkage in inode->i_dentry list
    
Operations:
    stat(/home/user/file):
        Find dentry_A
        dentry_A->d_inode = inode 1234
        
    stat(/home/user/link):
        Find dentry_B
        dentry_B->d_inode = inode 1234 (same!)
        
    Same size, permissions, timestamps!

d_splice_alias():
    When creating dentry for inode
    Check if inode already has dentries
    If yes: link into alias list
    
inode deletion:
    Must unhash ALL dentries in alias list
    Otherwise: dangling dentry pointers
```

---

# 6. Adding Dentries

## d_alloc() and d_instantiate()

**Two-step creation:**

```
d_alloc(parent, name):
    
    Allocate struct dentry
    From dentry slab cache
    
    Set fields:
        dentry->d_parent = parent
        dentry->d_name = name
        dentry->d_inode = NULL (not yet!)
        dentry->d_lockref.count = 1
        
    Add to parent's children:
        list_add(&dentry->d_child, &parent->d_subdirs)
        
    NOT in hash table yet!
    NOT findable by lookup!
    
    Return dentry

d_instantiate(dentry, inode):
    
    Attach inode:
        dentry->d_inode = inode
        
    Add to inode's alias list:
        hlist_add_head(&dentry->d_alias,
                      &inode->i_dentry)
                      
    Still not in hash table!

d_add(dentry, inode):
    
    Combines:
        d_instantiate(dentry, inode)
        __d_add(dentry, inode)
        
    __d_add():
        Compute hash
        Insert into hash table
        Now findable!
```

## Complete Creation Flow

**Example: mkdir /tmp/newdir**

```
sys_mkdir("/tmp/newdir", 0755):
    
    namei walks to /tmp:
        Find tmp_dentry
        
    Check if "newdir" exists:
        __d_lookup_rcu(tmp_dentry, "newdir")
        → NULL (doesn't exist)
        
    Allocate dentry:
        new_dentry = d_alloc(tmp_dentry, "newdir")
        
        Now:
            new_dentry->d_parent = tmp_dentry
            new_dentry->d_name = "newdir"
            new_dentry->d_inode = NULL
            
    Ask filesystem to create:
        ext4_mkdir(tmp_inode, new_dentry, mode):
            
            Allocate inode on disk
            new_inode = ext4_new_inode()
            
            Write directory blocks
            Initialize . and .. entries
            
            Attach to dentry:
                d_instantiate(new_dentry, new_inode)
                
                Now:
                    new_dentry->d_inode = new_inode
                    
    Add to cache:
        d_add(new_dentry, new_inode)
        
        Insert into hash table
        
    Return success

Future lookups:
    stat("/tmp/newdir")
    → __d_lookup_rcu() → cache HIT!
    → Instant return
    → No disk access
```

---

# 7. Negative Dentries

## Caching Non-Existence

**The clever optimization:**

```
Without negative caching:
    stat("/usr/bin/nonexistent")
    
    Every time:
        Read /usr/bin directory
        Search for "nonexistent"
        Not found → ENOENT
        
    Repeat 1000 times:
        1000 directory searches!
        
With negative caching:
    First stat():
        Read /usr/bin directory
        Search for "nonexistent"
        Not found
        
        Create negative dentry:
            d_alloc(usr_bin_dentry, "nonexistent")
            d_add(dentry, NULL)
            
            dentry->d_inode = NULL!
            
        In hash table now!
        
    Next 999 times:
        __d_lookup_rcu()
        → Cache HIT!
        → d_inode == NULL
        → ENOENT instantly
        
    Zero disk reads after first!
```

## Use Cases

**Where negative caching shines:**

```
Tab completion:
    User types: ls xyz<TAB>
    
    Shell checks:
        /usr/bin/xyz?
        /usr/local/bin/xyz?
        /bin/xyz?
        /usr/sbin/xyz?
        ... 10+ paths
        
    First completion:
        10 directory searches
        10 negative dentries created
        
    Next completion:
        10 cache hits
        Instant!

PATH search:
    execve("python")
    
    Searches:
        /usr/local/bin/python?
        /usr/bin/python? ← found!
        
    First run:
        1 directory search (miss)
        1 directory search (hit)
        1 negative dentry
        
    Next 1000 runs:
        1 cache hit (negative)
        1 cache hit (positive)
        Zero disk access!

Build systems:
    Makefile checks:
        File exists? No → rebuild
        
    Thousands of checks
    Mostly for files that don't exist
    Negative cache = fast builds
```

## Negative to Positive Transition

**Creation after miss:**

```
Sequence:
    stat("/tmp/newfile") → ENOENT
        Create negative dentry
        
    Later: create("/tmp/newfile")
        d_lookup() finds negative dentry
        
        d_instantiate(dentry, new_inode)
        
        dentry->d_inode = new_inode
        No longer negative!
        
    stat("/tmp/newfile") → success
        Same dentry, now positive
        
Automatic transition!
No need to invalidate cache
```

---

# 8. Complete Flow - dcache in Action

## Scenario: Firefox Opens Certificate File

**Opening /etc/ssl/certs/ca-certificates.crt**

### Initial State

```
System running for hours
dcache warm (populated):
    / → cached
    /etc → cached
    /etc/ssl → cached
    /etc/ssl/certs → cached
    /etc/ssl/certs/ca-certificates.crt → cached
    
All in hash table
All with valid inodes
```

### Component 1: "etc"

```
namei.c: __d_lookup_rcu(root_dentry, "etc")

Compute hash:
    hash = hash_32(root_dentry ^ hash("etc"))
    bucket = hash & d_hash_mask
    
Find bucket:
    bucket_ptr = dentry_hashtable[bucket]
    
Walk chain (RCU):
    First entry:
        parent == root_dentry? YES
        name.hash == hash("etc")? YES
        name.len == 3? YES
        
    Read sequence:
        seq = read_seqcount_begin(&dentry->d_seq)
        seq = 42
        
    String compare:
        memcmp(dentry->d_name.name, "etc", 3)
        → 0 (match!)
        
    Found:
        *seqp = 42
        return etc_dentry

Time: ~15ns
```

### Component 2: "ssl"

```
__d_lookup_rcu(etc_dentry, "ssl")

Hash bucket:
    bucket = hash_32(etc_dentry ^ hash("ssl"))
    
Walk:
    First entry: match!
    
Return ssl_dentry

Time: ~15ns
```

### Component 3: "certs"

```
__d_lookup_rcu(ssl_dentry, "certs")

Cache HIT!

Return certs_dentry

Time: ~15ns
```

### Component 4: "ca-certificates.crt"

```
__d_lookup_rcu(certs_dentry, "ca-certificates.crt")

Longer name (20 chars):
    Hash and length match
    Parent match
    
    String compare:
        memcmp(dentry->d_name.name,
               "ca-certificates.crt", 20)
        → 0 (match!)
        
Cache HIT!

Return ca_certs_dentry

Time: ~20ns (longer string)
```

### Summary

```
Total dcache lookups: 4
Total time: ~15 + 15 + 15 + 20 = ~65ns

Zero disk reads!
Zero filesystem calls!
Zero locks taken! (all RCU)

Compare to cold cache:
    4 ext4_lookup() calls
    4 directory block reads
    ~400μs total
    
    6000× slower!
    
dcache makes this possible!
```

---

# 9. Physical Reality

## Memory Usage

**Typical system statistics:**

```
Each dentry:
    struct dentry: ~192 bytes
    Short name: inline (free!)
    Long name: extra kmalloc
    
System with 4GB RAM:
    Active dentries: 500K - 2M
    Memory used: 200-500MB
    
Is this too much?
    NO! This memory makes system usable
    99%+ of lookups avoid disk
    Worth every byte!

/proc/sys/fs/dentry-state:
    nr_dentry: 1,523,847       # Total dentries
    nr_unused: 891,234         # In LRU (reclaimable)
    age_limit: 45              # Seconds
    want_pages: 0              # Memory pressure

View with:
    cat /proc/sys/fs/dentry-state
    
Drop caches (testing):
    echo 2 > /proc/sys/vm/drop_caches
    # Clears dcache (and icache)
```

## Hash Table Sizing

**Scaling with RAM:**

```
At boot: alloc_large_system_hash()

Formula:
    numentries = nr_kernel_pages / 16
    Round to power of 2
    
Examples:
    1GB RAM:  ~65K buckets
    4GB RAM:  ~256K buckets
    16GB RAM: ~1M buckets
    64GB RAM: ~4M buckets
    
Average chain length:
    Total dentries / buckets
    
    1M dentries / 256K buckets = ~4 per bucket
    
Acceptable collision rate!

Collision handling:
    Linear chaining
    RCU walk
    Cheap hash/parent compare
    Expensive string compare (rare)
```

## Cache Hit Rates

**Production measurements:**

```
Busy web server:
    Lookups: 10,000,000/sec
    Hits: 9,985,000/sec
    Hit rate: 99.85%
    
    Those 15K misses/sec:
        → ext4_lookup()
        → Disk reads
        → New dentries added
        
    Those 9.985M hits/sec:
        → Pure memory
        → ~15ns each
        → ~150ms total time
        
Without dcache:
    10M lookups at 100μs each
    = 1000 seconds
    = System unusable!

Desktop system:
    Hit rate: 99.9%+
    Most accesses to same files
    /lib/libc.so.6
    /usr/bin/bash
    Always cached

First boot:
    Cold cache
    Every lookup misses
    Slow!
    
After warmup (30 seconds):
    Hot cache
    99%+ hits
    Fast!
```

---

# 10. Connections to Other Topics

## Connection to namei.c

**Primary user:**

```
Every walk_component() in namei.c:
    __d_lookup_rcu(parent, name)
    
Path lookup flow:
    namei → dcache lookup
    
    Cache hit:
        Continue instantly
        
    Cache miss:
        ext4_lookup()
        d_add()
        Continue
        
Tight coupling:
    namei can't work without dcache
    dcache designed for namei patterns
    
Understanding both:
    See complete picture
    How paths become inodes
```

## Connection to inode.c

**Dentry to inode relationship:**

```
dentry→inode linkage:
    dentry->d_inode = inode
    inode->i_dentry = list of dentries
    
Multiple dentries per inode:
    Hard links!
    /home/file and /tmp/link
    → Same inode
    → Different dentries
    
iput() when refcount → 0:
    Inode may be freed
    But dentries can stay (cached)
    
    Later access:
        Dentry exists (cache hit)
        But d_inode points to freed inode!
        
    Solution:
        iget() on cache hit
        Recreates inode if needed
        
Separate LRU:
    Dentries have LRU
    Inodes have LRU
    Independent reclaim
```

## Connection to ext4

**Filesystem integration:**

```
dcache miss → ext4_lookup():
    
    ext4_lookup(dir, dentry, flags):
        Search directory blocks
        Find entry for name
        Get inode number
        
        inode = ext4_iget(inode_number)
        
        d_add(dentry, inode)
        
    Now in dcache!
    Next lookup: cache hit!

ext4 only sees first lookup:
    All subsequent: dcache handles
    ext4 unaware of caching
    Clean separation
    
Negative dentries:
    ext4_lookup() returns NULL
    VFS creates negative dentry
    d_add(dentry, NULL)
```

## Connection to Memory Management

**SLUB allocator:**

```
Dentry allocation:
    dentry_cache = kmem_cache_create(
        "dentry",
        sizeof(struct dentry),
        ...)
        
    Per-CPU caches
    Fast allocation
    Fixed size objects
    
    d_alloc():
        kmem_cache_alloc(dentry_cache, GFP_KERNEL)
        
Memory reclaim:
    kswapd → shrinker_list
    
    dcache_shrinker:
        shrink_dcache_memory()
        Walk LRU lists
        Free dentries
        Return memory to SLUB
        
Shrinker priority:
    Low memory: aggressive
    Plenty memory: lazy
```

## Connection to VFS Mount

**Superblock relationship:**

```
Each dentry:
    dentry->d_sb = superblock
    
Per-superblock tracking:
    sb->s_dentry_lru
    All dentries for this filesystem
    
umount():
    shrink_dcache_sb(sb)
    
    Must free ALL dentries:
        Otherwise: dangling pointers
        Kernel crash!
        
    Walk s_dentry_lru
    Force free all dentries
    
    If can't free:
        umount fails (EBUSY)
        
Mount crossing:
    Dentry at mount point
    d_flags: DCACHE_MOUNTED
    
    namei sees flag
    Switches to child filesystem
```

---

# Summary

## What You've Mastered

**dcache.c is the speed multiplier:**

```
What it does:
    In-memory cache of dentries
    Maps (parent, name) → inode
    O(1) hash table lookup
    RCU lockless access
    
Key mechanisms:
    Global hash table
    Negative caching
    LRU-based reclaim
    Dentry tree structure
    Reference counting
```

---

## Key Takeaways

**The hash table:**

```
Structure:
    Array of buckets
    Each bucket: linked list
    RCU walkable
    
Hash key:
    hash(parent, name)
    Both needed for uniqueness
    
Lookup:
    __d_lookup_rcu()
    ~15ns average
    Zero locks (RCU)
    
Collision:
    Linear chaining
    Rare (good distribution)
    Quick rejections
```

**Four states:**

```
USED: refcount > 0, in use
UNUSED: refcount = 0, cached, in LRU
NEGATIVE: d_inode = NULL, cached miss
KILLED: unhashed, dying

Transitions:
    Allocation → use → cache → reclaim
    Or: cache → reuse → use
```

**Negative dentries:**

```
Cache "file not found":
    d_inode = NULL
    Still in hash table
    Still findable
    
Benefits:
    Tab completion fast
    PATH search fast
    Build systems fast
    
Transition:
    create() makes negative → positive
    Automatic!
```

**LRU reclaim:**

```
Per-superblock LRU:
    sb->s_dentry_lru
    
Memory pressure:
    Scan from tail
    Free unused dentries
    
umount:
    Must free all dentries
    Prevent dangling pointers
```

**Performance:**

```
Hit rate: 99%+
Lookup time: ~15ns (RCU)
Memory: 200-500MB typical
Worth it: 1000× speedup

Without dcache:
    Every lookup = disk read
    System unusable
```

**Physical reality:**

```
Each dentry: ~192 bytes
Inline names: free (36 chars)
Long names: extra allocation

Millions of dentries
Hash table: 256K-4M buckets
Scales with RAM
```

**Connections:**

```
namei.c: primary user
inode.c: dentry→inode linkage
ext4: cache miss handler
SLUB: dentry allocation
super.c: per-sb LRU
vmscan: shrinker framework
```

---

## You're Ready!

With this knowledge, you can:
- Understand why file access is fast
- Debug cache inefficiencies
- Analyze dcache memory usage
- Understand negative caching benefits
- See how RCU enables scalability
- Know why umount can fail (EBUSY)

> **The dentry cache is Linux's secret weapon - millions of lookups per second, 99%+ hit rate, and the reason your system feels responsive even with deep directory hierarchies.**

---

**The cache that makes everything fast.**