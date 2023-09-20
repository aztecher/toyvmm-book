# Virtual I/O Device (Virtio)

**chapter3 is currently available only in Japanese, but will soon be translated into English ;)**

このセクションではVMMのセカンドステップとして、Virtioの実装について触れることにする。  
VirtioについてはOASISが仕様の策定とメンテナンスを行なっている。  
現在は2022/07/01に公開された[version 1.2](http://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html)が最新のようである。

このセクションではVirtioに関する基礎的な知識の習得とその実装について触れる。  
また、Virtioをベースとした具体的なデバイスの実装として、`virtio-net`、`virtio-blk`についても実装していく。  
`virtio-net`が実装できると、起動したGuest VMにたいしてネットワーク経由で疎通することになるため、sshログインできたり、インターネットと繋いだりすることができるようになる。  
また、`virtio-blk`が実装できると、仮想マシン上でブロックデバイス、つまりDISK I/Oを取り扱うことができるようになる。
この二つの機能により、一般的に想定される「仮想マシン」の要件が概ね揃うことになるため、Virtioの実装は極めて重要である。  

本セクションの各トピックは次のようになっている

* [03-1. Virtio](./03-1_virtio.md)
* [03-2. Implement virtio in ToyVMM](./03-2_implement_virtio_in_toyvmm.md)
* [03-3. Implement virtio-net](./03-3_virtio-net.md)
* [03-4. Implement virtio-blk](./03-4_virtio-blk.md)
* [03-5. Play with VM](./03-5_play_with_vm.md)
* [03-6. Refactoring](./03-6_refactoring.md)

また、本資料は以下のコミットナンバーをベースとしている

* ToyVMM : 
* Firecracker :

上記のコミットナンバはVirtioの実装が一通り完了したのちにリファクタリングした後のコミットになっている。  
リファクタリングの内容などについては[03-6. Refactoring](./03-6_refactoring.md)に記載しているので必要に応じて参照されたい。

以降では実装されているソースコードをファイル名を用いて説明することになる。
説明の中で出てくるファイル名が指している実際のファイルパスをまとめておく。

| 説明の中で触れるファイル名 | ファイルパス                        |
|----------------------------|-------------------------------------|
| `mod.rs`                   | src/devices/virtio/mod.rs           |
| `queue.rs`                 | src/devices/virtio/queue.rs         |
| `mmio.rs`                  | src/devices/virtio/mmio.rs          |
| `status.rs`                | src/devices/virtio/status.rs        |
| `virtio_device.rs`         | src/devices/virtio/virtio_device.rs |
| `net.rs`                   | src/devices/virtio/net.rs           |
| `block.rs`                 | src/devices/virtio/block.rs         |

なお、今後ソースコードに修正がかかることが想定されるため上記のファイルパスは変更される可能性がある。  
あくまでこのファイルパスは上記のコミット番号におけるファイルパスだと認識してほしい　
