# Loading initrd

本稿では、VMを起動するにあたり`initrd`(`initramfs`)のロードやこれにまつわる設定について記載する。  
以降の記載では、`initrd`と書いたときも暗黙に`initramfs`を指しているとする。  
initramfs自体の説明は[Overview of booting linux](./02-1_overview_of_booting_linux.md)で既におこなっているのでそちらを確認されたい  

## Loading initrd and setup some parameters of kernel header

`initrd`をロードする関数は`load_initrd`に実装している。
引数としてはGuest用に確保したメモリと、`initrd`のファイルをOpenした`File`構造体(Read, Seekを実装している)の可変参照を渡している


```rust
fn load_initrd<F>(
    vm_memory: &memory::GuestMemoryMmap,
    image: &mut F,
) -> std::result::Result<InitrdConfig, StartVmError>
where F: Read + Seek {
    let size: usize;
    // Get image size
    match image.seek(SeekFrom::End(0)) {
        Err(e) => return Err(StartVmError::InitrdRead(e)),
        Ok(0) => {
            return Err(StartVmError::InitrdRead(io::Error::new(
                io::ErrorKind::InvalidData,
                "Initrd image seek returned a size of zero",
            )))
        }
        Ok(s) => size = s as usize,
    };
    // Go back to the image start
    image.seek(SeekFrom::Start(0)).map_err(StartVmError::InitrdRead)?;
    // Get the target address
    let address = arch::initrd_load_addr(vm_memory, size)
        .map_err(|_| StartVmError::InitrdLoad)?;

    // Load the image into memory
    //   - read_from is defined as trait methods of Bytes<A>
    //     and GuestMemoryMmap implements this trait.
    vm_memory
        .read_from(GuestAddress(address), image, size)
        .map_err(|_| StartVmError::InitrdLoad)?;

    Ok(InitrdConfig{
        address: GuestAddress(address),
        size,
    })
}
```

上記の処理でおこなっていること関数名の通りGuest用メモリへの`initrd`のロードであるが、内容としては以下の通り単純である
1. initrdのサイズの取得する（SeekFrom::End(0)としてファイルの末尾にカーソルを指定することでoffset=size取得をしている）
2. 1でサイズを取得するために動かしたカーソルを先頭に戻す
3. initrdをロードするべきGuestメモリのaddressを取得する
4. 上記Guestメモリのaddress位置にinitrdの中身を読み込む
5. `InitrdConfig`という構造体にGuestメモリのinitrd開始位置のアドレスとinitrdの値を詰めて返却する)

