# Understanding Virtio and PCIe Interface

**Date:** 2026-01-25
**Session ID:** virtio_pcie_intro
**Tags:** `Virtualization, I/O devices, Memory`

---

## Conclusion (5 sentences)

Virtio is a standardized way for virtual machines to talk to their host's devices efficiently, like having a universal language between guest and host. PCIe (PCI Express) is the "delivery method" that virtio uses - it's like a standardized mailbox system that all computers understand. When you write a virtio-PCIe interface, you're building both the mailbox (PCIe config space and BARs) and the message format (virtqueues) so the guest can send I/O requests to the host. The guest sees what looks like a real PCIe device, but it's actually your emulated device intercepting those accesses. Your codebase already implements this beautifully with virtio-blk and virtio-net devices!

---

## 🎭 The Big Picture: What is Virtio?

**Metaphor: The Restaurant with a Standard Menu**

Imagine you're traveling to different countries. Each restaurant has different ways to order food - some use buttons, some use voice, some use hand signals. Confusing, right?

**Virtio is like having McDonald's everywhere** - no matter which country (host operating system) you visit, you know exactly how to order (make I/O requests). The menu is standardized, the ordering process is the same.

In virtualization:
- **Guest OS** = You (the customer)
- **Host/Hypervisor** = The kitchen
- **Virtio** = The standardized ordering system
- **Your I/O request** = Your order

Without virtio, each hypervisor (VMware, KVM, QEMU) would need custom drivers. With virtio, one driver works everywhere!

---

## 🔌 Why PCIe for Virtio?

**Metaphor: The Universal Power Outlet**

Think about electrical outlets. Different countries have different shapes - you need adapters! But imagine if there was ONE universal outlet everyone agreed to use.

**PCIe is that universal outlet for computer devices.** Every operating system (Windows, Linux, macOS) already knows how to talk to PCIe devices. They have built-in drivers for the PCIe bus!

So when we implement **virtio over PCIe (virtio-pci)**, we get:
1. ✅ Guest OS already knows how to discover PCIe devices
2. ✅ No need to modify the guest OS kernel
3. ✅ Works on x86, ARM, any architecture with PCIe support
4. ✅ Standardized configuration space (256 bytes every PCIe device has)

---

## 📬 How Virtio-PCI Works: The Mailbox System

**Metaphor: An Apartment Building's Mailbox System**

Imagine an apartment building with mailboxes:

1. **Configuration Space** (256 bytes) = The building directory
   - Lists which apartment (device) you are
   - Your name (Vendor ID: 0x1AF4 for Red Hat)
   - Your apartment number (Device ID: 0x1041 for network, 0x1042 for block)
   - Which mailbox slots you own (BARs - Base Address Registers)

2. **BARs (Base Address Registers)** = Your personal mailbox locations
   - Each device gets up to 6 mailboxes (BAR0-BAR5)
   - Each mailbox is at a specific address in memory or I/O space
   - You tell the system "My mailbox for notifications is at address 0xFEBC0000"

3. **Virtqueues** = The actual letters/packages in the mailboxes
   - Guest puts requests in one queue (like outgoing mail)
   - Host puts responses in same queue (like receiving mail)
   - Ring buffer structure - circular, so we reuse the same space

---

## 🏗️ The Five Key Virtio-PCI Structures

Looking at the code (`virtio-pci.c`), virtio defines **5 capability structures**. Think of these as different departments in a post office:

