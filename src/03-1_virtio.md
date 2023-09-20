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
一方で、VM内部でのI/O要求が発生する度にVMExitが発生し、Hypervisor側でエミュレーション処理を実施した後、VMに処理を戻す必要があるため、大きなオーバーヘッドを伴うようになる。

上記のようなデバイスI/Oにおける仮想化のオーバーヘッドを低減する方法の一つとして提案され標準化されているフレームワークが「Virtio」である。
VirtioではHypervisorとVMの間で共有されたメモリ上にVirtqueueと呼ばれるキュー構造を作成し、これを用いてデータの入出力を行う方式を採用することで、VMExitによるモード遷移回数を極力抑制するような仕組みになっている。
一方で、Virtio向けのデバイスドライバは別途必要になり、これはカーネルビルド時の設定に依存することになる。とはいえ、現在の多くのOSではデフォルトでVirtioデバイスドライバはデフォルトでインストールされていると思われる。

### Virtioの構成要素

Virtioは概ね以下の構成要素からなる

* VirtQueue : ホスト・ゲスト間で共有されたメモリ領域上に構築されるキュー。Transport Layoutとも呼ばれる。これを利用してデータの入出力を実施する。  
* Virtio driver : Virtioをベースとしたデバイスに対するGuest側のドライバ。Front End Driverとも呼ばれる。
* Virtio device : Hypervisor側でエミュレーションするデバイスの実装。Back End Driverとも呼ばれる。

TODO: Overviewの図を書く  
TODO: [VirtIO Architecture](https://blogs.oracle.com/linux/post/introduction-to-virtio)あたりを参考に書くと良さそうかも  
TODO: [virtio: Towards a De-Facto Standard For Virtual I/O Devices](https://ozlabs.org/~rusty/virtio-spec/virtio-paper.pdf)あたりを読んでうまいことまとめる  

TBD: この辺りでVirtIOのInternalについては03-2で書いて説明、virtio-net,blkを-3,-4で記載する感じなので一旦ここでは概要だけ捉える

Hypervisorとゲストの間の通信方式として、PCI（Peripheral Component Interconnect）と、MMIOを（Memory Mapped I/O）のいずれかのTransport層を選択することができる。前者は`virtio-pci`、後者は`virtio-mmio`などと呼ばれ、本資料もこの呼称に準じることとする。

TODO: Overview図を書く。Transportとして選択肢があるよーみたいな図が欲しい

ToyVMMではひとまず`virtio-mmio`をTransport層として採用し、その上で`virtio-net`および`virtio-blk`を実装していくこととする。


### References

* [OASIS](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=virtio)
* [Virtio: An I/O virtualization framework for Linux](https://www.cs.cmu.edu/~412/lectures/Virtio_2015-10-14.pdf)
* [virtio: Towards a De-Facto Standard For Virtual I/O Devices](https://ozlabs.org/~rusty/virtio-spec/virtio-paper.pdf)
* [Introduction to VirtIO](https://blogs.oracle.com/linux/post/introduction-to-virtio)
