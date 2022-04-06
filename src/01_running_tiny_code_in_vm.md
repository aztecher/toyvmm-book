# 01. Running Tiny Code in VM

Tiny code execution is now based on [QuickStart](./quickstart.md) content, so please check it first.  

### DeepDive ToyVMM instruction and how to run tiny code in VM

This `main` function is a program that starts a VM using the KVM mechanism and executes the following small code inside the VM

```rust
code = &[
	0xba, 0xf8, 0x03, /* mov $0x3f8, %dx */
	0x00, 0xd8,       /* add %bl, %al */
	0x04, b'0',       /* add $'0', %al */
	0xee,             /* out %al, (%dx) */
	0xb0, '\n',       /* mov $'\n', %al */
	0xee,             /* out %al, (%dx) */
	0xf4,             /* hlt */
];
```

This code perform several register operations, but the initial state of the CPU regisers for this VM is set as follows.

```rust
    regs.rip = 0x1000;
    regs.rax = 2;
    regs.rbx = 2;
    regs.rflags = 0x2;
    vcpu.set_sregs(&sregs).unwrap();
    vcpu.set_regs(&regs).unwrap();
```

This will output the result of calculations (2 + 2) inside the VM from the IO Port, followed by a newline code as well.  
As you can see the result of running ToyVMM, hex value 0x34 (= '4') and 0xa (= New Line) are catched from I/O port

### How's work above code with rust-vmm libraries

Now, the following crate provided by rust-vmm is used to run these codes.

```bash
# Please see Cargo.toml
kvm-bindings
kvm-ioctls
vmm-sys-util
vm-memory
```

