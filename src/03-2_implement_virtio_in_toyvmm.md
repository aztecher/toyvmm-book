# Implement virtio in ToyVMM

本章では、Virtioの具体的な実装について記載する。  
[前節](./03-1_virtio.md)でも述べた通りToyVMMではTransportとしてひとまずMMIOを利用する。  

具体的な説明に入る前に、以下に今回実装したVirtioの実装の全体像を図示する。  

TODO: 実装ベースでのVirtioの全体図を差し込む

この図を適宜見返しつつ、以降の説明とコードを読み進めることで理解が深まるはずである。

### Virtioの実装

[前節](./03-1_virtio.md)でも述べた通り、Virtioそのものはフレームワークのようなものである。  
そのため実装においては、Virtio自体は抽象概念 (Rustなのでtrait) として実装する。  
構造体としての実装は`virtio-net`、`virtio-blk`などのVirtioをベースとしたデバイスで行い、それらはVirtioのtraitを実装するという形を考える。  
また、冒頭でも述べたとおりバスについても選択肢があるため、Transportも抽象概念として実装しデバイスに合わせて実装する形にする。  
最後にVirtQueueの実装が必要になるが、これはVirtioデバイスによって個数や利用方法は異なるものの、その構造自体は変化しないので素直に実装する。

### VirtQueueの実装

まず、VirtQueueの構造について把握しておく。VirtQueueは以下のようになっている。

TODO: 図

図にある通り、VirtQueueは主に `Desciptor Table`、`Available Ring`、`Used Ring`の3種類で構成されている。  
TODO: 図をいい感じに作って簡単に説明する。
TODO: Descriptor Table / Descriptor Entry / Avail Ring / Used Ringなどの各構造体の詳細を書いていく

`queue.rs`にVirtQueueの実装を行っている。
この実装もfirecrackerの実装をベースとしている。

後述するが、`Descriptor Table`、`Available Ring`、`Used Ring`の具体的なGuest Memory上のアドレスや`VirtQueue`のサイズなどは、Guest VM起動時のGuest側のdevice driverとのやりとりによって設定されることになる。
またdevce driverは初期化の段階で、全てのDescriptorの初期化を行い、nextの値には次のDescriptorのエントリ番号を入れてチェーンの作成を行うが、チェーンの構造についてはToyVMMに情報が提供されないので、`Descriptor Table`の先頭アドレスと仕様をベースとしてアドレスアクセスを行っていく必要がある。
基本的にDescriptorのエントリ単位で処理を行うことが想定される（このエントリの先に具体的なデータのアドレスが格納されている）
このデータ処理に伴って`Available Ring`や`Used Ring`を更新する形になる。

`Queue`という構造体が`VirtQueue`を表現する構造体になっているが、これは`iter`を実装している。
この`iter`の返り値である`AvailIter`が`Iterator`トレイトを実装しており（`next`関数を実装しており）、nextによって`DescriptorChain`構造体を取り出すような形になっている。  
これにより、コード上では以下のように表現できるようになっている。  

```rust
// queue : Queue構造体
// desc_chain: DescriptorChain構造体
for desc_chain in queue.iter(mem) {
  // desc_chain には、descriptorのaddr,len,flags,nextの値が格納されている
  // iterationする裏で、queue.avail_ringに関するデータの調整が行われている
}
```

`Iterator`の実装の中で`Available Ring`の処理や`DescriptorChain`のアドレスアクセスなどを隠蔽している。  
少しこの部分について説明していく。

まず`iter`関数であるが、ここで重要なのは`avail_ring`から`idx`（これはリング上の一番最新のエントリのindexを指す）を取得しているところである。  
`avail_ring`から`2byte`オフセットした位置（= `avail_ring`の先頭2byteは`flags`メンバ）から`u16`分取得するコードになっている。  
今後この手のコードが散見されることになるのでここで理解することが望ましい。

```rust
pub fn iter<'a, 'b>(&'b mut self, mem: &'a GuestMemoryMmap) -> AvailIter<'a, 'b> {
  // (省略)
  let queue_size = self.actual_size();
  let avail_ring = self.avail_ring;

  // Access the 'idx' member of available ring
  // skip 2byte (= u16 / 'flags' member) from avail_ring address
  // and get 2byte (= u16 / 'idx' member that represents the newest index of avail_ring) from that address.
  let index_addr = mem.checked_offset(avail_ring, 2).unwrap();
  let last_index = mem.read_obj(index_addr).unwrap();

  AvailIter {
      mem,
      desc_table: self.desc_table,
      avail_ring: self.avail_ring,
      next_index: self.next_avail,
      last_index: Wrapping(last_index),
      queue_size,
      next_avail: &mut self.next_avail,
  }
}
```

