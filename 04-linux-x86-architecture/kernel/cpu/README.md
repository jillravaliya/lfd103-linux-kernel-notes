# CPU Detection & Setup: How Linux Discovers Your CPU

> Ever wondered how Linux knows if your CPU has AVX? How it detects Intel vs AMD? How it finds Spectre vulnerabilities? How the same kernel works on thousands of different CPUs?

**You're about to discover how CPUs describe themselves!**

---

## What's Inside This Guide?

> We're exploring **arch/x86/kernel/cpu/** - The CPU DISCOVERY system!

Here's what discovers your hardware:

```
arch/x86/kernel/
└── cpu/                   ← CPU detection subsystem
    ├── common.c           ← Generic detection
    ├── intel.c            ← Intel-specific
    ├── amd.c              ← AMD-specific
    └── bugs.c             ← Vulnerability detection
```

**This code discovers:**
- CPU vendor (Intel, AMD, VIA)
- Features (SSE, AVX, AES, etc.)
- Cache sizes (L1, L2, L3)
- Core counts
- Known bugs (Spectre, Meltdown)
- Everything kernel needs!

---

# The Problem: One Kernel, Many CPUs

## The Challenge

```
Same Linux kernel must run on:
├── Intel Core i9 (2024) - Has AVX-512, 24 cores
├── AMD Ryzen 9 (2024) - Has AVX2, 16 cores
├── Old Pentium 4 (2004) - Only SSE2, 1 core
└── Hundreds of other CPUs!

Each CPU has different:
├── Features (AVX? AES? Virtualization?)
├── Bugs (Spectre? Meltdown?)
├── Capabilities
└── Quirks

Question: How does kernel adapt to each?
```

---

## Without Detection: Disaster!

```
Kernel assumes AVX exists:

    vmovdqa ymm0, [rsi]  // 256-bit AVX instruction
    
Running on old Pentium 4 (no AVX):
└── Invalid Opcode Exception! 
    Kernel crashes! 
    System unbootable!

Kernel ignores Spectre bug:
└── No mitigations applied
    System vulnerable! 
    Data leaked!

All BAD! 
```

---

## The Solution: Runtime Detection

```
At boot time:
    ↓
Detect CPU vendor
    ↓
Detect features
    ↓
Check for bugs
    ↓
Apply mitigations
    ↓
Enable features
    ↓
Configure optimally

Same kernel, works everywhere! 
```

---

# The CPUID Instruction

> **The CPU's self-description mechanism**

## What Is CPUID?

```
CPUID = Special x86 instruction that returns CPU info

Think of it like:
└── API to query hardware capabilities

How it works:
├── Input: EAX register (what you want to know)
├── Execute: CPUID instruction
└── Output: EAX, EBX, ECX, EDX (the information)

Example:
    MOV EAX, 0      // Function 0: Get vendor
    CPUID           // Execute
    // EBX, EDX, ECX now contain vendor string!

Hardware responds instantly! 
Cannot be faked! Built into silicon! 
```

---

## What Can You Query?

```
CPUID Functions:

EAX = 0: Vendor string
└── Returns: "GenuineIntel" or "AuthenticAMD"

EAX = 1: Features and model
└── Returns: Family, model, stepping, feature flags

EAX = 4: Cache information
└── Returns: L1, L2, L3 sizes

EAX = 7: Extended features
└── Returns: AVX2, SHA, BMI, etc.

EAX = 0x80000002-4: Brand string
└── Returns: "Intel(R) Core(TM) i9-13900K..."

Many more functions available!
```

---

# Complete Detection Flow

## Step 1: Detect Vendor

```
Execute CPUID with EAX=0:

Returns vendor string (12 characters):
├── Intel: "GenuineIntel"
├── AMD: "AuthenticAMD"
├── VIA: "CentaurHauls"
└── Cyrix: "CyrixInstead"

Kernel identifies:
    if (vendor == "GenuineIntel")
        cpu_vendor = INTEL;
    else if (vendor == "AuthenticAMD")
        cpu_vendor = AMD;

Vendor known! 
```

