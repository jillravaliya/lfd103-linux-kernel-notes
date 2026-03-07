# From Path String to Inode - Path Lookup

> Ever wondered how the kernel finds your file when you type open("/etc/passwd")? How symlinks get followed? How permission checking happens at every step? How the kernel crosses mount points seamlessly?

**You're about to find out!**

---

## What's This About?

This is **fs/namei.c** - the path resolution engine that translates string paths into actual inodes!

Here's where it fits in the VFS ecosystem:

```
fs/
├── vfs-core/
│   ├── namei.c             ← Path lookup (THIS!)
│   │   ├── Path walking    ← Component by component
│   │   ├── dcache lookup   ← Fast path (cached)
│   │   ├── Symlink follow  ← Recursive resolution
│   │   ├── Mount crossing  ← Filesystem boundaries
│   │   └── Permission check← Every component verified
│   ├── dcache.c            ← Directory entry cache
│   ├── inode.c             ← File metadata
│   ├── file.c              ← Open file state
│   └── file_table.c        ← File descriptor table
├── io/                     ← Read/write operations
└── ext4/                   ← Real filesystem
```

**Location in code:**

```
fs/
├── namei.c                 ← Path resolution (5500+ lines)
│   ├── path_lookupat()     ← Main lookup engine
│   ├── link_path_walk()    ← Component walker
│   ├── walk_component()    ← Single component lookup
│   ├── do_filp_open()      ← open() entry point
│   ├── follow_mount()      ← Mount point crossing
│   └── follow_link()       ← Symlink resolution

Entry points:
└── Every file operation starts here
    open(), stat(), unlink(), rename(), chmod()
```

**This covers:**
- String path to inode translation
- RCU lockless lookups (fast path)
- Directory entry caching
- Symlink resolution (nested and chained)
- Mount point detection and crossing
- Permission verification at every step
- Complete open() flow from syscall to fd

---

# The Fundamental Problem

**Physical Reality:**

```
Userspace sees paths as strings:
├── "/etc/passwd"
├── "/home/user/.bashrc"
├── "../config/settings.json"
├── "relative/path/file.txt"
└── All just character arrays!

Kernel operates on inodes:
├── inode 12345 on device 8:1
├── inode 67890 on device 8:2
└── Numeric identifiers with metadata

Someone must bridge this gap!
```

**The complexity:**

```
Simple case:
    /etc/passwd
    └── Just walk: / → etc → passwd
        Easy!

Complex cases:
    /etc/alternatives/python → /usr/bin/python3.10
    └── Symlink! Must follow chain!
    
    /proc/self/exe
    └── Crosses mount point from ext4 to procfs!
    
    ../../etc/passwd
    └── Relative path with .. components!
    
    /home/user/../user/./file
    └── . and .. handling needed!
    
    /very/deep/symlink/chain/loops/back
    └── Must detect infinite loops! (ELOOP)

All handled by namei.c!
```

---

# Without Path Resolution

**Imagine no namei.c:**

```
Every filesystem reimplements lookup:
├── ext4: Walk directories manually
├── btrfs: Walk B-trees manually
├── xfs: Walk B+trees manually
└── No code reuse!

No caching:
├── Every open() reads disk
├── /lib/x86_64-linux-gnu/libc.so.6 accessed 1000x/sec
├── Each access = disk read
└── System crawls!

No security:
├── Skip intermediate directory checks?
├── Open /etc/shadow even if /etc not readable?
└── Security holes everywhere!

No symlink support:
├── Applications must detect and follow
├── Each app implements differently
└── Inconsistent behavior!
```

**Complete chaos without unified path resolution!**

---

# The Layered Solution

**namei.c provides:**

```
Layer 1: STRING PARSING
Input: "/etc/passwd"
└── Split into components: ["", "etc", "passwd"]
    Handle / separator
    Identify relative vs absolute

Layer 2: COMPONENT LOOKUP
For each component:
└── Hash the name
    Check dcache (fast!)
    If miss: ask filesystem (slow)
    Return dentry + inode

Layer 3: SPECIAL HANDLING
Detect and handle:
└── . (current directory)
    .. (parent directory)
    Symlinks (follow recursively)
    Mount points (cross filesystems)

Layer 4: PERMISSION CHECKING
At every step:
└── Check execute permission on directories
    Check read/write/exec on final file
    Call LSM hooks (SELinux, AppArmor)

Layer 5: STATE MANAGEMENT
Track:
└── Current position (struct path)
    Symlink nesting depth
    Lookup flags (follow links? create? exclusive?)
    RCU sequence numbers

Result:
└── String → inode translation
    99%+ cache hit rate
    100-200ns typical lookup time
    Millions of lookups per second
```