I omit to describe about [vmm-sys-util](https://github.com/rust-vmm/vmm-sys-util) because it is only used to create temporary files at this point, so there is nothing special to mention about it.  

I will go through the code in order and describe how each crate is related to that.  
In this explanation, we will focus primary on the perspective of **what ioctl is performed as a result of a function call** (This is because the interface to manipulate KVM from the user space relies on the iocl system call)  
Also, please note that explanations of unimportant variables may be omitted.  
It should be noted that what is described here is not only the ToyVMM implementation, but also the firecracker implementation in a similaer form.  

First, we need to open `/dev/kvm` and acquire the file descriptor. This can be done by `Kvm::new()` of [`kvm_ioctls`](https://github.com/rust-vmm/kvm-ioctls) crate. Following this process, the [`Kvm::open_with_cloexec`](https://github.com/rust-vmm/kvm-ioctls/blob/d12f5776be0937a14da1ad8f9736653e8a2ad5ba/src/ioctls/system.rs#L69-L78) function issues an `open` system call as follows, returns a file descriptor as `Kvm` structure

```rust
let ret = unsafe { open("/dev/kvm\0".as_ptr() as *const c_char, open_flags) };
```

The result obtained from above is used to call the method `create_vm`, which results in the following `ioctl` being issued

```rust
vmfd = ioctl(kvmfd, KVM_CREATE_VM, 0)

where
  vmfd: from /dev/kvm
```

Please keep in mind that the file descriptor returned from above function will be used later when preparing the CPU.  
Anyway, we finish to crete a VM but it has no memory, cpu.  

Now, the next step is to prepare memory!
In `kvm_ioctls`'s example, memory is prepared as follows

```rust
// First, setup Guest Memory using mmap
let load_addr: *mut u8 = unsafe {
    libc::mmap(
        null_mut(),
        mem_size, // 0x4000
        libc::PROT_READ | libc::PROT_WRITE,
        libc::MAP_ANONYMOUS | libc::MAP_SHARED | libc::MAP_NORESERVE,
        -1,
        0,
    ) as *mut u8
};

// Second, setup kvm_userspace_memory_region sructure using above memory
// kvm_userspace_memory_region is defined in kvm_bindings crate
let mem_region = kvm_userspace_memory_region {
    slot,
    guest_phys_addr: guest_addr,  // 0x1000
    memory_size: mem_size as u64, // 0x4000
    userspace_addr: load_addr as u64,
    flags: KVM_MEM_LOG_DIRTY_PAGES,
};
unsafe { vm.set_user_memory_region(mem_region).unwrap() };

// retrieve slice from pointer and length (slice::form_raw_parts_mut)
//   > https://doc.rust-lang.org/beta/std/slice/fn.from_raw_parts_mut.html
// and write asm_code into this slice (&[u8], &mut [u8], Vec<u8> impmenent the Write trait!)
//   > https://doc.rust-lang.org/std/primitive.slice.html#impl-Write
unsafe {
    let mut slice = slice::from_raw_parts_mut(load_addr, mem_size);
    slice.write(&asm_code).unwrap();
}
```

Check `set_user_memory_region`. This function will issue the following ioctl as a result, attach the memory to VM

```rust
ioctl(vmfd, KVM_SET_USER_MEMORY_REGION, &mem_region)
```

ToyVMM, on the other hand, provides a utility functions for memory preparation.  
This difference is due to the fact that ToyVMM's implementation is similaer to firecracker's, but they are essentially doing the same thing.  

Let's look at the whole implementation first

```rust
// The following `create_region` functions operate based on file descriptor, so first, create a temporary file and write asm_code to it.
let mut file = TempFile::new().unwrap().into_file();
assert_eq!(unsafe { libc::ftruncate(file.as_raw_fd(), 4096 * 10) }, 0);
let code: &[u8] = &[
    0xba, 0xf8, 0x03, /* mov $0x3f8, %dx */
    0x00, 0xd8,       /* add %bl, %al */
    0x04, b'0',       /* add $'0', %al */
    0xee,             /* out %al, %dx */
    0xb0, b'\n',      /* mov $'\n', %al */
    0xee,             /* out %al, %dx */
    0xf4,             /* hlt */
];
file.write_all(code).expect("Failed to write code to tempfile");

// create_region funcion create GuestRegion (The details are described in the following)
let mut mmap_regions = Vec::with_capacity(1);
let region = create_region(
    Some(FileOffset::new(file, 0)),
    0x1000,
    libc::PROT_READ | libc::PROT_WRITE,
    libc::MAP_NORESERVE | libc::MAP_PRIVATE,
    false,
).unwrap();

// Vec named 'mmap_regions' contains the entry of GuestRegionMmap
mmap_regions.push(GuestRegionMmap::new(region, GuestAddress(0x1000)).unwrap());

// guest_memory represents as the vec of GuestRegion
let guest_memory = GuestMemoryMmap::from_regions(mmap_regions).unwrap();
let track_dirty_page = false;

// setup Guest Memory
vm.memory_init(&guest_memory, kvm.get_nr_memslots(), track_dirty_page).unwrap();
```

The `create_vm` consequently performs a mmap in the following way and returns a part of the structure (GuestMmapRegion) representing the GuestMemory

```rust
pub fn create_region(
    maybe_file_offset: Option<FileOffset>,
    size: usize,
    prot: i32,
    flags: i32,
    track_dirty_pages: bool,
) -> Result<GuestMmapRegion, MmapRegionError> {

...

let region_addr = unsafe {
    libc::mmap(
        region_start_addr as *mut libc::c_void,
        size,
        prot,
        flags | libc::MAP_FIXED,
        fd,
        offset as libc::off_t,
    )
};
let bitmap = match track_dirty_pages {
    true => Some(AtomicBitmap::with_len(size)),
    false => None,
};
unsafe {
    MmapRegionBuilder::new_with_bitmap(size, bitmap)
        .with_raw_mmap_pointer(region_addr as *mut u8)
        .with_mmap_prot(prot)
        .with_mmap_flags(flags)
        .build()
}
```

Let's check the structure about Memory here.
In `src/kvm/memory.rs`, the following Memory structure is defined based on [vm-memory](https://github.com/rust-vmm/vm-memory) crate

```
pub type GuestMemoryMmap = vm_memory::GuestMemoryMmap<Option<AtomicBitmap>>;
pub type GuestRegionMmap = vm_memory::GuestRegionMmap<Option<AtomicBitmap>>;
pub type GuestMmapRegion = vm_memory::MmapRegion<Option<AtomicBitmap>>;
```

The `MmapRegionBuilder` is also defined in the `vm-memory` crate, and this `build` method creates the `MmapRegion`.

This time, since we have performed the mmap myself in advance and passed that address to `with_raw_mmap_pointer`, [use that area to initialize](https://github.com/rust-vmm/vm-memory/blob/f6ef1b619b126324830c87a3554b7082a0490ae0/src/mmap_unix.rs#L145-L147). Otherwise, [mmap is performed in the `build` method](https://github.com/rust-vmm/vm-memory/blob/f6ef1b619b126324830c87a3554b7082a0490ae0/src/mmap_unix.rs#L164-L173). In any case, this `build` method will get the `MmapRegion` structure, but defines a synonym as described above, which is returned as the `GuestMmapRegion`. By calling the `create_region` function once, you can allocate and obtain one region of GuestMemory based on the information(`size`, `flags`, ...etc) specified in the argument.

The region allocated here is only mmapped from the virtual address space of the VMM process, and no further information is available. To use this area as Guest Memory, a `GuestRegionMmap` structure is created from this area. This is simple, specify the corresponding `GuestAddress` for this region and initialize `GuestRegionMmap` with a tuple of mmapped area and GuestAddress. In following code, the initialized `GuestRegionMmap` is pushed to Vec for subsequent processing.

```rust
map_regions.push(GuestRegionMmap::new(region, GuestAddress(0x1000)).unwrap());
```

Now, the `mmap_regions: Vec<GuestRegionMmap>` created as above represents the entire memory of the Guest VM, and each region that makes up the guest memory holds information on the area allocated by the VMM for the region and the top address of the Guest side.
The `GuestMemoryMmap` structure representing the Guest Memory is initialized from this Vec information and set to VM by the `memory_init` method.

```rust
let guest_memory = GuestMemoryMmap::from_regions(mmap_regions).unwrap();
vm.memory_init(&guest_memory, kvm.get_nr_memslots(), track_dirty_page).unwrap();
```

Next, let's check the operation of this `memory_init`. This calls `set_kvm_memory_regions` and the actual process is described there.

```rust
pub fn set_kvm_memory_regions(
    &self,
    guest_mem: &GuestMemoryMmap,
    track_dirty_pages: bool,
    ) -> Result<()> {
    let mut flags = 0u32;
    if track_dirty_pages {
        flags |= KVM_MEM_LOG_DIRTY_PAGES;
    }
    guest_mem
        .iter()
        .enumerate()
        .try_for_each(|(index, region)| {
            let memory_region = kvm_userspace_memory_region {
                slot: index as u32,
                guest_phys_addr: region.start_addr().raw_value() as u64,
                memory_size: region.len() as u64,
                userspace_addr: guest_mem.get_host_address(region.start_addr()).unwrap() as u64,
                flags,
            };
            unsafe { self.fd.set_user_memory_region(memory_region) }
        })
    .map_err(Error::SetUserMemoryRegion)?;
    Ok(())
}
```

Here we can see that `set_user_memory_region` is called using the necessary information while iterating the region.
In other words, it is processing the same as the example code except that there may be more than one region.

Now that we've gone through the explanation of memory preparation, let's take a look at the `vm-memory` crate!
The information presented here is only the minimum required, so please refer to [Design](https://github.com/rust-vmm/vm-memory/blob/main/DESIGN.md) or other sources for more details.
This will also be related to the above iteration, where we were able to call methods such as `sart_addr()` and `len()` to construct the necessary information for `set_user_memory_region`.

```rust
GuestAddress (struct) : Represent Guest Physicall Address (GPA)
FileOffset(struct) : Represents the start point within a 'File' that backs a 'GuestMemoryRegion'

GuestMemoryRegion(trait) : Represents a continuous region of guest physical memory / trait
GuestMemory(trait) : Represents a container for a immutable collection of GuestMemoryRegion object / trait

MmapRegion(struct) : Helper structure for working with mmaped memory regions
GuestRegionMmap(struct & implement GuestMemoryRegion trait) : Represents a continuous region of the guest's physical memory that is backed by a mapping in the virtual address space of the calling process
GuestMemoryMmap(struct & implement GuestMemory trait) : Represents the entire physical memory of the guest by tracking all its memory regions
```

[Since `GuestRegionMmap` implements the `GuestMemoryRegion` trait](https://github.com/rust-vmm/vm-memory/blob/f6ef1b619b126324830c87a3554b7082a0490ae0/src/mmap.rs#L436), there are implementations of functions such as `start_addr()` and `len()`, which were used in the above interation.
The following figure briefly summarizes the relationship between these structures

![vm-memory_overview](./vm-memory_overview.svg)

As you can see, what is being done is essentially the same.  

The final step is prepareing vCPU (vCPU is a CPU to be attached to a virtual machine).  
Currently, a VM has been created and memory containing instructions has been inserted, but these is no CPU, so the instructions can't be executed. Therefore, let's create a vCPU, associate it with the VM, and execute the instruction by running the vCPU!

Using the file descriptor obtained during VM creaion (`vmfd`), the resulting `ioctl` will be issued as follows.

```rust
vcpufd = ioctl(vmfd, KVM_CREATE_VCPU, 0)
```

The `create_vm` method that was just issued to obtain the `vmfd` is designed to return a `kvm_ioctls::VmFd` strucure as a result, and by execuing the `create_vcpu` method, which is a method of this structure, the above ioctl is consequently issued and returns the result as a `kvm_ioctls::VcpuFd` structure.  

`VcpuFd` provides utilities for getting and setting various CPU states.  
For example, if you want o get/set a register set from the vCPU, you would normally issue the following `ioctl`

```rust
ioctl(vcpufd, KVM_GET_SREGS, &sregs);
ioctl(vcpufd, KVM_SET_SREGS, &sregs);
```

For these, the following methods are available in `kvm_ioctls::VcpuFd`

```rust
get_sregs(&self) -> Result<kvm_sregs>
set_sregs(&self, sregs: &kvm_sregs) -> Result<()>
```

`VcpuFd` also provids a method called `run`, which issues the following insructions to actually run the vCPU.

```rust
ioctl(vcpufd, KVM_RUN, NULL)
```

and then, we can aquire return values that has the type `Result<VcpuExit>` resulting this method.

When running vCPU, exit occurs for various reasons. This is an instruction that the CPU cannot handle, and the OS usually tries to deal with it by invoking the corresponding handler.  
If this type of exit comes back from the VM's vCPU, as in the case, it will be necessary to write the appropriate code to handle the situation.  
`VcpuExit` is defined in `kvm_ioctls::VcpuExit` as enum.  
When Exit are occurred on several reasons in running vCPU, the exit reasons that are defined in kvm.h in linux kernel are wrapped to `VcpuExit`.  
Therefore, it is sufficient to write a process that pattern matches this result and appropriately handles the error to be handled.  

Now, there is a instruction that execute outputting values through I/O port and this will occur the `KVM_EXIT_IO_OUT`.  
`VcpuExit` wrap this exit reason as `IoOut`.

Originally (in C programm as example), we require to calculate appropriate offset to get output data from I/O port, but now, this process are implemented in `run` method and returned as VcpuExit that contains necessary values.  
So, we don't have to write these unsafe code (pointer offset calculation) and handle these exit as you will.  

```rust
loop {
    match vcpu.run().expect("vcpu run failed") {
        kvm_ioctls::VcpuExit::IoOut(addr, data) => {
            println!(
                "Recieved I/O out exit. \
                Address: {:#x}, Data(hex): {:#x}",
                addr, data[0],
            );
        },
        kvm_ioctls::VcpuExit::Hlt => {
            break;
        }
        exit => panic!("unexpected exit reason: {:?}", exit),
    }
}
```

In above, only handle `KVM_EXIT_IO_OUT` and `KVM_EXIT_HLT`, and the others will be processed as panic. (Although all exits should be handled, I want to focus on the description of KVM API example and keep it simply)  

Since we are here, let's take a look at the processing of the `run` method in some detail.  
Let's check the processing of `KVM_EXIT_IO_OUT`.  

If you look at the [LWN article](https://lwn.net/Articles/658511/), you will see that it calculates the offset and outputs the necessary information in the following way.  

```rust
case KVM_EXIT_IO:
    if (run->io.direction == KVM_EXIT_IO_OUT &&
	    run->io.size == 1 &&
	    run->io.port == 0x3f8 &&
	    run->io.count == 1)
	putchar(*(((char *)run) + run->io.data_offset));
    else
	errx(1, "unhandled KVM_EXIT_IO");
    break;
```

On the other hand, `run` method implemented in `kvm_ioctl::VcpuFd` is like bellow

```rust
...
let run = self.kvm_run_ptr.as_mut_ref();
match run.exit_reason {
    ...
    KVM_EXIT_IO => {
        let run_start = run as *mut kvm_run as *mut u8;
        // Safe because the exit_reason (which comes from the kernel) told us which
        // union field to use.
        let io = unsafe { run.__bindgen_anon_1.io };
        let port = io.port;
        let data_size = io.count as usize * io.size as usize;
        // The data_offset is defined by the kernel to be some number of bytes into the
        // kvm_run stucture, which we have fully mmap'd.
        let data_ptr = unsafe { run_start.offset(io.data_offset as isize) };
        // The slice's lifetime is limited to the lifetime of this vCPU, which is equal
        // to the mmap of the `kvm_run` struct that this is slicing from.
        let data_slice = unsafe {
            std::slice::from_raw_parts_mut::<u8>(data_ptr as *mut u8, data_size)
        };
        match u32::from(io.direction) {
            KVM_EXIT_IO_IN => Ok(VcpuExit::IoIn(port, data_slice)),
            KVM_EXIT_IO_OUT => Ok(VcpuExit::IoOut(port, data_slice)),
            _ => Err(errno::Error::new(EINVAL)),
        }
    }
		...
```

Let me explain a little. The `kvm_run` is provided by the [`kvm-bindings`](https://github.com/rust-vmm/kvm-bindings) crate, which is a structure automatically generated from a header file using [`bindgen`](https://github.com/rust-lang/rust-bindgen), so it is a structure like the linux kernel's `kvm_run` converted directory to Rust.  
First, `kvm_run` is obtained in the form of a pointer, a method of obtaining a pointer often used in Rust.  
This correspoinds to the first address of the `kvm_run` structure which is bound to `run_start` variable.  
And the information corresponding to `run->io(.member)` can be obtained from `run.__bindgen_anon_1.io`, although it is a bit tricky. The field named `__bindgen_anon_1` is the effect of automatic generation by `bindgen`.  
The data we want is at the first address of `kvm_run` plus `io.data_offset`. This process is performed in `run_start.offset(io.data_offset as isize)`. And the data size can be calculated from `io->size` and `io->count` (in the LWN example, it is 1byte, so it's taken directory from the offset by putchar). This part is calculated and stored in the value `data_size`, and `std::slice::from_raw_parts_mut` actually retrieves the data using this size.  
Finally, checking `io.direction`, we change the wrap type for `KVM_EXIT_IO_IN` or `KVM_EXIT_IO_OUT` respectively, and return the descired information such as `port` and `data_slice` together.  

As can be seen from the above, what is being done is clear.  
However, it still contains many unsafe operations because it involves pointer manipuration.  
We can see that by using these libraries, we are able to implement VMM on a stable implementation.  

Well, it's ben a long time comming, but let's take a look back at the rust-vmm crates we're using again.  

```rust
kvm-bindings : Library that includes structures automatically generated from kvm.h by bindgen.
kvm-ioctls : Library that hides ioctl and unsafe processes related to kvm operations and provides user-friendly sructures, functions and methods.  
vm-memory : Library that provides structures and operations to the Memory
```

This knowledge will come up again and again in future discussion and is basic and important.  