---

## Step 2: Get Model Information

```
Execute CPUID with EAX=1:

Returns in EAX register:
├── Family (bits 11-8)
├── Model (bits 7-4)
├── Stepping (bits 3-0)
└── Extended bits for newer CPUs

Example: Intel Core i9-13900K
├── Family: 6
├── Extended Model: 11
├── Model: 7
├── Calculated: Model = (11 << 4) + 7 = 183
└── Stepping: 1

This identifies exact CPU type! 
```

---

## Step 3: Detect Features

```
Execute CPUID with EAX=1:

Returns feature flags in ECX and EDX:

EDX flags (Standard):
├── Bit 0:  FPU - Floating Point Unit
├── Bit 6:  PAE - Physical Address Extension (>4GB RAM)
├── Bit 9:  APIC - Advanced Interrupt Controller
├── Bit 15: CMOV - Conditional Move
├── Bit 23: MMX - MMX instructions
├── Bit 25: SSE - SSE instructions
└── Bit 26: SSE2 - SSE2 instructions

ECX flags (Additional):
├── Bit 0:  SSE3
├── Bit 9:  SSSE3
├── Bit 19: SSE4.1
├── Bit 20: SSE4.2
├── Bit 25: AES - AES instructions
├── Bit 28: AVX - 256-bit vectors
└── Bit 30: RDRAND - Hardware RNG

Check each bit:
    if (ECX & (1 << 28)):
        CPU has AVX! 

Execute CPUID with EAX=7 for more:
├── AVX2
├── AVX-512
├── SHA extensions
└── Many more!
```

---

## Step 4: Get Brand String

```
Execute CPUID 3 times:
├── EAX = 0x80000002 → 16 characters
├── EAX = 0x80000003 → 16 characters
└── EAX = 0x80000004 → 16 characters

Total: 48 characters

Example:
└── "Intel(R) Core(TM) i9-13900K CPU @ 3.00GHz    "

This appears in /proc/cpuinfo! 
```

---

## Step 5: Detect Cache Sizes

```
Execute CPUID with EAX=4 (Intel) or 0x8000001D (AMD):

Returns cache information:
├── L1 Data: 32 KB per core
├── L1 Instruction: 32 KB per core
├── L2: 1 MB per core
└── L3: 36 MB shared

Used for optimization decisions! 
```

---

## Step 6: Count Cores and Threads

```
From CPUID function 1 and 11:

Returns topology:
├── Physical cores: 24
├── Logical threads: 32
└── Hyperthreading: Yes (on some cores)

Example: Intel Core i9-13900K
├── 8 P-cores (Performance) with HT = 16 threads
├── 16 E-cores (Efficiency) no HT = 16 threads
└── Total: 24 cores, 32 threads 
```

---

## Step 7: Detect Bugs

```
Check CPU against known vulnerability database:

Spectre (Branch prediction attack):
├── Affects: Most Intel, some AMD
├── Mitigation: IBRS, retpoline
└── Performance cost: 3-10% 

Meltdown (Memory read attack):
├── Affects: Most Intel (pre-2019)
├── Mitigation: KPTI (separate page tables)
└── Performance cost: 5-30% 

MDS (Microarchitectural Data Sampling):
├── Affects: Intel CPUs
└── Mitigation: Buffer clearing

For each bug:
    if (cpu_vulnerable):
        apply_mitigation()  // Slower but secure 
    else:
        skip_mitigation()   // Faster! 
```

---

## After Detection: Configuration

## Enable Features

```
If AVX detected:
    Enable in CR4 register
    Kernel can use 256-bit vectors! 

If AES-NI detected:
    Use hardware encryption
    10× faster than software! 

If RDRAND detected:
    Use hardware random number generator
    Better entropy!

If XSAVE detected:
    Use for FPU state save/restore
    Faster context switching! 
```