---

# 1. Key Data Structures

## struct nameidata

**The lookup state machine:**

```c
struct nameidata {
    // Current position
    struct path path;           // Where we are now
    struct qstr last;           // Last component name
    struct inode *inode;        // Current inode
    
    // Starting point
    struct path root;           // Root directory (/ or chroot)
    
    // Configuration
    unsigned int flags;         // LOOKUP_FOLLOW, LOOKUP_RCU, etc.
    unsigned int seq;           // RCU sequence number
    int last_type;              // LAST_NORM, LAST_ROOT, LAST_DOT
    
    // Symlink tracking
    unsigned depth;             // Nesting level (max 40)
    
    struct saved {
        struct path link;
        const char *name;
        unsigned seq;
    } *stack, internal[EMBEDDED_LEVELS];
};

Usage flow:
    Initialize nameidata
    Walk components one by one
    Update path as we progress
    Return final inode
```

## struct path

**Position in filesystem tree:**

```c
struct path {
    struct vfsmount *mnt;       // Which mount
    struct dentry *dentry;      // Which dentry
};

Why both needed:
    Same dentry can be mounted multiple times!
    
    /proc mounted at:
        /proc     → path{mnt1, proc_root}
        /mnt/proc → path{mnt2, proc_root}
        
    Different paths, same dentry!
    Must track mount separately!

Example:
    / → {ext4_mnt, root_dentry}
    /proc → {procfs_mnt, proc_root}
    /home → {ext4_mnt, home_dentry}
```

## struct qstr

**Qualified string (with hash):**

```c
struct qstr {
    union {
        struct {
            u32 hash;           // Precomputed hash
            u32 len;            // Length
        };
        u64 hash_len;           // Atomic compare
    };
    const unsigned char *name;  // The string
};

Why precompute hash:
    dcache lookup is hash table
    Hash computed once during walk
    Every dcache lookup uses it
    Saves repeated hashing!

Example:
    "passwd" in /etc/passwd
    → hash = hash_name("passwd", 6)
    → len = 6
    → name = "passwd"
    
    dcache_lookup(parent, qstr)
    Uses qstr->hash directly!
```

## Lookup Flags

**Control how lookup behaves:**

```
LOOKUP_FOLLOW:
    Follow symlinks (default for open)
    Without: return symlink itself (for lstat)

LOOKUP_DIRECTORY:
    Result must be directory
    Used by: opendir(), mkdir()

LOOKUP_AUTOMOUNT:
    Trigger automount if needed
    NFS, autofs use this

LOOKUP_RCU:
    Try lockless mode first
    Fall back to locked on conflict
    Default for all lookups!

LOOKUP_OPEN:
    We're opening file
    Allows filesystem optimizations

LOOKUP_CREATE:
    We're creating file
    May need to allocate inode

LOOKUP_EXCL:
    Fail if exists (O_EXCL)
    Used with LOOKUP_CREATE
```

---

# 2. RCU Lookup - The Fast Path

## The Scalability Problem

**Without RCU:**

```
Path lookup needs to read:
    Directory dentries
    Directory inodes
    Mount table
    
Traditional locking:
    Lock directory
    Read dentry
    Unlock directory
    
    Repeat for each component!

/usr/lib/x86_64-linux-gnu/libc.so.6
    = 5 components
    = 5 lock/unlock pairs
    
On busy system:
    Thousands of processes
    All doing lookups simultaneously
    Lock contention on every directory
    
Result: Severe bottleneck!
```

## RCU Solution

**Lockless reading:**

```
Read without locks!
    Use sequence numbers to detect changes
    If data changed: retry
    If stable: use it
    
Sequence counter pattern:
    Before read:
        seq = read_seqcount_begin(&dentry->d_seq)
        
    Read dentry data
    
    After read:
        if read_seqcount_retry(&dentry->d_seq, seq):
            Data changed! Retry or fall back
        else:
            Data was stable! Safe to use

Common case (no writers):
    Zero locks needed!
    Just memory reads!
    
Rare case (concurrent modification):
    Detect via sequence number
    Fall back to locked mode
```

## RCU Walk Flow

