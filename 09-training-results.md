<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Mailbox & Streaming Protocol](08-mailbox-and-streaming.md) | [Index](README.md#reading-order-and-contents) | [Next: Persistence & Selection →](10-persistence-and-selection.md)

# Training results

## Scope and evidence

This document describes the result produced by each LPDDR5 PMU training run and the exact serialized representation consumed by quickboot. Byte offsets in the message are relative to the start of its `0x140`-byte structure. PHY indices denote 16-bit CSR/DMEM locations, not byte addresses.

The layouts and collection order below are **observed** in the loader, firmware interface definitions, and captured board data. Statements about robustness or implementation policy are explicitly marked as inferred recommendations; unresolved meanings are listed separately.

## Observed: result containers

A complete training result has three components. The message preserves both configuration and results, `phypara` preserves selected PHY state in a defined serialization order, and ACSM preserves a contiguous SRAM region.

| Component | Elements | Bytes | Runtime source |
|---|---:|---:|---|
| Message block | `0xa0` halfwords | `0x140` | DMEM index `0x58000` |
| `phypara` | 5170 halfwords | `0x2864` | 69 ordered PHY segments |
| ACSM | 4096 halfwords | `0x2000` | contiguous SRAM at index `0x41000` |

The complete message must be retained, including fields whose meaning is not yet known. Saving only the documented outputs is insufficient because quickboot restores the whole structure.

The host-visible message index is `0x58000`. For a 16-bit-aligned message field:

`DMEM index = 0x58000 + byte_offset / 2`

Eight-bit fields share a little-endian halfword. For example, `DRAMFreq` at byte `0x06` is at index `0x58003`, and `SequenceCtrl` at byte `0x10` is at index `0x58008`. The message occupies `0xa0` halfwords within a reserved `0x200`-halfword area.

## Observed: message outputs

### Status and general trained values

| Result | Byte offset(s) | Representation |
|---|---|---|
| `PmuRevision` | `0x02` | 16-bit firmware revision written by the PMU |
| `CsTestFail` | `0x0f` | zero means no rank failure; nonzero indicates a failure |
| CDD, channel A | `0x26`–`0x31` | 12 signed bytes |
| CDD, channel B | `0x41`–`0x4c` | 12 signed bytes |
| `TrainedVREFCA` | A0 `0x33`, A1 `0x34`, B0 `0x4e`, B1 `0x4f` | one byte per rank |
| `TrainedVREFDQ` | A0 `0x35`, A1 `0x36`, B0 `0x50`, B1 `0x51` | one byte per rank |

`InternalStatus` at `0x14` is a 16-bit output field documented as RFU; no additional meaning is established here.

### Margins

Each rank has four one-byte margin results, in this order: `RxClkDly`, `VrefDac`, `TxDqDly`, and `DeviceVref`.

| Rank | Byte range |
|---|---|
| A0 | `0x37`–`0x3a` |
| A1 | `0x3b`–`0x3e` |
| B0 | `0x52`–`0x55` |
| B1 | `0x56`–`0x59` |

Each value is the smallest distance from the trained center to the failing region in that dimension. A small value therefore indicates little margin, but these documents do not establish a universal pass/fail threshold.

### Additional LPDDR5 results

| Rank | `TrainedVREFDQU` | `TrainedDRAMDFE` | `TrainedDRAMDCA` |
|---|---:|---:|---:|
| A0 | `0xd0` | `0xd1` | `0xd2` |
| A1 | `0xd5` | `0xd6` | `0xd7` |
| B0 | `0xda` | `0xdb` | `0xdc` |
| B1 | `0xdf` | `0xe0` | `0xe1` |

`TrainedDRAMDFE` uses bit fields `[6:4]` and `[2:0]`; `TrainedDRAMDCA` uses `[7:4]` and `[3:0]`. Quickboot copies these values, and the CA/DQ VREF results, into the message's mode-register shadow entries.

The quickboot-related PLL outputs are `QBPllUPllProg0..3` at `0x130`, `0x132`, `0x134`, and `0x136`, followed by `QBPllCtrl1`, `QBPllCtrl4`, and `QBPllCtrl5` at `0x138`, `0x13a`, and `0x13c`. When `Pstate & 0x20` is non-zero (`Pstate = 0x30` on the reference profile), `DevInit` populates these seven 16-bit words by direct CSR read-back from `CSR[0x020088]`, `0x020089`, `0x02008a`, `0x02008c`, `0x02004a`, `0x02004b`, and `0x02004c` (`ARC 0x90040110`–`0x90040118` and `0x90040094`–`0x90040098`; see [05-message-block.md](05-message-block.md)).

## Observed: exact `phypara` serialization

`phypara` is **not** a linear CSR dump. It consists of exactly **69 segments and 5170 halfwords**. A segment is either:

- **scattered:** read each explicitly listed offset relative to the segment base, in list order; or
- **sequential:** read `count` consecutive halfwords beginning at the base, with stride one.

Values are appended to the output in segment order. Duplicate segment bases are intentional because different segments select disjoint offset subsets within the same base; across all 69 segments, all 5,170 CSR indices are unique, and position in the serialized stream is significant. Segments must not be sorted, merged, or reordered. Segment 48 is the largest contiguous group, reading 1,536 halfwords from base `0x90000` starting at offset `46` (`0x9002e`–`0x9062d`); segments 54 and 55 read 96 sequential halfwords each from base `0x20000` starting at offsets `292` (`0x20124`–`0x20183`) and `395` (`0x2018b`–`0x201ea`), respectively.

The authoritative complete list of bases, counts, and scattered/sequential offsets is [appendix-phypara-order.md](appendix-phypara-order.md). The count column totals 5170. Collection and restoration must use the same order.

### Classification of the 5,170 `phypara` entries by origin

Cross-referencing the 5,170 serialized `phypara` CSR indices against the host pre-training tables, the PMU training execution, and the host post-training table on the reference 6400 MT/s dual-rank x16 platform partitions every entry into five disjoint classes:

| Origin class | Count | Share | Description |
|---|---:|---:|---|
| `host-untouched-nz` | 2,162 | 41.8% | Never written by host pre/post tables, but read back **non-zero** after training (primarily ACSM/sequencer regions `0x90000` with 1,039 entries and `0x40000` with 415 entries, plus per-lane defaults in `0x10000`) |
| `post-table` | 1,602 | 31.0% | Written by the host post-training table (`phyinit_lp5_training()`) and preserved unchanged in `phypara` (primarily `0x90000` with 813 entries, `0x70000` with 328, and `0x60000` with 165) |
| `host-untouched-zero` | 965 | 18.7% | Never written by host tables and read back `0x0000` after training |
| `pre-unchanged` | 334 | 6.5% | Written by the host pre-training table + overrides (`phyinit_lp5_pre_training()`) and **not** modified by PMU training |
| `CHANGED` | 107 | 2.1% | Written by the host pre-training table and **modified during PMU training** |

The **107 `CHANGED` entries** (comparing the base pre-training table + topology override against the captured post-training `phypara`) break down by role as follows:

- **Trained per-lane read-eye delay centers (`0x01<r><l>4e` and `0x01<r><l>4f`, 70 entries changed from `0x0035`)**: Across the 36 lanes (`4 ranks × 9 lanes = 72` total `0x4e`/`0x4f` registers), 70 entries moved from their `0x0035` pre-training seed to `0x002d`–`0x004a` during `RdDQS` (`SequenceCtrl[3]`), while 2 entries (`rank 0, lane 0, 0x4f` and `rank 0, lane 8, 0x4e`) happened to converge back to `0x0035`. The companion registers `CSR[0x01<r><l>50]` and `CSR[0x01<r><l>51]` are not computed during `DevInit`: instead, the PMU saves their host pre-training table values (`0x0035`) into its per-lane context structure (`+0xc4`/`+0xc6`) and restores them unchanged, so all 72 `0x50`/`0x51` entries remain at `0x0035` (`pre-unchanged`) when `DisableTrainingLoop[1]` (`0x80000102 & 2`) is `0`.
- **Host `io_override` TX impedance entries (`0x01<r>040, 42..45` and `0x03<r>040..43`, 28 entries)**: Rank-global TX drive strength registers on `lane 0` updated from the base pre-training table (`0x0c0c` / `0x0e0e`) to `0x0f0f` (`PHY_R_30`, 30 Ω) by the host's electrical `io_override` pass before PMU start.
- **Master clock-period calibration and control in `DevInit` (`0x02xxxx`, 5 entries changed from pre-training table)**: `0x02000a` (CA period updated from `0x0040` to **`0x0082`**, or `0x0086` in `DevInit`-only runs; note that `0x020017` remains `0x0040` in `pre-unchanged`), `0x020011` (pre-training `0x006f` saved to DMEM `0x8000042a` and masked `& 0xff9f` to **`0x000f`**), `0x020012` (updated from `0xa1a1` to **`0x7353`**), `0x020186` (updated from `0x00a1` to **`0x0073`**), and `0x020187` (updated from `0x0040` to **`0x0041`**, copied from `CSR[0x0100d5]`).
- **ACSM/sequencer, `MemResetL`, and status updates (5 entries)**: `0x09080b` (`ARC 0x90121016`, `0x002f & 0xff9f` → `0x000f`), `0x09080c` (`ARC 0x90121018`, `0x006f & 0xff9f` → `0x000f`), `0x020060` (`MemResetL`, `0x0000` → `0x0002` → `0x0003` → `0x0001`), `0x020077` (`0x0001` → `0x0000`, cleared by hardware when `0x02f077 = 0` is strobed), and `0x0c0086` (`ARC 0x9018010c`, `0x0000` → `0x0002` after full training or `0x0003` after `DevInit`; hardware/host status register with 0 references in PMU firmware).

In addition, per-lane and per-rank trained CSRs in group `0x01` that are **not** present in the host pre-training table—such as write-leveling `0x0c`/`0x0d` (`0x009a`/`0x009b`), write DQ delays `0x10..0x13` (`0x0105`–`0x011a`), read/write offsets `0x1c..0x1f`, `0x24..0x27`, per-bit DCA delays `0x68..0x6f` (shifted by ~+4 taps during `DRAM DCA`), `0x01<r>0ad` (`0x0002` at `ARC 0x9002015a + r * 0x2000`, written during `Step 3.6` per-rank DQ calibration), `0x01<r>0d9` (`0x009c` = `156`, written across all active ranks right after `0x020187`), and `0xaa`/`0xb5`/`0xbe`—are classified under `host-untouched-nz` (written from `0x0000` by the PMU during training). Note that `CSR[0x01<r>0a3]` (`0x0b33` across `r = 0..3`, `2867` decimal in the first `phypara` slot) is written by the host pre-training table *before* the PMU starts, saved to DMEM `0x80000430`–`0x80000437`, temporarily masked `&= 0x000c` during `DevInit` Phase 3, and restored to `0x0b33`, so `0x01<r>0a3` belongs to `pre-unchanged`.

### Measured `CSR[0x4e]` / `CSR[0x4f]` read-eye center table (6400 MT/s reference)

On the reference 6400 MT/s dual-rank x16 board (`TX DS = 30 Ω`, `RX ODT = 60 Ω`), the 36 lanes (`rank = 0..3`, `lane = 0..8`) start at `0x35/0x35` before training and converge to the following `0x4e/0x4f` hex pairs (`0x2d`–`0x4a`, mean ~`0x3e.5` = 62.5 taps):

| Rank (`0x01<r><l>4e/4f`) | `L0` | `L1` | `L2` | `L3` | `L4` | `L5` | `L6` | `L7` | `L8` |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Rank 0 (`0x010<l>4e/4f`) | `34/35` | `42/3a` | `3c/43` | `3d/3e` | `49/3e` | `38/3c` | `44/43` | `3e/3b` | `35/38` |
| Rank 1 (`0x011<l>4e/4f`) | `43/3d` | `3c/42` | `3a/49` | `36/41` | `47/3f` | `40/40` | `43/45` | `44/3a` | `3a/3f` |
| Rank 2 (`0x012<l>4e/4f`) | `44/3d` | `38/3a` | `42/3d` | `3b/2d` | `3f/40` | `48/3e` | `44/3f` | `3e/38` | `3a/43` |
| Rank 3 (`0x013<l>4e/4f`) | `42/43` | `4a/3d` | `3a/3e` | `47/40` | `45/44` | `38/46` | `40/38` | `3d/3f` | `41/37` |

### Hardware power-on mirror group `0x110000` (`ARC 0x90220000`)

Outside the 69 `phypara` segments (which span groups `0x01` through `0x0c`), the DPHY exposes a secondary register block at CSR index base `0x110000` (`ARC 0x90220000`, `host DPHY_BASE + 0x440000`) with a per-lane layout mirroring group `0x010000` (`1,024` non-zero registers out of `1,180` sampled offsets). Neither the host pre/post-training tables nor the PMU firmware write to `0x110000`–`0x113fff`, and quickboot restores full functionality from `phypara` + `ACSM` without touching `0x110000`. The non-zero values in `0x110000` are therefore hardware power-on reset state (or read-only hardware shadow state) and require no software initialization.

During quickboot, the saved `phypara` byte stream is copied sequentially to DMEM index `0x58200`. The PMU consumes that serialized representation; the host does not individually scatter the values back to PHY CSRs in this path.

## Observed: collection sequence

After the training table completes, the loader:

1. sets bit 6 of PHY CSR `0x20037`;
2. selects host access and the required clocks;
3. reads all `0x140` message bytes from DMEM `0x58000`;
4. serializes the 5170 `phypara` halfwords using the 69-segment table;
5. reads 4096 contiguous ACSM halfwords from `0x41000`; and
6. reduces the clocks after collection.

Collection is per controller. DDR0 and DDR1 each produce an independent `0x49a4`-byte result record, even when their bytes happen to match.

The PMU publishes major message `0x07` on normal completion. `CsTestFail` and a completed mailbox exchange are useful checks, but persistence policy should additionally validate the resulting memory before replacing a known-good saved result.

## Observed: reference-board sample

The captured LPDDR5 envelope from the two-rank, 6400 MT/s reference board (`MT62F2G32D4DS`, envelope header `magic = 0x54524444`, `crc32 = 0x0f7e240d`, `chipid = 0x2000000000000000`, `version = 0`, `type = 1`, `cs_num = 2`, `data_rate = 6400`; controller records at envelope offsets `0x80` and `0x4a24`) contains the following per-controller values:

| Property | DDR0 | DDR1 |
|---|---|---|
| `MsgMisc` | `0x01` | `0x01` |
| `HdtCtrl` | `0x04` | `0x04` |
| Nonzero `phypara` entries | 3926 of 5170 | 3926 of 5170 |
| Nonzero ACSM entries | 1845 of 4096 | 1845 of 4096 |

The first twelve `phypara` halfword values in each record are:

`2867, 0, 156, 0, 1, 2, 3, 4, 5, 6, 7, 2879`

The first twelve ACSM halfword values in each record are:

`0, 0, 0, 0, 0, 0, 0, 0, 51672, 0, 0, 0`

The two captured records are byte-identical. This is a sample observation, **not** a format requirement or a reason to alias the controllers' storage.

## Inferred: practical sanity checks

A result with almost all-zero `phypara` or ACSM data is suspicious when compared with the reference sample, but nonzero counts alone do not prove that training succeeded. Sensible validation includes completion status, `CsTestFail`, structural sizes, and a real memory test. Values and margins can legitimately vary across boards, memory parts, runs, voltage, and temperature.

## Unknown or not established

- The detailed meaning of many message fields and PHY CSRs remains unknown; unknown bytes must still be preserved.
- No universal numerical acceptance thresholds are established for the margin fields.
- The captured equality of DDR0 and DDR1 does not prove that all boards or training runs produce identical records.
- A successful training protocol exchange does not, by itself, prove long-term memory stability.
