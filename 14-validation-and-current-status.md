<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Firmware Behavioral Requirements](13-firmware-behavioral-requirements.md) | [Index](README.md#reading-order-and-contents) | [Next: Open Questions & Gaps →](15-open-questions.md)

# Validation and hardware measurements

## Scope

This chapter records the hardware measurements obtained on the SpacemiT K3 reference platform, defines the validation criteria required to verify an independent LPDDR5 bootloader or PMU firmware implementation, and documents the bench safety constraints that must be respected during testing.

## Reference platform

| Property | Value |
|---|---|
| Board | SpacemiT K3 Pico-ITX |
| SoC | SpacemiT X100, 16 RISC-V cores (8 + 8), up to 2.2 GHz |
| DRAM | LPDDR5 `MT62F2G32D4DS`, 2 ranks, x16 (`x8_mode = 0`), 6400 MT/s, 16 GiB total (8 GiB × 2 controllers) |
| Operating system | Fedora Linux (kernel 6.18.3+), 16 GiB visible |
| Serial console | 115200 8N1 |
| Boot chain | BootROM → FSBL (SPL + DDR init) → OpenSBI / UEFI (EDK2) → GRUB → Linux |

## Observed boot and initialization timings

| Boot path | Measured duration | Observation method |
|---|---:|---|
| Persisted **quickboot** | **9 ms** | Loader console timestamp with valid `DDRT` envelope in SPI flash |
| **Full training** (both controllers) | **5611 ms** – **5803 ms** (~5.6 s) | Loader console timestamp after invalidating the `DDRT` envelope |
| Partial / abbreviated initialization (DQ sweep omitted) | **~5 ms** | Experimental PMU firmware run that completed `DevInit` and mailbox handshake without full DQ eye sweeps |
| Total cold boot to Linux login prompt | **~14 s** | Serial console wall-clock measurement (quickboot path) |

Quickboot is approximately **620× faster** than full training. Conversely, if a supposed full-training run completes in only a few milliseconds (such as ~5 ms), that duration is diagnostic evidence that heavy per-lane DQ/eye calibration loops were skipped, even if the PMU still emits major completion code `0x07` and the board boots into Linux under nominal room-temperature conditions.

## Reference training envelope and trace baselines

### Captured reference `DDRT` envelope (`MT62F2G32D4DS`, 6400 MT/s, 2 ranks)

| Field / Metric | Observed value (per controller) |
|---|---|
| Envelope `magic` | `0x54524444` (`"DDRT"`) |
| `type` | `1` (LPDDR5) |
| `cs_num` | `2` |
| `data_rate` | `6400` |
| `MsgMisc` | `0x01` |
| `HdtCtrl` | `0x04` (detailed debug threshold enabled in captured sample) |
| Non-zero `phypara` halfwords | `3926` / `5170` |
| Non-zero `acsm` halfwords | `1845` / `4096` |

### Captured full-training mailbox traffic (`HdtCtrl = 0x04`, two controllers)

- **22 major stage messages** across the two controllers (`0d 00 01 02 fd 06 01 02 fd 04 03 09 07` on controller 0);
- **9,153 streaming messages** (`0x08` frames);
- **124 distinct streaming message IDs**.

### Host loader payload capacity limits

When packaging PMU images into the host loader's decompression and transfer pipeline, the decompressed payloads must satisfy these size bounds:

| Image / Resource | Host DPHY Target | Capacity / Size Limit |
|---|---:|---:|
| Full-training PMU firmware (IMEM) | `0x50000` | Up to `65,536` bytes (`0x10000` bytes, 32,768 halfwords) |
| Quickboot PMU firmware (IMEM) | `0x50000` | Up to `26,624` bytes (`0x6800`-byte host decompression buffer) |
| Quickboot additional DMEM parameter table | `0x59622` | `1,420` bytes (710 halfwords, `0x58c` bytes) |

## Memory bandwidth and latency baseline

Single-thread memory benchmark (`membench`, 256 MiB working set, 3 repetitions, compiled with `-O2`) measured under Linux on the reference board:

| Metric | Full training (~5.6 s) | Abbreviated `DevInit`-only run (~5 ms) |
|---|---:|---:|
| Sequential read bandwidth | 8.516 GB/s | 8.599 GB/s |
| Sequential write bandwidth | 6.104 GB/s | 6.179 GB/s |
| Memory copy bandwidth | 3.191 GB/s | 3.250 GB/s |
| Pointer-chase latency (16 MiB) | 133.15 ns | 129.03 ns |
| Pointer-chase latency (64 MiB) | 166.80 ns | 167.07 ns |
| Pointer-chase latency (256 MiB) | 176.15 ns | 176.69 ns |

> **Important interpretation note:** Steady-state bandwidth and latency depend primarily on DDRC clock frequency and DRAM timing registers (`dramset1tmg*`, `dfitmg*`), not on analog PHY eye centering. Therefore, matching benchmark numbers (within ~2%) and booting Linux **do not prove** that PHY DQ training was executed or that adequate voltage/timing eye margins were achieved.

## Host-chain packaging vs. in-place binary patching during validation

When testing a locally built SPL/FSBL chain on the K3:

1. **Repackaged/signed `FSBL.bin` (`make` / `fsbl.json`) vs. in-place binary patching**: The U-Boot build (`tools/build_binary_file.py -c board/spacemit/k3/configs/fsbl.json`) wraps `spl/u-boot-spl.bin` with `AIHD` headers (`header0`, `header1`), CRC-32 checksums (`header_crc`, `img_crc`), and `SHA256+RSA2048` signatures (`signature0`, `signature1`) generated using the RSA-2048 private keys provided by SpacemiT in the open-source tree (`board/spacemit/k3/configs/key/root_key_prv.key` and `spl_key_prv.key`). Modifying bytes directly inside an already-built `FSBL.bin` without re-running `build_binary_file.py` invalidates `img_crc` and `signature1`, causing the BootROM to reject the image. Always rebuild `FSBL.bin` via `make` (or repackage `u-boot-spl.bin` with `build_binary_file.py`) so the headers, CRCs, and RSA signatures are regenerated.
2. **Isolating host smoke-test issues from PMU firmware**: Change only one subsystem at a time (either the host-side DDRC/loader code or the ARC PMU firmware image). Never claim full memory verification passed if the post-activation `0x4000`-byte host smoke test was bypassed.

## Seven-scenario hardware validation matrix

An independent implementation should be validated against the following seven hardware test scenarios before release:

| Scenario | Setup / Trigger | Expected Loader & PMU Behavior | Pass Criteria |
|---|---|---|---|
| **V1 — Minimal `DevInit` bring-up** | `SequenceCtrl = 0x0001`, `Quickboot = 0`, `HdtCtrl = 0xff` | PMU initializes PHY/DRAM without full DQ sweeps and emits `0x07` in a few milliseconds | Both controllers reach `stat[1:0] == 01`; result is **not** persisted to `bootinfo` |
| **V2 — Full training (cold)** | Erase/invalidate `DDRT` magic at `bootinfo+0x10000`; full `SequenceCtrl` stage mask, `Quickboot = 0` | Full 1D/2D training executes on DDR0 then DDR1 (~5.6 s total), emitting `0x07` on each | `CsTestFail == 0`, non-zero VREF/DFE/DCA and 4 margins per rank, 5170 `phypara` + 4096 ACSM collected, `0x4000` smoke test passes, `DDRT` saved |
| **V3 — Persisted quickboot (warm & cold)** | Reboot with valid `DDRT` envelope saved by V2 | Envelope magic/type/ranks/rate/CRC-32 accepted; `SequenceCtrl = 1`, `Quickboot = 1` path runs | Both controllers complete in ~9 ms, pass smoke test, and boot OS with 16 GiB visible |
| **V4 — Envelope mismatch / corruption fallback** | Flip 1 bit in saved `phypara` or change `data_rate`/`cs_num` in `bootinfo+0x10000` | Loader rejects envelope via header or CRC-32 check and falls back to V2 full training | Clean fallback to full training, new valid `DDRT` envelope written, subsequent reboot uses V3 quickboot |
| **V5 — PMU fatal error (`0x000000ff`) containment** | Force PMU error path (e.g. invalid `CsPresent` or injected `0xff` + `0x90040238` `4 -> 0` pulse) | Host detects `0x000000ff` without issuing normal ACK, stops PMU (`MICRO_RESET = 1`, `MUX = 0`), reads diagnostic `0x200c9` | Loader aborts cleanly, keeps host ports `0x20090`–`0x24090` disabled, and does **not** overwrite `bootinfo` |
| **V6 — Clock / mailbox timeout containment** | Induce DPLL lock timeout or stall PMU before `0x07` | Bounded host polling timer expires | Loader stops PMU, restores `MUX = 0`, prints last major/streaming ring buffer, leaves ports disabled |
| **V7 — OS stress and thermal/reboot stability** | 50+ cold/warm boot cycles plus multi-core Linux memory stress (`stress-ng --vm`, `memtester`, kernel build) | Consistent ~9 ms quickboot and zero corrected/uncorrected memory corruption | Zero hangs, zero bit flips, and stable bandwidth/latency (~8.5 GB/s read, ~6.1 GB/s write) |

## Validation criteria for an independent implementation

An implementation claiming full compliance with this specification must verify:

1. **Stage and work coverage**: Confirm that all stages enabled in `SequenceCtrl` execute to completion, with training duration, major stage transitions, and per-lane calibration sweeps consistent with full 1D/2D training rather than an early bypass.
2. **Trained output completeness**: Verify `CsTestFail == 0`, non-zero trained VREF/DFE/DCA/CDD outputs in the 320-byte message block, non-zero eye margins (`RxClkDly_Margin`, `VrefDac_Margin`, `TxDqDly_Margin`, `DeviceVref_Margin`), and populated `phypara` (5170 halfwords) and ACSM (4096 halfwords) snapshots on **both** DDR0 and DDR1.
3. **Post-activation memory verification**: Pass a non-destructive `0x4000`-byte address-pattern and complement smoke test across the mapped address ranges of both controllers before enabling normal boot continuation.
4. **Persistence and quickboot cycle**: Save the `0x9800`-byte `DDRT` envelope, reboot, verify that quickboot accepts the CRC-32 and restores DRAM in ~9 ms, and re-verify memory stability.
5. **Stress and margin validation**: Run multi-threaded memory stress tests across cold/warm reboots, voltage/temperature corners, and all supported topologies (`4800`/`6400` MT/s, 1/2 ranks, x8/x16).
6. **Error path containment**: Verify that clock lock timeouts, mailbox timeouts, and PMU fatal code `0x000000ff` are caught and reported explicitly without hanging silently or enabling DDRC ports on an uncalibrated interface.
7. **Clean production binary**: Confirm all diagnostic progress markers and verbose UART trace hooks are disabled, and verify via `objdump` that no `r16`–`r25` instructions or broken `libgcc` helpers exist in the final ARC ELF.

## Run-to-run repeatability, volatile CSRs, and failure signatures

### Measured run-to-run repeatability and acceptance tolerances

Comparing multiple independent cold-boot full-training runs on the reference 6400 MT/s dual-rank x16 platform shows that analog eye-centering sweeps have small natural run-to-run variation:

- **`CSR[0x01<r><l>4e]` and `CSR[0x01<r><l>4f]` read-eye centers**:
  - Controller 0: mean absolute difference **1.72 taps**, maximum **4 taps** (`L8` rank 0: `0x3e` vs. `0x42`); **50%** of lanes match within `±1` tap, **74%** within `±2` taps, and **89%** within `±3` taps.
  - Controller 1: mean absolute difference **2.36 taps**, maximum **7 taps** (`L5` rank 0: `0x2f` vs. `0x36`); **39%** of lanes match within `±1` tap, **62%** within `±2` taps, and **83%** within `±3` taps.
- **Practical comparison tolerance**: When validating an independent PMU firmware implementation against a reference `phypara` capture on the same board, require per-lane read/write eye centers (`0x4e`/`0x4f`, `0x10..0x13`, `0x68..0x6f`) to match within **`±3` taps on average** (and within **`±7` taps worst-case**), rather than demanding bit-for-bit equality on analog delay registers.

### Standalone validation criteria (without a reference capture)

When validating an independent trainer on a board without comparing against a prior `phypara` dump, verify all five standalone invariants:

1. **Execution duration**: Full training (`SequenceCtrl = 0x125f`) completes in the multi-second range (~4.4–5.8 s), never in a few milliseconds (`5–41 ms` indicates an early abort or skipped DQ sweeps).
2. **`DevInit` completion markers**: `CSR[0x020011]` (`ARC 0x90040022`) is masked from its `0x006f` pre-training value to `0x000f` (with `0x006f` saved at DMEM `0x8000042a`), `CSR[0x02000a]` holds the latched CA clock period (`0x0082`–`0x0086` at 6400 MT/s, updated away from its `0x0040` pre-training seed after the bit 10 falling-edge strobe on `CSR[0x0200e3]`, while `CSR[0x020017]` remains `0x0040`), `CSR[0x020012]`/`0x020186`/`0x020187` hold `0x7353`, `0x0073`, and `0x0041` (updated from `0xa1a1`, `0x00a1`, and `0x0040`), `CSR[0x01<r>0d9]` holds `0x009c` (`156`) across all active ranks, and `CSR[0x0c0086]` transitions to `3` after `DevInit` (or `2` after full training). Note that `CSR[0x01<r>0a3] == 0x0b33` is written by the host pre-training table *before* the PMU starts and therefore does not by itself prove `DevInit` executed.
3. **Trained read-eye centers**: Every active lane (`4 ranks × 9 lanes`) has `CSR[0x01<r><l>4e]` and `0x4f` updated away from the `0x0035` pre-training seed (with `0x50`/`0x51` remaining `0x0035` when `DisableTrainingLoop[1] = 0`).
4. **2D eye opening** (when contour streaming is enabled): Horizontal width is in the **70–124 tap** range (median ~102 taps) and vertical height is **21–32 voltage levels**.
5. **Self-repeatability**: Two consecutive cold-boot training runs of the new firmware agree within **`<= ±3` taps** across all 36 lanes (excluding the volatile CSRs below).

### Volatile / hardware-counter CSRs to exclude from bit-exact `phypara` diffs

Inside the 5,170-entry `phypara` dump, the live period-counter register and hardware sequencer/status registers can vary slightly across runs or between `DevInit`-only (`SequenceCtrl = 0x0001`) and full-training (`0x125f`) captures:

| CSR index | ARC address | Reference pre → post (`0x125f` / `0x0001`) | Reason |
|---:|---:|---|---|
| `0x02000a` | `0x90040014` | `0x0040` → `0x0082` (`130`) / `0x0086` (`134`) | Live CA period counter (`CSR[0x0300bf] & 0x01ff`) latched by the `CSR[0x0200e3]` bit 10 falling edge during `DevInit` (`±1..4` tick variation) |
| `0x020012` | `0x90040024` | `0xa1a1` → `0x7353` | Deterministic frequency bucket computed from `DRAMFreq` (`6400`) and `DfiFreqRatio` (`4`) |
| `0x020186` | `0x9004030c` | `0x00a1` → `0x0073` | Secondary frequency bucket `(bucket3 << 4) \| fine_hi` computed from `DRAMFreq` and `DfiFreqRatio` |
| `0x020187` | `0x9004030e` | `0x0040` → `0x0041` | Copied unmasked from rank-0 `CSR[0x0100d5]` (`ARC 0x900201aa`) during `DevInit` |
| `0x09080b` | `0x90121016` | `0x002f` → `0x000f` (or `0x0015`/`0x00ff`) | ACSM/sequencer hardware state |
| `0x09080c` | `0x90121018` | `0x006f` → `0x000f` (or `0x002d`/`0x0065`) | ACSM/sequencer hardware state |
| `0x0c0086` | `0x9018010c` | `0x0000` → `0x0002` (`0x125f`) / `0x0003` (`0x0001`) | Sticky PHY/microcontroller status bits (`1` during PMU execution, `3` after `DevInit`, `2` after full training) |

### Diagnostic failure signatures

When bringing up new PMU firmware or host table overrides, use the following symptom table to isolate the root cause quickly:

| Observed symptom | Typical duration | Root cause |
|---|---:|---|
| PMU hangs silently (no `0x07` or `0xff`; USB FDL gadget wedged on reset) | Timeout | Instruction referenced `r16`–`r25` (missing `-mrf16` at compile or link), unaligned 32-bit `ld` on a 6-byte table stride, 32-bit `ld`/`st` on a 16-bit `0x90xxxxxx` CSR, AUX `0x103`–`0x105` accessed as memory instead of `lr`/`sr`, or `libgcc` `__muldi3`/`__udivsi3` invoked |
| PMU emits fatal `0x000000ff` early (`DDR training failed!`) | **12–41 ms** | Invalid electrical configuration (`TX DS >= 60 Ω` or `OFF`), `.bss` linked at `0x80000400` (wiping host-supplied `phypara` / runtime state during startup), or `CsPresent`/`EnabledDQs` mismatch |
| PMU emits `0x07` far too fast (`DDR training consume 5 ms`–`28 ms`) | **5–28 ms** | 32-bit store to `0x80000458` clobbered delay divisor `0x8000045a`, or `SequenceCtrl` stage mask lacked `RdDQS` (`bit 3`) / `WrDq` (`bit 4`) / `DRAM DCA` (`bit 6`) |
| Late host message-block override (`SequenceCtrl`, `HdtCtrl`, `Train2DMisc`) has no effect | Normal (~5.6 s) | Host wrote `DPHY_BASE + 4 * 0x58000` *after* the pre-training phase closed (immediately before `pwrctl` bit 11); hardware discards those writes |
| Mailbox streaming takes 5–7 minutes with `Train2DMisc[0] = 1` | **>300 s** | Host receiver printed a fixed 15/16-word buffer per frame over 115200 bps UART instead of printing only `len + 1` words |
| `CSR[0x01<r><l>4e]` stays at `0x0035` after training | Variable | `RdDQS` (`SequenceCtrl[3]`) did not run, or `DisableTrainingLoop[1]` (`0x80000102 & 2`) was inadvertently set, averaging `0x4e` with `0x50` (`0x0035`) |
| `CSR[0x0300bf]` and `CSR[0x0100e0]` read `0x0240` (`CSR[0x02000a]` stays at `0x0040`) or `CSR[0x020011]` stays at `0x006f` | ~15 ms+ | Period-counter strobe omitted the **falling edge on bit 10 of `CSR[0x0200e3]`** (`0x0800 -> 0x0c00 -> delay(10) -> 0x0800` after `CSR[0x0200e5] &= ~0x001f`), leaving `0x0300bf & 0x01ff` at the unlatched pre-training value `0x0040` instead of `0x0082`–`0x0086`, or missing `volatile` on `0x9004xxxx` accesses caused the compiler to optimize out the strobe |

## Safe bench observation policy

> **DANGER — Do not access PHY/PMU MMIO from Linux (`/dev/mem`):**
> On the reference board, reading the DPHY/PMU region from user space after Linux booted immediately stalled the SoC interconnect—causing total loss of serial console and network and requiring a physical power-plug disconnect. (An early attempt also used the wrong base `0xc0960000`, which is unmapped on K3 where Instance 0 DPHY is at `0xcb800000`, but the prohibition applies equally to the true `0xcb800000`/`0xcc800000` windows once the OS is running.)
>
> All DPHY, IMEM, DMEM, and PMU CSR reads/writes must be performed **inside the first-stage bootloader (SPL)** while the DPHY clocks and PMU power domain are active and `MICRO_CONT_MUX_SEL` (`0xd0000`) is set appropriately, or via JTAG.

Additional bench precautions:

- **Live DMEM message zeroing**: Reading `0x58000` directly after the PMU phase may return zeros because the message area is consumed/cleared; always inspect the 320-byte copy captured by the result-collection sequence.
- **UART baud-rate bottleneck**: Setting `HdtCtrl = 0x04` or `0x05` and printing all 9,153 streaming messages at 115200 bps emits ~1.4 MiB of text and adds tens of seconds to boot time. Use verbose streaming only during targeted debugging.
- **Post-experiment cleanup**: Always restore the known-good `FSBL.bin` and `bootinfo` `DDRT` envelope after bench experiments and verify that quickboot returns to ~9 ms.

## Reproducing the baseline measurements

1. **Quickboot timing**: Boot normally with a valid `DDRT` envelope in `bootinfo` (`+0x10000`) and record the loader's reported DDR init duration (`~9 ms`).
2. **Full-training timing**: Erase or invalidate the first 4 KiB sector of the `DDRT` envelope at `bootinfo+0x10000`, reboot, and record the full-training duration (`~5.6 s`).
3. **Mailbox trace**: Boot an instrumented loader with `HdtCtrl = 0x04` and an invalidated envelope to capture the major and streaming message sequence over UART.
4. **Envelope inspection**: Read back the `0x9800` bytes at `bootinfo+0x10000` and parse the header and two `0x49a4`-byte controller records according to [10-persistence-and-selection.md](10-persistence-and-selection.md).

## Related chapters

- [06-full-initialization.md](06-full-initialization.md)
- [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md)
- [09-training-results.md](09-training-results.md)
- [10-persistence-and-selection.md](10-persistence-and-selection.md)
- [16-flash-build-and-recovery.md](16-flash-build-and-recovery.md)
