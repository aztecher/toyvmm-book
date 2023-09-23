# Implement virtio-blk device

ここではゲストの`virtio-blk`が利用するBlock deviceの実装を行っていくことにする。
仕様はVirtioと同様OASISの公式で公開されている[Block Device](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-2170002)に記載があるが、本実装はこの仕様に完全に一致している訳ではないので注意されたい。

本節これまでに出てきた概念は特に説明なしに利用するため、これまでの節を読んでいない場合は本節の内容を読み進める前に必ず確認されたい。  
更に、本説では`virtio-net`の実装部分や説明と重複する内容は適宜省略しているため、本節を確認する前に必ず前節の[Implement virtio-net device](./03-3_virtio-net.md)を一読してもらいたい。

### virtio-blkの仕組み

`virtio-blk`では単一のVirtqueueを利用してゲストからのDISK Read/Writeを表現する。
`virtio-net`とは異なり外部要因（Tapから受信）が介在せず、あくまでGuestからのI/O要求で駆動するため、Virtqueueの数が最低一つで動作するわけである。
当然こちらもVirtqueueの数は仕様上スケールさせることが認められているが、実装の単純化のために今回は行っていない。

以降では、より詳細な仕組みについて実装ベースで説明していくい。

### virtio-blkの実装詳細

`virtio-blk`の実装は`block.rs`に存在している
各種構造体の役割と関係は以下の図のようになっている。  

<div align="center">
<img src="./03_figs/blk-device_activated.svg", width="100%">
</div>

これまで述べてきたとおり具体的な実装は各デバイスの実装に依存しているが、それは`VirtioDevice` Traitによって抽象化されているため、各種デバイスの細部の仕組み以外はすべて`virtio-net`で示したものと同様に動作する。
そのため、上図もBlock Deviceの内部詳細が少し異なるくらいでそれ以外についてNet Deviceと全く同様になっている。

初期化時の`Device Type`の問い合わせや、`Features`の問い合わせなどは`Block`デバイスの具体的な実装で応答し、`Net`デバイスと同様にGuestアドレス上のQueueのアドレス位置などが設定・提供され、初期化ステップ完了とともに、`activate`関数が実行される。
`Block`デバイスの場合も`activate`関数の中で`Net`デバイスと同様に各種`file descriptor`を`epoll`で登録し、`epoll`に対してトリガがかかった際に実行するハンドラ(`BlockEpollHandler`)のセットアップを実施している。  
`Block`デバイスではI/Oエミュレーションするにあたり、ホスト側のファイル（`BlockDevice`として操作するもの）をOpenし、それに対してゲストから要求のあったRead/Writeを実施していく形になる。  
`epoll`に登録する`file descriptor`は`Virtqueue`用の`eventfd`、及び予期しない状況に陥った場合に停止させるための`eventfd`の合計２つである。
`Net`デバイスのケースと比較して、Tapデバイスがファイルに変わったこと、eventfdの数の変化に伴いハンドラが実行するEVENTの数が変わったこと以外に変化がないのが見て取れるだろう。

`Block`デバイスの場合、単一のVirtqueueに紐づくeventfdの発火のみが動作起点になるため、以降ではこの処理を確認していくこととする。


#### virtio-blkにおけるI/Oリクエスト

実装の説明に入る前に`virtio-blk`におけるI/Oリクエストについて説明する。

すでに述べたとおり、`virtio-blk`はゲストからのI/O要求は単一のVirtqueueを介してやり取りする。
一方で、ゲストから発生するI/O要求はかなり雑に考えても`Read`/`Write`の二種類が想定でき、それぞれのケースで処理するべき内容は大きく異なるはずである。
ホスト側ではこの要求の違いをどのように認識し、実際のI/Oをエミュレートすればいいかという疑問が当然発生することになる。

これを説明するためには、`virtio-blk`によってどのように`Descriptor Table`が利用されるかを理解する必要がある。
まず、ゲスト側のドライバがVirtqueueに詰めるデータは以下の構造のものになる。

> ```
> struct virtio_blk_req { 
>         le32 type; 
>         le32 reserved; 
>         le64 sector; 
>         u8 data[]; 
>         u8 status; 
> };
> ```
> Source: [Block Device: Device Operaiton](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-2850006)

実際はこれが`Descriptor Table`に以下ような3つのエントリとして作成され、それぞれのエントリが`next`によってチェーンを構成する形になっている。

一つ目の`Descriptor`エントリは、`type`、`reserved`、`sector`の3つのデータを格納しているアドレスを指し、二つ目の`Descriptor`エントリは、dataが書かれている領域の先頭アドレスを、三つ目の`Descriptor`エントリは、`status`が書かれている領域の先頭アドレスを指すような構造になる。

