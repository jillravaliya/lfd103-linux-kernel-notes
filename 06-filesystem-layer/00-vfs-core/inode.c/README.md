# What a File Really Is - The Inode

> Ever wondered what a file actually is underneath the filename? Why hard links exist? How deleted files stay alive while open? Why unlinking a file doesn't immediately free space?

**You're about to find out!**

---

## What's This About?

This is **fs/inode.c** - the inode manager, where the real file lives regardless of its name!

Here's where it fits in the VFS ecosystem:

```
fs/
├── vfs-core/
│   ├── namei.c             ← Path lookup (finds dentries)
│   ├── dcache.c            ← Dentry cache (name→inode cache)
│   ├── inode.c             ← Inode manager (THIS!)
│   │   ├── Inode cache     ← In-memory inode storage
│   │   ├── Reference count ← Track users
│   │   ├── Dirty tracking  ← Mark changes
│   │   ├── LRU reclaim     ← Memory management
│   │   └── Filesystem ops  ← Abstract interface
│   ├── file.c              ← Open file state
│   └── file_table.c        ← File descriptor table
```

**Location in code:**

```
fs/
├── inode.c                 ← Inode management (2500+ lines)
│   ├── iget_locked()       ← Get inode (cache or disk)
│   ├── iput()              ← Release reference
│   ├── alloc_inode()       ← Allocate new inode
│   ├── evict_inode()       ← Remove from memory
│   ├── __mark_inode_dirty()← Mark needs write
│   ├── new_inode()         ← Create brand new
│   └── insert_inode_hash() ← Add to cache
│
include/linux/
└── fs.h                    ← struct inode definition
    ├── File metadata       ← Size, permissions, times
    ├── Operations          ← read, write, lookup
    ├── Reference counting  ← i_count, i_nlink
    └── Cache linkage       ← Hash table, LRU
```

**This covers:**
- What an inode really is (not the filename!)
- Inode cache (hash table for fast access)
- Reference counting (tracking users)
- Dirty tracking (changes needing write)
- Hard links (multiple names, one file)
- Complete stat() flow with timing
- Filesystem abstraction (ext4, proc, tmpfs)

---

# The Fundamental Problem

**Physical Reality:**

```
Users think in terms of filenames:
├── /etc/passwd
├── /home/user/document.txt
├── /tmp/myfile
└── Paths are what we see

But filesystems operate on inodes:
├── inode 1234567 (file data, metadata)
├── inode 8901234 (directory blocks)
├── inode 5678901 (symbolic link)
└── Numbers identify the real files

The disconnect:
├── Filename is just a label
├── Inode is the actual file
├── One inode can have many names (hard links)
└── Must bridge this gap!
```

**The complexity:**

```
Multiple processes accessing same file:
    Process A: open("/etc/passwd")
    Process B: open("/etc/passwd")
    Process C: open("/etc/passwd")
    
    All must see SAME metadata:
        Same size
        Same permissions
        Same modification time
        
    Changes by one visible to all!
    
Hard links:
    ln /home/user/original /tmp/link
    
    Two paths, ONE file:
        /home/user/original → inode 123456
        /tmp/link → inode 123456 (same!)
        
    Delete /home/user/original:
        inode STILL EXISTS!
        Data accessible via /tmp/link
        
    Only when ALL names deleted:
        And no process has it open
        Then inode freed
        Then data deleted
        
Deleted while open:
    Process: fd = open("/tmp/file")
    Another: unlink("/tmp/file")
    
    File has no path anymore!
    But process can still read/write!
    Data freed only on close()
    
All handled by inode.c!
```

---

# Without Inode Management

**Imagine no inode.c:**

```
Every filesystem reimplements:
├── ext4: own inode cache
├── XFS: different inode cache
├── btrfs: yet another cache
└── No sharing between filesystems!

No unified interface:
├── VFS code must know ext4 internals
├── Different APIs for each filesystem
├── Massive code duplication
└── Maintenance nightmare

No shared metadata:
├── Process A: open file, sees size=1000
├── Process B: open same file, reads disk again
├── B writes, size=2000
├── A doesn't see change
└── Inconsistent view!

No reference counting:
├── When to free inode?
├── Race conditions everywhere
├── Use-after-free bugs
└── Data corruption inevitable

No caching:
├── Every stat() reads disk
├── Every permission check reads disk
├── Millions of unnecessary I/Os
└── System crawls
```

**The chaos:**

```
Filesystem-specific code everywhere:

VFS read():
    if filesystem == ext4:
        ext4_read(file)
    elif filesystem == xfs:
        xfs_read(file)
    elif filesystem == btrfs:
        btrfs_read(file)
    ... 50+ filesystems!
    
Complete disaster!

Delete open file:
    unlink("/tmp/file")
    File data deleted immediately
    Process still has fd open
    read(fd) → corrupted data
    Crash!
    
No hard link support:
    ln command doesn't work
    Copy files instead
    Wastes space
```

---

# The Inode Solution

**Unified in-memory representation:**

