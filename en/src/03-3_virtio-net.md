# Implement virtio-net device

In this section, we will proceed with the implementation of a Network Device as a specific Virtio Device. While the specification can be found in the official OASIS documentation [Network Device](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-2170001), please note that this implementation may not align perfectly with the specification. If you haven't read the previous sections, be sure to review them before continuing with this section.

### virtio-net Mechanism

In `virtio-net`, three types of Virtqueues are typically used: Transmit Queue, Receive Queue, and Control Queue.
The Transmit Queue is used for data transmission from the guest to the host, the Receive Queue for data transmission from the host to the guest. The Control Queue is used for guest-to-host operations related to NIC settings, such as setting promiscuous mode, enabling/disabling broadcast reception, and multicast reception.
For the sake of brevity, we are omitting the implementation of the Control Queue in this section. It is worth noting that the specification allows scaling the number of Virtqueues, but for simplicity, we are not implementing that.

In the following sections, we will provide detailed implementation-based explanations of the network device.

### Network Device Implementation Details

The implementation of `virtio-net` can be found in `net.rs`.
We will break down and explain the initialization phase and the post-initialization phase.

The following diagram primarily focuses on the initialization process of the Network Device:

<div align="center">
<img src="./03_figs/net-device_init_activate.svg", width="80%">
</div>

The `Net` struct implements the `VirtioDevice` Trait and is associated with the `MmioTransport`. As mentioned earlier, device-specific operations during MMIO Transport initialization depend on the implementation of this `Net` struct.

For example, during initialization, a query about the `Device Type` occurs. According to the specification, for a `Net` device, this should return `0x01`, and the `Net` struct implements it as follows:

```Rust
impl VirtioDevice for Net {
    fn device_type(&self) -> u32 {
        // types::NETWORK_CARD:u32 = 0x01
        types::NETWORK_CARD
    }
    ...
}
```

Similarly, queries about the `Device Feature` should be implemented to return device-specific values. Additionally, during initialization, the guest OS initializes the `Descriptor Table`, `Available Ring`, and `Used Ring` of the Virtqueues, and the addresses for each of these are notified. This allows the addresses to be stored for each queue so they can be referenced during actual processing.
Once the initialization steps are completed and the status is updated to a specific value, ToyVMM executes the `activate` function implemented in the device. In the case of the `Net` device, within this `activate` function, various file descriptors are registered with `epoll`, and the setup of handlers (e.g., `NetEpollHandler`) triggered by `epoll` is performed. The `Net` device emulates I/O by creating a Tap device on the host side and writing data received from the guest via Virtqueue to the Tap device for transmission (`Tx`), and writing incoming data from the Tap device to Virtqueue to notify the guest (`Rx`). Four file descriptors are registered with `epoll`: the `fd` of the `tap` device, an `eventfd` for notification of the Tx Virtqueue, an `eventfd` for notification of the Rx Virtqueue, and an `eventfd` for halting in unexpected situations.

Next, we will provide a detailed diagram of the Network Device in its activated state:

<div align="center">
<img src="./03_figs/net-device_activated.svg", width="100%">
</div>

When one of the registered file descriptors in `epoll` triggers an event, it dispatches the `NetEpollHandler` for event processing. `NetEpollHandler` varies its actions based on the event triggered. In any case, within the `NetEpollHandler`, it references the Virtqueue and performs I/O emulation.

One important point to note is that the initialization process of the device, based on `KVM_EXIT_MMIO`, is a processing call within the thread handling vCPU. In ToyVMM, this is done in a separate thread from the main thread. However, the thread responsible for executing I/O is also separate from the vCPU processing thread (currently handled in the main thread). To facilitate communication between these threads, channels are used to send the initialized `NetEpollHandler`. This allows I/O to be processed while the guest VM is running and CPU emulation is conducted in a separate thread.

As mentioned earlier, communication between the host and guest is primarily triggered by events related to Virtqueue Eventfds and Tap device file descriptors. In the following sections, we will provide more detailed explanations of how processing occurs in both the Tx and Rx cases.

#### Tx (Guest -> Host)

Let's start by examining the implementation of communication in the Guest -> Host direction (Tx) and provide a detailed explanation. Once again, for Tx, the `Descriptor Table`, `Available Ring`, and `Used Ring` function as follows:

- `Descriptor Table`: It contains descriptors that point to the data the Guest is trying to transmit.
- `Available Ring`: It stores the index of the descriptor pointing to the transmit data. The Host reads this index and processes Tx emuration.
- `Used Ring`: It stores the index of descriptors that have been processed on the Host side. The Guest reads this index to collect processed descriptors.

Tx initiates when the guest (guest device driver) prepares a packet, and control is transferred to ToyVMM when a Write operation occurs on `QueueNotify`.

Specifically, in the guest, the following steps are expected:

1. The guest sets the data address and length in the first Descriptor's `addr` and `len` fields.
2. The guest stores the index of the `Descriptor` pointing to the transmit data in the `Available Ring` entry, pointed to by the `Available Ring` index.
3. The guest increments the `Available Ring` index.
4. To notify the host of unprocessed data, the guest writes to the MMIO `QueueNotify`.

