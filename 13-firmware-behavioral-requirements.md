<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: ARC Execution Environment](12-arc-execution-environment.md) | [Index](README.md#reading-order-and-contents) | [Next: Validation & Measurements →](14-validation-and-current-status.md)

# Firmware behavioral requirements

## Purpose and scope

These normative requirements specify the processor, memory, peripheral, timing, and toolchain rules that any LPDDR5 PMU firmware implementation must satisfy to execute reliably on the SpacemiT K3 DDR PHY microcontroller. They apply to the ARC/PMU firmware image, not to host-side DDR controller programming.

In this document, **must** denotes a mandatory hardware or ABI requirement verified on the K3 platform. All addresses are in the PMU/ARC address space unless explicitly identified as a host CSR index.

## BR-01 — Target architecture, entry point, and reduced register file

- The target processor is a little-endian **ARCv2/EM** core.
- The image reset vector at offset `0x0` must branch to the effective startup entry at **`0x1fc`**. Hardware execution requires the `0x1fc` entry layout; linking a flat entry directly at `0x0` without the `0x1fc` startup sequence does not complete initialization.
- Startup must establish the stack top at **`sp = 0x80010000`**. For full-training firmware, the host-provided parameter table base is **`0x80000400`**.
- The K3 PMU core implements the 16-register reduced register file (`RF16`) and **does not implement general registers `r16`–`r25`**. Generated machine code must never read or write `r16`–`r25`.
- Both compilation and final linking must pass **`-mrf16`**. When Link-Time Optimization (`-flto`) is used, passing `-mrf16` only during compilation is insufficient because the linker (`LTRANS`) regenerates code using the full 32-register file unless `-mrf16` is also present on the link command line.
- If the compiler uses `gp` (`r26`) for its own small-data section, that pointer must be kept separate from the `0x80000400` parameter-table pointer so writes are never redirected into IMEM or invalid regions (passing **`-mno-sdata`** disables compiler `.sdata`/`.sbss` generation via `gp`).

## BR-02 — Startup memory placement and `.bss` isolation

- Firmware startup must initialize only memory regions owned by the firmware's own static variables. It must never clear host-supplied input state.
- Firmware `.bss` **must not overlap `0x80000400`**. That region holds the `phypara` and electrical configuration written by the host SPL (as well as runtime state words such as `0x8000042a`, `0x8000042c`, `0x80000430`–`0x80000437`, and the delay divisor at `0x80000458`/`0x8000045a`). Zeroing `0x80000400` during startup erases the input configuration and causes training to abort.
- Place firmware `.bss` and internal uninitialized globals at **`0x80004000`** (or higher unused DMEM below the stack and message scratch area).
- The compiled binary must fit both the 64 KiB PMU IMEM (`0x50000`) and the host loader's decompression buffer limit for its target boot mode (`0x10000` bytes for full training, `0x6800` bytes for quickboot).

## BR-03 — Peripheral/DMEM access widths, `volatile` qualification, and unaligned table strides

- Every DMEM (`0x8000xxxx`) and PMU/PHY peripheral CSR (`0x90xxxxxx`) access must be **`volatile`-qualified** and use the exact hardware width required by the register:
  - Standard PHY CSRs, mailbox registers, and 16-bit DMEM fields must be accessed with **16-bit halfword instructions (`ldh`/`sth`, `volatile uint16_t`)**. Using 32-bit accesses (`ld`/`st`, `uint32_t`) on `0x8000xxxx` or `0x90xxxxxx` reads zero, clobbers adjacent 16-bit registers, or stalls the internal peripheral bus.
  - Omitting `volatile` on `0x8000xxxx` or `0x90xxxxxx` pointers allows compiler optimizations to hoist polling reads out of loops or eliminate intermediate register writes (such as clearing/restoring `CSR[0x020011]` bits `[6:5]` around period measurement).
  - Single-byte message-block fields must be accessed with **8-bit byte instructions (`ldb`/`stb`, `volatile uint8_t`)** respecting little-endian byte order within the containing halfword.
- Packed tables whose element stride is **not a multiple of 4 bytes** (for example, 6-byte descriptor entries) must be accessed **byte-by-byte (`ldb`, `uint8_t *`)** and assembled explicitly in registers. A 32-bit dereference at an unaligned offset such as `base + 6` faults and hangs the ARC core.
- Pointer arithmetic and value extensions must preserve the exact byte stride and signed/unsigned semantics of each field.

## BR-04 — Calling convention and stage-enable gating

- Standard ARCv2 RF16 function calls pass arguments in **`r0`–`r7`** and return results in **`r0`**, saving return addresses in **`blink` (`r31`)** and preserving callee-saved registers within the reduced `r0`–`r15` set (`r13`–`r15`, `fp`, `gp`, `sp`).
- Conditional calibration and DQ-training stages must evaluate their hardware enable flags in the message block (`SequenceCtrl`, `CsPresentChA/B`, `EnabledDQsChA/B`, `X8Mode`, and `Quickboot`). Unconditionally skipping an enabled calibration stage leaves the PHY uncalibrated even if the firmware still emits major completion code `0x07`.

## BR-05 — PHY CSR addressing and hardware handshake sequences

- When forming a 16-bit ARC peripheral address (`0x90xxxxxx`) from a 16-bit PHY CSR index and channel/rank/lane selector bits, combine the base index and selector bits using **bitwise OR (`|`)** and double the result:
  ```text
  ARC_CSR_address = 0x90000000 | (2 * (csr_index | block_or_rank_bits))
  ```
- Descriptor and command-sequence transfers must use their dedicated hardware windows (`0x90000000 + 2 * idx` for read-side descriptors, `0x9001e000 + 2 * idx` for write-side descriptors, and `0x90082000 + 2 * idx` for command sequences).
- Hardware control pulse registers must observe their required settling delays:
  - Engine start/poll sequence: configure `0x90040040`/`0x90040042` and poll status at `0x90040062`.
  - Initialization/finalization pulses at `0x90040146` and `0x9005e140`: write `1`, wait at least `0x20` timer units, and write `0`.
  - Terminal status pulse at `0x90040238`: pulse `1 -> 0` on normal completion (`0x07`), or `4 -> 0` on fatal error (`0x000000ff`).
- Never write to unmapped offsets in `0x90xxxxxx`; invalid peripheral accesses stall the internal PHY bus.

## BR-06 — Hardware RTC timer accesses (`0x103`–`0x105`)

- The PMU hardware timer resides in ARC auxiliary registers **`0x103` (RTC control)**, **`0x104` (RTC count low)**, and **`0x105` (RTC count high)**.
- Firmware must access `0x103`–`0x105` exclusively with ARC **`lr` and `sr` instructions** (`__builtin_arc_lr` / `__builtin_arc_sr` or inline assembly). Treating these numbers as memory addresses causes timer polling loops to hang forever.
- To time a delay:
  1. Check AUX `0x103`; if the counter is not running, initialize and enable it (`sr 2, [0x103]` followed by `sr 1, [0x103]`).
  2. Sample the 64-bit counter (`lr` from `0x104` and `0x105`), compute the 64-bit expiration target, and poll until the 64-bit counter reaches the target while properly accounting for 32-bit low-word rollover into `0x105`.
  3. For short delays where the scaled tick count is `< 16`, perform a dummy 16-bit `volatile` read from `0x90040048` (`CSR[0x020024]`) per iteration to pace the loop on the peripheral bus.

## BR-07 — Hardware multiply/divide instructions and `libgcc` overrides

- Standard `arc-elf32` `libgcc` implementations of 64-bit multiplication (`__muldi3` / `__umulsi3_highpart`) and 32-bit integer division (`__udivsi3` / `__divsi3` / `__udivmodsi4`) hang on the K3 PMU core.
- Firmware must compile with **`-mmpy-option=wlh5 -mdiv-rem`** and use the hardware **`mpymu`** (unsigned 32×32 high-part multiply), **`mpyu`/`mpy`**, and **`divu`/`divs`** instructions:
  - Upper 32-bit products (`(uint64_t)a * b >> 32`) must emit `mpymu`.
  - Unsigned and signed 32-bit divisions must emit `divu` and `divs`.
- To prevent the compiler from emitting calls to broken `libgcc` helpers anywhere in the image, override **`__muldi3`, `__udivsi3`, and `__divsi3` globally** using `mpymu`/`mpyu`/`divu`.
- Compile the translation unit containing these global libcall overrides **without `-flto`** (while keeping `-mrf16`) so that LTO linking cannot discard or inline-bypass the overrides.
- Inspect the final linked ELF disassembly to verify that no `libgcc` `__umulsi3_highpart` or `__udivmodsi4` bodies remain and no `r16`–`r25` registers appear.

## BR-08 — Transient scratch isolation (`0x8000e000`–`0x8000e0ff`)

- The DMEM region **`0x8000e000`–`0x8000e0ff`** (host DPHY `0x5f000`–`0x5f07f`) is a transient scratch area read and cleared by the host SPL during execution; firmware must not store persistent state there.
- Any optional bring-up diagnostics in DMEM must be placed in unused memory above `.bss` (`0x80004000`) and below the stack (`0x80010000`), must not overlap `phypara` (`0x80000400`) or the message block (`0x80000000`–`0x8000013f`), and must be disabled in production builds.

## BR-09 — Mailbox handshake and 16-bit protocol compliance

- All PMU accesses to the mailbox registers—status **`0x90180008`** (`CSR[0x0c0004]`), low data/command **`0x90180064`** (`CSR[0x0c0032]`), doorbell/ACK **`0x90180066`** (`CSR[0x0c0033]`), and high data **`0x90180068`** (`CSR[0x0c0034]`)—must use **16-bit halfword operations (`ldh`/`sth`)**.
- Sending a 16-bit major message (or streaming introducer/count word) requires the exact 5-step sequence:
  1. Writing the 16-bit code to `0x90180064` (and high 16-bit halfword to `0x90180068` when sending a 32-bit word);
  2. Writing `0` to `0x90180066` (clear doorbell);
  3. Polling `0x90180008` bit 0 until `0` (mailbox ready);
  4. Writing `1` to `0x90180066` (assert doorbell);
  5. Polling `0x90180008` bit 0 until `1` (host acknowledged).
- **Do not clear `0x90180066` to `0` after step 5**: `0x90180066 = 0` belongs at step 2 of the next word transfer. Clearing `0x90180066` to `0` immediately after step 5 leaves `0x90180008` bit 0 at `1` while the doorbell is `0`, which can desynchronize the next transfer.
- Sending a streaming message (`0x08`) requires first sending command `0x08` with its handshake, then the count word `len`, then the 32-bit header `(id << 16) | len`, and finally each of the `len` 32-bit payload words as `(low_16 -> 0x90180064, high_16 -> 0x90180068)` with the full 5-step doorbell/ACK handshake per word.

## BR-10 — Verification invariants before hardware deployment

Before loading or flashing a newly built PMU firmware image, verify:

1. **Disassembly audit**: `arc-elf32-objdump -d` shows zero references to `r16`–`r25` and no unoverridden `libgcc` division/64-bit multiplication routines.
2. **Entry layout**: offset `0x0` branches to startup at `0x1fc`.
3. **Section placement**: `.bss` starts at or above `0x80004000` and does not touch `0x80000000`–`0x80003fff`.
4. **Image size**: binary size is within the 64 KiB IMEM limit (and `<= 0x6800` bytes if targeting quickboot).
5. **Runtime completeness**: full training executes all enabled `SequenceCtrl` stages, emits major `0x07`, leaves `CsTestFail == 0`, populates all 5170 `phypara` halfwords and 4096 ACSM halfwords, and passes post-activation memory verification.

## Related chapters

- [02-addressing-and-access.md](02-addressing-and-access.md)
- [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md)
- [12-arc-execution-environment.md](12-arc-execution-environment.md)
- [14-validation-and-current-status.md](14-validation-and-current-status.md)
- [16-flash-build-and-recovery.md](16-flash-build-and-recovery.md)