**LOOKUP_RCU mode:**

```
path_lookupat() with LOOKUP_RCU:

For each component:
    seq = read_seqbegin(&parent->d_seq)
    
    dentry = __d_lookup_rcu(parent, name, &seq)
    
    if (!dentry):
        Cache miss!
        Must take locks
        Return -ECHILD → retry locked
        
    Read dentry->d_inode (no lock!)
    
    Verify seq still valid:
        if read_seqretry(&dentry->d_seq, seq):
            Concurrent modification!
            Return -ECHILD → retry locked
            
    Continue to next component
    
Success path (all RCU):
    No locks taken
    ~100-200ns total time
    
Fallback path:
    -ECHILD returned
    Caller retries with locks
    ~500ns-1μs total time
    
Still fast! But 3-5× slower than RCU
```

## Performance Impact

**Real-world speedup:**

```
Before RCU lookup (2.6.38):
    Path lookup: ~500ns average
    Lock contention visible in profiles
    
After RCU lookup (2.6.39+):
    Path lookup: ~100-200ns average
    3-5× faster!
    No lock contention
    
Scalability:
    Before: Scales poorly beyond 8 cores
    After: Scales linearly to 100+ cores
    
This is why modern Linux handles:
    Millions of file operations per second
    On high core-count systems
```

---

# 3. Link Path Walk - Component by Component

## Main Algorithm

**link_path_walk() flow:**

```
Input: path string "etc/passwd"
       nameidata initialized to starting point

STEP 1: Handle leading slash (if present)
    Already handled by caller
    nd->path = root or cwd

STEP 2: Loop over components
    
    While path has more components:
        
        Extract next component:
            Find next '/' separator
            component = substring before '/'
            
        Compute hash:
            hash = hash_name(component, len)
            
        Look up component:
            walk_component(nd, component, hash)
            
        Check if last component:
            If not: must be directory
            If yes: save for caller to handle
            
STEP 3: Return
    nd->path points to parent of last component
    nd->last contains last component name
    Caller handles final step (open/create/delete)
```

## walk_component()

**Single component lookup:**

```
walk_component(nd, name):

Handle special names:
    if name == ".":
        Stay in current directory
        return 0
        
    if name == "..":
        Go to parent directory
        Check mount boundaries!
        follow_dotdot(nd)
        return 0

Try RCU dcache lookup:
    dentry = __d_lookup_rcu(parent, name, &seq)
    
    if found:
        Cache HIT!
        nd->path.dentry = dentry
        nd->inode = dentry->d_inode
        
        Check if mount point:
            if (d_mountpoint(dentry)):
                follow_mount(nd)
                
        return 0

Cache MISS - need filesystem lookup:
    Drop RCU mode (take locks)
    
    lookup_slow(nd, name):
        Lock parent inode
        
        Call filesystem lookup:
            dentry = parent->i_op->lookup(
                        parent_inode, dentry, flags)
        
        For ext4:
            ext4_lookup():
                Read directory blocks
                Search for name
                Return inode number
                iget() to get inode
                
        Add to dcache:
            d_add(dentry, inode)
            
        Unlock parent
        
    return dentry

Check if symlink:
    if S_ISLNK(inode->i_mode):
        if LOOKUP_FOLLOW:
            follow_link(nd)
```

## Directory vs File Handling

**Intermediate components:**

```
Walking /etc/passwd:
    
    Component "etc":
        Must be directory!
        
        Check:
            if !S_ISDIR(inode->i_mode):
                return -ENOTDIR
                
        Permission check:
            inode_permission(inode, MAY_EXEC)
            Need execute bit to enter!
            
        Continue walk

Last component "passwd":
    Don't check type yet
    Caller will verify (open wants file, mkdir wants non-exist)
```

---

# 4. Symlink Resolution

## The Symlink Problem

**Recursive path lookups:**

```
/etc/alternatives/python → /usr/bin/python3.10

Walking /etc/alternatives/python:
    
    Walk to /etc/alternatives/
    Look up "python"
    Found dentry
    
    Check inode type:
        S_ISLNK! It's a symlink!
        
    Read target:
        inode->i_op->get_link()
        Returns: "/usr/bin/python3.10"
        
    Now what?
        Must walk new path!
        But remember where we were!
        
    Solution:
        Save current state
        Start new walk with target
        When done: restore or continue
```

## Symlink Stack

**Iterative (not recursive):**