```
┌─────────────────────────────────────┐
│  PCI Configuration Space (256B)     │
│  ┌───────────────────────────────┐  │
│  │ Vendor ID: 0x1AF4            │  │ ← "I'm a virtio device"
│  │ Device ID: 0x1041/0x1042     │  │ ← "I'm network/block"
│  │ Command Register             │  │ ← "Am I enabled?"
│  │ BAR0, BAR1, BAR2...          │  │ ← "My mailbox addresses"
│  │ Capability Pointer → 0x40    │  │ ← "Start of capability list"
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│  Virtio Capabilities (5 types)      │
├─────────────────────────────────────┤
│ 1️⃣ COMMON Config                    │ ← Device/queue setup
│    - Device status                  │
│    - Feature negotiation            │
│    - Queue selection                │
├─────────────────────────────────────┤
│ 2️⃣ NOTIFY                           │ ← "Ring the doorbell!"
│    - Guest writes here to notify    │
├─────────────────────────────────────┤
│ 3️⃣ ISR (Interrupt Status)           │ ← "Why did you interrupt me?"
├─────────────────────────────────────┤
│ 4️⃣ DEVICE Config                    │ ← Device-specific settings
│    - Disk capacity (for blk)        │
│    - MAC address (for net)          │
├─────────────────────────────────────┤
│ 5️⃣ PCI Config Access                │ ← Alternative config method
└─────────────────────────────────────┘
```

---

## 💡 How a Virtio-PCI Transaction Works

Let's trace a **disk read request** through the code:

### Step 1: Guest Discovers Device
```c
// Guest scans PCI bus (writes to 0xCF8, reads from 0xCFC)
// Finds: Vendor 0x1AF4, Device 0x1042 → "Aha! Virtio block device!"
```

### Step 2: Guest Reads Capabilities
```c
// Guest: "Where are your mailboxes?"
// Host (code in virtio_pci_do_io): Returns BAR addresses
// BAR0 → 0xFEBC0000 (Common config at offset 0x0000)
//      → Notify at offset 0x3000
//      → ISR at offset 0x3004
//      → Device config at offset 0x3008
```

### Step 3: Feature Negotiation
```c
// Guest: "What features do you support?"
// Host: "I support packed virtqueues (VIRTIO_F_RING_PACKED)"
// Guest: "OK, I'll use packed format!"
```

### Step 4: Queue Setup
```c
// Guest writes to common config area:
virtio_pci_common_cfg->queue_select = 0;  // Select queue 0
virtio_pci_common_cfg->queue_size = 128;  // 128 descriptors
virtio_pci_common_cfg->queue_desc = 0x20000000;  // Desc addr
virtio_pci_common_cfg->queue_driver = 0x20001000;  // Driver addr
virtio_pci_common_cfg->queue_device = 0x20002000;  // Device addr
virtio_pci_common_cfg->queue_enable = 1;  // Enable!
```

### Step 5: Guest Sends I/O Request
```c
// Guest builds descriptor chain in virtqueue:
desc[0] → header (read sector 100)
desc[1] → data buffer (4KB, write-only)
desc[2] → status byte (write-only)

// Guest rings doorbell (writes to notify address)
// KVM traps this → vm_handle_mmio() → virtio_pci_do_io()
// → ioeventfd triggers → virtio_blk_worker_thread() wakes up
```

### Step 6: Host Processes Request
```c
// virtio_blk_worker_thread():
1. Read descriptor chain from guest memory
2. lseek(disk_fd, sector * 512, SEEK_SET)
3. read(disk_fd, guest_buffer, 4096)
4. Write status = VIRTIO_BLK_S_OK to guest memory
5. Trigger interrupt via irqfd → guest gets IRQ
```

### Step 7: Guest Receives Completion
```c
// Guest IRQ handler:
// - Reads ISR register (was interrupt from device? Yes!)
// - Checks virtqueue for completed descriptors
// - Processes data, marks request complete
```

---

## 🔑 Key Concepts in the Codebase

### 1. Configuration Space (pci.c:pci_dev_do_io)
Every PCIe device has a 256-byte config space. Like a business card:
```c
#define PCI_CFG_SPACE_SIZE 256
struct pci_dev {
    uint8_t config_space[PCI_CFG_SPACE_SIZE];
    // ...
};
```

Important registers:
- **0x00-0x01**: Vendor ID (0x1AF4 = Red Hat/Virtio)
- **0x02-0x03**: Device ID (0x1042 = virtio-blk)
- **0x04-0x05**: Command register (enable memory/IO)
- **0x10-0x27**: BAR0-BAR5 (6 base address registers)
- **0x34**: Capability pointer (start of capability list)

