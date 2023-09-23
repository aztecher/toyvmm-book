# Load Linux Kernel

このセクションではVMMのファーストステップとしてGuest VMを起動するための実装について触れることとする。  
VMMとしては必要最低限の機能である一方、Linux Kernelの起動にあたってさまざまな知識を要求される内容でもある。  

このセクションではGuest VMを起動するための最低限の事項について説明し、ToyVMMでどのように実装しているかについても触れていく
そのため、いくつかの細かいチャプターに分割しトピックごとに説明をしていくことにしよう

トピックとしては次のようになっている。  

* [02-1. Overview of Booting Linux](./02-1_overview_of_booting_linux.md)
* [02-2. ELF binary format and vmlinux structure](./02-2_elf_binary_format_and_vmlinux_structure.md)
* [02-3. Loading initrd](./02-3_loading_initrd.md)
* [02-4. Setup registers of vcpu](./02-4_setup_registers_of_vcpu.md)
* [02-5. Serial Console implementation](./02-5_serial_console_implementation.md)
* [02-6. ToyVMM implementation](./02-6_toyvmm_implementation.md)

また、本資料は以下のコミットナンバーをベースとしている

* ToyVMM : `27fdf196dfb31938f24785ca64e7233a6dc8fceb`
* Firecracker : `4bf121fc032cc2d94a20a3625f2af3918545154a`

本資料をToyVMMのコードを参照しながら確認する場合は参考にされたい。