上記の通り`iter`関数を呼び出すことで`AvailIter`が返却されるが、イテレーション処理ではこの`AvailIter`が実装している`next`関数が呼び出される。
`next`関数は`Option<T>`を返却するように実装し、値がある場合はその値がイテレーション時の要素になり、`None`を返すとイテレーション終了になる。
さて、では`AvailIter`の`next`関数の実装をみてみる。

```rust
impl<'a, 'b> Iterator for AvailIter<'a, 'b> {
  type Item = DescriptorChain<'a>;

  fn next(&mut self) -> Option<Self::Item> {
    if self.next_index == self.last_index {
      return None;
    }

    let offset = (4 + (self.next_index.0 % self.queue_size) * 2) as usize;
    let avail_addr = match self.mem.checked_offset(self.avail_ring, offset) {
      Some(a) => a,
      None => return None,
    };
    // This index is checked below in checked_new
    let desc_index: u16 = self.mem.read_obj(avail_addr).unwrap();

    self.next_index += Wrapping(1);

    let ret = DescriptorChain::checked_new(self.mem, self.desc_table, self.queue_size, desc_index);
    if ret.is_some() {
      *self.next_avail += Wrapping(1);
    }
    ret
  }
}
```

`self.last_index`が`iter`関数で取得した現在の`avail_ring`における最新要素のindex（= ここまで処理したい、という値）であり、`self.next_index`がこれまで処理してきたindexの値になる。
この`next`の中で`self.next_index`の値をインクリメントしていくため、Iterationをしていく中でいずれ`self.next_index`と`self.last_index`の値が一致する（= 必要なところまで処理しきる）ところまで進みNoneが帰ることで繰り返しが終了する、というのが大枠である。
さらにこの関数のなかで、`self.next_index`が指す`avail_ring`上の要素（= descriptorのindexを指している）を取り出し、それを利用して`DescriptorChain::checked_new`を呼び出している。
この`checked_new`関数では関数に渡した上記のindex値から具体的にその要素のアドレスを割り出した上でアドレスアクセスして、`desciptor`が指しているデータのアドレス(`addr`)やデータ長(`len`)、フラグ(`flags`)、次のdesciptorの番号(`next`)を取り出したうえで、`DescriptorChain`構造体を構成する。
`next`関数の帰り値としては上記で作成した`DesciptorChain`を返却しているので、ループ内部ではDescriptorの情報にアクセスする際は`DescriptorChain`構造体の該当するメンバにアクセスするだけで良い形にしてある。


### Guest <-> Hostの通知の発生経路

GuestとHostの間での通知は`ioeventfd`と`irqfd`の仕組みを利用する。

まずGuestからHostに対する通知であるが、これは`ioeventfd`を利用して実装する。
`ioeventfd`はGuest VMで発生したPIO/MMIOによるメモリへの書き込みをeventfdの通知に変化する仕組みである。
KVM APIとしては`KVM_IOEVENTFD`を利用し、引数に通知を受けるeventfdやMMIOのアドレスを受け取る形になっており、この引数のMMIOアドレスへの書き込みが、同じく引数のeventfdへの通知に変換される形になるため、KVM APIを呼び出したソフトウェア（= ToyVMM）側で、ゲストからの通知を受け取ることができる。
この仕組みはイベント通知の効率を向上させるための仕組みであり、従来のポーリングや割り込みハンドラによる方式に比べ軽量な実装になっている。

次にHostからGuestに対する通知だが、これは`irqfd`の仕組みを利用する。
`irqfd`はこれまでの実装でも既に利用しているが、`KVM_IRQFD`を利用し、引数に通知に利用するeventfdと対応して発火させたいGuestのirq番号を渡すことで、ToyVMM側でのeventfdへのwriteが、Guestのハードウェア割り込みとなってGuestへ通知が発生することになる。

具体的な使い方は以降で触れることとする。

### MMIO Transportの実装

ここからはMMIO Transportの実装について説明していく。
[Virtio Over MMIO](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1650002)が公式の仕様なので必要に応じてこちらを参照されたい。