<div align="center">
<img src="./03_figs/virtio-blk_virtqueue.svg", width="50%">
</div>

特に`type`にはI/Oの種類（`read`や`write`、それ以外のI/O要求）を示しており、ここの値を確認することでホスト側は振る舞いを変更する必要がある。

`read`の場合、２つ目の`Descriptor`が指すエントリはホスト側が実際のDiskから読み込んだデータを格納すべきアドレス領域として利用できる。
`sector`の値から読み取るべきセクタ位置を特定し、そこから必要な量のデータ（２つ目の`Descriptor`の`desc.len`の値）を読み込む。


`write`の場合、2つ目の`Descriptor`が指すエントリにはDiskに書き込むべきデータが格納されているため、データを読み出した上で`sector`の値で指定されているDiskのセクタ位置に書き込みを行う。

３つ目の`Descriptor`エントリには正常にI/Oのエミュレーションが完了したか、もしくは失敗したかなどを表現するステータス情報を書き込む。

上記のように、Disk I/Oの種別と、そのI/Oに対して必要なデータやバッファがVirtqueueを介して提供されるため、ホスト側はこれを仕様に従って解釈し、適切にI/Oをエミュレートする責務を追うことになる。


#### ToyVMMによるDisk I/Oの実装

ゲストから発生したDisk I/Oリクエストについて実装を見ながら詳細を説明していく。
ホスト側の処理以外は基本`Net` Deviceの`Tx`のケースと挙動が同じになるので`QueueNotify`でホスト側に処理が移譲されたところから説明していく。

MMIOの`QueueNotify`への書き込みはによって発火したEventFdは`epoll`の監視によって拾い上げられ`BlockEpollHandle`のハンドラ処理、具体的には`QUEUE_AVAIL_EVENT`に対応する処理が実行される。
実際は`process_queue`関数が呼び出され、この返り値が`true`である場合には`signal_used_queue`関数が呼ばれる。
後者の`signal_used_queue`はゲストに対して割り込みを入れているだけなので詳細に確認すべき処理は前者の`process_queue`関数である。

`process_queue`では以下のような形で処理が進んでいく。ぜひコードを確認しながら以降の解説を見てほしい。

1. 必要な変数の初期化
  * `used_desc_heads[(u16, u32), 256]` : 処理済みの`Descriptor`のindexとデータ長を格納するデータ。`process_queue`の処理の最後にこの値を元に`used_ring`へ値をセットする。
  * `used_count` : GuestからのI/O要求をどこまで処理したかを格納しておくカウンタ
2. Virtqueueをiterationして、停止するまでX~Yの処理を繰り返す
3. `Available Ring`が指す`Descriptor`を取り出し、`virtio-blk`の仕様に従ってパースし`Request`構造体を作成する。
  * `Request`構造体にはパースした結果（リクエスト種別、セクタ情報、データアドレス、データ長、ステータスアドレス）が格納されている
4. `execute`関数を呼び出し、`Request`構造体の内容から実施すべきI/Oリクエストを実施する。
  * I/Oに成功した際、Readの場合は読み込んだデータ長を返却し、writeなどそれ以外は0を返却する。この値は`used_ring`に書き込む値として利用する。
5. I/Oに成功したか失敗したかをstatusアドレスに書き込み、`used_ring`に必要な情報を書き込む
6. 上記の処理で一つ以上のリクエストを処理した場合は関数の戻り値として`true`を返却する。

以下の図はGuestからのI/O要求がReadだった場合の処理の図解である。

<div align="center">
<img src="./03_figs/blk-device_read.svg", width="100%">
</div>

また、以下の図がGuestからのI/O要求がWriteだった場合の処理の図解である。

<div align="center">
<img src="./03_figs/blk-device_write.svg", width="100%">
</div>


### virtio-blkの動作確認

ここでは実際の動作確認として、もはやinitrd.imgを使わず、Firecracker同様Ubuntuのrootfsイメージを利用して、UbuntuのOSを起動するようにしてみよう。  
`virtio-blk`向けBlockDeviceの実装できたことにより、UbuntuのrootfsイメージをVMの`/dev/vda`として認識させることができるようになったため、VMのカーネルのcmdlineの値に`root=/dev/vda`を指定すればこのUbuntuイメージからOSを起動することができるはずである。