```
Layer 1: ABSTRACT INODE
Every file represented by struct inode
└── VFS uses same structure
    Filesystem-independent
    Uniform interface
    Code reuse

Layer 2: INODE CACHE
Global hash table
└── Keyed by (superblock, inode_number)
    O(1) lookup
    Shared across all processes
    Eliminates disk reads

Layer 3: REFERENCE COUNTING
Tracks active users
└── i_count for open files
    i_nlink for directory entries
    Safe deletion only when both zero
    No use-after-free

Layer 4: DIRTY TRACKING
Changes marked for writeback
└── __mark_inode_dirty()
    Writeback list management
    Periodic sync to disk
    fsync() forces immediate write

Layer 5: FILESYSTEM EMBEDDING
Each filesystem extends base inode
└── ext4_inode_info contains struct inode
    container_of() gets full structure
    Zero overhead abstraction
    Maximum flexibility

Layer 6: OPERATIONS POINTERS
Function pointers for filesystem ops
└── i_op->lookup(), i_fop->read()
    Virtual function calls
    Each filesystem implements differently
    Same VFS code everywhere

Result:
└── Filename is just dentry label
    Inode is the real file
    Hard links work naturally
    Delete while open works
    Shared metadata guaranteed
    99%+ cache hit rate
```

---

# 1. Key Data Structures

## struct inode

**The core representation:**

```c
struct inode {
    // Identity
    umode_t i_mode;                 // Type + permissions
    unsigned long i_ino;            // Inode number
    dev_t i_rdev;                   // Device (block/char)
    struct super_block *i_sb;       // Which filesystem
    
    // Ownership
    kuid_t i_uid;                   // Owner user
    kgid_t i_gid;                   // Owner group
    
    // Size and allocation
    loff_t i_size;                  // File size in bytes
    blkcnt_t i_blocks;              // 512-byte blocks used
    
    // Timestamps
    struct timespec64 i_atime;      // Access time
    struct timespec64 i_mtime;      // Modification time
    struct timespec64 i_ctime;      // Status change time
    
    // Reference counting
    atomic_t i_count;               // Active users
    unsigned int i_nlink;           // Hard link count
    
    // Operations (THE KEY!)
    const struct inode_operations *i_op;
    const struct file_operations *i_fop;
    const struct address_space_operations *i_aop;
    
    // Page cache
    struct address_space *i_mapping;
    struct address_space i_data;
    
    // Locking
    struct rw_semaphore i_rwsem;
    
    // Cache linkage
    struct hlist_node i_hash;       // In inode cache
    struct list_head i_lru;         // In LRU when unused
    
    // Writeback tracking
    struct list_head i_io_list;     // Dirty list
    unsigned long i_state;          // I_DIRTY, I_NEW, etc.
    
    // Dentry aliases (hard links)
    struct hlist_head i_dentry;     // All dentries
    
    // Filesystem private data embedded here
};

Size: ~600 bytes base
      ~800 bytes typical (with filesystem data)

The inode IS the file:
    Everything else is metadata about it
```

## i_mode Encoding

**File type and permissions in one field:**

```
i_mode is 16 bits:

Bits 15-12: File type
    S_IFREG  = 0100000  // Regular file
    S_IFDIR  = 0040000  // Directory
    S_IFLNK  = 0120000  // Symbolic link
    S_IFBLK  = 0060000  // Block device (/dev/sda)
    S_IFCHR  = 0020000  // Character device (/dev/null)
    S_IFIFO  = 0010000  // Named pipe (FIFO)
    S_IFSOCK = 0140000  // Unix socket

Bits 11-9: Special bits
    S_ISUID  = 04000    // setuid
    S_ISGID  = 02000    // setgid
    S_ISVTX  = 01000    // sticky bit

Bits 8-0: Permissions
    rwxrwxrwx = 0777
    rw-r--r-- = 0644
    rwxr-xr-x = 0755

Check macros:
    S_ISREG(inode->i_mode)  // Is regular file?
    S_ISDIR(inode->i_mode)  // Is directory?
    S_ISLNK(inode->i_mode)  // Is symlink?

Example:
    i_mode = 0100644
    
    0100000 = S_IFREG (regular file)
    0000644 = rw-r--r-- (owner:rw, group:r, other:r)
    
    S_ISREG() returns true
    Permission bits: 0644
```

## struct inode_operations

**What you can do with an inode:**

