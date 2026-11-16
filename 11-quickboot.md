<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Persistence & Selection](10-persistence-and-selection.md) | [Index](README.md#reading-order-and-contents) | [Next: ARC Execution Environment →](12-arc-execution-environment.md)

# Quickboot

## Scope and evidence

Quickboot reuses a previous training result instead of requesting the normal calibration sequence. The flow, offsets, and mappings here are **observed** in the loader and its message layout. Recommendations and unresolved behavior are separated at the end.

Quickboot is valid only after the complete persistent envelope passes the magic, memory-type, rank-count, data-rate, and full-envelope CRC checks described in `10-persistence-and-selection.md`. An invalid or incomplete result must lead to full training.

On the reference board, accepted quickboot took approximately **9 ms**, compared with approximately **5.6 seconds** for full training. These timings are measurements from that board, not guaranteed constants.

## Observed: prerequisites and high-level behavior

Before quickboot-specific work begins, DDR clock and DDRC configuration follow the common initialization path. The quickboot IMEM and additional DMEM images must also be available.

Quickboot does **not** apply the normal pre-training and training tables and does not request a new training sequence. Each controller is initialized sequentially using its own saved `0x49a4` result record.

The host modifies a working copy of the saved message:

- `SequenceCtrl = 1`, requesting only device initialization (`DevInit`); and
- `Quickboot = 1`, selecting result reuse.

The normal training bits are therefore not requested. The host also copies selected trained outputs into mode-register shadow entries before restoring DMEM.

## Observed: source-result to MR-shadow mapping

All offsets below are byte offsets within the `0x140`-byte message. The left columns are the **trained result sources**; the right columns are the **MR-shadow destinations** written by the host before DMEM restoration. Rank order is always **A0, A1, B0, B1**.

| Trained source | Source offsets, A0/A1/B0/B1 | MR shadow | Destination offsets, A0/A1/B0/B1 |
|---|---|---|---|
| `TrainedVREFCA` | `0x33`, `0x34`, `0x4e`, `0x4f` | MR12 | `0x6e`, `0x6f`, `0x70`, `0x71` |
| `TrainedVREFDQ` | `0x35`, `0x36`, `0x50`, `0x51` | MR14 | `0x76`, `0x77`, `0x78`, `0x79` |
| `TrainedVREFDQU` | `0xd0`, `0xd5`, `0xda`, `0xdf` | MR15 | `0x7a`, `0x7b`, `0x7c`, `0x7d` |
| `TrainedDRAMDFE` | `0xd1`, `0xd6`, `0xdb`, `0xe0` | MR24 | `0x9a`, `0x9b`, `0x9c`, `0x9d` |
| `TrainedDRAMDCA` | `0xd2`, `0xd7`, `0xdc`, `0xe1` | MR30 | `0xae`, `0xaf`, `0xb0`, `0xb1` |

The copy is byte-for-byte; the host does not recalculate the training values. `TrainedDRAMDFE` uses fields `[6:4]` and `[2:0]`, and `TrainedDRAMDCA` uses `[7:4]` and `[3:0]`.

These destinations are message-resident shadow values. They are not five immediate host writes to the DRAM. The restored quickboot firmware subsequently consumes the message and applies the saved configuration.

## Observed: DMEM and SRAM restore semantics

The saved result is restored into different runtime regions at different points in the sequence:

| Saved component | Runtime destination | Restore semantics |
|---|---|---|
| Modified message, `0x140` bytes | DMEM index `0x58000` | Copy the complete message after setting quickboot controls and MR shadows |
| `phypara`, `0x2864` bytes | DMEM index `0x58200` (`0x58200`–`0x59631`) | Copy all 5170 halfwords sequentially (`ARC 0x80000400`–`0x80002c63`), preserving the 69-segment serialized order |
| Additional quickboot DMEM image | DMEM index `0x5aa00` (`0x5aa00`–`0x5acc5`) | Zero `[0x59632 .. 0x5aa00)`, then load the 1,420-byte (710-halfword) quickboot image at `0x5aa00` (`ARC 0x80005400`–`0x8000598b`) |
| ACSM, `0x2000` bytes | ACSM SRAM index `0x41000` | Restore 4096 contiguous halfwords after stopping the PMU |

The serialized `phypara` stream ends at index **`0x59632`** (`0x58000 + 0x200 + 5170`), where `0x200` is the reserved message area in halfwords (larger than the active `0xa0`-halfword message). Rather than placing the extra quickboot table immediately at `0x59632`, the host clears `[0x59632 .. 0x5aa00)` to zero, loads the extra DMEM image starting at **`0x5aa00`** (`ARC 0x80005400`, matching the quickboot PMU's global pointer **`gp = 0x80005400`**), and clears `[0x5acc6 .. 0x60000)` to zero.

The `phypara` stream is not scattered over MMIO CSRs by the host: the quickboot PMU firmware reads `0x80000400` sequentially using its initialized `phypara` cursor in `gp`-relative DMEM.

In the reference interface, the additional quickboot DMEM payload at `0x5aa00` (`ARC 0x80005400`–`0x8000598b`) is a fixed **1,420-byte (`0x58c`, 710-halfword, `491` non-zero)** firmware data/script table delivered compressed (LZOP) in the loader (not part of the per-boot saved `DDRT` envelope). Three key offsets inside that `gp = 0x80005400` block are used directly by the quickboot PMU:

- **Halfwords `14`–`15` (`ARC 0x8000541c`, byte offset `0x1c`)**: expected PHY canary words `0x6745`, `0x00f2`;
- **Byte offset `0x78` (`ARC 0x80005478`, 32-bit pointer)**: initialized to **`0x80000400`** (DMEM `0x58200`), serving as the monotonic `phypara` deserialization cursor across the quickboot firmware's three restore passes;
- **Byte offset `0xc8` (`ARC 0x800054c8`, `0x12` bytes = three 6-byte `[int32 csr_idx_or_neg1, int16 val]` script records)**: executed by the pre-reset topology helper to write **`CSR[0x02006e] = 0x0001`**, **`CSR[0x070060] = 0x0000`**, and **`CSR[0x070011] = 0x0000`** (`0xFFFFFFFF` in the 32-bit index field selects a delay record instead of a CSR store).

Clearing the surrounding gaps (`[0x59632 .. 0x5aa00)` and `[0x5acc6 .. 0x60000)`) is part of the runtime environment contract; an implementation must not leave stale training data in those ranges.

PSTATE SRAM at index `0xa0000` has restoration support in the architecture, but that restoration is commented out in the observed current flow. It is **not** restored by this quickboot sequence.

ACSM restoration is deliberately separate from DMEM restoration. The loader stops the PMU and clears bit 6 of `0x20037` before restoring the contiguous ACSM contents. It does not write ACSM to DMEM `0x58200` or include it in the extra DMEM image.

## Observed: ordered quickboot flow

1. Set `SequenceCtrl = 1` and `Quickboot = 1` in the working message.
2. Populate MR12, MR14, MR15, MR24, and MR30 shadows from the trained results using the exact table above.
3. Run MemReset and wait **30 microseconds plus 2 milliseconds**.
4. Set clocks to 7, select APB-only operation, and disable ECC.
5. Load quickboot firmware into IMEM at PHY index `0x50000`.
6. Restore the complete modified message at DMEM `0x58000` and serialized `phypara` at `0x58200`.
7. Clear `[0x59632 .. 0x5aa00)`, load the additional 710-halfword DMEM table at **`0x5aa00`**, and clear `[0x5acc6 .. 0x60000)`.
8. Release the PMU with `MICRO_RESET = 9, 1, 0` and consume/acknowledge its mailbox messages.
9. Stop the PMU, clear `0x20037` bit 6, and restore ACSM at `0x41000`.
10. Write `0xd00e7 = 0x0600`, set clocks to 2, and set the microcontroller mux to 1.

MemReset uses PHY indices `0x20090` and `0x20060`, plus timing values at `0x3f0a2`, `0x3f042`, `0x3f043`, and `0x3f0a3` (`0xf00`, `0xf0f`, `0x0f0f`, and `0x800`, respectively). APB-only and ECC-disable operations use `0xd0036 = 0` and `0xc0086 = 1`.

The normal completion message is major message `0x07`. Mailbox waits in the reference flow have no useful timeout guarantee, so a stalled PMU can stall boot.

## Observed: what quickboot does not do

- It does not execute normal pre-training/training tables.
- It does not perform a fresh result collection.
- It does not repair an incompatible or corrupt saved result.
- It does not restore PSTATE SRAM in the current flow.
- It does not inherently rewrite the persistent envelope.

The saved message returns through the reuse flow with its trained values; the quickboot control changes belong to the runtime working copy and are not evidence of a new training run.

## Observed: quickboot PMU internal execution sequence

Unlike the full-training firmware (`gp = 0x80000400`), the dedicated quickboot PMU image (`25,756` bytes decompressed) enters at `0x1fc` with `sp = 0x80010000` and **`gp = 0x80005400`** (`DMEM 0x5aa00`). Internally, it **forces `SequenceCtrl = 1` (`0x80000010 = 1`) and `Quickboot = 1` (`byte[0x80000019] = 1`)**, skipping all `WrLvl`, `RxEn`, `RdDQS`, `WrDq`, and `DRAM DCA` training sweeps as well as the long `200 ms + 2 s` `tINIT` wait. Running full-training stage `SequenceCtrl = 0x0007` (`DevInit + WrLvl + RxEn`) is therefore **not** equivalent to what the quickboot PMU executes.

Instead, quickboot re-aligns the PHY by advancing the monotonic `phypara` cursor (`ARC 0x80005478`, initialized to `0x80000400` / DMEM `0x58200`) across **three deserialization passes** interleaved with DRAM reset, clock/PLL switching, mode-register writes, and hardware DLL/clock recalibration:

1. **Phase 0, commit strobe, canary, and QBPll**:
   - Evaluates the `MsgMisc[0]` (`byte[0x80000001] & 1`) startup branch (`CSR[0x03f01a] = 0`, `CSR[0x01f01a] = 0`, `CSR[0x0200aa] = 0`, `CSR[0x02001a] = 3`) and issues the commit pulse on **`CSR[0x02f052]` (`ARC 0x9005e0a4`)**: when `MsgMisc[0] == 1`, it writes **`0x02f052 = 1` -> `delay(10)` -> `0x02f052 = 0`** (ensuring the write to `1` precedes the delay so the pulse has non-zero width; repeated 6 times across the flow).
   - Verifies the `0x6745`/`0x00f2` hardware canary, writes `PmuRevision = 0x2409`, derives frequency/timing multipliers (`0x80005b4c`/`5b4d`), saves `CSR[0x020009]`, and loads QBPll fields from message block `0x130`–`0x13d` (`CSR[0x020088..8a, 8c]`, `CSR[0x02004a..4c]`).
2. **`phypara` Pass 1 (early rank/CA/control restoration + `0x0c0086`)**:
   - Forces `Quickboot = 1` (`byte[0x80000019] = 1`) and `SequenceCtrl = 1` (`0x80000010 = 1`) around the early restore.
   - Restores per-rank group `0x01` entries (`0x01<r>0a3, 56, d9, 80..8c, 0e`, 11 halfwords per rank), CA `0x06015a/5c`, `0x060120..2c` (+ optional `0x061dde`), direct group `0x02` (`0x020090, 51, 77, 49, 71, 84, 85, a5, c6, 19`, `0x020184..8a`, `0x020200`, `0x020300..27`), `0x0908ff`, `0x0c00f0..ff`, and **`CSR[0x0c0086]` (`ARC 0x9018010c`)** directly from the `phypara` stream.
3. **`phypara` Pass 2 (CA/DQ per-lane early restoration)**:
   - Restores per-channel/lane CA CSRs (`CSR[0x030001 + (ch << 11) + (x << 20) + (lane << 8)]` for `lane = 0..9`, plus `0x030006, 0x030008, 0x030002, 0x030005`, consuming `14` halfwords per sub-iteration).
   - Restores per-rank/lane DQ CSRs with an outer stride of **`47` halfwords (`0x2f`) per lane** (`94` per pair, `188` per rank): `11` CSR offsets in the outer stride are specified here (`0x010028, 0x01005c, 0x01005e, 0x010060, 0x010062, 0x010064, 0x010066, 0x01002a, 0x010015, 0x010020, 0x01005a`), while the remaining **`36` intermediate halfwords** are consumed in nested `4`-halfword inner loops (not yet enumerated pair-by-pair).
4. **DMEM 6-byte startup script & DQ block reset**:
   - Copies `0x12` bytes from `ARC 0x800054c8` and executes the 3 `(int32 idx, int16 val)` records: **`CSR[0x02006e] = 0x0001`**, **`CSR[0x070060] = 0x0000`**, **`CSR[0x070011] = 0x0000`**.
   - Programs `CSR[0x02008b]` (`Pstate` bit 0 when bit 7 clear), `CSR[0x020072] = CsPresentChA`, `CSR[0x020073] = CsPresentChB`, and executes the DQ block reset sequence (`CSR[0x020303] |= 0x10`, clears `0x020310` and `0x02030d`, pulses `CSR[0x020311] = 1 -> delay(130,000,000, 0) -> 0`, clears `0x020320`, restores `0x020303`).
5. **DRAM reset, clock/PLL transitions, and mode-register writes (MRW)**:
   - Runs the two-branch DRAM reset/`tINIT` sequence (`byte[0x80000005] == 0` full path vs. non-zero short path: `CSR[0x01f00f] = (0x01000f & 0xff78) | 6`, `0x020061 = 1 -> 0`, `0x0200e2 = 3 -> 2 -> 0`, `0x020200 = 0x33`, `0x020078` and `0x0200a2` reset/clock toggles, ending with `CSR[0x0200f8] = 0`).
   - Enables and switches clocks (`CSR[0x0200a2] = (0x0200a2 & 0xfffc) | 1`, `delay(4)`, `0x0200f8 = 1`, `delay(0x1f)` before DRAM init; `delay(0x1f)`, `0x0200f8 = 0`, `delay(4)`, `0x0200a2 = (0x0200a2 & 0xfffc) | 2` to switch to PLL).
   - Skips the `200 ms + 2 s` `tINIT` wait because `Quickboot == 1`, constructs the `0x36`-byte/channel MR table from message-block MR bytes (`0x5a..0x9a`, including host-populated MR12/14/15/24/30 shadows), submits CA MRWs per active rank and broadcast (`cs = 0xf`), issues explicit `MR7`, clears/sets opcode bits, issues per-channel `MR6` (`0x22, 0x40` and `<byte> | 2`), `MR40` (`0x02`), and `MR7` (`0x1e`), and clears `CSR[0x02006e] = 0`.
6. **Hardware DLL / clock recalibration (core per-boot re-alignment)**:
   - After `delay(0, 0x40)`, asserts `CSR[0x02f310] = 1` (`ARC 0x9005e620`) and `CSR[0x02f0f8] = 1` (`ARC 0x9005e1f0`).
   - On the primary calibration path, computes `u = CSR[0x020049] & 0x01f0`, toggles `CSR[0x02f049] = u | 0x0400` (`ARC 0x9005e092`), waits `delay(1000)`, sets `CSR[0x02f0e0] = 1` (`ARC 0x9005e1c0`), waits `delay(1000000)`, writes `CSR[0x02f049] = u | 0x0200`, sets `CSR[0x02f0c3] = 1` (`ARC 0x9005e186`), clears `CSR[0x02f0e0] = 0`, and copies **`CSR[0x02004e + pstate] = CSR[0x0200cf] & 0x001f`** (`ARC 0x9004009c <- 0x9004019e & 0x1f`) (on the alternate branch, sets `CSR[0x02f0c3] = 1` directly).
   - Pulses `ARC 0x9013e050 = 1` or `0x9013e05a = 1` (`CSR[0x09f028]`/`0x09f02d` depending on hardware variant) and restores **`CSR[0x02f009 + pstate]` (`ARC 0x9005e012`)** from the initial `CSR[0x020009]` reading.
7. **`phypara` Pass 3 (post-DLL override restore) and completion**:
   - **After** DLL recalibration finishes, deserializes the remaining `phypara` segments sequentially: `CSR[0x020088, 89, 8a, 8c]`, `CSR[0x02004c, 4a, 4b]`, CA group `0x0300xx`, **`CSR[0x090803..0x09081f]`** and `CSR[0x090903..0x090938]` (including **`CSR[0x090808]`** and **`CSR[0x090813]`**, which is why restoring them before DLL calibration fails), `CSR[0x09001d..0x09002d]`, `CSR[0x090701..0x090708]`, and multi-pstate/rank/lane DQ loops (including **`0x01<r><l>4e/4f/50/51`** at `ARC 0x9002009c..a2`).
   - Streams margin fields (`0x37`–`0x3e`, `0x52`–`0x59`), triggers final commit strobe, and emits major completion **`0x07`**.

| CSRs restored directly at Pass 3 entry blocks *(offsets relative to each sub-block's cursor)* | CSR | Cursor offset | CSR | Cursor offset |
|---|---|---:|---|---:|
| Master clock / Pll | `0x020088` | `[0]` | `0x090811`..`0x090819` | `[0xc]`..`[0x14]` |
| | `0x020089`, `0x02008a`, `0x02008c` | `[1]`, `[2]`, `[3]` | `0x09081b`..`0x09081f` | `[0x16]`..`[0x1a]` |
| ACSM / PHY clock dividers | `0x090804`, `0x090806`, `0x090807`, **`0x090808`**, `0x090809`, `0x09080b`..`0x09080e` | `[1]`, `[3..10]` | `0x090903` | `[0x1b]` |
| Latency / calibration blocks | `0x09001d`..`0x09002d` | `[0x601]`..`[0x611]` | `0x090701`..`0x090708` | `[0x613]`..`[0x61a]` |

*(Note: `[n]` indices in the left and right groups belong to successive monotonic cursor positions within Pass 3, not a single shared base.)*

## Inferred: why direct host-only CSR restoration cannot substitute for quickboot

The reference quickboot sequence does **not** restore the 5,170 `phypara` halfwords over PHY MMIO CSRs from the host. Attempting to emulate quickboot by running full `DevInit` (`SequenceCtrl = 1`, `Quickboot = 0`) and writing saved `phypara` entries directly from the host over PHY MMIO omits three mandatory hardware steps performed only by the quickboot path:

1. **Quickboot PMU firmware execution (`lp5_quickboot` image + DMEM `0x58200` / `0x5aa00`)**: The host places the modified `0x140`-byte message at DMEM `0x58000` (with reserved `0x200`-halfword stride), the serialized 5,170-halfword `phypara` payload at **DMEM `0x58200`**, and the 710-halfword quickboot table at **DMEM `0x5aa00`** (`ARC 0x80005400`, with zero-fill across `[0x59632 .. 0x5aa00)` and `[0x5acc6 .. 0x60000)`), then executes the dedicated quickboot PMU image (`MICRO_RESET = 9, 1, 0` → completion `0x07` / failure `0xff`). Only the quickboot PMU firmware interleaves the three `phypara` deserialization passes with DRAM reset, MRW bursts, and the **`0x02f049`/`0x02f0e0`/`0x02f0c3` hardware DLL/clock recalibration** before applying **Pass 3 (`0x090808/13`, `0x01<r><l>4e..51`)**. Without running that ordered sequence, PHY FIFO/DLL pointers remain misaligned with the current boot clock phase, causing intermittent per-byte-lane read failures (such as a burst/word-offset shift on lane 1 or `0xff`/bit flips on lane 2; ~3/10 boots succeed purely by luck).
2. **Pre-PMU MR-shadow updates (`MR12`, `MR14`, `MR15`, `MR24`, `MR30`)**: Before starting the quickboot PMU firmware, the host must populate the `MR12`/`MR14`/`MR15`/`MR24`/`MR30` message-block shadows from the trained VREF/DFE/DCA outputs so the quickboot firmware applies trained DRAM mode registers (restoring ACSM after halt does not replace those pre-PMU shadow updates).
3. **Clearing `CSR[0x020037]` bit 6 before ACSM restoration**: Result collection sets `CSR[0x020037] |= BIT(6)`, and quickboot clears `CSR[0x020037] &= ~BIT(6)` after stopping the PMU and before writing ACSM (`0x41000`); leaving bit 6 set leaves the PHY in the wrong state for ACSM operation.

For diagnostic experiments that still inspect or patch individual CSRs directly over MMIO after `DevInit` (`SequenceCtrl = 1`, `Quickboot = 0`), registers split into three rules:

1. **Keep live or recompute from the current boot (never overwrite from an envelope)**:
   - `CSR[0x090808]` and `CSR[0x090813]`, derived from the live non-`phypara` period registers `CSR[0x020314]` and `CSR[0x020315]` (`(val << 8) | 1`);
   - `CSR[0x02000a]`, `CSR[0x020017]`, `CSR[0x020187]`, `CSR[0x020011]`, and `CSR[0x0c0086]`;
   - **`CSR[0x01<r>0ad]`** (`2` after `Step 3.6`) and **`CSR[0x01<r>0af]`** (live calibration readout from `CSR[0x01<r>0d6]`: `0x38`–`0x3b` in SPI vs. `0x3a`–`0x40` in FDL, so overwriting `0x01<r>0af` from an SPI envelope on an FDL boot injects 2–5 taps of error).
2. **Keep `DevInit` exit values intact (skip overwriting)**:
   - `CSR[0x020020] = 0`, `CSR[0x020027] = 0`, `CSR[0x020028] = 1`, `CSR[0x02002a] = 1`, `CSR[0x020031] = 0`, `CSR[0x020041] = 0`, `CSR[0x020060] = 3` (`MemResetL`), and `CSR[0x020078] = 0`. Nothing in DDRC activation requires changing these after `DevInit`. *(Other strobe/period CSRs such as `0x020021`, `0x0200a2`, `0x0200e3`, `0x0200e5`, `0x020314`, `0x020315`, `0x02f0a0`, `0x02f0a5`, `0x02f077`, and `0x01f0xx` are absent from the saved `phypara` table.)*
3. **Trained per-lane delays and commit triad granularity**:
   - `CSR[0x01<r><l>4e]`/`4f`, `CSR[0x01<r>00f]`, `CSR[0x01<r>014]`, and `CSR[0x01<r>057]` must match the active RX ODT (`CSR[0x010048/4a/4c/4d]`) and boot path. When emitting the configuration commit triad (`CSR[0x020028] = 1` → `CSR[0x02f0a5] = 1` → `CSR[0x02f077] = 0`, strict order, no wait between the three stores), issue it once per **logical CSR block** (`0x01`, `0x02`, `0x03`, `0x07`, `0x09`, `0x0c`, `0x0d`) rather than after each of the 69 serialization segments.

## Inferred: implementation safeguards

A replacement implementation should validate the envelope before touching the PHY, use a separate mutable message copy, and preserve the exact source/destination rank order. It should restore all three saved components rather than treating the MR mappings as a substitute for `phypara` or ACSM.

Adding bounded mailbox/PMU waits and a fallback to full training is prudent, but that is new policy. A fallback after arbitrary PMU corruption may require a full hardware reset; it must not be assumed to recover every halted state.

After quickboot, memory should still be validated when diagnosing instability. An intact CRC proves consistency of stored bytes, not that those coefficients remain suitable after a board, memory-device, firmware, voltage, or temperature change outside the checked metadata.

## Unknown or not established

- CRC and metadata validation cannot establish the physical suitability of old coefficients under every changed operating condition.
- The documents do not define a universal quickboot completion deadline or a guaranteed in-place recovery sequence after a PMU hang.
- The exact `36` intermediate DQ halfwords consumed inside the nested `4`-halfword inner loops of `phypara` Pass 2 (between the `11` named outer indices of the `47`-halfword per-lane stride) and the pair-by-pair serialization layout inside Pass 3's multi-P-state/rank/lane branches (`0x02004a..4c`, `0x0300xx`, `0x0909xx`, `0x0100xx`) have not yet been fully enumerated; the remainder of the `710`-halfword table at `0x5aa00` outside `[0x1c..0x1f]`, `[0x78..0x7b]`, and `[0xc8..0xd9]` should still be loaded opaquely.
- The semantic details of all restored ACSM entries remain partially open; their complete opaque preservation is nevertheless required.
- PSTATE SRAM restoration is absent from the observed flow, and its necessity for other firmware/configurations is not established.
- The reference timing does not guarantee the same latency on other boards or software stacks.
