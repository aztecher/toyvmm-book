# Implement virtio-net

`virtio-net`はVirtioベースのnetwork cardの実装である。
仕様はVirtioと同様OASISの公式で公開されている、[Network Device](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-2170001)に記載があるが、本実装はこの仕様に完全に一致している訳ではないので注意されたい。
本セクションはこれまでに出てきた概念は特に説明なしに利用するため、[Virtio](./03-1_virtio.md)、[Implement virtio in ToyVMM](./03-2_implement_virtio_in_toyvmm.md)を呼んでない場合は、本セクションの内容を読み進める前に必ず確認されたい。
また、以降の`virtio-net`の説明は`MMIO Transport`をベースとして説明することとする。

### virtio-netの仕組み

`virtio-net`では2N個のVirtQueueを利用してデータの転送処理を行う。
それぞれのVirtQueueは片側方向の通信にのみ利用することになるため、TX/RXを実装するために少なくとも2つのVirtQueueを利用することになり、それを必要に応じて横スケールすると2N個ということになる。
仕様としては、さらにcontrolqというデバイス管理、制御用のVirtqueueを利用することが記載されているが、少なくともfirecrackerではこのデバイスの管理、制御にはioeventfd／irqfdによる軽量なHost-Guest間通信が利用されているようでありToyVMMもこれに習っている。

virtio-netの大まかな仕組みを以下の図に示す。  


TODO: 図を書く
TODO: 図の説明を簡単に書く

以降ではより詳細な仕組みについて実装ベースで説明していく。

### virtio-netの実装詳細に入る前に

まず、GuestVM起動の流れでToyVMMとして以下のような処理を実施する。

1. `EpollContext`の初期化処理でepoll fdの作成、必要なeventfdの登録、その他epollの監視／対処ループで利用するデータの初期化。
2. `DeviceManager`の初期化処理でMMIOに利用するGuestメモリアドレスやレンジ、IRQを決定する。
3. `virtio-net`向けにepoll監視で処理する際に利用するtokenを作成やchannelの初期化を実施し、`virtio-net`初期化に利用する`EpollConfig`を初期化。
4. 3で作成した`EpollConfig`を伴って`virtio-net`を初期化する。この際にHost側にTapデバイスを作成する。
5. `DeviceManager`に対して作成した`virtio-net`を`mmio`として登録する。この際に以下の内容が実施される。その後busにこのデバイスをMMIO開始アドレスとlengthを伴って登録する。
  * MmioTransportの初期化。ここでVirtQueueに利用するEventfdの作成や、VirtQueue自体の初期化を実施する。  
  * VmRequestsを作成する。このVmRequestsは次の処理で利用される、`ioeventfd`や`irqfd`などのKVM API呼び出しリクエストの一覧である。
  * VirtQueue用のEventfdが2個(RX/TX)あり、これはGuest上での`QueueNotify` MMIOを拾う目的で`ioeventfd`呼び出しリクエストとして登録。
  * Guestのvirtio-net deviceのIRQに割り込みをかけるためのEventfdが1個あり、これは`irqfd`呼び出しリクエストとして登録。
6. `ioeventfd`, `irqfd`を呼び出して、Guest-Host通信のための紐付け処理を行う。
  * 5で作成したVmRequestsをループしてそれぞれAPIコールをかけて登録する。
7. busをcloneして、Vcpu処理スレッドでの`VcpuExit::MmioRead`、`VcpuExit::MmioWrite`のハンドリングに利用する。
8. メインスレッドではepollを監視する

上記の流れを図に起こすと以下のようになる。  

TODO: 図


### virtio-netの実装詳細

`virtio-net`の実装は`net.rs`に存在している。
各種構造体の役割と関係は以下の図のようになっている。

TODO: 図を書く
TODO: 役割をざっくりと記載する。

`Net`構造体は`VirtioDevice`traitを実装しており、`MmioTransport`に紐づけられるような仕組みになっている。
そのため、デバイスの初期化や設定時の振る舞いはこのこの実装に依存する形になる。  