```
# Run ToyVMM with kernel and rootfs (no initrd.img)
$ sudo -E cargo run -- boot_kernel -k vmlinux.bin -r ubuntu-18.04.ext4                                                       [22:46:37]
...
warning: `toyvmm` (bin "toyvmm") generated 4 warnings
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/toyvmm boot_kernel -k vmlinux.bin -r ubuntu-18.04.ext4`
[    0.000000] Linux version 4.14.174 (@57edebb99db7) (gcc version 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)) #2 SMP Wed Jul 14 11:47:24 UTC 2021
[    0.000000] Command line: console=ttyS0 reboot=k panic=1  root=/dev/vda virtio_mmio.device=4K@0xd0000000:5 virtio_mmio.device=4K@0xd0001000:6
...

# Instead of Alpine rootfs (initrd.img), Ubuntu rootfs is used and startup.
Welcome to Ubuntu 18.04.1 LTS!

...

Ubuntu 18.04.1 LTS 7e47bb8f2f0a ttyS0

# Please type root/root and login!

7e47bb8f2f0a login: root
Password:
Last login: Mon Aug 14 13:28:29 UTC 2023 on ttyS0
Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.14.174 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

# You can verify that the launched VM is ubuntu-based.
root@7e47bb8f2f0a:~# uname -r
4.14.174
root@7e47bb8f2f0a:~# cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.1 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.1 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic

# And you can also find that this VM mount /dev/vda as rootfs.

root@7e47bb8f2f0a:~# lsblk
NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda  254:0    0  384M  0 disk /


root@7e47bb8f2f0a:~# ls -lat /
total 36
drwxr-xr-x 12 root root   360 Aug 14 13:47 run
drwxr-xr-x 11 root root  2460 Aug 14 13:46 dev
dr-xr-xr-x 12 root root     0 Aug 14 13:46 sys
drwxrwxrwt  7 root root  1024 Aug 14 13:46 tmp
dr-xr-xr-x 57 root root     0 Aug 14 13:46 proc
drwxr-xr-x  2 root root  3072 Jul 20  2021 sbin
drwxr-xr-x  2 root root  1024 Dec 16  2020 home
drwxr-xr-x 48 root root  4096 Dec 16  2020 etc
drwxr-xr-x  2 root root  1024 Dec 16  2020 lib64
drwxr-xr-x  2 root root  5120 May 28  2020 bin
drwxr-xr-x 20 root root  1024 May 13  2020 .
drwxr-xr-x 20 root root  1024 May 13  2020 ..
drwxr-xr-x  2 root root  1024 May 13  2020 mnt
drwx------  4 root root  1024 Apr  7  2020 root
drwxr-xr-x  2 root root  1024 Apr  3  2019 srv
drwxr-xr-x  6 root root  1024 Apr  3  2019 var
drwxr-xr-x 10 root root  1024 Apr  3  2019 usr
drwxr-xr-x  9 root root  1024 Apr  3  2019 lib
drwx------  2 root root 12288 Apr  3  2019 lost+found
drwxr-xr-x  2 root root  1024 Aug 21  2018 opt
```

上記の通り、VMは/dev/vdaとして渡したUbuntu OSを起動し、ログイン後にUbuntuベースのOSになっていることや意図通りrootfsをマウントしていることがわかる。  
さらに、これまでのinitrd.imgではRAM上にrootfsが展開されていたので揮発性があったが、今回はDISKとして永続化されているrootfsをベースに起動しているため、一度VM内部で作成したファイルは再度VMを起動した際も確認することができる。

```
# Create sample file (hello.txt) in first VM boot and reboot.

root@7e47bb8f2f0a:~# echo "HELLO UBUNTU" > ./hello.txt
root@7e47bb8f2f0a:~# cat hello.txt
HELLO UBUNTU
root@7e47bb8f2f0a:~# reboot -f
Rebooting.

# After second boot, you can also find 'hello.txt'.  

Ubuntu 18.04.1 LTS 7e47bb8f2f0a ttyS0

7e47bb8f2f0a login: root
Password:
Last login: Mon Aug 14 13:57:27 UTC 2023 on ttyS0
Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.14.174 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
root@7e47bb8f2f0a:~# cat hello.txt
HELLO UBUNTU
```

`virtio-net`/`virtio-blk`両方のDeviceの実装を終え、かなりシンプルではあるが必要な機能を最小限有したVMを作成するに至ったと言えよう

### Reference

* [OASIS](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=virtio)
* [ハイパーバイザの作り方](https://syuu1228.github.io/howto_implement_hypervisor/)
* [自作VMMのvirtio-blk対応](https://blog.bobuhiro11.net/2022/04-12-gokvm6.html)
* [OSDev.org - Virtio#Block_Device_Packets](https://wiki.osdev.org/Virtio#Block_Device_Packets)
