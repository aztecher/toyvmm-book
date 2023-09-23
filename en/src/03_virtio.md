# Virtual I/O Device (Virtio)

In this section, as the second step of VMM, we will delve into the implementation of Virtio.  
The Virtio specification is maintained by OASIS.  
The latest version appears to be [version 1.2](http://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html), which was published on July 1, 2022.  
The terminology related to Virtio in this document follows the definitions in version 1.2, so if you want to confirm the meaning of specific terms, please refer to the OASIS page.

In this section, we will cover fundamental knowledge about Virtio and its implementation.  
Additionally, as concrete implementations based on Virtio, we will work on `virtio-net` and `virtio-blk`.  
Once `virtio-net` is implemented, you will be able to communicate with a booted Guest VM over the network, enabling SSH login and internet connectivity.  
Moreover, with `virtio-blk` implemented, you will handle block devices, meaning DISK I/O, within the virtual machine.
With these two functionalities, you will have most of the requirements for a typical "virtual machine" in place, making Virtio implementation highly significant.

The topics in this section are structured as follows:

* [03-1. Virtio](./03-1_virtio.md)
* [03-2. Implement virtio in ToyVMM](./03-2_implement_virtio_in_toyvmm.md)
* [03-3. Implement virtio-net](./03-3_virtio-net.md)
* [03-4. Implement virtio-blk](./03-4_virtio-blk.md)

This document is based on the following commit numbers:

* ToyVMM: 58cf0f68a561ee34a28ae4e73481f397f2690b51
* Firecracker: cfd4063620cfde8ab6be87ad0212ea1e05344f5c

From this point onwards, we will explain the implemented source code using file names.  
Here are the actual file paths referred to by the file names mentioned in the explanations:

| File Name Mentioned in Explanations | File Path                                      |
|-------------------------------------|------------------------------------------------|
| `mod.rs`                            | src/vmm/src/devices/virtio/mod.rs              |
| `queue.rs`                          | src/vmm/src/devices/virtio/queue.rs            |
| `mmio.rs`                           | src/vmm/src/devices/virtio/mmio.rs             |
| `status.rs`                         | src/vmm/src/devices/virtio/status.rs           |
| `virtio_device.rs`                  | src/vmm/src/devices/virtio/virtio_device.rs    |
| `net.rs`                            | src/vmm/src/devices/virtio/net.rs              |
| `block.rs`                          | src/vmm/src/devices/virtio/block.rs            |

Please note that these file paths may change in the future as source code is updated.  
Consider these file paths to be associated with the commit numbers mentioned earlier.
