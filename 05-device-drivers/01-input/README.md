# From Keypress to Character - The Input Layer

> Ever wondered how pressing 'A' on your keyboard reaches your application? How the same code works for PS/2, USB, and Bluetooth keyboards? How touchpads detect gestures?

**You're about to find out!**

---

## What's This About?

This is **drivers/input/** - the unified framework for ALL human input devices!

Here's where it fits:

```
drivers/

04-device-drivers/

├── 00-foundation/      ← device registration in the kernel 
│   └── driver-core/    ← hardware exposed as files
└── 01-input/           ← input events to userspace  (THIS!)
|    ├── input.c        ← Core input layer
|    ├── evdev.c        ← Event device interface
|    ├── keyboard/      ← Keyboard drivers
|    ├── mouse/         ← Mouse drivers
|    ├── touchscreen/   ← Touchscreen drivers
|    └── joystick/      ← Joystick drivers
|
├── 02-block/           ← storage read and write
├── 03-net/             ← packet send and receive
├── 04-usb/             ← USB device communication
├── 05-pci/             ← PCIe device discovery
├── 06-char/            ← character device data flow
└── 07-gpu/             ← display and rendering pipeline

```

**This covers:**
- Unified input abstraction (PS/2, USB, Bluetooth all produce same events)
- Event-based interface (type, code, value)
- /dev/input/ filesystem (event0, event1, mice)
- Driver to application flow (hardware to screen)
- Multiple consumers (X server, console, applications)
- Keyboards, mice, touchpads, touchscreens

---

# The Fundamental Problem

**Physical Reality:**

```
Your computer has many input devices:

Keyboards:
├── PS/2 keyboard → Scan codes on I/O port 0x60
├── USB keyboard → HID reports via USB packets
└── Bluetooth keyboard → RFCOMM data over wireless

Mice:
├── PS/2 mouse → Different protocol than PS/2 keyboard
├── USB mouse → HID reports
└── Touchpad → Absolute coordinates + gestures

All different protocols, formats, interfaces
```

**The problem:**

```
Without unified abstraction:

X server must handle:
├── PS/2 keyboard scan codes (0x1E = 'A')
├── USB HID usage codes (0x04 = 'A')
├── Bluetooth keyboard data
├── PS/2 mouse deltas
├── USB mouse packets
└── Touchpad absolute positions

Every application reimplements device handling
Every device type needs different code
Maintenance nightmare for thousands of devices
```

**How do you provide ONE interface for ALL input devices?**

---

# Without an Input Subsystem

**Imagine no unified layer:**

```
Terminal application wants keyboard input:

Must implement:
├── PS/2 keyboard driver (read port 0x60, decode scan codes)
├── USB keyboard driver (parse HID reports)
├── Bluetooth keyboard driver (handle wireless protocol)
└── Serial keyboard driver (parse RS232 data)

For EVERY application
Thousands of lines of code duplicated
Device-specific bugs everywhere
No code reuse


X server needs mouse and keyboard:

Must handle:
├── 50+ keyboard types
├── 100+ mouse models
├── Touchpads from 20+ manufacturers
└── Each with unique quirks

10,000+ lines of device-specific code
Update one driver, must update X server
Cannot use standard tools (cat, dd)


Hotplug scenario:

USB keyboard plugged in:
└── Who detects it?
    Who loads the driver?
    How does X server know?
    No automatic mechanism

User must restart X server manually
Unplug and replug to make it work
Poor user experience
```

**No abstraction means chaos!**

---

# The Three-Layer Solution

**Universal event translation:**

```
Layer 1: HARDWARE DRIVERS (device-specific)
├── atkbd.c (PS/2 keyboard)
├── psmouse.c (PS/2 mouse)
├── usbhid.c (USB keyboards/mice)
├── synaptics.c (touchpad)
└── Many more...

These know:
├── Hardware protocols (PS/2, USB, Bluetooth)
├── Registers and I/O ports
└── IRQ handling

These produce: input_event structures


Layer 2: INPUT CORE (routing and matching)
├── Receives events from ALL drivers
├── Routes to ALL interested handlers
├── Manages device/handler connections
└── No hardware-specific code


Layer 3: HANDLERS (consumers)
├── evdev: Creates /dev/input/eventX (raw events)
├── kbd: Virtual console keyboard
├── mousedev: Legacy /dev/input/mice
└── joydev: Joystick interface

These consume: input_event structures


Result:
PS/2 'A' keypress → input_event(EV_KEY, KEY_A, 1)
USB 'A' keypress → input_event(EV_KEY, KEY_A, 1)
BT 'A' keypress → input_event(EV_KEY, KEY_A, 1)

Same event, different hardware
Applications see ONE format
```

---

# 1. The Event Structure

## One Format for Everything

**struct input_event:**

```c
struct input_event {
    struct timeval time;   // When did this happen?
    __u16 type;           // What TYPE of event?
    __u16 code;           // What specific event?
    __s32 value;          // What value?
};

Size: 24 bytes (on 64-bit)
Atomic unit of input
```

**Key press example:**

```
User presses 'A':

input_event {
    time = { tv_sec=1234567890, tv_usec=123456 }
    type = EV_KEY (1)
    code = KEY_A (30)
    value = 1  // 1=pressed, 0=released, 2=repeat
}

User releases 'A':

input_event {
    time = { tv_sec=1234567890, tv_usec=234567 }
    type = EV_KEY (1)
    code = KEY_A (30)
    value = 0  // Released
}
```

**Mouse movement example:**

```
User moves mouse 5 pixels right, 3 pixels down:

Event 1: { time, EV_REL, REL_X, +5 }
Event 2: { time, EV_REL, REL_Y, +3 }
Event 3: { time, EV_SYN, SYN_REPORT, 0 }

Synchronization event marks end of atomic group
All events between syncs happened simultaneously
```

**Event types:**

```
EV_SYN  (0x00) - Synchronization delimiter
EV_KEY  (0x01) - Key/button press or release
EV_REL  (0x02) - Relative movement (mouse delta)
EV_ABS  (0x03) - Absolute position (touchscreen)
EV_MSC  (0x04) - Miscellaneous
EV_SW   (0x05) - Switch (lid close, headphone jack)
EV_LED  (0x11) - LED control (Caps Lock indicator)
EV_SND  (0x12) - Sound (keyboard beep)
EV_REP  (0x14) - Key repeat settings
EV_FF   (0x15) - Force feedback (rumble)
EV_PWR  (0x16) - Power button special handling
```

---

# 2. Key Codes and Event Codes

## The Universal Key Language

**Common keyboard codes:**

```
Letter keys:
KEY_A = 30      KEY_B = 48      KEY_C = 46
KEY_Z = 44      KEY_SPACE = 57  KEY_ENTER = 28

Function keys:
KEY_F1 = 59     KEY_F2 = 60     KEY_F12 = 88

Modifiers:
KEY_LEFTSHIFT = 42    KEY_RIGHTSHIFT = 54
KEY_LEFTCTRL = 29     KEY_RIGHTCTRL = 97
KEY_LEFTALT = 56      KEY_RIGHTALT = 100

Special:
KEY_ESC = 1           KEY_BACKSPACE = 14
KEY_TAB = 15          KEY_CAPSLOCK = 58
KEY_UP = 103          KEY_DOWN = 108
KEY_LEFT = 105        KEY_RIGHT = 106
KEY_VOLUMEUP = 115    KEY_VOLUMEDOWN = 114
KEY_MUTE = 113        KEY_POWER = 116
```

**Mouse codes:**

```
Buttons:
BTN_LEFT = 272       // Left mouse button
BTN_RIGHT = 273      // Right mouse button
BTN_MIDDLE = 274     // Middle button/wheel click

Relative axes:
REL_X = 0            // Horizontal movement delta
REL_Y = 1            // Vertical movement delta
REL_WHEEL = 8        // Scroll wheel vertical
REL_HWHEEL = 11      // Horizontal scroll
```

**Touchscreen/touchpad codes:**

```
Absolute axes:
ABS_X = 0            // X position (0 to max_x)
ABS_Y = 1            // Y position (0 to max_y)
ABS_PRESSURE = 24    // Touch pressure
ABS_DISTANCE = 25    // Hover distance

Multitouch:
ABS_MT_POSITION_X = 53   // Finger X position
ABS_MT_POSITION_Y = 54   // Finger Y position
ABS_MT_TRACKING_ID = 57  // Which finger (finger 0, 1, 2...)
ABS_MT_PRESSURE = 58     // Individual finger pressure

Touch buttons:
BTN_TOUCH = 330          // Touching screen
BTN_TOOL_FINGER = 325    // Finger detected (not necessarily touching)
```

---

# 3. The Architecture

## How Components Connect

**input_dev (the device):**

```c
struct input_dev {
    const char *name;          // "AT Translated Set 2 keyboard"
    const char *phys;          // Physical path
    struct input_id id;        // Bus, vendor, product, version
    
    unsigned long evbit[NBITS(EV_MAX)];    // Supported event types
    unsigned long keybit[NBITS(KEY_MAX)];  // Supported keys
    unsigned long relbit[NBITS(REL_MAX)];  // Supported relative axes
    unsigned long absbit[NBITS(ABS_MAX)];  // Supported absolute axes
    
    unsigned long key[NBITS(KEY_MAX)];     // Current key states
    
    int (*open)(struct input_dev *dev);
    void (*close)(struct input_dev *dev);
    
    struct list_head h_list;   // Handler connections
    struct device dev;         // Device model integration
};

One input_dev = One physical device
Describes capabilities (what events it can generate)
Maintains current state (which keys currently pressed)
```

**input_handler (the consumer):**

```c
struct input_handler {
    const char *name;          // "evdev", "kbd", "mousedev"
    
    void (*event)(struct input_handle *, unsigned int, unsigned int, int);
    bool (*match)(struct input_handler *, struct input_dev *);
    int (*connect)(struct input_handler *, struct input_dev *,
                  const struct input_device_id *id);
    void (*disconnect)(struct input_handle *);
    
    const struct file_operations *fops;  // For /dev/input/
    int minor;                           // Base minor number
    
    struct list_head h_list;   // Connected devices
};

One handler type per consumer interface
evdev: Raw events to userspace
kbd: Virtual console keyboard handling
```

**input_handle (the connection):**

```c
struct input_handle {
    struct input_dev *dev;      // Points to device
    struct input_handler *handler;  // Points to handler
    
    struct list_head d_node;    // In device's handler list
    struct list_head h_node;    // In handler's device list
};

Links one device to one handler
Device can have multiple handles (multiple handlers)
Handler can have multiple handles (multiple devices)
```

**Connection flow:**

```
Kernel has two lists:

input_dev_list: [keyboard, mouse, touchpad, ...]
input_handler_list: [evdev, kbd, mousedev, ...]

New device registered:
└── For each handler:
    └── Does handler want this device?
        └── handler->match() checks
            If yes:
            └── handler->connect()
                Create input_handle
                Link device <-> handler

New handler registered:
└── For each device:
    └── Same matching process

Result: Dynamic mesh of connections
Each device connected to relevant handlers
```

---

# 4. The /dev/input/ Interface

## How Applications Access Events

**evdev handler:**

```
Creates one /dev/input/eventX per input device

/dev/input/
├── event0 → Keyboard (major=13, minor=64)
├── event1 → Mouse (major=13, minor=65)
├── event2 → Touchpad (major=13, minor=66)
├── event3 → Power button (major=13, minor=67)
├── event4 → USB keyboard (major=13, minor=68)
└── mice → All mice combined (major=13, minor=63)

Each eventX is a character device
Standard read/write/ioctl interface
```

**Reading events:**

```c
Application code:

int fd = open("/dev/input/event0", O_RDONLY);

struct input_event ev;
while (1) {
    ssize_t n = read(fd, &ev, sizeof(ev));
    if (n == sizeof(ev)) {
        if (ev.type == EV_KEY) {
            if (ev.code == KEY_A && ev.value == 1)
                printf("A key pressed\n");
            else if (ev.code == KEY_A && ev.value == 0)
                printf("A key released\n");
        }
    }
}

read() blocks until event available
Returns exactly one input_event structure
Application processes event
```

**Ring buffer:**

```
For each open /dev/input/eventX:

evdev maintains circular buffer:
├── Size: 64 events (1536 bytes)
├── head: Where driver writes
└── tail: Where application reads

Driver produces event:
└── evdev_event() called
    Write to buffer[head]
    head = (head + 1) % 64
    If head == tail: buffer full, drop event
    Wake up waiting readers

Application reads:
└── read() from buffer[tail]
    tail = (tail + 1) % 64
    Return event to userspace

Buffer full scenario:
└── Events dropped
    Special SYN_DROPPED event sent
    Application must resync device state
```

---

# 5. Complete Keypress Flow

## From Metal Contact to Screen

**Initial state:**

```
USB keyboard connected and recognized
usbhid driver loaded
/dev/input/event0 opened by X server
X server blocked in read(), waiting for input
System idle
```

**Step 1: Physical keypress**

```
User presses 'A' key:

Keycap pushes down
Spring compresses
Metal contacts close (physical switch)

Keyboard matrix scanning:
├── Row/column intersection detected
├── Keyboard controller registers keypress
└── Scan code determined: 0x04 (USB HID usage for 'A')

Time: 0 ms
```

**Step 2: USB transaction**

```
USB keyboard firmware:

Generate HID report:
├── Byte 0: Modifier keys (0x00 = none pressed)
├── Byte 1: Reserved (0x00)
├── Bytes 2-7: Up to 6 simultaneous keys
└── Report: [0x00, 0x00, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00]

USB host controller polls keyboard:
├── Interrupt endpoint polled every 1 ms
├── Or keyboard initiates transfer on interrupt IN endpoint
└── HID report transmitted over USB bus

Time: 0.5 ms
```

**Step 3: USB interrupt**

```
USB host controller:

Transfer complete
DMA buffer contains HID report
Raise IRQ 16 to CPU

CPU (TOPIC 6 - APIC):
├── APIC receives IRQ
├── Lookup interrupt vector in IDT
└── Call interrupt handler: xhci_irq()

xhci_irq() (USB controller driver):
├── Check transfer ring
├── Transfer complete for keyboard endpoint
├── Data available: HID report
└── Schedule bottom half (softirq or workqueue)

Time: 0.6 ms
```

**Step 4: USB-HID driver**

```
usbhid_irq() called:

Read HID report: [0x00, 0x00, 0x04, ...]

Parse HID report:
├── Byte 0: No modifiers
├── Byte 2: 0x04 = HID usage code

Translate HID to Linux keycode:
├── HID usage 0x04 (Keyboard a and A)
└── Linux keycode: KEY_A (30)

Compare with previous state:
├── Previous: KEY_A not pressed
├── Current: KEY_A pressed
└── State change detected

Call input layer:
input_report_key(input_dev, KEY_A, 1);

Time: 0.7 ms
```

**Step 5: Input core**

```
input_report_key() calls input_event():

input_event(dev, EV_KEY, KEY_A, 1)

Validate event:
├── Device supports EV_KEY? Check evbit
├── Device supports KEY_A? Check keybit
└── Both valid

Update device state:
└── dev->key[KEY_A] = 1 (mark as pressed)

Route to all connected handlers:
├── evdev_event(evdev_handle, EV_KEY, KEY_A, 1)
├── kbd_event(kbd_handle, EV_KEY, KEY_A, 1)
└── Each handler processes simultaneously

Driver sends sync:
input_sync(dev);
└── Generates EV_SYN/SYN_REPORT event
    Marks end of atomic event batch

Time: 0.8 ms
```

**Step 6: evdev handler**

```
evdev_event() called:

Create input_event structure:
event = {
    .time = { current_time },
    .type = EV_KEY,
    .code = KEY_A (30),
    .value = 1
};

Write to ring buffer:
├── buffer[head] = event
├── head = (head + 1) % 64
└── Buffer not full, event stored

Sync event arrives:
event = {
    .time = { current_time },
    .type = EV_SYN,
    .code = SYN_REPORT,
    .value = 0
};

Write sync to buffer

Wake up waiting readers:
wake_up_interruptible(&evdev->wait);

Time: 0.9 ms
```

**Step 7: X server wakes**

```
X server was blocked in read():

Kernel wakes X server:
├── Scheduler marks process runnable (TOPIC 4)
├── Next context switch: X server runs
└── read() system call returns

X server reads from /dev/input/event0:
├── copy_to_user() copies event from kernel buffer
├── Event 1: { EV_KEY, KEY_A, 1 }
└── Event 2: { EV_SYN, SYN_REPORT, 0 }

X server receives both events

Time: 1.5 ms
```

**Step 8: X server processes**

```
X server input handling:

Translate Linux keycode to X11 keycode:
├── KEY_A (30) → X11 keycode 38
└── Using keymap from xkbcomp

Apply modifiers:
├── Shift not pressed
└── Lowercase 'a'

Look up keysym:
├── Keycode 38 + no modifiers = keysym 'a'
└── Using XKB keyboard layout

Generate X11 event:
├── KeyPress event
├── keycode = 38
├── keysym = 'a'
└── state = 0 (no modifiers)

Send to focused window:
└── X protocol: send event to client

Time: 2.0 ms
```

**Step 9: Terminal emulator**

```
Terminal emulator receives X11 KeyPress:

Process key:
├── Keysym 'a' received
├── UTF-8 encode: 0x61
└── Character 'a'

Display on screen:
├── Look up glyph for 'a' in font
├── Render to framebuffer
└── Character appears at cursor position

Send to PTY (if typing in shell):
├── write(pty_fd, "a", 1)
└── Data goes to shell/application

Time: 2.5 ms
```

**Total time: About 2-3 milliseconds from keypress to character on screen**

---

# 6. PS/2 Keyboard

## The Legacy Interface

**PS/2 physical interface:**

```
PS/2 connector (6-pin mini-DIN):
├── Pin 1: Data
├── Pin 2: Not used
├── Pin 3: Ground
├── Pin 4: +5V power
├── Pin 5: Clock
└── Pin 6: Not used

Serial protocol:
├── Clock: Square wave generated by keyboard
├── Data: One bit per clock pulse
├── 11 bits per byte (start, 8 data, parity, stop)
└── Bidirectional (host can send to keyboard too)

Intel 8042 controller:
├── Legacy keyboard controller chip
├── I/O ports: 0x60 (data), 0x64 (command/status)
├── Generates IRQ 1 for keyboard
└── IRQ 12 for mouse
```

**Scan codes:**

```
Keyboard sends scan codes when keys pressed/released:

Make codes (key pressed):
0x01 = ESC
0x02 = 1
0x1E = A
0x1F = S
0x20 = D
0x2C = Z
0x1C = ENTER
0x39 = SPACE

Break codes (key released):
0x81 = ESC released (0x01 | 0x80)
0x9E = A released (0x1E | 0x80)

Extended codes (0xE0 prefix):
0xE0 0x48 = UP arrow
0xE0 0x50 = DOWN arrow
0xE0 0x4B = LEFT arrow
0xE0 0x4D = RIGHT arrow
```

**PS/2 keypress flow:**

```
Key pressed:

Metal contacts close
Keyboard controller detects
Scan code 0x1E sent serially on data line
11 bits transmitted (about 1 ms)

8042 controller receives:
├── Assembles bits into byte
├── Stores in buffer at I/O port 0x60
└── Raises IRQ 1

CPU handles IRQ 1:
IRQ handler: atkbd_interrupt()

Read from port 0x60:
asm("inb $0x60, %al");
└── Returns: 0x1E

Look up in translation table:
├── Scan code 0x1E → KEY_A
└── Report to input layer

input_report_key(kbd_dev, KEY_A, 1);
input_sync(kbd_dev);

Same path as USB after this point
evdev receives event
Application gets input_event structure
```

---

# 7. Mouse Input

## Relative Movement

**Mouse events:**

```
Mouse generates relative deltas:

Move mouse right 5 pixels:
├── Event: { EV_REL, REL_X, +5 }
├── Event: { EV_REL, REL_Y, 0 }
└── Event: { EV_SYN, SYN_REPORT, 0 }

Move mouse down 3 pixels:
├── Event: { EV_REL, REL_X, 0 }
├── Event: { EV_REL, REL_Y, +3 }
└── Sync

Scroll wheel up:
├── Event: { EV_REL, REL_WHEEL, +1 }
└── Sync

Left button press:
├── Event: { EV_KEY, BTN_LEFT, 1 }
└── Sync

Left button release:
├── Event: { EV_KEY, BTN_LEFT, 0 }
└── Sync
```

**Why relative, not absolute?**

```
Mouse doesn't know screen dimensions
Mouse sensor measures movement, not position
Reports delta since last reading

Cursor position maintained by:
├── X server (graphical mode)
└── GPM daemon (console mode)

They integrate deltas:
cursor_x += event.value (for REL_X)
cursor_y += event.value (for REL_Y)

Clamp to screen boundaries:
if (cursor_x < 0) cursor_x = 0;
if (cursor_x > screen_width) cursor_x = screen_width;
```

**Mouse polling rate:**

```
USB mouse:
├── Default: 125 Hz (8 ms interval)
├── Gaming mice: Up to 8000 Hz (0.125 ms interval)
└── Higher polling = lower latency, more CPU overhead

PS/2 mouse:
├── Interrupt-driven (not polled)
├── Sends packet when movement occurs
└── Slightly lower latency than low-rate USB
```

---

# 8. Touchpad

## Absolute Position with Gestures

**Touchpad as absolute device:**

```
Unlike mouse, touchpad knows finger position absolutely:

Capacitive sensing grid:
├── Array of sensors (e.g., 100x80)
├── Finger changes capacitance
├── Controller measures each sensor
└── Calculates finger position

Reports EV_ABS events:

Finger touches:
├── { EV_KEY, BTN_TOUCH, 1 }
├── { EV_ABS, ABS_X, 4500 }
├── { EV_ABS, ABS_Y, 3200 }
├── { EV_ABS, ABS_PRESSURE, 80 }
└── { EV_SYN, SYN_REPORT, 0 }

Finger moves:
├── { EV_ABS, ABS_X, 4550 }
├── { EV_ABS, ABS_Y, 3200 }
└── Sync

Finger lifts:
├── { EV_KEY, BTN_TOUCH, 0 }
└── Sync
```

**Multitouch:**

```
Two fingers touching:

Finger 1:
├── { EV_ABS, ABS_MT_TRACKING_ID, 0 }
├── { EV_ABS, ABS_MT_POSITION_X, 4000 }
├── { EV_ABS, ABS_MT_POSITION_Y, 3000 }
└── { EV_ABS, ABS_MT_PRESSURE, 75 }

Finger 2:
├── { EV_ABS, ABS_MT_TRACKING_ID, 1 }
├── { EV_ABS, ABS_MT_POSITION_X, 5000 }
├── { EV_ABS, ABS_MT_POSITION_Y, 3000 }
└── { EV_ABS, ABS_MT_PRESSURE, 60 }

Sync:
└── { EV_SYN, SYN_REPORT, 0 }

Both fingers reported in one batch
Tracking IDs distinguish fingers
```

**libinput processing:**

```
Modern Linux input stack:

Kernel driver:
├── Reports raw touchpad data
├── Finger positions, pressure
└── Button states

libinput (userspace library):
├── Reads /dev/input/eventX
├── Processes raw data
├── Implements:
│   ├── Palm rejection
│   ├── Tap-to-click
│   ├── Two-finger scrolling
│   ├── Three-finger gestures
│   ├── Pinch-to-zoom
│   └── Edge scrolling
└── Outputs processed events to compositor

Why userspace?
├── Complex algorithms easier to update
├── User-configurable settings
├── No kernel recompile needed
└── Can update without rebooting

X server or Wayland compositor receives processed events
```

---

# 9. Driver Registration

## How Drivers Report Events

**Registering an input device:**

```c
Driver initialization:

Step 1: Allocate input_dev
struct input_dev *input = input_allocate_device();

Step 2: Fill in information
input->name = "AT Translated Set 2 keyboard";
input->phys = "isa0060/serio0/input0";
input->id.bustype = BUS_I8042;
input->id.vendor = 0x0001;
input->id.product = 0x0001;
input->id.version = 0x0100;

Step 3: Set capabilities
set_bit(EV_KEY, input->evbit);   // Can generate key events
set_bit(EV_REP, input->evbit);   // Supports key repeat

// Declare which keys this device can generate
for (i = KEY_ESC; i < KEY_UNKNOWN; i++)
    set_bit(i, input->keybit);

// Set key repeat parameters
input->rep[REP_DELAY] = 250;    // 250ms before repeat starts
input->rep[REP_PERIOD] = 33;    // 33ms between repeats (30 Hz)

Step 4: Set callbacks
input->open = atkbd_open;       // Called when device opened
input->close = atkbd_close;     // Called when device closed

Step 5: Register
int err = input_register_device(input);

Input core:
├── Add to input_dev_list
├── Match with handlers
│   └── evdev connects: creates /dev/input/event0
├── Send uevent
└── udev creates device node with proper permissions

Device ready for use
```

**Reporting events from driver:**

```c
Interrupt handler receives key press:

Key 'A' pressed:
input_report_key(input, KEY_A, 1);
input_sync(input);

Key 'A' released:
input_report_key(input, KEY_A, 0);
input_sync(input);

Mouse moved:
input_report_rel(input, REL_X, delta_x);
input_report_rel(input, REL_Y, delta_y);
input_sync(input);

Touchscreen touched:
input_report_abs(input, ABS_X, x_position);
input_report_abs(input, ABS_Y, y_position);
input_report_key(input, BTN_TOUCH, 1);
input_sync(input);

Always call input_sync() after events
Marks end of atomic event batch
```

---

# 10. Physical Reality

## Hardware to Software Timing

**Timing breakdown:**

```
T=0.0 ms: Key pressed (physical contact)

T=0.1 ms: Keyboard controller detects
└── Internal scanning circuit reads matrix
    Debouncing logic confirms stable press

T=0.5 ms: USB polling / PS/2 IRQ
└── USB: Host controller polls endpoint
    PS/2: IRQ 1 fires immediately

T=0.6 ms: CPU interrupt handler runs
└── Context switch to interrupt handler (TOPIC 4)
    xhci_irq() or atkbd_interrupt()

T=0.7 ms: Driver processes, calls input_report_key()
└── usbhid parses HID report
    atkbd translates scan code
    Both produce same: input_event(EV_KEY, KEY_A, 1)

T=0.8 ms: evdev writes to ring buffer
└── Event stored in kernel memory
    wake_up_interruptible() called

T=0.9 ms: Scheduler wakes X server
└── X server marked runnable
    Next timeslice: X server runs

T=1.5 ms: X server reads event
└── read() syscall (TOPIC 1)
    copy_to_user() transfers event
    X server has input_event structure

T=2.0 ms: X server processes and forwards
└── Translate keycode
    Apply keyboard layout
    Send X11 event to focused window

T=2.5 ms: Character appears on screen
└── Terminal emulator renders glyph
    Framebuffer updated
    Monitor displays character

Total latency: 2-3 milliseconds
```

**Memory layout:**

```
evdev ring buffer:

Located in kernel memory:
├── Physical RAM page (e.g., 0x1A2B3000)
├── Virtual kernel address: 0xFFFF888...
└── Size: 64 × 24 bytes = 1536 bytes

Structure:
events[64] = {
    [0] = { time, type, code, value },
    [1] = { ... },
    ...
    [63] = { ... }
};

head index: Next write position
tail index: Next read position

When full:
└── head == tail
    Drop incoming events
    Report SYN_DROPPED to application
```

---

# 11. Connections to Other Topics

## Input Subsystem in Context

**Connection to Interrupts:**

```
TOPICS 3 & 6: Interrupt handling, APIC

Input subsystem relies on interrupts:
├── IRQ 1: PS/2 keyboard
├── IRQ 12: PS/2 mouse
├── USB IRQs: USB keyboards/mice
└── GPIO IRQs: Power button, touchscreen

Without interrupts:
└── Must poll continuously (wasteful)
    Or miss events entirely
    Interrupts essential for responsive input
```

**Connection to Driver Core:**

```
TOPIC 16: file_operations, cdev, /dev/

evdev uses driver core mechanisms:
├── /dev/input/eventX are character devices
├── major=13, minor=64+ for event devices
├── struct file_operations evdev_fops
├── evdev_read() implements read()
├── evdev_ioctl() implements ioctl()
└── Standard VFS path: open/read/close

Input subsystem builds on driver infrastructure
```

**Connection to Device Model:**

```
TOPIC 15: Bus/Device/Driver model

input_dev contains struct device:
├── Shows in /sys/class/input/
├── Sends uevents on plug/unplug
├── udev creates /dev/input/ entries
└── Hotplug works automatically

Example /sys/ layout:
/sys/class/input/
├── event0 → /sys/devices/.../input/input0/event0
├── event1 → /sys/devices/.../input/input1/event1
└── mice → /sys/devices/.../input/mice

Device model integration essential
```

**Connection to Scheduling:**

```
TOPIC 4: Context switching, wait queues

evdev_read() uses wait queues:

Application calls read():
└── If no events available:
    wait_event_interruptible(evdev->wait, ...)
    Process sleeps

Event arrives:
└── wake_up_interruptible(&evdev->wait)
    Scheduler makes process runnable
    read() returns with event

Efficient: CPU free while waiting
No busy-waiting
```

---

# Summary

## What You've Mastered

**You now understand the input subsystem:**

```
What it is: Unified layer for ALL input devices
Why needed: Hardware diversity, multiple consumers
Three layers: Drivers → Core → Handlers
Event structure: type, code, value + timestamp
Complete flow: Keypress to screen in 2-3 ms
Device types: Keyboard, mouse, touchpad, touchscreen
Physical reality: IRQ timing, ring buffer, polling rates
Connections: Interrupts, driver core, device model, scheduling
```

---

## Key Takeaways

**Unified abstraction:**

```
Many hardware types:
├── PS/2 keyboards (scan codes, port 0x60)
├── USB keyboards (HID reports, USB packets)
└── Bluetooth keyboards (wireless protocol)

All produce same output:
└── input_event(EV_KEY, KEY_A, 1)

Single format for applications
Hardware differences hidden
```

**Three-layer architecture:**

```
Drivers (hardware-specific):
└── atkbd, usbhid, psmouse
    Know hardware protocols
    Translate to input_event

Input core (routing):
└── input.c
    Routes events to handlers
    No hardware knowledge

Handlers (consumers):
└── evdev, kbd, mousedev
    Process events
    Provide interfaces to userspace
```

**Event-based interface:**

```
Atomic events:
├── Each event: type + code + value
├── Batched with EV_SYN marker
└── Timestamp for precise ordering

Examples:
├── Key: EV_KEY, KEY_A, 1 (pressed)
├── Mouse: EV_REL, REL_X, +5 (moved right)
└── Touch: EV_ABS, ABS_X, 4500 (position)
```

**Multiple consumers:**

```
One device, many handlers:

Physical keyboard
├── evdev: /dev/input/event0 (X server reads)
├── kbd: Virtual console keyboard handling
└── All receive same events simultaneously

Dynamic connections:
└── Handlers register/unregister
    Devices hotplug
    System adapts automatically
```

---

## The Big Picture

**What input subsystem enables:**

```
Unified interface:
└── cat /dev/input/event0 works for any keyboard

Standard tools:
└── evtest, libinput debug-events

Hotplug support:
└── Plug USB keyboard, instantly works

Multiple consumers:
└── X server and console both get events

Hardware abstraction:
└── Replace PS/2 with USB, no application changes

This is why "everything just works" in modern Linux
```

---

## You're Ready!

With this knowledge, you can:
- Understand how every keypress reaches applications
- Debug input device issues
- Write input device drivers
- Process raw input events
- Understand touchpad gesture recognition
- Build input-handling applications

> **The input subsystem is the bridge from your fingers to the screen - you now know how Linux makes all input devices speak one language.**

---

**The universal input translator.**