重要なのは、`VirtioDevice`traitで実装しなければならない、`activate`関数である。  
この関数はデバイスの初期化処理の流れの中で、初期化が成功し`DEVICE_OK`が書き込まれたタイミングで実行されるように実装している関数である。
この関数の中で、VirtQueueに対応づけているEventfdをepollに登録したり、Epollに割り込みが入った際に起動するハンドラ(`NetEpollHandler`)の初期化を行ったりしている。
この処理は、`VcpuExit`に伴って呼び出される処理、つまりvCPUを処理するながれで呼び出される処理であり、メインスレッドとは異なる。
一方でEpollを監視し適切にハンドルするのはメインスレッドであるため、初期化した`NetEpollHandler`をメインスレッドに送るためにchannelの機構を利用している。  
こうすることで、Guest VMの起動を別スレッドで進めつつメインスレッドは他の仕事を行い、Guest VMの起動の流れで初期化と設定が完了した`virtio-net`に関わるハンドラをchannel経由でメインスレッドから参照できるようになるため、メインスレッドのEpoll監視でこのVirtQueueに関して割り込み（`QueueNotify`）が発生した際にメインスレッドで拾ってハンドラに渡すことができ、vCPU（= Guest VMの処理）とは非同期にデバイスのI/Oが処理できるようになる。

上述したとおり、Host-Guest間での通信は基本的にVirtQueueに紐づいたEventfdやTapデバイスのfdの発火を基点として`NetEpollHandler`が担当することになる。
以降では、TX/RXそれぞれのケースでどのように処理が実行されていくかについてより詳細に説明をしていく。

#### TX (Guest -> Host)

まずはGuest -> Host方向の通信(TX)について実装を見ながら詳細を説明する。
ここでもVirtQueueの動きも重要になるが、VirtQueueの説明は[Implement virtio in ToyVMM](./03-2_implement_virtio_in_toyvmm.md)でおこなっているのでここでは省略とする。
また、同じく[Implement virtio in ToyVMM](./03-2_implement_virtio_in_toyvmm.md)で確認したGuest VMの起動シーケンスで初期化された前提で話を進める。従って、Virtqueueは以下のような値で初期化されているとする。

