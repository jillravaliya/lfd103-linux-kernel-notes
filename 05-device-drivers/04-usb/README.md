# From Plug to Device - The USB Subsystem

> Ever wondered what happens in that 2-second pause after plugging in a USB drive? How your keyboard connects through USB but appears as /dev/input/event0? How the same USB port handles keyboards, storage, webcams, and network adapters?

**You're about to find out!**

---

## What's This About?

This is **drivers/usb/** - the subsystem that manages Universal Serial Bus devices and provides unified access to peripherals!

Here's where it fits in the driver ecosystem:

```
drivers/
04-device-drivers/
├── 00-foundation/      ← Device registration in kernel
├── 01-input/           ← Input events to userspace
├── 02-block/           ← Storage read and write
├── 03-net/             ← Packet send and receive
├── 04-usb/             ← USB device communication
│   ├── URB             ← USB Request Block (like bio/sk_buff)
│   ├── Enumeration     ← Plug-in device discovery
│   ├── USB classes     ← HID, Mass Storage, CDC, Audio
│   ├── Host controllers ← xHCI, EHCI (PCIe devices)
│   └── Hub management  ← Device tree topology
├── 05-pci/             ← PCIe device discovery
├── 06-char/            ← Character device data flow
└── 07-gpu/             ← Display and rendering pipeline
```

**Location in code:**

```
drivers/usb/
├── core/
│   ├── hub.c           ← Hub detection, enumeration
│   ├── urb.c           ← URB (USB Request Block)
│   ├── hcd.c           ← Host Controller Driver interface
│   ├── driver.c        ← USB driver model
│   └── message.c       ← USB control transfers
├── host/
│   ├── xhci.c          ← xHCI (USB 3.x host controller)
│   ├── ehci-hcd.c      ← EHCI (USB 2.0 host controller)
│   └── uhci-hcd.c      ← UHCI (USB 1.1 host controller)
└── storage/
    ├── usb.c           ← USB Mass Storage
    └── transport.c     ← SCSI-over-USB

drivers/hid/usbhid/
└── hid-core.c          ← USB HID (keyboards, mice)

drivers/usb/serial/     ← USB serial adapters
drivers/net/usb/        ← USB network adapters
sound/usb/              ← USB audio devices
```

**This covers:**
- USB protocol fundamentals (transfer types, descriptors)
- URB lifecycle (like bio for block, sk_buff for net)
- Complete enumeration flow (plug-in to device ready)
- USB classes (HID, Mass Storage, CDC, Audio)
- Host controller architecture (xHCI, EHCI)
- Hotplug and disconnect handling

---

# The Fundamental Problem

**Physical Reality:**

```
USB devices everywhere:

Input devices:
├── Keyboards (104 keys, USB HID)
├── Mice (optical, wireless)
└── Game controllers (Xbox, PlayStation)

Storage devices:
├── Flash drives (USB Mass Storage)
├── External hard drives (SATA over USB)
└── Card readers (SD, microSD)

Audio/Video:
├── Webcams (USB Video Class)
├── Microphones (USB Audio Class)
└── Headsets (Audio + HID controls)

Communication:
├── WiFi adapters (USB network)
├── Bluetooth dongles (USB HID + vendor)
└── USB-to-serial (CDC ACM)

All different device types
All different manufacturers
All need to work through same USB port
```

**The problem:**

```
Without unified USB layer:

Every driver reinvents USB:
├── Keyboard driver must know USB protocol
├── Storage driver must enumerate device
├── Webcam driver must manage host controller
└── Each driver: thousands of lines of USB code

Plug in USB keyboard:
├── Must send USB reset signal
├── Must read device descriptor
├── Must assign USB address
├── Must configure endpoints
└── Every driver does this differently

Result:
├── Massive code duplication
├── Inconsistent behavior
├── Impossible to maintain
└── New device types require USB expertise


Host controller diversity:

Three different USB host controller types:
├── UHCI (Intel, USB 1.1)
├── OHCI (Others, USB 1.1)  
├── EHCI (USB 2.0)
└── xHCI (USB 3.x, handles all speeds)

Each has different:
├── Register layouts
├── DMA mechanisms
├── Interrupt handling
└── Programming interfaces

Keyboard driver must support all four controllers
Impossible to maintain


Speed negotiation nightmare:

USB speeds:
├── Low Speed: 1.5 Mbps (keyboard)
├── Full Speed: 12 Mbps (audio)
├── High Speed: 480 Mbps (storage)
├── SuperSpeed: 5 Gbps (USB 3.0)
└── SuperSpeed+: 10 Gbps (USB 3.1)

Host must detect speed:
├── Read electrical signals
├── Negotiate capabilities
├── Configure timing
└── Each driver reimplements this


Topology management chaos:

USB is a tree:
    Host Controller
    └── Root Hub
        ├── Port 1: External Hub
        │   ├── Port 1: Keyboard
        │   ├── Port 2: Mouse
        │   └── Port 3: Flash Drive
        └── Port 2: Webcam

Who manages this?
├── Hub detection
├── Port power management
├── Device addressing (1-127 per bus)
└── Hot-plug detection

Every driver managing topology = chaos
```

**How do you provide ONE interface for ALL USB devices while handling protocol, enumeration, and host controller diversity?**

---

# Without USB Subsystem

**Imagine no unified USB layer:**

```
Keyboard driver wants to read keys:

Must handle entire USB stack:
├── Initialize USB host controller
│   └── xHCI: 50+ registers, complex ring buffers
│   └── EHCI: different registers, different mechanism
│   └── Must support all controller types
├── Detect device connection
│   └── Monitor D+ line voltage
│   └── Detect speed (LS/FS/HS/SS)
├── Enumerate device
│   └── Send reset (10ms delay required)
│   └── Read device descriptor (partial, then full)
│   └── Assign unique address (1-127)
│   └── Read configuration descriptor
│   └── Set configuration
├── Parse HID report descriptor
│   └── Self-describing format (complex parser)
├── Set up endpoint polling
│   └── Build USB transfer descriptors
│   └── Program DMA addresses
│   └── Configure interrupt timing
└── Handle completion interrupts

Thousands of lines of USB protocol code
Just to read keyboard keys
Duplicated in mouse driver, gamepad driver, etc.


USB storage driver nightmare:

Plug in USB flash drive:
├── Must implement entire enumeration sequence
├── Must discover it's mass storage class
├── Must implement SCSI-over-USB protocol
│   └── CBW (Command Block Wrapper) format
│   └── SCSI commands (READ, WRITE, etc.)
│   └── CSW (Command Status Wrapper) parsing
├── Must handle bulk transfers
│   └── Large data, error correction
│   └── Retry logic on errors
├── Must integrate with block layer
└── Must handle safe removal

Then USB webcam needs different protocol
Then USB network adapter needs different protocol
Every device type: thousands of duplicate lines


No code reuse:

Common operations done by every driver:
├── USB reset sequence (10ms delay, check status)
├── Descriptor reading (device, config, interface, endpoint)
├── Address assignment (must be unique, 1-127)
├── Configuration selection
├── Endpoint setup
└── Host controller interaction

No shared infrastructure
Every driver from scratch
Maintenance nightmare


Hub management impossible:

External USB hub plugged in:
├── Which driver handles it?
├── Hub itself is a USB device
├── But also manages downstream devices
├── Port power control
├── Per-port status monitoring
└── Cascade (hub connected to hub)

Without USB core:
├── Every driver detects hubs differently
├── Topology becomes inconsistent
└── Devices behind hubs don't work


Power management chaos:

USB has complex power rules:
├── Bus-powered: max 500mA (USB 2.0), 900mA (USB 3.0)
├── Self-powered: uses external power
├── Selective suspend: idle devices sleep
├── Remote wakeup: device can wake host
└── Per-port power control

Every driver manages power separately:
├── Inconsistent suspend behavior
├── Battery drain from forgotten devices
└── Wake from keyboard doesn't work
```

**Absolute chaos without USB abstraction!**

---

# The Layered Solution

**Unified USB infrastructure:**

```
Layer 1: USB CORE (drivers/usb/core/)
├── Hub management (detection, enumeration)
├── Device addressing (unique per bus)
├── Descriptor parsing (device, config, interface)
├── URB abstraction (like bio/sk_buff)
└── Power management (suspend, resume)

Provides: Complete USB protocol implementation
Handles: All hardware complexity


Layer 2: HOST CONTROLLER DRIVERS (drivers/usb/host/)
├── xhci.c (USB 3.x + 2.0 + 1.1 all-in-one)
├── ehci-hcd.c (USB 2.0 only)
├── uhci-hcd.c (USB 1.1 Intel)
└── ohci-hcd.c (USB 1.1 Others)

These know: Controller registers, DMA rings, interrupts
Provide: Generic URB submission interface


Layer 3: USB CLASS DRIVERS (various locations)
├── HID (drivers/hid/usbhid/) → input subsystem
├── Mass Storage (drivers/usb/storage/) → block layer
├── CDC ACM (drivers/usb/class/cdc-acm.c) → TTY layer
├── CDC ECM (drivers/net/usb/) → network layer
└── Audio (sound/usb/) → ALSA

These know: What the device does (keyboard, storage, etc.)
Don't know: USB protocol details (handled by core)


Result:
├── Keyboard driver: 500 lines (just HID parsing)
├── Storage driver: 1000 lines (just SCSI-over-USB)
├── Network driver: 800 lines (just packet handling)
└── USB protocol: shared by all (15,000 lines once)

Code reuse achieved
Consistent behavior across all devices
```

---

# 1. USB Protocol Fundamentals

## Transfer Types and Speeds

**USB speeds:**

```
USB 1.0/1.1:
├── Low Speed: 1.5 Mbps
│   └── Keyboards, mice (minimal data)
└── Full Speed: 12 Mbps
    └── Audio devices, older webcams

USB 2.0:
└── High Speed: 480 Mbps
    └── Flash drives, modern webcams

USB 3.0 (SuperSpeed):
└── 5 Gbps
    └── Fast external drives

USB 3.1 (SuperSpeed+):
└── 10 Gbps
    └── High-end storage

USB 3.2 and USB4:
├── 20 Gbps (dual lane)
└── 40 Gbps (Thunderbolt compatible)

Physical wiring:
├── USB 2.0: D+ and D- differential pair (2 wires)
└── USB 3.x: Adds SSTX+/SSTX- and SSRX+/SSRX- pairs (full duplex)
```

**Four transfer types:**

```
1. CONTROL (bidirectional):
   Purpose: Device setup, configuration
   Characteristics:
   ├── Always works (guaranteed bandwidth)
   ├── Endpoint 0 on all devices
   └── Used for enumeration
   
   Example: Reading device descriptor
   8-byte setup packet format:
   ├── bmRequestType (direction, type, recipient)
   ├── bRequest (GET_DESCRIPTOR, SET_ADDRESS, etc.)
   ├── wValue (parameter)
   ├── wIndex (parameter)
   └── wLength (data length)


2. BULK (unidirectional):
   Purpose: Large data transfers
   Characteristics:
   ├── Error correction (CRC, retry)
   ├── No bandwidth guarantee (best effort)
   └── High throughput when bus available
   
   Used for: USB storage, printers
   Example: Read 64KB from flash drive


3. INTERRUPT (unidirectional):
   Purpose: Small, periodic data
   Characteristics:
   ├── Guaranteed poll interval (1ms, 8ms, etc.)
   ├── Low latency (scheduled regularly)
   └── Misleading name - host POLLS device
   
   Used for: Keyboards, mice, gamepads
   Example: Keyboard reports every 8ms
   Note: Device doesn't interrupt host
         Host asks "any keys?" every 8ms


4. ISOCHRONOUS (unidirectional):
   Purpose: Streaming data
   Characteristics:
   ├── Guaranteed bandwidth
   ├── NO error correction (drop bad data)
   └── Time-sensitive (audio/video)
   
   Used for: Webcams, microphones, speakers
   Why no retry: Audio glitch better than delay
   Example: 1 dropped sample = tiny click (acceptable)
            Retransmit = timing off = horrible
```

## Descriptors (Device Self-Description)

**Device descriptor:**

```c
struct usb_device_descriptor {
    __u8  bLength;              // 18 bytes
    __u8  bDescriptorType;      // 0x01 (Device)
    __le16 bcdUSB;              // USB version (0x0200 = USB 2.0)
    __u8  bDeviceClass;         // Class code
    __u8  bDeviceSubClass;      // Subclass
    __u8  bDeviceProtocol;      // Protocol
    __u8  bMaxPacketSize0;      // Max packet size for endpoint 0
    __le16 idVendor;            // Vendor ID (0x8086 = Intel)
    __le16 idProduct;           // Product ID
    __le16 bcdDevice;           // Device version
    __u8  iManufacturer;        // String index
    __u8  iProduct;             // String index
    __u8  iSerialNumber;        // String index
    __u8  bNumConfigurations;   // Number of configurations
};

Example (SanDisk flash drive):
├── idVendor: 0x0781 (SanDisk)
├── idProduct: 0x5581 (Ultra)
├── bDeviceClass: 0x00 (defined per interface)
├── bMaxPacketSize0: 64 bytes
└── bNumConfigurations: 1
```

**Configuration descriptor:**

```c
struct usb_config_descriptor {
    __u8  bLength;              // 9 bytes
    __u8  bDescriptorType;      // 0x02 (Configuration)
    __le16 wTotalLength;        // Total length of all descriptors
    __u8  bNumInterfaces;       // Number of interfaces
    __u8  bConfigurationValue;  // Configuration ID
    __u8  iConfiguration;       // String index
    __u8  bmAttributes;         // Self-powered? Remote wakeup?
    __u8  bMaxPower;            // Max current (× 2mA)
};

Example:
├── bNumInterfaces: 1
├── bmAttributes: 0x80 (bus-powered)
└── bMaxPower: 250 (500mA)
```

**Interface descriptor:**

```c
struct usb_interface_descriptor {
    __u8  bLength;              // 9 bytes
    __u8  bDescriptorType;      // 0x04 (Interface)
    __u8  bInterfaceNumber;     // Interface ID
    __u8  bAlternateSetting;    // Alternate setting
    __u8  bNumEndpoints;        // Number of endpoints
    __u8  bInterfaceClass;      // Class code
    __u8  bInterfaceSubClass;   // Subclass
    __u8  bInterfaceProtocol;   // Protocol
    __u8  iInterface;           // String index
};

Class codes (bInterfaceClass):
├── 0x01: Audio
├── 0x02: CDC (Communications)
├── 0x03: HID (Human Interface Device)
├── 0x08: Mass Storage
├── 0x09: Hub
└── 0xFF: Vendor-specific

Example (USB storage):
├── bInterfaceClass: 0x08 (Mass Storage)
├── bInterfaceSubClass: 0x06 (SCSI)
├── bInterfaceProtocol: 0x50 (Bulk-Only Transport)
└── bNumEndpoints: 2 (Bulk IN, Bulk OUT)
```

**Endpoint descriptor:**

```c
struct usb_endpoint_descriptor {
    __u8  bLength;              // 7 bytes
    __u8  bDescriptorType;      // 0x05 (Endpoint)
    __u8  bEndpointAddress;     // Endpoint address
    __u8  bmAttributes;         // Transfer type
    __le16 wMaxPacketSize;      // Max packet size
    __u8  bInterval;            // Polling interval
};

bEndpointAddress encoding:
├── Bit 7: Direction (0=OUT, 1=IN)
└── Bits 0-3: Endpoint number (0-15)

Examples:
├── 0x81: Endpoint 1, IN direction (device to host)
├── 0x02: Endpoint 2, OUT direction (host to device)
└── 0x00: Endpoint 0 (control, always bidirectional)

bmAttributes (transfer type):
├── 0x00: Control
├── 0x01: Isochronous
├── 0x02: Bulk
└── 0x03: Interrupt
```

---

# 2. The URB Structure

## Universal Request Block

**What is URB?**

```c
struct urb {
    // List management
    struct list_head urb_list;  // In device's URB list
    
    // Device and endpoint
    struct usb_device *dev;     // Which USB device
    struct usb_host_endpoint *ep; // Which endpoint
    unsigned int pipe;          // Encodes: direction + endpoint + type
    
    // Transfer type (determined by pipe)
    // Control, Bulk, Interrupt, or Isochronous
    
    // Data buffer
    void *transfer_buffer;      // CPU-accessible buffer
    dma_addr_t transfer_dma;    // DMA address (if DMA-capable)
    u32 transfer_buffer_length; // Buffer size
    u32 actual_length;          // Bytes actually transferred
    
    // Control transfer specific
    unsigned char *setup_packet; // 8-byte setup packet
    dma_addr_t setup_dma;       // DMA address of setup
    
    // Completion
    usb_complete_t complete;    // Callback function
    void *context;              // Data for callback
    
    // Status
    int status;                 // 0 = success, negative = error
    
    // Interrupt/Isochronous specific
    int interval;               // Polling interval (ms)
    
    // Isochronous specific
    int number_of_packets;      // Number of ISO packets
    struct usb_iso_packet_descriptor iso_frame_desc[];
};

URB is to USB what:
├── bio is to block devices
├── sk_buff is to network devices
└── Fundamental unit of USB I/O
```

**Pipe encoding:**

```
Pipe = 32-bit integer encoding endpoint info

Helper macros:
usb_rcvbulkpipe(dev, endpoint)
    → Receive (IN), Bulk transfer, endpoint N

usb_sndbulkpipe(dev, endpoint)
    → Send (OUT), Bulk transfer, endpoint N

usb_rcvintpipe(dev, endpoint)
    → Receive (IN), Interrupt transfer, endpoint N

usb_sndctrlpipe(dev, 0)
    → Send (OUT), Control transfer, endpoint 0

usb_rcvisocpipe(dev, endpoint)
    → Receive (IN), Isochronous transfer, endpoint N

Example:
pipe = usb_rcvbulkpipe(udev, 1)
    → Read from endpoint 1 using bulk transfer
```

**URB lifecycle:**

```
1. Allocate:
   urb = usb_alloc_urb(0, GFP_KERNEL)

2. Fill:
   usb_fill_bulk_urb(urb, dev, pipe, buffer, len, callback, context)
   OR
   usb_fill_int_urb(urb, dev, pipe, buffer, len, callback, context, interval)
   OR
   Manually fill fields for control/isochronous

3. Submit:
   usb_submit_urb(urb, GFP_KERNEL)
   ├── USB core validates URB
   ├── Passes to host controller driver
   └── Returns immediately (asynchronous)

4. Host controller processes:
   ├── Schedules transfer based on type
   ├── Performs USB transaction on bus
   └── DMA transfers data

5. Completion:
   ├── Host controller signals completion
   ├── URB callback function called
   └── urb->status indicates success/failure

6. Cleanup:
   usb_free_urb(urb)
```

---

# 3. Complete USB Enumeration

## Plug-In to Device Ready

**Physical connection:**

```
User plugs USB flash drive:

Electrical signals:
├── D+ line: 3.3V (high-speed device signal)
├── D- line: floating
└── VBUS: 5V power applied

Hub port detects connection:
├── Voltage change on D+ detected
├── Hub chip registers device presence
└── Hub prepares status change report

Hub is itself a USB device with interrupt endpoint
Host polls hub every interval (typically 256ms)
```

**Step 1: Hub reports device**

```
Host controller polls hub:
└── "Any status changes?"

Hub responds:
└── "Port 3: Device connected!"

Kernel hub driver (hub_irq):
├── Reads hub status via control transfer
├── Port 3: CCS bit set (Current Connect Status)
├── Schedules workqueue: hub_event()
└── Returns from interrupt context quickly

hub_event() runs (workqueue context):
└── Can sleep, can take time
    Processes all hub events
```

**Step 2: Port reset and speed detection**

```
hub_port_reset():
├── Send USB reset signal to port
│   └── SET_PORT_FEATURE(PORT_RESET)
│       D+ and D- both driven low for 10ms
│
├── Wait for reset to complete
│   └── msleep(10)  // USB spec requirement
│
├── Clear reset feature
│   └── CLEAR_PORT_FEATURE(PORT_C_RESET)
│
└── Read port status
    └── GET_PORT_STATUS

Speed detection (from port status):
├── USB_PORT_STAT_HIGH_SPEED set?
│   └── Speed = 480 Mbps (High Speed)
├── USB_PORT_STAT_LOW_SPEED set?
│   └── Speed = 1.5 Mbps (Low Speed)
└── Neither?
    └── Speed = 12 Mbps (Full Speed)

USB 3.0 devices:
└── SuperSpeed negotiation happens earlier
    Speed detected during link training
```

**Step 3: Address assignment**

```
Device initially at address 0:
└── All new devices use address 0 temporarily
    Only one device can be at address 0 simultaneously

Read partial device descriptor:
├── Only first 8 bytes initially
├── Need bMaxPacketSize0 before full communication
└── Control transfer to address 0, endpoint 0

Setup packet (GET_DESCRIPTOR):
bmRequestType: 0x80 (device-to-host, standard, device)
bRequest: 0x06 (GET_DESCRIPTOR)
wValue: 0x0100 (device descriptor type)
wIndex: 0
wLength: 8 (just first 8 bytes)

Device responds:
└── First 8 bytes of device descriptor
    Includes bMaxPacketSize0 = 64 (typical for HS)

Choose unique address:
├── Address range: 1-127
├── Find unused address: 7 (for example)
└── Addresses unique per bus

Assign address (SET_ADDRESS):
bmRequestType: 0x00 (host-to-device, standard, device)
bRequest: 0x05 (SET_ADDRESS)
wValue: 7 (new address)
wIndex: 0
wLength: 0

Device now responds to address 7:
└── All future communication uses address 7
    Must wait 2ms (USB spec: device needs time)
```

**Step 4: Read full descriptors**

```
Read full device descriptor (18 bytes):
└── Now at address 7, endpoint 0

Response includes:
├── idVendor: 0x0781 (SanDisk)
├── idProduct: 0x5581 (Ultra model)
├── bDeviceClass: 0x00 (class defined per interface)
├── iManufacturer: 1 (string descriptor index)
├── iProduct: 2
├── iSerialNumber: 3
└── bNumConfigurations: 1

Read configuration descriptor:
└── GET_DESCRIPTOR(CONFIGURATION)
    Returns ALL descriptors at once:
    ├── Configuration descriptor
    ├── Interface descriptor(s)
    └── Endpoint descriptor(s)

Configuration descriptor shows:
├── bNumInterfaces: 1
├── bmAttributes: 0x80 (bus-powered)
└── bMaxPower: 250 (500mA maximum)

Interface descriptor shows:
├── bInterfaceClass: 0x08 (Mass Storage)
├── bInterfaceSubClass: 0x06 (SCSI transparent)
├── bInterfaceProtocol: 0x50 (Bulk-Only Transport)
└── bNumEndpoints: 2

Endpoint descriptors:
Endpoint 1:
├── bEndpointAddress: 0x81 (IN, endpoint 1)
├── bmAttributes: 0x02 (Bulk)
└── wMaxPacketSize: 512

Endpoint 2:
├── bEndpointAddress: 0x02 (OUT, endpoint 2)
├── bmAttributes: 0x02 (Bulk)
└── wMaxPacketSize: 512

Read string descriptors:
├── String 1: "SanDisk"
├── String 2: "Ultra"
└── String 3: "AA00BB11CC22DD33" (serial number)
```

**Step 5: Set configuration**

```
Activate configuration:
└── SET_CONFIGURATION(1)

bmRequestType: 0x00
bRequest: 0x09 (SET_CONFIGURATION)
wValue: 1 (configuration number)
wIndex: 0
wLength: 0

Device now fully configured:
├── Endpoints activated
├── Interface ready for use
└── Device state: USB_STATE_CONFIGURED
```

**Step 6: Device model registration**

```
usb_new_device():
├── device_add(&udev->dev)
│   └── Adds to device model hierarchy
│
├── Send uevent to userspace
│   └── udev daemon receives notification
│
└── For each interface:
    ├── Create interface device
    ├── device_add(&intf->dev)
    └── device_attach(&intf->dev)
        Triggers driver matching

Driver matching:
├── Iterate registered USB drivers
├── Check id_table for each driver
└── If match: call driver->probe()
```

**Step 7: Driver probe**

```
usb_storage driver matches:
└── id_table: USB_INTERFACE_INFO(
        USB_CLASS_MASS_STORAGE,
        US_SC_SCSI,
        US_PR_BULK)
    MATCH!

usb_storage_probe(intf, id):
├── Allocate driver state
│   └── struct us_data *us
│
├── Save endpoint pipes
│   ├── us->send_bulk_pipe = usb_sndbulkpipe(dev, 2)
│   └── us->recv_bulk_pipe = usb_rcvbulkpipe(dev, 1)
│
├── Register as SCSI host
│   └── scsi_add_host(us->host, dev)
│
└── Scan for LUNs
    └── scsi_scan_host(us->host)

SCSI layer probes device:
├── Sends INQUIRY command
├── Sends READ CAPACITY
└── Creates block device

Block layer creates nodes:
├── /dev/sdb (whole device)
└── /dev/sdb1 (partition 1)

Total time: 1-3 seconds from plug to ready
```

**Step 8: Userspace reaction**

```
udev receives uevent:
└── ACTION=add
    DEVTYPE=usb_device
    SUBSYSTEM=block
    DEVNAME=/dev/sdb1

udev rules execute:
└── /etc/udev/rules.d/80-udisks2.rules
    Triggers udisks2 daemon

udisks2:
├── Detects new block device
├── Reads partition table
├── Reads filesystem type (FAT32, ext4, NTFS)
├── Mounts to /media/username/DriveName/
└── Notifies desktop environment

Desktop notification:
└── "USB Drive inserted" popup
    File manager window opens
```

---

# 4. USB Classes

## Standardized Device Types

**Mass Storage (Class 0x08):**

```
USB storage protocol:
└── SCSI commands wrapped in USB bulk transfers

Command Block Wrapper (CBW):
struct bulk_cb_wrap {
    __le32 Signature;        // 0x43425355 "USBC"
    __u32  Tag;              // Command ID
    __le32 DataTransferLength; // Expected data
    __u8   Flags;            // Direction (IN/OUT)
    __u8   Lun;              // Logical Unit Number
    __u8   Length;           // CB length (1-16)
    __u8   CB[16];           // Command Block (SCSI command)
};

Example (READ):
1. Host sends CBW:
   ├── Signature: 0x43425355
   ├── Tag: 42
   ├── DataTransferLength: 4096
   ├── Flags: 0x80 (IN direction)
   └── CB: [0x28, 0x00, 0x00, 0x00, 0x10, 0x00, 0x00, 0x00, 0x08, 0x00]
           READ(10) command: LBA 16, count 8 blocks

2. Device sends data:
   └── 4096 bytes via bulk IN endpoint

3. Host receives CSW:
   └── Command Status Wrapper

Command Status Wrapper (CSW):
struct bulk_cs_wrap {
    __le32 Signature;        // 0x53425355 "USBS"
    __u32  Tag;              // Matches CBW Tag
    __le32 Residue;          // Difference (expected - actual)
    __u8   Status;           // 0=success, 1=fail, 2=phase error
};

Integration:
USB storage driver → SCSI mid-layer → Block layer
└── Appears as /dev/sdb like any SCSI disk
```

**HID (Human Interface Device, Class 0x03):**

```
HID devices: Keyboards, mice, game controllers, touchscreens

HID Report Descriptor:
└── Self-describing format
    Tells kernel what data means

Keyboard example:
Report descriptor declares:
├── Usage Page: Generic Desktop
├── Usage: Keyboard
├── Input Report Format:
│   ├── Byte 0: Modifier keys (8 bits)
│   │   └── Ctrl, Shift, Alt, GUI (Windows key)
│   ├── Byte 1: Reserved
│   └── Bytes 2-7: Key codes (up to 6 simultaneous)

Keypress report:
[0x00, 0x00, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00]
└── Key 0x04 = 'A' key pressed

Mouse example:
Report format:
├── Byte 0: Buttons (bits)
├── Byte 1: X movement (signed delta)
├── Byte 2: Y movement (signed delta)
└── Byte 3: Wheel (signed delta)

Report:
[0x01, 0x05, 0xFF, 0x00]
└── Left button pressed, moved right 5, up 1

HID driver:
├── Parses report descriptor
├── Maps to Linux input events
└── Feeds input subsystem
    HID usage 0x04 → KEY_A
    Button bit 0 → BTN_LEFT
    X delta → REL_X
```

**CDC (Communications Device Class, Class 0x02):**

```
CDC ACM (Abstract Control Model):
└── USB to serial adapter

Creates: /dev/ttyACM0
Used by: Arduino, modems, debug consoles

Data format:
├── Control endpoint: Line settings (baud, parity)
├── Bulk IN: Receive data from device
└── Bulk OUT: Send data to device

CDC ECM/NCM (Ethernet Control Model):
└── USB network adapter

Creates: eth1 (or similar)
Used by: Phone tethering, USB Ethernet adapters

Integration:
└── Creates net_device
    Same as any Ethernet adapter
    sk_buff → URB → USB device

iPhone tethering:
├── Presents CDC NCM interface
├── Linux cdc_ncm driver creates net_device
├── DHCP assigns IP address
└── Packets flow: TCP/IP → sk_buff → URB → phone
```

**Audio (Class 0x01):**

```
USB audio uses isochronous transfers:
├── Guaranteed bandwidth
├── No error correction
└── Constant data rate

USB headset:
├── Microphone: Isochronous IN endpoint
└── Headphones: Isochronous OUT endpoint

Why isochronous:
├── Audio is time-sensitive
├── 1 dropped sample = tiny click (acceptable)
└── Delayed retransmit = audio glitch (unacceptable)

Packet rate:
└── 8 kHz sample rate
    1 packet per 1ms
    8 samples per packet

Integration:
└── USB audio driver → ALSA
    Appears as sound card
    aplay/arecord work normally
```

---

# 5. Host Controller Architecture

## xHCI (Extensible Host Controller Interface)

**What is xHCI:**

```
Modern USB host controller:
└── Handles ALL USB speeds in one controller
    ├── USB 3.2 (SuperSpeed+): 10 Gbps
    ├── USB 3.0 (SuperSpeed): 5 Gbps
    ├── USB 2.0 (High Speed): 480 Mbps
    └── USB 1.1 (Full/Low Speed): 12 Mbps / 1.5 Mbps

xHCI is a PCIe device:
└── Uses PCIe for host communication
    DMA for data transfer
    MSI-X for interrupts
```

**xHCI ring architecture:**

```
Three types of rings (like NVMe):

1. Command Ring (host → controller):
   └── Commands from driver to hardware
       Address Device, Configure Endpoint, etc.

2. Event Ring (controller → host):
   └── Events from hardware to driver
       Transfer Complete, Port Status Change, etc.

3. Transfer Rings (per endpoint, per device):
   └── Actual USB transfers
       One ring per endpoint on each device

Ring buffer structure:
┌────────────────────────────┐
│ TRB 0  (Transfer Request   │
│ TRB 1   Block)             │
│ TRB 2                      │
│ TRB 3  ← Enqueue pointer   │
│ TRB 4                      │
│ ...                        │
│ TRB N  ← Dequeue pointer   │
└────────────────────────────┘

Driver writes TRBs, advances enqueue
Controller reads TRBs, advances dequeue
```

**Transfer Request Block (TRB):**

```c
struct xhci_generic_trb {
    __le32 field[4];  // 16 bytes total
};

Normal TRB (for data transfer):
├── Field 0: Data buffer pointer (low 32 bits)
├── Field 1: Data buffer pointer (high 32 bits)
├── Field 2: Transfer length, TD size
└── Field 3: Flags (cycle bit, interrupt, etc.)

Event TRB (completion):
├── Field 0: Transfer pointer or command pointer
├── Field 1: Transfer status
├── Field 2: Completion code, slot ID
└── Field 3: Type, cycle bit

Command TRB:
└── Different format per command type
    Address Device, Configure Endpoint, etc.
```

**xHCI workflow:**

```
URB submission:
1. Driver receives URB from USB core
2. Translates URB to xHCI TRBs
3. Writes TRBs to transfer ring
4. Rings doorbell register
5. Returns (asynchronous)

xHCI processing:
1. Sees doorbell ring
2. Reads TRBs from transfer ring
3. Performs USB transaction on bus
4. Writes event TRB to event ring
5. Raises MSI-X interrupt

Completion:
1. Driver receives interrupt
2. Reads event ring
3. Finds completed transfer
4. Updates URB status
5. Calls URB completion callback
```

**EHCI (older USB 2.0 controller):**

```
EHCI characteristics:
├── USB 2.0 High Speed only (480 Mbps)
├── Requires companion controller for USB 1.1
│   └── UHCI or OHCI handles Full/Low Speed
└── More complex than xHCI

Queue heads and transfer descriptors:
└── Different structure than xHCI rings
    Linked lists of descriptors

Modern systems:
└── xHCI replaces EHCI + UHCI/OHCI
    Single controller for all speeds
```

---

# 6. URB Lifecycle Example

## Keyboard Polling

**Driver initialization:**

```c
usbhid_probe(struct usb_interface *intf, const struct usb_device_id *id)
{
    // Allocate URB
    urb = usb_alloc_urb(0, GFP_KERNEL);
    
    // Allocate DMA-coherent buffer
    buf = usb_alloc_coherent(dev, 8, GFP_ATOMIC, &buf_dma);
    
    // Fill interrupt URB
    usb_fill_int_urb(
        urb,
        dev,
        usb_rcvintpipe(dev, endpoint->bEndpointAddress),
        buf,                    // Data buffer
        8,                      // Buffer size
        usbhid_irq,             // Completion callback
        hid,                    // Context
        endpoint->bInterval     // Poll interval (8ms)
    );
    
    urb->transfer_dma = buf_dma;
    urb->transfer_flags |= URB_NO_TRANSFER_DMA_MAP;
    
    // Submit URB
    usb_submit_urb(urb, GFP_KERNEL);
}
```

**Host controller scheduling:**

```
xHCI schedules periodic polling:
└── Every 8ms (bInterval from endpoint descriptor)

Time = 0ms:
└── Host: "Keyboard, send report on IN endpoint 1"
    Keyboard: No keys pressed
    Response: Zero-length packet (ZLP)

Time = 8ms:
└── Host polls again
    Keyboard: Still no keys
    Response: ZLP

Time = 16ms:
└── User presses 'A' key
    Host polls
    Keyboard: "Key pressed!"
    Response: [0x00, 0x00, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00]
```

**URB completion:**

```c
void usbhid_irq(struct urb *urb)
{
    // Check status
    if (urb->status != 0) {
        // Error handling
        if (urb->status != -ENOENT &&
            urb->status != -ECONNRESET &&
            urb->status != -ESHUTDOWN) {
            // Unexpected error, log it
        }
        goto resubmit;
    }
    
    // Check if we received data
    if (urb->actual_length > 0) {
        // Process HID report
        hid_input_report(
            hid,
            HID_INPUT_REPORT,
            urb->transfer_buffer,
            urb->actual_length,
            1  // Interrupt context
        );
        
        // This feeds into input subsystem:
        // input_report_key(KEY_A, 1)
        // input_sync()
    }
    
resubmit:
    // Re-submit for continued polling
    usb_submit_urb(urb, GFP_ATOMIC);  // In interrupt context
}
```

**Data flow:**

```
USB keyboard keypress → application:

1. Physical: Key pressed
2. Keyboard: Builds HID report [0x00, 0x00, 0x04, ...]
3. USB: Host polls, keyboard responds
4. xHCI: DMA writes report to buffer
5. xHCI: Raises interrupt
6. Driver: usbhid_irq() called
7. HID: Parses report (0x04 = 'A' key)
8. Input: input_report_key(KEY_A, 1)
9. Input: input_sync()
10. Evdev: Writes to /dev/input/event0 ring buffer
11. X server: Reads from /dev/input/event0
12. Application: Receives KeyPress event
13. Terminal: Displays 'a' character

Total time: ~10-20ms (8ms poll interval dominates)
```

---

# 7. Hotplug and Disconnect

## Safe Removal

**Physical disconnect:**

```
User removes USB drive:

Electrical change:
├── D+ line voltage drops
├── Hub detects disconnect
└── Hub port status changes

Hub reports (via interrupt endpoint):
└── "Port 3: Connect Status Change"

hub_event() workqueue:
└── Reads port status
    CCS bit clear (device not connected)
```

**Disconnect sequence:**

```c
usb_disconnect(struct usb_device *udev)
{
    // For each interface on device
    for (i = 0; i < udev->actconfig->desc.bNumInterfaces; i++) {
        struct usb_interface *intf = udev->actconfig->interface[i];
        
        // Disconnect driver
        if (intf->dev.driver) {
            struct usb_driver *driver = to_usb_driver(intf->dev.driver);
            
            // Call driver's disconnect
            driver->disconnect(intf);
        }
    }
    
    // Release device resources
    usb_disable_device(udev, 0);
    
    // Remove from device model
    device_del(&udev->dev);
}
```

**USB storage disconnect:**

```c
usb_storage_disconnect(struct usb_interface *intf)
{
    struct us_data *us = usb_get_intfdata(intf);
    
    // Kill all pending URBs
    usb_kill_urb(us->current_urb);
    
    // Remove SCSI host
    scsi_remove_host(us->host);
    
    // This cascades to block layer:
    // - /dev/sdb removed
    // - Partitions gone
    // - Mounted filesystems unmounted
    
    // Free resources
    scsi_host_put(us->host);
}
```

**Why safe eject matters:**

```
USB drives have write cache:

Normal write:
├── Application: write()
├── Kernel: Data written to page cache
├── USB storage: Data queued
├── Device: Data in internal buffer
└── NAND flash: Not yet written!

Pull device out:
└── Data in buffer lost
    Filesystem corruption
    Files incomplete

Safe eject:
├── Kernel sends SYNCHRONIZE CACHE command
├── Device flushes buffer to NAND
├── Wait for completion
└── Now safe to remove

Modern alternatives:
├── Mount with 'sync' option (slower)
├── O_SYNC flag on critical files
└── Disable write cache on device
```

---

# 8. Power Management

## USB Power Rules

**Power delivery:**

```
USB power specifications:

USB 2.0:
├── VBUS: 5V
├── Max per port: 500mA (2.5W)
└── Low power mode: 100mA

USB 3.0:
├── VBUS: 5V
├── Max per port: 900mA (4.5W)
└── Low power mode: 150mA

USB Power Delivery (USB-PD):
├── Up to 100W (20V @ 5A)
├── Negotiated voltage: 5V, 9V, 12V, 15V, 20V
└── Used for: Laptop charging, monitors, docks

Configuration descriptor declares:
└── bMaxPower: Units of 2mA
    bMaxPower = 250 → 500mA
    
Bus-powered device:
└── Gets power from USB VBUS
    bmAttributes bit 6 = 0

Self-powered device:
└── External power supply
    bmAttributes bit 6 = 1
    Only uses USB for data
```

**Selective suspend:**

```
Inactive device enters suspend:

Suspend signal:
├── Bus enters SE0 state (D+ and D- both low)
├── Held for 3ms
└── Device detects and enters suspend mode

Suspended device:
└── Current draw: < 2.5mA (USB 2.0)
                 < 5mA (USB 3.0)

Resume:
├── Host sends K-state signal
├── Device wakes up
└── Or device can signal remote wakeup

Remote wakeup:
└── Device can wake suspended host
    Keyboard press wakes laptop
    USB descriptor enables this
    bmAttributes bit 5 = 1
```

**Linux USB autosuspend:**

```
Configuration:
/sys/bus/usb/devices/X-Y/power/
├── autosuspend_delay_ms
│   └── Delay before auto-suspend (default: 2000ms)
├── control
│   └── "auto" or "on" (enable/disable autosuspend)
└── level
    └── Current power state

Example (disable autosuspend for device 1-1):
echo -1 > /sys/bus/usb/devices/1-1/power/autosuspend_delay_ms
echo on > /sys/bus/usb/devices/1-1/power/control

Some devices have buggy suspend:
└── Driver adds USB_QUIRK_RESET_RESUME
    Full reset on resume instead of resume signaling
```

---

# 9. Physical Reality

## USB Signaling and Topology

**Differential signaling:**

```
USB 2.0 uses D+ and D- pair:

Differential receiver:
└── Measures V(D+) - V(D-)
    Immune to common-mode noise

Signaling:
├── J state: D+ high, D- low
├── K state: D+ low, D- high
├── SE0 (Single Ended Zero): Both low
└── SE1 (Single Ended One): Both high (illegal, error)

Speed detection:
Low Speed device (1.5 Mbps):
└── Pull-up on D- (device pulls D- to 3.3V)

Full/High Speed device (12/480 Mbps):
└── Pull-up on D+ (device pulls D+ to 3.3V)

Hub detects which line is pulled up:
└── Determines device speed category
```

**USB 3.0 dual-bus architecture:**

```
USB 3.0 cable contains:

Legacy USB 2.0 bus:
├── D+ and D- differential pair
├── VBUS and GND
└── Backward compatibility

SuperSpeed bus:
├── SSTX+ and SSTX- (SuperSpeed Transmit)
├── SSRX+ and SSRX- (SuperSpeed Receive)
├── Full duplex (simultaneous TX and RX)
└── Independent from USB 2.0 bus

Both buses active simultaneously:
└── USB 3.0 device in USB 3.0 port
    Negotiates SuperSpeed
    Falls back to USB 2.0 if needed

USB 3.0 device in USB 2.0 port:
└── Only USB 2.0 pins connect
    Works at High Speed (480 Mbps)
```

**Hub architecture:**

```
Hub internals:

Upstream port:
└── Connection to host or parent hub

Downstream ports:
└── Connections to devices or child hubs

Transaction Translator (TT):
└── Bridges different speeds
    High Speed host → Full/Low Speed device
    
    Host sends at 480 Mbps
    TT buffers
    TT sends to device at 12 Mbps
    Device responds at 12 Mbps
    TT buffers response
    TT sends to host at 480 Mbps

Split transactions:
└── HS host doesn't wait for FS device
    TT handles timing
    Keeps HS bus busy

Per-port power control:
└── Can disable individual ports
    Overcurrent protection per port
```

**Topology example:**

```
Physical USB tree:

xHCI Controller (PCIe device)
└── Root Hub (built into xHCI)
    ├── Port 1: External Hub
    │   ├── Port 1: Keyboard (Low Speed)
    │   ├── Port 2: Mouse (Low Speed)
    │   ├── Port 3: Flash Drive (High Speed)
    │   └── Port 4: Webcam (High Speed)
    ├── Port 2: Phone (High Speed)
    └── Port 3: Nothing

In /sys/bus/usb/devices/:
├── usb1 (root hub)
├── 1-1 (external hub)
├── 1-1.1 (keyboard)
├── 1-1.2 (mouse)
├── 1-1.3 (flash drive)
├── 1-1.4 (webcam)
└── 1-2 (phone)

Device path format:
└── bus-port.port.port...
    Example: 1-1.3 = Bus 1, Port 1, Sub-port 3
```

---

# 10. Connections to Other Topics

## USB in the Kernel Ecosystem

**Connection to Device Model:**

```
USB uses device model infrastructure:

struct usb_device:
└── Contains struct device
    Integrated into device hierarchy

struct usb_interface:
└── Contains struct device
    One per interface on device

Driver registration:
usb_register(&usb_storage_driver)
└── Calls driver_register()
    Uses device model matching
    probe() called when matched

sysfs hierarchy:
/sys/bus/usb/
├── devices/
│   ├── usb1 (controller)
│   ├── 1-1 (hub)
│   └── 1-1.3 (flash drive)
└── drivers/
    ├── usb-storage/
    └── usbhid/

Device matching uses id_table:
└── Vendor/Product ID or Class code
    Same pattern as PCIe, I2C, etc.
```

**Connection to PCIe:**

```
USB host controller is PCIe device:

xhci_pci_probe():
├── pci_enable_device(pdev)
├── pci_set_master(pdev)  // Enable DMA
├── pci_request_regions(pdev, "xhci_hcd")
├── ioremap(pci_resource_start(pdev, 0), size)
│   └── Map xHCI registers (BAR 0)
├── pci_alloc_irq_vectors(pdev, 1, nvecs, PCI_IRQ_MSIX)
│   └── MSI-X for event ring interrupts
└── request_irq(pci_irq_vector(pdev, i), xhci_irq, ...)

All PCIe mechanisms apply:
├── BARs for registers
├── DMA for data transfer
├── MSI-X for interrupts
└── IOMMU for address translation
```

**Connection to interrupts:**

```
xHCI uses MSI-X:

One vector per event ring:
└── Typically one per CPU
    Parallel interrupt handling

Interrupt handler:
xhci_irq():
├── Read event ring
├── Process completion events
├── Call URB callbacks
└── Re-enable interrupts

URB callbacks run in interrupt context:
└── Must be fast
    Cannot sleep
    Use GFP_ATOMIC for allocations
```

**Connection to DMA:**

```
URB data buffers use DMA:

usb_alloc_coherent():
└── Allocates DMA-capable memory
    Cache-coherent
    CPU and USB controller both access

dma_map_single():
└── For arbitrary kernel memory
    Returns bus address
    IOMMU may translate

Transfer rings in DMA memory:
└── xHCI reads ring via DMA
    No CPU copying
    Zero-copy to/from device
```

**Connection to block layer:**

```
USB storage → Block layer:

usb-storage driver:
├── Receives URBs from USB core
├── Wraps SCSI commands in CBW/CSW
├── Submits bulk URBs
└── Presents as SCSI host adapter

SCSI layer:
├── Sends SCSI commands
├── Creates gendisk
└── Integrates with block layer

Block layer sees:
└── /dev/sdb just like SATA/NVMe
    bio → request → SCSI → USB → URB

Same block layer mechanisms:
├── Request queues
├── I/O schedulers
├── Page cache integration
└── Partition handling
```

**Connection to input subsystem:**

```
USB HID → Input subsystem:

usbhid driver:
├── Polls via interrupt URBs
├── Receives HID reports
├── Parses report descriptor
└── Calls input layer

Input layer integration:
hid_input_report():
└── input_report_key(KEY_A, 1)
    input_report_rel(REL_X, delta)
    input_sync()

Result:
└── /dev/input/event0 receives events
    Same as PS/2 keyboard
    Applications see no difference
```

**Connection to network drivers:**

```
USB network adapters:

CDC NCM/ECM drivers:
├── Create net_device
├── Implement net_device_ops
└── ndo_start_xmit() → URB submission

Packet flow:
sk_buff → USB URB → Bulk OUT → Device
Device → Bulk IN → URB completion → sk_buff

Same network stack:
└── TCP/IP sees eth1
    No knowledge of USB
    Full network functionality
```

---

# Summary

## What You've Mastered

**You now understand USB:**

```
What it is: Universal peripheral bus with host/device architecture
Why needed: Unified protocol for diverse device types
Four transfer types: Control, Bulk, Interrupt, Isochronous
Key structure: URB (USB Request Block, like bio/sk_buff)
Complete flow: Plug-in → enumeration → driver probe → ready
```

---

## Key Takeaways

**Unified protocol:**

```
Diverse devices:
├── Keyboards (interrupt, 1.5 Mbps)
├── Storage (bulk, 5 Gbps)
├── Audio (isochronous, 12 Mbps)
└── Network (bulk, 480 Mbps)

All use same USB protocol:
└── Enumeration (read descriptors)
    Class drivers (HID, Mass Storage, CDC)
    URB abstraction (common submission mechanism)
```

**Host always initiates:**

```
Key principle:
└── Host sends all transactions
    Devices only respond when asked
    
"Interrupt" transfers:
└── Misleading name
    Host polls device periodically
    Not true hardware interrupts
```

**Enumeration magic:**

```
Plug device → 1-3 seconds → Ready

Steps:
├── Hub detects connection
├── Port reset and speed detection
├── Assign address (initially 0, then unique)
├── Read descriptors (device, config, interface, endpoint)
├── Set configuration
├── Driver matching and probe
└── Class driver creates device nodes

Happens transparently:
└── User just sees "USB drive ready"
```

**URB abstraction:**

```
Like bio for block, sk_buff for network:

URB lifecycle:
├── Allocate (usb_alloc_urb)
├── Fill (pipe, buffer, callback)
├── Submit (usb_submit_urb)
├── Controller processes
├── Completion callback
└── Re-submit or free

Asynchronous:
└── Submit returns immediately
    Callback called when done
```

**USB classes:**

```
Standardized device types:

Mass Storage (0x08):
└── SCSI over USB
    Creates /dev/sdb
    Block layer integration

HID (0x03):
└── Keyboards, mice, gamepads
    Self-describing reports
    Input subsystem integration

CDC (0x02):
└── Serial (ttyACM0)
    Network (eth1)
    TTY/network integration

Audio (0x01):
└── Isochronous transfers
    ALSA sound card
```

**xHCI host controller:**

```
Modern controller handles all speeds:
└── USB 3.2, 3.0, 2.0, 1.1 in one

Ring buffer architecture:
├── Command ring (host → controller)
├── Event ring (controller → host)
└── Transfer rings (per endpoint)

PCIe device:
└── Uses DMA, MSI-X, ioremap
    Same patterns as NVMe, NICs
```

**Hot-plug and power:**

```
Hot-plug:
└── Hub detects, reports change
    Enumeration starts automatically
    Driver probe, device appears

Safe eject:
└── SYNCHRONIZE CACHE command
    Flushes device write cache
    Prevents data corruption

Power management:
├── Selective suspend (idle devices sleep)
├── Remote wakeup (keyboard wakes laptop)
└── USB-PD (up to 100W charging)
```

---

## The Big Picture

**Complete USB stack:**

```
Physical connection:
└── User plugs device
    D+/D- voltage change
    Hub detects

Enumeration:
└── Reset, assign address, read descriptors
    Set configuration
    1-3 seconds total

Driver matching:
└── Class code or Vendor/Product ID
    probe() called
    Class driver initializes

Device operation:
└── URBs flow between driver and hardware
    Data transferred via DMA
    Completions via interrupts

Integration:
├── HID → Input subsystem
├── Mass Storage → Block layer
├── CDC → Network/TTY layers
└── Audio → ALSA

Result:
└── Applications see standard devices
    No USB knowledge required
    /dev/input/event0, /dev/sdb, /dev/ttyACM0
```

---

## You're Ready!

With this knowledge, you can:
- Understand USB device enumeration
- Debug USB device issues
- Write USB class drivers
- Troubleshoot hotplug problems
- Optimize USB data transfers
- Understand USB power management

> **USB is the universal peripheral interface - you now know how Linux makes keyboards, storage, audio, and network all work through one subsystem, with enumeration, class drivers, and URBs providing the abstraction that makes it all work seamlessly.**

---

**The protocol that connects everything.**
