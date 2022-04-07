# QuickStart

## Prerequisites

Since this project is based on KVM, it's desiable to have KVM setup in the development environment.
In addition, Docker installation is required since the code testing and execution is basically intended to be performed inside a Docker container.

## Run ToyVMM!

Running `make run` executes cargo run on the development environment, and running `make run_container` executes it inside the container. Currently running code equivalent to the example of LWN article 「[Using the KVM API](https://lwn.net/Articles/658511/)」and similar to [kvm_ioctls' example](https://docs.rs/kvm-ioctls/latest/kvm_ioctls/#example---running-a-vm-on-x86_64)

```bash
$ make run
sudo -E cargo run
   Compiling bitflags v1.3.2
   Compiling libc v0.2.121
   Compiling vmm-sys-util v0.9.0
   Compiling vm-memory v0.7.0
   Compiling kvm-bindings v0.5.0
   Compiling kvm-ioctls v0.11.0
   Compiling toyvmm v0.1.0 (/home/mmichish/Documents/rust/toyvmm)
    Finished dev [unoptimized + debuginfo] target(s) in 5.43s
     Running `target/debug/toyvmm`
Recieved I/O out exit. Address: 0x3f8, Data(hex): 0x34
Recieved I/O out exit. Address: 0x3f8, Data(hex): 0xa
sudo rm -rf target
```

## What's next?

Check [01. Running Tiny Code in VM](./01_running_tiny_code_in_vm.md) for a detailed explanation of the ToyVMM implementation!
