# Overview of Booting Linux

### General Booting Mechanism

In Linux, the operating system starts by executing programs in the following order:

1. BIOS
2. Boot Loader (GRUB)
3. Linux Kernel (vmlinuz)
4. init

The BIOS program is stored in the ROM on the motherboard. When you power on your computer, the CPU is instructed to start executing code from a specific address mapped to this ROM area. The BIOS performs hardware detection and initialization, then searches for the OS boot drive (HDD/SSD, USB flash drive, etc.). During this process, the boot drive needs to be formatted in either MBR or GPT format, depending on the BIOS type, as shown in the table below:

| BIOS \ DISK Format | MBR | GPT |
|--------------------|-----|-----|
| Legacy BIOS        | ◯   | -   |
| UEFI               | ◯ * | ◯   |

\* UEFI supports Legacy Boot Mode and thus supports MBR.

Next, I will explain the process of searching for the OS when using MBR. But before going into details, let's briefly review the structure of MBR. The MBR structure explained here assumes HDD/SSD or USB flash memory and implicitly assumes the presence of the Partition Entry, as described later. Please note that this document uses the terms provided on [Wikipedia](https://en.wikipedia.org/wiki/Master_boot_record), so keep that in mind.

MBR is a 512-byte sector located at the beginning of the boot drive and consists of three main parts:

1. Bootstrap code area (446 bytes)
2. Partition Entry (64 bytes = 16 bytes * 4)
3. Boot Signature (2 bytes)

I won't go into the details of MBR here, but the Boot code area contains machine code programs (Boot Loaders) to boot the OS, and the Partition Entry stores information about the logical partitions on that disk. It's worth noting that the Boot code area is only 446 bytes in size, so Boot Loaders are typically stored elsewhere. A minimal program is placed in the Boot code area to load the actual Boot Loader into memory.  
The critical part here is the "Boot Signature," which contains a 2-byte value used to ensure that the drive is a bootable device. When the BIOS searches for the OS boot drive, it reads the first sector (512 bytes), checks if the last 2 bytes (Boot Signature) are `0x55` and `0xAA`, and identifies the drive as a bootable disk. It loads the first sector (512 bytes) from that disk into memory at `0x7c00 - 0x7fff` and begins executing the program from `0x7c00`.

Now, as a simple validation, let's check the Boot Signature on your machine. In this example, you are using a virtual machine with the boot drive labeled as `vda`. On a regular machine, it might be something like `sda`. By writing the first sector's content to a file and examining the 2 bytes at an offset of 510 bytes, you should see the `0x55` `0xAA` signature as expected.

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

Now, back to our discussion. After confirming the Boot Signature, the BIOS identifies the disk as a bootable disk and loads the first sector (512 bytes) from it into memory at address `0x7c00`. The program execution starts from `0x7c00`.

Moving on, once the Boot Loader is loaded into memory, it takes on the responsibility of loading the Linux Kernel and initramfs from the disk and starting the kernel. In recent years, GRUB has become a common choice as a Boot Loader. I'll skip the detailed workings of the Boot Loader for now. The essential point is that the Boot Loader needs to load the specified kernel and initrd from the disk.

To achieve this, one straightforward method would be to inform the Boot Loader of the location of the kernel file on the disk. However, if you look at the contents of `grub.cfg`, you'll notice that the kernel and initrd locations are specified in the form of file paths. This means that the Boot Loader must have the ability to interpret the file system. In practice, several Boot Loaders can interpret various file systems and locate the kernel based on directory path information. However, it's essential to note that Boot Loaders are limited to supporting specific file system formats, and they cannot interpret other formats. The Boot Loader loads the specified kernel and RAM disk from `grub.cfg`, and by jumping to the kernel's entry point, it hands over the execution to the kernel, completing its own processing.

Before delving into the details of the kernel's processing, let's briefly organize some information about the kernel file. The kernel file is generally named `vmlinuz*`. You might be familiar with a kernel file located at `/boot/vmlinuz-*`, which is believed to be the kernel. However, this file is in the `bzImage` format. You can easily check this using the `file` command. The `bzImage` includes the actual kernel binary along with several other files used for low-level initialization. In this document, I'll refer to the kernel file in the `bzImage` format as `vmlinuz` and the actual kernel binary in executable format as `vmlinux.bin`.