### 2. BARs - Base Address Registers (pci.c:pci_dev_activate_bar)
Think of BARs as parking spaces. Each device can reserve up to 6 spaces:
```c
#define PCI_NUM_BARS 6
```

When guest writes to a BAR:
1. Code calculates the size (must be power-of-2)
2. Registers that memory region on the MMIO bus
3. Future accesses to that address → routed to the device

```c
pci_dev_activate_bar(dev, bar_num);
// → bus_add_dev(&mmio_bus, &dev->bar_region[bar_num])
```

### 3. Virtqueues (virtq.c)
The actual data transfer mechanism. Like a circular conveyor belt:

```c
#define VIRTQ_SIZE 128  // 128 slots on the belt

struct virtq {
    struct packed_desc ring[VIRTQ_SIZE];  // The descriptors
    uint16_t avail_wrap_count;  // Did we wrap around?
    uint16_t used_wrap_count;
    // ...
};
```

**Packed format** is more efficient than old "split" format:
- One ring instead of three
- Better cache locality
- Wrap counter instead of flags

### 4. IOEventfd / IRQfd (vm.c)
**Metaphor: Automatic door sensors**

- **IOEventfd**: Motion sensor on guest→host door
  - Guest writes to notify address
  - KVM writes to eventfd
  - Worker thread wakes up (no VM exit needed in fast path!)

```c
vm_ioeventfd_register(vm, notify_addr, &blk->ioeventfd, 2);
// When guest writes notify_addr → ioeventfd gets signaled
```

- **IRQfd**: Doorbell from host→guest
  - Write to eventfd
  - KVM injects interrupt to guest
  - Guest IRQ handler runs

```c
vm_irqfd_register(vm, IRQ_NUM, &blk->irqfd);
// When you write to irqfd → guest gets interrupt
```

---

## 🎓 Flash Cards (Quiz Yourself!)

### Q1: What are the three main components of a virtio-PCI device?

**Answer:**
1. **PCI Configuration Space** (256 bytes) - Device identity and BARs
2. **Virtio Capabilities** (5 types) - Common config, Notify, ISR, Device config, PCI config
3. **Virtqueues** - Actual data transfer rings between guest and host

---

### Q2: What is the purpose of a BAR (Base Address Register)?

**Answer:**
A BAR tells the guest OS where in memory (or I/O space) to access this device's registers. It's like a parking space assignment - "my mailbox is at address 0xFEBC0000, size 16KB."

---

### Q3: How does the guest notify the host that it has new work in the virtqueue?

**Answer:**
The guest writes to the **notify address** (configured via the Notify capability). This triggers an **ioeventfd**, which wakes up the host's worker thread to process the virtqueue requests.

---

### Q4: How does the host notify the guest that a request is complete?

**Answer:**
The host writes to an **irqfd** (interrupt eventfd), which causes KVM to inject an interrupt into the guest. The guest's interrupt handler then checks the virtqueue for completed requests.

---

### Q5: Why use packed virtqueues instead of split virtqueues?

**Answer:**
**Packed virtqueues** are more efficient because:
- Single ring instead of three separate rings (descriptor, available, used)
- Better CPU cache locality (less memory scattered)
- Wrap counter makes it easier to detect new entries
- Defined in Virtio 1.1 spec as an optimization

---

## 📚 References

**Key Files in Codebase:**
- `src/pci.h`, `src/pci.c` - PCI/PCIe infrastructure
- `src/virtio-pci.h`, `src/virtio-pci.c` - Virtio PCI transport
- `src/virtio-blk.h`, `src/virtio-blk.c` - Virtio block device
- `src/virtio-net.h`, `src/virtio-net.c` - Virtio network device
- `src/virtq.h`, `src/virtq.c` - Virtqueue implementation
- `src/bus.h`, `src/bus.c` - Device bus abstraction
- `src/vm.h`, `src/vm.c` - VM core functionality

**Specifications:**
- Virtio 1.0/1.1 Specification: https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.html
- PCI Local Bus Specification
- PCIe Base Specification

---

**End of Session**
