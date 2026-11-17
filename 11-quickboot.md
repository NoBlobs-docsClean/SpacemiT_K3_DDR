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

Unlike the full-training firmware (`gp = 0x80000400`), the dedicated quickboot PMU image (`25,756` bytes decompressed) is a standalone program entering at `0x1fc` (`sp = 0x80010000`, **`gp = 0x80005400`** / DMEM `0x5aa00`) rather than executing `07-training-behavior.md`'s full-training `Step 3.1–6.3` dispatcher (`Quickboot != 0` branches). Internally, it **forces `SequenceCtrl = 1` (`0x80000010 = 1`) and `Quickboot = 1` (`byte[0x80000019] = 1`)**, skips all `WrLvl`, `RxEn`, `RdDQS`, `WrDq`, and `DRAM DCA` training sweeps as well as the `200 ms + 2 s` `tINIT` wait, and **never executes `Step 3.6` per-rank DQ calibration** (`CSR[0x01<r>0d5]` has `0` references in the quickboot firmware). Consequently, neither `SequenceCtrl = 0x0007` (`DevInit + WrLvl + RxEn`) nor running full-training `DevInit` with `Quickboot = 1` (which executes `Step 3.6` and overwrites `CSR[0x01<r>0ad]`/`0x01<r>0af`) is equivalent to the quickboot PMU sequence.

