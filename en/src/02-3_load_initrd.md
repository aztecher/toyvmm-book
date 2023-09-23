# Loading initrd

In this document, we will discuss loading and configuring `initrd` (`initramfs`) in order to boot a VM. When we mention `initrd` in the following sections, we are implicitly referring to `initramfs`. A detailed explanation of `initramfs` itself can be found in [Overview of booting Linux](./02-1_overview_of_booting_linux.md), so please refer to that section for more information.

## Loading initrd and setting up kernel header parameters

The function responsible for loading `initrd` is implemented as `load_initrd`. It takes two arguments: the memory allocated for the Guest and a mutable reference to the `File` structure representing the opened `initrd` file (implementing `Read` and `Seek` traits).

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

The function performs the following steps:
1. Retrieves the size of the `initrd` by seeking to the end of the file and then returning to the start.
2. Calculates the target address in Guest memory where the `initrd` should be loaded.
3. Loads the contents of the `initrd` file into the specified Guest memory address.
4. Returns an `InitrdConfig` structure containing the Guest memory address and size of the loaded `initrd`.

Once the `initrd` is loaded into memory, we need to configure the kernel's setup header. This header information is defined by the [Boot Protocol](https://docs.kernel.org/x86/boot.html#the-real-mode-kernel-header). In ToyVMM, these settings are primarily configured in the `configure_system` function. The table below outlines the relevant settings, which are documented in the Boot Protocol:

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

These values are written to Guest memory starting at address `0x7000`. The `0x7000` address is also stored in RSI, a vCPU register, for reference during VM startup. For details on vCPU register setup, please refer to [Setup registers of vCPU](./02-4_setup_registers_of_vcpu).

## Setup E820

Configuring the E820 for the Guest OS allows reporting of available memory regions to the OS and BootLoader. The settings for this are aligned with the implementation in Firecracker. The following code illustrates how the E820 entries are added based on the Guest memory configuration:

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

It would be better to understand the design of the entire address space for the Guest VM, considering the code for starting a Guest VM in ToyVMM. Therefore, I'll list the current memory design for the Guest in the following table. Please note that this information may change in the future.

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

Upon examining the code, you can see that the address range that is designed independently of the GuestMemory size (roughly `0x0 ~ HIGH_MEMORY_START`) is commonly registered as "Usable" in the E820, ranging from `0` to `EBDA_START` (`0x9FBFF`).

Subsequently, the range registered in the E820 changes depending on how much GuestMemory is allocated. In the current implementation, the GuestMemory is set to reserve 128MB of memory by default, so the Guest Memory ranges from `0x0` to `0x7FF_FFFF`. In this range, `vmlnux.bin` content and `initrd.img` are mapped.

In other words, the logic `guest_mem.last_addr() = 0x7FF_FFFF < 0xD000_0000 = end_32bit_gap_start` applies, so the range `HIGH_MEMORY_START ~ guest_mem.last_addr()` is additionally registered. In the future, as you expand, if the GuestMemory size exceeds 4GB, you will register the ranges `0x10_0000 ~ 0xD000_0000` and `0x1_000_0000 ~ guest_mem.last_addr()`.

You will be able to confirm the console output when starting the VM shortly. Here, I've provided part of the output to show that the E820 entries you configured are registered:

```bash
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x0000000007ffffff] usable
```


## References

- [Linuxのブートシーケンスの基礎まとめ](https://nishidy.hatenablog.com/entry/2016/09/08/230637)
- [Linuxカーネルユーザ・管理者ガイド - 初期RAMdディスクを使用する](https://doc.kusakata.com/admin-guide/initrd.html)
- [initrd](https://manpages.ubuntu.com/manpages/bionic/ja/man4/initrd.4.html)
- [initramfs(initrd)のinitをbusyboxだけで書いてみた](https://www.gcd.org/blog/2007/09/129/)
- [initramfsとinitrdについて](https://blog.goo.ne.jp/pepolinux/e/4d1f4b6e0f5b5ed389f
