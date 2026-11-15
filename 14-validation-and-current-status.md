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

### Controlled thermal (`H1`–`H3`) vs. forced-air ambient (`C1`–`C3`) FDL runs and boot-path baseline shift (`CSR[0x020011]`)

To isolate pure thermal drift from boot-path differences, seven full-training captures at 6400 MT/s (`SequenceCtrl = 0x125f`) were compared on the reference board:
- **`SPI` (cold SPI flash boot)**: The persisted `training_envelope.bin` captured during normal cold boot from SPI flash;
- **`C1`, `C2`, `C3` (forced-air ambient USB/FDL boot)**: Three consecutive runs staged over USB `fastboot stage FSBL.bin` + `fastboot continue` (resetting via watchdog between runs) under continuous forced-air fan cooling at ambient room temperature;
- **`H1`, `H2`, `H3` (heated USB/FDL boot)**: Three consecutive runs using the exact same USB/FDL staging and watchdog reset loop with external thermal heating applied to the SoC/DRAM packages.

| Run | Condition & Boot Path | Duration (ms) | `CSR[0x020011]` (`DDR0`/`1`) | `CSR[0x02000a]` (`DDR0`/`1`) | `CSR[0x020187]` (`DDR0`/`1`) | `0x4e/0x4f` median (`DDR0`/`1`, taps) | `0x4e/0x4f` min–max (`DDR0` / `DDR1`, taps) | `TrainedVREFCA_A0` (`DDR0`/`1`) | `TrainedVREFDQ_A0` (`DDR0`/`1`) | `RxClkDly_Margin_A0` (`DDR0`/`1`) |
|---|---|---:|---|---|---|---:|---:|---|---|---|
| **`SPI`** | Cold boot (SPI flash) | `5,611` | `0x000f` / `0x000f` | `130` / `127` | `65` / `66` | **`62.5` / `62.0`** | `45–74` / `54–73` | `0x27` / `0x27` | `0x26` / `0x26` | `0x14` / `0x16` |
| **`C1`** | Ambient fan (USB FDL) | `5,717` | `0x006f` / `0x006f` | `129` / `131` | `64` / `66` | **`81.5` / `80.5`** | `65–95` / `70–91` | `0x3f` / `0x3f` | `0x26` / `0x27` | `0x16` / `0x16` |
| **`C2`** | Ambient fan (USB FDL) | `5,719` | `0x006f` / `0x006f` | `130` / `128` | `65` / `65` | **`82.0` / `81.0`** | `68–95` / `69–89` | `0x3f` / `0x3f` | `0x26` / `0x26` | `0x18` / `0x16` |
| **`C3`** | Ambient fan (USB FDL) | `5,718` | `0x006f` / `0x006f` | `129` / `129` | `65` / `65` | **`82.0` / `80.5`** | `68–93` / `69–91` | `0x3f` / `0x3f` | `0x26` / `0x27` | `0x17` / `0x16` |
| **`H1`** | Heated (USB FDL) | `5,720` | `0x006f` / `0x006f` | `130` / `130` | `65` / `65` | **`81.5` / `81.5`** | `64–95` / `68–94` | `0x3f` / `0x3f` | `0x26` / `0x26` | `0x17` / `0x16` |
| **`H2`** | Heated (USB FDL) | `5,717` | `0x006f` / `0x006f` | `126` / `129` | `63` / `65` | **`81.5` / `81.0`** | `67–94` / `68–94` | `0x3f` / `0x3f` | `0x26` / `0x26` | `0x17` / `0x16` |
| **`H3`** | Heated (USB FDL) | `5,717` | `0x006f` / `0x006f` | `129` / `128` | `65` / `64` | **`81.5` / `82.0`** | `66–95` / `69–91` | `0x3f` / `0x3f` | `0x26` / `0x25` | `0x16` / `0x17` |

Three hardware conclusions follow directly from this controlled comparison:

1. **Pure thermal drift (`H1`–`H3` vs. `C1`–`C3` under the same USB/FDL boot path) is `<= 1 tap`**:
   - Across all 72 `0x4e`/`0x4f` registers on `DDR0`, the 3-run mean is **`81.55` taps** under forced-air ambient cooling (`C1`–`C3`) vs. **`81.41` taps** under heating (`H1`–`H3`), a net shift of **`-0.14` taps**.
   - On `DDR1`, the 3-run mean is **`80.28` taps** (`C1`–`C3`) vs. **`81.31` taps** (`H1`–`H3`), a net shift of **`+1.03` taps**—smaller than the natural run-to-run mean absolute difference (`1.29`–`2.00` taps across consecutive runs in the same condition).
   - `TrainedVREFCA` (`0x3f`), `TrainedVREFDQ` (`0x25`–`0x28`), and the four eye margins (`RxClkDly_Margin = 0x15..0x18`, `VrefDac_Margin = 0x22..0x27`, `TxDqDly_Margin = 0x14..0x19`, `DeviceVref_Margin = 0x1c..0x24`) are indistinguishable between `C1`–`C3` and `H1`–`H3`.