When control is handed over from the BootLoader to `vmlinuz`, `vmlinuz` performs low-level initialization, then decompresses the kernel core, loads it into memory, and transfers control to the kernel's entry routine. Once all initialization processes are completed, the kernel creates a `tmpfs` filesystem, unpacks the `initramfs` placed in RAM by the BootLoader into it, and starts the `init` script located in the root directory.

This `init` script prepares to mount the main filesystem stored on the disk and mounts other important filesystems. `initramfs` contains various device drivers and allows mounting root filesystems in different formats. After this is done, the root is switched to the main root filesystem, and the `/sbin/init` binary stored there is executed.

`/sbin/init` is the first process to be launched in the system (with PID=1), and it serves as the parent for all other processes responsible for starting other processes. There are various implementations of `init`, such as `SysVinit` and `Upstart`, but what is commonly used in recent systems like CentOS and Ubuntu is `Systemd`. The ultimate responsibility of `init` is to further prepare the system and ensure that the necessary services are running and the system is in a state where users can log in when the boot process is complete.

This is a very high-level overview of the process from powering on to the OS booting up.

### initrd and initramfs

In the previously discussed Linux boot process, we introduced `initramfs`, a file system that is unpacked into memory. However, what we often encounter is `/boot/initrd.img`. Here, we will explain the differences between `initrd` and `initramfs`.

`initrd` stands for "initial RAM **disk**, while `initramfs` stands for "initial RAM **File System**". Although they are different in nature, they serve the same purpose, which is to provide the necessary commands, libraries, and modules for mounting the root file system and launching the `/sbin/init` script located in the root file system. 

The challenge that both `initrd` and `initramfs` address is that the system you want to boot originally resides in some storage device. To load it, you need appropriate device drivers and a file system for mounting. 

`initrd` and `initramfs` both address this issue, but they use different methods. As their names suggest, `initrd` uses a block device, while `initramfs` uses a RAM file system based on `tmpfs`. Traditionally, `initrd` was used, but starting from Kernel 2.6, `initramfs` became available, and it is now the more common choice. 

The shift from `initrd` to `initramfs` occurred because `initrd` had several issues:

1. A RAM disk is a mechanism that creates a pseudo block device in RAM, treating it as if it were a secondary storage device. However, because of this behavior, it inadvertently consumes memory cache, similar to regular block devices, leading to unnecessary memory usage. Furthermore, mechanisms such as paging come into play, consuming more memory capacity.

2. A RAM disk requires a file system driver, such as ext2, to format and interpret its data.

3. RAM disks have a fixed size, which can lead to problems; if they are too small, they may not accommodate all the necessary scripts, and if they are too large, they waste memory.

To address these issues, `initramfs` was developed. It is a lightweight, memory-based file system that can be flexibly sized and is based on `tmpfs`. It is not a block device, so it doesn't interfere with memory caching or paging, and it doesn't require file system drivers for block devices. Additionally, it resolves the fixed size problem.
Whether using `initrd` or `initramfs`, both methods provide tools inside them to mount the root file system and switch to it. The startup script, `/sbin/init`, located in the file system, is then executed.

#### Inspecting the contents of initramfs

Let's unpack and examine the contents of an `initramfs`. We'll use an Ubuntu 20.04.2 LTS `initrd` for this example. (Note: The file named `initrd` is actually a proper `initramfs`). An `initramfs` consists of several files concatenated in CPIO format. When you extract it directly using the `cpio` command, you'll see only the initial files (like `AuthenticAMD.bin`) as follows:

```bash
$ mkdir initrd-work && cd initrd-work
$ sudo cp /boot/initrd.img ./
$ cat initrd.img | cpio -idvm
.
kernel
kernel/x86
kernel/x86/microcode
kernel/x86/microcode/AuthenticAMD.bin
62 blocks
```

You can extract all the files using a combination of `dd` and `cpio`, but there's a handy tool called `unmkinitramfs` that can do this for you:

```bash
$ mkdir extract
$ unmkinitramfs initrd.img extract
$ ls extract
early  early2  main
```

After extracting, you'll see directories like `early`, `early2`, and `main`. For instance, `early` contains the same files that were seen when extracted with `cpio`. The most crucial part is under `main`, where the contents of the file system root are stored:

```bash
$ ls extract/early/kernel/x86/microcode
AuthenticAMD.bin
$ ls extract/early2/kernel/x86/microcode
GenuineIntel.bin
$ ls extract/main
bin  conf  cryptroot  etc  init  lib  lib32  lib64  libx32  run  sbin  scripts  usr  var
```

By chrooting into this extracted content, you can pseudo-operate the Linux boot-time RAM filesystem and understand what operations can be performed:

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

As shown above, there is an `init` script file in the root directory, which is the script executed after extracting `initramfs`. The `init` script reads the contents of `/proc/cmdline` and extracts disk information (e.g., `root=/dev/sda1`) to perform the necessary mounting operations. If this information is missing, this `init` script in the Ubuntu 20.04LTS initrd would encounter an error.

In the case of ToyVMM, we use an `initramfs` based on `firecracker-initrd`.
Therefore, the behavior might differ slightly.

#### About firecracker-initrd

In ToyVMM, we use [firecracker-initrd](https://github.com/marcov/firecracker-initrd). Firecracker-initrd creates an initrd.img (`initramfs`) based on Alpine Linux. Unlike the Ubuntu initrd we discussed earlier, it does not include additional CPIO files like microcode, so you can simply extract it to see the root filesystem:

```
$ cat initrd.img | cpio -idv
$ ls
bin  dev  etc  home  init  initrd.img  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

Alpine Linux typically unpacks a filesystem into RAM during normal boot, and then the OS starts. Afterward, decisions like whether to write the OS to a disk using `setup-alpine` depend on specific needs. In ToyVMM, when you boot a VM using this initramfs, it doesn't immediately mount the root file system by default. Instead, it simply unpacks the file system into RAM, and Alpine Linux starts. This is different from the traditional approach where you load the boot area into secondary storage and inform the init script via `/proc/cmdline`.

## Boot Sequence of Linux Kernel in ToyVMM

Now, let's compare what we've discussed so far with the Linux boot process in ToyVMM:

| Boot Process (on Linux) | ToyVMM                                                               |
|-------------------------|----------------------------------------------------------------------|
| BIOS                    | Not implemented yet                                                  |
| Boot Loader             | **Requires implementation: Loading vmlinux/initrd.img, basic setup** |
| Linux Kernel            | Processed by `vmlinux.bin`                                           |
| init                    | Processed by `init` scripts (from firecracker-initrd's `initrd.img`) |

The current implementation of ToyVMM does not support loading `bzImage` and instead uses the ELF binary `vmlinux.bin`. It currently omits BIOS-related functions.

For the Boot Loader's tasks, such as loading `vmlinux.bin` and `initrd.img` into memory, implementation is needed. The Linux Kernel itself is processed by `vmlinux.bin`, while the `init` process is handled by the `init` scripts found in `initrd.img` from the `firecracker-initrd`.

For more detailed implementation instructions, you can refer to [02-6_minimal_vmm_implementation](./02-6_minimal_vmm_implementation.md).

References

* [MBR(Master Boot Records)の構造](https://www.hazymoon.jp/OpenBSD/arch/i386/stand/mbr/mbr_structure.html)
* [Initrd(4) - Linux man page](https://linux.die.net/man/4/initrd)
* [Initramfsのしくみ](https://gihyo.jp/admin/serial/01/ubuntu-recipe/0384)
* [Initramfs/ガイド](https://wiki.gentoo.org/wiki/Initramfs/Guide/ja)
* [Kernel Boot Process](https://0xax.gitbooks.io/linux-insides/content/Booting/)
* [What's the Difference Between initrd and initramfs](https://www.baeldung.com/linux/initrd-vs-initramfs)
* [bzImage](https://wiki.bit-hive.com/linuxkernelmemo/pg/bzImage)
* [Initデーモンを理解する](http://www.seinan-gu.ac.jp/~shito/old_pages/hacking/sh/boot_shutdown.html)
* [Linuxがブートするまで](https://keichi.dev/post/linux-boot/)
* [filesystems/ramfs-rootfs-initramfs.txt](http://archive.linux.or.jp/JF/JFdocs/kernel-docs-2.6/filesystems/ramfs-rootfs-initramfs.txt.html)
