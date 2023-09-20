# Coffee Break: Customize Initrd

以降の内容の都合上、initrd.imgにいくつかのパッケージやドライバをインストールする必要が出てきた。  

ここで今後toyvmmで利用するinitrd.imgのカスタマイズをできるようにしておく。



これまでtoyvmmは[marcov/firecracker-initrd](https://github.com/marcov/firecracker-initrd)で作成されるinitrd.imgを利用させてもらっていたが、これを調整する形で独自のimageを作成できるようにする。


* lspciが欲しかったのでpci-utilsを追加した。

```bash
localhost:~# which lspci
/usr/bin/lspci
localhost:~# ls -lat /usr/share/hwdata/
total 1308
drwxr-xr-x    2 root     root            60 Apr 29 01:22 .
drwxr-xr-x    8 root     root           160 Apr 29 01:22 ..
-rw-r--r--    1 root     root       1336789 Nov  1 18:41 pci.ids

# ただしPCIデバイスを認識していないので出てこない
# これは続いてのvirtioの部分でやる
localhost:~# lspci
pcilib: Cannot open /proc/bus/pci
lspci: Cannot find any working access method.
```


DeviceTreeを組み込みたいんだけどどうすればいいんだろうな
PCI Transport実装する方が直接的で早いかもしれんけど