```
Old approach (recursive):
    follow_link() calls path_walk()
    path_walk() finds symlink
    follow_link() calls path_walk()
    ...
    
    Problem: Stack overflow on deep chains!

Modern approach (iterative):
    Explicit stack in nameidata
    
    nd->stack[] array
    nd->depth counter
    
    On symlink:
        Push current state to stack
        nd->depth++
        Start walking target
        
    On completion:
        nd->depth--
        Pop from stack
        Continue
        
    No recursion! No stack overflow!

Maximum depth: 40 symlinks
    if (nd->depth > 40):
        return -ELOOP
        
    Prevents infinite loops:
        a → b → c → a (cycle!)
```

## Symlink Types

**Absolute vs relative:**

```
Absolute symlink:
    Target: /usr/bin/python3.10
    
    Restart from root:
        nd->path = nd->root
        Walk "/usr/bin/python3.10"

Relative symlink:
    Target: ../lib/libpython.so
    
    Continue from symlink's directory:
        nd->path = symlink's parent
        Walk "../lib/libpython.so"

Nested symlinks:
    a → b → c → /final/path
    
    Depth 0: walking to a
    Depth 1: a is symlink, walk to b
    Depth 2: b is symlink, walk to c
    Depth 3: c is symlink, walk to /final/path
    Depth 2: back to c
    Depth 1: back to b
    Depth 0: back to a, use final result
```

---

# 5. Mount Point Crossing

## Mount Detection

**Every component checked:**

```
After each walk_component():
    
    Check if dentry is mount point:
        d_mountpoint(dentry)
        
    Implementation:
        dentry->d_flags & DCACHE_MOUNTED
        
    If set:
        follow_mount(nd)
```

## follow_mount()

**Crossing to child filesystem:**

```
follow_mount(nd):

    Current: nd->path = {parent_mnt, mountpoint_dentry}
    
    Find mounted filesystem:
        mnt = lookup_mnt(nd->path)
        
    If found:
        Switch to child:
            nd->path.mnt = mnt
            nd->path.dentry = mnt->mnt_root
            
    Example:
        Walking /proc/self
        
        At /proc:
            {ext4_mnt, proc_dentry}
            d_mountpoint(proc_dentry) = true
            
        lookup_mnt():
            Returns procfs_mnt
            
        After follow_mount:
            {procfs_mnt, proc_root_dentry}
            
        Now in procfs!
        Continue walking "self"
```

## Upward Traversal

**Going past mount root:**

```
cd /proc; cd ..

At /proc:
    nd->path = {procfs_mnt, proc_root}
    
Component "..":
    follow_dotdot(nd):
    
        Check if at mount root:
            if (nd->path.dentry == nd->path.mnt->mnt_root):
                At mount root!
                
                Find parent mount:
                    parent_path = nd->path.mnt->mnt_mountpoint
                    
                Switch to parent:
                    nd->path = parent_path
                    
                Now: {ext4_mnt, proc_dentry}
                
        Not at mount root:
            Normal ..:
                nd->path.dentry = nd->path.dentry->d_parent
```

## Stacked Mounts

**Multiple mounts on same point:**

```
mount /dev/sda1 /mnt
mount /dev/sda2 /mnt    # Stack on top!

Walking /mnt:
    lookup_mnt() returns topmost mount
    Most recent mount wins
    
    This enables:
        Bind mounts
        Overlayfs
        Containers
        
Unmount:
    Remove top mount
    Reveals previous mount underneath
```

---

# 6. Permission Checking

## inode_permission()

**Every component verified:**

```
inode_permission(inode, mask):

mask can be:
    MAY_EXEC (1)  - Execute/search permission
    MAY_WRITE (2) - Write permission
    MAY_READ (4)  - Read permission

Fast paths:
    
    If kernel thread:
        return 0  // Kernel can do anything
        
    If root (uid == 0) and not exec on non-exec file:
        return 0  // Root bypasses most checks
        
Main check:
    generic_permission(inode, mask)
```

## generic_permission()

**Unix permission bits:**