```c
struct inode_operations {
    // Directory operations
    struct dentry *(*lookup)(
        struct inode *, 
        struct dentry *, 
        unsigned int);
    // Find entry in directory
    // ext4_lookup() reads directory blocks
    
    int (*create)(
        struct user_namespace *,
        struct inode *, 
        struct dentry *, 
        umode_t, 
        bool);
    // Create new file
    
    int (*mkdir)(
        struct user_namespace *,
        struct inode *, 
        struct dentry *, 
        umode_t);
    // Create directory
    
    int (*unlink)(
        struct inode *, 
        struct dentry *);
    // Remove file (decrement i_nlink)
    
    int (*rmdir)(
        struct inode *, 
        struct dentry *);
    // Remove directory
    
    int (*rename)(
        struct user_namespace *,
        struct inode *, 
        struct dentry *,
        struct inode *, 
        struct dentry *, 
        unsigned int);
    // Rename/move file
    
    int (*link)(
        struct dentry *, 
        struct inode *,
        struct dentry *);
    // Create hard link
    
    int (*symlink)(
        struct user_namespace *,
        struct inode *,
        struct dentry *, 
        const char *);
    // Create symbolic link
    
    // Attribute operations
    int (*getattr)(
        struct user_namespace *,
        const struct path *,
        struct kstat *, 
        u32, 
        unsigned int);
    // stat() implementation
    
    int (*setattr)(
        struct user_namespace *,
        struct dentry *,
        struct iattr *);
    // chmod, chown, truncate
    
    // Permission checking
    int (*permission)(
        struct user_namespace *,
        struct inode *, 
        int);
    // Custom permission logic
    
    // Symlink reading
    const char *(*get_link)(
        struct dentry *,
        struct inode *,
        struct delayed_call *);
    // readlink() implementation
};

Each filesystem implements these:
    ext4: ext4_dir_inode_operations
    procfs: proc_dir_inode_operations
    tmpfs: shmem_dir_inode_operations
    
VFS calls them uniformly:
    inode->i_op->lookup(...)
    Dispatches to correct filesystem
```

---

# 2. The Inode Cache

## Global Hash Table

**Fast inode lookup:**

```
inode_hashtable[] allocated at boot

Key: hash(superblock_pointer, inode_number)

Why both:
    inode 1234 on /dev/sda1 (ext4)
    inode 1234 on /dev/sda2 (ext4)
    Different filesystems!
    Must distinguish!
    
Structure:
    
    [0]:  inode_A → inode_B → NULL
    [1]:  NULL
    [2]:  inode_C → NULL
    [3]:  inode_D → NULL
    ...
    
Hash function:
    hash = (sb ^ ino) % hash_size
    
Bucket size:
    Depends on RAM
    Typical: 256K-1M buckets
    
Average chain length:
    1-2 inodes per bucket
    Very low collision rate
```

## iget_locked() - Main Entry Point

**Get inode from cache or disk:**

```
iget_locked(sb, ino):
    "Give me inode number N from filesystem sb"

STEP 1: Search cache
    
    hash = hash(sb, ino)
    bucket = inode_hashtable[hash]
    
    for each inode in bucket:
        if inode->i_sb == sb AND
           inode->i_ino == ino:
            
            CACHE HIT!
            
            Wait if being read:
                while inode->i_state & I_NEW:
                    wait_on_inode(inode)
                    
            return inode

STEP 2: Cache miss - allocate new
    
    inode = alloc_inode(sb):
        
        Filesystem-specific allocation:
            if sb->s_op->alloc_inode:
                inode = sb->s_op->alloc_inode(sb)
                // ext4: allocates ext4_inode_info
                // Returns embedded struct inode
            else:
                inode = kmem_cache_alloc(inode_cachep)
                
    Initialize:
        inode_init_always(sb, inode)
        inode->i_sb = sb
        inode->i_ino = ino
        
    Mark as NEW (being read):
        inode->i_state = I_NEW
        
    Add to hash:
        insert_inode_hash(inode)
        
STEP 3: Return locked inode
    
    return inode
    
    Caller MUST:
        - Read from disk
        - Fill in metadata
        - Call unlock_new_inode()

Timing:
    Cache hit: ~20ns
    Cache miss: ~100μs (includes disk read)
```

## Reading from Disk

**Filesystem fills inode:**

```
After iget_locked() returns I_NEW inode:

ext4_iget(sb, ino):
    
    inode = iget_locked(sb, ino)
    
    if !(inode->i_state & I_NEW):
        return inode  // Cache hit!
        
    // Cache miss - read from disk
    
    // Calculate disk location
    block_group = (ino - 1) / EXT4_INODES_PER_GROUP
    offset_in_group = (ino - 1) % EXT4_INODES_PER_GROUP
    
    // Read inode table block
    bh = sb_bread(sb, inode_table_block + offset/inodes_per_block)
    
    // Point to disk inode
    raw_inode = (struct ext4_inode *)(bh->b_data + offset)
    
    // Copy disk fields to memory
    inode->i_mode = le16_to_cpu(raw_inode->i_mode)
    inode->i_uid = le32_to_cpu(raw_inode->i_uid_low)
    inode->i_size = le32_to_cpu(raw_inode->i_size_lo)
    inode->i_atime.tv_sec = le32_to_cpu(raw_inode->i_atime)
    inode->i_mtime.tv_sec = le32_to_cpu(raw_inode->i_mtime)
    inode->i_ctime.tv_sec = le32_to_cpu(raw_inode->i_ctime)
    inode->i_blocks = le32_to_cpu(raw_inode->i_blocks_lo)
    inode->i_nlink = le16_to_cpu(raw_inode->i_links_count)
    
    // Copy ext4-specific fields
    ei = EXT4_I(inode)
    memcpy(ei->i_data, raw_inode->i_block, sizeof(ei->i_data))
    
    // Set operations based on type
    if S_ISREG(inode->i_mode):
        inode->i_op = &ext4_file_inode_operations
        inode->i_fop = &ext4_file_operations
        inode->i_mapping->a_ops = &ext4_aops
        
    elif S_ISDIR(inode->i_mode):
        inode->i_op = &ext4_dir_inode_operations
        inode->i_fop = &ext4_dir_operations
        
    elif S_ISLNK(inode->i_mode):
        inode->i_op = &ext4_symlink_inode_operations
        
    // Release disk buffer
    brelse(bh)
    
    // Mark ready!
    unlock_new_inode(inode)
    // Wakes any waiting threads
    // Clears I_NEW flag
    
    return inode

Timing breakdown:
    iget_locked(): ~5ns (hash lookup)
    sb_bread(): ~100μs (disk read)
    Field copying: ~50ns
    unlock_new_inode(): ~10ns
    
Total: ~100μs for cache miss
```