2. **Boot-path baseline shift (USB/FDL staging vs. SPI cold boot) and `CSR[0x020011]` save/restore mechanism**:
   - In `Step 3.3` (`Quickboot == 0`), the PMU saves `CSR[0x020011]` (`0x006f`) to DMEM `halfword[0x8000042a]` and masks `CSR[0x020011] &= 0xff9f` (`0x000f`). At the end of `DevInit` (`Step 6.3`, also gated on `Quickboot == 0`), the PMU restore helper writes `halfword[0x8000042a]` (`0x006f`) back to `CSR[0x020011]` alongside restoring `CSR[0x01<r>0a3] = 0x0b33` (`0x80000430 + 2 * r`). Thus, `CSR[0x020011] = 0x006f` in all six USB/FDL runs (`C1`–`C3` and `H1`–`H3`) is the exact value restored from `0x8000042a` at `Step 6.3`, whereas the SPI cold-boot envelope (`0x000f`) reflects a state where the `& 0xff9f` mask ran without the `Step 6.3` restore remaining in effect.
   - Alongside `CSR[0x020011] = 0x006f` (and `CSR[0x0c0086] = 0x0001` vs. `0x0000` in SPI, and `CSR[0x01<r>0af] = 0x003a..0x0040` vs. `0x0038..0x003b` in SPI), every USB/FDL run shifts the `0x4e`/`0x4f` read-eye centers by **`+19` taps** (`62.5 / 62.0` taps in `SPI` → `81.5–82.0 / 80.5–82.0` taps in FDL) and shifts all four `TrainedVREFCA` outputs (`A0..B1`) from **`0x27`–`0x2c` (`39`–`44`) to `0x3f` (`63`)**. Because `0x3f` (`63`) is the maximum value of a 6-bit DAC field (`0..63`), all four CA VREF outputs hitting `0x3f` simultaneously indicates upper-rail saturation of the CA VREF search in the USB/FDL operating regime, even though training completes (`0x07`) and the 1D/2D DQ margins remain normal.
3. **Validation rule — Treat SPI cold boot and USB/FDL staging as two distinct operating points**:
   - Never use the SPI cold-boot envelope as a ground-truth reference for `CSR[0x01<r><l>4e]`/`0x4f` eye centers or `TrainedVREFCA` on a USB/FDL-staged run (or vice versa). Always compare runs taken through the same boot path (SPI vs. SPI, or USB/FDL vs. USB/FDL), checking `CSR[0x020011]` (`0x000f` vs. `0x006f`) first to confirm which operating point applies.

### Standalone validation criteria (without a reference capture)

When validating an independent trainer on a board without comparing against a prior `phypara` dump, verify all five standalone invariants:

1. **Execution duration**: Full training (`SequenceCtrl = 0x125f`) completes in the multi-second range (~4.4–5.8 s), never in a few milliseconds (`5–41 ms` indicates an early abort or skipped DQ sweeps).
2. **`DevInit` completion markers**: `CSR[0x02000a]` holds the latched CA clock period (`0x007e`–`0x0086` at 6400 MT/s, updated away from its `0x0040` pre-training seed after the bit 10 falling-edge strobe on `CSR[0x0200e3]`, while `CSR[0x020017]` remains `0x003e`–`0x0040`), `CSR[0x020012]`/`0x020186`/`0x020187` hold `0x7353`, `0x0073`, and `0x0041`–`0x0042` (updated from `0xa1a1`, `0x00a1`, and `0x0040`), `CSR[0x01<r>0ad]` holds `0x0002`, `CSR[0x01<r>0af]` holds `0x0038`–`0x0040` across `r = 0..3`, `CSR[0x01<r>0d9]` holds `0x009c` (`156`) across all active ranks, and `CSR[0x020060]` (`MemResetL`) holds **`0x0003`** (not `0x0001`, because `Misc == 0x00`). Note that `CSR[0x01<r>0a3] == 0x0b33` is written by the host pre-training table *before* the PMU starts (and restored in `Step 6.3` along with `CSR[0x020011] = 0x006f`), while `CSR[0x0c0086]` (`ARC 0x9018010c`) is a host-side register (`training_table_54` writes `0x0001`, `0` references in the PMU firmware) and must **not** be used as a PMU-side `DevInit` self-check.
3. **Trained read-eye centers**: Every active lane (`4 ranks × 9 lanes`) has `CSR[0x01<r><l>4e]` and `0x4f` updated away from the `0x0035` pre-training seed (with `0x50`/`0x51` remaining `0x0035` when `DisableTrainingLoop[1] = 0`).
4. **2D eye opening** (when contour streaming is enabled): Horizontal width is in the **70–124 tap** range (median ~102 taps) and vertical height is **21–32 voltage levels**.
5. **Self-repeatability**: Two consecutive training runs of the new firmware under the same boot path agree within **`<= ±3` taps** across all 36 lanes (excluding the volatile CSRs below).