MMIO TransportはPCIサポートのない仮想環境などで簡単に利用できる方式である。  
firecrackerではPCIのサポートは無く、基本的にMMIO Transportのみサポートしているようである。  
MMIO Transportは特定のメモリ領域(Register)へのRead/Writeを通して、デバイス操作を行う方式である。  

MMIO TransportはPCIのような汎用のDevice discovery仕組みを利用できない。
そのため、MMIOにおけるDevice discoveryは[MMIO Device Discovery](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1660001)で説明のある通り、ゲストOSに対してそのメモリマップされたデバイスの位置と割り込み位置について提供する必要がある。
この方法としてDevice Treeに記載するという方式が公式では書かれているが、それ以外に[カーネル起動時のコマンドライン引数に埋め込む方法](https://docs.kernel.org/admin-guide/kernel-parameters.html)が存在しており、Guest VM起動時にこのコマンドライン引数が動的に調整できる利点もあったため今回は後者の方式をとることとした。

この方式では以下のような書式に従って記述することでMMIO deviceのdiscoveryを行うようにGuest VMに情報を提供することができる。  

```bash
virtio_mmio.device=<size>@<baseaddr>:<irq>

example:
virtio_mmio.device=4K@0xd0000000:5
```

この場合、Guest VMは0xd0000000のアドレスを起点として、決まったオフセット（Register）にRead/Writeを実施することでデバイスの初期化や設定を実施する。
[MMIO Device Register Layout](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1670002)に記載してあるのがその全てである。
ToyVMMの観点からみると、それぞれのRegisterに対してRead/Writeが発生した際に、上記のSpecに従った処理が実施されることを保証しなければならず、これを実装するというのがMMIO Transportの実装の中核であるといえる。
この実装部分の詳細は後述するが、MMIO Transportではoffsetが0x050の位置にWriteすることが、デバイス（= ToyVMMと読み替えて良い）に対して、処理すべきデータがバッファに存在する、ということを通知する処理に該当する。
つまり、このアドレスと、実装するVirtioデバイスのVirtQueueに紐づくEventfdを`KVM_IOEVENTFD`によって紐づけることで、Guestで発生したNotify（MMIOへの書き込み）が、直接Eventfdへの割り込みとして通知されるようにできる。

さらに、ここでIRQの情報を提示しているため、ゲストは指定されたIRQに対して割り込みが入った場合にこのデバイスに対するハンドラを起動するようにセットアップを進める。  
ToyVMMの観点からみると、Guest VMに対して見せたVirtioデバイスに対して割り込みをかけたい時（Guest VMに処理を移譲したい場合）には、このIRQに対して割り込みをかけることができれば良いということになる。
つまり、Eventfdを作成して上記のIRQ番号とともに`KVM_IRQFD`で登録し、以降はToyVMM側でこのEventfdに対してwriteすることで割り込みがかけられる。

TODO: Guest/Host通知は図が欲しい

#### MMIO Transport - MMIO Device Register Layoutに対応する実装

MMIO Transportの実装は`mmio.rs`に存在している。

TODO: ここは変わる(Transportはabstractにしたいので)

`MmioTransport`は`BusDevice`を実装しており、デバイスの初期化などによって発生する`VcpuExit::MmioRead`、`VcpuExit::MmioWrite`へのI/Oを従来の`VcpuExit::IoIn`、`VcpuExit::IoOut`と同じように処理できるようにしている。
この`BusDevice` traitを満たすために`read`、`write`を実装するが、ここで[MMIO Device Register Layout](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-1670002)の仕様に従って処理するような実装がなされている。
詳細の説明はここでは行わないが、仕様を見ながらソースコードを確認すれば容易に理解できるだろう。

この部分の処理は主にGuest VMの起動時に生じるvirtio deviceに対するdevice driverの初期化シーケンスで実施される初期化や設定などで呼び出されることになる。  
以降では、実際にGuest VMが起動するタイミングで実施されるやり取りをデバッグコードを仕込むことで観察してみることで理解の助けとする。

#### Guest VMの起動シーケンスでの初期化、設定処理

Guest OSにはVirtioのデバイスドライバ（ゲスト側のドライバ）が存在しており、これが仕様通りにVirtioデバイスの初期化をすることを期待する。
これは起動時のカーネルコマンドラインで指定したvirtioデバイスのMMIOレンジの情報をもとに、Guest VMがR/Wをかけるはずであり、Hypervisor側としてはVMExitが発生して適切にトラップする必要があるコード部に該当するため、デバッグコードを仕込むことは容易である。
実際にデバッグコードを仕込んで、ゲストOSの起動シーケンス中にMMIOの領域に対して発生したR/Wを観測してみる。

具体的な処理の流れをみる前に、仕様上での初期化処理について整理しておく。
デバイスは`virtio-net`に該当するネットワークデバイスの初期化をしようとしている。
デバイスの初期化の仕様は、MMIO Transportにおけるデバイスの初期化[MMIO-specific Initialization And Device Operation]()、汎用的なデバイスの初期化[General Initialization And Device Operation]()、デバイス特有の初期化[]()に分かれる。
これらを合わせると仕様上は概ね以下のような流れになる。

1. Magic Numberの読み出し。Device ID、Device Type、Vendor ID等の読み出し
2. Deviceのリセット
3. ACKNOWLEDGEステータスビットを設定
4. Device feature bitsを読み出し、OSとドライバが解釈できるfeature bitsをデバイスに設定。
5. FEATURES_OKステータスビットを設定
6. デバイス固有の設定を実施する（Virtqueueの検出や設定の書き込みなど）
7. DRIVER_OKステータスビットを設定し、この時点でデバイスはLiveの状態になる。

さて、上記を念頭においてでは実際のGuest VM起動時の処理について確認してみよう。
以下では、デバッグコードを仕込んだ上でゲストOSを起動した際に、デバッグコードにより出力された情報に対して、私が説明のためにコメントを付け加えたものである。

```bash
# offset 0x00 からmagic numberを読み出し
# Little Endianなので元の値は 116,114,105,118
# 116(10) = 74(16)
# 114(10) = 72(16)
# 105(10) = 69(16)
# 118(10) = 76(16)
# 従って、0x74726976 (magic number)
MmioRead: addr = 0xd0000000, data = [118, 105, 114, 116]
# offset 0x04 から device id(0x02)を読み出し
MmioRead: addr = 0xd0000004, data = [2, 0, 0, 0]
# offset 0x08 から device typeを(net = 0x01)読み出し
MmioRead: addr = 0xd0000008, data = [1, 0, 0, 0]
# offset 0x0c から vendor id (virtio vendor id = 0x00) を読み出し
MmioRead: addr = 0xd000000c, data = [0, 0, 0, 0]

# この辺りはDevice Initializationのフェーズ(3.1.1 Driver Requirements: Device Initialization)
# offset 0x70(= Status) に0を書き込み -> デバイスステータスをリセット
MmioWrite: addr = 0xd0000070, data = [0, 0, 0, 0]
# offset 0x70 から読み込み。今デバイスはリセットされている
MmioRead: addr = 0xd0000070, data = [0, 0, 0, 0]
# offset 0x70 に0x01 = ACKNOWLEDGE をセット。
MmioWrite: addr = 0xd0000070, data = [1, 0, 0, 0]
# offset 0x70 から読み込み。確認か?
MmioRead: addr = 0xd0000070, data = [1, 0, 0, 0]
# offset 0x70(= Status)に0x02 = Device(2)を'加算した値'(1 + 2 = 3)を書き込み
MmioWrite: addr = 0xd0000070, data = [3, 0, 0, 0]

# Device/DriverのFeature bitsに関する処理。
# Deviceは自身が持つ機能セット(feature bits)を提供し、デバイスの初期化時にドライバはこれを読んだ上で、受け入れる機能サブセットをデバイスに指示する。
# まずは、Guest OS上のvirtio device driverがfeature bitsを読みだす処理
# offset 0x14(=DeviceFeatureSel) に0x01をSet(次の処理に関わる)
MmioWrite: addr = 0xd0000014, data = [1, 0, 0, 0]
# offset 0x10(=DeviceFeatures)にたいしてRead.
# 読みだすのはDeviceFeatures bitだが、(DeviceFeatureSel * 32) + 31 bitsを返却する
# 今DeviceFeatureSel=1なので、DeviceFeatures bitsのうち、64~32 bitsを返却する。
# virtio-netだと、DeviceFeatureSel=0x0000_0001_0000_4c83 (64bit)なので、
# 先頭0x0000_0001がLittle Endiandで取り出される。
MmioRead: addr = 0xd0000010, data = [1, 0, 0, 0]
# offset 0x14(=DeviceFeatureSel)に0x00をSet(次の処理に関わる)
MmioWrite: addr = 0xd0000014, data = [0, 0, 0, 0]
# offset 0x10(=DeviceFeatures)にたいしてRead.
# 今DeviceFeatureSel=0なので、DeviceFeatures bitsのうち、31~0 bitsを返却する。
# virtio-netだと、DeviceFeatureSel=0x0000_0001_0000_4c83 (64bit)なので、
# 0x0000_4c83がLittle Endiandで取り出される。以下、値の確認
# Little Endianを戻す: 0,0,76,131
# 76(10) = 4c
# 131(10) = 83
# 0x00004c83 -> avail_features(0x100004c83)のうち、VIRTIO_F_VERSION_1で立てているbit(0x100000000)を無視したbit
# つまり、
# * virtio_net_sys::VIRTIO_NET_F_GUEST_CSUM
# * virtio_net_sys::VIRTIO_NET_F_CSUM
# * virtio_net_sys::VIRTIO_NET_F_GUEST_TSO4
# * virtio_net_sys::VIRTIO_NET_F_GUEST_UFO
# * virtio_net_sys::VIRTIO_NET_F_HOST_TSO4
# * virtio_net_sys::VIRTIO_NET_F_HOST_UFO
# のfeature bitの情報を返却している
MmioRead: addr = 0xd0000010, data = [131, 76, 0, 0]
# feature bitsの読み出しはここまでで、ここからは受け入れる機能サブセットをデバイスに指示する処理
# 処理としては読み出し時と似たような感じで、DriverFeatureSel bitに0x00/0x01を書き込んだ上で、DriverFeatureに設定したいfeature bitsを書き込んでいく
# まず、offset 0x24(DriverFeatureSel/activate guest feature) に対して、0x01をSet(= acked_featuresに0x01を設定)
MmioWrite: addr = 0xd0000024, data = [1, 0, 0, 0]
# offset 0x20(DriverFeatures)に対して0x01(= 0x0000_0001, 先に読み出した値の片方)をSet（DriverFeatureSelに0x01が設定されているので、
# 32bit shiftが起きて、0x0000_0001_0000_0000 が実際は設定される）
MmioWrite: addr = 0xd0000020, data = [1, 0, 0, 0]
# offset 0x24(DeviceFeatureSel/activate guest feature)に対して、0x00をSet(= acked_featuresに0x00を設定)
MmioWrite: addr = 0xd0000024, data = [0, 0, 0, 0]
# offset 0x20(DeviceFeatures)に対して、0x0000_4c83（先に読み出した値の片方）をSet（DriverFeatureSelに0x00が設定されているので、そのまま加算する形で書き込む）
MmioWrite: addr = 0xd0000020, data = [131, 76, 0, 0]
# ここまでで、Feature bitsの処理が完了する。

# offset 0x70(= Status)を読み込み -> 直近で0x03が指定されているので、0x03が返却されるのはよい。
MmioRead: addr = 0xd0000070, data = [3, 0, 0, 0]
# offset 0x70(= Status)に0x08 = FEATURES_OK(8)を'加算した値'(3 + 8 = 11)を書き込み
MmioWrite: addr = 0xd0000070, data = [11, 0, 0, 0]
# offset 0x70(= Status)から読み込み。当然11が返る
MmioRead: addr = 0xd0000070, data = [11, 0, 0, 0]

# device-specific setupはここからになる(4.2.3.2 Virtqueue Configuration)
# offset 0x30(=QueueSel)に 0x00 をSet(self.queue_select)
MmioWrite: addr = 0xd0000030, data = [0, 0, 0, 0]
# offset 0x44(QueueReady)を読み込んで、まだreadyではないので0x0が返却され、これは期待通り。
MmioRead: addr = 0xd0000044, data = [0, 0, 0, 0]
# offset 0x34(QueueNumMax)を読み込んで、queueのサイズを確認。
MmioRead: addr = 0xd0000034, data = [0, 1, 0, 0]
# offset 0x38(QueueNum)に先ほど読み込んだQueueNumをセット(q.size)
MmioWrite: addr = 0xd0000038, data = [0, 1, 0, 0]

# Virtual queue's 'descriptor' area 64bit long physical address
# offset 0x80(QueueDescLow = lo(q.desc_table) / lower 32bits of the address)に、
# QueueSelレジスタで選択したqueue（0）のdiscriptor領域の位置をSet -> 122,209,64,0 -> 0x7ad14000
MmioWrite: addr = 0xd0000080, data = [0, 64, 209, 122]
# 同上。但し0x84(QueueDescHigh = hi(q.desc_table) / higher 32 bits of the address)の残りの部分をSet.
MmioWrite: addr = 0xd0000084, data = [0, 0, 0, 0]
# 二つ合わせると、0x0000_0000_7ad1_4000 (q.desc_table) が先頭アドレス

# Virtual queue's 'driver' area 64bit log physical address
# offset 0x90(QueueDeviceLow = lo(q.avail_ring) / lower 32bits of the address)に、
# QueueSelレジスタで選択したqueue(0)のdriver領域(avail_ring)の位置をSet -> 122,209,80,0 -> 0x7ad15000
MmioWrite: addr = 0xd0000090, data = [0, 80, 209, 122]
# 同上。但し0x94(QueueDeviceHigh = hi(q.avail_ring) / higher 32 bits of the address)の残りの部分をSet.
MmioWrite: addr = 0xd0000094, data = [0, 0, 0, 0]
# 二つ合わせると、0x0000_0000_7ad1_5000 (q.avail_ring)
# q.desc_tableのアドレスレンジ: q.avail_ring - q.desc_table = 0x1000 = 512(10)

# Virtual queue's 'device' area 64bit long physical address
# offset 0xa0(QueueDeviceLow = lo(q.used_ring) / lower 32bits of the address)に、
# QueueSelレジスタで選択したqueue(0)のdevice領域 (q.used_ring) の位置をSet -> 122,209,96.0 -> 0x7ad16000
MmioWrite: addr = 0xd00000a0, data = [0, 96, 209, 122]
# 同上。但し0xa4（QueueDeviceHigh = hi(q.used_ring) / higher 32bits of the address）の残り部分をSet.
MmioWrite: addr = 0xd00000a4, data = [0, 0, 0, 0]
# 二つ合わせると、0x0000_0000_7ad1_6000 (q.used_ring)
# q.avail_ringのアドレスレンジ: q.used_ring - q.avail_ring = 0x1000 = 512(10)

# offset 0x44 (QueueReady = q.ready) に対して0x1を書き込んでReadyにしている
MmioWrite: addr = 0xd0000044, data = [1, 0, 0, 0]

# 上記と同じ流れをもう片方のqueue(1)でも実施している
MmioWrite: addr = 0xd0000030, data = [1, 0, 0, 0]
MmioRead: addr = 0xd0000044, data = [0, 0, 0, 0]
MmioRead: addr = 0xd0000034, data = [0, 1, 0, 0]
MmioWrite: addr = 0xd0000038, data = [0, 1, 0, 0]
MmioWrite: addr = 0xd0000080, data = [0, 128, 196, 122]
MmioWrite: addr = 0xd0000084, data = [0, 0, 0, 0] # q.desc_table = 0x0000_0000_7ad1_8000
MmioWrite: addr = 0xd0000090, data = [0, 144, 196, 122]
MmioWrite: addr = 0xd0000094, data = [0, 0, 0, 0] # q.avail_ring = 0x0000_0000_7ad1_9000
MmioWrite: addr = 0xd00000a0, data = [0, 160, 196, 122]
MmioWrite: addr = 0xd00000a4, data = [0, 0, 0, 0] # q.used_ring = 0x0000_0000_7ad1_a000
MmioWrite: addr = 0xd0000044, data = [1, 0, 0, 0]
# ここまでで、device-specificなセットアップ（virtio-net向けの2つのqueueのセットアップ）が完了

# offset 0x70(= Status)を読み込み -> 0x11が書いてあるのでそれが返却される
MmioRead: addr = 0xd0000070, data = [11, 0, 0, 0]
# offset 0x70(= Status)に0x04 = DRIVER_OK(4)を'加算した値'(11 + 4 = 15)を書き込み
MmioWrite: addr = 0xd0000070, data = [15, 0, 0, 0]
# offset 0x70(= Status) を読み込んで値確認かな?
MmioRead: addr = 0xd0000070, data = [15, 0, 0, 0]
# ここまでで、Device Initializationのフェーズ(3.1.1 Driver Requirements: Device Initialization)は完了する
```

冷静に一つ一つ解釈していくと、仕様通りの挙動になっていることがわかる。

### PCI Transportの実装

TODO

### 実装の全体Overview

改めて、全体のOverviewについて示しておく。


### Reference

* [OASIS](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=virtio)
* [ハイパーバイザの作り方](https://syuu1228.github.io/howto_implement_hypervisor/)
* [The Definitive KVM (Kernel-based Virtual Machine) API Documentation](https://docs.kernel.org/virt/kvm/api.html)
* [QEMU](https://github.com/qemu/qemu/)