---

# 3. Reference Counting

## i_count - Active Users

**Tracks who's using inode:**

```
i_count = atomic_t (atomic reference count)

Who holds references:
    
    Dentry:
        dentry->d_inode holds reference
        i_count++
        
    Open file:
        file->f_inode holds reference
        i_count++
        
    Current directory:
        current->fs->pwd.dentry->d_inode
        i_count++
        
    Any kernel code using inode:
        igrab(inode)
        i_count++

Rules:
    i_count > 0: inode in use, cannot free
    i_count = 0: inode unused, can cache or evict
    
Operations:
    igrab(inode): i_count++
    iput(inode): i_count--
```

## iput() - Release Reference

**The release path:**

```
iput(inode):

Decrement and check:
    
    if atomic_dec_and_lock(&inode->i_count, &inode->i_lock):
        // Count hit 0! We have the lock!
        
        Check if keep alive:
            
            if inode->i_nlink > 0:
                // File still has directory entries
                // Keep in cache (LRU)
                
                inode->i_state |= I_REFERENCED
                list_lru_add(&inode_lru, &inode->i_lru)
                spin_unlock(&inode->i_lock)
                return
                
            if !inode_unhashed(inode):
                // Still in hash table
                // Keep cached
                
                list_lru_add(&inode_lru, &inode->i_lru)
                spin_unlock(&inode->i_lock)
                return
                
        Truly dead:
            // No directory entries AND
            // No one using it
            
            inode->i_state |= I_FREEING
            spin_unlock(&inode->i_lock)
            
            evict(inode)  // Free it!
            
    else:
        // Count still > 0
        // Someone else using it
        // Nothing to do
        
Timing:
    Common case (count>0): ~5ns
    LRU add case: ~20ns
    Evict case: ~100μs (disk write if dirty)
```

## Example: Multiple Opens

**Reference counting in action:**

```
Process A:
    fd = open("/etc/passwd", O_RDONLY)
    
    Lookup finds dentry (dcache hit)
    inode = dentry->d_inode
    igrab(inode)  // i_count = 1
    
    Create struct file
    file->f_inode = inode
    // (already counted via dentry)
    
Process B:
    fd = open("/etc/passwd", O_RDONLY)
    
    Same dentry found
    Same inode
    igrab(inode)  // i_count = 2
    
    Create new struct file
    file->f_inode = inode
    
Process A:
    close(fd)
    
    iput(file->f_inode)  // i_count = 1
    Still > 0, inode stays
    
Process B:
    close(fd)
    
    iput(file->f_inode)  // i_count = 0
    
    But i_nlink > 0 (file exists in /etc)
    Add to LRU
    Keep cached
    
Later: memory pressure
    
    Evict from LRU
    evict_inode() called
    Memory freed
    
Next open("/etc/passwd"):
    
    iget_locked() - cache miss
    Read from disk
    ~100μs instead of ~20ns
```

---

# 4. Hard Links and i_nlink

## Multiple Names, One File

**The hard link mechanism:**

```
Original file:
    ln /home/user/original newname
    
Result:
    /home/user/original → inode 123456
    /home/user/newname → inode 123456
    
Same inode number!
Both dentries point to same inode!

inode->i_nlink = 2
    Two directory entries
    Must delete both to free inode
    
inode->i_dentry list:
    [dentry_original, dentry_newname]
    Tracks all aliases

stat on either path:
    Same inode
    Same size, permissions, times
    Same data blocks
    
write via either path:
    Modifies same inode
    Change visible via both paths
```

## Deletion with Hard Links

**Progressive unlinking:**

```
Initial state:
    /home/user/file1 → inode 999
    /home/user/file2 → inode 999
    inode->i_nlink = 2

rm /home/user/file1:
    
    VFS unlink():
        Remove dentry from parent directory
        inode->i_nlink--  // Now 1
        __mark_inode_dirty(inode)
        d_delete(dentry)
        dput(dentry)
        
    Result:
        /home/user/file2 still exists
        inode still alive (i_nlink=1)
        Data intact
        
rm /home/user/file2:
    
    VFS unlink():
        Remove last dentry
        inode->i_nlink--  // Now 0!
        __mark_inode_dirty(inode)
        
    inode->i_nlink = 0:
        No directory entries left!
        
        But check i_count:
            If i_count > 0:
                Someone has it open!
                Keep inode alive
                Delete later on last close
                
            If i_count = 0:
                Immediately evict:
                    evict_inode()
                    Free disk blocks
                    Free inode
```

## Deleted While Open

**The secure temp file pattern:**