```
mode = inode->i_mode  // e.g., 0644

Extract permission bits:
    Owner: (mode >> 6) & 7  // rwx------
    Group: (mode >> 3) & 7  // ---rwx---
    Other: mode & 7         // ------rwx

Determine which to use:
    
    if current_fsuid() == inode->i_uid:
        Use owner bits
        
    elif in_group_p(inode->i_gid):
        Use group bits
        
    else:
        Use other bits

Check access:
    
    if MAY_READ requested:
        Need read bit (4) set
        
    if MAY_WRITE requested:
        Need write bit (2) set
        
    if MAY_EXEC requested:
        Need exec bit (1) set
        
    All requested must be granted:
        if (mode & mask) == mask:
            return 0  // Allowed
        else:
            return -EACCES  // Denied
```

## LSM Hooks

**Mandatory access control:**

```
After DAC check:
    security_inode_permission(inode, mask)

Calls all loaded LSM modules:
    
    SELinux:
        Check security context
        type enforcement rules
        
    AppArmor:
        Check profile for this path
        allowed operations
        
    Other LSMs:
        Smack, TOMOYO, etc.

Any LSM can deny:
    Even if DAC allows!
    
Example:
    File mode: 0644 (readable by all)
    DAC: Allowed
    
    SELinux:
        Process context: user_t
        File context: shadow_t
        Policy: user_t cannot read shadow_t
        Denied!
        
    Final result: -EACCES
```

---

# 7. do_filp_open() - The Entry Point

## Syscall to namei.c

**open() flow:**

```
Userspace:
    fd = open("/etc/passwd", O_RDONLY)

Syscall:
    sys_openat(AT_FDCWD, "/etc/passwd", O_RDONLY, 0)
    
    → do_sys_open()
        → do_filp_open()  [fs/namei.c]

do_filp_open(dfd, filename, open_flags):
    
    Build open operation:
        op.open_flag = open_flags
        op.lookup_flags = LOOKUP_FOLLOW
        
        if O_CREAT:
            op.lookup_flags |= LOOKUP_CREATE
            
        if O_DIRECTORY:
            op.lookup_flags |= LOOKUP_DIRECTORY

Try RCU mode first:
retry:
    file = path_openat(&nd, op, flags | LOOKUP_RCU)
    
    if file == ERR_PTR(-ECHILD):
        RCU failed, retry locked:
            file = path_openat(&nd, op, flags)
            
    if file == ERR_PTR(-ESTALE):
        Stale file handle, retry:
            file = path_openat(&nd, op, flags | LOOKUP_REVAL)
            
    return file
```

## path_openat()

**Main lookup driver:**

```
path_openat(nd, op, flags):

Initialize nameidata:
    nd->flags = flags
    nd->depth = 0
    
Set starting point:
    
    if absolute path (starts with /):
        nd->path = current->fs->root
        
    elif AT_FDCWD:
        nd->path = current->fs->pwd
        
    else:
        nd->path = fd_to_path(dfd)

Walk to last component:
    
    s = path_init(nd, flags)
    
    while (!(nd->flags & LOOKUP_JUMPED)):
        link_path_walk(s, nd)
        
Handle last component:
    
    do_last(nd, file, op)

Return:
    return file
```

## do_last()

**Final component handling:**

```
do_last(nd, file, op):

Lookup last component:
    
    lookup_fast(nd):
        Try RCU lookup
        
    if cache miss:
        lookup_open(nd, file, op):
            
            If O_CREAT and doesn't exist:
                vfs_create()
                
            If exists and O_EXCL:
                return -EEXIST
                
            lookup_real(dir, dentry, nd->flags)

Check type:
    
    if S_ISLNK and LOOKUP_FOLLOW:
        trailing_symlink(nd)
        
    if S_ISDIR and not LOOKUP_DIRECTORY:
        if open for write:
            return -EISDIR

Open the file:
    
    vfs_open(nd->path, file):
        
        file->f_path = nd->path
        file->f_inode = inode
        file->f_op = inode->i_fop
        
        Call filesystem open:
            file->f_op->open(inode, file)
            
        For ext4:
            ext4_file_open()

return 0
```

---

# 8. Complete Flow - open("/etc/passwd", O_RDONLY)

## Initial State

**Before syscall:**

```
Process state:
    PID: 1234
    Current directory: /home/user
    Root directory: /
    
Filesystem state:
    / mounted: ext4 on /dev/sda1
    /proc mounted: procfs
    
dcache state:
    Partially populated:
        / → cached
        /etc → probably cached
        /etc/passwd → probably cached
        
File descriptor table:
    0: stdin
    1: stdout
    2: stderr
    3-1023: unused
```

## Syscall Entry

**System call transition:**