---

## Apply Vendor Optimizations

```
Intel CPUs:
├── Use Intel-specific features
├── Enable Hyperthreading awareness
└── Use SYSENTER (older models)

AMD CPUs:
├── Use AMD power management
├── Different cache strategy
└── Prefer SYSCALL

Each vendor gets optimal config! 
```

---

## Set Up Workarounds

```
If Meltdown vulnerable:
    Enable KPTI:
    ├── Separate kernel/user page tables
    ├── Switch CR3 on every syscall
    ├── Performance: -5% to -30% 
    └── But secure! 

If Spectre vulnerable:
    Enable mitigations:
    ├── IBRS (branch restrictions)
    ├── Retpoline (software fix)
    ├── Performance: -3% to -15% 
    └── But secure! 

Security vs Performance trade-off! 
```

---

## Real Example: Intel Core i9-13900K

```
Complete detection results:

Vendor: GenuineIntel 
Family: 6
Model: 183 (Raptor Lake, 13th Gen)
Stepping: 1

Features Detected:
├── SSE, SSE2, SSE3, SSSE3, SSE4.1, SSE4.2 
├── AVX, AVX2
├── FMA, AES-NI, SHA 
├── RDRAND, RDSEED 
├── VMX (Virtualization) 
└── Hundreds more!

Brand String:
└── "Intel(R) Core(TM) i9-13900K CPU @ 3.00GHz"

Cache:
├── L1 Data: 48 KB per core
├── L1 Instruction: 32 KB per core
├── L2: 2 MB per core
└── L3: 36 MB shared

Topology:
├── 24 physical cores (8 P + 16 E)
└── 32 logical threads

Vulnerabilities:
├── Spectre: Affected (apply IBRS)
├── Meltdown: NOT affected (fixed in hardware!)
└── MDS, TAA: NOT affected 

Configuration:
├── Enable: AVX2, AES-NI, SHA
├── Mitigation: IBRS only (for Spectre)
└── Optimization: Hybrid scheduler 

Result: Kernel fully optimized! 
```

---

# Where Information Is Stored

## Kernel Structure

```
struct cpuinfo_x86 {
    u8 x86_vendor;           // INTEL or AMD
    u8 x86;                  // Family
    u8 x86_model;            // Model number
    u8 x86_stepping;         // Stepping
    
    char x86_vendor_id[16];  // "GenuineIntel"
    char x86_model_id[64];   // Brand string
    
    int x86_cache_size;      // L3 in KB
    u32 x86_capability[];    // Feature flags
    u32 x86_bugs[];          // Bug flags
};

Available throughout kernel! 
```

---

## User Space: /proc/cpuinfo

```bash
$ cat /proc/cpuinfo

processor	: 0
vendor_id	: GenuineIntel
cpu family	: 6
model		: 183
model name	: Intel(R) Core(TM) i9-13900K
stepping	: 1
cpu MHz		: 3000.000
cache size	: 36864 KB
flags		: fpu vme de pse tsc msr pae mce cx8 apic
              sep mtrr pge mca cmov pat pse36 clflush
              mmx fxsr sse sse2 ht syscall nx rdtscp
              lm constant_tsc pni pclmulqdq ssse3 fma
              cx16 sse4_1 sse4_2 movbe popcnt aes
              xsave avx f16c rdrand lahf_lm abm
              3dnowprefetch invpcid_single ibrs ibpb
              stibp fsgsbase bmi1 avx2 smep bmi2
              erms invpcid rdseed adx smap clflushopt
              sha_ni xsaveopt xsavec xgetbv1 arat
              md_clear flush_l1d arch_capabilities
bugs		: spectre_v1 spectre_v2
bogomips	: 6000.00

Applications can read this! 
```

---

# Physical Reality

## CPUID in Silicon

