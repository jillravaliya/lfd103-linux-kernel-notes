# From Wire to Socket - The Network Device Layer

> Ever wondered how a packet from Google reaches your browser? How NICs handle millions of packets per second without overwhelming the CPU? How the same code works for Intel, Realtek, and Broadcom cards?

**You're about to find out!**

---

## What's This About?

This is **drivers/net/** - the layer that controls Network Interface Cards and bridges hardware to the TCP/IP stack!

Here's where it fits in the driver ecosystem:

```
drivers/
04-device-drivers/
├── 00-foundation/      ← Device registration in kernel
├── 01-input/           ← Input events to userspace
├── 02-block/           ← Storage read and write
├── 03-net/             ← Packet send and receive
│   ├── net_device      ← NIC abstraction
│   ├── sk_buff         ← Packet structure
│   ├── NAPI            ← Interrupt optimization
│   ├── DMA rings       ← TX/RX descriptor rings
│   └── Hardware offloads ← Checksum, TSO, GRO, RSS
├── 04-usb/             ← USB device communication
├── 05-pci/             ← PCIe device discovery
├── 06-char/            ← Character device data flow
└── 07-gpu/             ← Display and rendering pipeline
```

**Location in code:**

```
drivers/net/
├── ethernet/
│   ├── intel/
│   │   ├── igb/        ← Intel Gigabit (desktop)
│   │   ├── ixgbe/      ← Intel 10GbE (server)
│   │   └── ice/        ← Intel 100GbE (modern)
│   ├── realtek/
│   │   └── r8169.c     ← Realtek (consumer)
│   └── broadcom/
│       └── bnxt_en/    ← Broadcom (data center)
└── wireless/
    └── intel/
        └── iwlwifi/    ← Intel WiFi

net/core/
├── dev.c               ← Core network device management
├── skbuff.c            ← sk_buff management
└── net-sysfs.c         ← /sys/class/net/

net/ipv4/
├── ip_input.c          ← IP receive path
├── ip_output.c         ← IP send path
├── tcp_input.c         ← TCP receive
└── tcp_output.c        ← TCP send
```

**This covers:**
- Network device abstraction (net_device structure)
- NAPI polling (interrupt to poll mode transition)
- sk_buff packet structure (zero-copy traversal)
- DMA ring buffers (TX and RX descriptor rings)
- Complete receive flow (wire to application)
- Complete send flow (application to wire)
- Hardware offloads (checksum, TSO, GRO, RSS)

---

# The Fundamental Problem

**Physical Reality:**

```
Your system has network connectivity:

Network Interface Cards (NICs):
├── Intel i219-V (integrated on motherboard)
├── Realtek RTL8111 (common consumer)
├── Broadcom NetXtreme (data center)
└── Intel X550 (10GbE server)

All different hardware:
├── Different register layouts
├── Different DMA mechanisms
├── Different interrupt handling
└── Different capabilities

But all need to:
├── Send packets to the network
├── Receive packets from the network
└── Interface with TCP/IP stack
```

**The problem:**

```
Without unified abstraction:

TCP/IP stack wants to send packet:
├── Intel NIC: Write to register 0x3800, DMA descriptor format A
├── Realtek NIC: Write to register 0x20, DMA descriptor format B
├── Broadcom NIC: Write to register 0x0040, DMA descriptor format C
└── Every NIC different

TCP/IP must know every NIC hardware detail
Impossible to maintain
No code reuse


Interrupt handling nightmare:

1 Gbps network = 1,488,000 packets/second (minimum 64-byte frames)

One interrupt per packet approach:
├── 1,488,000 interrupts per second
├── CPU spends all time in interrupt handlers
├── No time for actual packet processing
├── System becomes unresponsive
└── 100Gbps completely impossible

Need better interrupt strategy


Data copying disaster:

Packet arrives at NIC:
├── Copy 1: NIC buffer → kernel buffer
├── Copy 2: kernel buffer → socket buffer
├── Copy 3: socket buffer → user buffer
└── 3 copies per packet at line rate

At 10Gbps:
├── ~1.2 million packets/second
├── 3 copies each = 3.6 million copy operations
└── CPU saturated just copying data

Need zero-copy mechanism
```

**How do you provide ONE interface for ALL network hardware while handling millions of packets per second?**

---

# Without Network Drivers

**Imagine no unified network layer:**

```
TCP/IP stack sends data:

Must handle each NIC type differently:

Intel NIC path:
├── Allocate Intel-specific descriptor
├── Write to Intel control registers (offset 0x3800)
├── Use Intel DMA format
├── Handle Intel-specific interrupts
└── Intel-specific cleanup

Realtek NIC path:
├── Allocate Realtek-specific descriptor
├── Write to Realtek control registers (offset 0x20)
├── Use Realtek DMA format
├── Handle Realtek-specific interrupts
└── Realtek-specific cleanup

TCP/IP code: thousands of lines of hardware-specific logic
Maintenance nightmare
Every new NIC requires TCP/IP modifications


Interrupt storm at high packet rates:

Web server receiving 100,000 requests/second:
├── 100,000 HTTP requests
├── Each request: multiple packets
├── ~500,000 packets/second total

Old interrupt-per-packet approach:
├── 500,000 interrupts/second
├── Each interrupt: context switch overhead
├── Interrupt handler run time: ~2μs minimum
├── Total CPU time: 500,000 × 2μs = 1 second
└── 100% CPU just handling interrupts

No CPU time left for:
├── TCP processing
├── Socket management
├── Application processing
└── System becomes unresponsive


No optimization opportunities:

Multiple small packets for same TCP flow:
├── Packet 1: 64 bytes (minimum Ethernet frame)
├── Packet 2: 64 bytes
├── Packet 3: 64 bytes
└── Process each individually through entire stack

Could merge into larger units:
├── Reduces stack traversals
├── Better cache locality
└── But no layer exists to do this optimization


Hardware capabilities unused:

Modern NICs can do:
├── Checksum calculation in hardware
├── TCP segmentation in hardware
├── Packet filtering in hardware
└── Multi-queue support for parallelism

But without unified driver layer:
├── Each protocol stack reimplements
├── Or capabilities go unused
└── Performance left on table
```

**Chaos without network device abstraction!**

---

# The Three-Layer Solution

**Unified network interface:**

```
Layer 1: HARDWARE DRIVERS (device-specific)
├── igb.c (Intel Gigabit Ethernet)
├── r8169.c (Realtek)
├── bnxt_en (Broadcom)
└── Many more...

These know:
├── Hardware registers and layout
├── DMA descriptor formats
├── Interrupt handling specifics
└── Device initialization

These produce: packets as sk_buff structures


Layer 2: NETWORK CORE (net/core/)
├── Provides net_device abstraction
├── NAPI polling infrastructure
├── sk_buff memory management
├── Queue management (qdisc)
└── Hardware-independent algorithms


Layer 3: PROTOCOL STACK (net/ipv4/, net/ipv6/)
├── IP routing and forwarding
├── TCP connection management
├── UDP socket handling
└── Application socket interface

Uses: net_device operations (ndo_start_xmit, etc.)
No hardware knowledge


Result:
TCP wants to send → net_device->ndo_start_xmit() → Intel driver
TCP wants to send → net_device->ndo_start_xmit() → Realtek driver
TCP wants to send → net_device->ndo_start_xmit() → Broadcom driver

Same interface, any hardware
```

---

# 1. The net_device Structure

## Universal NIC Abstraction

**What is net_device?**

```c
struct net_device {
    char name[IFNAMSIZ];        // "eth0", "enp3s0", "wlan0"
    
    // Hardware identification
    unsigned char dev_addr[6];  // MAC address
    unsigned int mtu;           // Maximum Transmission Unit (1500)
    unsigned short type;        // ARPHRD_ETHER, ARPHRD_IEEE80211
    
    // State
    unsigned int flags;         // IFF_UP, IFF_RUNNING, IFF_PROMISC
    unsigned long state;        // Link state flags
    
    // Statistics
    struct net_device_stats stats;
        // tx_packets, rx_packets, tx_bytes, rx_bytes
        // tx_errors, rx_errors, tx_dropped, rx_dropped
    
    // Operations (THE KEY INTERFACE)
    const struct net_device_ops *netdev_ops;
    const struct ethtool_ops *ethtool_ops;
    
    // Transmit queues
    struct netdev_queue *_tx;   // Array of TX queues
    unsigned int num_tx_queues; // One per CPU ideally
    unsigned int real_num_tx_queues;
    
    // NAPI (polling)
    struct list_head napi_list; // NAPI structures for RX
    
    // Device model integration
    struct device dev;          // Links to driver model
    
    // Driver private data
    void *priv;                 // Points to driver-specific structure
};

One net_device per network interface
Provides uniform API to upper layers
Hides all hardware differences
```

**net_device_ops (the operation table):**

```c
struct net_device_ops {
    // Lifecycle
    int (*ndo_open)(struct net_device *dev);
        // Called when: ip link set eth0 up
        // Driver: allocate resources, start hardware
    
    int (*ndo_stop)(struct net_device *dev);
        // Called when: ip link set eth0 down
        // Driver: stop hardware, free resources
    
    // Transmit (MOST IMPORTANT)
    netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb,
                                   struct net_device *dev);
        // Called by: TCP/IP stack to send packet
        // Driver: queue packet to NIC hardware
    
    // Queue management
    u16 (*ndo_select_queue)(struct net_device *dev,
                           struct sk_buff *skb);
        // Choose which TX queue (for multi-queue)
    
    // Configuration
    int (*ndo_set_mac_address)(struct net_device *dev, void *addr);
    int (*ndo_change_mtu)(struct net_device *dev, int new_mtu);
    int (*ndo_set_rx_mode)(struct net_device *dev);
        // Multicast/promiscuous mode
    
    // Statistics
    void (*ndo_get_stats64)(struct net_device *dev,
                           struct rtnl_link_stats64 *stats);
    
    // Advanced features
    int (*ndo_set_features)(struct net_device *dev,
                           netdev_features_t features);
        // Enable/disable hardware offloads
    
    // Error handling
    void (*ndo_tx_timeout)(struct net_device *dev, unsigned int txqueue);
        // Called when transmit queue stalls
};

Like file_operations for NICs
All hardware differences abstracted behind these function pointers
```

---

# 2. The sk_buff Structure

## Universal Packet Container

**What is sk_buff?**

```c
struct sk_buff {
    // List management
    struct sk_buff *next;
    struct sk_buff *prev;
    
    // Device and routing
    struct net_device *dev;     // Which NIC received/will send
    struct dst_entry *dst;      // Routing destination
    
    // Protocol information
    __be16 protocol;            // ETH_P_IP, ETH_P_ARP, ETH_P_IPV6
    
    // Buffer pointers (THE MAGIC)
    unsigned char *head;        // Start of allocated buffer
    unsigned char *data;        // Current data pointer
    sk_buff_data_t tail;        // End of data
    sk_buff_data_t end;         // End of buffer
    
    unsigned int len;           // Length of data
    
    // Protocol header offsets
    __u16 mac_header;           // Offset to Ethernet header
    __u16 network_header;       // Offset to IP header
    __u16 transport_header;     // Offset to TCP/UDP header
    
    // Fragmentation (for zero-copy)
    unsigned char *head_frag;
    unsigned int data_len;      // Data in fragments
    __u8 nr_frags;              // Number of page fragments
    skb_frag_t frags[];         // Array of page fragments
    
    // Checksums
    __u8 ip_summed;             // CHECKSUM_NONE, CHECKSUM_PARTIAL, etc.
    __wsum csum;                // Checksum value
    
    // Timestamps
    ktime_t tstamp;             // When packet received
    
    // Socket owner
    struct sock *sk;            // Which socket owns this
};

EVERY network packet in kernel is an sk_buff
Moves through entire stack (NIC → IP → TCP → socket)
Zero-copy via pointer manipulation
```

**sk_buff buffer layout:**

```
Memory allocation:

    ┌──────────────────────────────────────────┐
    │ head                                     │ ← Buffer start
    │                                          │
    │ [headroom - space for adding headers]   │
    │                                          │
    ├──────────────────────────────────────────┤
    │ data ← skb->data                         │
    │                                          │
    │ [Ethernet header: 14 bytes]              │
    │   - Destination MAC (6 bytes)            │
    │   - Source MAC (6 bytes)                 │
    │   - EtherType (2 bytes)                  │
    │                                          │
    │ [IP header: 20 bytes minimum]            │
    │   - Version, IHL, TOS                    │
    │   - Total length                         │
    │   - Identification, Flags                │
    │   - TTL, Protocol                        │
    │   - Source IP (4 bytes)                  │
    │   - Destination IP (4 bytes)             │
    │                                          │
    │ [TCP header: 20 bytes minimum]           │
    │   - Source port (2 bytes)                │
    │   - Destination port (2 bytes)           │
    │   - Sequence number (4 bytes)            │
    │   - Acknowledgment (4 bytes)             │
    │   - Flags, Window                        │
    │   - Checksum, Urgent pointer            │
    │                                          │
    │ [Application data: N bytes]              │
    │                                          │
    │ tail ← skb->tail                         │
    ├──────────────────────────────────────────┤
    │ [tailroom - space for adding trailers]  │
    │                                          │
    │ end                                      │ ← Buffer end
    └──────────────────────────────────────────┘

Key insight: NO DATA COPYING as packet traverses layers!
Just pointer manipulation!
```

**Layer traversal (receive path):**

```
Packet arrives at NIC:
├── skb->data points to Ethernet header
├── All headers present in buffer

Driver processes:
└── eth_type_trans(skb, netdev)
    Reads EtherType field: 0x0800 (IP)
    Sets: skb->protocol = ETH_P_IP
    Advances: skb->data += 14 (skip Ethernet header)
    Stores: skb->mac_header = original data position

IP layer receives:
└── skb->data now points to IP header
    Reads IP header, validates
    Advances: skb->data += 20 (skip IP header)
    Stores: skb->network_header = IP header position

TCP layer receives:
└── skb->data now points to TCP header
    Reads TCP header, finds socket
    Advances: skb->data += 20 (skip TCP header)
    Stores: skb->transport_header = TCP header position

Socket receives:
└── skb->data now points to application payload
    copy_to_user(user_buffer, skb->data, len)
    Application gets data

Header pointers preserved:
└── Can walk backwards if needed
    mac_header, network_header, transport_header all valid
```

---

# 3. DMA Ring Buffers

## How NICs Transfer Packets

**What are DMA rings?**

```
DMA = Direct Memory Access
Ring = Circular buffer of descriptors

NIC and driver share two ring buffers:
├── RX ring (receive)
└── TX ring (transmit)

Each descriptor contains:
├── DMA address (where in RAM)
├── Length
├── Status/control flags
└── Done bit (hardware sets when complete)
```

**RX ring (receive path):**

```
Pre-initialization:

Driver allocates:
├── 256 sk_buff structures
├── Each backed by one page (4096 bytes)
└── DMA maps each page

Driver fills RX ring:
descriptor[0].addr = dma_addr_0
descriptor[0].length = 2048
descriptor[0].status = 0 (not done)

descriptor[1].addr = dma_addr_1
descriptor[1].length = 2048
descriptor[1].status = 0

... 256 descriptors total ...

Tells NIC:
├── RX ring base address: 0x12345000
├── RX ring size: 256
└── NIC stores these in hardware registers


Operation:

NIC maintains HEAD pointer (next descriptor to fill)
Driver maintains TAIL pointer (next descriptor to process)

    ┌───────────────────────────────────┐
    │ Descriptor 0  [DONE]              │ ← TAIL (driver processing)
    │ Descriptor 1  [DONE]              │
    │ Descriptor 2  [DONE]              │
    │ Descriptor 3  [DONE]              │
    │ Descriptor 4  [IN USE]            │ ← HEAD (NIC writing)
    │ Descriptor 5  [FREE]              │
    │ ...                               │
    │ Descriptor 255 [FREE]             │
    └───────────────────────────────────┘

Packet arrives:
1. NIC DMAs packet to descriptor[HEAD].addr
2. NIC sets descriptor[HEAD].status = DONE
3. NIC advances HEAD = (HEAD + 1) % 256
4. NIC raises interrupt (or driver polls)

Driver processes:
1. Check descriptor[TAIL].status == DONE
2. Get sk_buff from descriptor[TAIL]
3. dma_unmap_single(descriptor[TAIL].addr)
4. skb_put(skb, descriptor[TAIL].length)
5. Pass sk_buff to network stack
6. Allocate new sk_buff for this descriptor
7. Advance TAIL = (TAIL + 1) % 256
```

**TX ring (transmit path):**

```
Pre-initialization:

Driver allocates TX ring:
├── 256 descriptor slots
└── Initially all empty

Tells NIC:
├── TX ring base address
└── TX ring size


Operation:

Driver maintains HEAD (next free descriptor)
NIC maintains TAIL (next descriptor to transmit)

    ┌───────────────────────────────────┐
    │ Descriptor 0  [TRANSMITTED]       │ ← TAIL (NIC completed)
    │ Descriptor 1  [QUEUED]            │
    │ Descriptor 2  [QUEUED]            │ ← HEAD (driver adding)
    │ Descriptor 3  [FREE]              │
    │ ...                               │
    └───────────────────────────────────┘

TCP wants to send packet:
1. Driver gets sk_buff from TCP
2. dma_map_single(skb->data, skb->len)
3. descriptor[HEAD].addr = dma_addr
4. descriptor[HEAD].length = skb->len
5. descriptor[HEAD].cmd = TX | EOP
6. Advance HEAD = (HEAD + 1) % 256
7. Write HEAD to NIC register (doorbell)

NIC transmits:
1. Read descriptor[TAIL]
2. DMA read from descriptor[TAIL].addr
3. Transmit packet on wire
4. Set descriptor[TAIL].status = DONE
5. Advance TAIL = (TAIL + 1) % 256
6. Optionally raise interrupt

Driver cleanup (in poll or separate):
1. Check descriptor[old_tail].status == DONE
2. dma_unmap_single(descriptor[old_tail].addr)
3. kfree_skb(skb) (free the sk_buff)
4. Descriptor now available for reuse
```

**Why rings work:**

```
Batch processing:
└── Multiple packets queued before doorbell
    Reduces register writes
    Better PCIe efficiency

Pre-allocated buffers:
└── RX buffers allocated ahead of time
    No allocation in fast path
    Consistent performance

Lock-free operation:
└── Single writer (NIC for RX, driver for TX)
    Single reader (driver for RX, NIC for TX)
    No locks needed in many cases

Burst handling:
└── Large ring buffers bursts
    Software can't process fast enough
    Ring holds packets temporarily
```

---

# 4. NAPI (New API)

## The Interrupt Optimization

**The interrupt storm problem:**

```
1 Gbps Ethernet with minimum frames:
├── 64-byte frames (minimum Ethernet)
├── Frame time: 512 nanoseconds (64 bytes × 8 bits / 1e9 bits/sec)
└── Maximum rate: 1,488,095 frames/second

Old interrupt-per-packet approach:
├── 1,488,095 interrupts/second
├── Interrupt overhead: ~1-2 microseconds each
├── Total CPU time: ~1.5 seconds per second
└── Impossible! System locks up

10 Gbps: 10× worse
100 Gbps: 100× worse
Clearly doesn't scale
```

**NAPI solution:**

```
Hybrid interrupt/polling approach:

Quiet (no packets):
└── Interrupts enabled
    Waiting for first packet

Burst starts:
└── First packet arrives
    Interrupt fires
    Driver: "Disable interrupts, start polling"

Polling (burst active):
└── Driver polls ring buffer
    Processes packets without interrupts
    Gets packet 1, 2, 3, ... 1000
    All without any interrupts

Burst ends:
└── No more packets in ring
    Driver: "Re-enable interrupts"
    Return to quiet mode

Result:
├── 1 interrupt per burst instead of 1 per packet
├── 1000 packets: 1 interrupt instead of 1000
└── 1000× reduction in interrupt load
```

**NAPI state machine:**

```
struct napi_struct {
    struct list_head poll_list;    // In poll list when active
    unsigned long state;           // NAPI_STATE_SCHED, etc.
    
    int (*poll)(struct napi_struct *, int budget);
                                   // Driver's poll function
    int weight;                    // Max packets per poll (64)
    
    struct net_device *dev;        // Which device
};

State transitions:

    [IDLE - Interrupts enabled]
            │
            │ Packet arrives
            ↓
    [IRQ Handler runs]
            │
            │ napi_schedule(&napi)
            │ Disable NIC interrupts
            │
            ↓
    [SCHEDULED - In poll list]
            │
            │ Softirq: NET_RX_SOFTIRQ
            │
            ↓
    [POLLING]
            │
            │ napi->poll(napi, budget=64)
            │
            ├──→ [Processed < budget packets]
            │         │
            │         │ Burst done!
            │         │ napi_complete()
            │         │ Re-enable NIC interrupts
            │         │
            │         ↓
            │    [Return to IDLE]
            │
            └──→ [Processed == budget packets]
                      │
                      │ More packets likely coming
                      │ Stay in poll list
                      │ Will be called again soon
                      │
                      ↓
                 [POLLING continues]
```

**Complete NAPI flow:**

```
Step 1: Interrupt handler (fast path)

igb_intr(int irq, void *data):
    // Read interrupt cause
    icr = readl(hw->hw_addr + E1000_ICR)
    
    // RX interrupt?
    if (icr & E1000_ICR_RXT0):
        // Schedule NAPI
        napi_schedule(&adapter->napi)
        
        // Disable further NIC interrupts
        igb_irq_disable(adapter)
    
    return IRQ_HANDLED

Returns immediately
Total time: < 1 microsecond


Step 2: NAPI softirq (scheduled by napi_schedule)

net_rx_action():
    // Process all scheduled NAPI instances
    while (!list_empty(&sd->poll_list) && budget > 0):
        napi = first_entry(&sd->poll_list)
        
        // Call driver's poll function
        work = napi->poll(napi, weight=64)
        
        budget -= work


Step 3: Driver poll function

igb_poll(struct napi_struct *napi, int budget):
    cleaned = 0
    
    // Process up to 'budget' packets
    while (cleaned < budget):
        // Check next RX descriptor
        rx_desc = &rx_ring->desc[tail]
        
        // Done bit set?
        if (!(rx_desc->status & E1000_RXD_STAT_DD)):
            break  // No more packets
        
        // Get sk_buff
        skb = rx_ring->buffer_info[tail].skb
        
        // Unmap DMA
        dma_unmap_single(dev, rx_desc->addr, 
                        rx_desc->length, DMA_FROM_DEVICE)
        
        // Set sk_buff data length
        skb_put(skb, rx_desc->length)
        
        // Determine protocol
        skb->protocol = eth_type_trans(skb, netdev)
        
        // Pass to network stack
        napi_gro_receive(napi, skb)
        
        // Allocate new buffer for this descriptor
        alloc_new_rx_buffer(rx_ring, tail)
        
        tail = (tail + 1) % ring_size
        cleaned++
    
    // Processed less than budget?
    if (cleaned < budget):
        // Burst is over
        napi_complete_done(napi, cleaned)
        
        // Re-enable NIC interrupts
        igb_irq_enable(adapter)
    
    return cleaned
```

**NAPI benefits:**

```
Interrupt reduction:
└── 1000 packet burst: 1 interrupt vs 1000 interrupts
    999 interrupts saved
    ~1ms CPU time saved

Batch processing:
└── Process multiple packets in sequence
    Better cache locality
    More efficient CPU usage

Fairness:
└── Budget limits (64 packets per NAPI call)
    Other work gets CPU time
    System remains responsive

Scalability:
└── Works at 1 Gbps
    Works at 10 Gbps
    Works at 100 Gbps
    Essential for modern networking
```

---

# 5. Hardware Offloads

## Doing Work in Silicon

**Checksum offload:**

```
Problem: TCP/UDP checksums expensive

Software checksum:
├── Sum all words in packet
├── Takes ~50-100 CPU cycles per packet
└── At 1 million packets/sec: significant CPU

Hardware checksum:
├── NIC calculates checksum during DMA
├── Zero CPU cycles
└── Free! (hardware does it anyway while transferring)


Transmit checksum offload:

TCP prepares packet:
└── Fills in all fields EXCEPT checksum
    Sets: skb->ip_summed = CHECKSUM_PARTIAL
    NIC will fill in checksum

Driver transmit:
└── Sees CHECKSUM_PARTIAL flag
    Sets descriptor flag: E1000_TXD_CMD_TCP
    NIC calculates and inserts checksum during TX


Receive checksum offload:

NIC receives packet:
└── Calculates checksum during RX DMA
    Sets descriptor flag if checksum valid

Driver processes:
└── Sees valid checksum flag
    Sets: skb->ip_summed = CHECKSUM_UNNECESSARY
    TCP stack skips checksum verification

Result: Both directions, zero CPU for checksums
```

**TSO (TCP Segmentation Offload):**

```
Problem: Large send requires many segments

Application sends 64 KB:
├── TCP MSS = 1460 bytes (typical)
├── Segments needed: 64KB / 1460 = ~44 segments
└── CPU must build 44 separate packets

Each segment needs:
├── IP header
├── TCP header with sequence numbers
├── Ethernet header
└── Lots of CPU work


With TSO:

TCP builds one super-packet:
├── Full 64 KB of data
├── Single TCP/IP header template
└── Sets: skb->gso_size = 1460

Driver sees TSO request:
└── Sets descriptor flag: E1000_TXD_CMD_TSO
    Provides TCP header template
    NIC does segmentation

NIC segments automatically:
├── Splits 64KB into 44 × 1460-byte segments
├── Creates proper headers for each
├── Updates sequence numbers
└── Transmits all 44 frames

CPU work: 1 packet instead of 44
Massive savings for bulk transfers
```

**GRO (Generic Receive Offload):**

```
Problem: Many small packets from same flow

Typical scenario:
├── Web server receiving large HTTP POST
├── Arrives as many 1460-byte packets
└── All consecutive in same TCP connection

Without GRO:
└── Process each packet individually
    Packet 1: full stack traversal
    Packet 2: full stack traversal
    ...
    Packet 44: full stack traversal
    44 stack traversals total


With GRO (in napi_gro_receive):

Check incoming packet:
└── Same TCP flow as previous packet?
    Same src/dst IP and port?
    Consecutive sequence numbers?
    
    YES → Merge into existing sk_buff:
        Append data to sk_buff
        Update length fields
        Keep first packet, add data from second

Result:
└── 44 packets merged into one large sk_buff
    One stack traversal instead of 44
    26× reduction in processing

Application receives:
└── Large read instead of many small reads
    Better throughput
    Less overhead
```

**RSS (Receive Side Scaling):**

```
Problem: Single RX queue bottleneck

Traditional NIC:
└── All packets → single RX queue → single CPU
    CPU 0 handles all packets
    CPUs 1-7 idle
    Bottleneck at high rates


RSS-capable NIC:

Multiple RX queues:
└── Queue 0 → CPU 0
    Queue 1 → CPU 1
    Queue 2 → CPU 2
    Queue 3 → CPU 3

Hash function:
└── Hash(src IP, dst IP, src port, dst port)
    Result: 0-3 (which queue)

Packet arrives:
└── NIC calculates hash
    Hash = 0 → Queue 0 → CPU 0 processes
    Hash = 1 → Queue 1 → CPU 1 processes
    Hash = 2 → Queue 2 → CPU 2 processes
    Hash = 3 → Queue 3 → CPU 3 processes

Same flow always same queue:
└── Flow A always → Queue 0 → CPU 0
    No lock contention on socket
    CPU cache hot for that flow

Result:
└── 4 CPUs processing in parallel
    4× throughput
    Scales to 100+ Gbps with enough queues
```

---

# 6. Complete Receive Flow

## Wire to Application

**Scenario: curl receives HTTP response**

**Step 1: Physical layer**

```
Packet arrives on Ethernet cable:
├── Electrical signals (differential pairs)
├── PHY chip converts analog → digital
└── NIC MAC receives Ethernet frame

NIC validates:
├── Destination MAC = our MAC address?
├── FCS (Frame Check Sequence) correct?
└── If yes: accept, if no: drop

Frame structure:
[Dest MAC][Src MAC][EtherType=0x0800][IP header][TCP header][Data][FCS]
```

**Step 2: DMA to RAM**

```
NIC DMA engine:
├── Current RX descriptor: descriptor[HEAD]
├── DMA address: 0xABCD0000 (pre-allocated page)
└── Length: 2048 bytes

DMA write:
└── Copy entire frame to physical address 0xABCD0000
    Via PCIe bus (no CPU involvement)
    Memory controller writes to RAM

Descriptor update:
└── Set descriptor[HEAD].status = DONE
    Set descriptor[HEAD].length = 1514 (frame size)
    Advance HEAD = (HEAD + 1) % 256

Total time: ~1-2 microseconds
```

**Step 3: Interrupt**

```
NIC raises interrupt:
└── MSI-X write to APIC address
    CPU 3 interrupt vector triggered

igb_intr() handler:
└── Read interrupt cause register
    ICR shows: RX packet received
    
    napi_schedule(&adapter->napi)
    igb_irq_disable(adapter)
    
    return IRQ_HANDLED

NET_RX_SOFTIRQ scheduled:
└── Will run when CPU returns from interrupt

Interrupt handler time: < 1 microsecond
```

**Step 4: NAPI poll**

```
net_rx_action() softirq:
└── Calls napi->poll(napi, budget=64)

igb_poll():
    // Process descriptors
    rx_desc = &rx_ring->desc[tail]
    
    if (rx_desc->status & DONE):
        // Get pre-allocated sk_buff
        skb = rx_ring->buffer_info[tail].skb
        
        // Unmap DMA
        dma_unmap_single(dev, rx_desc->addr, 
                        rx_desc->length, DMA_FROM_DEVICE)
        
        // Set data length
        skb_put(skb, rx_desc->length)
        
        // skb->data now points to Ethernet frame
        
        // Determine protocol
        skb->protocol = eth_type_trans(skb, netdev)
        // Returns: ETH_P_IP (0x0800)
        // Advances skb->data past Ethernet header
        
        // Hardware checksum
        if (rx_desc->flags & HW_CSUM_VALID):
            skb->ip_summed = CHECKSUM_UNNECESSARY
        
        // Pass to network stack
        napi_gro_receive(napi, skb)
```

**Step 5: Network stack entry**

```
napi_gro_receive(napi, skb):
└── Attempt GRO merge
    Check: Same TCP flow as buffered packet?
    
    If no merge:
        netif_receive_skb(skb)

netif_receive_skb(skb):
└── skb->protocol = ETH_P_IP
    Lookup protocol handler
    
    Call: ip_rcv(skb, dev, pt, orig_dev)
```

**Step 6: IP layer**

```
ip_rcv(skb, dev, pt, orig_dev):
    // Validate IP header
    iph = ip_hdr(skb)
    
    // Checks
    if (iph->version != 4): drop
    if (iph->ihl < 5): drop
    if (ip_fast_csum(iph) && !hw_csum): drop
    
    // Store network_header offset
    skb->network_header = skb->data - skb->head
    
    // Advance past IP header
    skb_pull(skb, iph->ihl * 4)
    
    // Routing
    ip_route_input(skb, iph->daddr, iph->saddr, iph->tos, dev)
    
    // For us?
    ip_local_deliver(skb)

ip_local_deliver(skb):
    // Check protocol field
    protocol = iph->protocol = 6 (TCP)
    
    // Find protocol handler
    ipprot = inet_protos[IPPROTO_TCP]
    
    // Call TCP
    ipprot->handler(skb) // tcp_v4_rcv(skb)
```

**Step 7: TCP layer**

```
tcp_v4_rcv(skb):
    // Get TCP header
    th = tcp_hdr(skb)
    
    // Store transport_header offset
    skb->transport_header = skb->data - skb->head
    
    // Find socket
    sk = __inet_lookup_skb(&tcp_hashinfo, skb,
                           th->source, th->dest)
    
    // Found curl's socket!
    
    // Advance past TCP header
    skb_pull(skb, th->doff * 4)
    
    // Now skb->data points to HTTP payload
    
    // Process TCP
    tcp_v4_do_rcv(sk, skb)

tcp_rcv_established(sk, skb, th, len):
    // Check sequence numbers
    if (TCP_SKB_CB(skb)->seq == tp->rcv_nxt):
        // In-order packet
        
        // Add to socket receive queue
        __skb_queue_tail(&sk->sk_receive_queue, skb)
        
        // Update received sequence number
        tp->rcv_nxt += skb->len
        
        // Send ACK (delayed or immediate)
        tcp_send_ack(sk)
        
        // Wake sleeping process
        sk->sk_data_ready(sk)
```

**Step 8: Application wakes**

```
curl was sleeping:
└── read(socket_fd, buffer, 4096)
    Blocked in tcp_recvmsg()
    Waiting on: sk->sk_sleep

wake_up_interruptible(&sk->sk_sleep):
└── Scheduler marks curl runnable
    Next timeslice: curl runs

tcp_recvmsg():
    // Get sk_buff from queue
    skb = skb_peek(&sk->sk_receive_queue)
    
    // Copy data to user
    skb_copy_datagram_msg(skb, offset, msg, len)
        // Eventually: copy_to_user()
    
    // Remove from queue if fully consumed
    skb_unlink(skb, &sk->sk_receive_queue)
    kfree_skb(skb)
    
    return bytes_copied

curl:
└── read() returns with HTTP data
    Parses response
    Displays to user

Total time: ~6-10 microseconds (wire to application)
```

---

# 7. Complete Send Flow

## Application to Wire

**Scenario: curl sends HTTP GET request**

**Step 1: Application write**

```
curl:
└── write(socket_fd, "GET / HTTP/1.1\r\n...", len)
    
    sys_write() → sock_write() → tcp_sendmsg()
```

**Step 2: TCP layer**

```
tcp_sendmsg(sk, msg, size):
    // Allocate sk_buff
    skb = sk_stream_alloc_skb(sk, size, GFP_KERNEL)
    
    // Copy from user
    copy_from_user(skb->data, msg->msg_iter, size)
    
    // Build TCP header
    th = tcp_hdr(skb)
    skb_push(skb, sizeof(struct tcphdr))
    
    th->source = inet_sport  // curl's port
    th->dest = inet_dport    // 80 (HTTP)
    th->seq = htonl(tp->write_seq)
    th->ack_seq = htonl(tp->rcv_nxt)
    th->doff = 5
    th->window = htons(tcp_select_window(sk))
    
    // Checksum (or hardware offload)
    if (CHECKSUM_PARTIAL supported):
        skb->ip_summed = CHECKSUM_PARTIAL
        // NIC will calculate
    else:
        th->check = tcp_v4_check(skb->len, saddr, daddr, skb)
    
    // Store transport_header
    skb->transport_header = skb->data - skb->head
    
    // Send to IP layer
    ip_queue_xmit(sk, skb, &inet->cork.fl)
```

**Step 3: IP layer**

```
ip_queue_xmit(sk, skb, fl):
    // Route lookup
    rt = ip_route_output_key(net, fl4)
    // Result: interface=eth0, gateway=192.168.1.1
    
    // Build IP header
    iph = ip_hdr(skb)
    skb_push(skb, sizeof(struct iphdr))
    
    iph->version = 4
    iph->ihl = 5
    iph->tos = inet->tos
    iph->tot_len = htons(skb->len)
    iph->id = htons(atomic_inc(&ip_ident))
    iph->ttl = ip4_dst_hoplimit(dst)
    iph->protocol = IPPROTO_TCP
    iph->saddr = fl4->saddr
    iph->daddr = fl4->daddr
    
    // Checksum
    ip_send_check(iph)
    
    // Store network_header
    skb->network_header = skb->data - skb->head
    
    // Fragment if needed (rare)
    if (skb->len > mtu):
        ip_fragment(net, sk, skb, mtu, ip_finish_output2)
    else:
        ip_finish_output2(net, sk, skb)
```

**Step 4: Ethernet layer**

```
ip_finish_output2(net, sk, skb):
    // ARP lookup for gateway MAC
    neigh = ip_neigh_for_gw(rt, skb, &is_v6gw)
    
    if (!neigh):
        // No ARP entry, send ARP request first
        arp_solicit(...)
    
    // Have MAC address
    neigh_output(neigh, skb, is_v6gw)

dev_queue_xmit(skb):
    // Build Ethernet header
    eth = (struct ethhdr *)skb->data
    skb_push(skb, ETH_HLEN)
    
    memcpy(eth->h_dest, gateway_mac, 6)
    memcpy(eth->h_source, our_mac, 6)
    eth->h_proto = htons(ETH_P_IP)
    
    // Store mac_header
    skb->mac_header = skb->data - skb->head
    
    // Select TX queue
    txq = netdev_pick_tx(dev, skb, NULL)
    
    // Traffic control (qdisc)
    q = rcu_dereference_bh(txq->qdisc)
    q->enqueue(skb, q, &to_free)
    
    __qdisc_run(q)
    
    // Eventually: ops->ndo_start_xmit(skb, dev)
```

**Step 5: Driver transmit**

```
igb_xmit_frame(skb, netdev):
    tx_ring = adapter->tx_ring[queue_index]
    
    // DMA map
    dma = dma_map_single(dev, skb->data, skb->len, DMA_TO_DEVICE)
    
    // Get TX descriptor
    tx_desc = &tx_ring->desc[tx_ring->next_to_use]
    
    // Fill descriptor
    tx_desc->buffer_addr = cpu_to_le64(dma)
    tx_desc->cmd_type_len = cpu_to_le32(
        skb->len |
        E1000_TXD_CMD_EOP |
        E1000_TXD_CMD_RS |
        E1000_TXD_CMD_IFCS
    )
    
    if (skb->ip_summed == CHECKSUM_PARTIAL):
        tx_desc->cmd_type_len |= E1000_TXD_CMD_TCP
    
    // Store sk_buff reference
    tx_ring->buffer_info[tx_ring->next_to_use].skb = skb
    tx_ring->buffer_info[tx_ring->next_to_use].dma = dma
    
    // Advance ring pointer
    tx_ring->next_to_use++
    if (tx_ring->next_to_use == tx_ring->count):
        tx_ring->next_to_use = 0
    
    // Ring doorbell (tell NIC)
    writel(tx_ring->next_to_use, tx_ring->tail)
```

**Step 6: NIC transmission**

```
NIC hardware:
    // Read TX descriptor
    desc = tx_ring[tail]
    
    // DMA read packet data
    pcie_dma_read(desc->buffer_addr, desc->length)
    
    // Calculate checksum if requested
    if (desc->cmd & E1000_TXD_CMD_TCP):
        calculate_tcp_checksum(packet)
        insert_checksum(packet)
    
    // Add FCS (Frame Check Sequence)
    calculate_fcs(packet)
    append_fcs(packet)
    
    // Send to PHY
    transmit_frame(packet)
    
    // Mark descriptor done
    desc->status |= E1000_TXD_STAT_DD
    
    // Advance tail
    tail++
    
    // Optional: raise interrupt

PHY chip:
└── Convert digital → analog
    Drive electrical signals on Ethernet cable
    Frame transmitted on wire

Total time: ~5-10 microseconds
```

**Step 7: TX completion**

```
Later (in NAPI poll or TX IRQ):

igb_clean_tx_irq(tx_ring, budget):
    while (budget--):
        tx_desc = &tx_ring->desc[tx_ring->next_to_clean]
        
        // Check done bit
        if (!(tx_desc->status & E1000_TXD_STAT_DD)):
            break  // Not done yet
        
        // Get buffer info
        tx_buffer = &tx_ring->buffer_info[tx_ring->next_to_clean]
        
        // Unmap DMA
        dma_unmap_single(dev, tx_buffer->dma,
                        tx_buffer->length, DMA_TO_DEVICE)
        
        // Free sk_buff
        dev_kfree_skb_any(tx_buffer->skb)
        
        // Clear buffer info
        tx_buffer->skb = NULL
        
        // Advance
        tx_ring->next_to_clean++
```

---

# 8. Traffic Control (qdisc)

## Shaping and Prioritization

**What is qdisc?**

```
Queueing Discipline = Traffic control layer

Sits between network layer and driver:

    IP layer produces packet
            ↓
    [qdisc - Traffic Control]
        ├── Queue management
        ├── Scheduling
        ├── Rate limiting
        └── Prioritization
            ↓
    Driver transmits packet

Can shape, limit, prioritize traffic before NIC sees it
```

**Default qdisc: pfifo_fast**

```
Three priority bands (FIFO each):

    Band 0: High priority
        ├── Interactive traffic (SSH, DNS)
        └── Low latency required
    
    Band 1: Normal priority
        ├── Most traffic (HTTP, SMTP)
        └── Default band
    
    Band 2: Bulk traffic
        ├── Background transfers (rsync, FTP)
        └── Can be delayed

Packet arrives:
└── Check TOS (Type of Service) field in IP header
    TOS → determines band
    
    Transmit order:
    1. Empty band 0 completely
    2. Then band 1
    3. Then band 2

Result:
└── SSH remains responsive even during large download
    Interactive traffic prioritized
```

**Other qdiscs:**

```
htb (Hierarchical Token Bucket):
└── Rate limiting per class
    "This class gets max 10 Mbps"
    "That class gets max 1 Mbps"
    Used by ISPs for customer limits

fq_codel (Fair Queueing + Controlled Delay):
└── Per-flow fairness
    Separate queue per TCP flow
    Flows don't interfere
    Active queue management (AQM)
    Reduces bufferbloat

tbf (Token Bucket Filter):
└── Simple rate limiting
    Tokens added at fixed rate
    Packet consumes tokens
    No tokens = packet delayed

Configuration:
└── tc qdisc show dev eth0
    tc qdisc add dev eth0 root fq_codel
    tc qdisc add dev eth0 root handle 1: htb default 12
```

---

# 9. Multi-Queue and RSS

## Parallel Processing

**The single-queue problem:**

```
Traditional NIC:
└── Single RX queue
    Single TX queue
    All packets processed by one CPU

High packet rate:
└── CPU 0 receives all packets
    Processes interrupts
    Runs NAPI poll
    Handles TCP/IP stack
    
    CPUs 1-7: idle
    
    Bottleneck at CPU 0
    Can't scale beyond single-core performance
```

**Multi-queue solution:**

```
Modern NIC (Intel X550):
└── 8 RX queues
    8 TX queues
    Each queue independent

RSS (Receive Side Scaling):
└── NIC hashes each packet:
    hash = hash(src_ip, dst_ip, src_port, dst_port)
    queue = hash % num_queues
    
    Packet routed to specific queue

Result:
├── Flow A → Queue 0 → CPU 0
├── Flow B → Queue 1 → CPU 1
├── Flow C → Queue 2 → CPU 2
└── Flow D → Queue 3 → CPU 3

All CPUs working in parallel
Scales to 100+ Gbps


Benefits:

Same flow always same CPU:
└── TCP socket processed by same CPU
    No lock contention
    CPU cache hot for that flow
    Better performance

Interrupt distribution:
└── Each queue has own MSI-X vector
    Interrupts routed to different CPUs
    Load balanced

TX queues:
└── Each CPU transmits on its own queue
    No contention
    XPS (Transmit Packet Steering) assigns
```

**Configuration:**

```
View queues:
└── ls -la /sys/class/net/eth0/queues/
    rx-0/ rx-1/ rx-2/ rx-3/
    tx-0/ tx-1/ tx-2/ tx-3/

Set number of queues:
└── ethtool -L eth0 combined 8

View RSS hash:
└── ethtool -n eth0 rx-flow-hash tcp4

XPS configuration:
└── /sys/class/net/eth0/queues/tx-0/xps_cpus
    echo f > xps_cpus  (CPUs 0-3)
```

---

# 10. Physical Reality

## What Actually Happens

**NIC chip internals:**

```
Modern 10GbE NIC (Intel X550):

Silicon contains:
├── PCIe interface (Gen 3 x8)
├── DMA engines (multiple)
├── MAC (Media Access Control)
├── Packet buffer memory (on-chip SRAM)
├── Checksum calculation units
├── TSO/GRO engines
├── RSS hash calculator
├── Multiple queue managers
└── Interrupt controller (MSI-X)

External:
└── PHY chip (converts digital ↔ analog)
    SFP+ module (fiber optic)
    Or RJ45 (copper Ethernet)
```

**Timing breakdown:**

```
Receive path (10GbE, 1500-byte packet):

T=0μs:     Frame arrives on wire
T=1.2μs:   PHY receives complete frame (1500 bytes / 10Gbps)
T=1.3μs:   MAC validates FCS
T=1.4μs:   DMA write starts
T=2.5μs:   DMA write completes (PCIe Gen 3 x8: ~6 GB/s)
T=2.6μs:   Interrupt posted (MSI-X write)
T=3.0μs:   CPU receives interrupt
T=3.5μs:   NAPI scheduled
T=4.0μs:   Driver poll function runs
T=5.0μs:   IP layer processing
T=6.0μs:   TCP layer processing
T=7.0μs:   Socket queue
T=8.0μs:   Application wakes

Total: ~8 microseconds (wire to application)

At 10 Gbps with minimum frames:
└── 14.88 million packets/second
    67 nanoseconds between packets
    NAPI essential (polling, not interrupts)
```

**Physical Ethernet:**

```
10GBASE-T (copper, RJ45):
└── 4 twisted pairs
    Differential signaling on each
    Symbol rate: 800 MHz
    Encoding: complex (PAM-16)

10GBASE-SR (fiber, multimode):
└── 850nm wavelength
    300m maximum distance
    Low cost for data centers

10GBASE-LR (fiber, singlemode):
└── 1310nm wavelength
    10km maximum distance
    Long-haul connections

PHY chip:
└── Handles encoding/decoding
    Clock recovery
    Signal equalization
    Link training
```

**DMA and PCIe:**

```
Receive DMA:
└── Packet arrives at NIC
    NIC writes to RX descriptor address
    PCIe Memory Write TLP (Transaction Layer Packet)
    Memory controller updates RAM
    No CPU involvement

Doorbell register:
└── Driver writes TX queue tail pointer
    writel(tail, hw->hw_addr + E1000_TDT)
    This is ioremap'd address
    PCIe write to NIC BAR
    NIC sees new tail value
    Begins transmit

MSI-X interrupt:
└── NIC writes to APIC address range
    PCIe write to 0xFEExxxxx
    APIC interprets as interrupt
    Delivers to CPU
    Same mechanism as all PCIe devices
```

---

# 11. Connections to Other Topics

## Network Drivers in Context

**Connection to PCIe:**

```
NIC is PCIe device

Driver probe:
└── pci_enable_device()
    pci_set_master() (enable DMA)
    pci_request_regions() (claim BARs)
    ioremap() (map NIC registers)
    pci_alloc_irq_vectors() (MSI-X)
    request_irq() (register handlers)

All PCIe mechanisms from TOPIC 21 apply
```

**Connection to ioremap:**

```
NIC registers accessed via ioremap:

igb_probe():
└── hw->hw_addr = ioremap(pci_resource_start(pdev, 0), size)
    
    Register writes:
    writel(value, hw->hw_addr + offset)
    
    Doorbell rings:
    writel(tail, tx_ring->tail)
    
    All via ioremap from TOPIC 12
```

**Connection to interrupts:**

```
MSI-X for each queue:

Setup:
└── pci_alloc_irq_vectors(pdev, num_queues, num_queues,
                          PCI_IRQ_MSIX)
    
    For each queue:
    request_irq(pci_irq_vector(pdev, i),
               igb_msix_ring, 0, name, q_vector)

Interrupt delivery:
└── NIC writes to APIC address
    APIC routes to CPU
    Handler runs
    
    Same mechanism as TOPICS 3 and 6
```

**Connection to DMA and memory:**

```
sk_buff allocation:

alloc_skb(size, GFP_ATOMIC):
└── From SLAB allocator (TOPIC 11)
    
DMA mapping:
└── dma_map_single(dev, skb->data, len, DMA_FROM_DEVICE)
    IOMMU translates if present
    Returns bus address for NIC
    
    From memory zones (TOPIC 10)
    GFP_ATOMIC in interrupt context

Ring buffers:
└── dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL)
    Physically contiguous
    CPU and device both access
    Cache coherent
```

**Connection to block layer:**

```
Same pattern as NVMe and SATA:

Both use:
├── Ring buffer descriptors
├── DMA for data transfer
├── Doorbell register writes
├── Interrupt on completion
└── Polling for high rates

Network: RX/TX rings
Block: Submission/Completion queues

Same fundamental hardware interface pattern
```

**Connection to device model:**

```
net_device in sysfs:

/sys/class/net/eth0/
├── address (MAC)
├── mtu
├── speed
├── carrier (link up/down)
├── statistics/
│   ├── rx_packets
│   ├── tx_packets
│   ├── rx_bytes
│   └── tx_bytes
└── queues/
    ├── rx-0/
    └── tx-0/

uevents for hotplug:
└── NIC inserted
    uevent sent
    NetworkManager receives
    DHCP runs
    IP configured
    
    Device model from TOPIC 15
```

---

# Summary

## What You've Mastered

**You now understand network drivers:**

```
What they are: Software bridge between NICs and TCP/IP stack
Why needed: Hardware diversity, interrupt optimization, zero-copy
Three layers: Hardware drivers → Network core → Protocol stack
Key structures: net_device (NIC), sk_buff (packet), NAPI, DMA rings
Complete flows: Wire → socket receive, socket → wire transmit
```

---

## Key Takeaways

**Unified abstraction:**

```
Many NIC types:
├── Intel (register layout A, DMA format A)
├── Realtek (register layout B, DMA format B)
└── Broadcom (register layout C, DMA format C)

All look same to TCP/IP:
└── netdev->ndo_start_xmit(skb, dev)
    Hardware differences hidden
    Protocol stack hardware-agnostic
```

**NAPI polling:**

```
Problem: Interrupt per packet = CPU overload

Solution:
├── First packet: interrupt
├── Disable interrupts
├── Poll ring buffer (no more interrupts)
├── Process burst of packets
└── Re-enable interrupts when done

Result:
└── 1 interrupt per 1000 packets
    CPU can handle 100 Gbps
```

**sk_buff zero-copy:**

```
Packet traverses entire stack:
└── NIC → Ethernet → IP → TCP → Socket → Application

No data copying:
└── Just pointer manipulation
    skb->data advances through headers
    mac_header, network_header, transport_header preserved
    
    6-layer traversal, zero copies
```

**DMA rings:**

```
RX ring:
└── Pre-allocated pages
    NIC DMAs directly to RAM
    Driver polls descriptors
    Zero allocation in fast path

TX ring:
└── Driver builds descriptors
    NIC DMAs from RAM
    Transmits on wire
    Driver cleans up later
```

**Hardware offloads:**

```
Checksum: NIC calculates, CPU free
TSO: One large send → NIC segments into many
GRO: Many small receives → merged into one
RSS: Multiple queues → parallel CPUs

Silicon does work:
└── CPU focuses on protocol logic
    Not bit manipulation
    Scales to 100+ Gbps
```

**Multi-queue scaling:**

```
Single queue:
└── All packets → CPU 0
    CPUs 1-7 idle
    Bottleneck

Multi-queue:
└── Flow A → Queue 0 → CPU 0
    Flow B → Queue 1 → CPU 1
    Flow C → Queue 2 → CPU 2
    Flow D → Queue 3 → CPU 3
    
    Parallel processing
    True scaling
```

---

## The Big Picture

**Complete packet path:**

```
Receive:
└── Wire → PHY → NIC MAC → DMA → RAM
    → Interrupt → NAPI → Driver poll
    → GRO → IP layer → TCP layer
    → Socket queue → Application read()

Send:
└── Application write() → Socket → TCP
    → IP → Ethernet → qdisc → Driver
    → DMA descriptor → Doorbell → NIC
    → PHY → Wire

Both directions: ~5-10 microseconds
No data copying (zero-copy via sk_buff)
Hardware offloads in silicon
Multi-queue for parallel scaling
```

---

## You're Ready!

With this knowledge, you can:
- Understand how packets reach applications
- Debug network performance issues
- Tune network stack parameters
- Write network device drivers
- Optimize for high-throughput workloads
- Understand RSS, GRO, TSO internals

---

**The abstraction that makes networking universal.**

> **Network drivers are the bridge from electrical signals to application sockets - you now know how Linux makes all NICs speak one language while handling millions of packets per second.**