```
Process opens file:
    fd = open("/tmp/tempfile", O_RDWR|O_CREAT, 0600)
    
    inode->i_count = 1
    inode->i_nlink = 1
    
Process unlinks it:
    unlink("/tmp/tempfile")
    
    inode->i_nlink = 0  // No directory entry!
    
    But i_count = 1  // Still open!
    
State:
    File has NO path
    Cannot access via any filename
    But data still exists
    Process can still read/write fd
    
Why useful:
    Secure temp file
    No one else can open it
    No race conditions
    Automatically cleaned up on exit
    
Process continues:
    write(fd, data, len)  // Works!
    read(fd, buf, len)    // Works!
    lseek(fd, 0, SEEK_SET)  // Works!
    
Process closes:
    close(fd)
    
    iput(inode)  // i_count → 0
    
    Now both i_count=0 AND i_nlink=0
    
    evict_inode():
        ext4_evict_inode():
            ext4_truncate(inode)  // Free all blocks
            ext4_free_inode(inode)  // Free inode
            
    Data actually deleted NOW!
    Space reclaimed!
    
Timing:
    unlink(): ~10μs (remove directory entry)
    close(): ~100μs (free all blocks)
    
This is how /tmp cleanup works!
```

---

# 5. Dirty Tracking

## __mark_inode_dirty()

**Marking changes:**

```
When inode changes:
    
    chmod("/etc/file", 0755):
        inode->i_mode changed
        __mark_inode_dirty(inode, I_DIRTY_SYNC)
        
    write(fd, data, len):
        inode->i_size changed
        __mark_inode_dirty(inode, I_DIRTY_DATASYNC)
        
    chown("/etc/file", uid, gid):
        inode->i_uid, i_gid changed
        __mark_inode_dirty(inode, I_DIRTY_SYNC)
        
    touch("/etc/file"):
        inode->i_mtime changed
        __mark_inode_dirty(inode, I_DIRTY_SYNC)

Dirty flags:
    
    I_DIRTY_SYNC:
        Metadata changed (mode, uid, times)
        
    I_DIRTY_DATASYNC:
        Metadata affecting data (size)
        More urgent than SYNC
        
    I_DIRTY_PAGES:
        Page cache has dirty pages

__mark_inode_dirty(inode, flags):
    
    Already dirty with same flags?
        if (inode->i_state & flags) == flags:
            return  // Nothing to do
            
    Mark dirty:
        inode->i_state |= flags
        
    Add to writeback list:
        wb = inode_to_wb(inode)
        list_move(&inode->i_io_list, &wb->b_dirty)
        
    Wake writeback if sync mode:
        if sync_mode == WB_SYNC_ALL:
            wb_wakeup(wb)
```

## Writeback Path

**Getting dirty inodes to disk:**

```
Periodic writeback (every 30s):
    
    writeback.c runs:
        
        For each dirty inode in wb->b_dirty:
            
            if inode->i_state & I_FREEING:
                continue  // Being freed
                
            writeback_single_inode(inode):
                
                __writeback_single_inode(inode, wbc):
                    
                    Write pages:
                        if I_DIRTY_PAGES:
                            do_writepages(mapping, wbc)
                            
                    Write inode:
                        write_inode(inode, wbc):
                            sb->s_op->write_inode(inode, wbc)
                            // ext4_write_inode()
                            
                    Clear dirty flags:
                        inode->i_state &= ~I_DIRTY
                        
fsync() forces immediate write:
    
    fsync(fd):
        
        file_fsync(file):
            
            filemap_write_and_wait(inode->i_mapping)
            // Write all dirty pages
            
            sync_inode_metadata(inode, 1)
            // Write inode NOW
            
Timing:
    Periodic: happens in background
    fsync(): blocks until complete (~1-10ms)
```

---

# 6. Filesystem Embedding

## The container_of() Pattern

**Extending the base inode:**

```
Each filesystem embeds struct inode:

ext4:
    struct ext4_inode_info {
        // ext4-specific fields
        __le32 i_data[15];           // Block pointers
        __u32 i_flags;               // ext4 flags
        struct ext4_ext_cache i_cached_extent;
        // ... many more fields ...
        
        struct inode vfs_inode;      // EMBEDDED at end!
    };
    
Allocation:
    ext4_alloc_inode(sb):
        
        ei = kmem_cache_alloc(ext4_inode_cachep)
        
        return &ei->vfs_inode  // Return embedded pointer
        
Access ext4 data from VFS inode:
    
    EXT4_I(inode):
        return container_of(inode, 
                           struct ext4_inode_info,
                           vfs_inode)
        
    container_of magic:
        Given: pointer to embedded member
        Returns: pointer to containing struct
        
        Uses offsetof() to calculate:
            container_ptr = member_ptr - offsetof(type, member)
            
    Example:
        inode = &ei->vfs_inode
        ei = EXT4_I(inode)
        
        ei = (ext4_inode_info *)
             ((char *)inode - offsetof(ext4_inode_info, vfs_inode))

Other filesystems:

procfs:
    struct proc_inode {
        struct pid *pid;
        int fd;
        union proc_op op;
        struct proc_dir_entry *pde;
        struct inode vfs_inode;
    };
    
    PROC_I(inode) = container_of(...)
    
tmpfs:
    struct shmem_inode_info {
        spinlock_t lock;
        unsigned long flags;
        struct shared_policy policy;
        struct simple_xattrs xattrs;
        struct inode vfs_inode;
    };
    
    SHMEM_I(inode) = container_of(...)

Benefits:
    VFS sees uniform struct inode
    Filesystem sees rich extended struct
    Zero overhead (just pointer arithmetic)
    Maximum flexibility
    No function call overhead
```