```
What actually happens:

CPUID instruction executed
    ↓
CPU microcode activates
    ↓
Reads from internal ROM/fuses
    ↓
Information burned into silicon!
    ↓
Returns in nanoseconds

Vendor ID: Hardcoded in ROM at manufacturing
Features: Reflect actual silicon capabilities
Cache sizes: Actual hardware configuration

Hardware truth, cannot be faked! 
```

---

## Feature Flags = Real Hardware

```
AVX flag set?
└── CPU physically HAS 256-bit ALUs! 

AES-NI flag set?
└── CPU has AES encryption circuits! 

VMX flag set?
└── CPU has virtualization hardware! 

Flags tell the truth about silicon! 
```

---

# Connections to Other Topics

## System Calls (Topic 1)

```
Detection determines system call mechanism:

Intel (old): Use SYSENTER/SYSEXIT
Intel (new): Use SYSCALL/SYSRET
AMD: Always use SYSCALL/SYSRET

Optimal mechanism chosen! 
```

## Interrupts (Topic 3)

```
APIC detection:

CPUID function 1, EDX bit 9:
└── If set: APIC present! 

x2APIC detection:
└── CPUID function 1, ECX bit 21
    Use if available (faster!)
```

## Context Switching (Topic 4)

```
XSAVE feature:

CPUID function 1, ECX bit 26:
└── If set: Use XSAVE for FPU/SSE/AVX 
    Faster context switching!

Hyperthreading:
└── Affects scheduler decisions
    Don't run same process on HT siblings
```

## Memory (mm/)

```
PAE: Physical Address Extension
└── CPUID bit 6: Can use >4GB RAM 

Huge pages:
└── CPUID bit 3: PSE support 

NX bit: No eXecute
└── W^X enforcement possible
```

---

# Summary

## What Is CPU Detection?

```
Discovering CPU capabilities via CPUID:
├── Vendor (Intel, AMD, etc.)
├── Features (SSE, AVX, AES, etc.)
├── Bugs (Spectre, Meltdown)
└── Configuration needed
```

## Why Does It Exist?

```
One kernel, many CPUs:
├── Use available features 
├── Avoid missing features 
├── Apply bug fixes 
└── Optimize per-vendor 
```

## How Does It Work?

```
CPUID instruction:
├── Hardware query API
├── Input: Function number
├── Output: CPU information
└── Built into all x86 CPUs!
```

## What's Detected?

```
Everything:
├── Vendor: Intel, AMD, VIA
├── Model: Exact CPU type
├── Features: Hundreds of flags
├── Cache: L1, L2, L3 sizes
├── Topology: Cores, threads
├── Bugs: Known vulnerabilities
└── Brand: Human-readable name
```

## Complete Flow

```
Boot → Query vendor → Query features →
Query cache → Query topology → Check bugs →
Apply mitigations → Enable features →
Configure optimally → Ready!
```

---

## The Big Reveal

**One boot line:**

```
[  0.001234] CPU: Intel Core i9-13900K (family: 0x6, model: 0xb7)
```

**Hides this universe:**

```
Hundreds of CPUID queries
Feature checking
Bug scanning
Mitigation decisions
Optimization choices
Complete CPU profiling

All in milliseconds! 
```

---

## Your New Powers

```
Read /proc/cpuinfo with understanding
Know what your CPU can do
Check vulnerability status
Understand security advisories
Tune kernel mitigations
Write CPU-aware code
```

---

## A Final Thought

**Every time you see:**

```bash
$ grep flags /proc/cpuinfo
flags: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge
       mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall
       nx rdtscp lm pni pclmulqdq ssse3 fma cx16 sse4_1 sse4_2
       movbe popcnt aes xsave avx f16c rdrand ...
```

**Remember:**

```
Not just abbreviations
    ↓
Each flag = Silicon capability
    ↓
Detected via CPUID
    ↓
Stored in kernel
    ↓
Used for optimization
    ↓
Applied for security
    ↓
That's CPU detection!
```

---

> You now understand how CPUs describe themselves to the kernel! 