```
Userspace:
    int fd = open("/etc/passwd", O_RDONLY);

Assembly:
    mov rax, 2              ; SYS_open
    mov rdi, path_ptr       ; "/etc/passwd"
    mov rsi, 0              ; O_RDONLY
    syscall

Kernel entry:
    do_syscall_64()
    → sys_openat(AT_FDCWD, "/etc/passwd", O_RDONLY, 0)
    → do_sys_open()
    → do_filp_open()
```

## Path Lookup Initialization

**path_openat() setup:**

```
Initialize nameidata:
    
    nd.flags = LOOKUP_FOLLOW | LOOKUP_RCU
    nd.depth = 0
    nd.seq = 0
    
Set starting point (absolute path):
    
    nd.path = current->fs->root
    
    // root = {ext4_vfsmount, root_dentry}
    nd.path.mnt = ext4_vfsmount
    nd.path.dentry = root_dentry
    nd.inode = root_dentry->d_inode

Path string:
    "/etc/passwd"
    
    Skip leading '/':
        path = "etc/passwd"
```

## Component Walk - "etc"

**First component lookup:**

```
link_path_walk("etc/passwd", nd):

Extract component:
    Find next '/': "etc" + "/" + "passwd"
    component = "etc"
    len = 3

Compute hash:
    hash = hash_name("etc", 3)
    → hash = 0xABCD1234 (example)

walk_component(nd, "etc", hash):
    
    RCU dcache lookup:
        seq = read_seqbegin(&root_dentry->d_seq)
        
        dentry = __d_lookup_rcu(root_dentry, "etc", hash)
        
        Search hash table:
            bucket = hash & (DCACHE_SIZE - 1)
            
            For each dentry in bucket:
                if (dentry->d_parent == root_dentry &&
                    dentry->d_name.hash == hash &&
                    dentry->d_name.len == 3 &&
                    memcmp(dentry->d_name.name, "etc", 3) == 0):
                    
                    CACHE HIT!
                    found = dentry
                    break
                    
        Result: etc_dentry found!
        
    Verify sequence:
        if read_seqretry(&root_dentry->d_seq, seq):
            Concurrent modification!
            Fall back to locked mode
        else:
            Stable! Continue
            
    Update state:
        nd->path.dentry = etc_dentry
        nd->inode = etc_dentry->d_inode

Not last component - verify directory:
    
    if !S_ISDIR(nd->inode->i_mode):
        return -ENOTDIR
        
    Permission check:
        inode_permission(nd->inode, MAY_EXEC)
        
        Check execute bit:
            mode = 0755 (drwxr-xr-x)
            other bits = 5 (r-x)
            MAY_EXEC = 1
            5 & 1 = 1 → Allowed!

Continue to next component
```

## Component Walk - "passwd"

**Last component lookup:**

```
Remaining path: "passwd"
No more '/' → last component!

Don't walk yet:
    nd->last.name = "passwd"
    nd->last.len = 6
    nd->last.hash = hash_name("passwd", 6)
    nd->last_type = LAST_NORM

Return from link_path_walk:
    nd->path = /etc
    nd->last = "passwd"
    
Caller (do_last) handles final component
```

## Final Component Handling

**do_last() processing:**

```
do_last(nd, file, op):

Lookup "passwd" in /etc:
    
    RCU dcache lookup:
        dentry = __d_lookup_rcu(etc_dentry, "passwd", hash)
        
        CACHE HIT!
        passwd_dentry found
        
    Update:
        nd->path.dentry = passwd_dentry
        nd->inode = passwd_dentry->d_inode

Check if symlink:
    
    if S_ISLNK(nd->inode->i_mode):
        Not a symlink, continue
        
Check permissions:
    
    inode_permission(nd->inode, MAY_READ)
    
    File mode: 0644 (rw-r--r--)
    We are: regular user (not owner, not group)
    Other bits: 4 (r--)
    MAY_READ = 4
    4 & 4 = 4 → Allowed!

Open file:
    
    vfs_open(&nd->path, file):
        
        file->f_path = nd->path
        file->f_inode = passwd_inode
        file->f_mapping = passwd_inode->i_mapping
        file->f_op = &ext4_file_operations
        
        Call filesystem open:
            ext4_file_open(passwd_inode, file)
            
            Returns 0 (success)

return file
```

## File Descriptor Installation

**Back in do_sys_open():**

