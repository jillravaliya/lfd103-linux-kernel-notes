# From Peripheral to CPU - The PCIe Subsystem

> Ever wondered how your NVMe SSD, graphics card, and network adapter all connect to your CPU? How the kernel discovers what devices are present? How each device gets its own memory-mapped registers?

**You're about to find out!**

---

## What's This About?

This is **drivers/pci/** - the subsystem that manages Peripheral Component Interconnect Express, the high-speed bus connecting all major peripherals!

Here's where it fits in the driver ecosystem:

```
drivers/
04-device-drivers/
├── 00-foundation/      ← Device registration in kernel
├── 01-input/           ← Input events to userspace
├── 02-block/           ← Storage read and write
├── 03-net/             ← Packet send and receive
├── 04-usb/             ← USB device communication
├── 05-pci/             ← PCIe device discovery and initialization
│   ├── BDF addressing  ← Bus:Device.Function unique ID
│   ├── Configuration   ← 4KB config space per device
│   ├── BARs            ← Base Address Registers (MMIO)
│   ├── MSI-X           ← Message Signaled Interrupts
│   ├── DMA             ← Bus mastering for devices
│   └── Power mgmt      ← D-states, ASPM link states
├── 06-char/            ← Character device data flow
└── 07-gpu/             ← Display and rendering pipeline
```

**Location in code:**

```
drivers/pci/
├── pci.c               ← Core PCI infrastructure
├── probe.c             ← Device discovery and probing
├── bus.c               ← PCI bus operations
├── msi.c               ← MSI/MSI-X interrupt setup
├── iov.c               ← SR-IOV (virtual functions)
├── host/               ← PCIe root complex drivers
│   └── pcie-dw.c       ← DesignWare PCIe (ARM SoCs)
└── pcie/
    ├── aer.c           ← Advanced Error Reporting
    └── aspm.c          ← Active State Power Management

include/linux/
└── pci.h               ← struct pci_dev, pci_driver, APIs

Architecture-specific:
└── arch/x86/pci/       ← x86 PCI initialization
```

**This covers:**
- PCIe topology and addressing (Root Complex, bridges, endpoints)
- BDF addressing (Bus:Device.Function)
- Configuration space (vendor ID, BARs, capabilities)
- Driver lifecycle (probe, remove, power management)
- MSI and MSI-X interrupts (memory-write based)
- DMA bus mastering (device-initiated memory access)
- Complete NVMe initialization from cold boot

---

# The Fundamental Problem

**Physical Reality:**

```
Modern computer with diverse peripherals:

Storage:
├── NVMe SSD (Samsung 980 Pro, PCIe Gen 4 x4)
├── SATA SSD (connected via AHCI controller, PCIe)
└── External USB drive (via xHCI controller, PCIe)

Network:
├── Ethernet (Intel I219-V, integrated PCIe)
├── WiFi (Intel AX200, M.2 PCIe)
└── Bluetooth (often integrated with WiFi)

Graphics:
├── Integrated GPU (Intel Iris Xe, in CPU die)
└── Discrete GPU (NVIDIA RTX 3080, PCIe x16)

Other:
├── USB controller (xHCI, PCIe)
├── Audio controller (HDA, PCIe)
└── Thunderbolt controller (PCIe)

All connected via PCIe bus
All different hardware from different vendors
All need CPU access to their registers
All need to DMA to system RAM
```

**The problem:**

```
Without unified PCIe layer:

Device addressing chaos:
├── How does CPU know where NVMe registers are?
├── Physical address space limited (4GB in 32-bit)
├── Each device needs unique address range
├── Conflicts if two devices claim same address
└── Who assigns these addresses?

NVMe driver must:
├── Find NVMe controller on bus somehow
├── Determine where its registers are mapped
├── Claim those memory regions
├── Enable device power
├── Configure interrupts
└── Enable DMA

Network driver must:
├── Do all the same things
├── But for completely different hardware
└── Reinvent all discovery/setup logic

Result:
├── Massive code duplication
├── Addressing conflicts
├── No hotplug support
└── Every driver reimplements enumeration


Interrupt handling disaster:

Old PCI (shared interrupts):
├── All devices share IRQ lines (IRQ 11, IRQ 15)
├── NIC triggers interrupt → handler must poll NIC
├── GPU triggers interrupt → handler must poll GPU
├── Both on same IRQ? Poll both!
└── Terrible for performance

Need per-device interrupts:
├── Each device should have own interrupt
├── Routed to specific CPU for locality
└── No shared interrupt overhead


DMA security nightmare:

Devices can write to RAM directly:
├── Malicious device writes kernel code → exploit
├── Buggy device writes random memory → crash
├── Who validates device memory access?
└── Need IOMMU but who configures it?
```

**How do you provide ONE system for device discovery, addressing, interrupts, and DMA across all PCIe hardware?**

---

# Without PCIe Subsystem

**Imagine no unified PCIe layer:**

```
Every driver finds its own device:

NVMe driver boot:
├── Must scan entire physical memory space
├── Looking for NVMe controller signature
├── Read vendor ID from every possible address?
│   └── 0xFC000000: read → bus error
│   └── 0xFC100000: read → bus error
│   └── 0xFC200000: read → 0x8086 (Intel!)
│       └── Is this NVMe? Check device ID...
└── 65,536 possible device locations

Network driver boot:
├── Does same scan independently
├── Conflicts with NVMe driver scanning
└── Both waste boot time

No standardized initialization:
├── Each driver invents own method
├── Register access protocols differ
└── Power state transitions inconsistent


Address assignment chaos:

Physical address space layout:
├── 0x00000000 - 0x7FFFFFFF: System RAM (2GB)
├── 0x80000000 - 0xFFFFFFFF: Devices (2GB available)

Someone must assign:
├── NVMe: 0xFC000000 - 0xFC003FFF (16KB)
├── NIC:  0xFC004000 - 0xFC007FFF (16KB)
├── GPU:  0xD0000000 - 0xDFFFFFFF (256MB)
└── xHCI: 0xFC008000 - 0xFC00BFFF (16KB)

Who does assignment?
├── BIOS/firmware? Kernel? Drivers?
├── What if two drivers want same range?
└── No coordination mechanism


Interrupt configuration nightmare:

NVMe needs interrupts:
├── How many? (one per CPU for parallelism)
├── Which interrupt vectors? (who assigns?)
├── Routing to which CPUs? (who decides?)
└── Each driver separately negotiates with APIC

No MSI-X support:
├── Stuck with shared legacy interrupts
├── Performance terrible
└── Modern hardware capabilities unused


DMA impossible without coordination:

NVMe wants to DMA:
├── Driver: "Write to physical address 0x12340000"
├── But virtual memory means physical addresses change
├── Who maps pages for DMA?
├── IOMMU needs page tables but who sets them up?
└── One wrong DMA address = system crash


Hotplug impossible:

User inserts PCIe NVMe drive:
├── How does kernel know device inserted?
├── Who reads configuration?
├── Who assigns memory addresses?
├── Who loads driver?
└── Everything manual, nothing automatic
```

**Complete chaos without PCIe infrastructure!**

---

# The Hierarchical Solution

**Unified PCIe architecture:**

```
Layer 1: HARDWARE TOPOLOGY
Root Complex (in CPU/chipset)
├── Root Port 0 → NVMe SSD (PCIe Gen 4 x4)
├── Root Port 1 → PCIe Switch
│   ├── Downstream Port 0 → Network card
│   └── Downstream Port 1 → Sound card
├── Root Port 2 → Graphics card (x16)
└── Root Port 3 → xHCI USB controller

Each connection is point-to-point
Full duplex simultaneous communication
Speeds: Gen 1 (2.5 GT/s) to Gen 6 (64 GT/s)


Layer 2: ADDRESSING AND CONFIGURATION
BDF: Bus:Device.Function format
├── 00:02.0 → Integrated GPU
├── 01:00.0 → NVMe SSD
├── 02:00.0 → Network card
└── 03:00.0 → Graphics card

Each device: 4KB configuration space
├── Vendor ID (0x8086 = Intel)
├── Device ID (specific product)
├── BARs (Base Address Registers)
├── Capabilities (MSI-X, Power Management)
└── Device-specific registers


Layer 3: KERNEL INFRASTRUCTURE
PCI core (drivers/pci/):
├── Enumerates devices at boot
├── Assigns BARs to physical addresses
├── Creates struct pci_dev for each device
├── Matches drivers to devices
└── Manages power states

Provides to drivers:
├── pci_enable_device()
├── pci_set_master() (enable DMA)
├── pci_request_mem_regions()
├── ioremap() (map BARs)
├── pci_alloc_irq_vectors() (MSI-X)
└── request_irq() (register handlers)


Result:
├── NVMe driver: 50 lines of PCIe setup
├── Network driver: same 50 lines
├── Shared infrastructure: 50,000 lines once
└── Every driver benefits
```

---

# 1. PCIe Topology and Speeds

## The Hardware Hierarchy

**PCIe architecture:**

```
Point-to-point serial links (not a shared bus):

Old PCI (parallel):
└── All devices share one bus
    Only one device talks at once
    Maximum ~133 MB/s shared

PCIe (serial):
└── Each device has dedicated link to host
    All devices talk simultaneously
    Speeds scale with lanes and generation

Lane composition:
One lane = two differential pairs:
├── TX+ / TX- (transmit from host)
└── RX+ / RX- (receive to host)
    Full duplex (simultaneous both directions)
```

**PCIe speeds:**

```
Generation and bandwidth:

Gen 1 (PCIe 1.0):
├── Raw: 2.5 GT/s (GigaTransfers per second)
├── Encoding: 8b/10b (20% overhead)
└── Effective: ~250 MB/s per lane

Gen 2 (PCIe 2.0):
├── Raw: 5.0 GT/s
├── Encoding: 8b/10b
└── Effective: ~500 MB/s per lane

Gen 3 (PCIe 3.0):
├── Raw: 8.0 GT/s
├── Encoding: 128b/130b (1.5% overhead)
└── Effective: ~1 GB/s per lane

Gen 4 (PCIe 4.0):
├── Raw: 16.0 GT/s
├── Encoding: 128b/130b
└── Effective: ~2 GB/s per lane

Gen 5 (PCIe 5.0):
├── Raw: 32.0 GT/s
├── Encoding: 128b/130b
└── Effective: ~4 GB/s per lane

Gen 6 (PCIe 6.0):
├── Raw: 64.0 GT/s
├── Encoding: FLIT (low overhead)
└── Effective: ~8 GB/s per lane

Lane configurations:
├── x1: 1 lane (NVMe in M.2 2230)
├── x4: 4 lanes (most NVMe SSDs)
├── x8: 8 lanes (high-end NICs)
└── x16: 16 lanes (graphics cards)

Examples:
NVMe SSD (x4 Gen 4):
└── 4 lanes × 2 GB/s = 8 GB/s bidirectional

Graphics card (x16 Gen 5):
└── 16 lanes × 4 GB/s = 64 GB/s each direction

Network card (x8 Gen 3):
└── 8 lanes × 1 GB/s = 8 GB/s
```

**Physical topology:**

```
Typical desktop system:

CPU
├── Integrated PCIe lanes (from CPU die)
│   ├── Root Port 0: Graphics card (x16)
│   └── Root Port 1: M.2 NVMe slot (x4)
└── DMI link to Chipset (x4 Gen 3)

Chipset (PCH)
├── Root Port 0: Ethernet controller
├── Root Port 1: WiFi card (M.2)
├── Root Port 2: xHCI USB controller
├── Root Port 3: SATA controller
└── Root Port 4: Additional M.2 slot

PCIe switch (optional, for expansion):
└── One upstream port connects to Root Port
    Multiple downstream ports for devices
    Example: 1 upstream → 4 downstream
    Shares upstream bandwidth

Addressing depth:
├── Bus 0: Root Complex
├── Bus 1: Devices on Root Port 0
├── Bus 2: Behind PCIe switch
└── Up to 256 buses total
```

---

# 2. BDF Addressing

## Bus:Device.Function Format

**BDF structure:**

```
Every PCIe device has unique address:

Format: BB:DD.F

Bus (BB): 0-255 (8 bits)
└── Which PCIe bus segment

Device (DD): 0-31 (5 bits)
└── Which device on that bus

Function (F): 0-7 (3 bits)
└── Which function within device
    Multi-function devices: one chip, many functions

Examples:
00:00.0 → Host bridge (Root Complex)
00:02.0 → Integrated GPU (Intel)
00:1f.3 → Audio controller
01:00.0 → NVMe SSD (bus 1, first device)
02:00.0 → Discrete GPU
03:00.0 → Network card

lspci command shows these:
$ lspci
00:00.0 Host bridge: Intel Corporation
00:02.0 VGA compatible controller: Intel HD Graphics
00:1f.3 Audio device: Intel HD Audio
01:00.0 Non-Volatile memory controller: Samsung 980 PRO
02:00.0 VGA compatible controller: NVIDIA RTX 3080
03:00.0 Ethernet controller: Intel I219-V
```

**Configuration space access:**

```
Each device has 4KB configuration space:

Legacy method (x86 I/O ports):
├── CONFIG_ADDRESS (0xCF8): Write BDF + offset
├── CONFIG_DATA (0xCFC): Read/write data
└── Limited to first 256 bytes

Modern method (ECAM - Enhanced Configuration Access Mechanism):
└── Memory-mapped configuration space
    Base address + (Bus << 20) + (Dev << 15) + (Func << 12) + Offset
    
Example ECAM access:
Base: 0xE0000000
Device 01:00.0 offset 0x10:
└── Address: 0xE0000000 + (1 << 20) + (0 << 15) + (0 << 12) + 0x10
            = 0xE0100010

Direct memory read from this address reads BAR0
```

---

# 3. Configuration Space

## Standard Header Format

**Configuration space layout:**

```
Offset  Size  Field                    Purpose
0x00    2     Vendor ID                0x8086 (Intel), 0x10DE (NVIDIA)
0x02    2     Device ID                Specific product identifier
0x04    2     Command                  Memory access enable, bus master
0x06    2     Status                   Capability list, error flags
0x08    1     Revision ID              Silicon revision
0x09    3     Class Code               0x010802 (NVMe), 0x020000 (Ethernet)
0x0C    1     Cache Line Size          Cache optimization hint
0x0D    1     Latency Timer            Legacy PCI timing
0x0E    1     Header Type              0x00 (endpoint) or 0x01 (bridge)
0x0F    1     BIST                     Built-in self test

0x10    4     BAR0                     Base Address Register 0
0x14    4     BAR1                     Base Address Register 1
0x18    4     BAR2                     Base Address Register 2
0x1C    4     BAR3                     Base Address Register 3
0x20    4     BAR4                     Base Address Register 4
0x24    4     BAR5                     Base Address Register 5

0x28    4     Cardbus CIS Pointer      Legacy
0x2C    2     Subsystem Vendor ID      Board manufacturer
0x2E    2     Subsystem ID             Board-specific ID
0x30    4     Expansion ROM BAR        Option ROM location

0x34    1     Capabilities Pointer     Offset to first capability
0x3C    1     Interrupt Line           Legacy IRQ number
0x3D    1     Interrupt Pin            INTA/INTB/INTC/INTD
0x3E    1     Min Grant                Legacy timing
0x3F    1     Max Latency              Legacy timing

0x40+   ...   Extended capabilities    MSI, MSI-X, Power Mgmt, etc.
```

**Vendor and Device ID:**

```
Identification:

Vendor ID examples:
├── 0x8086: Intel Corporation
├── 0x10DE: NVIDIA Corporation
├── 0x1022: AMD
├── 0x144D: Samsung Electronics
└── 0x1B4B: Marvell

Device ID:
└── Vendor-specific product identifier
    Example: Samsung 980 PRO = 0xA80A

Subsystem IDs:
└── Board manufacturer specifics
    Same GPU chip, different board vendor
    0x1043 = ASUS, 0x1458 = Gigabyte

Class code (3 bytes):
├── Base class (0x01 = Mass Storage)
├── Sub-class (0x08 = NVMe)
└── Programming interface (0x02 = NVMe spec)

Class code examples:
├── 0x010802: NVMe controller
├── 0x020000: Ethernet controller
├── 0x030000: VGA controller
├── 0x0C0330: xHCI USB controller
└── 0x060400: PCI bridge
```

---

# 4. BARs (Base Address Registers)

## Memory-Mapped Device Registers

**What are BARs:**

```
BAR = Base Address Register
Purpose: Expose device memory/registers to CPU

Devices have registers for control:
├── NVMe: doorbell registers, admin queue pointers
├── NIC: TX/RX ring pointers, control registers
└── GPU: command buffer pointers, display registers

These registers need CPU-accessible addresses:
└── Device cannot choose addresses itself
    CPU memory space is limited
    Need coordination to avoid conflicts

BAR mechanism:
1. Device declares size needed (via BAR)
2. BIOS/firmware assigns physical address
3. Driver maps address via ioremap()
4. Driver reads/writes device registers
```

**BAR detection and assignment:**

```
Firmware (BIOS/UEFI) at boot:

For each device:
1. Write all 1s to BAR register
2. Read back BAR value
3. Decode size from read value

Example NVMe device:
Write: 0xFFFFFFFF to BAR0
Read back: 0xFFFF0000

Decoding:
└── Lowest set bit indicates size
    0xFFFF0000 means bits 0-15 are writable
    Size = 2^16 = 64KB

Assign address:
└── Find free 64KB-aligned region
    Example: 0xFC000000
    Write 0xFC000000 to BAR0

Now device registers accessible at:
├── 0xFC000000 + 0x0000: Register 0
├── 0xFC000000 + 0x1000: Register 4096
└── 0xFC000000 + 0xFFFF: Last register

Driver usage:
void __iomem *regs = ioremap(0xFC000000, 65536);
writel(value, regs + NVME_REG_CC);
```

**BAR types:**

```
Memory BAR (most common):
├── Bit 0: 0 (indicates memory BAR)
├── Bits 1-2: Type (00 = 32-bit, 10 = 64-bit)
├── Bit 3: Prefetchable (1 = yes, 0 = no)
└── Bits 4-31 (or 63): Base address

32-bit BAR:
└── Single BAR register
    Address < 4GB
    
64-bit BAR:
└── Uses two consecutive BAR slots
    BAR[n] = low 32 bits
    BAR[n+1] = high 32 bits
    Allows addressing > 4GB

Prefetchable:
├── CPU can cache reads (speculative reads OK)
├── Used for: GPU framebuffers, large buffers
└── Not used for: Control registers (state changes)

I/O BAR (legacy):
├── Bit 0: 1 (indicates I/O space)
├── Uses x86 I/O port space (inb/outb)
└── Avoid in modern drivers
```

**Reading BARs in code:**

```c
Kernel PCI resource structure:

struct pci_dev *pdev;  // From probe()

// Get BAR 0 physical address
phys_addr_t bar_phys = pci_resource_start(pdev, 0);
// Example: 0xFC000000

// Get BAR 0 size
resource_size_t bar_size = pci_resource_len(pdev, 0);
// Example: 65536

// Check BAR flags
unsigned long bar_flags = pci_resource_flags(pdev, 0);
// IORESOURCE_MEM, IORESOURCE_PREFETCH, etc.

// Claim the BAR (prevent other drivers)
pci_request_mem_regions(pdev, "nvme");

// Map into virtual address space
void __iomem *bar_virt = ioremap(bar_phys, bar_size);
// Returns: 0xFFFF888100000000 (virtual address)

// Now access registers
u32 version = readl(bar_virt + NVME_REG_VS);
writel(config, bar_virt + NVME_REG_CC);

// Cleanup
iounmap(bar_virt);
pci_release_mem_regions(pdev);
```

---

# 5. Key Data Structures

## struct pci_dev

**Device representation:**

```c
struct pci_dev {
    // Addressing
    struct pci_bus *bus;        // Which bus
    unsigned int devfn;         // Device (5 bits) + Function (3 bits)
    unsigned short vendor;      // 0x8086 (Intel)
    unsigned short device;      // Product ID
    unsigned short subsystem_vendor;
    unsigned short subsystem_device;
    unsigned int class;         // 0x010802 (NVMe)
    u8 revision;                // Silicon revision
    
    // Configuration space cache
    // Kernel reads these once at enumeration
    
    // Resources (BARs)
    struct resource resource[DEVICE_COUNT_RESOURCE];
    // resource[0-5] = BAR0-5
    // resource[6] = Expansion ROM
    // resource[7-16] = Additional (bridges, SR-IOV)
    
    // Interrupt
    unsigned int irq;           // Legacy IRQ number (IRQ 11, etc.)
    
    // State
    unsigned int is_enabled:1;  // pci_enable_device() called?
    unsigned int is_busmaster:1; // DMA enabled?
    pci_power_t current_state;  // D0, D1, D2, D3hot, D3cold
    
    // MSI-X
    unsigned int msix_enabled:1;
    int msix_cap;               // Offset to MSI-X capability
    struct msix_entry *msix_table;
    
    // Device model integration
    struct device dev;          // Embedded device structure
    
    // Driver
    struct pci_driver *driver;  // Which driver claimed this
    
    // NUMA
    int numa_node;              // Which NUMA node (for locality)
    
    // Error handling
    struct pci_error_handlers *err_handler;
    
    // SR-IOV
    struct pci_sriov *sriov;    // Virtual Functions
};

Access:
struct pci_dev *pdev = ...;
printk("Device %04x:%04x at %02x:%02x.%x\n",
       pdev->vendor, pdev->device,
       pdev->bus->number,
       PCI_SLOT(pdev->devfn),
       PCI_FUNC(pdev->devfn));
```

## struct pci_driver

**Driver interface:**

```c
struct pci_driver {
    const char *name;           // "nvme", "igb", "xhci_hcd"
    
    // Device matching
    const struct pci_device_id *id_table;
    
    // Lifecycle
    int (*probe)(struct pci_dev *dev,
                 const struct pci_device_id *id);
    void (*remove)(struct pci_dev *dev);
    
    // Power management
    int (*suspend)(struct pci_dev *dev, pm_message_t state);
    int (*resume)(struct pci_dev *dev);
    
    // Error handling (AER)
    const struct pci_error_handlers *err_handler;
    
    // Device model driver (embedded)
    struct device_driver driver;
};

Registration:
static struct pci_driver nvme_driver = {
    .name       = "nvme",
    .id_table   = nvme_id_table,
    .probe      = nvme_probe,
    .remove     = nvme_remove,
    .suspend    = nvme_suspend,
    .resume     = nvme_resume,
};

module_pci_driver(nvme_driver);
// Expands to module_init() and module_exit()
```

## struct pci_device_id

**Matching table:**

```c
struct pci_device_id {
    __u32 vendor;               // Match vendor ID
    __u32 device;               // Match device ID
    __u32 subvendor;            // Match subsystem vendor
    __u32 subdevice;            // Match subsystem device
    __u32 class;                // Match class code
    __u32 class_mask;           // Mask for class matching
    kernel_ulong_t driver_data; // Driver private data
};

Matching macros:

PCI_DEVICE(vendor, device)
└── Match specific vendor and device
    Example: PCI_DEVICE(0x8086, 0x0953)  // Intel NVMe

PCI_DEVICE_CLASS(class, class_mask)
└── Match by class code
    Example: PCI_DEVICE_CLASS(0x010802, 0xFFFFFF)  // Any NVMe

PCI_ANY_ID
└── Wildcard (match any value)

Example matching table:
static const struct pci_device_id nvme_id_table[] = {
    // Match ANY NVMe controller (by class code)
    { PCI_DEVICE_CLASS(0x010802, 0xFFFFFF) },
    
    // Specific Samsung controller
    { PCI_DEVICE(0x144D, 0xA80A) },
    
    // Specific Intel controller
    { PCI_DEVICE(0x8086, 0x0953) },
    
    // Terminator
    { 0, }
};
MODULE_DEVICE_TABLE(pci, nvme_id_table);
```

---

# 6. Driver Lifecycle

## Registration and Probe

**Driver registration:**

```
Module initialization:

module_init(nvme_init)
└── pci_register_driver(&nvme_driver)
    └── driver_register(&nvme_driver.driver)
        └── bus_add_driver(&pci_bus_type)
            └── Add to pci_bus_type.drivers list
            
            For each existing pci_dev:
            └── Try to match
                If match: call probe()

Matching algorithm:
For each pci_device_id in driver's id_table:
    If vendor matches AND device matches:
    └── MATCH!
    
    OR if class matches (masked):
    └── MATCH!
    
    Call: driver->probe(pdev, matched_id)
```

**Complete probe sequence:**

```c
Example: nvme_probe()

int nvme_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    struct nvme_dev *dev;
    int result;
    
    // Step 1: Enable device
    result = pci_enable_device(pdev);
    // Powers up device
    // Enables config space access
    // Sets pdev->is_enabled = 1
    
    // Step 2: Enable bus mastering (DMA!)
    pci_set_master(pdev);
    // Sets Command register bit 2
    // Now device can initiate PCIe transactions
    // Required for DMA
    
    // Step 3: Request memory regions (claim BARs)
    result = pci_request_mem_regions(pdev, "nvme");
    // Marks BARs as in use by this driver
    // Prevents other drivers from claiming
    // Visible in /proc/iomem
    
    // Step 4: Get BAR info
    phys_addr_t bar_phys = pci_resource_start(pdev, 0);
    resource_size_t bar_size = pci_resource_len(pdev, 0);
    
    // Step 5: Map BAR into virtual address space
    dev->bar = ioremap(bar_phys, bar_size);
    if (!dev->bar) {
        result = -ENOMEM;
        goto release_regions;
    }
    
    // Step 6: Configure DMA
    result = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
    if (result) {
        // Try 32-bit
        result = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
    }
    // Tells kernel: device can DMA to these addresses
    
    // Step 7: Set up interrupts (MSI-X)
    int nr_vecs = pci_alloc_irq_vectors(pdev,
                                        1,              // min
                                        num_cpus(),     // max
                                        PCI_IRQ_MSIX);
    if (nr_vecs < 0) {
        // Fall back to MSI or legacy
        result = nr_vecs;
        goto unmap;
    }
    
    // Step 8: Register IRQ handlers
    for (i = 0; i < nr_vecs; i++) {
        int irq = pci_irq_vector(pdev, i);
        result = request_irq(irq, nvme_irq_handler,
                           0, "nvme", &dev->queues[i]);
    }
    
    // Step 9: Initialize hardware
    nvme_reset_controller(dev);
    // Device-specific initialization
    // Write to registers via ioremap'd BAR
    
    // Step 10: Register with subsystem
    nvme_init_ctrl(&dev->ctrl, &pdev->dev);
    nvme_scan_namespaces(&dev->ctrl);
    // Creates /dev/nvme0, /dev/nvme0n1
    
    // Save pointer for remove()
    pci_set_drvdata(pdev, dev);
    
    return 0;
    
unmap:
    iounmap(dev->bar);
release_regions:
    pci_release_mem_regions(pdev);
    pci_disable_device(pdev);
    return result;
}
```

**Device removal:**

```c
void nvme_remove(struct pci_dev *pdev)
{
    struct nvme_dev *dev = pci_get_drvdata(pdev);
    
    // Step 1: Quiesce device (stop new work)
    nvme_dev_disable(dev, true);
    
    // Step 2: Unregister from subsystem
    nvme_remove_namespaces(&dev->ctrl);
    // /dev/nvme0n1 removed
    
    // Step 3: Free interrupts
    pci_free_irq_vectors(pdev);
    
    // Step 4: Unmap BAR
    iounmap(dev->bar);
    
    // Step 5: Release BARs
    pci_release_mem_regions(pdev);
    
    // Step 6: Disable device
    pci_disable_device(pdev);
    
    // Step 7: Free driver data
    kfree(dev);
}
```

---

# 7. MSI and MSI-X Interrupts

## Memory-Write Based Interrupts

**Legacy interrupts (INTx):**

```
Old PCI interrupt model:

Physical interrupt lines:
├── INTA, INTB, INTC, INTD (4 lines)
├── Shared among many devices
└── IRQ numbers assigned (IRQ 11, IRQ 15, etc.)

Problems:
Device raises interrupt:
└── Pulls physical line low (electrical signal)
    Shared with other devices
    CPU receives interrupt
    Must poll all devices on that IRQ
    "Was it NVMe? Was it NIC? Was it GPU?"
    
Polling overhead:
├── Every interrupt must check multiple devices
├── Cannot route to specific CPU
└── Always goes to CPU 0 (typically)

Sharing conflicts:
├── High-frequency device and low-frequency share IRQ
├── Handler for low-frequency device called constantly
└── Wastes CPU cycles
```

**MSI (Message Signaled Interrupts):**

```
Revolution: Interrupts as memory writes!

Instead of physical wire:
└── Device writes to special memory address
    PCIe memory write transaction
    Destination: APIC (interrupt controller)

Mechanism:
Device configured with:
├── Address: 0xFEExxxxx (magic APIC range)
└── Data: Interrupt vector number

Device raises interrupt:
└── PCIe write: Address=0xFEE00000, Data=0x30
    Write travels to Root Complex
    Root Complex sees 0xFEExxxxx destination
    Routes to APIC
    APIC interprets as interrupt vector 0x30
    Delivers to appropriate CPU

Advantages:
├── No shared physical lines
├── Can route to specific CPU (via address)
├── Up to 32 MSI vectors per device
└── Each vector can have own handler

Configuration:
MSI Capability in config space:
├── Message Address Register (0xFEExxxxx)
├── Message Data Register (vector number)
├── Message Control (enable, count)
└── Mask bits (per-vector masking)
```

**MSI-X (Extended):**

```
MSI-X improvements over MSI:

MSI limitations:
├── Maximum 32 vectors
├── Vectors must be consecutive
└── Limited flexibility

MSI-X features:
├── Up to 2048 vectors per device
├── Each vector independently configurable
├── Each vector has own address/data pair
└── Per-vector masking

MSI-X table (in device BAR memory):
Each entry (16 bytes):
├── Message Address Lo (4 bytes)
├── Message Address Hi (4 bytes)
├── Message Data (4 bytes)
└── Vector Control (4 bytes)
    Bit 0: Mask bit (1 = masked)

Example table:
Entry 0:  // For Queue 0
├── Address: 0xFEE00000 (CPU 0 APIC)
├── Data: 0x30 (vector 0x30)
└── Mask: 0

Entry 1:  // For Queue 1
├── Address: 0xFEE01000 (CPU 1 APIC)
├── Data: 0x31 (vector 0x31)
└── Mask: 0

NVMe with 8 queues:
└── 8 MSI-X vectors
    Each queue completes → different vector → different CPU
    True parallel I/O completion
```

**Setting up MSI-X:**

```c
Example: NVMe driver

int nvme_setup_irqs(struct nvme_dev *dev)
{
    int nr_vecs;
    int i;
    
    // Request MSI-X vectors
    nr_vecs = pci_alloc_irq_vectors(dev->pdev,
                                    1,           // min vectors
                                    num_online_cpus(), // max vectors
                                    PCI_IRQ_MSIX);
    if (nr_vecs < 0) {
        // MSI-X failed, try MSI
        nr_vecs = pci_alloc_irq_vectors(dev->pdev, 1, 1, PCI_IRQ_MSI);
        if (nr_vecs < 0) {
            // Fall back to legacy
            nr_vecs = pci_alloc_irq_vectors(dev->pdev, 1, 1,
                                           PCI_IRQ_LEGACY);
        }
    }
    
    // What we got
    dev->num_vecs = nr_vecs;
    
    // Register handler for each vector
    for (i = 0; i < nr_vecs; i++) {
        int irq = pci_irq_vector(dev->pdev, i);
        // Returns Linux IRQ number for this MSI-X vector
        
        int cpu = cpumask_local_spread(i, dev->numa_node);
        // Spread across CPUs in same NUMA node
        
        irq_set_affinity_hint(irq, get_cpu_mask(cpu));
        // Route interrupt to specific CPU
        
        request_irq(irq, nvme_irq, 0, "nvme-q", &dev->queues[i]);
        // Register handler
    }
    
    return 0;
}

Kernel internally:
pci_alloc_irq_vectors():
└── Enables MSI-X in config space
    Allocates interrupt vectors from system
    Programs MSI-X table with APIC addresses
    Returns count of vectors obtained
```

**Interrupt flow (MSI-X):**

```
NVMe completes I/O request:

1. NVMe hardware:
   └── Writes completion entry to completion queue in RAM (via DMA)
       Determines which MSI-X vector to use (based on queue)
       Reads MSI-X table entry for that vector
       Issues PCIe memory write:
           Address: 0xFEE02000 (from table)
           Data: 0x32 (vector number from table)

2. PCIe Root Complex:
   └── Receives memory write
       Destination 0xFEExxxxx is special (APIC range)
       Routes to APIC (interrupt controller)

3. APIC:
   └── Sees write to 0xFEE02000
       Determines: CPU 2 should receive this
       Interrupt vector: 0x32
       Delivers interrupt to CPU 2

4. CPU 2:
   └── Interrupt received (vector 0x32)
       Looks up in IDT (Interrupt Descriptor Table)
       Calls: nvme_irq() handler
       Handler processes completion queue
       Wakes waiting processes

Total latency: ~1-2 microseconds
```

---

# 8. DMA (Bus Mastering)

## Device-Initiated Memory Access

**What is DMA:**

```
Direct Memory Access = Device reads/writes RAM without CPU

Traditional I/O (programmed I/O):
└── CPU reads device register
    CPU writes to memory
    CPU reads device register again
    CPU writes to memory again
    Repeat 1000 times...
    CPU busy entire time

DMA:
└── CPU tells device: "Read from RAM address X, write to Y"
    Device does everything via PCIe
    CPU does other work
    Device interrupts when done
    CPU just handles result

PCIe terminology:
├── Requester: Who initiates transaction (CPU or device)
├── Completer: Who responds to transaction
└── Bus master: Device allowed to be requester
```

**Enabling bus mastering:**

```c
pci_set_master(pdev);

What this does:
└── Reads PCI Command register (offset 0x04)
    Sets bit 2: Bus Master Enable
    Writes back to Command register
    
    Now device can initiate PCIe transactions:
    ├── Memory Read (device reads host RAM)
    ├── Memory Write (device writes host RAM)
    └── Device becomes bus master

Without pci_set_master():
└── Device can only respond to CPU requests
    Cannot DMA
    Cannot write completions
    Cannot be useful for I/O
```

**DMA mapping:**

```c
DMA directions:

DMA_TO_DEVICE:
└── CPU writes data, device reads it
    Example: Write command to NVMe

DMA_FROM_DEVICE:
└── Device writes data, CPU reads it
    Example: Read data from NVMe

DMA_BIDIRECTIONAL:
└── Both directions possible

Simple mapping:
void *cpu_addr = kmalloc(4096, GFP_KERNEL);
dma_addr_t dma_addr = dma_map_single(&pdev->dev,
                                     cpu_addr, 4096,
                                     DMA_TO_DEVICE);
// Returns bus address that device should use

// Give dma_addr to device (write to register)
writel(dma_addr, dev->bar + DMA_SRC_REG);

// Device DMA reads from dma_addr
// ...

// After device is done
dma_unmap_single(&pdev->dev, dma_addr, 4096, DMA_TO_DEVICE);

Scatter-gather:
For non-contiguous memory (many pages):

struct scatterlist sg[10];
sg_init_table(sg, 10);

for (i = 0; i < 10; i++) {
    sg_set_page(&sg[i], pages[i], PAGE_SIZE, 0);
}

int nents = dma_map_sg(&pdev->dev, sg, 10, DMA_TO_DEVICE);
// Returns number of DMA segments
// May coalesce adjacent pages

// Device uses PRPs (Physical Region Pages) or SGLs
```

**DMA coherent memory:**

```c
For shared structures (rings, descriptors):

// Allocate memory accessible by both CPU and device
void *cpu_addr;
dma_addr_t dma_addr;

cpu_addr = dma_alloc_coherent(&pdev->dev, 4096,
                               &dma_addr, GFP_KERNEL);
// cpu_addr: Virtual address for CPU
// dma_addr: Bus address for device
// Both point to same physical memory
// Cache coherent (no manual flushing needed)

// CPU writes to cpu_addr
*(u32 *)cpu_addr = 0x12345678;
// Device can immediately read from dma_addr
// No cache flush needed

// Free
dma_free_coherent(&pdev->dev, 4096, cpu_addr, dma_addr);

Use cases:
├── Descriptor rings (NVMe submission/completion queues)
├── Command structures
└── Status buffers
```

**IOMMU (I/O Memory Management Unit):**

```
Without IOMMU:
└── Device DMA address = physical address
    Device can write anywhere in RAM
    Security issue: malicious device can attack kernel

With IOMMU (Intel VT-d, AMD-Vi):
└── Device DMA address != physical address
    IOMMU translates device addresses
    Like MMU for devices

IOMMU page tables:
├── Per-device address space
├── Only pages explicitly mapped are accessible
└── Unmapped access → IOMMU fault

dma_map_single() with IOMMU:
└── Allocates IOMMU page table entry
    Maps device address → physical address
    Returns device address
    
    Device can only access mapped pages
    Cannot access arbitrary memory
    Security enforced in hardware

dma_unmap_single():
└── Removes IOMMU mapping
    Device can no longer access those pages

Benefits:
├── Security: Isolate devices
├── Virtualization: Give VM direct device access
└── Address translation: Device sees contiguous memory
                        even if physical memory fragmented
```

---

# 9. Complete NVMe Initialization Flow

## From Cold Boot to /dev/nvme0n1

**Hardware power-on:**

```
System boots:

PCIe link training (LTSSM - Link Training State Machine):
└── Host and device negotiate
    
    States:
    ├── Detect: Sense device presence
    ├── Polling: Exchange training sequences
    ├── Configuration: Negotiate width and speed
    └── L0: Active state, link ready
    
    Result:
    └── NVMe connected via x4 link
        Speed negotiated: Gen 4 (16 GT/s)
        Bandwidth: 8 GB/s each direction

Training takes ~10-100ms at boot
Hotplug takes ~100-500ms
```

**BIOS/UEFI configuration:**

```
Firmware scans PCIe:

Enumerate all devices:
└── Walk PCIe tree from Root Complex
    Read config space of each device
    
    Find NVMe at Bus 1, Device 0, Function 0:
    └── Read vendor ID: 0x144D (Samsung)
        Read device ID: 0xA80A (980 Pro)
        Read class: 0x010802 (NVMe controller)
        
    Assign BARs:
    └── Device reports need: 16KB (BAR0 64-bit)
        Find free region: 0xFC000000
        Write to BAR0: 0xFC000000
        Write to BAR1: 0x00000000 (high 32 bits)
        
    NVMe registers now at: 0xFC000000 - 0xFC003FFF

Boot from NVMe (if bootable):
└── UEFI driver may initialize NVMe minimally
    Read boot loader from disk
    Start kernel
```

**Linux PCI enumeration:**

```
Kernel boot (early):

arch/x86/pci/common.c:
└── pci_subsys_init()
    
    Scan PCIe hierarchy:
    └── pci_scan_root_bus()
        
        For each bus:
        └── For each device (0-31):
            └── For each function (0-7):
                Read vendor ID
                If not 0xFFFF (present):
                └── Read full config space
                    Allocate struct pci_dev
                    Parse capabilities
                    Add to bus->devices list

Found NVMe at 01:00.0:
└── Allocate pci_dev structure
    Fill in:
    ├── vendor = 0x144D
    ├── device = 0xA80A
    ├── class = 0x010802
    ├── resource[0].start = 0xFC000000
    ├── resource[0].end = 0xFC003FFF
    └── resource[0].flags = IORESOURCE_MEM | IORESOURCE_64BIT
    
    Add to device model:
    └── device_add(&pdev->dev)
        Creates /sys/bus/pci/devices/0000:01:00.0/
        
        Trigger driver matching
```

**nvme driver probe:**

```c
Driver already registered (module_init ran):
└── pci_register_driver(&nvme_driver)

Device matching:
└── nvme_id_table:
    { PCI_DEVICE_CLASS(0x010802, 0xFFFFFF) }
    
    01:00.0 class 0x010802 MATCHES
    
    Call: nvme_probe(pdev, id)

nvme_probe execution:

Step 1: pci_enable_device(pdev)
└── Enables config space access
    Powers up device (D0 state)
    
Step 2: pci_set_master(pdev)
└── Enables bus mastering in Command register
    DMA now possible
    
Step 3: pci_request_mem_regions(pdev, "nvme")
└── Claims BAR ownership
    /proc/iomem shows:
    fc000000-fc003fff : nvme

Step 4: Map BAR
bar_phys = pci_resource_start(pdev, 0) // 0xFC000000
bar_size = pci_resource_len(pdev, 0)   // 16384
dev->bar = ioremap(bar_phys, bar_size)
// Returns virtual: 0xFFFF888100000000

Step 5: DMA configuration
dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64))
// Device supports 64-bit DMA addresses
// No DMA bouncing needed

Step 6: MSI-X setup
nr_vecs = pci_alloc_irq_vectors(pdev, 1, num_cpus, PCI_IRQ_MSIX)
// Returns: 8 (8 CPU system)
// MSI-X enabled in config space
// MSI-X table programmed with APIC addresses

for (i = 0; i < 8; i++):
    irq = pci_irq_vector(pdev, i)
    request_irq(irq, nvme_irq, 0, "nvme", &queues[i])
// 8 handlers registered, one per queue

Step 7: NVMe controller reset
// Read version register
version = readl(dev->bar + NVME_REG_VS)
// Example: 0x00010400 (NVMe 1.4)

// Disable controller
writel(0, dev->bar + NVME_REG_CC)

// Wait for CSTS.RDY to clear
while (readl(dev->bar + NVME_REG_CSTS) & NVME_CSTS_RDY):
    msleep(1)

// Allocate admin queue (DMA coherent)
admin_sq = dma_alloc_coherent(dev, 4096, &admin_sq_dma, GFP_KERNEL)
admin_cq = dma_alloc_coherent(dev, 4096, &admin_cq_dma, GFP_KERNEL)

// Tell controller about admin queues
writeq(admin_sq_dma, dev->bar + NVME_REG_ASQ)
writeq(admin_cq_dma, dev->bar + NVME_REG_ACQ)
writel(63, dev->bar + NVME_REG_AQA)  // 64 entries each

// Enable controller
writel(NVME_CC_ENABLE | NVME_CC_CSS_NVM | ...,
       dev->bar + NVME_REG_CC)

// Wait for ready
while (!(readl(dev->bar + NVME_REG_CSTS) & NVME_CSTS_RDY)):
    msleep(1)
// Controller ready!

Step 8: Identify controller
// Build Identify command
cmd.opcode = nvme_admin_identify
cmd.nsid = 0
cmd.cns = NVME_ID_CNS_CTRL
cmd.dptr.prp1 = dma_addr_of_id_buffer

// Submit to admin queue
admin_sq[sq_tail] = cmd
sq_tail++

// Ring doorbell
writel(sq_tail, dev->bar + NVME_SQ0TDBL)

// NVMe controller:
// - Reads command via DMA
// - Executes identify
// - DMA writes result to id_buffer
// - DMA writes completion to admin_cq
// - Raises MSI-X interrupt

// Handler wakes us
wait_for_completion(&nvme_admin_completion)

// Parse identify data
id->mn = "Samsung SSD 980 PRO 1TB      "
id->sn = "S5EVNF0R123456A"
id->nn = 1  // 1 namespace

Step 9: Create I/O queues
for (i = 1; i <= 8; i++):
    // Create I/O completion queue
    nvme_create_cq(dev, i, cq_dma[i], 1024, i)
    
    // Create I/O submission queue
    nvme_create_sq(dev, i, sq_dma[i], 1024, i)
// 8 I/O queue pairs created

Step 10: Identify namespace
nvme_identify_ns(dev, 1, &ns_id)
ns_id.nsze = 2000409264  // Total sectors
ns_id.lbaf[0].ds = 9     // 512-byte sectors

Step 11: Register with block layer
disk = blk_mq_alloc_disk(&nvme_tagset, ns)
sprintf(disk->disk_name, "nvme%dn%d", 0, 1)
// disk_name = "nvme0n1"

set_capacity(disk, ns_id.nsze)
disk->fops = &nvme_bdev_ops

device_add_disk(&ctrl->device, disk, nvme_ns_attr_groups)
// /dev/nvme0n1 created!
// /sys/block/nvme0n1/ created!

Probe complete!
```

**Userspace sees device:**

```
udev receives uevent:
└── ACTION=add
    DEVTYPE=disk
    DEVNAME=/dev/nvme0n1
    
Rules execute:
└── Create symlinks
    Update /dev/disk/by-id/
    Update /dev/disk/by-path/
    
Partition detection:
└── Kernel reads partition table
    Creates /dev/nvme0n1p1, /dev/nvme0n1p2
    
Filesystem mount:
└── systemd mounts root filesystem
    or
    User sees drive in file manager
```

---

# 10. Physical Reality

## Silicon and Signaling

**PCIe physical layer:**

```
Differential signaling:

Each lane = 2 differential pairs:
├── TX+ / TX- (transmit from host to device)
└── RX+ / RX- (receive from device to host)

Differential signaling:
└── Measure voltage difference: V(+) - V(-)
    Immune to common-mode noise
    Both wires pick up same noise
    Difference cancels it out

Typical voltages:
├── High: +800mV on +, -800mV on -
├── Low: -800mV on +, +800mV on -
└── Swing: ~1.6V differential

Signal rates:
├── Gen 3: 8 GT/s per lane
├── Gen 4: 16 GT/s per lane
└── Gen 5: 32 GT/s per lane
    
    8 GT/s = 8 billion transitions per second
    = 0.125 nanosecond per bit
    = Requires precise timing
```

**Physical connectors:**

```
PCIe slot form factors:

x1 slot:
├── Length: 25mm
├── Pins: 36 (18 per side)
└── 1 lane (TX pair + RX pair)

x4 slot:
├── Length: 39mm
├── Pins: 64
└── 4 lanes

x8 slot:
├── Length: 56mm
├── Pins: 98
└── 8 lanes

x16 slot (graphics card):
├── Length: 89mm
├── Pins: 164
├── 16 lanes
└── Up to 75W power from slot

M.2 slot (NVMe):
├── Key M (M.2 2280 common)
├── 75 pins
├── Up to x4 lanes
├── 3.3V power (up to 10W)
└── Compact: 22mm × 80mm

External power for graphics:
├── 6-pin PCIe: +12V × 3 = 75W
├── 8-pin PCIe: +12V × 3 + sense × 2 = 150W
└── High-end cards: 2-3 connectors = 300-450W
```

**Link training (LTSSM):**

```
Link Training State Machine states:

Detect:
└── Host checks for device presence
    Measures impedance on TX lines
    Device present? Proceed

Polling:
└── Both sides send training sequences
    TS1 (Training Sequence 1)
    TS2 (Training Sequence 2)
    Establish bit and symbol lock
    Synchronize clocks

Configuration:
└── Negotiate link width
    Request x16, fall back to x8, x4, x1 if needed
    Lane reversal if cable reversed
    
Recovery:
└── Speed negotiation
    Try Gen 4 (16 GT/s)
    If errors: fall back to Gen 3
    If errors: fall back to Gen 2
    Until stable link achieved

L0 (Active):
└── Normal operation
    Data transfers
    TLPs (Transaction Layer Packets) flow
    
Training time: ~10-100ms
Re-training: ~1-10ms
```

**Transaction Layer Packets:**

```
What actually travels on PCIe wires:

TLP types:
├── Memory Read Request (MRd)
├── Memory Write Request (MWr)
├── Completion with Data (CplD)
├── Completion without Data (Cpl)
└── Message (Msg) - includes MSI

TLP structure (Memory Write):
┌─────────────────────────────────┐
│ Header (12-16 bytes)            │
│  ├── Format/Type (MWr)          │
│  ├── Length (in DW)             │
│  ├── Requester ID (BDF)         │
│  ├── Tag                        │
│  ├── Address (32 or 64-bit)    │
│  └── Last DW Byte Enable        │
├─────────────────────────────────┤
│ Data Payload (0-4096 bytes)    │
│  ← Actual data being written   │
├─────────────────────────────────┤
│ ECRC (4 bytes, optional)        │
│  ← End-to-end CRC              │
└─────────────────────────────────┘

MSI interrupt = Memory Write TLP:
└── Address: 0xFEExxxxx (APIC range)
    Data: Interrupt vector
    No response needed
    APIC sees write as interrupt

Example flow:
CPU reads NVMe register:
└── MRd TLP: Address=0xFC000000, Length=1 DW
    Device responds:
    CplD TLP: Data=0x00010400 (version register)

DMA write:
└── Device initiates MWr TLP
    Address=0x12340000 (system RAM)
    Data=4KB of read data
    No response (posted write)
```

---

# 11. Connections to Other Topics

## PCIe as Foundation

**Connection to ioremap:**

```
Every PCIe driver uses ioremap:

pci_resource_start(pdev, 0) → 0xFC000000 (physical)
ioremap(0xFC000000, size) → 0xFFFF888100000000 (virtual)

BAR access requires ioremap:
└── Physical address not directly accessible
    Must map through page tables
    Creates non-cacheable mapping (write-through)
    
    readl(bar_virt + offset) → PCIe read
    writel(value, bar_virt + offset) → PCIe write

TOPIC 12 (ioremap) IS HOW PCIe BARs WORK
```

**Connection to interrupts:**

```
MSI-X is core interrupt mechanism:

pci_alloc_irq_vectors() → Allocates interrupt vectors
request_irq() → Registers handlers (TOPIC 3)
MSI-X write → APIC delivery (TOPIC 6)

Every modern device interrupt flow:
└── Device MSI-X write
    APIC receives (TOPIC 6)
    IDT lookup (TOPIC 3)
    Handler runs
    
TOPICS 3 + 6 + 21 = Complete interrupt picture
```

**Connection to DMA and memory:**

```
DMA uses memory subsystem:

dma_alloc_coherent():
└── Uses buddy allocator (TOPIC 10)
    Physically contiguous pages
    From ZONE_DMA or ZONE_DMA32 if needed
    
dma_map_single():
└── May use SWIOTLB bounce buffers (TOPIC 10)
    IOMMU page table setup
    Returns bus address for device
    
Memory allocated in TOPIC 10
Mapped for DMA in TOPIC 21
```

**Connection to block layer:**

```
NVMe is PCIe device:

nvme_probe():
└── PCIe discovery and BAR mapping (TOPIC 21)
    Creates block device (TOPIC 18)
    
Data path:
└── bio → request (TOPIC 18)
    NVMe driver: build command
    Write to doorbell (PCIe MMIO from TOPIC 21)
    DMA transfer (TOPIC 21)
    MSI-X interrupt (TOPIC 21)
    Completion processed (TOPIC 18)

TOPIC 18 (block) built on TOPIC 21 (PCIe)
```

**Connection to network drivers:**

```
NICs are PCIe devices:

igb_probe():
└── pci_enable_device() (TOPIC 21)
    pci_set_master() (TOPIC 21)
    ioremap BAR (TOPIC 12 + TOPIC 21)
    pci_alloc_irq_vectors() (TOPIC 21)
    
    Create net_device (TOPIC 19)

Packet flow:
└── sk_buff → driver (TOPIC 19)
    DMA mapping (TOPIC 21)
    PCIe write doorbell (TOPIC 21)
    NIC DMA reads data (TOPIC 21)
    Transmit complete MSI-X (TOPIC 21)

TOPIC 19 (network) uses TOPIC 21 (PCIe) foundation
```

**Connection to USB:**

```
xHCI controller is PCIe device:

xhci_pci_probe():
└── ALL PCIe setup (TOPIC 21):
    pci_enable_device()
    pci_set_master()
    ioremap()
    pci_alloc_irq_vectors()
    
    Then xHCI-specific init (TOPIC 20)

USB data:
└── URB → xHCI driver (TOPIC 20)
    xHCI writes to ring (via ioremap from TOPIC 21)
    xHCI DMA (bus mastering from TOPIC 21)
    xHCI MSI-X (TOPIC 21)

TOPIC 20 (USB) entirely dependent on TOPIC 21
```

**Connection to device model:**

```
PCIe uses device model:

struct pci_dev contains struct device:
└── /sys/bus/pci/devices/0000:01:00.0/
    /sys/bus/pci/drivers/nvme/
    
pci_register_driver():
└── Uses driver_register() (TOPIC 15)
    Bus matching via pci_bus_type
    probe() called on match
    
Hotplug events:
└── PCIe hotplug → uevent (TOPIC 15)
    udev receives notification
    Module loads automatically

TOPIC 15 (device model) underlies TOPIC 21
```

**Connection to boot:**

```
Boot sequence uses PCIe:

BIOS/UEFI:
└── Enumerates PCIe (TOPIC 13 + TOPIC 21)
    Assigns BARs
    Boots from NVMe (PCIe device)
    
Linux:
└── pci_subsys_init() (TOPIC 21)
    Discovers all devices
    Drivers probe (TOPIC 21)
    Root filesystem on NVMe (TOPIC 18 + TOPIC 21)

TOPIC 13 (boot) → TOPIC 21 (PCIe enum) → all drivers
```

---

# Summary

## What You've Mastered

**You now understand PCIe:**

```
What it is: High-speed serial bus connecting peripherals to CPU
Why needed: Unified addressing, configuration, DMA, interrupts
Architecture: Point-to-point links, BDF addressing, hierarchy
Key mechanism: BARs for registers, MSI-X for interrupts, DMA for data
Driver pattern: probe → enable → map → configure → register
```

---

## Key Takeaways

**Point-to-point architecture:**

```
Not a shared bus:
└── Each device has dedicated link
    All communicate simultaneously
    Bandwidth scales with lanes (x1, x4, x16)
    Speeds scale with generation (Gen 1-6)

Example bandwidths:
├── NVMe (x4 Gen 4): 8 GB/s
├── Graphics (x16 Gen 5): 64 GB/s
└── Network (x8 Gen 3): 8 GB/s
```

**BDF addressing:**

```
Bus:Device.Function (BB:DD.F):
└── Unique identifier for every device
    00:02.0, 01:00.0, 02:00.0
    
4KB configuration space:
└── Vendor/Device ID
    Class code
    BARs (registers location)
    Capabilities (MSI-X, Power, AER)
```

**BARs (Base Address Registers):**

```
Device exposes memory regions:
└── BIOS assigns physical addresses at boot
    Driver maps with ioremap()
    Driver reads/writes device registers
    
Example:
└── BAR0 at 0xFC000000
    ioremap() → virtual address
    readl(virt + offset) → device register read
```

**Driver lifecycle:**

```
Standard pattern:
└── pci_register_driver()
    Device matched by id_table
    probe() called:
        pci_enable_device()
        pci_set_master() (enable DMA)
        pci_request_mem_regions()
        ioremap()
        pci_alloc_irq_vectors()
        Hardware initialization
        Subsystem registration
```

**MSI-X interrupts:**

```
Memory-write based:
└── No physical interrupt wires
    Device writes to APIC address (0xFEExxxxx)
    Write interpreted as interrupt
    
Benefits:
├── Up to 2048 vectors per device
├── Per-vector routing to specific CPU
└── No sharing conflicts

NVMe example:
└── 8 MSI-X vectors (one per I/O queue)
    Queue 0 → CPU 0
    Queue 1 → CPU 1
    True parallel completion
```

**DMA bus mastering:**

```
Device-initiated memory access:
└── pci_set_master() enables
    Device can read/write host RAM
    No CPU involvement in data transfer
    
dma_map_single():
└── Maps memory for device access
    IOMMU provides security
    Returns bus address for device
    
dma_alloc_coherent():
└── Shared memory (CPU and device)
    Used for descriptor rings
    Cache coherent, no manual flushing
```

**Complete NVMe flow:**

```
Cold boot to /dev/nvme0n1:
└── PCIe link training (10-100ms)
    BIOS assigns BAR (0xFC000000)
    Linux enumerates device (01:00.0)
    nvme_probe():
        Enable device
        Enable DMA
        Map BAR via ioremap
        Setup MSI-X (8 vectors)
        Reset controller
        Create admin queue
        Identify controller
        Create I/O queues
        Register with block layer
    /dev/nvme0n1 appears!
```

**Physical reality:**

```
Differential signaling:
└── TX+/TX-, RX+/RX- pairs per lane
    8-64 GT/s per lane
    Noise immunity
    
LTSSM link training:
└── Detect → Polling → Config → L0
    Negotiates width and speed
    10-100ms at boot
    
TLPs (Transaction Layer Packets):
└── Memory Read, Memory Write, Completion
    MSI = Memory Write to 0xFEExxxxx
    Actual data format on wire
```

---

## The Big Picture

**PCIe is the foundation:**

```
Everything connects via PCIe:
├── Storage: NVMe (x4), AHCI
├── Network: 1GbE, 10GbE, 100GbE
├── USB: xHCI controller
├── Graphics: Integrated or discrete GPU
├── Audio: HDA controller
└── Thunderbolt: PCIe over cable

Unified infrastructure:
├── Discovery: BDF addressing
├── Configuration: 4KB config space
├── Register access: BARs + ioremap
├── Interrupts: MSI-X to APIC
├── Data transfer: DMA bus mastering
└── All drivers follow same pattern

Benefits:
├── Code reuse (PCI core 50K lines)
├── Driver simplicity (50 line init)
├── Security (IOMMU isolation)
├── Performance (parallel, dedicated links)
└── Hotplug support (automatic discovery)
```

---

## You're Ready!

With this knowledge, you can:
- Understand device discovery and addressing
- Write PCIe device drivers
- Configure BARs and interrupts
- Use DMA for efficient data transfer
- Debug PCIe initialization issues
- Understand lspci and sysfs output
- See how all drivers connect to hardware

> **PCIe is the universal peripheral interconnect - you now know how Linux discovers devices, assigns resources, and provides the foundation that makes NVMe, network cards, USB controllers, and graphics cards all work through one unified subsystem.**

---

**The backbone connecting everything.**