---

# 7. Creating New Inodes

## new_inode()

**Allocating brand new inode:**

```
Creating new file:

open("/tmp/newfile", O_CREAT):
    
    VFS → ext4_create():
        
        STEP 1: Allocate inode
            
            inode = new_inode(sb):
                
                inode = alloc_inode(sb)
                // Filesystem-specific allocation
                
                Initialize:
                    inode->i_sb = sb
                    inode->i_blkbits = sb->s_blocksize_bits
                    inode->i_flags = 0
                    inode->i_count = 1
                    inode->i_nlink = 1
                    
                Generate inode number:
                    inode->i_ino = ext4_get_new_ino(sb)
                    // Allocates from inode bitmap
                    
                Timestamps:
                    inode->i_atime = current_time(inode)
                    inode->i_mtime = current_time(inode)
                    inode->i_ctime = current_time(inode)
                    
                Add to hash:
                    insert_inode_hash(inode)
                    
                Mark dirty:
                    __mark_inode_dirty(inode, I_DIRTY)
                    
                return inode
                
        STEP 2: Set type and permissions
            
            inode->i_mode = mode | S_IFREG  // Regular file
            inode->i_uid = current_fsuid()
            inode->i_gid = current_fsgid()
            
        STEP 3: Set operations
            
            inode->i_op = &ext4_file_inode_operations
            inode->i_fop = &ext4_file_operations
            inode->i_mapping->a_ops = &ext4_aops
            
        STEP 4: Add directory entry
            
            ext4_add_nondir(handle, dentry, &inode)
            // Links dentry to inode
            // Adds entry to parent directory
            
        STEP 5: Journal transaction
            
            ext4_mark_inode_dirty(handle, inode)
            // Logs creation in journal
            // Atomic commit
            
    Result:
        New inode created
        Directory entry added
        Journaled transaction
        Visible to other processes
        
Timing:
    Allocate inode: ~5μs
    Directory entry: ~10μs
    Journal commit: ~100μs
    Total: ~115μs
```

---

# 8. Complete Flow - stat("/etc/passwd")

## Initial State

**Before syscall:**

```
System warm:
    dcache populated
    /etc/passwd dentry cached
    inode already in memory
    
Inode cache:
    inode 1234567 cached
    i_count = 1 (held by dentry)
    i_nlink = 1
    In hash table
```

## Syscall Entry

```
Userspace:
    struct stat statbuf;
    stat("/etc/passwd", &statbuf)

Kernel:
    sys_newstat("/etc/passwd", &statbuf)
    → vfs_stat(&path, &statbuf, ...)
```

## Path Lookup

**Finding the inode:**

```
vfs_stat():
    
    STEP 1: Lookup path
        
        filename_lookup("/etc/passwd", ...):
            
            Path walk (namei.c):
                "/" → root_dentry (dcache hit)
                "etc" → etc_dentry (dcache hit)
                "passwd" → passwd_dentry (dcache hit)
                
            Returns: path = {passwd_dentry, ext4_mnt}
            
        Time: ~30ns (3 dcache hits)
        
    STEP 2: Get inode
        
        inode = passwd_dentry->d_inode
        
        Already in memory!
        No iget_locked() needed!
        Dentry holds reference
        
        Time: ~5ns (pointer dereference)
```

## Get Attributes

**Filling stat structure:**

```
vfs_getattr(&path, &kstat, ...):
    
    STEP 1: Permission check
        
        inode_permission(inode, MAY_READ):
            
            Check i_mode:
                mode = 0644
                owner = root
                current = regular user
                
            Other bits: r-- = 4
            MAY_READ = 4
            Allowed!
            
        Time: ~10ns
        
    STEP 2: Call filesystem getattr
        
        inode->i_op->getattr(&path, &kstat, ...):
            
            ext4_getattr():
                
                Copy from inode:
                    kstat.dev = inode->i_sb->s_dev
                    kstat.ino = inode->i_ino           // 1234567
                    kstat.mode = inode->i_mode          // 0100644
                    kstat.nlink = inode->i_nlink        // 1
                    kstat.uid = inode->i_uid            // 0 (root)
                    kstat.gid = inode->i_gid            // 0 (root)
                    kstat.size = inode->i_size          // 2837
                    kstat.atime = inode->i_atime
                    kstat.mtime = inode->i_mtime
                    kstat.ctime = inode->i_ctime
                    kstat.blksize = inode->i_sb->s_blocksize
                    kstat.blocks = inode->i_blocks
                    
                return 0
                
        Time: ~10ns (memory copies)
```

## Copy to Userspace

