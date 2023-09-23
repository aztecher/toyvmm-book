# Virtio

### Virtual I/O Device (Virtio) とは

Virtioは、[OASIS](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=virtio) によって標準化されている仮想デバイスに関する仕様であり、ホストシステムとゲストシステム（仮想マシン）間で効率的なデータ転送や通信を行うための仮想デバイスインターフェイスを提供している。

Virtioをベースとした実装に`virtio-net`（仮想ネットワークデバイス）、`virtio-blk`（仮想ブロックデバイス）などが存在し、その名の通りそれぞれがネットワークデバイス／ブロックデバイスとしての振る舞いを実装している。  
これによりゲストOSはあたかも実際のネットワークデバイスやブロックデバイスを利用するかのようにI/Oを実施することができるようになっている。

Virtioは、KVMをはじめとする主要な仮想化技術と互換性があり、Linux、Windows、FreeBSDなど多くのゲストOSに対応しているため、仮想化環境において広く採用されており、業界標準の仕様となっている。

### なぜVirtioが必要か?

そもそもVM上でI/Oを発生させたい時、Hypervisorの観点からはどう処理すればいいだろうか？
何よりもまず、VM起動時にデバイスを認識させなければならず、素朴には各種PCIデバイスのエミュレーションを行う必要があるだろう。  
それ以外にも、そのデバイスに対してI/Oを発生させた際に、そのデバイスの振る舞いを模倣する必要がある。
この類のハードウェアエミュレータとして有名で広く利用されるソフトウェアとして[QEMU](https://www.qemu.org/)が存在する。

実ハードウェアを完全にソフトウェアでエミュレーションできれば、ゲストOSに付属している実ハードウェア向けのデバイスドライバをそのまま利用できるというメリットがある。  
しかしこの方式は、VM内部でのI/O要求が発生する度にVMExitが発生し、Hypervisor側でエミュレーション処理を実施した後、VMに処理を戻す必要があるため、大きなオーバーヘッドを伴うことになる。

上記のようなデバイスI/Oにおける仮想化のオーバーヘッドを低減する方法の一つとして提案され標準化されているフレームワークが「Virtio」である。
VirtioではHypervisorとVMの間で共有されたメモリ上にVirtqueueと呼ばれるキュー構造を作成し、これを用いてデータの入出力を行う方式を採用することで、VMExitによるモード遷移回数を抑制するような仕組みになっている。
一方で、Virtio向けのデバイスドライバは別途必要になり、これはカーネルビルド時の設定に依存することになるが、現在の多くのOSではデフォルトでVirtioデバイスドライバはデフォルトでインストールされている。

### Virtioの構成要素

Virtioは概ね以下の構成要素からなる

* Virtqueue : ホスト・ゲスト間で共有されたメモリ領域上に構築されるキュー。これを介してデータの入出力を実施する。  
* Virtio driver : Virtioをベースとしたデバイスに対するゲスト側のドライバ。
* Virtio device : ホスト側でエミュレーションするデバイスの実装。

<img src="./03_figs/virtio-overview.svg" width="100%">

図にもある通り、ゲストから発行されたI/OリクエストはVirtqueueを仲介してホストに渡り、ホストからの返答も同様にVirtqueueを仲介してゲストに戻ってくる。
より詳細な挙動や実装についての解説は次節で行う。

また、Virtioデバイスをゲストに対してさらす際、特定のTransport methodを選択して提供することが可能である。
代表的なものとしてはPCI（Peripheral Component Interconnect）を利用した「Virtio Over PCI Bus」方式と、MMIO（Memory Mapped I/O）を利用した「Virtio Over MMIO Bus」である。
ゲストには、特定のTransportに紐づくドライバである`virtio-pci`や`virtio-mmio`などが存在し、更に特定のデバイスタイプ向けのvirtioドライバ（`virtio-net`, `virtio-blk`）が存在する。

ToyVMMではひとまず`virtio-mmio`をTransportとして採用し、その上で`virtio-net`向けのNetworkデバイス、及び`virtio-blk`向けのBlockデバイスを実装していくこととする。

### References

* [OASIS](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=virtio)
* [Virtio: An I/O virtualization framework for Linux](https://www.cs.cmu.edu/~412/lectures/Virtio_2015-10-14.pdf)
* [virtio: Towards a De-Facto Standard For Virtual I/O Devices](https://ozlabs.org/~rusty/virtio-spec/virtio-paper.pdf)
* [Introduction to VirtIO](https://blogs.oracle.com/linux/post/introduction-to-virtio)
* [Virtio on Linux](https://docs.kernel.org/driver-api/virtio/virtio.html)
