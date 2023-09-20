# Implement virtio-blk

`virtio-blk`はVirtioベースのblock deviceの実装である。
仕様はVirtioと同様OASISの公式で公開されている、[Block Device](https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.html#x1-2170002)に記載があるが、本実装はこの仕様に完全に一致している訳ではないので注意されたい。
本セクションはこれまでに出てきた概念は特に説明なしに利用するため、[Virtio](./03-1_virtio.md)、[Implement virtio in ToyVMM](./03-2_implement_virtio_in_toyvmm.md)を読んでない場合は、本セクションの内容を読み進める前に必ず確認されたい。
また、以降の`virtio-block`の説明は`MMIO Transport`をベースとして説明することとする。

### virtio-blkの仕組み

`virtio-blk`ではN個のVirtQueueを利用してDisk I/Oを実現する
TODO: なぜ一つのVirtQueueで完結するのか

virtio-blkの大まかのな仕組みを以下の図に示す。

TODO: 図を書く
TODO: 図の説明を簡単に書く

### virtio-blkの実装詳細に入る前に

`virtio-net`の実装でもあったが、まずはGuest VM起動の流れでToyVMMとして以下のような初期化処理を実施する

TODO: 実装を追いながら確認

1. `EpollContext`の初期化処理でepoll fdの作成、必要なeventfdの登録、その他epollの監視／対処ループで利用するデータの初期化。
2. `DeviceManager`の初期化処理でMMIOに利用するGuestメモリアドレスやレンジ、IRQを決定する。
3. `virtio-blk`向けにepoll監視で処理する際に利用するtokenを作成やchannelの初期化を実施し、`virtio-blk`初期化に利用する`EpollConfig`を初期化。
4. 3で作成した`EpollConfig`を伴って`virtio-blk`を初期化する。
5. `DeviceManager`に対して作成した`virtio-blk`を`mmio`として登録する。この際に以下の内容が実施される。その後busにこのデバイスをMMIO開始アドレスとlengthを伴って登録する。
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

### virtio-blkの実装詳細

`virtio-blk`の実装は`block.rs`に存在している
各種構造体の役割と関係は以下の図のようになっている。  

TODO: 図を書く
TODO: 役割をざっくりと記載する


`Block`構造体は`VirtioDevice`traitを実装しており、`MmioTransport`に紐づけられるような仕組みになっている。
そのため、デバイスの初期化や設定時の振る舞いはこのこの実装に依存する形になる。  

TODO: 以降の記載はvirtio-netからコピペしたやつなので改めて実装を確認すること

重要なのは、`VirtioDevice`traitで実装しなければならない、`activate`関数である。  
この関数はデバイスの初期化処理の流れの中で、初期化が成功し`DEVICE_OK`が書き込まれたタイミングで実行されるように実装している関数である。
この関数の中で、VirtQueueに対応づけているEventfdをepollに登録したり、Epollに割り込みが入った際に起動するハンドラ(`NetEpollHandler`)の初期化を行ったりしている。
この処理は、`VcpuExit`に伴って呼び出される処理、つまりvCPUを処理するながれで呼び出される処理であり、メインスレッドとは異なる。
一方でEpollを監視し適切にハンドルするのはメインスレッドであるため、初期化した`NetEpollHandler`をメインスレッドに送るためにchannelの機構を利用している。  
こうすることで、Guest VMの起動を別スレッドで進めつつメインスレッド他の仕事を行い、Guest VMの起動の流れで初期化と設定が完了した`virtio-net`に関わるハンドラをchannel経由でメインスレッドから参照できるようになるため、メインスレッドのEpoll監視でこのVirtQueueに関して割り込み（`QueueNotify`）が発生した際にメインスレッドで拾ってハンドラに渡すことができ、vCPU（= Guest VMの処理）とは非同期にデバイスのI/Oが処理できるようになる。

上述したとおり、Host-Guest間での通信は基本的にVirtQueueに紐づいたEventfdやTapデバイスのfdの発火を基点として`NetEpollHandler`が担当することになる。
以降では、TX/RXそれぞれのケースでどのように処理が実行されていくかについてより詳細に説明をしていく。

#### Read

#### Write


### virtio-blkの動作確認

実装中はこれまで通りinitrd.imgを読み込み、RAM上にrootfsを展開するようにしておき、ブロックデバイス（/dev/vda)として適当に作成したファイルを渡すなどしてデバッグするのが良い。

ここでは実際にvirtio-blkの実装が完了した上での動作確認として、もはやinitrd.imgを使わないようにし、Firecracker同様Ubuntuのrootfsイメージを利用して、UbuntuのOSを起動するようにしてみよう。  
virtio-blkが実装できたことにより、UbuntuのrootfsイメージをVMの`/dev/vda`として渡すことができ、VMのカーネルのcmdlineの値に`root=/dev/vda`を指定する。  
これによって、kernelは（ここは要確認）ブロックデバイスからOSを起動するようになる。

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

virtio-net/virtio-blkの実装を終え、かなりシンプルではあるが必要な機能を最小限有したVMを作成するに至った。

### Reference

* [OASIS](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=virtio)
* [ハイパーバイザの作り方](https://syuu1228.github.io/howto_implement_hypervisor/)
* [自作VMMのvirtio-blk対応](https://blog.bobuhiro11.net/2022/04-12-gokvm6.html)