```
cp_new_stat(&kstat, &statbuf):
    
    Convert kernel stat to user stat:
        statbuf.st_dev = kstat.dev
        statbuf.st_ino = kstat.ino
        statbuf.st_mode = kstat.mode
        statbuf.st_nlink = kstat.nlink
        statbuf.st_uid = kstat.uid
        statbuf.st_gid = kstat.gid
        statbuf.st_size = kstat.size
        statbuf.st_atime = kstat.atime.tv_sec
        statbuf.st_mtime = kstat.mtime.tv_sec
        statbuf.st_ctime = kstat.ctime.tv_sec
        statbuf.st_blksize = kstat.blksize
        statbuf.st_blocks = kstat.blocks
        
    copy_to_user(userspace_ptr, &statbuf, sizeof(statbuf))
    
Time: ~20ns

Return to userspace:
    return 0

Total timing breakdown:
    Syscall overhead: ~50ns
    Path lookup: ~30ns
    Get inode: ~5ns
    Permission check: ~10ns
    getattr: ~10ns
    Copy to user: ~20ns
    Syscall return: ~50ns
    
    Grand total: ~175ns

All from cached inode!
Zero disk reads!
```

---

# 9. Inode LRU and Memory Pressure

## The LRU List

**Reclaiming unused inodes:**

```
inode_lru: per-superblock LRU lists

Entry conditions:
    i_count drops to 0
    i_nlink > 0 (file exists)
    Not dirty
    
    Add to LRU tail

Memory pressure:
    prune_icache_sb(sb, nr_to_scan):
        
        freed = 0
        
        list_for_each_entry_safe(inode, &sb->s_inode_lru):
            
            Check if reclaimable:
                
                if i_count > 0:
                    // Referenced again!
                    list_move(&inode->i_lru, &lru_head)
                    continue
                    
                if i_state & I_DIRTY:
                    // Dirty, must write first
                    continue
                    
                if i_state & I_REFERENCED:
                    // Referenced recently
                    inode->i_state &= ~I_REFERENCED
                    list_move(&inode->i_lru, &lru_head)
                    continue
                    
            Evict:
                list_del(&inode->i_lru)
                evict_inode(inode)
                
                freed++
                if freed >= nr_to_scan:
                    break
                    
        return freed

Aging algorithm:
    First pass: mark I_REFERENCED
    Second pass: if still I_REFERENCED, keep
    Third pass: if not referenced, evict
    
    Two-handed clock algorithm
    (same as page cache)
```

## evict_inode()

**Final cleanup:**

```
evict_inode(inode):
    
    STEP 1: Filesystem cleanup
        
        sb->s_op->evict_inode(inode):
            
            ext4_evict_inode(inode):
                
                if i_nlink == 0:
                    // File deleted
                    
                    ext4_truncate(inode)
                    // Free all data blocks
                    
                    ext4_free_inode(handle, inode)
                    // Mark inode free in bitmap
                    // Journal the deletion
                    
                else:
                    // Just evicting from cache
                    // Flush any dirty data
                    
                    truncate_inode_pages_final(&inode->i_data)
                    clear_inode(inode)
                    
    STEP 2: Remove from cache
        
        __remove_inode_hash(inode)
        // Remove from hash table
        
    STEP 3: Free memory
        
        destroy_inode(inode):
            
            sb->s_op->destroy_inode(inode):
                
                ext4_destroy_inode(inode):
                    
                    ei = EXT4_I(inode)
                    kmem_cache_free(ext4_inode_cachep, ei)
                    
Timing:
    No disk (cache evict): ~10μs
    With disk (delete): ~100μs-1ms
```

---

# 10. Physical Reality

## Memory Usage

**Inode cache sizing:**

```
Per-inode memory:
    Base struct inode: ~600 bytes
    ext4_inode_info: ~800 bytes total
    procfs: ~400 bytes total
    tmpfs: ~700 bytes total

Typical system (16GB RAM):
    Cached inodes: 200,000-500,000
    Memory: 160-400 MB
    
    2.5% of RAM for inode cache
    
Check current:
    /proc/sys/fs/inode-state:
        nr_inodes: 234567
        nr_unused: 123456
        preshrink: 0
```

## Disk vs Memory Size

**Storage efficiency:**

```
ext4 disk inode: 256 bytes (or 128 bytes)
    
    i_mode: 2 bytes
    i_uid: 2 bytes
    i_size: 4 bytes
    i_atime: 4 bytes
    i_blocks: 4 bytes
    i_block[15]: 60 bytes
    ... (rest of 256 bytes)

Memory inode: ~800 bytes
    
    3× larger!
    
Why:
    Filesystem-specific state
    Page cache linkage
    Writeback tracking
    Lock structures
    Operations pointers
    Reference counting
    Hash/LRU linkage
    
Trade-off:
    3× memory cost
    Eliminates 99% of disk reads
    Worth it!
```

## Cache Hit Rates

**Real-world performance:**

```
Busy server metrics:
    
    Inode lookups: 1,000,000/sec
    Cache hits: 980,000 (98%)
    Cache misses: 20,000 (2%)
    
Those 20,000 misses:
    20,000 × 100μs = 2 seconds of I/O
    
Those 980,000 hits:
    980,000 × 20ns = 0.02 seconds
    
Without cache:
    1,000,000 × 100μs = 100 seconds
    50× slower!
```

