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
3. **Per-stage CSR write attribution and `DevInit` vs. full-training diff (Resolved)**: `SequenceCtrl` stage sweeps (`0x0000` through `0x125f`) established the exact runtime and CSR footprint of every training stage, the origin classification of all 5,170 `phypara` entries (including the 107 `CHANGED` entries and 72 `CSR[0x4e]`/`CSR[0x4f]` read-eye centers), and the `DevInit` group `0x02` clock-period calibration sequence (see [07-training-behavior.md](07-training-behavior.md) and [09-training-results.md](09-training-results.md)).
4. **2D eye contour streaming format (`0x0076` / `0x0077`) and PHY ODT/drive response curves (Resolved)**: Enabling `Train2DMisc[0] = 1` with `HdtCtrl = 0x05` streams 1,296 2D eye contours (`0x00760004` header + 64× `0x00770003` row slices), and sweeping PHY TX drive strength (`0x01<r>040`) and RX ODT (`0x01<r>048`) established the pass/fail step function (`<= 40 Ω`) and monotonic eye-width curve (`42` to `106` taps; see [04-parts-topologies-and-timings.md](04-parts-topologies-and-timings.md)).

---

## Unresolved structural and protocol questions

### 1. DDRC timing derivation for non-reference parts (host side)

Host-side DDRC timing registers depend on the selected LPDDR5 part, rank count, x8/x16 organization, and data rate (`4800` or `6400` MT/s). For parts other than the reference `MT62F2G32D4DS`, host timing values must be derived from the LPDDR5 device datasheet (`ceil(ns * rate / 2000)`) and validated on hardware.

### 2. Remaining payload schemas for streaming message IDs and `sub = 0..17` in `0x0076`

The streaming wire framing (`header = (id << 16) | len` followed by `len` 32-bit payload words) is established (see [08-mailbox-and-streaming.md](08-mailbox-and-streaming.md) and [appendix-streaming-catalog.md](appendix-streaming-catalog.md)), and the 2D eye contour pair (`0x0076`/`0x0077`) plus selected patterns (`0x007e` 31-word array, `0x0104` 128-byte sweep, `0x00b1`/`0x00b4`/`0x011a` packed 8-bit lane masks, `0x01e0`/`0x01e9` frequency reports) are identified. However, the exact field-by-field schema of the remaining streaming IDs and the exact meaning of the `sub = 0..17` sub-sweep index in `0x0076` (`9 bits × 2 seeds` vs. `2 iterations × 9 sub-phases`) remain unconfirmed.

### 3. Major stage byte to training-phase mapping

A full-training capture emits the major message sequence `0d 00 01 02 fd 06 01 02 fd 04 03 09 07`. Aside from `0x07` (normal completion), `0x08` (streaming introducer), and `0x000000ff` (fatal error), the exact one-to-one mapping between intermediate stage codes (such as `0x0d`, `0xfd`, `0x06`, `0x04`, `0x03`, `0x09`) and `SequenceCtrl` algorithm steps is not confirmed.

---

## Unresolved training and behavioral questions

### 4. Internal eye-acceptance and retry criteria

How the PMU training algorithm decides that a candidate delay/VREF point or an entire 1D/2D eye passes—including the base comparison count for `BitTimeControl = 0`, allowed bit-error count (if any), minimum acceptable eye width/margin (`RxClkDly_Margin`, `VrefDac_Margin`, `TxDqDly_Margin`, `DeviceVref_Margin`), and maximum retraining loop iterations—is not established.

### 5. Quantitative relationship between the `0x0076`/`0x0077` contour midpoint (~62.5 taps) and `CSR[0x4e]` (`0x2d`–`0x4a`)

While `RdDQS` (`SequenceCtrl[3]`) both emits the 2D read eye contours (`0x0076`/`0x0077`, whose horizontal midpoint is ~62.5 taps) and trains `CSR[0x01<r><l>4e]`/`0x4f` (`0x2d`–`0x4a`, i.e. 45–74 taps), the exact mathematical transformation between the DQS/gate contour coordinate system and the final per-lane DQ delay value written into `CSR[0x4e]` is not yet closed.

### 6. Host post-activation smoke-test behavior in source-built loaders

When building the first-stage loader (`FSBL.bin`) from source with a different GCC toolchain, the post-activation host memory smoke test (`0x4000`-byte pattern/complement check) was observed to hang independently of the PMU firmware, whereas the factory prebuilt `FSBL.bin` completes its post-training flow. Resolving whether this discrepancy stems from compiler code generation, 64-bit pointer/addressing above 4 GiB, cache/MMU state, or DDRC port/region setup requires a controlled host-side investigation.

### 7. Multi-P-state training and P-state SRAM (`0xa0000`)

The message block supports `Pstate` selection and a 15-P-state DMA mode (`Pstate[7]`), and the DPHY exposes a 16,384-halfword P-state SRAM window at `0xa0000`. However, only single-P-state operation was observed on the reference board, and P-state SRAM restoration is inactive in the observed quickboot sequence.

---

## Unresolved hardware questions

### 8. Bit-level definitions of internal PHY CSRs, physical lane-to-DQ mapping, and mirror group `0x110000`

While CSR access widths (16-bit), the `group/rank/lane/offset` index structure, the 69 `phypara` serialization segments, and the per-stage CSR write footprints are known, several hardware-level questions remain open:

- the exact physical mapping from `lane = 0..8` within each rank (`0..3`) to `ChA`/`ChB` DQ byte lanes and DBI/ECC slices (`L4..L7` on Controller 1 exhibit ~20-tap smaller `0x4e` values than `L0..L3, L8`);
- the hardware role of live mirror group `0x110000` (`ARC 0x90220000`, 1,180 CSRs accessed during training, 1,024 non-zero, not included in `phypara`); and
- the internal hardware mechanism that updates `CSR[0x09080b]` and `CSR[0x09080c]` during training.

### 9. Expected PHY hardware signature/version value

The PMU firmware checks a hardware signature/version register early in execution and aborts on mismatch, and writes `PmuRevision` (`0x002`) to the message block. The complete table of accepted silicon revision codes is not cataloged.

### 10. Temperature derating curves and 4800 MT/s hardware validation

DDRC derating registers (`0x00b04`–`0x00b08`, `0x10104`–`0x10108`) and 4800 MT/s clock/timing tables exist in the interface, but hardware validation in this study was performed at 6400 MT/s under ambient bench temperature.

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
