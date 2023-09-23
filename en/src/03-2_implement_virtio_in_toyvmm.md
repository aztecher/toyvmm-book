# Implementing Virtio in ToyVMM

In this section, we will delve into the implementation of Virtio in ToyVMM. There are three main topics covered in this discussion:

1. Implementation of Virtqueue
2. Implementation of lightweight notifications between the guest and host using irqfd and ioeventfd
3. Implementation of the MMIO Transport

As mentioned in the [previous section](./03-1_virtio.md), ToyVMM initially utilizes MMIO as the transport method for Virtio. Before diving into the detailed explanation, let's start by illustrating an overview of the Virtio implementation in this context.

<div align="center">
<img src="../03_figs/virtio_implementation.svg" width="100%">
</div>

By referring to this diagram as needed, we can better understand the explanations and code that follow.

### Implementation Approach

In the implementation, the VirtioDevice itself is implemented as an abstract concept (`Trait`), and concrete devices like `Net` and `Block` are created to fulfill this trait. Similarly, since there are options for transport methods like `PCI` and `MMIO` (with MMIO being used here), we treat transport as an abstract concept and implement it according to the specific implementation, which in this case is `MMIO`.

<div align="center">
<img src="../03_figs/virtio_related_structs.svg" width="50%">
</div>

Finally, we need to implement Virtqueues. While the number and usage of Virtqueues can vary depending on the implemented Virtio device, the structure of the Virtqueue remains consistent. We'll provide more details on this later.

### Virtqueue Implementation

#### Virtqueue Deep-Dive

Before delving into the implementation of Virtqueue, let's gain a more detailed understanding of the typical Virtqueue structure. A Virtqueue is composed of three main elements: the `Descriptor Table`, the `Available Ring`, and the `Used Ring`. Here's what each of them does:

* `Descriptor Table` : A table that holds entries (`Descriptor`) that store information such as addr and size of data to be shared between Host and Guest.  
* `Available Ring` : Structure that manages `Descriptor` that stores information that the Guest wants to notify to the Host.
* `Used Ring` : Structure that manages `Descriptor` that stores information that the Host wants to notify to the Guest.

<div align="center">
<img src="../03_figs/virtqueue.svg" width="80%">
</div>


We'll explore each of these elements in detail while understanding how they cooperate. First, the `Descriptor Table` contains data structures like `Descriptor` (as indicated in the diagram) gathered together.

> ```C
> struct virtq_desc { 
>         /* Address (guest-physical). */ 
>         le64 addr; 
>         /* Length. */ 
>         le32 len; 
>  
> /* This marks a buffer as continuing via the next field. */ 
> #define VIRTQ_DESC_F_NEXT   1 
> /* This marks a buffer as device write-only (otherwise device read-only). */ 
> #define VIRTQ_DESC_F_WRITE     2 
> /* This means the buffer contains a list of buffer descriptors. */ 
> #define VIRTQ_DESC_F_INDIRECT   4 
>         /* The flags as indicated above. */ 
>         le16 flags; 
>         /* Next field if flags & NEXT */ 
>         le16 next; 
> };
> ```
> Source: [2.7.5 The Virtqueue Descriptor Table](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-430005)

