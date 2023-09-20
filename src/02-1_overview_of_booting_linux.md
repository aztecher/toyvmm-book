# Overview of Booting Linux

### 一般的なブートの仕組み

Linuxでは大まかに以下のようにプログラムが順番に動作していくことでOSが起動していく

1. BIOS
2. ブートローダ (GRUB)
3. Linuxカーネル（vmlinuz）
4. init

BIOSはマザーボード上のROMにプログラムが格納されている。  
我々が電源を投入すると、CPUはこの領域がマップされているアドレスから処理を実行するようになっている。  
BIOSはハードウェアの検出、初期化を実行し、その後OSのブートドライブ（HHD/SSD、USBフラッシュメモリなど）を探索する。  
この時、ブートドライブはMBR、もしくはGPTの形式でフォーマットされている必要があり、これらのフォーマットとBIOSの関係はそれぞれ以下のように対応する

| BIOS \ DISK Format | MBR | GPT |
|--------------------|-----|-----|
| Legacy BIOS        | ◯   | -   |
| UEFI               | ◯ * | ◯   |

\* UEFIはLegacy Boot Modeのサポートがあるため、MBRをサポートしている

以降ではMBRを利用する場合のOS探索について説明する  
詳しい説明に入る前に、MBRの構造について簡単に整理しておく。
以降で説明するMBRの構造はHDD／SSDやUSBフラッシュメモリなどの場合を想定し記載しており、後述するPartition Entryの存在を暗黙に仮定しているので注意されたい。
なお、本資料では[Wikipedia](https://en.wikipedia.org/wiki/Master_boot_record)で記載されている名称を引用しているので注意されたい。

MBRはブートドライブの先頭セクタに512byte書き込まれており、大きく分けて3つの領域が存在している。

1. Bootstrap code are (446 byte)
2. Partition Entry (64 byte = 16 byte * 4)
3. Boot Signature (2 byte)

MBRについてここでは詳細に説明はしないが、Boot code areaにはOSをブートする機械語のプログラム（Boot Loader）が、Partition Entryにはそのディスクの論理パーティション情報が格納されている。  
（余談だが、Boot code areaは446byteしかないため、Boot Loaderを直接実装するのではなく、Boot Loaderは別の場所に格納しておき、そのブートローダをメモリに読み込むために最小限のプログラムを配置することもあるようだ）  
ここで重要なのは3つ目の「Boot Signature」であり、ここに格納されている2byteの値は、当該ドライブがブートドライブかどうかを担保するために利用される。  
具体的には、BIOSがOSのブートドライブを探索する時、先頭1セクタ（512byte）を読み込み、最後の2byte（Boot Signature）がブートドライブであることを示すシグネチャ（`0x55 0xaa`）であることを確認する。
このシグネチャが亜確認できた場合、当該ディスクをブートディスクと判定し、先頭1セクタ(512byte)をメインメモリの0x7c00から0x7fffに読み込んで、0x7c00からプログラムを実行していく。

さて、これまでの議論の簡単な裏付けとして、手元のマシンでBoot Signatureを確認してみる。  
仮想マシンなので、ブートディスクは`vda`と表示されている。通常のマシンなら`sda`などだろう。
この`vda`から先頭1セクタ分の内容をファイルに書き出し、`hexdump`で510byteオフセットした位置から2byte確認してみると、確かに`0x55` `0xaa`の値が確認できる。


```bash
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0     11:0    1    2M  0 rom
vda    252:0    0  300G  0 disk
├─vda1 252:1    0    1M  0 part
└─vda2 252:2    0  300G  0 part /

$ sudo dd if=/dev/vda of=mbr bs=512 count=1
1+0 records in
1+0 records out
512 bytes (512 B) copied, 0.000214802 s, 2.4 MB/s

$ hexdump -s 510 -C mbr
000001fe  55 aa                                             |U.|
00000200
```

話を戻すと、BIOSによってMBRの情報を元にブートディスク上に格納されていたブートローダがメモリ上に展開され実行されていくことになる。ブートローダはカーネルとinitramfsをDISKからメモリに読み込み、カーネル起動する役目を持ったプログラムであり、近年では一般的にGRUBが利用されることが多い。ブートローダの詳細の処理内容についてもここでは省略とする。  

重要な点としては、ブートローダはDISK上に格納されたカーネルなどを読み込む必要があるという点である。
これを達成する素朴な方法は我々がDISK上のカーネルファイルの位置をブートローダに対して教えることであろう。
しかしgrub.cfgの内容を見てみると、カーネルやinitrdの位置をファイルパスの形でしか指定していないことを確認できるだろう。
これはブートローダがファイルシステムを解釈する能力を有している必要があることを意味する。
実際に、Boot Loaderいくつかのファイルシステムを解釈でき、ファイルシステム上のディレクトリパス情報からカーネルを探し出すことができる。
ただし、当然ながらBoot Loaderは特定のファイルシステムのみのサポートに留まるため、それ以外のフォーマットのものを解釈することはできないので注意されたい。
ブートローダによってgrub.cfgで指定されたカーネルとRAMディスクをメモリ上にロードし、カーネルの先頭アドレスにジャンプすることで、処理をカーネルへと引き渡し自身の処理を終える

カーネルの処理の話に入る前に、カーネルファイルについて少し整理しておく。
カーネルファイルは一般に`vmlinuz*`という名前がついているファイルである
我々にとって馴染みのあるカーネルファイルは`/boot/vmlinuz-*.img`と思われるが、このファイルは`bzImage`形式のファイルである。これは`file`コマンドで簡単に確認することができる。  
この`bzImage`はカーネル本体を含む圧縮バイナリファイルの他に、低レベルの初期化を行うためのファイルなどいくつかのファイルが含まれる形式になっている。
この`bzImage`を適切に解凍することでカーネルの実行バイナリを手に入れることもできる。
本資料では`bzImage`形式のカーネルを`vmlinuz`、実行バイナリ形式のカーネルを`vmlinux.bin`と表記する。

さてブートローダから`vmlinuz`に処理が引き渡されると`vmlinuz`は低レベルの初期化処理を実施後、カーネル本体を解凍、メモリにロードし、カーネルのエントリールーチンに処理を移す。
カーネルは全ての初期化処理を終えると、`tmpfs`ファイルシステムを作成し、ブートローダがRAM上に配置した`initramfs`をそこに展開し、ディレクトリルートにある`init`スクリプトを起動する。
この`init`スクリプトはDISK上に格納されている本命のファイルシステムをマウントするために必要な準備を整え、本命のファイルシステムやその他に重要なファイルシステムをマウントする。この時`initramfs`にはいくつかのデバイスドライバなども含まれているため、多様なフォーマットのルートファイルシステムをマウントすることが可能である。
さらにこれが完了すると、ルートを本命のルートファイルシステムに切り替え、そこに格納されている`/sbin/init` バイナリを起動する。

`/sbin/init`はシステムで最初に起動されるプロセス（PID=1が付与されるプロセス）であり、他のプロセスを起動させる役割を持っている全てのプロセスの親となるものである。
initにはさまざまな実装（`SysVinit`, `Upstart`）があるが、最近のCentOSやUbuntuなどで利用されているのは`Systemd`である。
initの最終的な責務は、システムの更なる準備とブートプロセスが終わった時点で必要なサービスが実行されておりユーザがログイン可能な状態まで持っていくことである。

以上が、非常に大雑把ではあるが電源投入からOSが起動するまでの流れである。

### initrdとinitramfs

上記に記載したLinux起動処理の中に、メモリ上に展開するファイルシステムである`initramfs`を紹介したが、我々がよく目にするのは`/boot/initrd.img`であろうと思う。ここでは`initrd`と`initramfs`との違いについて説明する
`initrd`は「initial RAM **disk**」、`initramfs`「initial RAM **File System**」であり両者は別物であるが、提供したい機能は同じで「本命のルートファイルシステムのマウントに必要なコマンド、ライブラリ、モジュール」を提供し、本命のルートファイルシステム上に存在する`/sbin/init`スクリプトを起動することである。  
もともと本来起動したいシステムは何かしらの記憶装置に書き込まれているが、これを読み込むには適切なデバイスドライバの存在と、これをマウントするファイルシステムが存在していないといけないという問題がある。  
`initrd`/`initramfs`は両方ともこの問題を解決する。

`initrd`と`initramfs`は上記の機能を提供するための**方式**が異なっており、名前の通りであるが`initrd`はブロックデバイス、`initramfs`は（`tmpfs`をもとにした）RAM filesystemの方式になっている。
従来は`initrd`を利用していたが、Kernel 2.6以降で`initramfs`が利用できるようになっており、現在はこちらの方式を利用することの方が一般的と思われる。
`initrd`から`initramfs`に移りわってきたのにはもちろん、`initrd`には問題があり、`initramfs`はそれの解決が測られたからである。
`initrd`には概ね以下のような問題が存在していた

1. RAM diskはRAM上に擬似的なブロックデバイスを作成し、これをあたかも二次記憶のように取り扱う仕組みであるため、通常のブロックデバイスと同様にメモリキャッシュ機構が働いてしまうために不必要にキャッシュメモリを消費する。さらにはページングのような機構が働いてしまうことで一層メモリを逼迫してしまう。
2. RAM diskはそのデータをフォーマットし解釈するためのext2のようなファイルシステムドライバーが必要である。
3. RAM diskのブロックデバイスは固定サイズになるため、あまりに小さいと必要なスクリプトを全て収めることができず、大きすぎると無駄にメモリを利用する

これを解決するために考案され、現在のデフォルトになっているのが`initramfs`である。  
`initramfs`はサイズを柔軟に設定できる軽量なメモリ内ファイルシステムである`tmpfs`をベースとして作られたfilesystemである。  
当然これはブロックデバイスではないので、キャッシュやページングでメモリを汚すこともなく、ブロックデバイスに対するファイルシステムドライバも不要で、さらに固定長という問題もうまく解決している。

`initrd`/`initramfs`いずれの方式にせよ、その中に格納されているツールを利用して本命のルートファイルシステムをマウントしそちらにルートを切り替えた上で、そのファイルシステム上に存在しているスタートアップスクリプトである`/sbin/init`を起動する。

#### initramfsの中身を確認する

`initramfs`の内容を展開し中身を確認してみる。Ubuntu 20.04.2 LTSの`initrd`を展開してみる。  
（注意: `initrd`という命名のファイルだが、このファイルはれっきとした`initramfs`である）。  
`initramfs`はいくつかのファイルをCPIOの形式にしたものが連結されているため、そのまま`cpio`コマンドで解凍しても以下のように冒頭のファイル（AuthenticAMD.bin）のみしか出てこない

```bash
$ mkdir initrd-work && cd initrd-work
$ sudo cp /boot/initrd.img ./
$ cat initrd.img| cpio -idvm
.
kernel
kernel/x86
kernel/x86/microcode
kernel/x86/microcode/AuthenticAMD.bin
62 blocks
```

`dd`/`cpio`の組み合わせで全てのファイルが展開できるが、`unmkinitramfs`という便利なコマンドがあるので今回はこちらを利用する

```bash
$ mkdir extract
$ unmkinitramfs initrd.img extract
$ ls extract
early  early2  main
```

解凍した結果、`early`, `early2`, `main`というディレクトリが作成されていることがわかる
例えばこの`early`は先ほどCPIOで解凍した際に出てきたファイルが確認できる
重要なのは、`main`の配下で、その中のコンテンツとしてファイルシステムルートの内容が格納されている

```bash
$ ls extract/early/kernel/x86/microcode
AuthenticAMD.bin
$ ls extract/early2/kernel/x86/microcode
GenuineIntel.bin
$ ls extract/main
bin  conf  cryptroot  etc  init  lib  lib32  lib64  libx32  run  sbin  scripts  usr  var
```

ここで解凍した内容に対してchrootすると、Linux起動時のRAM filesystemの内容を擬似的に操作でき、どのような操作ができるか把握することができる。

```
$ sudo chroot extract/main /bin/sh

BusyBox v1.30.1 (Ubuntu 1:1.30.1-4ubuntu6.3) built-in shell (ash)
Enter 'help' for a list of built-in commands.

# ls
scripts    init       run        etc        var        usr        conf
lib64      bin        lib        libx32     lib32      sbin       cryptroot
# pwd
/
# which mount
/usr/bin/mount
# exit
```

上記に示す通り、ルートに`init`というスクリプトファイルが入っており、これが`initramfs`を展開したのちに起動されるスクリプトである
このスクリプトを全て解説することはしないが、`init`スクリプトの中で`/proc/cmdline`の中身を読んでおり、ここから本来のroot filesystemが格納しているディスク情報（`root=/dev/sda1`のような記載）を拾い、マウント処理を実施しているようであった。
一方、この辺りが空の場合、Ubuntu 20.04LTSのinitrdから抜き出したこの`init`ファイルではエラーになるようだった。

今回のToyVMMでは以降説明する`firecracker-initrd`をベースとした`initramfs`を利用しているためこの辺りの挙動は少し異なる。

#### firecracker-initrdについて

ToyVMMでは、[firecracker-initrd](https://github.com/marcov/firecracker-initrd)を利用させてもらっている。
firecracker-initrdはAlpineをベースとしてinitrd.img（`initramfs`）を作成してくれる。
上記でみたUbuntuのinitrdとは異なり、microcodeなど追加のCPIOファイルは含まれないため、単純に解凍するだけでroot filesystemが確認できる

```
$ cat initrd.img | cpio -idv
$ ls
bin  dev  etc  home  init  initrd.img  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

Alpine Linuxは通常起動時にRAM上にファイルシステムが展開された上でOSが起動する。その後ニーズに応じて`setup-alpine`でDISKにOSを焼いたりするかなど決定する。
今回はこのAlpine Linuxのinitを使用しているため、この`initramfs`を利用して起動したVMは、デフォルトでは本命のルートファイルシステムをマウントせず、単純にRAM上にファイルシステムを展開しAlpine Linuxが起動することになる。  
これは、従来通りのOSのようにboot領域を二次記憶に置いた上で`/proc/cmdline`でinitスクリプトに伝えるという流れとは異なるものであるということを理解しておきたい。

## Boot sequence of linux kernel in ToyVMM

ここでこれまで議論してきた内容とToyVMMでのLinuxブートについての比較をしてみる

| Boot process (on Linux) | ToyVMM                                                               |
|-------------------------|----------------------------------------------------------------------|
| BIOS                    | Not implemented yet                                                  |
| Boot Loader             | **Require: vmlinux/initrd.img loading, basic setup required**        |
| Linux Kernel            | Processed by `vmlinux.bin`                                           |
| init                    | Processed by `init` scripts (from firecracker-initrd's `initrd.img`) |

現在のToyVMMの実装では`bzImage`の読み込みについてはサポートしておらず、ELFバイナリである`vmlinux.bin`を利用することとする。
現時点の実装ではBIOS関係については実装を省略している。  
BootLoaderが行う処理のうち、`vmlinux.bin`や`initrd.img`をメモリにロードするなどの処理を実装する必要がある。
Linux Kernel自体は`vmlinux.bin`が、`init`の処理は`initrd.img`内部の`init`スクリプトが担当するため、上記の処理を実装することで既存のLinux Kernelを起動すること自体は可能である。
より詳細の実装については[02-6_minimal_vmm_implementation](./02-6_minimal_vmm_implementation.md)で説明する。

## References

* [MBR(Master Boot Records)の構造](https://www.hazymoon.jp/OpenBSD/arch/i386/stand/mbr/mbr_structure.html)
* [Initrd(4) - Linux man page](https://linux.die.net/man/4/initrd)
* [Initramfsのしくみ](https://gihyo.jp/admin/serial/01/ubuntu-recipe/0384)
* [Initramfs/ガイド](https://wiki.gentoo.org/wiki/Initramfs/Guide/ja)
* [Kernel Boot Process](https://0xax.gitbooks.io/linux-insides/content/Booting/)
* [What's the Difference Between initrd and initramfs](https://www.baeldung.com/linux/initrd-vs-initramfs)
* [bzImage](https://wiki.bit-hive.com/linuxkernelmemo/pg/bzImage)
* [Initデーモンを理解する](http://www.seinan-gu.ac.jp/~shito/old_pages/hacking/shell/sh/boot_shutdown.html)
* [Linuxがブートするまで](https://keichi.dev/post/linux-boot/)
* [filesystems/ramfs-rootfs-initramfs.txt](http://archive.linux.or.jp/JF/JFdocs/kernel-docs-2.6/filesystems/ramfs-rootfs-initramfs.txt.html)