```
TX Virtqueueの初期値
- max_size   = 256
- size       = 16
- ready      = true
- desc_table = 0x7ad1_8000
- avail_ring = 0x7ad1_9000
- used_ring  = 0x7ad1_a000
- next_avail = 0
- next_used  = 0
````

TXの場合は`Descriptor Table`、`Available Ring`、`Used Ring`はそれぞれ以下のように機能する

* `Descriptor Table` : Guestが送信しようとしているデータを指すようなDescriptorが格納されている
* `Available Ring` : 送信データを含むDescriptorのindexを格納しており、これをHostが読み取って必要なDescriptorを辿って処理する
* `Used Ring` : Host側で処理が完了したDescriptorのindexを格納し、Guestはこれを読み取って処理済みDescriptorを回収する

さてTXのケースの具体的な処理について実装をもとに説明してみる。
TXは、Guest（Guestのデバイスドライバ）がパケットを準備し、`QueueNotify`に対してWriteが走ることでToyVMMに制御が映る。

具体的に、Guestではまず以下のような処理が行われることが期待される。

1. Devce driverは初期化の段階で、全てのDescriptorの初期化を行い、nextの値には次のDescriptorのエントリ番号を入れてチェーンの作成を行う。
2. チェーンの先頭Descriptorの番号を記録
3. 先頭Descriptorのaddrにデータのアドレス、lenにデータ長を代入
4. Available Ringのidxが指すAvailable Ringの空きエントリに、3のDescriptor indexを代入
5. Available Ringのidxの値をインクリメントして次の秋エントリを指す
(6. virtio header(=mmioの)の...はない気がする)
6. 未処理データがあることをホストに通知するために、MMIOの`QueueNotify`へ書き込み
7. `QueueNotify`のアドレスへの書き込みはioeventfdによってeventfdへの割り込みとして変換されるため、TX VirtQueueに割り込みがかかる。

TODO: ここまでの処理を遷移図で記載する.

ここからは処理がホスト側（ToyVMM）に移ってくる
割り込みによってToyVMMに処理が移動してくるとメインスレッドのepoll監視で拾い上げて、`NetEpollHandler`の`handle_event`関数を呼び出す。
この場合は`TX_QUEUE_EVENT`に該当するTokenがepollから返却されるため、`process_tx`関数が呼ばれることになる。

`process_tx`では以下のような形で処理が進んでいく。コードと比較しながら確認してほしい。

1. 必要な変数等の初期化
  * `frame[0u8; 65562]` : Guest側で用意されたデータをコピーするバッファ。
  * `used_desc_heads[0u16; 256]` : 処理済みの`descriptor`のindexを格納。最後に`used_ring`へ値をセットする
  * `used_count` : Guest側のデータをどこまで読み出したか保存しておくカウンタ
2. TX用のVirtQueueをiterationして、3~5までの処理を繰り返す。
  * Iterationの詳細は下図に記載
  * 1回のiterationで、`Available Ring`の要素をひとつ確認し、その要素が指すDescriptorのデータを取り出して、必要な情報とともに`DescriptorChain`構造体にまとめて返却する。
  * 従って、コード上のIterationの結果として取り出される要素`avail_desc`には、「`Available Ring`の要素が指す`Descriptor`が指すデータ情報やDescriptor自身のindex情報などを含んだ`DescriptorChain`」が返却される
  * 次のiterationでは次の要素を取り出し、このiterationは`Available ring`のidxの値と、現在読み出したindexが一致すると停止するため、TXの場合はGuestが詰めた`Available Ring`の要素分だけ取り出すような繰り返しを実施する。
3. ループしながら、`Descriptor`が指すデータ情報を読み出してバッファに積み込み、もし`next`が指す`Descriptor`があればそれをたどってデータを読みだす。 
4. 読み出したデータをtapに書き込む
5. 処理した`Descriptor`（`Available Ring`が指しているDescriptor）のindexを最後に`used_ring`に書き込むために一時的に変数に保存する。
6. 2~5のiterationで格納した処理済み`Descriptor`を、`used_ring`に書き込む
7. Guestにたいして割り込みをかけ、処理をGuestに移譲する。

TODO: Iterationのイメージ図を書く
TODO: ここまでの処理を遷移図で記録する

```memo
last_index = 1 (Guestによって、ここまでavail_ring書き込まれてるよ、の値)
AvailIter {
  mem,
  desc_table,
  avail_ring,
  next_index = 0,
  last_index = 1,
  queue_size = 16,
  next_avail = 0,
}

next()のなかで、
1. avail_ring[0]の中身(desc_index)の取得
2. next_index += 1
3. DesciptorChainを1のdesc_indexを伴って初期化 -> desc_indexの挿している中身が展開される
4. 3に成功したら、next_availの値も引き上げ
4. 3の結果を返す
次回のループでは、next_index = last_index、つまりGuestが用意したデータは使いおわった、になるためNoneを返してループが進まない。
```

以降の処理はGuest側（device driver）に移譲され、以下のように処理されることが期待される。

1. `Used Ring`のidxを確認し、既に処理したindex位置（最初は`0`）との差分がある場合は、その差分を埋めるように`Used Ring`の要素(`Descriptor index`)を確認しする。  
2. `Desciptor index`が指す`Descriptor`はホストで処理されたものになるため、空き`Descriptor`チェーンに戻し記録していたDescriptor番号を更新する。
3. 1~2の処理を`Used Ring`のidxの値と記録しているindex位置の値に差分がなくなるまで繰り返す。

TODO: ここまでの処理を遷移図で記録する

以上でTXの処理が完了する。


#### RX (Host -> Guest)

次に、Host -> Guest方向の通信(TX)について実装を見ながら詳細を説明する。
ここでもVirtQueueの動きが重要になるが、VirtQueueの説明は[Implement virtio in ToyVMM](./03-2_implement_virtio_in_toyvmm.md)でおこなっているのでここでは省略とする。
また、同じく[Implement virtio in ToyVMM](./03-2_implement_virtio_in_toyvmm.md)で確認したGuest VMの起動シーケンスで初期化された前提で話を進める。従って、Virtqueueは以下のような値で初期化されているとする。

```
RX Virtqueueの初期値
- max_size   = 256
- size       = 16
- ready      = true
- desc_table = 0x7ad1_4000
- avail_ring = 0x7ad1_5000
- used_ring  = 0x7ad1_6000
- next_avail = 0
- next_used  = 0
````