Instead, quickboot re-aligns the PHY by advancing the monotonic `phypara` cursor (`ARC 0x80005478`, initialized to `0x80000400` / DMEM `0x58200`) across **three deserialization passes** interleaved with DRAM reset, clock/PLL switching, mode-register writes, and hardware DLL/clock recalibration. All quickboot delays use the two standard RTC helpers (`delay_dfi_clks(n)`, waiting `(n + 1) >> shift` `AUX_RTC` ticks and returning immediately if `< 0x10`, where `shift = ~(CSR[0x0c0080] >> 2) & 1` is `0` under the host's `CSR[0x0c0080] = 7` setting or `1` when `0x0c0080 = 3`; and `delay_ps_or_clks(ps, min_clks)`, which takes picoseconds in argument 1 or `min_clks + 1` ticks in argument 2):

1. **Phase 0, commit strobe, canary, and QBPll**:
   - Evaluates the `MsgMisc[0]` (`byte[0x80000001] & 1`) startup branch (`CSR[0x03f01a] = 0`, `CSR[0x01f01a] = 0`, `CSR[0x0200aa] = 0`, `CSR[0x02001a] = 3`) and issues the commit pulse on **`CSR[0x02f052]` (`ARC 0x9005e0a4`)**: when `MsgMisc[0] == 1`, it writes **`0x02f052 = 1` -> `delay_dfi_clks(10)` -> `0x02f052 = 0`** (`11` ticks `< 0x10` at `shift = 0`, or `5` ticks at `shift = 1`; ensuring the write to `1` precedes the delay so the pulse has non-zero width; issued **5 times** across the flow: before canary, after Pass 1, after Pass 2/`SequenceCtrl = 1`, after Step 5 DRAM reset before MRW, and after Pass 3).
   - Verifies the `0x6745`/`0x00f2` hardware canary, writes `PmuRevision = 0x2409`, derives frequency/timing multipliers (`0x80005b4c`/`5b4d`), saves `CSR[0x020009]`, and loads QBPll fields from message block `0x130`–`0x13d` (`CSR[0x020088..8a, 8c]`, `CSR[0x02004a..4c]`).
2. **`phypara` Pass 1 (early rank/CA/control restoration + `0x01<r><l>4e..51` + `0x02000a/12/17` + `0x0c0086`)**:
   - Forces `Quickboot = 1` (`byte[0x80000019] = 1`) and `SequenceCtrl = 1` (`0x80000010 = 1`) around the early restore.
   - Restores **`46` group `0x01` base CSRs** (`0x010000, 03, 04, 06, 0c, 0d, 14, 2e, 30, 33, 35, 38, 3a, 3b, 3c, 3e, 40, 42, 43, 48, 4a, 4b, 4e..51, 56, 57, 80..87, a3, a5, ad, af, d0..d4, d9`—including per-rank `0x01<r>0a3, 56, d9, 80..87`, **`0x01<r>00c`** and its rank-1 write-leveling counterpart **`0x01<r>00d`** via a 2-iteration loop over `0x90020018 + 2*i` immediately after the 9-lane `0x4e..51` loop, **`0x01<r>0ad`** and **`0x01<r>0af`** from the envelope, RX ODT `0x010048`/`0x01004a`/`0x01004b`, and the **9-lane loop restoring `CSR[0x01<r><l>4e..51]`** at `ARC 0x9002009c..a2`), CA group `0x06015a/5c`/`0x060120..2c` (+ optional `0x061dde`), `33` CA group `0x03xxxx` base CSRs (`0x03002e..53, 90..96, ad/ae, 330, eef`), direct group `0x02` (`0x02000a/12/17` period registers from `phy[0x16..0x18]`—quickboot performs **no** period calibration measurement and **`CSR[0x020187]` has `0` references**—plus `0x020090, 51, 77, 49, 71, 84, 85, a5, c6, 19`, `0x020184..8a`, `0x020200`, `0x020300..27`), `0x0908ff`, `0x0c00f0..ff`, and **`CSR[0x0c0086]` (`ARC 0x9018010c`)** directly from the `phypara` stream (matching Segments `#0..#25` = `860` halfwords of `phypara`).
   - Because Steps 4–5 execute next, `0x020049` (`-> 0x01f0`), `0x020200` (`0x33 -> 0`), `0x020303` (RMW `(v & 0xef) | 0x10 -> v & 0xff`), `0x0200a2` (`0/1/2`), `0x020078` (`0`), `0x0200f8` (`0`/`1`), and `0x0200c3`/`0x0200e0..e2` are subsequently overwritten, while **`0x01<r>0a3` (not masked in quickboot)**, `0x01<r>056`, `0x01<r>0d9`, `0x01<r>080..87`, **`0x01<r>00c`/`0x01<r>00d`**, **`0x01<r><l>4e..51`**, **`0x01<r>0ad`/`0x01<r>0af`**, `0x02000a/12/17`, `0x06xxxx`, `0x020090/51/77/71/84/85/a5/c6/19`, `0x020184..8a`, `0x0908ff`, `0x0c00f0..ff`, and `0x0c0086` survive intact through quickboot completion.
3. **`phypara` Pass 2 (complete CA/DQ early restoration — `47` halfwords per half-iteration)**:
   - Restores the `5` Pass 2 CA base CSRs (`CSR[0x030001 + (ch << 11) + (x << 20) + (lane << 8)]` for `lane = 0..9`, plus `0x030002`, `0x030005`, `0x030006`, and `0x030008`, consuming `14` halfwords per sub-iteration).
   - Restores all **`47` halfwords (`0x2f`) per half-iteration** (`94` per pair, `188` per rank, matching Segment `#30` of [appendix-phypara-order.md](appendix-phypara-order.md)) across **`30` group `0x01` offsets** (`0x10`–`0x13`, `0x15`–`0x16`, `0x20`–`0x21`, `0x24`–`0x2b`, `0x5a`–`0x67`):
     - **First `36` halfwords (`phy[0..35]`) — 9-lane inner loop (`lane = 0..8`)**: restores `4` Write-DQ (`WrDq`) delay CSRs per lane (**`0x01<r><l>24/25`**, **`0x01<r><l>10/11`**, **`0x01<r><l>12/13`**, and **`0x01<r><l>26/27`**, even offsets in half-iteration `0` and `+1` odd offsets in half-iteration `1`);
     - **Final `11` halfwords (`phy[36..46]`)**: restores `0x01<r>05a` (`phy[0x24]`), `0x5c`, `0x5e`, `0x60`, `0x62`, `0x64`, `0x66`, `0x2a`, `0x28`, `0x15`, `0x20` in half-iteration `0`, and their `+1` odd partners (`0x5b`, `0x5d`, `0x5f`, `0x61`, `0x63`, `0x65`, `0x67`, `0x2b`, `0x29`, `0x16`, `0x21`) in half-iteration `1`.
4. **DMEM 6-byte startup script & DQ block reset**:
   - Copies `0x12` bytes from `ARC 0x800054c8` and executes the 3 `(int32 idx, int16 val)` records: **`CSR[0x02006e] = 0x0001`**, **`CSR[0x070060] = 0x0000`**, **`CSR[0x070011] = 0x0000`**.
   - Programs `CSR[0x02008b]` (`Pstate` bit 0 when bit 7 clear), `CSR[0x020072] = CsPresentChA`, `CSR[0x020073] = CsPresentChB`, **skips the `Quickboot == 0` `CSR[0x01<r>0a3] &= 0x000c` mask** (because Pass 1 already set `Quickboot = 1`, leaving `CSR[0x01<r>0a3]` untouched from the envelope), and executes the DQ block reset sequence (`CSR[0x020303] |= 0x10`, clears `0x020310` and `0x02030d`, pulses `CSR[0x020311] = 1 -> delay_ps_or_clks(130,000,000, 0) [130 µs] -> 0`, clears `0x020320`, restores `0x020303`).
5. **DRAM reset, clock/PLL transitions, and mode-register writes (MRW)**:
   - Runs the two-branch DRAM reset/`tINIT` sequence (`byte[0x80000005] == 0` full path vs. non-zero short path: `CSR[0x01f00f] = (0x01000f & 0xff78) | 6`, `0x020061 = 1 -> 0`, `0x0200e2 = 3 -> 2 -> 0`, `0x020200 = 0x33`, `0x020078` and `0x0200a2` reset/clock toggles, ending with `CSR[0x0200f8] = 0`).
   - Enables and switches clocks (`CSR[0x0200a2] = (0x0200a2 & 0xfffc) | 1`, `delay_dfi_clks(4)` [`5`/`2` ticks], `0x0200f8 = 1`, `delay_dfi_clks(0x1f)` [`32`/`16` ticks] before DRAM init; `delay_dfi_clks(0x1f)`, `0x0200f8 = 0`, `delay_dfi_clks(4)`, `0x0200a2 = (0x0200a2 & 0xfffc) | 2` to switch to PLL).
   - Skips the `200 ms + 2 s` `tINIT` wait because `Quickboot == 1`, clears the `32×` delay multiplier flag (`byte[0x80005a64] = 0`), constructs the `0x36`-byte/channel MR table from message-block MR bytes (`0x5a..0xca`, including host-populated MR12/14/15/24/30 shadows, with initial **`MR16 |= 0x41`** on Pass 0), executes the Pass 0 18-MRW burst, submits **`MR16 |= 0x45`** + `cmd = 7` ACSM `NOP` (`delay = max(8, freq_clks)`), clears **`MR16 &= 0xbc`** and sets `MR28 |= 2` when `(MR28_A0 & 0x20) == 0`, executes the Pass 1 MRW burst and ZQCAL MPC, then executes the **quickboot-specific per-rank writes (`MR40 = 0x40` and `MR37 = 0x40` via ACSM MRW mode `cmd = 6` with wait `0x22` and `cs_mask = 1 << r` for each populated rank `r = 0..1`)** followed by the shared `Step 6.2` tail (if `(MR28_A0 & 0x20) == 0`, broadcast **`MR28 = MR28_A0 | 2`** with `cs_mask = 3` and wait `0x22`; ACSM **`cmd = 0x28`** [8-halfword descriptor `hw[0]=0x0001, hw[3]=0x0028, hw[5]=0x0003`, others `0` -> 4-halfword ACSM SRAM slot **`c028 0000 0000 0010`**, matching `CA_rising = 0x40`, `cs_mask = 3`, `delay = 2`]; and ACSM **`cmd = 7` `NOP`** [`0`-filled descriptor + `delay = 0x1e` wait]), and enables `CSR[0x01f001] = 1` when `byte[0x80000096]` bit 4 is set.
   - **Pre-DLL direct-clock switch, per-rank post-MRW step, and RX-enable clear (strict order before Step 6)**: immediately after the MRW/NOP tail and before DLL recalibration starts, the quickboot sequence:
     1. Performs a **second switch back to direct/bypass clock** (`CSR[0x0200a2] = (CSR[0x0200a2] & 0xfffc) | 1`, `delay_dfi_clks(4)`, `CSR[0x0200f8] = 1`, `delay_dfi_clks(0x1f)`—reverting the PLL switch that ran before the MRW bursts);
     2. Clears **`CSR[0x02006e] = 0`**;
     3. Whenever `SequenceCtrl[15:8] == 0` (true under `SequenceCtrl = 1`), iterates across each populated rank (`cs = 1 << r` for `r = 0, 1` via `CSR[0x020072/73]`, `CSR[0x02007a/7e] = 0` for the per-rank post-MRW completion check), restores broadcast rank select (`cs = 3`), and refreshes `CSR[0x020072/73]` and **`CSR[0x090808]`/`0x090813`** from `CSR[0x020314]`/`0x020315`;
     4. Clears **`CSR[0x01f0b3] = 0` and `CSR[0x01f0b4] = 0`** (`ARC 0x9003e166`/`0x9003e168`) immediately prior to Step 6. *(Quickboot issues the MRW bursts while in PLL mode **before** Step 7 Pass 3 restores `CSR[0x090808]`/`0x090813`, and Step 5 already performs the `CSR[0x020078] = 2` / `CSR[0x0200a2] = 2` + `delay(0, 10000)` PLL-lock wait.)*
6. **Hardware DLL / clock recalibration (core per-boot re-alignment)**:
   - Calls `delay_ps_or_clks(0, 0x40)`. Because `byte[0x80005a64] == 0` here, the `32×` frequency multiplier is **inactive** (not `2048`-clock scaled): `delay_ps_or_clks(0, 0x40)` waits **`0x40 + 1 = 65` minimum ticks** (`66` RTC ticks at `shift = 0`, `33` at `shift = 1`), then asserts `CSR[0x02f310] = 1` (`ARC 0x9005e620`) and `CSR[0x02f0f8] = 1` (`ARC 0x9005e1f0`).
   - On the primary `PllBypassEn == 0` calibration path: computes `u = CSR[0x020049] & 0x01f0` (`0x01f0` in the `0x03f0` reference capture), toggles `CSR[0x02f049] = u | 0x0400` (`ARC 0x9005e092`), waits `delay_ps_or_clks(1000, 0)` (**`1 ns`**), sets `CSR[0x02f0e0] = 1` (`ARC 0x9005e1c0`), waits `delay_ps_or_clks(1000000, 0)` (**`1 µs`**, the only long settling delay—`~801` DFI clocks at 6400/4), **samples `CSR[0x0200cf]` (`ARC 0x9004019e`) *while* `CSR[0x02f0e0] == 1` and `CSR[0x02f049] == u | 0x0400`**, and only *after* that read writes the release register `CSR[0x02f0c3] = 1` (`ARC 0x9005e186`), clears `CSR[0x02f0e0] = 0`, restores `CSR[0x02f049] = u | 0x0200`, and stores the sampled 5-bit field **`CSR[0x02004e + pstate] = sampled_200cf & 0x001f`** (`ARC 0x9004009c <- 0x9004019e & 0x1f`) (if `PllBypassEn != 0`, it sets `CSR[0x02f0c3] = 1` directly). **Once `CSR[0x02f0e0]` is cleared to `0`, hardware returns `CSR[0x0200cf]` to its idle/reset value `0x001f`**; reading `CSR[0x0200cf]` *after* `0x02f0e0 = 0` or from host software after PMU halt therefore reads `0x001f` (even when `CSR[0x02004e]` holds the calibrated value). **No status register is polled after `CSR[0x02f0c3] = 1`**, and no comparison is performed on `CSR[0x02004e]` (`CSR[0x0200cf]` is not part of `phypara`; in the 6400 MT/s reference captures `CSR[0x02004e]` holds **`0x000b`–`0x000c` on `DDR0` and `0x000c`–`0x000d` on `DDR1`**). Because Step 6 is an unpolled settling sequence, slightly longer waits do not break the protocol handshake (though extreme increases remain unvalidated by bench testing).
   - Writes **`CSR[0x09f02d] = 1` (`ARC 0x9013e05a = 1`) as a steady level (never cleared back to `0`)** for K3 PHY version codes `8`–`10` (decoded version `!= 0`; `CSR[0x09f028]` at `ARC 0x9013e050` applies only to version `0`; neither register is in `phypara`), and restores **`CSR[0x02f009 + pstate]` (`ARC 0x9005e012`)** from the initial `CSR[0x020009]` reading.
7. **`phypara` Pass 3 (post-DLL override restore) and completion**:
   - **After** DLL recalibration finishes, deserializes the remainder of `phypara` across its multi-branch loops (Segments `#38..#68` = `3,502` halfwords; **`384` distinct base CSRs across groups `0x00`–`0x0c`**, *not* `0x01<r><l>4e..51` which belong to Pass 1):
     - Group `0x00` (`1`: `0x00000`),
     - Group `0x01` (**`75` base CSRs** across `0x010001..0x0100eb`, including `0x010019, 1b, 22, 23` under PHY revision `>= 7` and RX ODT **`0x01004c` / `0x01004d`**—*not* `0x010048/4a`),
     - Group `0x02` (`61` base CSRs across `0x020000..0x020311`, starting with `0x020088, 89, 8a, 8c` and `0x02004c, 4a, 4b`),
     - Group `0x03` (`28` base CSRs across `0x030000..0x0300e9`, including `0x030000/03/04/07/1a/37/60`, `0x030080..89`, `0x0300a0..ab`, `0x0300e8/e9`—splitting CA restoration with Pass 1 and Pass 2),
     - Group `0x07` (`37` base CSRs across `0x070011..0x0700d7`),
     - Group `0x09` (`144` base CSRs across `0x09001c..0x090c1f`: `0x09001c..2d`, `0x090700..1b`, **`0x090803..0x090827`** including **`0x090805`**, **`0x090808`**, **`0x09080f`**, and **`0x090813`**, and **`0x090900..0x090987`** including **`0x090903..38`**), and
     - Group `0x0c` (`41` base CSRs across `0x0c0000..0x0c0121`).
   - Streams margin fields (`0x37`–`0x3e`, `0x52`–`0x59`), triggers final commit strobe, and emits major completion **`0x07`**.

> **Ordering constraint vs. full `DevInit` (`Quickboot = 0`) & Pass 3 safety:** Prepending a full `Quickboot = 0` `DevInit` before Pass 1/2 (`DevInit -> Pass 1/2 -> Step 6 -> Pass 3`) is **not** valid/equivalent: `Quickboot = 0` `DevInit` measures and overwrites `CSR[0x02000a/12/17]`, executes `Step 3.6`, and runs DRAM reset/`tINIT` and initial MRW/ACSM bursts while the PHY still holds untrained defaults instead of Pass 1/2 state (the same ordering defect that limits host-only post-`DevInit` MMIO restores to ~3/10 passing boots). Conversely, none of Step 5 (DRAM reset), MRW, or Step 6 (DLL recalibration) overwrites any Pass 3 CSR, so writing non-divider Pass 3 entries earlier is benign **provided `CSR[0x090808]` and `CSR[0x090813]` are re-applied/recomputed from `CSR[0x020314]`/`0x020315` *after* Step 6 DLL recalibration** (in case DLL/clock recalibration shifts the live clock outputs).
>
> **Group `0x01` partitioning rule across the 69 `phypara` segments:** Across the entire 5,170-halfword stream in [appendix-phypara-order.md](appendix-phypara-order.md), Pass 1 consumes Segments `#0..#25` (`860` halfwords), Pass 2 consumes Segments `#26..#37` (`808` halfwords), and Pass 3 consumes Segments `#38..#68` (`3,502` halfwords). Partitioning group `0x01` entries (`0x01<r><l><off>`) by CSR offset covers 100% of group `0x01` entries in `phypara`:
> 1. **Pass 2 (`30` offsets):** `0x10`–`0x13`, `0x15`–`0x16`, `0x20`–`0x21`, `0x24`–`0x2b`, `0x5a`–`0x67` (`4` per-lane `WrDq` offsets × 2 + `11×2` per-rank offsets; Segments `#30..#37`);
> 2. **Pass 1 (`46` base CSRs):** `0x10000, 03, 04, 06, 0c, 0d, 14, 2e, 30, 33, 35, 38, 3a, 3b, 3c, 3e, 40, 42, 43, 48, 4a, 4b, 4e..51, 56, 57, 80..87, a3, a5, ad, af, d0..d4, d9` (`Segments #0..#3, #14..#17, #22..#25`);
> 3. **Pass 3 (`75` base CSRs):** `0x10001, 02, 05, 07, 09..0b, 0f, 17, 18, 19, 1a, 1b, 1c..1f, 22, 23, 2f, 36, 37, 39, 3d, 44, 45, 4c, 4d, 68..6f, 72, 75, 89..8c, 8e, 91..97, 9e, a2, a4, a6..a8, aa, ac, ae, b0..b5, b9..bb, be, da, de, e8..eb` (`Segments #57..#68`, where `0x19, 1b, 22, 23` execute under PHY revision `>= 7`).
> 4. Any group `0x01` offset not listed above is absent from the `phypara` serialization table.

| CSRs restored directly at Pass 3 entry blocks *(offsets relative to each sub-block's cursor)* | CSR | Cursor offset | CSR | Cursor offset |
|---|---|---:|---|---:|
| Master clock / Pll | `0x020088` | `[0]` | `0x090811`..`0x090819` | `[0xc]`..`[0x14]` |
| | `0x020089`, `0x02008a`, `0x02008c` | `[1]`, `[2]`, `[3]` | `0x09081b`..`0x09081f` | `[0x16]`..`[0x1a]` |
| ACSM / PHY clock dividers | `0x090804`, `0x090806`, `0x090807`, **`0x090808`**, `0x090809`, `0x09080b`..`0x09080e` | `[1]`, `[3..10]` | `0x090903` | `[0x1b]` |
| Latency / calibration blocks | `0x09001d`..`0x09002d` | `[0x601]`..`[0x611]` | `0x090701`..`0x090708` | `[0x613]`..`[0x61a]` |

*(Note: `[n]` indices in the left and right groups belong to successive monotonic cursor positions within Pass 3, not a single shared base.)*

## Inferred: why direct host-only CSR restoration cannot substitute for quickboot

The reference quickboot sequence does **not** restore the 5,170 `phypara` halfwords over PHY MMIO CSRs from the host. Attempting to emulate quickboot by running full `DevInit` (`SequenceCtrl = 1`, `Quickboot = 0`) and writing saved `phypara` entries directly from the host over PHY MMIO omits three mandatory hardware steps performed only by the quickboot path:

1. **Quickboot PMU firmware execution (`lp5_quickboot` image + DMEM `0x58200` / `0x5aa00`)**: The host places the modified `0x140`-byte message at DMEM `0x58000` (with reserved `0x200`-halfword stride), the serialized 5,170-halfword `phypara` payload at **DMEM `0x58200`**, and the 710-halfword quickboot table at **DMEM `0x5aa00`** (`ARC 0x80005400`, with zero-fill across `[0x59632 .. 0x5aa00)` and `[0x5acc6 .. 0x60000)`), then executes the dedicated quickboot PMU image (`MICRO_RESET = 9, 1, 0` → completion `0x07` / failure `0xff`). Only the quickboot PMU firmware interleaves Pass 1 (`0x01<r><l>4e..51`, `0x01<r>0ad/af`, `0x01<r>0a3` unmasked, `0x0c0086`), Pass 2, DRAM reset/`tINIT`, MRW bursts, the **`0x02f049`/`0x02f0e0`/`0x02f0c3` hardware DLL/clock recalibration**, and post-DLL **Pass 3 (`0x090803..27`, `0x090900..87`, `0x01004c/4d`, `0x03xxxx`, `0x07xxxx`, `0x0cxxxx`)**. Without running that ordered sequence, PHY FIFO/DLL pointers remain misaligned with the current boot clock phase, causing intermittent per-byte-lane read failures (such as a burst/word-offset shift on lane 1 or `0xff`/bit flips on lane 2; ~3/10 boots succeed purely by luck).
2. **Pre-PMU MR-shadow updates (`MR12`, `MR14`, `MR15`, `MR24`, `MR30`)**: Before starting the quickboot PMU firmware, the host must populate the `MR12`/`MR14`/`MR15`/`MR24`/`MR30` message-block shadows from the trained VREF/DFE/DCA outputs so the quickboot firmware applies trained DRAM mode registers (restoring ACSM after halt does not replace those pre-PMU shadow updates).
3. **Clearing `CSR[0x020037]` bit 6 before ACSM restoration**: Result collection sets `CSR[0x020037] |= BIT(6)`, and quickboot clears `CSR[0x020037] &= ~BIT(6)` after stopping the PMU and before writing ACSM (`0x41000`); leaving bit 6 set leaves the PHY in the wrong state for ACSM operation.

### Rank-1 misalignment hypotheses when splitting Pass 1 / Pass 2 to the host *(Inferred)*

When an independent quickboot sequence replaces or anticipates portions of Pass 1 / Pass 2 on the host before starting the PMU (and observes intermittent coarse burst shifts such as an 8-byte offset isolated to Rank 1 across ~1/3 of boots), four specific hardware-contract sensitivities must be checked against the reference ordering above:

1. **Second direct-clock switch before DLL sampling**: If the post-MRW switch back to direct/bypass clock (`CSR[0x0200a2] = (.. & 0xfffc) | 1`, `delay_dfi_clks(4)`, `CSR[0x0200f8] = 1`, `delay_dfi_clks(0x1f)`) is omitted after the MRW bursts, Step 6 DLL recalibration samples `CSR[0x0200cf]` while the PHY remains in PLL mode (`CSR[0x0200f8] = 0`), which can plausibly introduce an intermittent 1-beat (`8`-byte on a 64-bit bus) phase offset.
2. **Per-rank `MR40 = 0x40` / `MR37 = 0x40` issuance**: In Step 5, `MR40` and `MR37` are programmed **per rank** (`cs_mask = 1 << r` for `r = 0, 1`), not as a single shared broadcast. Omitting the separate Rank 1 pass or collapsing both ranks leaves Rank 1's impedance/OCD uncalibrated or tied to Rank 0's path.
3. **Per-rank post-MRW step under `SequenceCtrl == 1`**: Between `CSR[0x02006e] = 0` and clearing `CSR[0x01f0b3]`/`0x01f0b4 = 0`, the reference sequence switches rank select individually (`cs = 1 << r` for each populated rank `r = 0, 1`) when `SequenceCtrl[15:8] == 0`; omitting Rank 1's iteration diverges from the reference rank-select state prior to the `CSR[0x090808]`/`0x090813` refresh and Step 6.
4. **Host-side Pass 1 / Pass 2 slicing, commit strobe, and reset ordering**: Host-preloaded Pass 1 / Pass 2 writes must issue the `CSR[0x02f052]` commit pulse (`1 -> delay_dfi_clks(10) -> 0` when `MsgMisc[0] == 1`) before Step 4 DQ block reset (`CSR[0x020303..20]`), preserve both Rank 0 and Rank 1 address offsets (`188`-halfword per-rank stride across Pass 2 `0x01<r><l>10..13`/`24..27` and separate Rank 1 offsets for Pass 1 `0x4e..51`), and defer `CSR[0x090808]`/`0x090813` (Pass 3) until **after** Step 6 DLL recalibration completes.

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
- The pair-by-pair serialization layout across Pass 3's `384` base CSRs (`0x00`–`0x0c` multi-P-state/rank/lane branches) has not yet been fully enumerated (`Pass 1` and `Pass 2`'s `47`-halfword DQ restore are completely specified); the remainder of the `710`-halfword table at `0x5aa00` outside `[0x1c..0x1f]`, `[0x78..0x7b]`, and `[0xc8..0xd9]` should still be loaded opaquely.
- The semantic details of all restored ACSM entries remain partially open; their complete opaque preservation is nevertheless required.
- PSTATE SRAM restoration is absent from the observed flow, and its necessity for other firmware/configurations is not established.
- The reference timing does not guarantee the same latency on other boards or software stacks.