---

# 11. Connections to Other Topics

## Connection to dcache.c

**Paired caches:**

```
Dentry-inode relationship:
    dentry->d_inode = inode
    inode->i_dentry = list of dentries
    
Reference counting:
    dentry holds inode reference
    dput() → iput()
    
Lookup flow:
    dcache hit → inode already there
    dcache miss → may still have inode cached
    Both miss → disk read
    
Hard links:
    Multiple dentries, one inode
    inode->i_dentry tracks all
```

## Connection to namei.c

**Path walk uses inodes:**

```
namei.c finds dentry:
    dentry→d_inode is the result
    
Permission checks:
    inode_permission(inode, mask)
    Reads i_mode, i_uid, i_gid
    
Type checks:
    S_ISDIR(inode->i_mode)
    S_ISLNK(inode->i_mode)
    Determines next step
    
Operations:
    inode->i_op->lookup() for directories
    inode->i_op->get_link() for symlinks
```

## Connection to file.c

**Open file uses inode:**

```
struct file:
    file->f_inode = inode
    file->f_op = inode->i_fop
    
Reference:
    open() increments i_count
    close() decrements via iput()
    
Operations:
    read(fd) → file->f_op->read()
    write(fd) → file->f_op->write()
    Both set by inode type
```

## Connection to ext4

**Filesystem integration:**

```
ext4 operations:
    ext4_iget(): read disk inode
    ext4_write_inode(): write to disk
    ext4_evict_inode(): free blocks
    
container_of():
    VFS inode → ext4_inode_info
    EXT4_I(inode) everywhere
    
Disk format:
    256-byte ext4_inode on disk
    800-byte ext4_inode_info in memory
    Conversion in ext4_iget()
```

## Connection to writeback.c

**Dirty inode handling:**

```
__mark_inode_dirty():
    Add to wb->b_dirty list
    
writeback.c:
    Periodic scan of dirty list
    Calls write_inode()
    → sb->s_op->write_inode()
    → ext4_write_inode()
    
fsync():
    sync_inode_metadata()
    Forces immediate write
```

## Connection to page cache

**Address space linkage:**

```
inode->i_mapping:
    Points to address_space
    Contains all file pages
    
read/write operations:
    Check i_mapping for cached pages
    Cache miss: read via i_aop->readpage()
    Cache hit: instant access
    
This is how page cache connects to files!
```

---

# Summary

## What You've Mastered

**inode.c is the real file:**

```
What it does:
    Manages in-memory inodes
    Caches file metadata
    Tracks references
    Handles deletion
    Provides filesystem abstraction

Key mechanisms:
    Inode cache (hash table)
    Reference counting (i_count, i_nlink)
    LRU reclaim (memory management)
    Dirty tracking (writeback integration)
    Filesystem embedding (container_of)
```

## Key Takeaways

**The fundamental insight:**

```
Filename ≠ File!

Filename is dentry (label)
Inode is the file (data + metadata)

One inode, many names:
    Hard links work naturally
    
Zero names, inode still alive:
    While open (i_count > 0)
    Delete on close works
    
This is UNIX philosophy:
    Everything is a file (inode)
    Names are just pointers
```

**Inode cache:**

```
Global hash table
Key: (superblock, inode_number)
O(1) lookup
98% hit rate typical

iget_locked():
    Cache hit: ~20ns
    Cache miss: ~100μs (disk read)
    
5000× speed difference!
```

**Reference counting:**

```
i_count: active users
    > 0: in use
    = 0: cache or evict
    
i_nlink: directory entries
    > 0: file exists
    = 0: file deleted
    
Both must be 0 to free!
```

**Hard links:**

```
ln creates second name
Both point to same inode
i_nlink increments

Delete one name:
    i_nlink decrements
    Inode survives
    
Delete all names:
    i_nlink = 0
    If i_count = 0: evict now
    If i_count > 0: evict on close
```

**Deleted while open:**

```
unlink() sets i_nlink = 0
But i_count > 0 (open file)

File has no path:
    Cannot open new fd
    Existing fd works fine
    
close() sets i_count = 0:
    Now both zero
    evict_inode()
    Free disk blocks
    
Secure temp file pattern!
```

**Filesystem embedding:**

```
struct ext4_inode_info:
    ... ext4 fields ...
    struct inode vfs_inode;
    
alloc returns &ei->vfs_inode
container_of() gets full struct

VFS: uniform interface
Filesystem: rich data
Zero overhead!
```

**Physical reality:**

```
~800 bytes per cached inode
200K-500K inodes typical
160-400 MB memory
98% cache hit rate
Worth every byte!
```

**Connections:**

```
dcache: dentry→inode pairing
namei: path walk result
file.c: open file inode reference
ext4: disk inode I/O
writeback: dirty inode flushing
page cache: file data caching
```

---

## You're Ready!

With this knowledge, you can:
- Understand what files really are
- Debug hard link issues
- Know how deletion works
- See filesystem abstraction layer
- Understand reference counting
- Know why cache matters so much

> **The inode is the file - everything else is just metadata about how to find and use it. You now understand the core abstraction that makes UNIX filesystems work.**

---

**The file beneath the name.**