RXの場合は`Descriptor Table`、`Available Ring`、`Used Ring`はそれぞれ以下のように機能する

* `Descriptor Table` : 
* `Available Ring` : 
* `Used Ring` : 

さてRXのケースの具体的な処理について実装をもとに説明してみる。
RXの場合、Guest側のドライバは初期化処理の流れの中で、Hostからのデータを格納する場所の情報をあらかじめ用意する。

具体的に、Guestではまず以下のような処理が行われていることが期待される。

1. Devce driverは初期化の段階で、全てのDescriptorの初期化を行い、nextの値には次のDescriptorのエントリ番号を入れてチェーンの作成を行う。
2. チェーンの先頭Descriptorの番号を記録
3. Available Ringのidxが指すAvailable Ringの空きエントリに、Descriptor chainの先頭indexを代入
5. Available Ringのidxの値をインクリメントして次の空きエントリを指す
(6. virtio header(=mmioの)の...はない気がする)
6. ホストに通知するために、MMIOの`QueueNotify`へ書き込み
7. `QueueNotify`のアドレスへの書き込みはioeventfdによってeventfdへの割り込みとして変換されるため、RX VirtQueueに割り込みがかかる。

TODO: ここまでの処理を遷移図で記載する

ここからは処理がホスト側（ToyVMM）に移ってくる
割り込みによってToyVMMに処理が移動してくるとメインスレッドのepoll監視で拾い上げて、`NetEpollHandler`の`handle_event`関数を呼び出す。  
この場合は`RX_QUEUE_EVENT`に該当するTokenがepollから返却されるため、それに対応する処理が実施されるが、初期化段階ではパケットを受け取っていないことが想定されるため、特に何もしない。
実際の受信処理は、Host側のTapデバイスにパケットが着弾した際に実施されることになる。以下、詳細を示す。

1. Tapデバイスでパケットを受信すると、epoll監視によって`process_rx`の処理が実施される。
2. `process_rx`ではTapデバイスに`read`システムコールをかけることで受信したパケットを取得し、`rx_buf`に格納する。この時読み込んだパケットのデータ量も取得しておく。
3. `rx_single_frame`関数を呼び出す。この関数が具体的なHostからGuestへのパケット受け渡し処理に該当する。以下の通りに処理が進む。
  * Guest側の処理によって`Available Ring`に格納されていることが想定されている、空き`Descriptor`をひとつだけ取り出し初期化する。
  * `rx_buf`からのデータ読み出し量`limit`を計算する。
    * 最大で読み出せる量は、`Descriptor`の指すデータ長に依存する。
    * 上記の値と、`rx_buf`に読み出した量（`rx_count`）の比較をして、小さい方を`rx_buf`からの読み出し量として決定する。
  * `rx_buf`から`limit`分を読み出して、空き`Descriptor`の`addr`に対して書き込む。この際に書き込んだ量は保存しておく(`write_count`)
  * `limit`の計算時、`Descriptor`の`len`の方が小さい場合、上記の処理では`rx_buf`にはまだ読みだすべきデータが余ることになる。
    * その場合は今参照している`Descriptor`の`next`を取り出して同様の処理をループして実施していく。
    * 上記によって、`rx_buf`のデータを全て読み出す。そのデータは長さによっては`next`でチェーンされていく、というような形になっている。
  * `used_ring`の値に、上記の処理につかった空き`Descriptor`のindexを格納して、割り込みをかけ、処理をGuestに移譲する。

