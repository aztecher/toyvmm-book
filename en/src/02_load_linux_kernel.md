# Load Linux Kernel

In this section, we will explain upon the implementation of launching a Guest VM as the first step in VMM. While our VMM has minimal functionality, booting the Linux Kernel demands a variety of knowledge.

In this section, we will explain the essential aspects of launching a Guest VM and delve into how it is implemented in ToyVMM. To achieve this, we will divide it into several detailed chapters and provide explanations for each topic.

The topics are as follows:

* [02-1. Overview of Booting Linux](./02-1_overview_of_booting_linux.md)
* [02-2. ELF binary format and vmlinux structure](./02-2_elf_binary_format_and_vmlinux_structure.md)
* [02-3. Loading initrd](./02-3_loading_initrd.md)
* [02-4. Setup registers of vcpu](./02-4_setup_registers_of_vcpu.md)
* [02-5. Serial console implementation](./02-5_serial_console_implementation.md)
* [02-6. ToyVMM implementation](./02-6_toyvmm_implementation.md)

Additionally, this document is based on the following commit numbers:

* ToyVMM: `27fdf196dfb31938f24785ca64e7233a6dc8fceb`
* Firecracker: `4bf121fc032cc2d94a20a3625f2af3918545154a`

If you refer to this document while inspecting ToyVMM's code, it may be beneficial.
