<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Open Questions & Gaps](15-open-questions.md) | [Index](README.md#reading-order-and-contents) | [Next: Appendix: Message Block Fields →](appendix-message-block-fields.md)

# Flash, build, and recovery

## Scope and safety

This chapter documents the build, flashing, and recovery procedures for testing first-stage bootloader (SPL/FSBL) and PMU firmware images on the SpacemiT K3 Pico-ITX platform.

Before modifying SPI NOR flash:

- Maintain a verified full SPI NOR backup (`bootinfo`, `fsbl`, `env`, and the persisted `DDRT` training envelope at `bootinfo+0x10000`).
- Record exact artifact SHA-256 hashes, compiler versions, build flags, and configuration parameters for every test run.
- Connect a 115200 8N1 serial console and ensure physical access to disconnect and reconnect board power.
- **Preserve the UEFI/EDK2 partition**: on K3 boards using a UEFI boot flow, the SPI partition named `uboot` stores the EDK2 firmware volume (`fit`), not a standalone U-Boot payload. Overwriting `uboot` with `u-boot.itb` breaks the UEFI boot chain.
- Ensure a working USB Firmware Download (**FDL** / fastboot) recovery path before erasing or replacing `fsbl`.

> **DANGER — Never access PHY/PMU MMIO from Linux (`/dev/mem`):**
> Reading or writing DPHY/PMU MMIO registers after the OS boots stalls the SoC interconnect and requires a physical power cycle. Perform all PHY/PMU inspection from the bootloader while the PMU is powered, or via JTAG.

---

## `FSBL.bin` packaging, CRC, and RSA-2048 signing

Every `FSBL.bin` image on the K3—both the factory image and images built from the open-source U-Boot tree—is packaged by `tools/build_binary_file.py` using `board/spacemit/k3/configs/fsbl.json`:

### 1. Structure and SpacemiT-provided signing keys

`fsbl.json` constructs `FSBL.bin` with the following modules:

- **`ROTPK` (`0x000`–`0x0ff`)**: 256-byte RSA-2048 root public key extracted from `board/spacemit/k3/configs/key/root_key_prv.key`.
- **`image_header` (`header0` + `crc0` at `0x100`)**: `magic = "AIHD"`, `version = 1`, `secure = 0`, `load_addr = 0xc0800000`, and `header_crc = crc32(header0)`.
- **`image_config` + `oem_pubkey`**: key table, `rotpk_hash = SHA256(root_key)`, and the 256-byte RSA-2048 SPL public key extracted from `key/spl_key_prv.key`.
- **`cert0` (`signature0`)**: `SHA256+RSA2048` signature over `header0 + crc0 + keydata + oem_key`, signed with `key/root_key_prv.key`.
- **`spl` (`header1` + `crc1` + `u-boot-spl.bin` + `signature1`)**: second `AIHD` header (`imgsize = sizeof(fsbl)`), `header_crc = crc32(header1)`, **`img_crc = crc32(fsbl)`**, the `u-boot-spl.bin` payload, and **`signature1`** (`SHA256+RSA2048` over `header1 + crc1 + fsbl`, signed with `key/spl_key_prv.key`).

Because SpacemiT includes the private keys (`root_key_prv.key` and `spl_key_prv.key`) directly in `board/spacemit/k3/configs/key/` and `secure = 0` is set in the header (with `CONFIG_SPACEMIT_SECURE_BOARD` and `CONFIG_FIT_SIGNATURE` disabled for subsequent FIT stages), any `FSBL.bin` built from source via `make` is automatically signed and accepted by the K3 BootROM.

### 2. Why in-place binary patching of `FSBL.bin` fails

Modifying bytes directly inside an already-packaged `FSBL.bin` (for example, replacing an embedded PMU firmware payload in-place with a hex editor, even if compressed and decompressed sizes match) invalidates **`crc1.img_crc`** and **`signature1`**, causing the BootROM to reject the image. Always rebuild `FSBL.bin` through `make` (or re-run `tools/build_binary_file.py -c board/spacemit/k3/configs/fsbl.json -o FSBL.bin`) after changing SPL code or embedded PMU blobs. Keep an unmodified copy of a known-good `FSBL.bin` as a recovery artifact.

---

## Building the SPL/FSBL chain from source

From the bootloader source tree root, build in-tree using a 64-bit RISC-V GNU toolchain:

```sh
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- k3_defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j"$(nproc)"
```

Key generated artifacts:

- `FSBL.bin` — first-stage bootloader containing SPL and embedded DDR/PMU initialization payloads;
- `bootinfo_spinor.bin` — SPI NOR boot header;
- `u-boot-env-default.bin` — default environment image;
- `u-boot.itb` — later-stage FIT image (do **not** flash over the EDK2 `uboot` partition on UEFI systems).

---

## Building independent ARC/PMU firmware safely

When compiling independent PMU firmware for the embedded ARCv2 EM core, enforce all constraints specified in [12-arc-execution-environment.md](12-arc-execution-environment.md) and [13-firmware-behavioral-requirements.md](13-firmware-behavioral-requirements.md):

1. **Target, hardware multiply/divide, and reduced register file**: Compile and link with `-mcpu=em -mrf16 -mmpy-option=wlh5 -mdiv-rem -mno-sdata -mlittle-endian`.
2. **Pass `-mrf16` at both compile and link stages**: When `-flto` is enabled, LTRANS regenerates machine code at link time and will emit illegal `r16`–`r25` instructions unless `-mrf16` is present on the linker command line.
3. **Startup and linker layout**:
   - Place the reset branch at `0x0` targeting effective startup at **`0x1fc`**.
   - Set initial stack top to `0x80010000`.
   - Keep `.bss` at **`0x80004000`** so startup zeroing never overwrites host-supplied `phypara` data at `0x80000400` (including ODT settings at `0x80000424`).
4. **Hardware arithmetic libcall overrides**: Compile the translation unit that overrides `__muldi3`, `__udivsi3`, and `__divsi3` (using hardware `mpymu`/`mpyu`/`divu` instructions) **without `-flto`** so LTO cannot discard the global replacements.
5. **Post-link verification**: Audit the linked ELF before packaging:
   ```sh
   arc-elf32-objdump -d firmware.elf | grep -E '\br(1[6-9]|2[0-5])\b'
   ```
   This check must return zero instruction matches. Also confirm the binary fits within the 64 KiB IMEM window (`<= 0x10000` bytes for full training, `<= 0x6800` bytes for quickboot decompression).

Example toolchain invocation pattern:

```sh
ARC_CC=arc-elf32-gcc

# Compile firmware sources with -mrf16, hardware multiply/divide, and -mno-sdata
"$ARC_CC" -std=gnu23 -mcpu=em -mrf16 -mmpy-option=wlh5 -mdiv-rem -mno-sdata \
  -mlittle-endian -Os -flto \
  -ffreestanding -fno-builtin -ffunction-sections -fdata-sections \
  -c firmware_main.c -o firmware_main.o

# Compile global arithmetic libcall overrides WITHOUT -flto
"$ARC_CC" -std=gnu23 -mcpu=em -mrf16 -mmpy-option=wlh5 -mdiv-rem -mno-sdata \
  -mlittle-endian -Os \
  -ffreestanding -fno-builtin \
  -c pmu_libcalls.c -o pmu_libcalls.o

# Link with -mrf16 explicitly passed to the linker driver
"$ARC_CC" -mcpu=em -mrf16 -mmpy-option=wlh5 -mdiv-rem -mno-sdata \
  -flto -nostdlib -static \
  -Wl,--gc-sections -Wl,--no-warn-mismatch \
  -Wl,-T,pmu_firmware.ld -o firmware.elf startup.o firmware_main.o pmu_libcalls.o -lgcc
```

---

## SPI NOR flash layout and update procedure

The reference board uses an 8 MiB Winbond SPI NOR flash (JEDEC `0xef 0x6017`, 4 KiB erase size) divided into six partitions:

| Partition | SPI range | Size | Role |
|---|---|---:|---|
| `bootinfo` | `0x000000`–`0x020000` | 128 KiB | BootROM header; stores the `0x9800`-byte `DDRT` training envelope at offset **`+0x10000`** |
| `fsbl` | `0x020000`–`0x0a0000` | 512 KiB | First-stage bootloader (`FSBL.bin`: SPL + LZOP-compressed PMU images) |
| `env` | `0x0a0000`–`0x0b0000` | 64 KiB | Bootloader environment |
| `esos` | `0x0b0000`–`0x1b0000` | 1 MiB | Real-time co-processor firmware (`esos.itb`) |
| `opensbi` | `0x1b0000`–`0x210000` | 384 KiB | RISC-V OpenSBI runtime (`fw_dynamic.itb`) |
| `uboot` | `0x210000`–`0x800000` | ~5.9 MiB | EDK2/UEFI FIT image on UEFI systems — **do not overwrite with `u-boot.itb`** |

Always verify partition names via `/proc/mtd` rather than assuming fixed `/dev/mtdX` indices across different kernel configurations.

### Flashing `FSBL.bin` from Linux

When updating `fsbl` from a running Linux system:

```sh
flash_erase /dev/mtdX 0 0
mtd_debug write /dev/mtdX 0 <fsbl-byte-length> FSBL.bin
```

- Flash `FSBL.bin` only to the `fsbl` partition.
- Do not touch `bootinfo` unless intentionally invalidating or restoring the `DDRT` training envelope at `bootinfo+0x10000`.
- Erasing the first 4 KiB sector at `bootinfo+0x10000` invalidates `magic` (`0x54524444`) and forces full training on the next boot.

---

## FDL staging and recovery procedures

### Non-destructive RAM staging via FDL (recommended for testing)

When the `fsbl` partition in SPI NOR is erased, the BootROM automatically enters USB Firmware Download (**FDL**) mode on power-up. This allows staging and executing a candidate `FSBL.bin` directly in RAM without wearing or risking SPI flash:

```sh
fastboot stage FSBL.bin
fastboot continue
```

*(Select the specific USB device path with `-s usb:...` if multiple fastboot devices are attached.)*

### Recovering from a PMU or bus hang

If a candidate firmware image hangs the PMU or stalls the DPHY/APB bus:

1. **Power-cycle the board**: After a PMU hang, the USB DFU/FDL gadget and internal PHY state remain wedged; toggling UART DTR/RTS or issuing a warm reset does not recover the core. Physically disconnect power, wait for capacitors to discharge, and reconnect power.
2. **Enter FDL**: With `fsbl` erased (or FDL straps active), allow the board to enumerate in fastboot/FDL mode.
3. **Stage a known-good image**:
   ```sh
   fastboot stage known_good_FSBL.bin
   fastboot continue
   ```
4. **Restore SPI flash**: Note that in the standard BootROM/U-Boot recovery environment, `fastboot flash fsbl` is a no-op for SPI NOR. To write SPI NOR permanently during recovery, either boot into Linux via a staged `FSBL.bin` and use `flash_erase`/`mtd_debug`, or stage U-Boot and use U-Boot `sf` / `mtd` commands against the staged download buffer.

---

## Post-test validation and rollback checklist

After every hardware test run:

1. Capture the complete 115200 8N1 serial log, recording the loader banner, training mode (full training vs. quickboot), reported execution time, and detected DRAM capacity/rate.
2. Verify that no mailbox timeout or PMU error (`0xff`) occurred.
3. Record whether the post-activation host memory smoke test was enabled and passed.
4. When testing is complete, restore the known-good `FSBL.bin` and valid `DDRT` envelope in `bootinfo+0x10000`, perform a cold boot, and confirm that quickboot completes in **~9 ms**.

## Related chapters

- [10-persistence-and-selection.md](10-persistence-and-selection.md)
- [12-arc-execution-environment.md](12-arc-execution-environment.md)
- [13-firmware-behavioral-requirements.md](13-firmware-behavioral-requirements.md)
- [14-validation-and-current-status.md](14-validation-and-current-status.md)
