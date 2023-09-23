# ToyVMM Implementation

To summarize our previous discussions, we have successfully created a minimal VMM with essential features. This ToyVMM is a straightforward VMM with the following functionalities:

- It can boot a Guest OS using `vmlinuz` and `initrd`.
- After the Guest OS boots, it can handle input and output as a Serial Terminal, allowing you to monitor and interact with the Guest's state.

## Run the Linux Kernel!

Let's actually boot the Linux Kernel.

First, prepare `vmlinux.bin` and `initrd.img`. Place them in the root directory of the ToyVMM repository. You can download `vmlinux.bin` as follows:

```bash
# Download vmlinux.bin
wget https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/x86_64/kernels/vmlinux.bin
cp vmlinux.bin <TOYVMM WORKING DIRECTORY>
```

For `initrd.img`, you can create it using [marcov/firecracker-initrd](https://github.com/marcov/firecracker-initrd), which includes an Alpine Linux root filesystem:

```bash
# Create initrd.img
# Using marcov/firecracker-initrd (https://github.com/marcov/firecracker-initrd)
git clone https://github.com/marcov/firecracker-initrd.git
cd firecracker-initrd
bash ./build.sh
# After the above commands, the initrd.img file will be located in build/initrd.img.
# Please move it to the working directory of ToyVMM.
cp build/initrd.img <TOYVMM WORKING DIRECTORY>
```

With these preparations completed, let's launch the Guest VM:

```bash
$ make run_linux
```

Here, we'll skip the output of the boot sequence, which will be displayed on the standard output. Once the boot is complete, you'll see the Alpine Linux screen, and it will prompt you for login credentials. You can log in using `root` as both the username and password:

```
Welcome to Alpine Linux 3.15
Kernel 4.14.174 on an x86_64 (ttyS0)

(none) login: root
Password:
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <http://wiki.alpinelinux.org/>.

You can set up the system with the command: setup-alpine

You may change this message by editing /etc/motd.

login[1058]: root login on 'ttyS0'
(none):~#
```

Great! You have successfully booted the Guest VM and can operate it. You can also execute commands within the Guest VM. For example, running the basic `ls` command results in the following output:

```
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

Well done! At this point, you have created a minimal VMM. However, there are some limitations:

- It can only be operated through a serial console. You might want to implement virtio-net for networking.
- Implementing virtio-blk for block devices.
- Handling PCI devices is not yet supported.

The creation of ToyVMM served several personal objectives, including:

- Deepening the understanding of virtualization.
- Gaining a better understanding of virtio.
- Learning about PCI passthrough:
  - Exploring technologies like VFIO.
  - Understanding peripheral technologies like mdev, libvfio, and VDPA.

While we have completed the creation of a minimal VMM, the direction you take it from here is up to you. ToyVMM is a great starting point, and you can choose to extend it in various ways. If you're reading this and you're an enthusiastic geek, I encourage you to give it a try! And if possible, I'd be delighted to receive feedback on ToyVMM.
