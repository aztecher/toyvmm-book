# ToyVMM implememtation

これまでの話を合わせると、必要最低限の機能を保有したVMMが作成できる。  
このToyVMMは以下の機能を持っているシンプルなVMMである

- vmlinuz, initrdを利用して、Guest OSを起動できる
- Guest OS起動後はSerial Terminalとして入出力を受け、Guestの状態確認や操作を実施することができる。

## Run linux kernel !

実際にLinux Kernelを起動する

まずは、`vmlinux.bin`と`initrd.img`を用意します。いずれもtoyvmmのリポジトリルートに配置する。
`vmlinux.bin`は以下を参考にダウンロードする

```bash
# Download vmlinux.bin
wget https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/x86_64/kernels/vmlinux.bin
cp vmlinux.bin <TOYVMM WORKING DIRECTORY>
```

`initrd.img`は、[marcov/firecracker-initrd](https://github.com/marcov/firecracker-initrd.git)を利用し、alpineのrootfsを含むinitrd.imgを作成する。

```bash
# Create initrd.img
# Using marcov/firecracker-initrd (https://github.com/marcov/firecracker-initrd)
git clone https://github.com/marcov/firecracker-initrd.git
cd firecracker-initrd
bash ./build.sh
# After above commands, initrd.img file wil be located on build/initrd.img.
# So, please move it to the working directory of toyvmm.
cp build/initrd.img <TOYVMM WORKING DIRECTORY>
```

以上で準備完了ある。以下のコマンドを実行しGuest VMを起動してみよう！

```bash
$ make run_linux
```

ここではブートシーケンスの出力については省略する。Guest VMの起動シーケンスが標準出力に表示されていくだろう。
起動が完了すると、以下のようにAlpine Linuxの画面とともにloginが要求されるため、root/rootでログインしよう.  

```bash
Welcome to Alpine Linux 3.15
Kernel 4.14.174 on an x86_64 (ttyS0)

(none) login: root
Password:
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <http://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

login[1058]: root login on 'ttyS0'
(none):~#
```

素晴らしい！確かにGuest VMを起動し操作できています！
もちろん、Guest VM内部でコマンドを実行することもできます。  
例えば最も基本的なlsコマンドを実行した結果は以下のようになります。

```bash
(none):~# ls -lat /
total 0
drwx------    3 root     root            80 Sep 23 06:44 root
drwxr-xr-x    5 root     root           200 Sep 23 06:44 run
drwxr-xr-x   19 root     root           400 Sep 23 06:44 .
drwxr-xr-x   19 root     root           400 Sep 23 06:44 ..
drwxr-xr-x    7 root     root          2120 Sep 23 06:44 dev
dr-xr-xr-x   12 root     root             0 Sep 23 06:44 sys
dr-xr-xr-x   55 root     root             0 Sep 23 06:44 proc
drwxr-xr-x    2 root     root          1780 May  7 00:55 bin
drwxr-xr-x   26 root     root          1040 May  7 00:55 etc
lrwxrwxrwx    1 root     root            10 May  7 00:55 init -> /sbin/init
drwxr-xr-x    2 root     root          3460 May  7 00:55 sbin
drwxr-xr-x   10 root     root           700 May  7 00:55 lib
drwxr-xr-x    9 root     root           180 May  7 00:54 usr
drwxr-xr-x    2 root     root            40 May  7 00:54 home
drwxr-xr-x    5 root     root           100 May  7 00:54 media
drwxr-xr-x    2 root     root            40 May  7 00:54 mnt
drwxr-xr-x    2 root     root            40 May  7 00:54 opt
drwxr-xr-x    2 root     root            40 May  7 00:54 srv
drwxr-xr-x   12 root     root           260 May  7 00:54 var
drwxrwxrwt    2 root     root            40 May  7 00:54 tmp
```

お疲れ様でした。ひとまずここまで来ればminimal VMMと言って良いものではないかと思います。
とはいえ現状以下のような問題点があります

- serialコンソール経由でしか操作できない -> virtio-netを実装したい
- virtio-blkを実装したい
- PCIデバイスを扱えていない

実は、ToyVMMを作成し始めた個人的な目的の一つとしては以下のような目標がありました。  

- virtualizationの理解を深める
- virtioについて理解を深める
- pci passthroughについて理解を深める
  - vfioなどの技術について詳細を知る
  - mdev/libvfio/vdpaなどの周辺技術について知る

MinimalなVMMの作成は完了しましたが、この先VMMとしてどのように成長させていくかというのは人それぞれで多くの選択肢があると思います。
ToyVMMでは今後上記のようなtopicについて扱っていきたいと考えています！  
ToyVMMをベースにして別の方向に拡張するというのも面白いでしょう。もしこれを読んでくれているGeekがいればぜひ挑戦してください。そしてもしよければToyVMMにもfeedbackしてくれると大変嬉しく思います。


