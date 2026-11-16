<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Validation & Measurements](14-validation-and-current-status.md) | [Index](README.md#reading-order-and-contents) | [Next: Flash, Build & Recovery →](16-flash-build-and-recovery.md)

# Open questions and specification gaps

## Scope

This chapter catalogs the structural, behavioral, and hardware details that are **not** established by this specification. An independent implementation team must treat these items as open engineering questions to be resolved through controlled hardware experimentation or memory datasheets—never by consulting proprietary vendor firmware source code or binaries.

---

## Resolved structural facts (for historical clarity)

Several questions that were open in early hardware bring-up are now **fully resolved**:

1. **Second DDR controller base address (Resolved)**: The K3 SoC exposes two complete controller instances: Instance 0 at `DDRC = 0xcb000000`, `CFG = 0xcb600000`, `DPHY = 0xcb800000` and Instance 1 at `DDRC = 0xcc000000`, `CFG = 0xcc600000`, `DPHY = 0xcc800000` (see [02-addressing-and-access.md](02-addressing-and-access.md)).
2. **Exact `phypara` serialization order and segment start offsets (Resolved)**: The 5170 halfwords collected after training and restored during quickboot follow the exact 69-segment table documented in [appendix-phypara-order.md](appendix-phypara-order.md), including the non-zero starting offsets on sequential segments 48 (`start_offset = 46`), 54 (`292`), and 55 (`395`), yielding 5,170 unique CSR indices.
3. **Per-stage CSR write attribution and `DevInit` vs. full-training diff (Resolved)**: `SequenceCtrl` stage sweeps (`0x0000` through `0x125f`) established the exact runtime and CSR footprint of every training stage, the origin classification of all 5,170 `phypara` entries (including the 107 `CHANGED` entries and 72 `CSR[0x4e]`/`CSR[0x4f]` read-eye centers), the two-tier `delay_dfi_clks` / `delay_ps_or_clks` timing model, the `DevInit` group `0x02` clock-period calibration and frequency-bucket formula (`DRAMFreq / DfiFreqRatio` → `0x7353` / `0x0073`), the `CSR[0x020031]` ACSM sequencer enable, and the `0x80000400`–`0x80000e4e` DMEM parameter layout (see [07-training-behavior.md](07-training-behavior.md), [09-training-results.md](09-training-results.md), and [12-arc-execution-environment.md](12-arc-execution-environment.md)).
4. **2D eye contour streaming format (`0x0076` / `0x0077`) and PHY ODT/drive response curves (Resolved)**: Enabling `Train2DMisc[0] = 1` with `HdtCtrl = 0x05` streams 1,296 2D eye contours (`0x00760004` header + 64× `0x00770003` row slices), and sweeping PHY TX drive strength (`0x01<r>040`) and RX ODT (`0x01<r>048`) established the pass/fail step function (`<= 40 Ω`) and monotonic eye-width curve (`42` to `106` taps; see [04-parts-topologies-and-timings.md](04-parts-topologies-and-timings.md)).
5. **PHY hardware version (`CSR[0x0c00ee]`), signature canary (`0x00f26745` at `CSR[0x09002e]`/`0x09002f`), and `PmuRevision = 0x2409` (Resolved)**: At startup, the PMU decodes `CSR[0x0c00ee] & 0x0fff` (`ARC 0x901801dc`) into `byte[0x80000403]` (`8`–`10` on K3), writes the 32-bit signature canary `0x00f26745` (`0x6745`, `0x00f2`) to the 16-bit registers `CSR[0x09002e]`/`0x09002f` (`ARC 0x9012005c`/`0x9012005e`), verifies that they match the host-supplied comparison words at DMEM `0x80000424` (`0x6745`) and `0x80000426` (`0x00f2`) (aborting with `0x000000ff` on mismatch), and writes `PmuRevision = 0x2409` to `0x80000002` (see [07-training-behavior.md](07-training-behavior.md) and [12-arc-execution-environment.md](12-arc-execution-environment.md)).
6. **Absence of LPDDR5 `MRR` readback in the PMU firmware (Resolved)**: The PMU firmware never issues LPDDR5 Mode Register Reads (`MRR`) and contains no `MRR` readback CSR path; all mode-register interactions use `MRW` and `MPC` bursts through the ACSM sequencer.
7. **Hardware transfer-and-compare engine, 2D eye centroid/margin calculation, general 8-halfword ACSM command encoding, and per-stage hardware CSR flows (Resolved)**: Data comparison during sweeps runs entirely in PHY hardware via pattern generator setup (`CSR[0x07f061..64] = 0xffff`, per-lane `CSR[0x07<lane>024..2b] = 0x5a3c`, `CSR[0x07f011] = 0x0016`, `CSR[0x07f065] = 0x0000`, `CSR[0x07f0d6/d7] = 0x0100`, `CSR[0x07f060] = 0x008c`), DRAM pattern MRs (`MR20[5:0]` preserved, `MR33 = 0x3c`, `MR34 = 0x5a`, `MR31 = MR32 = 0x00` or `byte[0x800000e6]`), `MR46 = 0x04 -> 0x00` gated by `CSR[0x01f001]`, ACSM entry-`200` (`CSR[0x041320..]`) per-rank write-then-read bursts (`0x001c`/`0x0060` followed by `0x002c`/`0x0020`), outer `CSR[0x01<dbyte><lane>4e/4f]` scan + inner `CSR[0x01<dbyte>fb5]` / `CSR[0x01<dbyte>f10..13]` (or `0x26..29` for `WrDq`) scan, boolean hardware pass/fail reads (`CSR[0x01<dbyte><lane>b7]`, `0 == PASS`), per-dbyte sticky clears (`CSR[0x01f0bb] = 0x03ff -> 0` between bursts), 2D centroid `(X_c, Y_c)` weighting (`Train2DMisc[12:11]`), minimum-margin accumulation (`0x037`–`0x03e`/`0x052`–`0x059`), general `8`-halfword ACSM command descriptor encoding (`hw[0..7]`), execution order (`SequenceCtrl` + `DisableTrainingLoop`), and host DMEM re-access between PMU runs (`0xd0099`/`0xd0000`/`0xc0080`) are specified in [07-training-behavior.md](07-training-behavior.md), [11-quickboot.md](11-quickboot.md), and [12-arc-execution-environment.md](12-arc-execution-environment.md).

---

## Unresolved structural and protocol questions

### 1. DDRC timing derivation for non-reference parts (host side)

Host-side DDRC timing registers depend on the selected LPDDR5 part, rank count, x8/x16 organization, and data rate (`4800` or `6400` MT/s). For parts other than the reference `MT62F2G32D4DS`, host timing values must be derived from the LPDDR5 device datasheet (`ceil(ns * rate / 2000)`) and validated on hardware.

### 2. Remaining payload schemas for streaming message IDs and `sub = 0..17` in `0x0076`

The streaming wire framing (`header = (id << 16) | len` followed by `len` 32-bit payload words) is established (see [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md) and [appendix-streaming-catalog.md](appendix-streaming-catalog.md)), and the 2D eye contour pair (`0x0076`/`0x0077`) plus selected patterns (`0x007e` 31-word array, `0x0104` 128-byte sweep, `0x00b1`/`0x00b4`/`0x011a` packed 8-bit lane masks, `0x01e0`/`0x01e9` frequency reports) are identified. However, the exact field-by-field schema of the remaining streaming IDs and the exact meaning of the `sub = 0..17` sub-sweep index in `0x0076` (`9 bits × 2 seeds` vs. `2 iterations × 9 sub-phases`) remain unconfirmed.

### 3. Major stage byte to training-phase mapping

A full-training capture emits the major message sequence `0d 00 01 02 fd 06 01 02 fd 04 03 09 07`. While the high-level milestone grouping (`0x0d` entry, `0x00 0x01 0x02 0xfd` initial pass, `0x06 0x01 0x02 0xfd` retraining pass, `0x04 0x03 0x09` eye/DCA/CA optimization, `0x07` completion) is documented in [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md), the intermediate stage codes are passed dynamically at runtime and should be treated as informational progress items.

---

## Unresolved training and behavioral questions

### 4. Remaining eye-acceptance thresholds and truncated-eye boundary policy

The hardware transfer-and-compare primitive, 2D centroid `(X_c, Y_c)` weighting (`Train2DMisc[12:11]`), and minimum-margin accumulation (`0x037`–`0x03e`/`0x052`–`0x059`) are resolved in [07-training-behavior.md](07-training-behavior.md). What remains open is the base comparison count for `BitTimeControl = 0`, any non-zero error tolerance, truncated-eye boundary clamping, minimum acceptable margin floors, and maximum retraining loop bounds.

### 5. Quantitative relationship between the `0x0076`/`0x0077` contour midpoint (~62.5 taps) and `CSR[0x4e]` (`0x2d`–`0x4a`)

While `RdDQS` (`SequenceCtrl[3]`) both emits the 2D read eye contours (`0x0076`/`0x0077`, whose horizontal midpoint is ~62.5 taps) and trains `CSR[0x01<r><l>4e]`/`0x4f` (`0x2d`–`0x4a`, i.e. 45–74 taps), the exact mathematical transformation between the DQS/gate contour coordinate system and the final per-lane DQ delay value written into `CSR[0x4e]` is not yet closed.

### 6. Host post-activation smoke-test behavior in source-built loaders

When building the first-stage loader (`FSBL.bin`) from source with a different GCC toolchain, the post-activation host memory smoke test (`0x4000`-byte pattern/complement check) was observed to hang independently of the PMU firmware, whereas the factory prebuilt `FSBL.bin` completes its post-training flow. Resolving whether this discrepancy stems from compiler code generation, 64-bit pointer/addressing above 4 GiB, cache/MMU state, or DDRC port/region setup requires a controlled host-side investigation.

### 7. Remaining quickboot `phypara` Pass 3 pair-by-pair serialization and multi-P-state SRAM (`0xa0000`)

The host-side quickboot sequence and the overall standalone quickboot PMU hardware flow—entry at `0x1fc` with `gp = 0x80005400`, forcing `SequenceCtrl = 1` and `Quickboot = 1` (not executing full-training `Step 3.1–6.3`, period calibration [`CSR[0x02000a/12/17]` restored in Pass 1; `CSR[0x020187]` has `0` references], `Step 3.6` [`0x01<r>0d5` absent], `RxEn`/`WrLvl` sweeps, or the `200 ms + 2 s` `tINIT` wait), the three-pass deserialization from `0x80005478` (`0x80000400`, DMEM `0x58200`) **with Pass 1 (`46` group `0x01` bases including `0x01<r><l>4e..51`, `0x01<r>00c/00d`, unmasked `0x01<r>0a3`, `0x01<r>0ad/af`, `0x0c0086`; Segments `#0..#25`) and Pass 2 (complete `47`-halfword per-half-iteration DQ restore: `9 lanes × 4` `WrDq` delays `0x10..13`/`0x24..27` + `11` pairs across `30` offsets, plus `5` CA base CSRs; Segments `#26..#37`) preceding DRAM reset and MRW**, CA (`0x03xxxx`) split across all three passes (`33`/`5`/`28` base CSRs), the `710`-halfword quickboot DMEM table loaded at **`0x5aa00`** (`ARC 0x80005400`: `0x6745`/`0x00f2` canary at `0x1c`, cursor at `0x78`, 3-entry `CSR[0x02006e]/0x070060/0x070011` startup script at `0xc8`), the `0x02f052` commit strobe, DQ block reset, MRW burst + per-rank `MR40 = 0x40`/`MR37 = 0x40` + `MR28` / `cmd = 0x28` / `NOP` tail (prior to Step 7 ACSM dividers), and the **`0x02f049`/`0x02f0e0`/`0x02f0c3` DLL/clock recalibration** (`1 ns`/`1 µs`/`65` min ticks without `32×` scaling, **sampling `CSR[0x0200cf] & 0x1f` while `0x02f0e0 == 1` and `0x02f049 == u | 0x0400` before clearing `0x02f0e0 = 0`**, no poll after `0x02f0c3 = 1`, `CSR[0x09f02d] = 1` level on K3 PHY version `8`–`10`, `CSR[0x02004e] = 0x0b`–`0x0d` copied from `CSR[0x0200cf] & 0x1f`) followed by Pass 3 (`Segments #38..#68`, `384` base CSRs across `0x00`–`0x0c`, including `75` group `0x01` bases with `0x010019/1b/22/23` and `0x01004c/4d`, `0x090803..27` with `0x090808/13` requiring post-DLL ordering, `0x090900..87`)—are documented in [11-quickboot.md](11-quickboot.md).

Two sub-items remain unenumerated in the quickboot specification:
1. The exact pair-by-pair serialization order across Pass 3's `384` base CSRs (`0x00`–`0x0c` multi-P-state/rank/lane branches); Pass 1 and Pass 2 (`47`-halfword DQ + `5` CA base CSRs) are completely specified.
2. Multi-P-state SRAM (`0xa0000`) restoration, which is inactive (`0`) in the single-P-state flow observed on the reference board.

---

## Unresolved hardware questions

### 8. Remaining ACSM training command details and `MRR` absence

The Address/Command State Machine (ACSM) SRAM (`0x41000`–`0x41fff`, `ARC 0x90082000`–`0x90083ffe`), its group `0x02` sequencer control registers (`CSR[0x020020]`–`0x020031`, active-high `ACSM_DONE` on `CSR[0x020021]`), its clock dividers (`CSR[0x090808]`/`0x090813`), the 4-halfword `MRW`/`MPC`/`NOP` encoding, the general 8-halfword (`hw[0..7]`) command descriptor format (`hw[3]` command code, `hw[4]` payload, `hw[5]` condition/cs, `hw[7]` attribute word), and the entry-`200` (`CSR[0x041320..]`) read/write training burst are documented in [07-training-behavior.md](07-training-behavior.md).

What remains open within the ACSM subsystem is the detailed bitfield encoding of the `hw[7]` attribute/timing control word and the non-`MRW` bus command codes (`0x000c`, `0x0018`, `0x0038`, `0x0040`, `0x0050`, `0x0058`, `0x0060`, `0x0068`, `0x0078`, `0x0080`, `0x00d0`). Note that if an independent trainer wishes to read LPDDR5 mode registers (`MR5`–`MR8`) via `MRR`, that requires defining a new host/ACSM `MRR` sequence, as the reference PMU firmware never performs `MRR`.

### 9. Remaining internal PHY CSR details, `PHY DCA` registers, and odd-P-state banking

While CSR access widths (16-bit), the `group/rank/lane/offset` index structure, the 69 `phypara` serialization segments, and the per-stage CSR programming sequences for `WrLvl`, `RxEn`, `RdDQS`, `WrDq`, `MxRdLat`, `DRAM DCA`, and `LPCA` are known ([07-training-behavior.md](07-training-behavior.md)), four specific hardware details remain open:
1. Additional group `0x01` CSRs adjusted inside `PHY DCA` (`SequenceCtrl[7]`/`[8]`, alongside `CSR[0x01f007]`, `0x010000`, `0x01004e`, `0x01f009`, and per-lane `0x01<dbyte><lane>80`);
2. Live odd-P-state hardware confirmation of whether parity-indexed bank selection (`(Pstate & 1) << 20` on ARC `0x9002xxxx`) maps group `0x01` CSRs to mirror group `0x110000` (`ARC 0x90220000`);
3. Initial population source of the optional per-dbyte custom AC buffer (`0x80000444`, read when `EnCustomSettings != 0`);
4. The exact physical mapping from `lane = 0..8` within each rank (`0..3`) to `ChA`/`ChB` DQ byte lanes and DBI/ECC slices (`L4..L7` on Controller 1 exhibit ~20-tap smaller `0x4e` values than `L0..L3, L8`).

### 10. Temperature derating curves and 4800 MT/s hardware measurements

DDRC derating registers (`0x00b04`–`0x00b08`, `0x10104`–`0x10108`) and 4800 MT/s clock/timing tables exist in the host interface, and the PMU firmware's frequency-dependent formulas (`DRAMFreq = 4800` → `byte[0x80000b68] = 6`, `CSR[0x020012] = 0x5232`, `CSR[0x020186] = 0x0052`) are established, but all hardware training captures in this study were recorded at 6400 MT/s (`MT62F2G32D4DS`); empirical 4800 MT/s `phypara` and eye-margin captures have not yet been measured on the bench.

---

## Policy for handling open questions in a clean-room implementation

- **Separate measured facts from assumptions**: Document clearly in code comments whenever a timeout, margin floor, or retry count is an implementation choice rather than a hardware requirement.
- **Treat unknown message codes and reserved bytes as opaque**: Log unknown major/streaming IDs without aborting, and preserve all 320 message-block bytes and 5170 `phypara` halfwords without filtering or reordering.
- **Close gaps by hardware measurement**: Validate any new timing calculation, eye-margin threshold, or host memory test through controlled bench experiments (recording artifact hashes, topology, and serial logs) rather than assuming Linux login or benchmark speed implies full calibration equivalence.

## Related chapters

- [04-parts-topologies-and-timings.md](04-parts-topologies-and-timings.md)
- [07-training-behavior.md](07-training-behavior.md)
- [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md)
- [13-firmware-behavioral-requirements.md](13-firmware-behavioral-requirements.md)
- [14-validation-and-current-status.md](14-validation-and-current-status.md)