Now, let's shift our focus to the host side, which is handled by ToyVMM. The EventFd triggered by the write to MMIO's `QueueNotify` is picked up by epoll monitoring. It triggers the NetEpollHandle's handler processing, specifically, the execution of the `TX_QUEUE_EVENT` corresponding operation.

<div align="center">
<img src="./03_figs/net-device_tx_1.svg", width="100%">
</div>

The implementation calls the `process_tx` function.  
In `process_tx`, the processing proceeds as follows:

1. Initialization of necessary variables, including:
   - `frame[0u8; 65562]`: A buffer to copy the data prepared by the guest.
   - `used_desc_heads[0u16; 256]`: Data for storing the index of processed Descriptors and for updating the Used Ring at the end.
   - `used_count`: A counter to keep track of how much data has been read from the guest.
2. Iteration over the TX Virtqueue until it stops, repeating steps 3 to 5.
3. Reading the data information (located at `addr`) pointed to by the Descriptor and loading it into the buffer. If the `next` field points to another Descriptor, it is followed, and the data is read out.
4. Writing the read data to the Tap device.
5. Storing the index of processed Descriptors (the Descriptor pointed to by the Available Ring) in `used_desc_heads`.
6. Updating the `Used Ring` with the information of processed Descriptors' indexes and the total amount of data stored.
7. Writing to the `eventfd` associated with the irq to trigger an interrupt and delegate processing to the guest.

<div align="center">
<img src="./03_figs/net-device_tx_2.svg", width="100%">
</div>

On the guest side, the following steps are expected:

1. Check the index of the `Used Ring`, and if there is a difference between the index of processed entries and the previously recorded index, check and process the Descriptor indexes to fill this gap.
2. The Descriptors pointed to by these Descriptor indexes have been processed on the host side, so they are returned to the chain of free Descriptors, and the recorded Descriptor numbers are updated.
3. Repeat steps 1 and 2 until there is no difference between the index of the `Used Ring` and the recorded index position.

This completes the Tx processing.

#### Rx (Host -> Guest)

Next, let's explain the communication from the host to the guest (Rx) while referring to the implementation.
In the case of Rx, the `Descriptor Table`, `Available Ring`, and `Used Ring` function as follows:

- `Descriptor Table`: It contains descriptors that point to received data, allowing the Guest to access the received data from the Tap.
- `Available Ring`: It is used for the transfer of completed empty descriptors from the Guest's side.
- `Used Ring`: It stores the index of descriptors pointing to received data, which the Guest reads to process the necessary descriptors.

Comparing Rx to Tx, you can see that the roles of the `Available Ring` and `Used Ring` are reversed.

Unlike Tx, Rx requires handling two types of event triggers: incoming packets from the Tap device and completion notifications from the guest for the Rx Virtqueue. Handling Rx is more complex compared to Tx due to the need to manage these two types of event triggers.

First, let's discuss the basic Rx processing flow, followed by considerations for cooperative behavior.

##### Basic Rx Processing Flow

The host receives data from the Tap device and needs to notify the guest by filling the Rx Virtqueue with data. To do this, some basic setup is required for the Rx Virtqueue, such as knowing where to place the data. It's important to remember that, from the perspective of ToyVMM, each element of the Virtqueue consists only of guest memory addresses, and necessary operations are performed based on Virtqueue memory access.

Returning to the guest, the following steps are expected:

1. After initializing Descriptor chains and other settings, the guest assigns the index of the head of the free Descriptor chain to the empty entry pointed to by the `Available Ring` index.
2. The guest increments the `Available Ring` index.
3. To notify the host, the guest writes to MMIO `QueueNotify`.

On the host side, when Rx Virtqueue notification is received from the guest, it interprets this as the **Rx data space is ready for address access.** 

<div align="center">
<img src="./03_figs/net-device_rx_1.svg", width="100%">
</div>

Suppose the Tap device receives a packet at this point. By detecting the trigger of the Tap's file descriptor, the `NetEpollHandler` is dispatched, and it performs event processing for `RX_TAP_EVENT`. This processing mainly involves calling the `process_rx` function. However, there are certain conditions under which this may not happen, which we will discuss later.

`process_rx` proceeds as follows:

1. `process_rx` processes as many frames as possible received from the Tap by looping until no data can be read.
2. If a successful read occurs from the Tap, the size of the read data is stored in `self.rx_count`, and the `rx_single_frame` function, which processes a single frame, is called.
3. In `rx_single_frame`, the first entry from the Available Ring is retrieved, and the beginning of the free Descriptor chain that this entry points to is extracted.
4. The received single frame's data is stored in the Descriptor, calculating the size along the way. If the received frame cannot fit into a single Descriptor, the `next` field of the Descriptor is followed to continue storing data.
5. The `Used Ring` of the Rx Virtqueue is updated with information about the index of the Descriptor containing Rx data and the total amount of data stored.
6. An interrupt is triggered by writing to the `eventfd` associated with the irq to delegate processing to the guest.