TODO: ここまでの処理を遷移図で記載する

以降の処理はGuest（device driver）に移譲され、以下のように処理されることが期待される。

1. `Used Ring`のidxを確認し、既に処理したindex位置（最初は`0`）との差分がある場合は、その差分を埋めるように`Used Ring`の要素(`Descriptor index`)を確認する
2. `Descriptor index`が指す`Descriptor`はホストから書き込まれたデータが格納されているのでこれを処理し、処理したら秋`Descriptor Chain`へ戻し、`Available Ring`を更新する。
3. 1~2の処理を`Used Ring`のidxの値と記録しているindex一の値に差分がなくなるまで繰り返す。

TODO: ここまでの処理を遷移図で記録する

以上でRXの処理が完了する。

### virtio-netの動作確認

実際に`virtio-net`でGuest側に作成したネットワークインターフェイスを通して通信してみる。  
以下はGuest内部で`ip addr`コマンドを実行した結果であり、`eth0`がインターフェイスとして認識されている。  
これが`virtio-net`で作成したGuest側のデバイスであり、IPはついていない状態。

```bash
localhost:~# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 02:32:22:01:57:33 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::32:22ff:fe01:5733/64 scope link
       valid_lft forever preferred_lft forever
```

ホスト側も確認してみる。
ToyVMMではTapデバイスをホスト側に作成し、固定のIPアドレス（`192.168.0.10/24`）を付与するようにしている。

```bash
140: vmtap0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 1000
    link/ether c6:69:6d:65:05:cf brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.10/24 brd 192.168.0.255 scope global vmtap0
       valid_lft forever preferred_lft forever
```

Guest側にIPアドレスを付与する。Hostと同じサブネットレンジのアドレスを適当に付与すしている。

```bash
localhost:~# ip addr add 192.168.0.11/24 dev eth0
```

Guestの中からHostのTapインターフェイスのIPに向けてping

```bash
localhost:~# ping -c 5 192.168.0.10
PING 192.168.0.10 (192.168.0.10): 56 data bytes
64 bytes from 192.168.0.10: seq=0 ttl=64 time=0.394 ms
64 bytes from 192.168.0.10: seq=1 ttl=64 time=0.335 ms
64 bytes from 192.168.0.10: seq=2 ttl=64 time=0.334 ms
64 bytes from 192.168.0.10: seq=3 ttl=64 time=0.321 ms
64 bytes from 192.168.0.10: seq=4 ttl=64 time=0.330 ms

--- 192.168.0.10 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 0.321/0.342/0.394 ms
```

逆にHostからGuestの`virtio-net`インターフェイスのIPに向けてping

```bash
[mmichish@mmichish ~]$ ping -c 5 192.168.0.11
PING 192.168.0.11 (192.168.0.11) 56(84) bytes of data.
64 bytes from 192.168.0.11: icmp_seq=1 ttl=64 time=0.410 ms
64 bytes from 192.168.0.11: icmp_seq=2 ttl=64 time=0.366 ms
64 bytes from 192.168.0.11: icmp_seq=3 ttl=64 time=0.385 ms
64 bytes from 192.168.0.11: icmp_seq=4 ttl=64 time=0.356 ms
64 bytes from 192.168.0.11: icmp_seq=5 ttl=64 time=0.376 ms

--- 192.168.0.11 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4114ms
rtt min/avg/max/mdev = 0.356/0.378/0.410/0.028 ms
```

ICMPだけであるが問題なく通信ができることまで確認できた

### Reference

* [OASIS](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=virtio)
* [ハイパーバイザの作り方](https://syuu1228.github.io/howto_implement_hypervisor/)