A `Descriptor` represents the data to be transferred and the location of the next data in the chain.
* `addr` is the actual address of the data (guest's physical address), and the length of the data can be obtained from `len`.
* `flags` provide information about whether there is a next descriptor, whether it's write-only, and other flags.
* `next` indicates the number of the next descriptor, allowing Descriptor Table to be processed sequentially.

Usually, one Descriptor is used to send one piece of data. It's important to note that even if you allocate contiguous memory in virtual address space, if the physical addresses are not contiguous, each Descriptor will be needed for each physical page, resulting in multiple Descriptors being sent sequentially.

Next is the `Available Ring`. The `Available Ring` is structured as follows:

> ```C
> struct virtq_avail { 
> #define VIRTQ_AVAIL_F_NO_INTERRUPT      1 
>         le16 flags; 
>         le16 idx; 
>         le16 ring[ /* Queue Size */ ]; 
>         le16 used_event; /* Only if VIRTIO_F_EVENT_IDX */ 
> }
> ```
> Source: [2.7.6 The Virtqueue Available Ring](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-490006)

The `Available Ring` is used to specify the Descriptors that need to be notified from the guest to the host.
* `flags` are used for temporary interrupt suppression and other purposes.
* `idx` points to the index of the newest entry in the `ring`.
* `ring` is the actual ring body, holding Descriptor numbers.
* `used_event` is also used for interrupt suppression but is only necessary if `VIRTIO_F_EVENT_IDX` is enabled.

The guest writes the location of the actual data to the Descriptor and the index information to the `Available Ring` (specifically in the `ring` field). It's important to note that the host needs to remember the index of the last processed `ring`. The guest can only provide information about the current state of the ring and the latest index (`idx` field). Therefore, the host compares the last processed entry number with the latest index information (`idx`) and checks for any differences (indicating new entries). If there are differences, it means there are new entries to process. The host then refers to the ring and retrieves the Descriptor index based on the index difference, obtains the data from the Descriptor, and processes it accordingly, depending on the specific device's implementation.

Finally, there's the `Used Ring`, which is the reverse of the `Available Ring`, meaning it's used to specify Descriptors that need to be notified from the host to the guest.

> ```C
> struct virtq_used { 
> #define VIRTQ_USED_F_NO_NOTIFY  1 
>         le16 flags; 
>         le16 idx; 
>         struct virtq_used_elem ring[ /* Queue Size */]; 
>         le16 avail_event; /* Only if VIRTIO_F_EVENT_IDX */ 
> }; 
>  
> /* le32 is used here for ids for padding reasons. */ 
> struct virtq_used_elem { 
>         /* Index of start of used descriptor chain. */ 
>         le32 id; 
>         /* 
>          * The number of bytes written into the device writable portion of 
>          * the buffer described by the descriptor chain. 
>          */ 
>         le32 len; 
> };
> ```
> Source: [2.7.8 The Virtqueue Used Ring](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-540008)

* `flags` are used for temporary interrupt suppression and other purposes.
* `idx` points to the index of the newest entry in the `ring`.
* `ring` is the actual ring body, holding Descriptor numbers.
* `used_event` is also used for interrupt suppression but is only necessary if `VIRTIO_F_EVENT_IDX` is enabled.

When returning notifications from the host to the guest, the descriptor is used to inform the guest of the location of the data to be returned, corresponding to the reply data. The index of the descriptor is stored in the `ring` of the `Used Ring` and the `idx` value is updated to point to the newest index in the `ring` before returning control to the guest.

However, unlike the `Available Ring` the elements of the `ring` are accompanied by a structure (`virtq_used_elem`).

- `id` is the head entry of the descriptor chain (the same as `virtq_avail.idx`).
- `len` stores information such as the total amount of I/O performed on the descriptor chain referred to by `id` on the host side.

The following diagram summarizes what has been explained so far.

<div align="center">
<img src="../03_figs/virtqueue_desc_avail_used_flow.svg" width=100%>
</div>

This concludes the necessary knowledge for implementing Virtqueue.

# Virtqueue implementation on ToyVMM

In ToyVMM, the implementation of Virtqueues is located in `queue.rs`.

The concrete addresses of the `Descriptor Table`, `Available Ring`, and `Used Ring` in guest memory are configured through interactions with the guest-side Device Driver during the guest VM startup. While we'll delve into this exchange as we peek into actual I/O requests from the guest, for now, let's focus on this fact.

ToyVMM needs to perform address accesses based on the specific starting addresses and Virtio specifications. In essence, it operates on a per-descriptor basis (where each descriptor points to the address of the actual data). During data processing, it updates the `Available Ring` and `Used Ring`.

Now, let's explore the code. The `Queue` structure in ToyVMM represents a Virtqueue and is defined as follows:

```Rust
#[derive(Clone)]
/// A virtio queue's parameters
pub struct Queue {
    /// The maximal size in elements offered by the device
    max_size: u16,

    /// The queue size in elements the driver selected
    pub size: u16,

    /// Indicates if the queue is finished with configuration
    pub ready: bool,

    /// Guest physical address of descriptor table
    pub desc_table: GuestAddress,

    /// Guest physical address of the available ring
    pub avail_ring: GuestAddress,

    /// Guest physical address of the used ring
    pub used_ring: GuestAddress,

    next_avail: Wrapping<u16>,
    next_used: Wrapping<u16>,
}
```

In this structure, you can see the definitions for the `Descriptor Table`, `Available Ring`, and `Used Ring`, which represent the specific addresses in guest memory. These addresses are initialized during interactions with the guest's Device Driver, as mentioned earlier. From ToyVMM's perspective, these are merely physical memory addresses belonging to the guest, and ToyVMM accesses them based on Virtio specifications.

Now, let's delve into address access using the code.

ToyVMM abstracts the sequence of operations to get `Descriptor` from the state of `Available Ring` is hiddden as Virtqueue iteration, and in actual device implementations that utilize Virtqueues, you will find code structured like this:

```rust
// queue: 'Queue' struct
// desc_chain: 'DescriptorChain' struct
for desc_chain in queue.iter(mem) {
    // 'desc_chain' contains the 'addr,' 'len,' 'flags,' and 'next' values of the descriptor
    // Behind the iteration, data related to 'queue.avail_ring' is adjusted.
}
```

Behind the scenes of this iteration, let's explain what's happening. First, the `iter` function is implemented in the `Queue` structure, and it creates an `AvailIter` structure. To create this `AvailIter`, it fetches the latest `idx` in the `Available Ring` from the `GuestMemory` and the `avail_ring`'s starting address.

```Rust
/// A consuming iterator over all available descriptor chain heads offered by the driver
pub fn iter<'a, 'b>(&'b mut self, mem: &'a GuestMemoryMmap) -> AvailIter<'a, 'b> {
    ... // validation codes
    let queue_size = self.actual_size();
    let avail_ring = self.avail_ring;

    // Access the 'idx' fields of available ring
    // skip 2 bytes (= u16 / 'flags' member) from avail_ring address
    // and get 2 bytes (= u16 / 'idx' member representing the newest index of avail_ring) from that address.
    let index_addr = mem.checked_offset(avail_ring, 2).unwrap();
    let last_index: u16 = mem.read_obj(index_addr).unwrap();

    AvailIter {
        mem,
        desc_table: self.desc_table,
        avail_ring: self.avail_ring,
        next_index: self.next_avail,
        last_index: Wrapping(last_index),
        queue_size,
        next_avail: &mut self.next_avail,
    }
}
```

As you can see, the `iter` function returns an `AvailIter`. Inside the `next` function of `AvailIter`, if `self.next_index` equals `self.last_index`, it returns `None`, indicating the end of iteration. The `next_index` tracks the processed index values.

Inside the `next` function, the element pointed to by `self.next_index` in the `Available Ring` (which corresponds to a descriptor index) is retrieved. The `DescriptorChain::checked_new` function is called using this retrieved value, and the result value is returned as the element during iteration.

The `checked_new` function calculates the address of the element pointed to by the index value and accesses it, extracting information like the `addr`, `len`, `flags`, and `next` of the descriptor. Finally, it constructs a `DescriptorChain` structure with this information.

```Rust
fn checked_new(
    mem: &GuestMemoryMmap,
    desc_table: GuestAddress,
    queue_size: u16,
    index: u16,
) -> Option<DescriptorChain> {
    if index >= queue_size {
        return None;
    }

    // The size of each element of the descriptor table is 16 bytes
    // - le64 addr  = 8 bytes
    // - le32 len   = 4 bytes
    // - le16 flags = 2 bytes
    // - le16 next  = 2 bytes
    // So, the calculation of the offset of the address
    // indicated by desc_index is 'index * 16'
    let desc_head = match mem.checked_offset(desc_table, (index as usize) * 16) {
        Some(a) => a,
        None => return None,
    };
    // These reads can't fail unless Guest memory is hopelessly broken
    let addr = GuestAddress(mem.read_obj(desc_head).unwrap());
    mem.checked_offset(desc_head, 16)?;
    let len = mem.read_obj(desc_head.unchecked_add(8)).unwrap();
    let flags: u16 = mem.read_obj(desc_head.unchecked_add(12)).unwrap();
    let next: u16 = mem.read_obj(desc_head.unchecked_add(14)).unwrap();
    let chain = DescriptorChain {
        mem,
        desc_table,
        queue_size,
        ttl: queue_size,
        index,
        addr,
        len,
        flags,
        next,
    };
    if chain.is_valid() {
        Some(chain)
    } else {
        None
    }
}
```

Since the `next` function returns a `DescriptorChain`, you access the descriptor's information when processing within the loop by accessing the relevant members of the `DescriptorChain` structure.

Although I have not mentioned it much so far, `Used Ring` also needs to be updated on the host side.
However, this is not a difficult process and can be implemented by defining the following functions and calling them as necessary.

```rust
/// Puts an available descriptor head into the used ring for use by the guest
pub fn add_used(&mut self, mem: &GuestMemoryMmap, desc_index: u16, len: u32) {
    if desc_index >= self.actual_size() {
        // TODO error
        return;
    }
    let used_ring = self.used_ring;
    let next_used = (self.next_used.0 % self.actual_size()) as u64;

    // virtq_used structure has 4 byte entry before `ring` fields, so skip 4 byte.
    // And each ring entry has 8 bytes, so skip 8 * index.
    let used_elem = used_ring.unchecked_add(4 + next_used * 8);
    // write the descriptor index to virtq_used_elem.id
    mem.write_obj(desc_index, used_elem).unwrap();
    // write the data length to the virtq_used_elem.len
    mem.write_obj(len, used_elem.unchecked_add(4)).unwrap();

    // increment the used index that is the last processed in host side.
    self.next_used += Wrapping(1);

    // This fence ensures all descriptor writes are visible before the index update is.
    fence(Ordering::Release);
    mem.write_obj(self.next_used.0, used_ring.unchecked_add(2))
        .unwrap();
```

Please remember this underlying mechanism as it forms the basis for the actual I/O implementation in the virtio-net and virtio-blk devices, which we will explain in the following sections.

### Implementation of Lightweight Communication between Guest and Host using irqfd and ioeventfd

So far, we've discussed the implementation of Virtqueues, but now let's delve into another crucial aspect related to Virtqueues: the "notification" mechanism required for communication between the host and guest when using Virtqueues. In Virtio, after filling Virtqueues with data, a mechanism for notifying the host from the guest or the guest from the host becomes necessary. Understanding how this notification is realized is essential.

In essence, notifications between the guest and host are achieved using the mechanisms `ioeventfd` and `irqfd`. Both of these mechanisms are provided through the `KVM API`.

First, for notifications from the guest to the host, we use `ioeventfd`. `ioeventfd` transforms memory writes caused by PIO/MMIO operations in the guest VM into eventfd notifications. The `KVM_IOEVENTFD` is used as part of the KVM API, where you provide the eventfd for notifications and the address for MMIO. Writes to this MMIO address are converted into notifications to the specified eventfd. As a result, software on the host side (in this case, ToyVMM) can receive notifications from the guest via eventfd. This mechanism enhances event notification efficiency, making it a more lightweight implementation compared to traditional polling or interrupt handler-based methods.

<div align="center">
<img src="../03_figs/ioeventfd.svg" width="50%">
</div>

Next, for notifications from the host to the guest, we use the `irqfd` mechanism. Although we've used `irqfd` in previous implementations, we employ `KVM_IRQFD` here. By passing the eventfd to be used for notifications and the IRQ number corresponding to the desired guest IRQ when using `KVM_IRQFD`, writes to the eventfd on the ToyVMM side are converted into hardware interrupts for the specified guest IRQ.

<div align="center">
<img src="../03_figs/ioeventfd.svg" width="50%">
</div>

Using the notification features based on the KVM API mentioned above, we achieve communication between the guest and host. Specific usage details will be discussed in the following section, "Implementation of MMIO Transport."

### Implementation of MMIO Transport

Now, let's delve into the implementation of MMIO Transport.

[Virtio Over MMIO](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1650002) provides the official specification for MMIO Transport, and you may want to refer to it as needed.

MMIO Transport is a method that can be easily used in virtual environments without PCI support, and it appears that Firecracker primarily supports MMIO Transport. MMIO Transport operates by performing device operations through Read/Write to specific memory regions.

MMIO Transport does not utilize a generic Device discovery mechanism like PCI. Therefore, Device discovery in MMIO involves providing information about the memory-mapped device's location and interrupt position to the guest OS, as described in [MMIO Device Discovery](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1660001). While the official documentation suggests documenting this in the Device Tree, an alternative method is to embed it in the kernel's command-line arguments during startup, as documented [here](https://docs.kernel.org/admin-guide/kernel-parameters). This latter method is used in this context because ToyVMM can dynamically adjust these command-line arguments at guest VM startup.

With this method, you can provide information to the guest VM to perform device discovery. The following format is used to describe MMIO device discovery:

```bash
(format)
virtio_mmio.device=<size>@<baseaddr>:<irq>

(example)
virtio_mmio.device=4K@0xd0000000:5
```

In this case, the guest VM uses the address `0xd0000000` as the starting point and performs Read/Write at predetermined offsets (register positions) to initialize and configure the device. The details are described in [MMIO Device Register Layout](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1670002).

From ToyVMM's perspective, it's crucial to ensure that processing according to the specifications is carried out for each Register when Read/Write operations occur. This is the core of the MMIO Transport implementation. Typically, IO to the MMIO region is processed as `KVM_EXIT_MMIO`, and handling this correctly allows initialization and configuration of the device through this flow.

On the other hand, notifications for I/O via Virtqueue, which we've discussed so far, are managed using ioeventfd and irqfd. In MMIO Transport, writing to the address offset `0x050` from the base address corresponds to the process of notifying the device that "data to be processed is present in the buffer." In other words, by associating this address with Eventfd using `KVM_IOEVENTFD` and then writing code to handle the desired Virtio Device's processing through this Eventfd, Notify events (writes to MMIO) generated by the guest can be directly notified as interrupts to the Eventfd.  
Additionally, since IRQ information is provided to the guest via the command line, the guest sets up to launch the corresponding device handler when an interrupt occurs at the specified IRQ. Conversely, when you want to trigger an interrupt in the Virtio device presented to the Guest VM (when you want to delegate processing to the Guest VM), you can do so by writing to this IRQ. Essentially, by creating Eventfd and registering it with IRQ using `KVM_IRQFD`, you can trigger interrupts by writing to this Eventfd from the ToyVMM side.

The figure below summarizes the above discussion, and ToyVMM implements this scheme:

<div align="center">
<img src="../03_figs/mmio_transport.svg" width="80%">
</div>

### MMIO Transport - Implementation Corresponding to MMIO Device Register Layout

The implementation of MMIO Transport can be found in the `mmio.rs` file. The `MmioTransport` structure, like the `I/O Bus` we discussed in the [Serial Console implementation](./02-5_serial_console_implementation.md), implements the `BusDevice` trait and is registered within the `Bus` structure. It allows handling MMIO I/O in response to `KVM_EXIT_MMIO`, similar to how traditional `VcpuExit::IoIn` and `VcpuExit::IoOut` are processed.

Therefore, the `MmioTransport` implements the `read` and `write` functions required to satisfy the `BusDevice`. These functions contain specific logic for handling register accesses, which are essentially the device emulation processes. Naturally, this implementation follows the specifications of the [MMIO Device Register Layout](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1670002). Here's a portion of the `read` function as an example:

```Rust
impl BusDevice for MmioTransport {
    // OASIS: MMIO Device Register Layout
    #[allow(clippy::bool_to_int_with_if)]
    fn read(&mut self, offset: u64, data: &mut [u8]) {
        match offset {
            0x00..=0xff if data.len() == 4 => {
                let v = match offset {
                    0x0 => MMIO_MAGIC_VALUE,
                    0x04 => MMIO_VERSION,
                    0x08 => self.device.device_type(),
                    0x0c => VENDOR_ID,
                    0x10 => {
                        self.device.features(self.features_select)
                            | if self.features_select == 1 { 0x1 } else { 0x0 }
                    }
                    0x34 => self.with_queue(0, |q| q.get_max_size() as u32),
                    0x44 => self.with_queue(0, |q| q.ready as u32),
                    0x60 => self.interrupt_status.load(Ordering::SeqCst) as u32,
                    0x70 => self.driver_status,
                    0xfc => self.config_generation,
                    _ => {
                        println!("unknown virtio mmio register read: 0x{:x}", offset);
                        return;
                    }
                };
                LittleEndian::write_u32(data, v);
            }
            0x100..=0xfff => self.device.read_config(offset - 0x100, data),
            _ => {
                // WARN!
                println!(
                    "invalid virtio mmio read: 0x{:x}:0x{:x}",
                    offset,
                    data.len()
                );
            }
        }
    }
```

A detailed explanation of the `read` and `write` functions would be quite extensive, so I'll skip it here. However, as you can see, the implementation is straightforward, and you can easily understand it by referring to the specification while examining the source code.

The processing in this part is essential for handling initialization and configuration of Virtio devices during the initialization sequence called by the Device Driver from the Guest VM during startup. By adding debugging code here, you can observe the device initialization sequence initiated by the guest.

#### Observing MMIO Device Initialization during Guest VM Startup Sequence

The Guest OS includes the Virtio device driver (on the guest side), which is expected to perform Virtio device initialization according to the specification. In this MMIO-based implementation, during startup, the guest VM is supposed to perform R/W operations on the MMIO range of the Virtio device based on the information of the MMIO range specified in the kernel command-line. As Hypervisor, it's necessary to trap this and handle it appropriately since it corresponds to the code section where VMExit occurs during the guest VM's boot. Debugging code can be easily incorporated for observation.

Before we examine the specific processing flow, let's organize the initialization process according to the specification. In the following discussion, we will use the initialization of a Virtio network device (`virtio-net`) as an example. Device initialization specification is divided into three parts: Initialization in MMIO Transport ([MMIO-specific Initialization And Device Operation](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1700003)), General Initialization And Device Operation ([General Initialization And Device Operation](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1060003)), and Device-specific Initialization. Combining these, the flow is generally as follows:

1. Read the Magic Number. Read Device ID, Device Type, Vendor ID, and other information.
2. Reset the device.
3. Set the ACKNOWLEDGE status bit.
4. Read the Device feature bits and set the device with feature bits that the OS and driver can interpret.
5. Set the FEATURES_OK status bit.
6. Perform device-specific settings (detecting and configuring Virtqueues, writing configuration).
7. Set the DRIVER_OK status bit, and at this point, the device is in a live state.

Now, keeping this in mind, let's examine the actual processing during the guest VM's startup. Below is an example where I have added debugging code, and the output generated by the debugging code is annotated with comments for explanation.

```bash
# Read the magic number from offset 0x00
# Since it's Little Endian, the original values are 116, 114, 105, 118
# 116(10) = 74(16)
# 114(10) = 72(16)
# 105(10) = 69(16)
# 118(10) = 76(16)
# Therefore, it's 0x74726976 (magic number)
MmioRead: addr = 0xd0000000, data = [118, 105, 114, 116]
# Read device id (0x02) from offset 0x04
MmioRead: addr = 0xd0000004, data = [2, 0, 0, 0]
# Read device type (net = 0x01) from offset 0x08
MmioRead: addr = 0xd0000008, data = [1, 0, 0, 0]
# Read vendor id (virtio vendor id = 0x00) from offset 0x0c
MmioRead: addr = 0xd000000c, data = [0, 0, 0, 0]

# This part is Device Initialization Phase (3.1.1 Driver Requirements: Device Initialization)
# Write 0 to offset 0x70 (= Status) to reset the device status
MmioWrite: addr = 0xd0000070, data = [0, 0, 0, 0]
# Read from offset 0x70, and now the device is reset
MmioRead: addr = 0xd0000070, data = [0, 0, 0, 0]
# Write 0x01 to offset 0x70 (= Status) to set the ACKNOWLEDGE bit
MmioWrite: addr = 0xd0000070, data = [1, 0, 0, 0]
# Read from offset 0x70, perhaps for confirmation?
MmioRead: addr = 0xd0000070, data = [1, 0, 0, 0]
# Add 0x02 = Device(2) to offset 0x70 (= Status), so 0x70 is 0x03
MmioWrite: addr = 0xd0000070, data = [3, 0, 0, 0]

# Processing for Device/Driver Feature bits.
# The device provides its own feature set (feature bits),
# and the driver reads it and instructs the device which feature subset to accept.
# 
# First, the Virtio device driver in the Guest OS reads the feature bits
# Write 0x01 to offset 0x14 (= DeviceFeatureSel) to select the next operation
MmioWrite: addr = 0xd0000014, data = [1, 0, 0, 0]
# Read from offset 0x10 (= DeviceFeatures).
# It reads DeviceFeatures bit, and it returns (DeviceFeatureSel * 32) + 31 bits.
# Now DeviceFeatureSel=1, so it returns the DeviceFeatures bits of 64~32 bits.
# For virtio-net, DeviceFeatureSel=0x0000_0001_0000_4c83 (64-bit),
# so it returns 0x0000_0001 in Little Endian.
MmioRead: addr = 0xd0000010, data = [1, 0, 0, 0]
# Write 0x00 to offset 0x14 (= DeviceFeatureSel) for the next operation
MmioWrite: addr = 0xd0000014, data = [0, 0, 0, 0]
# Read from offset 0x10 (= DeviceFeatures).
# Now DeviceFeatureSel=0, so it returns the lower 32 bits of DeviceFeatures.
# For virtio-net, DeviceFeatureSel=0x0000_0001_0000_4c83 (64-bit),
# so it returns 0x0000_4c83 in Little Endian.
# Now, Confirmation of Values of 0x0000_4c83
# Reversed Little Endian: 0,0,76,131
# 76(10) = 4c
# 131(10) = 83
# 0x00004c83 -> Ignoring the bit set by VIRTIO_F_VERSION_1 (0x100000000) in avail_features (0x100004c83)
# In other words,
# * virtio_net_sys::VIRTIO_NET_F_GUEST_CSUM
# * virtio_net_sys::VIRTIO_NET_F_CSUM
# * virtio_net_sys::VIRTIO_NET_F_GUEST_TSO4
# * virtio_net_sys::VIRTIO_NET_F_GUEST_UFO
# * virtio_net_sys::VIRTIO_NET_F_HOST_TSO4
# * virtio_net_sys::VIRTIO_NET_F_HOST_UFO
# The feature bits of this information are returned.
MmioRead: addr = 0xd0000010, data = [131, 76, 0, 0]
# The reading of feature bits is done here, and from here, it instructs the device about the feature subset to accept.
# The process is similar to reading, where you write 0x00/0x01 to DriverFeatureSel bit
# and then write the feature bits you want to set to DriverFeature.
# First, write 0x01 to offset 0x24 (DriverFeatureSel/activate guest feature) to set 'acked_features' to 0x01
MmioWrite: addr = 0xd0000024, data = [1, 0, 0, 0]
# Write 0x01 (= 0x0000_0001, one of the values read earlier) to offset 0x20 (DriverFeatures).
# Since DriverFeatureSel is set to 0x01, a 32-bit shift occurs, and 0x0000_0001_0000_0000 is actually set.
MmioWrite: addr = 0xd0000020, data = [1, 0, 0, 0]
# Write 0x00 to offset 0x24 (DriverFeatureSel/activate guest feature) to set 'acked_features' to 0x00
MmioWrite: addr = 0xd0000024, data = [0, 0, 0, 0]
# Write 0x0000_4c83 (the other value read earlier) to offset 0x20 (DriverFeatures).
MmioWrite: addr = 0xd0000020, data = [131, 76, 0, 0]
# The processing of Feature bits is completed here.

# Read offset 0x70(= Status) -> Since 0x03 was specified most recently, returning 0x03 is good.
MmioRead: addr = 0xd0000070, data = [3, 0, 0, 0]
# Write the value (3 + 8 = 11) obtained by 'adding' 0x08 = FEATURES_OK(8) to offset 0x70(= Status)
MmioWrite: addr = 0xd0000070, data = [11, 0, 0, 0]
# Read from offset 0x70(= Status). Naturally, 11 is returned.
MmioRead: addr = 0xd0000070, data = [11, 0, 0, 0]

# Device-specific setup starts from here (4.2.3.2 Virtqueue Configuration)
# Write 0x00 to offset 0x30 (= QueueSel) to select self.queue_select
MmioWrite: addr = 0xd0000030, data = [0, 0, 0, 0]
# Read from offset 0x44 (= QueueReady), and it's not ready yet, so it returns 0x0 as expected
MmioRead: addr = 0xd0000044, data = [0, 0, 0, 0]
# Read from offset 0x34 (= QueueNumMax) to check the queue size
MmioRead: addr = 0xd0000034, data = [0, 1, 0, 0]
# Write the previously read QueueNum to offset 0x38 (= QueueNum)
MmioWrite: addr = 0xd0000038, data = [0, 1, 0, 0]

# Virtual queue's 'descriptor' area 64-bit long physical address
# Write the location of the descriptor area of the selected queue (0)
# to offset 0x80 (= QueueDescLow = lo(q.desc_table) / lower 32 bits of the address)
MmioWrite: addr = 0xd0000080, data = [0, 64, 209, 122]
# Same as above, but set the remaining part of 0x84 (QueueDescHigh = hi(q.desc_table) / higher 32 bits of the address)
MmioWrite: addr = 0xd0000084, data = [0, 0, 0, 0]
# Combining the two, it's 0x0000_0000_7ad1_4000 (q.desc_table) as the base address

# Virtual queue's 'driver' area 64-bit log physical address
# Write the location of the driver area (avail_ring) of the selected queue (0)
# to offset 0x90 (= QueueDeviceLow = lo(q.avail_ring) / lower 32 bits of the address)
MmioWrite: addr = 0xd0000090, data = [0, 80, 209, 122]
# Same as above, but set the remaining part of 0x94 (QueueDeviceHigh = hi(q.avail_ring) / higher 32 bits of the address)
MmioWrite: addr = 0xd0000094, data = [0, 0, 0, 0]
# Combining the two, it's 0x0000_0000_7ad1_5000 (q.avail_ring)
# Address range of q.desc_table: q.avail_ring - q.desc_table = 0x1000 = 512(10)

# Virtual queue's 'device' area 64-bit long physical address
# Write the location of the device area (used_ring) of the selected queue (0) to offset 0xa0 (= QueueDeviceLow = lo(q.used_ring) / lower 32bits of the address)
MmioWrite: addr = 0xd00000a0, data = [0, 96, 209, 122]
# Same as above, but set the remaining part of 0xa4 (QueueDeviceHigh = hi(q.used_ring) / higher 32 bits of the address)
MmioWrite: addr = 0xd00000a4, data = [0, 0, 0, 0]
# Combining the two, it's 0x0000_0000_7ad1_6000 (q.used_ring)
# Address range of q.avail_ring: q.used_ring - q.avail_ring = 0x1000 = 512(10)

# Write 0x1 to offset 0x44 (QueueReady = q.ready) to make it Ready
MmioWrite: addr = 0xd0000044, data = [1, 0, 0, 0]

# The same process is performed for the other queue (1)
MmioWrite: addr = 0xd0000030, data = [1, 0, 0, 0]
MmioRead: addr = 0xd0000044, data = [0, 0, 0, 0]
MmioRead: addr = 0xd0000034, data = [0, 1, 0, 0]
MmioWrite: addr = 0xd0000038, data = [0, 1, 0, 0]
MmioWrite: addr = 0xd0000080, data = [0, 128, 196, 122]
MmioWrite: addr = 0xd0000084, data = [0, 0, 0, 0] # q.desc_table = 0x0000_0000_7ad1_8000
MmioWrite: addr = 0xd0000090, data = [0, 144, 196, 122]
MmioWrite: addr = 0xd0000094, data = [0, 0, 0, 0] # q.avail_ring = 0x0000_0000_7ad1_9000
MmioWrite: addr = 0xd00000a0, data = [0, 160, 196, 122]
MmioWrite: addr = 0xd00000a4, data = [0, 0, 0, 0] # q.used_ring = 0x0000_0000_7ad1_a000
MmioWrite: addr = 0xd0000044, data = [1, 0, 0, 0]
# Device-specific setup (setup of two queues for virtio-net) is completed here

# Read from offset 0x70 (= Status) and return 0x11, which was written recently
MmioRead: addr = 0xd0000070, data = [11, 0, 0, 0]
# Write 0x04 (DRIVER_OK(4)) to offset 0x70 (= Status) to 'add' it to the current value (11 + 4 = 15)
MmioWrite: addr = 0xd0000070, data = [15, 0, 0, 0]
# Read from offset 0x70 (= Status), and naturally, it returns 15
MmioRead: addr = 0xd0000070, data = [15, 0, 0, 0]
# Device Initialization Phase (3.1.1 Driver Requirements: Device Initialization) is completed here
```

When interpreted carefully, it becomes evident that the behavior aligns with the specifications. For reading and writing device-specific configurations, It execute the appropriate function from the `VirtioDevice` associated with the `MmioTransport` initialization. In other words, the `VirtioDevice` Trait requires implementations to provide the necessary information for such operations.

Additionally, during the initialization process, there are multiple MMIO writes to offset=0x70. These correspond to updating the status as the initialization sequence progresses. ToyVMM confirms the completion of these status updates (`ACKNOWLEDGE` -> `DRIVER` -> `FEATURES_OK` -> `DRIVER_OK` transition). After the `DRIVER_OK` status update, ToyVMM calls the `activate` function to perform device-specific activation procedures (e.g., setting up epoll and its handlers). The specifics of this activation process are delegated to individual device implementations.

### Summary

In this section, we provided a detailed explanation of the `Virtio` mechanisms within ToyVMM. In the following sections, we will introduce concrete implementations of actual devices that were not covered in this section, specifically Network Devices and Block Devices. We will also verify the execution of specific I/O operations using the code implemented according to the Virtio principles.

### Reference

* [OASIS](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=virtio)
* [Creating a Hypervisor](https://syuu1228.github.io/howto_implement_hypervisor/)
* [The Definitive KVM (Kernel-based Virtual Machine) API Documentation](https://docs.kernel.org/virt/kvm/api.html)
* [QEMU](https://github.com/qemu/qemu/)
* [Introduction to VirtIO](https://blogs.oracle.com/linux/post/introduction-to-virtio)
* [Virtqueues and Virtio Ring: How the Data Travels](https://www.redhat.com/ja/blog/virtqueues-and-virtio-ring-how-data-travels)