The following diagram illustrates the process of writing the received data into the Descriptor chain using the Available Ring:

<div align="center">
<img src="./03_figs/net-device_rx_2.svg", width="100%">
</div>

Once the data from the Tap device has been written, the `Used Ring` is updated, and an interrupt is sent to the guest.

<div align="center">
<img src="./03_figs/net-device_rx_3.svg", width="100%">
</div>

On the guest side, the guest checks the `Used Ring` index, references the Descriptor pointed to by new entries, retrieves and processes Rx data, and performs any necessary operations. It then updates the Available Ring, signaling to the host that it is ready to accept new data.


##### When Tap Trigger Occurs Without Rx Virtqueue Preparation

It is expected that there may be cases where Tap receives a packet when Rx Virtqueue is not ready. In such cases, even if data is extracted from Tap, it is impossible to obtain information about where to store it, preventing further processing.

To address this, a mechanism to delay Tap device processing until Rx Virtqueue is prepared is required. In the ToyVMM code, this is controlled using a flag called `deferred_rx`.

When this flag is set, ToyVMM's Rx-related processing follows the following strategy:

- When `RX_QUEUE_EVENT` is triggered, indicating that the Rx Virtqueue is ready to receive data from the guest, data is immediately retrieved from the Tap device, and processing continues. If processing is completed at this point, the flag is cleared.
- When `TAP_RX_EVENT` is triggered, processing is temporarily paused to check the status of the Rx Virtqueue. If processing can proceed, it continues, and if processing is completed, the flag is cleared. If processing cannot proceed or the amount of data in the Virtqueue is smaller than the received data, the flag is not cleared, and it waits for the Rx Virtqueue to be ready again.

##### When Tap Reception Exceeds the Prepared Rx Virtqueue

Another case to consider is when the data received by Tap exceeds the capacity of the prepared Virtqueue, as briefly mentioned above. In this case, the strategy is essentially the same, and the processing is temporarily interrupted until the next Virtqueue is prepared, controlled by the `deferred_rx` flag. When the Virtqueue is ready, processing resumes.

### Verification of virtio-net Operation

Let's test whether communication between the host and guest is possible using the implemented `Virtio` mechanism and the Network Device. Below is the result of executing the `ip addr` command inside the guest. `eth0` is recognized as a virtual NIC.

```
localhost:~# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 02:32:22:01:57:33 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::32:22ff:fe01:5733/64 scope link
       valid_lft forever preferred_lft forever
```

Let's also check on the host side. In ToyVMM, a Tap device is created on the host side. So assign an IP address (`192.168.0.10/24`).

```bash
140: vmtap0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 1000
    link/ether c6:69:6d:65:05:cf brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.10/24 brd 192.168.0.255 scope global vmtap0
       valid_lft forever preferred_lft forever
```

Additionally, assign an IP address to the guest side. Here, an address within the same subnet range as the host is assigned.

```
localhost:~# ip addr add 192.168.0.11/24 dev eth0
```

Now that everything is set up, let's ping the IP address of the host's Tap interface from within the guest. You should receive responses as follows:

```
localhost:~# ping -c 5 192.168.0.10
PING 192.168.0.10 (192.168.0.10): 56 data bytes
64 bytes from 192.168.0.10: seq=0 ttl=64 time=0.394 ms
64 bytes from 192.168.0.10: seq=1 ttl=64 time=0.335 ms
64 bytes from 192.168.0.10: seq=2 ttl=64 time=0.334 ms
64 bytes from 192.168.0.10: seq=3 ttl=64 time=0.321 ms
64 bytes from 192.168.0.10: seq=4 ttl=64 time=0.330 ms

--- 192.168.0.10 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 0.321/0.342/0.394 ms
```

Conversely, if you ping the IP address of the `virtio-net` interface in the guest from the host, you should also receive responses:

```bash
[mmichish@mmichish ~]$ ping -c 5 192.168.0.11
PING 192.168.0.11 (192.168.0.11) 56(84) bytes of data.
64 bytes from 192.168.0.11: icmp_seq=1 ttl=64 time=0.410 ms
64 bytes from 192.168.0.11: icmp_seq=2 ttl=64 time=0.366 ms
64 bytes from 192.168.0.11: icmp_seq=3 ttl=64 time=0.385 ms
64 bytes from 192.168.0.11: icmp_seq=4 ttl=64 time=0.356 ms
64 bytes from 192.168.0.11: icmp_seq=5 ttl=64 time=0.376 ms

--- 192.168.0.11 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4114ms
rtt min/avg/max/mdev = 0.356/0.378/0.410/0.028 ms
```

Although this is a simple confirmation using ICMP, it confirms that communication is functioning properly!

### Reference

- [OASIS](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=virtio)
- [Creating a Hypervisor](https://syuu1228.github.io/howto_implement_hypervisor/)
