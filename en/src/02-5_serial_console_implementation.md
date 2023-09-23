# Serial Console implementation

## About Serial UART and ttyS0

UART(Universal Asynchronous Receiver/Transmitter) is an asynchronous serial communication standard used to connect computers and microcontrollers to peripheral devices. UART allows for the conversion of parallel and serial signals, enabling the conversion of input parallel data into serial data and transmitting it to the other party over a communication line. Integrated circuits designed for this purpose, known as 8250 UART devices, were manufactured, followed by various other families.

Now, in this case, we are attempting to boot the Guest OS (Linux), and having a serial console is quite useful for debugging and other purposes. A serial console sends all console outputs of the Guest to the serial port. With the serial terminal properly configured, you can remotely monitor the system's boot status or log in to the system via the serial port. In this instance, we will use this method to check the state of a Guest VM running on ToyVMM and perform operations within the Guest.

To output console messages to the serial port, it is necessary to set `console=ttyS0` as a kernel boot parameter. In the current implementation of ToyVMM, this value is provided as the default.

The challenge lies on the side that receives this, the serial terminal. Since the I/O port address corresponding to the serial port is fixed, ToyVMM's layer will receive instructions like `KVM_EXIT_IO` for the nearby address. In other words, it needs to properly handle output information to the serial console issued from the Guest OS and other necessary setup requests. This can be achieved by emulating the UART device. Furthermore, by emulating the device, if we can output console output to the standard output and reflect our standard input to the Guest VM, when starting the VM from ToyVMM, we can confirm the boot information and perform operations on the Guest from our local terminal.

In summary, we need to create something like the conceptual diagram below:

<img src="../02_figs/overview_serial_device.svg" width="100%">

We will explain this in detail in the following sections.

## Serial UART

For detailed information about Serial UART, you can refer to the following resources by Lammet Bies and Wikibooks, which provide rich information:

* [Serial UART information (Lammet Bies)](https://www.lammertbies.nl/comm/info/serial-uart)
* [Serial Programming / 8250 UART Programming (Wikibooks)](https://en.wikibooks.org/wiki/Serial_Programming/8250_UART_Programming)

The following figures are based on Lammet's document, with a brief explanation of each bit of each register. Although this diagram was created by me personally in writing this document, it is attached in the hope that it will help readers understand the meaning of each register and bit. However, the meaning of each register and bit is not explained in this document, so please refer to the above document for confirmation:

<img src="../02_figs/serial_uart_registers.svg" width="100%">

Basically, UART operations are performed by manipulating the registers and bits shown above. In our case, we need to emulate this in software, and we plan to do this using [rust-vmm/vm-superio](https://github.com/rust-vmm/vm-superio). In the following sections, we'll briefly compare the implementation of [rust-vmm/vm-superio](https://github.com/rust-vmm/vm-superio) with the above specifications.

## Software Implementation of Serial Device using rust-vmm/vm-superio

### Initial Value Settings/RW Implementation

Here, we will review the implementation of the serial device using [rust-vmm/vm-superio](https://github.com/rust-vmm/vm-superio) while comparing it with the above specifications. I encourage you to obtain the code from the link provided and inspect it for yourself. The following content is based on version `vm-superio-0.6.0`, so please note that it may have changed in the latest code.

First, let's organize some initial values for certain variables. [rust-vmm/vm-superio](https://github.com/rust-vmm/vm-superio) was originally designed for VMM usage, so it initializes certain register values and doesn't anticipate changes.

| Variable                 | DEFAULT VALUE | Meaning                    | REGISTER   |
|--------------------------|---------------|----------------------------|------------|
| baud_divisor_low         | 0x0c          | Baud rate 9600 bps         |            |
| baud_divisor_high        | 0x00          | Baud rate 9600 bps         |            |
| interrupt_enable         | 0x00          | No interrupts enabled      | IER        |
| interrupt_identification | 0b0000_0001   | No pending interrupt       | IIR        |
| line_control             | 0b0000_0011   | 8-bit word length          | LCR        |
| line_status              | 0b0110_0000   | (1)                        | LSR        |
| modem_control            | 0b0000_1000   | (2)                        | MCR        |
| modem_status             | 0b1011_0000   | (3)                        | MSR        |
| scratch                  | 0b0000_0000   |                            | SCR        |
| in_buffer                | Vec::new()    | Vector values (buffer)     | -          |

* (1) Setting THR empty-related bits. Setting these bits means that data can be received at any time. This represents the assumption that it will be used as a virtual device.
* (2) Many UARTs enable interrupts by default by setting Auxiliary Output 2 to 1.
* (3) Connected state and hardware data flow initialization.

Now, let's look at the processing when a write request is received. As a result of `KVM_EXIT_IO`, we receive the address where IO occurred and the data to be written. On the ToyVMM side, we calculate the appropriate device (in this case, the Serial UART device) and its offset from the base address based on these values and call the `write` function defined in `vm-superio`. The following content is a simplified table representing the processing of `Serial::write`. In general, it involves straightforward register value modification, with a few exceptions:

| Variable         | OFFSET(u8) | Additional Conditions   | Write                              |
|------------------|------------|-------------------------|------------------------------------|
| DLAB_LOW_OFFSET  | 0          | is_dlab_set = true      | Modify `self.baud_divisor_low`     |
| DLAB_HIGH_OFFSET | 1          | is_dlab_set = true      | Modify `self.baud_divisor_high`    |
| DATA_OFFSET      | 0          | - (is_dlab_set = false) | (1)                                |
| IER_OFFSET       | 1          | - (is_dlab_set = false) | (2)                                |
| LCR_OFFSET       | 3          | -                       | Modify `self.line_control`         |
| MCR_OFFSET       | 4          | -                       | Modify `self.modem_control`        |
| SCR_OFFSET       | 7          | -                       | Modify `self.scratch`              |

* (1) Depending on the current state of the Serial, we handle cases where LOOP_BACK_MODE (MCR bit 4) is enabled and when it is not enabled. 
  - If it is enabled, it simulates passing what is written to the transmit register directly to the receive register (loopback), which is not important in this context.
  - If it is not enabled, it writes the data to be written to the output and depends on the existing configuration to generate interrupts.
    - As shown in the table above, we do not support changing IIR due to write from outside, and the default value is set to `0b0000_0001`.
    - If the THR empty bit flag of IER is set for IER_OFFSET, it sets the corresponding flag for THR empty in IIR and triggers an interrupt.
* (2) Among the bits of IER, only bits 0-3 are masked, and the result is written back to `self.interrupt_enable`.

Next, let's look at the processing when a read request is received. Similarly, we present the processing of `Serial::read` in a simplified table. Unlike write, in the case of read, it mainly involves returning data as the result.

| Variable         | OFFSET(u8) | Additional Conditions   | Read                               |
|------------------|------------|-------------------------|------------------------------------|
| DLAB_LOW_OFFSET  | 0          | is_dlab_set = true      | Read `self.baud_divisor_low`       |
| DLAB_HIGH_OFFSET | 1          | is_dlab_set = true      | Read `self.baud_divisor_high`      |
| DATA_OFFSET      | 0          | - (is_dlab_set = false) | (1)                                |
| IER_OFFSET       | 1          | - (is_dlab_set = false) | Read `self.interrupt_enable`       |
| IIR_OFFSET       | 2          | -                       | (2)                                |
| LCR_OFFSET       | 3          | -                       | Read `self.line_control`           |
| MCR_OFFSET       | 4          | -                       | Read `self.modem_control`          |
| LSR_OFFSET       | 5          | -                       | Read `self.line_status`            |
| MSR_OFFSET       | 6          | -                       | (3)                                |
| SCR_OFFSET       | 7          | -                       | Read `self.scratch`                |

* (1) Reads data from the buffer held by the Serial structure. In the current implementation, this buffer is only filled by write in loopback mode, so read operations related to this region are not issued in the boot sequence of the OS.
* (2) Returns the result of `self.interrupt_identification` | `0b1100_0000 (FIFO enabled)` and resets it to the default value.
* (3) Depending on whether the current state is loopback mode, it handles differently.
  - In the case of loopback, it adjusts appropriately (not important for this context).
  - In the case of non-loopback, it straightforwardly returns the value of `self.modem_status`.

## Usage of rust-vmm/vm-superio in ToyVMM

In ToyVMM, we use [rust-vmm/vm-superio](https://github.com/rust-vmm/vm-superio) to handle `KVM_EXIT_IO` contents. Additionally, two things need to be considered:

* Outputting console output destined for the serial port to the standard output to allow monitoring of the boot sequence and internal state of the Guest VM.
* Passing the content of standard input to the Guest VM.

In the following sections, we'll go through each of these in order.

### Outputting Console Output Destined for the Serial Port to Standard Output

To monitor the boot sequence and internal state of the Guest VM, we will redirect console output destined for the serial port to the standard output. "Console output destined for the serial port" corresponds to the case of `KVM_EXIT_IO_OUT` where `KVM_EXIT_IO` is issued for the "IO Port address for Serial". The code section below handles this:

```rust
...
loop {
  match vcpu.run() {
      Ok(run) => match run {
          ...
          VcpuExit::IoOut(addr, data) => {
              io_bus.write(addr as u64, data);
          }
      ...  
      }
    }
}
...
```

Here, as a result of `KVM_EXIT_IO_OUT`, we receive the address and data to be written. On the ToyVMM side, we simply call `io_bus.write` with these values. The setup for this `io_bus` is done as follows:

```rust
let mut io_bus = IoBus::new();
let com_evt_1_3 = EventFdTrigger::new(EventFd::new(libc::EFD_NONBLOCK).unwrap());
let stdio_serial = Arc::new(Mutex::new(SerialDevice {
    serial: serial::Serial::with_events(
        com_evt_1_3.try_clone().unwrap(),
        SerialEventsWrapper {
            buffer_read_event_fd: None,
        },
        Box::new(std::io::stdout()),
    ),
}));
io_bus.insert(stdio_serial.clone(), 0x3f8, 0x8).unwrap();
vm.fd().register_irqfd(&com_evt_1_3, 4).unwrap();
```

The setup above requires some explanation, so let's go through it step by step. In essence, it accomplishes the following:

* Initializes an I/O Bus represented by `IoBus` and initializes the eventfd for interrupts.
* Initializes the Serial Device. During initialization, we provide an `eventfd` for generating interrupts in the Guest and an FD (`std::io::stdout()`) for standard output.
* Registers the Serial Device we initialized with the `IoBus`. During registration, we specify `0x3f8` as the base and `0x8` as the range.
  * This means that the range of `0x8` starting from the base `0x3f8` represents the address space used by this Serial Device.

#### Handling the I/O Bus

The address value passed via `KVM_EXIT_IO` becomes the value within the entire address space. On the other hand, the `read/write` implementation in [rust-vmm/vm-superio](https://github.com/rust-vmm/vm-superio) works based on an offset value from the Serial Device's base address. Therefore, there's a need for processing to bridge this gap.

You could simply calculate the offset, but in Firecracker, considering future extensibility (using I/O Ports for devices other than Serial), there's a `Bus` structure representing the I/O Bus. This structure allows devices to be registered along with `BusRange` (a structure representing the base address and address range for devices on the bus). Furthermore, when an I/O at a specific address occurs, the mechanism checks that address, retrieves the device registered in the corresponding address range, and performs I/O on that device using the offset from the base address.

For instance, the `write` function is implemented as follows, where it retrieves the registered device and its offset based on the address information using the `get_device` function, and then calls the `write` function implemented in that device with the offset.

```rust
pub fn write(&self, addr: u64, data: &[u8]) -> bool {
    if let Some((offset, dev)) = self.get_device(addr) {
        // OK to unwrap as lock() failing is a serious error condition and should panic.
        dev.lock()
            .expect("Failed to acquire device lock")
            .write(offset, data);
        true
    } else {
        false
    }
}
```

Let's consider the Serial device as an example. As mentioned earlier, `KVM_EXIT_IO_OUT` for the Serial device from the Guest VM occurs within an address range of 8 bytes with a base address of `0x3f8`. ToyVMM's IoBus also registers the Serial Device with the same address base and range. For example, when you trap an instruction that writes `0b1001_0011` to `0x3fb` as `KVM_EXIT_IO_OUT`, it interprets this instruction as writing `0b1001_0011` to `LCR` at the position `0x3` from the base address `0x3f8`.

#### Interrupt Notification to Guest VM via eventfd/irqfd

Now, let's discuss KVM and interrupts. We will reference some Linux source code, mainly from version `v4.18`.

:warning: The following information is mainly based on source code and may not capture all the details of state transitions. If you find any inaccuracies, please let me know in the comments.

In [rust-vmm/vm-superio](https://github.com/rust-vmm/vm-superio), during Serial initialization, it requires an [`EventFd`](https://docs.rs/vmm-sys-util/0.6.1/vmm_sys_util/eventfd/struct.EventFd.html) as its first argument. This is a wrapper for [eventfd](https://man7.org/linux/man-pages/man2/eventfd.2.html) in Linux. Eventfd allows inter-process and process-to-kernel event notifications.

Next is irqfd. irqfd is a mechanism based on eventfd that allows injecting interrupts into a VM. In simple terms, it's like having one end of eventfd held by KVM, and the other end's notifications are interpreted as interrupts to the Guest VM. This irqfd-based interrupt is meant to emulate interrupts from the external world to the Guest VM, which corresponds to regular system interrupts from peripheral devices in a typical system. The reverse direction of interrupts is handled using the `ioeventfd` mechanism, which we'll omit for now.

Let's examine how irqfd is connected to Guest VM interrupts by looking at the source code. When you perform an ioctl with `KVM_IRQFD` against KVM, it goes through the KVM processing with the data passed to `kvm_irqfd` and `kvm_irqfd_assign`. In the `kvm_irqfd_assign` function, an instance of the `kvm_kernel_irqfd` structure is created. At this point, settings are made based on additional information passed during the ioctl. Particularly, the `gsi` field in the `kvm_kernel_irqfd` structure is set based on the value passed as an argument during the ioctl. This `gsi` corresponds to the index of the interrupt table for the Guest, so when making the ioctl, you specify which interrupt table entry you want to use along with the eventfd. ToyVMM sets this up with a line like this:

```rust
vm.fd().register_irqfd(&com_evt_1_3, 4).unwrap();
```

This is defined as a method in the `kvm_ioctl::VmFd` structure.

```rust
pub fn register_irqfd(&self, fd: &EventFd, gsi: u32) -> Result<()> {
    let irqfd = kvm_irqfd {
        fd: fd.as_raw_fd() as u32,
        gsi,
        ..Default::default()
    };
    // Safe because we know that our file is a VM fd, we know the kernel will only read
    // the correct amount of memory from our pointer, and we verify the return result.
    let ret = unsafe { ioctl_with_ref(self, KVM_IRQFD(), &irqfd) };
    if ret == 0 {
        Ok(())
    } else {
        Err(errno::Error::last())
    }
}
```

In other words, in the aforementioned setup, the eventfd (`com_evt_1_3`) used by the Serial device has been configured with GSI=4 (the Guest VM's interrupt table index for the COM1 port). Therefore, any `write` operation performed on `com_evt_1_3` results in an interrupt being sent to the Guest VM as if it were generated from COM1. From the Guest's perspective, this means that an interrupt originated from the Serial device downstream of COM1, leading to the invocation of the Guest VM's COM1 interrupt handler.

Now, let's discuss the setup of the Guest-side Interrupt Table (GSI: Global System Interrupt Table) and how and when it's established. In short, these tables are set up by issuing an ioctl to KVM with `KVM_CREATE_IRQCHIP`. This operation creates two interrupt controllers, the `PIC` and `IOAPIC` (internally, the `kvm_pic_init` function handles PIC initialization, registers read/write ops, and sets it in `kvm->arch.vpic`. Similarly, `kvm_ioapic_init` initializes the IOAPIC, registers read/write ops, and sets it in `kvm->arch.vioapic`). These hardware components, such as the PIC and IOAPIC, are implemented within KVM for the purpose of acceleration, so there's no need to emulate them separately. While you could delegate this task to qemu, we'll omit this detail here since we're not using it.

Furthermore, the `kvm_setup_default_irq_routing` function sets up default IRQ routing. This process determines which handler will be invoked for each GSI-based interrupt. Let's take a closer look at the contents of `kvm_setup_default_irq_routing`. This function calls `kvm_set_irq_routing`, where the essential processing takes place. Here, a `kvm_irq_routing_table` is created and populated with `kvm_kernel_irq_routing_entry` structures that represent the mapping from GSI to IRQ.

The `kvm_kernel_irq_routing_entry` structures are populated using a loop that iterates through a `default_routing` array. Here's how `default_routing` is defined along with related macros:

```C
#define SELECT_PIC(irq) \
    ((irq) < 8 ? KVM_IRQCHIP_PIC_MASTER : KVM_IRQCHIP_PIC_SLAVE)

#define IOAPIC_ROUTING_ENTRY(irq) \
    { .gsi = irq, .type = KVM_IRQ_ROUTING_IRQCHIP, \
      .u.irqchip = { .irqchip = KVM_IRQCHIP_IOAPIC, .pin = (irq) } }

#define ROUTING_ENTRY1(irq) IOAPIC_ROUTING_ENTRY(irq)

#define PIC_ROUTING_ENTRY(irq) \
    { .gsi = irq, .type = KVM_IRQ_ROUTING_IRQCHIP, \
      .u.irqchip = { .irqchip = SELECT_PIC(irq), .pin = (irq) % 8 } }

#define ROUTING_ENTRY2(irq) \
    IOAPIC_ROUTING_ENTRY(irq), PIC_ROUTING_ENTRY(irq)

static const struct kvm_irq_routing_entry default_routing[] = {
    ROUTING_ENTRY2(0), ROUTING_ENTRY2(1),
    ROUTING_ENTRY2(2), ROUTING_ENTRY2(3),
    ROUTING_ENTRY2(4), ROUTING_ENTRY2(5),
    ROUTING_ENTRY2(6), ROUTING_ENTRY2(7),
    ROUTING_ENTRY2(8), ROUTING_ENTRY2(9),
    ROUTING_ENTRY2(10), ROUTING_ENTRY2(11),
    ROUTING_ENTRY2(12), ROUTING_ENTRY2(13),
    ROUTING_ENTRY2(14), ROUTING_ENTRY2(15),
    ROUTING_ENTRY1(16), ROUTING_ENTRY1(17),
    ROUTING_ENTRY1(18), ROUTING_ENTRY1(19),
    ROUTING_ENTRY1(20), ROUTING_ENTRY1(21),
    ROUTING_ENTRY1(22), ROUTING_ENTRY1(23),
};
```

As you can see, IRQ numbers 0-15 are passed to `ROUTING_ENTRY2`, and IRQ numbers 16-23 are passed to `ROUTING_ENTRY1`. `ROUTING_ENTRY2` calls both `IOAPIC_ROUTING_ENTRY` and `PIC_ROUTING_ENTRY`, while `ROUTING_ENTRY1` calls `IOAPIC_ROUTING_ENTRY` only, creating structures with the necessary information.

These structures are used to set up each `.u.irqchip.irqchip` value (`KVM_IRQCHIP_PIC_SLAVE`, `KVM_IRQCHIP_PIC_MASTER`, `KVM_IRQCHIP_IOAPIC`) appropriately in the `kvm_set_routing_entry` function, depending on the IRQ. This function performs callbacks (`kvm_set_pic_irq`, `kvm_set_ioapic_irq`) and any necessary configurations when an interrupt occurs. We'll discuss these callbacks in more detail later.

```C
int kvm_set_routing_entry(struct kvm *kvm,
                          struct kvm_kernel_irq_routing_entry *e,
                          const struct kvm_irq_routing_entry *ue)
{
    /* We can't check irqchip_in_kernel() here as some callers are
     * currently initializing the irqchip. Other callers should therefore
     * check kvm_arch_can_set_irq_routing() before calling this function.
     */
    switch (ue->type) {
    case KVM_IRQ_ROUTING_IRQCHIP:
        if (irqchip_split(kvm))
            return -EINVAL;
        e->irqchip.pin = ue->u.irqchip.pin;
        switch (ue->u.irqchip.irqchip) {
        case KVM_IRQCHIP_PIC_SLAVE:
            e->irqchip.pin += PIC_NUM_PINS / 2;
            /* fall through */
        case KVM_IRQCHIP_PIC_MASTER:
            if (ue->u.irqchip.pin >= PIC_NUM_PINS / 2)
                return -EINVAL;
            e->set = kvm_set_pic_irq;
            break;
        case KVM_IRQCHIP_IOAPIC:
            if (ue->u.irqchip.pin >= KVM_IOAPIC_NUM_PINS)
                return -EINVAL;
            e->set = kvm_set_ioapic_irq;
            break;
        default:
            return -EINVAL;
        }
        e->irqchip.irqchip = ue->u.irqchip.irqchip;
        break;
...
```

Now, let's return to the discussion of `irqfd`. Although not mentioned earlier, the `kvm_irqfd_assign` function includes the `init_waitqueue_func_entry(&irqfd->wait, irqfd_wakeup)` process, registering `irqfd_wakeup` with `&irqfd->wait->func`. This function is called when an interrupt occurs, and it invokes `schedule_work(&irqfd->inject)`.

The `inject` field is also initialized within the `kvm_irqfd_assign` function, resulting in a call to the `irqfd_inject` function. Inside `irqfd_inject`, the `kvm_set_irq` function is called. 

The `kvm_set_irq` function lists entries with the incoming IRQ number and calls their `set` callbacks. This means that functions like `kvm_set_pic_irq` and `kvm_set_ioapic_irq`, as described earlier, will be called based on the routing information.

<img src="../02_figs/kvm_irq_gsi_routing.svg" width="100%">

The following explanation will go into a little more depth on interrupt processing, but since they are not necessary for understanding ToyVMM, you may skip to [ToyVMM serial console](#toyvmm-serial-console).

Let's take a closer look at the `kvm_set_pic_irq` handler, which is responsible for handling interrupts. While this discussion slightly deviates from the main topic, it's a good opportunity to explore it more thoroughly.
`kvm_set_pic_irq` simply utilizes the `kvm_pic_set_irq` function, passing the relevant parameters.

```C
static int kvm_set_pic_irq(struct kvm_kernel_irq_routing_entry *e,
                           struct kvm *kvm, int irq_source_id, int level,
                           bool line_status)
{
    struct kvm_pic *pic = kvm->arch.vpic;
    return kvm_pic_set_irq(pic, e->irqchip.pin, irq_source_id, level);
}
```

Let's inspect the implementation of `kvm_pic_set_irq`:

```C
int kvm_pic_set_irq(struct kvm_pic *s, int irq, int irq_source_id, int level)
{
    int ret, irq_level;

    BUG_ON(irq < 0 || irq >= PIC_NUM_PINS);

    pic_lock(s);
    irq_level = __kvm_irq_line_state(&s->irq_states[irq],
                                     irq_source_id, level);
    ret = pic_set_irq1(&s->pics[irq >> 3], irq & 7, irq_level);
    pic_update_irq(s);
    trace_kvm_pic_set_irq(irq >> 3, irq & 7, s->pics[irq >> 3].elcr,
                          s->pics[irq >> 3].imr, ret == 0);
    pic_unlock(s);

    return ret;
}
```

In `pic_set_irq1`, the IRQ level is set, and then `pic_update_irq` calls the `pic_irq_request` and updates the `kvm->arch.vpic` structure.

```C
/*
 * raise irq to CPU if necessary. must be called every time the active
 * irq may changejjj
 */
static void pic_update_irq(struct kvm_pic *s)
{
	int irq2, irq;

	irq2 = pic_get_irq(&s->pics[1]);
	if (irq2 >= 0) {
		/*
		 * if irq request by slave pic, signal master PIC
		 */
		pic_set_irq1(&s->pics[0], 2, 1);
		pic_set_irq1(&s->pics[0], 2, 0);
	}
	irq = pic_get_irq(&s->pics[0]);
	pic_irq_request(s->kvm, irq >= 0);
}

/*
 * callback when PIC0 irq status changed
 */
static void pic_irq_request(struct kvm *kvm, int level)
{
	struct kvm_pic *s = kvm->arch.vpic;

	if (!s->output)
		s->wakeup_needed = true;
	s->output = level;

}
```

After that, `kvm_pic_set_irq` invokes `pic_unlock` function.  
This function is a little more import because if the `wakeup_needed` field is `true`, then invokes `kvm_vcpu_kick` function for vCPU.


```C
static void pic_unlock(struct kvm_pic *s)
    __releases(&s->lock)
{
    bool wakeup = s->wakeup_needed;
    struct kvm_vcpu *vcpu;
    int i;

    s->wakeup_needed = false;

    spin_unlock(&s->lock);

    if (wakeup) {
        kvm_for_each_vcpu(i, vcpu, s->kvm) {
            if (kvm_apic_accept_pic_intr(vcpu)) {
                kvm_make_request(KVM_REQ_EVENT, vcpu);
                kvm_vcpu_kick(vcpu);
                return;
            }
        }
    }
}

void kvm_vcpu_kick(struct kvm_vcpu *vcpu)
{
    int me;
    int cpu = vcpu->cpu;

    if (kvm_vcpu_wake_up(vcpu))
        return;

    me = get_cpu();
    if (cpu != me && (unsigned)cpu < nr_cpu_ids && cpu_online(cpu))
        if (kvm_arch_vcpu_should_kick(vcpu))
            smp_send_reschedule(cpu);
    put_cpu();
}
```

And the result of invoking `smp_send_reschedule` function in `kvm_vcpu_kick`, `native_smp_send_reschedule` function is called.

```C

static void native_smp_send_reschedule(int cpu)
{
    if (unlikely(cpu_is_offline(cpu))) {
        WARN_ON(1);
        return;
    }
    apic->send_IPI(cpu, RESCHEDULE_VECTOR);
}
```

By invoking `smp_send_reschedule`, an IPI (Inter-Processor Interrupt) is sent to another CPU, prompting it to reschedule. This results in an interrupt being inserted into the vCPU, causing a `VMExit`. Consequently, the vCPU is scheduled when the interrupt is delivered.


Now, let's briefly review the process of how interrupts are inserted. When `KVM_RUN` is executed, the following steps are performed (focusing solely on interrupt insertion, omitting other extensive processing):

```
kvm_arch_vcpu_ioctl_run
 -> vcpu_run
 -> vcpu_enter_guest
 -> inject_pending_event
 -> kvm_cpu_has_injectable_intr
```

Within `kvm_cpu_has_injectable_intr`, the `kvm_cpu_has_extint` function is called. In this case, it likely returns `1`, probably based on the value of `s->output` set by `pic_irq_request`.

Therefore, the following part of the `inject_pending_event` function is reached:

```C
	} else if (kvm_cpu_has_injectable_intr(vcpu)) {
		/*
		 * Because interrupts can be injected asynchronously, we are
		 * calling check_nested_events again here to avoid a race condition.
		 * See https://lkml.org/lkml/2014/7/2/60 for discussion about this
		 * proposal and current concerns.  Perhaps we should be setting
		 * KVM_REQ_EVENT only on certain events and not unconditionally?
		 */
		if (is_guest_mode(vcpu) && kvm_x86_ops->check_nested_events) {
			r = kvm_x86_ops->check_nested_events(vcpu, req_int_win);
			if (r != 0)
				return r;
		}
		if (kvm_x86_ops->interrupt_allowed(vcpu)) {
			kvm_queue_interrupt(vcpu, kvm_cpu_get_interrupt(vcpu),
					    false);
			kvm_x86_ops->set_irq(vcpu);
		}
	}
```

Finally, `kvm_x86_ops->set_irq(vcpu)` is called, and this triggers the `vmx_inject_irq` callback function. In this process, it inserts the interrupt by setting `VMCS` (`Virtual Machine Control Structure`) with `VMX_ENTRY_INTR_INFO_FIELD`. While not elaborated on here, explaining `VMCS` would require delving into hypervisor implementation details, which is beyond the scope of this discussion. It may be added as supplementary information in the documentation in the future.

In summary, this is the flow of interrupt processing using the PIC as an example.

#### ToyVMM serial console

Now, at this point, let's temporarily set aside the exploration of interrupts and return to discussing the implementation of ToyVMM. Considering the previous discussions, let's organize what processes are being executed within ToyVMM and what happens behind the scenes.

In ToyVMM, before performing `register_irqfd` as mentioned earlier, a function called `setup_irqchip` is actually executed. This function acts as a thin wrapper and internally makes calls to `create_irq_chip` and `create_pit2`. 

```rust
#[cfg(target_arch = "x86_64")]
pub fn setup_irqchip(&self) -> Result<()> {
    self.fd.create_irq_chip().map_err(Error::VmSetup)?;
    let pit_config = kvm_pit_config {
        flags: KVM_PIT_SPEAKER_DUMMY,
        ..Default::default()
    };
    self.fd.create_pit2(pit_config).map_err(Error::VmSetup)
}
```

What's important here is the `create_irq_chip` function. Internally, it calls the `KVM_CREATE_IRQCHIP` API, as mentioned earlier, to initialize the interrupt controller and IRQ routing. Following this setup, `register_irqfd(&com_evt_1_3, 4)` is executed on the configured Guest VM, which, as explained earlier, calls functions like `kvm_irqfd_assign` to set up interrupt handlers. This completes the setup of interrupt-related configurations using the KVM API.


Now, let's revisit the interrupts coming from `com_evt_1_3`. As previously discussed, one end of the interrupt is passed to KVM along with `GSI=4` through `register_irqfd`. Consequently, any `write` issued from the other end is treated as an interrupt to the Guest VM as if it were sent to the COM1 port. On the other hand, the other end of `com_evt_1_3` is passed to the Serial Device, making writes to the eventfd on the Serial Device side (occurring after processing through `Serial::write` or through the invocation of `Serial::enqueue_raw_byte`) the actual interrupt triggers. In essence, this setup enables the Guest VM and the software-implemented Serial Device to interact in a manner similar to regular server and Serial Device communication.

Furthermore, to represent a Serial Console, we've configured `stdout` as the destination for writes corresponding to the Serial Device's output in this case. Therefore, when handling `KVM_EXIT_IO_OUT` and writing to THR, the data is passed to `stdout`, resulting in console messages being output to standard output. This effectively realizes the desired Serial Console functionality.

#### Controlling the Guest VM via Standard Input

Finally, to manipulate the Guest VM using standard input, we want to reflect the contents of standard input into the Guest VM. The [`Serial`](https://docs.rs/vm-superio/0.1.1/vm_superio/serial/struct.Serial.html) struct provided by [rust-vmm/vm-superio](https://github.com/rust-vmm/vm-superio) offers a helper function called [`enqueue_raw_bytes`](https://docs.rs/vm-superio/0.1.1/vm_superio/serial/struct.Serial.html#method.enqueue_raw_bytes). This helper function allows us to send data to the Guest VM without needing to handle low-level register operations or interrupts explicitly, as the function handles these operations internally.

To achieve this, we need to read input from the program and pass it directly to this method. We can set up standard input in raw mode, and the main thread can poll it while waiting for input. When input is received, we can use `enqueue_raw_bytes` to send it to the Guest VM. Since each vCPU of the Guest VM is executed in a separate thread, polling standard input in the main thread won't affect the processing of the Guest VM.

Here is a basic implementation:

```rust
let stdin_handle = io::stdin();
let stdin_lock = stdin_handle.lock();
stdin_lock
    .set_raw_mode()
    .expect("failed to set terminal raw mode");
let ctx: PollContext<Token> = PollContext::new().unwrap();
ctx.add(&exit_evt, Token::Exit).unwrap();
ctx.add(&stdin_lock, Token::Stdin).unwrap();
'poll: loop {
    let pollevents: PollEvents<Token> = ctx.wait().unwrap();
    let tokens: Vec<Token> = pollevents.iter_readable().map(|e| e.token()).collect();
    for &token in tokens.iter() {
        match token {
            Token::Exit => {
                println!("vcpu requested shutdown");
                break 'poll;
            }
            Token::Stdin => {
                let mut out = [0u8; 64];
                tx.send(true).unwrap();
                match stdin_lock.read_raw(&mut out[..]) {
                    Ok(0) => {
                        println!("eof!");
                    }
                    Ok(count) => {
                        stdio_serial
                            .lock()
                            .unwrap()
                            .serial
                            .enqueue_raw_bytes(&out[..count])
                            .expect("failed to enqueue bytes");
                    }
                    Err(e) => {
                        println!("error while reading stdin: {:?}", e);
                    }
                }
            }
            _ => {}
        }
    }
}
```

This is a straightforward implementation, but it achieves the desired functionality.

## Check UART Request When Booting the Linux Kernel

In the previous sections, we discussed the software implementation of the Serial UART and how it's used internally within ToyVMM. While it works effectively, it's important to examine the UART communication during the Linux Kernel boot process.

Fortunately, due to the VMM's architecture, we need to handle `KVM_EXIT_IO`, which allows us to intercept all requests sent to the serial port by injecting debug code into this handling process.

I won't go into detail about the code inserted for debugging purposes here, as it's quite straightforward to insert debug code at the appropriate locations. Instead, I'll provide annotations in three specific formats to make it clear and understandable when looking at requests made to the `0x3f8 (COM1)` register during OS startup.

```bash
[Format 1 - Read]
r($register) = $data
  - Description

- r           = Read operation
- $register   = The register corresponding to the offset calculated using the device's address (0x3f8)
- $data       = Data read from $register
- Description = Explanation

[Format 2 - Write]
w($register = $data)
  - Description

- w           = Write operation
- $register   = The register corresponding to the offset calculated using the device's address (0x3f8)
- $data       = Data to be written to $register
- Description = Explanation

[Format 3 - Write (character)]
w(THR = $data = 0xYY) -> 'CHAR'

- w(THR ...)  = Write operation to THR
- $data       = Binary data to be written to $register
- 0xYY        = $data converted to hexadecimal
- 'CHAR'      = 0xYY converted to a character based on the ASCII code table
```

Now, the following is a somewhat lengthy representation of requests made to the `0x3f8 (COM1)` register during OS startup, formatted according to the above annotations:

```bash
# Initial setup, configuring baud rate, etc.
w(IER = 0)
w(LCR = 10010011)
  - DLAB         = 1   (DLAB: DLL and DLM accessible)
  - Break signal = 0   (Break signal disabled)
  - Parity       = 010 (No parity)
  - Stop bits    = 0   (1 stop bit)
  - Data bits    = 11  (8 data bits)
w(DLL = 00001100)
w(DLM = 0)
  - DLL = 0x0C, DLM = 0x00 (Speed = 9600 bps)
w(LCR = 00010011)
  - DLAB         = 0   (DLAB: RBR, THR, and IER accessible)
  - Break signal = 0   (Break signal disabled)
  - Parity       = 010 (No parity)
  - Stop bits    = 0   (1 stop bit)
  - Data bits    = 11  (8 data bits)
w(FCR = 0)
w(MCR = 00000001)
  - Reserved            = 00
  - Autoflow control    = 0
  - Loopback mode       = 0
  - Auxiliary output 2  = 0
  - Auxiliary output 1  = 0
  - Request to send     = 0
  - Data terminal ready = 1
r(IER) = 0
w(IER = 0)

# From here, the actual console output is being received through the serial port,
# and write operations (in this case, writing to stdout) are happening.

# Checking the content of r(LSR) to determine whether to write the next character
r(LSR) = 01100000
  - Errornous data in FIFO         = 0
  - THR is empty, and line is idle = 1
  - THR is empty                   = 1
  - Break signal received          = 0
  - Framing error                  = 0
  - Parity error                   = 0
  - Overrun error                  = 0
  - Data available                 = 0
    - Bits 5 and 6 are related to character transmission and used by UART
    - If bits 5 and 6 are set, it means UART is ready to accept a new character
      - Bit 6 = '1' means that all characters have been transmitted
      - Bit 5 = '1' means that UART is capable of receiving more characters

# Since the next character write is accepted here, we write the character we want to output.
w(THR = 01011011 = 0x5b) -> '['

# Following this, the same pattern repeats:
r(LSR) = 01100000
w(THR = 00100000 = 0x20) -> ' '
# The above operation repeats 3 more times.
# ...

r(LSR) = 01100000
w(THR  = 00110000 = 0x30) -> '0'
r(LSR) = 01100000
w(THR  = 00101110 = 0x2e) -> '.'
r(LSR) = 01100000
w(THR  = 00110000 = 0x30) -> '0'
# The above operation repeats 5 more times

r(LSR) = 01100000
w(THR  = 01011101 = 0x5d) -> ']'
r(LSR) = 01100000
w(THR  = 00100000 = 0x20) -> ' '
r(LSR) = 01100000
w(THR  = 01001100 = 0x4c) -> 'L'
r(LSR) = 01100000
w(THR  = 01101001 = 0x69) -> 'i'
r(LSR) = 01100000
w(THR  = 01101110 = 0x6e) -> 'n'
r(LSR) = 01100000
w(THR  = 01110101 = 0x75) -> 'u'
r(LSR) = 01100000
w(THR  = 01111000 = 0x78) -> 'x'
r(LSR) = 01100000
w(THR  = 00100000 = 0x20) -> ' '
r(LSR) = 01100000
w(THR  = 01110110 = 0x76) -> 'v'
r(LSR) = 01100000
w(THR  = 01100101 = 0x65) -> 'e'
r(LSR) = 01100000
w(THR  = 01110010 = 0x72) -> 'r'
r(LSR) = 01100000
w(THR  = 01110011 = 0x73) -> 's'
r(LSR) = 01100000
w(THR  = 01101001 = 0x69) -> 'i'
r(LSR) = 01100000
w(THR  = 01101111 = 0x6f) -> 'o'
r(LSR) = 01100000
w(THR  = 01101110 = 0x6e) -> 'n'
r(LSR) = 01100000
w(THR  = 00100000 = 0x20) -> ' '
r(LSR) = 01100000
w(THR  = 00110100 = 0x34) -> '4'
r(LSR) = 01100000
w(THR  = 00101110 = 0x2e)-> '.'
r(LSR) = 01100000
w(THR  = 00110001 = 0x31) -> '1'
r(LSR) = 01100000
w(THR  = 00110100 = 0x34) -> '4'
r(LSR) = 01100000
w(THR  = 00101110 = 0x2e) -> '.'
r(LSR) = 01100000
w(THR  = 00110001 = 0x31) -> '1'
r(LSR) = 01100000
w(THR  = 00110111 = 0x37) -> '7'
r(LSR) = 01100000
w(THR  = 00110100 = 0x34) -> '4'
r(LSR) = 01100000
w(THR  = 00100000 = 0x20) -> ' '
r(LSR) = 01100000
w(THR  = 00101000 = 0x28) -> '('
r(LSR) = 01100000
w(THR  = 01000000 = 0x40) -> '@'
w(LSR) = 01100000
r(THR  = 00110101 = 0x35) -> '5'
r(LSR) = 01100000
w(THR  = 00110111 = 0x37) -> '7'
r(LSR) = 01100000
w(THR  = 01100101 = 0x65) -> 'e'
r(LSR) = 01100000
w(THR  = 01100100 = 0x64) -> 'd'
r(LSR) = 01100000
w(THR  = 01100101 = 0x65) -> 'e'
r(LSR) = 01100000
w(THR  = 01100010 = 0x62) -> 'b'
r(LSR) = 01100000
w(THR  = 01100010 = 0x62) -> 'b'
r(LSR) = 01100000
w(THR  = 00111001 = 0x39) -> '9'
r(LSR) = 01100000
w(THR  = 00111001 = 0x39) -> '9'
r(LSR) = 01100000
w(THR  = 01100100 = 0x64) -> 'd'
r(LSR) = 01100000
w(THR  = 01100010 = 0x62) -> 'b'
r(LSR) = 01100000
w(THR  = 00110111 = 0x37) -> '7'
r(LSR) = 01100000
w(THR  = 00101001 = 0x29) -> ')'

# Concatenating the output, we get the following line:
[    0.000000] Linux version 4.14.174 (@57edebb99db7)

# This matches the content of the first line output during OS boot.
```

Of course, Linux Kernel startup UART requests continue beyond this, and more complex operations take place. However, I won't delve further into these requests here. If you are interested, I encourage you to explore them in detail.

## Reference

* [Serial UART information](https://www.lammertbies.nl/comm/info/serial-uart)
* [Wikibooks : Serial Programming / 8250 UART Programming](https://en.wikibooks.org/wiki/Serial_Programming/8250_UART_Programming)
* [rust-vmm/vm-superio](https://github.com/rust-vmm/vm-superio)
* [Interrupt request(PC architecture)](https://en.wikipedia.org/wiki/Interrupt_request_(PC_architecture))
* [Linux Serial Console](https://www.kernel.org/doc/html/v4.15/admin-guide/serial-console.html)
* [KVM IRQFD Implementation](https://xzpeter.org/htmls/2017_12_07_kvm_irqfd/kvm_irqfd_implementation.html)
* [KVMのなかみ（KVM internals）](https://rkx1209.hatenablog.com/entry/2016_01_01_101456)
* [ハイパーバイザーの作り方~ちゃんと理解する仮想化技術~ 第2回 intel VT-xの概要とメモリ仮想化](https://syuu1228.github.io/howto_implement_hypervisor/part2.html)
* [External Interrupts in the x86 system. Part1. Interrupt controller evolution](https://habr.com/en/post/446312/)