```
Allocate file descriptor:
    
    fd = get_unused_fd_flags(O_RDONLY)
    
    Search fd table:
        0: used (stdin)
        1: used (stdout)
        2: used (stderr)
        3: free! ← Use this
        
    fd = 3

Install file:
    
    fd_install(fd, file)
    
    current->files->fdt->fd[3] = file
    file->f_count++ (reference count)

Return to userspace:
    
    return 3

Userspace:
    
    open() returns 3
    
    Now can:
        read(3, buffer, size)
        close(3)
```

## Timing Breakdown

**Performance analysis:**

```
Total time (cache hot):
    200-500 nanoseconds

Breakdown:
    Syscall overhead: ~50ns
    nameidata init: ~20ns
    Component "etc":
        RCU dcache lookup: ~50ns
        Permission check: ~20ns
    Component "passwd":
        RCU dcache lookup: ~50ns
        Permission check: ~20ns
    vfs_open: ~30ns
    fd_install: ~20ns
    Syscall return: ~50ns

Cache cold (disk reads needed):
    100 microseconds - 10 milliseconds
    
    Component miss:
        ext4_lookup(): read directory block
        Disk I/O: 100μs (SSD) - 10ms (HDD)
```

---

# 9. Physical Reality

## dcache Hit Rates

**Real-world caching:**

```
Production Linux server:

dcache capacity:
    Millions of dentries
    Limited by memory
    Typical: 10-50% of RAM for dcache

Common paths (always hot):
    /lib/x86_64-linux-gnu/libc.so.6
    /usr/bin/bash
    /etc/passwd
    /proc/self/
    
    Accessed: thousands of times per second
    Always in cache
    ~100ns lookup time

Rare paths (cache miss):
    First access after boot
    Infrequently accessed files
    Random file access
    
    Requires disk read
    ~100μs-10ms
    Then cached for subsequent access

Hit rate on typical system:
    99%+ for hot paths
    90%+ overall
    
    Without dcache:
    Every lookup = disk read
    System unusable!
```

## RCU Effectiveness

**Lock-free scalability:**

```
Single-core system:
    RCU benefit minimal
    Already no contention
    
Multi-core system (8+ cores):
    
    Without RCU:
        Every lookup takes locks
        Lock contention visible
        Scales poorly
        
    With RCU:
        Common case: zero locks
        No contention
        Linear scaling to 100+ cores
        
Measurement:
    Path lookups per second:
        4 cores: 5M/sec
        8 cores: 10M/sec
        16 cores: 20M/sec
        32 cores: 40M/sec
        
    Linear! RCU makes this possible
```

## Path Length Impact

**Component count matters:**

```
/a
    1 component
    1 dcache lookup
    ~100ns

/etc/passwd
    2 components
    2 dcache lookups
    ~150ns

/usr/lib/x86_64-linux-gnu/libc.so.6
    5 components
    5 dcache lookups
    ~250ns

/very/deep/directory/structure/with/many/components/file.txt
    9 components
    9 dcache lookups
    ~400ns

Each component:
    ~50ns dcache lookup (RCU)
    ~20ns permission check
    ~70ns total per component
    
Shallow paths preferred!
```

---

# 10. Connections to Other Topics

## Connection to dcache.c

**Heavily dependent:**

```
Every walk_component() call:
    __d_lookup_rcu() from dcache.c
    
Cache hit (99%):
    Instant return
    No filesystem involvement
    
Cache miss:
    lookup_slow() asks filesystem
    d_add() adds to cache
    
namei.c success depends on dcache efficiency!
Next topic: dcache.c internals
```

## Connection to inode.c

**Final result:**

```
After path walk:
    nd->inode = target inode
    
Inode contains:
    File metadata (size, permissions)
    Block map (where data lives)
    Operations (read, write, etc.)
    
open() → namei.c finds inode → file.c uses it
```

## Connection to ext4/namei.c

**Cache miss handling:**

```
On dcache miss:
    parent->i_op->lookup() called
    
For ext4:
    ext4_lookup(parent_inode, dentry, flags)
    
    Read directory blocks
    Search HTree (hash tree)
    Find entry
    Return inode number
    iget() creates/finds inode
    
VFS namei.c → calls → ext4 namei.c
Different layers, same names!
```

## Connection to mount (super.c)

**Filesystem boundaries:**

