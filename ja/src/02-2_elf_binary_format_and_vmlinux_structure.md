# ELF binary format and vmlinux structure

本稿執筆時、ToyVMMでVMを起動する際に利用するカーネルはELF形式の`vmlinux.bin`を前提としている。  
そのため、VMMの内部ではELF形式を解釈し、適切にカーネルをVM用に用意したメモリ領域にロードする必要がある。 
この処理は [`rust-vmm/linux-loader`](https://github.com/rust-vmm/linux-loader) crateで実装されており、ToyVMMではこのcrateを利用するため実装としては隠蔽されてしまうが、このcrateの中でどのように処理されているかを知ることは重要だと判断したため、本章を設けELFバイナリのロードに関する解説を記載することとした。  

## ELF Binary Format

ELFのファイルフォーマットは以下のようになっている

<img src="./02_figs/elf_format.svg" width="100%">

上記の通り、ELFファイルフォーマットは基本的に`ELF Header`、`Program Header Table`、`Segument(Sections)`、`Section Header Table`からなる。  
ELFファイルはシステムローダが利用する場合は`Program Header Table`に記述された`Segment`の集合として取り扱われ、コンパイラ・アセンブラ・リンカは`Section Header Table`に記述された`Section`の集合として扱われる。  

`ELF Header`はこのELFファイルの全体的な情報を保持している。  
`Program Header Table`の各エントリである`Program Header`は、それぞれが対応する`Segument`についてのHeader情報を保持している。つまり、`Program Header`の数だけ、`Segment`が存在していることになる。  
また、この`Segument`はさらに複数の`Seciton`という単位に分割でき、この`Section`単位でヘッダ情報を保持しているのが`Section Header Table`である。  

`ELF Header`は常にファイルオフセットの先頭から始まっており、ELFデータを読み込むために必要となる情報を保持している。
以下に`ELF Header`の内容を一部抜粋する。全体の構成を知りたい場合は[Man page of ELF](https://linuxjm.osdn.jp/html/LDP_man-pages/man5/elf.5.html)を参考にされたい

| Attribute     | Meaning                                                         |
|---------------|-----------------------------------------------------------------|
| `e_entry`     | このELFプロセスを開始する際のエントリポイントとなる仮想アドレス |
| `e_phoff`     | `Program Header Table`が存在する場所のファイルオフセット値      |
| `e_shoff`     | `Section Header Table`が存在する場所のファイルオフセット値      |
| `e_phentsize` | `Program Header Table`にある1エントリのサイズ                   |
| `e_phnum`     | `Program Header Table`中のエントリの個数                        |
| `e_shentsize` | `Section Header Table`にある1エントリのサイズ                   |
| `e_shnum`     | `Section Header Table`中のエントリの個数                        |

上記で抜粋した内容から、`Program Header`や`Section Header`の各エントリの情報を取り出すことが可能であると分かるであろう。


ここで、`Program Header`の内容を一部抜粋する。

| Attribute     | Meaning                                                                                                     |
|---------------|-------------------------------------------------------------------------------------------------------------|
| `p_type`      | この`Program Header`が指す`Segment`の種類を表現しており、解釈の方法についてのヒントを与える                 |
| `p_offset`    | この`Program Header`が指す`Segment`のファイルオフセット値                                                   |
| `p_paddr`     | 物理アドレスが意味を持つシステムでは、この値は`Program Header`が指す`Segment`の物理アドレスを指す           |
| `p_filesz`    | この`Program Header`が指す`Segment`のファイルイメージのバイト数                                             |
| `p_memsz`     | この`Program Header`が指す`Segment`のメモリイメージのバイト数                                               |
| `p_flags`     | この`Program Header`が指す`Segment`の情報を示すフラグで、実行可能、書き込み可能、読み取り可能を表現している |

上述の通り、`Program Header`の中身を解釈することで、当該セグメントの位置やサイズ、どの様に解釈すべきかの情報を手に入れることができる。
今回の内容はこの`Program Header`の構造まで把握できていれば十分であるため、`Section Header`やそのほかの詳細については省略する。  
興味がある方は、[Man page of ELF](https://linuxjm.osdn.jp/html/LDP_man-pages/man5/elf.5.html)等を参考に確認されたい。

さて、後述するが今回取り扱う`vmlinux.bin`は`Program Header`の数が5個で、内4つの`p_type`の値が`PT_LOAD`、最後の一つだけ`PT_NOTE`になっているという大変簡単な構造になっている。
ここで、`PT_LOAD`、`PT_NOTE`についてのみ、[Man page of ELF](https://linuxjm.osdn.jp/html/LDP_man-pages/man5/elf.5.html)からその詳細内容を部分的に抜粋する。一部情報を削っているため、必要に応じて参考資料を確認されたい。

| `p_type`      | Meaning                                                                                           |
|---------------|---------------------------------------------------------------------------------------------------|
| `PT_LOAD`     | この要素は`p_filesz`と`p_memsz`で記述される読み込み可能な`Segment`である。                        |
| `PT_NOTE`     | この要素はロケーションとサイズのための補助情報が書き込まれている                                  |

`PT_LOAD`では、ファイルのバイト列はメモリセグメントの先頭に対応づけされているため、`p_offset`を利用して得られる、セグメントのメモリアドレスからサイズ分（基本的には`p_memsz`を利用する）をCOPYすることでセグメントの内容を読み込むことができる。

以上で必要最低限なELFの知識を身につけることができたので、次は実際に`vmlinux.bin`をダンプしてみて中身を確認してみる。

## vmlinxの解析

それではここで`vmlinux`の内容を少し解析してみよう。  
この解析内容の一部は今後重要な要素になってくるため是非把握してもらいたい。  
`readelf`コマンドはELFフォーマットのファイルを理解しやすい形でダンプしてくれる非常に心強いツールである。
ここではvmlinuxのELF Header(`-h`)、Program Header（`-l`）をそれぞれ表示してみる

```bash
$ readelf -h -l vmlinux.bin
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1000000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          21439000 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         5
  Size of section headers:           64 (bytes)
  Number of section headers:         36
  Section header string table index: 35

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                 0x0000000000b72000 0x0000000000b72000  R E    0x200000
  LOAD           0x0000000000e00000 0xffffffff81c00000 0x0000000001c00000
                 0x00000000000b0000 0x00000000000b0000  RW     0x200000
  LOAD           0x0000000001000000 0x0000000000000000 0x0000000001cb0000
                 0x000000000001f658 0x000000000001f658  RW     0x200000
  LOAD           0x00000000010d0000 0xffffffff81cd0000 0x0000000001cd0000
                 0x0000000000133000 0x0000000000413000  RWE    0x200000
  NOTE           0x0000000000a031d4 0xffffffff818031d4 0x00000000018031d4
                 0x0000000000000024 0x0000000000000024         0x4

 Section to Segment mapping:
  Segment Sections...
   00     .text .notes __ex_table .rodata .pci_fixup __ksymtab __ksymtab_gpl __kcrctab __kcrctab_gpl __ksymtab_strings __param __modver
   01     .data __bug_table .vvar
   02     .data..percpu
   03     .init.text .altinstr_aux .init.data .x86_cpu_dev.init .parainstructions .altinstructions .altinstr_replacement .iommu_table .apicdrivers .exit.text .smp_locks .data_nosave .bss .brk
   04     .notes
```

ELF Headerをみてみると、Entry point address (`e_entry`)の値としてProgram Headerの最初のセグメントの物理アドレスの値である(`0x0100_0000`)が格納されていることが分かる。この値は`rust-vmm/linux-loader`の実装としてkernelをロードした際の返り値として返却される値であり、かつvCPUの`eip`（命令アドレスレジスタ）に設定する値でもあるため重要である。  
また、ELF HeaderのNumber of program headers(`e_phnum`)の値である`5`と同じ数のProgram Headerが確認でき、Program Headerを出力をみると先頭4つはTypeが`LOAD`、最後は`NOTE`となっていることが確認できる。  
また、1つ目、および4つ目の`LOAD`セグメントはFlagを確認すると`E(xecutable)`がマークされており、この辺りに実行可能コードが存在していることも分かる。
特に1つめのエントリは実際にカーネルの実行バイナリのエントリポイントに該当する内容が配置されていることが期待される。  
今回はこれ以上の深追いは控えておくが、興味がある人はELFのSpecificationをもとにさらに解析をしてみるのも面白いかもしれない。

## ToyVMMでの実装

ToyVMMでは、`src/builder.rs`の中の`load_kernel`関数の中でvmlinuxの読み込みを実施している。  
この関数には、カーネルファイルへのパス情報などが含まれている`boot_config`とVM向けに確保したメモリ(`guest_memory`)を渡している。
`load_kernel`が実施していることは単純で、`boot_config`からカーネルファイルへのパスを取得し、`linux-loader`の`Elf`という構造体を`Loader`という名前で取り扱い、この構造体に実装されているELF形式のLinuxのローディング処理を適切な引数を伴って実行しているだけである。  

```
use linux_loader::elf::Elf as Loader;

let entry_addr = Loader::load::<File, memory::GuestMemoryMmap>(
    guest_memory,
    None,
    &mut kernel_file,
    Some(GuestAddress(arch::x86_64::get_kernel_start())),
).map_err(StartVmError::KernelLoader)?;
```

さて、ここから`linux-loader`の実装について深掘りしてみよう。
`linux-loader`では、`KernelLoader` traitが定義されており、その定義は以下のようになっている

```
/// Trait that specifies kernel image loading support.
pub trait KernelLoader {
    /// How to load a specific kernel image format into the guest memory.
    ///
    /// # Arguments
    ///
    /// * `guest_mem`: [`GuestMemory`] to load the kernel in.
    /// * `kernel_offset`: Usage varies between implementations.
    /// * `kernel_image`: Kernel image to be loaded.
    /// * `highmem_start_address`: Address where high memory starts.
    ///
    /// [`GuestMemory`]: https://docs.rs/vm-memory/latest/vm_memory/guest_memory/trait.GuestMemory    .html
    fn load<F, M: GuestMemory>(
        guest_mem: &M,
        kernel_offset: Option<GuestAddress>,
        kernel_image: &mut F,
        highmem_start_address: Option<GuestAddress>,
    ) -> Result<KernelLoaderResult>
    where
        F: Read + Seek;
}
```

コメントから推測できるように、このtraitが実装しているべき`load`関数は、特定のカーネルイメージフォーマットをGuestMemoryに読み込むような実装になっていることを要求している。
linux-loaderではx86_64向けの実装として、ELF形式の他にbzImage形式のカーネルの読み込みについても実装が存在しているようであるが、ひとまず今回はELF向けの実装を利用する。

さて、先のToyVMM側のコードで利用していた`Elf`構造体（`Loader`と名前を変えてimportした構造体）はこの`KernelLoader` traitを実装しており、その`load`関数がELFファイルをロードする実装になっていることが期待できる。  
そのため、このload関数を見てみると以下のような処理になっていることがわかる。処理内容がすこし長いためコードの転載は控える。

1. ELFファイルの先頭から、ELFヘッダー分のデータを抜き出す
2. `loader_result`という変数名の`KernelLoaderResult`構造体のインスタンスを作成し、`kernel_load`メンバにELFヘッダの`e_entry`の値を格納しておく。この値はシステムが最初に制御を渡すアドレス、つまりプロセスを開始する仮想アドレスに該当する。
3. ELFファイルを先頭からプログラムヘッダテーブルが存在するアドレスまで（`e_phoff`分）シークし、プログラムヘッダテーブル数分（`e_phnum`分）ループしながら、ELFファイルに含まれているプログラムヘッダを全て抜き出す。
4. 上記のプログラムヘッダをループしつつ以下の内容を行う
    * ELFファイルの先頭から今確認しているプログラムヘッダに対応するセグメントまで（`p_offset`分）シーク
    * Guestのメモリに対して、`mem_offset`から算出したmemory regionのアドレス位置を先頭に、`kernel_image`（`p_offset`分シーク済みなので、プログラムヘッダに対応するセグメントのデータの先頭）から、セグメントのサイズ分(`p_filesz`分)だけを書き込む
    * `kernel_end`（GuestMemory上での読み込んだセグメントの末尾のアドレス）の値を更新し、`loader_result.kernel_end`（2回目以降のループでは前回の値が記録されている）と比較して大きい方の値を`loader_result.kernel_end`に格納しておく
5. 全てのプログラムヘッダをループ後、返り値として最終的な`loader_result`を返却する。

これはまさに上記でみたELFフォーマットを解釈し読み込むコードになっていることがわかる。  
また当該関数呼び出しの結果返却される`KernelLoaderResult`の値には、最終的なGuestMemory上でのカーネルの開始位置、終了位置の情報が含まれており、特にこの開始位置の情報は[Setup registers of vCPU](./02-4_setup_registers_of_vcpu.md)で利用する値になるため重要である。

## References

* [vmlinux](https://valinux.hatenablog.com/entry/20200910)
* [ELF Formatについて](https://www.hazymoon.jp/OpenBSD/annex/elf.html)
* [Understanding the ELF File Format](https://linuxhint.com/understanding_elf_file_format/)
* [ELF形式のヘッダ部分を解析する単純なプログラムを作ってみた](https://qiita.com/niwaka_dev/items/b915d1ffc7a677c74959)