さて、上記でGuestメモリ上に`initrd`をロードすることはできたが、実際にこの領域をカーネルがどのように把握するのかという疑問が残っている  
ブートローダの責務の一つにカーネルのセットアップヘッダを読み込み、いくつかのフィールドを埋めるというものがある。  
このセットアップヘッダの内容は[Boot Protocol](https://docs.kernel.org/x86/boot.html#the-real-mode-kernel-header)として定義されており、上記のinitrdに関係する内容はこの値として格納されるべき値の一つになっている.  

今回、ToyVMMではこれらの内容を主に`configure_system`関数で以下の通り設定している。  
以下の内容については[Boot Protocol](https://docs.kernel.org/x86/boot.html#the-real-mode-kernel-header)を参照している。  
ここでは下記以外の設定項目については設定をしていないため説明を省略する。

| Offset/Size | Name             | Meaning                                     | ToyVMM value            |
|-------------|------------------|---------------------------------------------|-------------------------|
| 01FE/2      | boot_flag        | 0xAA55 magic number                         | 0xaa55                  |
| 0202/4      | header           | Magic signature "HdrS" (0x53726448)         | 0x5372_6448             |
| 0210/1      | type_of_loader   | Boot loader identifier                      | 0xff (undefined)        |
| 0218/4      | ramdisk_image    | initrd load address (set by boot loader)    | GUEST ADDRESS OF INITRD |
| 021C/4      | ramdisk_size     | initrd size (set by boot loader)            | SIZE OF INITRD          |
| 0228/4      | cmd_line_ptr     | 32-bit pointer to the kernel command line   | 0x20000                 |
| 0230/4      | kernel_alignment | Physical addr alignment required for kernel | 0x0100_0000             |
| 0238/4      | cmdline_size     | Maximum size of the kernel command line     | SIZE OF CMDLINE STRING  |

上記の内容をGuestMemoryの`0x7000`に書き込むコードになっている。  
この`0x7000`のアドレスは後述するvCPUのRSIの値として書き込んでおく値になる。  
vCPUのレジスタ設定関係については[Setup registers of vCPU](./02-4_setup_registers_of_vcpu)に記載しているので、本稿読了後に参照されたい。


## Setup E820

Guest OSのE820のセットアップを行うことで、OSやBootLoaderに対して利用可能なメモリ領域の報告できるようにしたい。
この辺りの処理は基本的にFirecrackerの実装に合わせて実装している。  

```rust
add_e820_entry(&mut params, 0, EBDA_START, E820_RAM)?;
let first_addr_past_32bits = GuestAddress(FIRST_ADDR_PAST_32BITS);
let end_32bit_gap_start = GuestAddress(MMIO_MEM_START);
let himem_start = GuestAddress(HIGH_MEMORY_START);
let last_addr = guest_mem.last_addr();
if last_addr < end_32bit_gap_start {
    add_e820_entry(
        &mut params,
        himem_start.raw_value() as u64,
        last_addr.unchecked_offset_from(himem_start) as u64 + 1,
        E820_RAM)?;
} else {
    add_e820_entry(
        &mut params,
        himem_start.raw_value(),
        end_32bit_gap_start.unchecked_offset_from(himem_start),
        E820_RAM)?;
    if last_addr > first_addr_past_32bits {
        add_e820_entry(
            &mut params,
            first_addr_past_32bits.raw_value(),
            last_addr.unchecked_offset_from(first_addr_past_32bits) + 1,
            E820_RAM)?;
    }
}
```

上記のコードはToyVMMで起動するGuest VMのアドレス全体の設計を見ながら理解した方が良いだろう。そのため以下に、現状の実装におけるGuestのメモリ設計を以下の通り一覧にしておく。
この内容は今後変更される可能性があるため注意されたい。

| Guest Address           | Contents                  | Note                                         |
|-------------------------|---------------------------|----------------------------------------------|
| 0x0 - 0x9FBFF           | E820                      |                                              |
| 0x7000 - 0x7FFF         | Boot Params (Header)      | ZERO_PAGE_START(=0x7000)                     |
| 0x9000 - 0x9FFF         | PML4                      | Now only 1 entry (8byte), maybe expand later |
| 0xA000 - 0xAFFF         | PDPTE                     | Now only 1 entry (8byte), maybe expand later |
| 0xB000 - 0xBFFF         | PDE                       | Now 512 entry (4096byte)                     |
| 0x20000 -               | CMDLINE                   | Size depends on cmdline parameter len        |
| 0x100000                |                           | HIGH_MEMORY_START                            |
| 0x100000 - 0x7FFFFFF    | E820                      |                                              |
| 0x100000 - 0x20E3000    | vmlinux.bin               | Size depends on vmlinux.bin's size           |
| 0x6612000 - 0x7FFF834   | initrd.img                | Size depends on initrd.img's size            |
| 0x7FFFFFF               | GuestMemory last address  | based on (128 << 20 = 128MB = 0x8000000) - 1 |
| 0xD0000000              |                           | MMIO_MEM_START（4GB - 768MB）                |
| 0xD0000000 - 0xFFFFFFFF |                           | MMIO_MEM_START - FIRST_ADDR_PAST_32BIT       |
| 0x100000000             |                           | FIRST_ADDR_PAST_32BIT (4GB~)                 |

コードを確認すると、GuestMemoryのサイズに非依存で設計しているアドレス帯（大まかに`0x0 ~ HIGH_MEMORY_START`のレンジ）は`0~EBDA_START(0x9FBFF)`の領域を共通でE820にUsableで登録している。
その後、GuestMemoryをどの程度確保しているかに従ってE820に登録している範囲が変化する。
現在の実装では、GuestのMemoryはデフォルトで128MBのメモリを確保するように実装しているためGuest Memoryは全体で`0x0 ~ 0x7FF_FFFF`になる。今回はこのレンジに`vmlnux.bin`の内容や`initrd.img`がマップされている。
つまり`guest_mem.last_addr() = 0x7FF_FFFF < 0xD000_0000 = end_32bit_gap_start`のロジックに該当するので、`HIGH_MEMORY_START ~ guest_mem.last_addr()`のレンジを追加で登録している。
今後拡張していく中で、GuestMemoryのサイズが4GB超える場合は、`0x10_0000 ~ 0xD000_0000`と`0x1_000_0000 ~ guest_mem.last_addr()`のレンジを登録することになる。

後ほどVM起動時のコンソール出力を確認できるようになるが、ここでは確認のために先取ってVM起動時の一部の出力を添付する。
以下のように上記で設定したE820エントリが登録できている。

```bash
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x0000000007ffffff] usable
```

## References

* [Linuxのブートシーケンスの基礎まとめ](https://nishidy.hatenablog.com/entry/2016/09/08/230637)
* [Linuxカーネルユーザ・管理者ガイド - 初期RAMdディスクを使用する](https://doc.kusakata.com/admin-guide/initrd.html)
* [initrd](https://manpages.ubuntu.com/manpages/bionic/ja/man4/initrd.4.html)
* [initramfs(initrd)のinitをbusyboxだけで書いてみた](https://www.gcd.org/blog/2007/09/129/)
* [initramfsとinitrdについて](https://blog.goo.ne.jp/pepolinux/e/4d1f4b6e0f5b5ed389fcec1f711b1408)
* [initramfsについて](https://qiita.com/akachochin/items/d38b538fcabf9ff80531)
* [filesystem/ramfs-rootfs-initramfs.txt](http://archive.linux.or.jp/JF/JFdocs/kernel-docs-2.6/filesystems/ramfs-rootfs-initramfs.txt.html)