### Volatile / hardware-counter CSRs to exclude from bit-exact `phypara` diffs

Inside the 5,170-entry `phypara` dump, the live period-counter register and hardware sequencer/status registers can vary slightly across runs or between `DevInit`-only (`SequenceCtrl = 0x0001`) and full-training (`0x125f`) captures:

| CSR index | ARC address | Reference pre → post (`0x125f` / `0x0001`) | Reason |
|---:|---:|---|---|
| `0x02000a` | `0x90040014` | `0x0040` → `0x007f`–`0x0082` (`127`–`130`) / `0x0086` (`134`) | Live CA period counter (`CSR[0x0300bf] & 0x01ff`) latched by the `CSR[0x0200e3]` bit 10 falling edge during `DevInit` (`±1..4` tick variation) |
| `0x020017` | `0x9004002e` | `0x0040` → `0x003e`–`0x0040` (`62`–`64`) | Live DQ period counter (`CSR[0x0100e0] & 0x01ff`) latched alongside `0x02000a` |
| `0x020012` | `0x90040024` | `0xa1a1` → `0x7353` | Deterministic frequency bucket computed from `DRAMFreq` (`6400`) and `DfiFreqRatio` (`4`) (`f = 1600`, `f_hi = 9600`; note that `0x3221`/`0x4332` only occurs if a custom trainer mistakenly classifies `2 * T_ca` instead of `DRAMFreq / DfiFreqRatio`) |
| `0x020186` | `0x9004030c` | `0x00a1` → `0x0073` | Secondary frequency bucket `(bucket3 << 4) \| fine_hi` computed from `DRAMFreq` and `DfiFreqRatio` |
| `0x020187` | `0x9004030e` | `0x0040` → `0x0041`–`0x0042` (`65`–`66`) | Copied unmasked from rank-0 read-only hardware register `CSR[0x0100d5]` (`ARC 0x900201aa`) during `DevInit` |
| `0x09080b` | `0x90121016` | `0x002f` → `0x000f` (or `0x0015`/`0x00ff`) | Masked `& 0xff9f` alongside ACSM clock setup at `CSR[0x090808]` (`0x90121010 + 6`) |
| `0x09080c` | `0x90121018` | `0x006f` → `0x000f` (or `0x002d`/`0x0065`) | Masked `& 0xff9f` alongside ACSM clock setup at `CSR[0x090808]` (`0x90121010 + 8`) |
| `0x0c0086` | `0x9018010c` | `0x0001` → `0x0000` (SPI) / `0x0001` (FDL) | Host-side ECC register (`training_table_54` writes `0x0001`; `0` references in PMU firmware) |

### Diagnostic failure signatures

When bringing up new PMU firmware or host table overrides, use the following symptom table to isolate the root cause quickly:

| Observed symptom | Typical duration | Root cause |
|---|---:|---|
| PMU hangs silently (no `0x07` or `0xff`; USB FDL gadget wedged on reset) | Timeout | Instruction referenced `r16`–`r25` (missing `-mrf16` at compile or link), unaligned 32-bit `ld` on a 6-byte table stride, 32-bit `ld`/`st` on a 16-bit `0x90xxxxxx` CSR, AUX `0x103`–`0x105` accessed as memory instead of `lr`/`sr`, or `libgcc` `__muldi3`/`__udivsi3` invoked |
| PMU emits fatal `0x000000ff` early (`DDR training failed!`) | **12–41 ms** | Invalid electrical configuration (`TX DS >= 60 Ω` or `OFF`), `.bss` linked at `0x80000400` (wiping host-supplied `phypara` / canary `0x80000424`/`0x426`), or `CsPresent`/`EnabledDQs` mismatch |
| PMU emits `0x07` far too fast (`DDR training consume 5 ms`–`28 ms`) | **5–28 ms** | 32-bit store to `0x80000458` clobbered delay divisor `0x8000045a`, or `SequenceCtrl` stage mask lacked `RdDQS` (`bit 3`) / `WrDq` (`bit 4`) / `DRAM DCA` (`bit 6`) |
| Late host message-block override (`SequenceCtrl`, `HdtCtrl`, `Train2DMisc`) has no effect | Normal (~5.6 s) | Host wrote `DPHY_BASE + 4 * 0x58000` *after* the pre-training phase closed (immediately before `pwrctl` bit 11); hardware discards those writes |
| Mailbox streaming takes 5–7 minutes with `Train2DMisc[0] = 1` | **>300 s** | Host receiver printed a fixed 15/16-word buffer per frame over 115200 bps UART instead of printing only `len + 1` words |
| `CSR[0x01<r><l>4e]` stays at `0x0035` after training | Variable | `RdDQS` (`SequenceCtrl[3]`) did not run, or `DisableTrainingLoop[1]` (`0x80000102 & 2`) was inadvertently set, averaging `0x4e` with `0x50` (`0x0035`) |
| `CSR[0x0300bf]` and `CSR[0x0100e0]` read `0x0240` (`CSR[0x02000a]` stays at `0x0040`) or `CSR[0x020011]` stays at `0x006f` | ~15 ms+ | Period-counter strobe omitted the **falling edge on bit 10 of `CSR[0x0200e3]`** (`0x0800 -> 0x0c00 -> delay_dfi_clks(10) -> 0x0800` after `CSR[0x0200e5] &= ~0x001f`), leaving `0x0300bf & 0x01ff` at the unlatched pre-training value `0x0040` instead of `0x0082`–`0x0086`, or missing `volatile` on `0x9004xxxx` accesses caused the compiler to optimize out the strobe |
| `CSR[0x0300bf]` and `CSR[0x0100e0]` saturate at **`0x17ff`** (`CSR[0x02000a] = 0x01ff = 511`) and `CSR[0x0100d5]` stays at `0x019a`–`0x01bf` (`410`–`447`) instead of `0x0041` (`65`) | ~15 ms+ | Counter latched, but the PHY PLL / target clock was stopped or unstable during measurement: verify that the 3-entry descriptor table at `0x800004cc` (`CSR[0x02006e] = 1`, `0x070060 = 0`, `0x070011 = 0`) was applied, `Step 3.5.12` waited the full **`400 µs` (`160,000` RTC ticks)** (`delay_ps_or_clks(0, 10000)` with `byte[0x80000a80] == 1`, `byte[0x80000b68] == 8`, `byte[0x80000b69] == 4`) rather than `12.5 µs` (`5,000` ticks), and `Step 3.6` per-rank DQ calibration executed before `Step 3.7` |
| `CSR[0x020021]` (`ARC 0x90040042`) reads `1` and an ACSM poll loop times out after `25,000` iterations during `Step 6.1`/`6.2` | ~15 ms+ | **Inverted `CSR[0x020021]` poll polarity**: `CSR[0x020021]` is active-high **`ACSM_DONE`** (`1` = done/idle/ready, `0` = busy/running), **not** active-high busy! Because a short one-shot ACSM burst (such as a 4-cycle NOP) finishes in ~5 ns, `CSR[0x020021]` already reads `1` (`DONE`) on the first poll iteration; the firmware must poll **`while (CSR[0x020021] == 0)`** (exiting as soon as `CSR[0x020021] != 0`), and keep `CSR[0x020031] = 0` for one-shot `DevInit` bursts |
| `CSR[0x090029]` (`ARC 0x90120052`) reads `0x0145` (`0x6745 & 0x01ff`) and the startup canary check fails | < 5 ms | Startup omitted decoding `CSR[0x0c00ee]` (`ARC 0x901801dc`) into `byte[0x80000403]` (`8`–`10` on K3) before the canary check, leaving `byte[0x80000403] == 0` and selecting 9-bit `CSR[0x090029]`/`0x09002a` instead of 16-bit `CSR[0x09002e]`/`0x09002f` (`ARC 0x9012005c`/`0x9012005e`) |
| `CSR[0x01<r>0ad]` (`ARC 0x9002015a + r * 0x2000`) reads `0x0005` (fallback) instead of `0x0002` after `Step 3.6` | ~15 ms+ | **`Step 3.6` mistakenly looped over lanes `l = 0..8` (`+ l * 0x200`) instead of iterating only once per rank `r = 0..3`**: `CSR[0x01<r>0d0..d5]` and `CSR[0x01<r>0ad]` exist only once per rank (at sub-block `0`), so `l = 0` reads `d[0..4] = {0, 0, 0x41, 0x85, 0xca}` (`k = 2`) and then `l = 1..8` read unimplemented zeros (`d[0..4] = {0, 0, 0, 0, 0}`) and overwrite `CSR[0x01<r>0ad]` with the fallback `k = 5` |
| Multi-unit ACSM NOP delays (`10` or `16` units in `Step 6.2.5`) execute for only `2` or `8` units | ~15 ms+ | **Missing upper 5 bits `r[7:3]` in `hw[7]` of the 8-halfword NOP pair**: the 8-bit repeat count `r = ceil(cycles / divisor) - 1` is split across `hw[3] = ((r & 7) << 12) \| 0x0b00` (low 3 bits) and `hw[7] = (r & 0xf8) << 7` (upper 5 bits); for `10` units (`r = 9`) and `16` units (`r = 15`), `hw[7]` must be `0x0400` |

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
