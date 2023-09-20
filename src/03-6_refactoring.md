# Refactoring

リファクタリングの内容を記載する。  
大きな変更を伴う部分のみではあるが、従来のディレクトリ構造から大きな変化があるため


Before


```bash
  tree -I 'target'                                                                                                           [09:24:32]
.
├── Cargo.lock
├── Cargo.toml
├── Dockerfile
├── initrd.img
├── LICENSE.txt
├── Makefile
├── README_ja.md
├── README.md
├── src
│   ├── arch
│   │   ├── gdt.rs
│   │   ├── mod.rs
│   │   ├── msr_index.rs
│   │   └── x86_64.rs
│   ├── builder.rs
│   ├── device_manager.rs
│   ├── devices
│   │   ├── bus.rs
│   │   ├── epoll.rs
│   │   ├── legacy
│   │   │   ├── mod.rs
│   │   │   └── serial.rs
│   │   ├── mod.rs
│   │   ├── util
│   │   │   ├── mod.rs
│   │   │   ├── net
│   │   │   │   ├── mod.rs
│   │   │   │   └── tap.rs
│   │  │   └── sys
│   │   │       ├── mod.rs
│   │   │       ├── net
│   │   │       │   ├── iff.rs
│   │   │       │   ├── if_tun.rs
│   │   │       │   ├── inn.rs
│   │   │       │   ├── mod.rs
│   │   │       │   └── sockios.rs
│   │   │       └── virtio
│   │   │           ├── mod.rs
│   │   │           └── net.rs
│   │   └── virtio
│   │       ├── block.rs
│   │       ├── mmio.rs
│   │       ├── mod.rs
│   │       ├── net.rs
│   │       ├── queue.rs
│   │       ├── status.rs
│   │       ├── types.rs
│   │       └── virtio_device.rs
│   ├── kvm
│   │   ├── memory.rs
│   │   ├── mod.rs
│   │   ├── vcpu.rs
│   │   └── vm.rs
│   ├── lib.rs
│   ├── main.rs
│   ├── utils
│   │   ├── byte_order.rs
│   │   ├── memory.rs
│   │   ├── mock_resources
│   │   │   ├── mod.rs
│   │   │   └── test_elf.bin
│   │   ├── mod.rs
│   │   ├── test_utils
│   │   │   └── mod.rs
│   │   └── util.rs
│   ├── vm_control.rs
│   ├── vmm.rs
│   └── vm_resources.rs
├── tests
│   └── integration_tests.rs
├── ubuntu-18.04.ext4
└── vmlinux.bin

15 directories, 57 files
```

After


TBD
* 一番上はworkspaceにする。
* src/toyvmm/Cargo.toml / CLI系、workspace Cargo.tomlのdefault-member
* src/vmm/Cargo.toml / VMMの実装
  * src/vmm/src/{arch,device_manager,devices,...}
* src/utils/Cargo.toml / Utility系
* src/net_gen/Cargo.toml / Tapデバイス系のCバインド
* src/virtio_gen/Cargo.toml / Virtio系のCバインド
* tests/{data,integration_tests} / Integration/Functional Test (Unit Test is under each files)
* tools / ツール系。開発補助やテスト、起動試す系。Dockerファイルとかにしてるといいかもしれない気持ちもある。

そのほか
* コード自体の全体的なリファクタリング
* .cargoってなんだ？環境統一できたりする？
* build.rsに何かさせられる？
* Travis CIのクレジット欲しい
* Rust Lint, Formattingのツールを導入して自動でやって欲しい