```
follow_mount() uses:
    lookup_mnt(path)
    
Searches mount table:
    For each mount:
        if (mount->mnt_mountpoint == path->dentry):
            Found child mount!
            
Returns vfsmount for child filesystem

Mount propagation:
    Shared mounts
    Slave mounts
    Bind mounts
    All handled by mount code
    namei.c just follows results
```

## Connection to security (LSM)

**Every component checked:**

```
inode_permission() calls:
    security_inode_permission()
    
Registered LSMs:
    SELinux: selinux_inode_permission()
    AppArmor: apparmor_inode_permission()
    
Example (SELinux):
    
    Check process security context
    Check file security context
    Query policy:
        allow user_t passwd_file_t:file { read }
        
    If allowed: return 0
    If denied: return -EACCES
    
Mandatory access control at every step!
```

## Connection to process (task_struct)

**Per-process context:**

```
current->fs:
    
    fs->root = chroot root (or /)
    fs->pwd = current working directory
    
Used by namei.c:
    
    Absolute path:
        nd->root = current->fs->root
        
    Relative path:
        nd->path = current->fs->pwd
        
chroot():
    Changes current->fs->root
    Process can't escape!
    
chdir():
    Changes current->fs->pwd
    Relative paths affected
```

---

# Summary

## What You've Mastered

**namei.c is the path resolution engine:**

```
What it does:
    Translates string paths → inodes
    Entry point for all file operations
    Handles complexity transparently

Key mechanisms:
    RCU lockless lookups (99% case)
    dcache integration (cache layer)
    Component-by-component walking
    Symlink resolution (iterative)
    Mount point crossing
    Permission verification
```

---

## Key Takeaways

**RCU lookup:**

```
Default mode: LOOKUP_RCU
    Read without locks
    Use sequence numbers
    Detect concurrent changes
    Fall back to locked mode if needed
    
Performance:
    RCU: ~100-200ns
    Locked: ~500ns-1μs
    3-5× speedup
    
Scalability:
    Linear scaling to 100+ cores
    Why modern Linux handles millions of ops/sec
```

**Component walk:**

```
link_path_walk():
    For each component:
        Hash name
        dcache lookup (RCU)
        Cache hit: continue
        Cache miss: ask filesystem
        Permission check
        Handle . and ..
        
    Last component:
        Save for caller
        Different handling based on operation
```

**Symlink handling:**

```
Iterative (not recursive):
    Stack in nameidata
    Push state on symlink
    Walk target path
    Pop on completion
    
Depth limit: 40
    Prevents infinite loops
    ELOOP on exceed
    
Types:
    Absolute: restart from root
    Relative: continue from symlink dir
```

**Mount crossing:**

```
Every component:
    Check d_mountpoint flag
    If set: lookup_mnt()
    Switch to child filesystem
    
Upward (..):
    Check if at mount root
    If yes: switch to parent mount
    Continue in parent
    
Enables:
    Multiple filesystems
    Seamless navigation
    /proc, /sys, /dev all work
```

**Permission checking:**

```
Every component:
    Directories: need execute (x)
    Final file: depends on operation
    
DAC (Unix bits):
    Owner/group/other
    Read/write/execute
    
MAC (LSM):
    SELinux, AppArmor
    After DAC check
    Can override DAC
```

**Complete flow:**

```
open("/etc/passwd"):
    
    Syscall entry
    Initialize nameidata
    Walk "etc" (dcache hit, ~50ns)
    Check execute permission (~20ns)
    Walk "passwd" (dcache hit, ~50ns)
    Check read permission (~20ns)
    vfs_open() (~30ns)
    fd_install() (~20ns)
    Return fd
    
Total: ~200-500ns (cache hot)
```

**Physical reality:**

```
dcache hit rate: 99%+
    Common paths always cached
    Disk reads rare
    
RCU effectiveness:
    Lockless 99% of time
    Linear core scaling
    
Path length:
    Each component: ~70ns
    Shallow paths faster
```

**Connections:**

```
dcache.c: Provides cache layer
inode.c: Final result
ext4/namei.c: Cache miss handler
super.c: Mount table
security/: LSM hooks
task_struct: root and pwd
```

---

## You're Ready!

With this knowledge, you can:
- Understand every file operation entry point
- Debug permission denied errors
- Analyze path lookup performance
- Understand container path isolation
- See how dcache impacts everything
- Know why RCU matters for scalability

> **Path resolution is the gateway to the entire VFS - you now understand how string paths become real file objects, and why Linux can handle millions of lookups per second.**

---

**The bridge from names to inodes.**
