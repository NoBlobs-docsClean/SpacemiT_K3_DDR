<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Full Initialization Sequence](06-full-initialization.md) | [Index](README.md#reading-order-and-contents) | [Next: Mailbox & Streaming Protocol →](08-mailbox-and-streaming.md)

# PHY training behavior

## Scope and confidence

This document describes the behavioral model required to understand and replace LPDDR5 PHY training: what is swept, what depends on what, how centers and margins are represented, and how success should be decided.

The stage names, message-block controls, output units, and selected diagnostics are observed interface facts. The detailed candidate ordering, exact pass/fail statistics, internal retry decisions, and numeric acceptance thresholds of the reference firmware are **not fully known**. Statements about a general sweep/center algorithm are a functional model, not a promise that every reference stage uses the same implementation.

Training runs on the PHY PMU. The host supplies a validated message block and initial PHY state, then services the mailbox without accessing training CSRs.

## Training domain

Training is constrained by:

- enabled P-state and data rate (`Pstate`, `DRAMFreq`, `PllBypassEn`, `DfiFreqRatio`);
- channel/rank population (`CsPresentChA/B`), device organization (`X8Mode`), and DQ counts (`EnabledDQsChA/B`);
- initial mode-register values and electrical overrides;
- enabled stages (`SequenceCtrl`) and CA/CS options (`CATrainOpt`);
- per-point comparison count (`BitTimeControl`);
- pattern selection, VREF ranges/steps, delay-scan shifts, and 2D optimization controls;
- initial calibrated state supplied by the pre-training tables.

DQ-related work is per channel, rank, byte/lane, and potentially bit. CA/CS work is per command group/channel/rank. Absent ranks and disabled DQs must not contribute results or failures.

The main adjustable dimensions are:

| Dimension | Typical target | Result units |
|---|---|---|
| Write-clock/write-data timing | PHY transmit path and DRAM sampling | PHY delay taps |
| Read-gate/read-DQS timing | PHY receive sampling | PHY delay taps |
| RX voltage reference | PHY receiver | PHY DAC codes/steps |
| TX/device voltage reference | DRAM receiver | DRAM DAC codes/steps |
| Duty cycle | DRAM, PHY RX, or PHY TX | DCA codes |
| DRAM transmit equalization | DRAM transmitter | DFE fields/coefficients |
| Command/address and CS timing | CA/CS groups | PHY delay taps and MR values |
| Maximum read latency | DFI read path | DFI clocks |

Tap-to-time and DAC-code-to-voltage conversions are PHY/DRAM-specific and are not established by this specification.

## Dependency model

The following is a **functional dependency order**, not a claim that bit numbers or every observed major-message ID directly encode execution order.

1. **Device/PHY initialization (`SequenceCtrl[0]`)** establishes reset, clocking, basic MR state, and a working command interface. All later stages depend on it.
2. **Write leveling (`[1]`)** establishes coarse WCK/CK and per-lane write timing. Reliable subsequent write-based tests depend on it.
3. **Read gate / receive enable (`[2]`)** finds a usable read-data capture window.
4. **Read DQS (`[3]`)** refines read sampling; read-eye/VREF optimization depends on a working gate/strobe.
5. **Write DQ (`[4]`)** refines the write eye after basic write alignment and a usable readback/test mechanism exist.
6. **DCA (`[6]`, `[7]`, `[8]`)** refines duty cycle in the DRAM and PHY. These stages consume already functional read/write paths and can change the timing results that later stages use.
7. **Maximum read latency (`[9]`)** measures the passing DFI latency and adds `DFIMRLMargin`; it depends on functional read training.
8. **DRAM TX DFE (`[10]`)** optimizes the DRAM transmit path and therefore depends on an operational read path. Later read-margin evaluation must use the final DFE setting.
9. **CA/CS (`[12]`)** requires a functioning command interface and may include CA VREF and LPDDR5 CS training. The exact placement of every CA/CS substep relative to all DQ/DCA substeps is not established by the available evidence.

The firmware also contains retraining loops. `DisableTrainingLoop` can suppress loops involving write leveling, read gate/read-DQ calibration, and write training. Consequently, a one-pass dependency graph is insufficient as an execution trace: later calibration may revisit earlier stages. A replacement should iterate only with explicit bounds and convergence checks.

### Measured per-stage execution time and CSR footprint (`SequenceCtrl` sweep)

Sweeping `SequenceCtrl` bit by bit on the reference 6400 MT/s dual-rank x16 platform reveals the exact execution duration and CSR write footprint of each stage (relative to a `SequenceCtrl = 0x0000` baseline):

| `SequenceCtrl` | Enabled stages | Duration (ms) | Incremental `phypara` entries changed vs. `0x0000` | Primary CSR / message-block effects |
|---|---|---:|---:|---|
| `0x0000` | None (PMU framework only) | 8 | 0 (baseline) | Enters and exits cleanly without touching `0x02xxxx` calibration CSRs |
| `0x0001` | `DevInit` | 15 | 68 (507 total CSR write ops) | Executes 6-phase PHY/DRAM initialization (507 CSR writes across groups `0x00`–`0x0c`), calibrates group `0x02` master clock period (`0x02000a`, `0x020017`, `0x020012`, `0x020186`, `0x020187`, `0x020011`), reads back `QBPll*` (`0x80000130`–`0x8000013c`), updates `0x09080b`/`0x09080c` and `0x0c0086` (`0x0003`) |
| `0x0007` | `+ WrLvl + RxEn` | 28 | 165 | Writes per-lane write-leveling and receive-enable coarse/fine CSRs (`0x01000c/0d = 0x009a/0x009b`, `0x010026 = 0x00ba`, etc.) |
| `0x000f` | `+ RdDQS` | 3,567 | 559 | **63% of total runtime (~3.54 s):** trains `CSR[0x4e]` and `CSR[0x4f]` across all 36 lanes (4 ranks × 9 lanes) and populates `RxClkDly_Margin` / `VrefDac_Margin` |
| `0x001f` | `+ WrDq` | 4,413 | 598 | **~846 ms:** trains write DQ eye delays (`0x01<r><l>10..13`, `0x01<r><l>be`) and populates `TxDqDly_Margin`, `DeviceVref_Margin`, and trained `MR14`/`MR15` |
| `0x005f` | `+ DRAM DCA` | 5,611 | 484 | **~1.20 s:** adjusts per-bit transmit delays (`0x01<r><l>68..6f` shifted by ~+4 taps) and updates `MR30` |
| `0x025f` | `+ MxRdLat` | 5,613 | 503 | **~2 ms:** updates DFI maximum read latency CSRs (`0x070011`, `0x070060..63`) and `0x090813` |
| `0x125f` | `+ LPCA` (full mask) | 5,624 | 587 | **~11 ms:** trains CA VREF (`MR12`) and LP5 CS timing (`0x030001 = 0x0004`) |

Note that running `SequenceCtrl = 0x0009` (`DevInit + RdDQS` only, skipping `WrLvl` and `RxEn` because the pre-training tables already hold nominal leveling values) completes in ~3.55 s and produces `CSR[0x4e]`/`CSR[0x4f]` values within 1–2 taps of a full `0x125f` run.

### `DevInit` master clock-period calibration (`0x02xxxx`)

Before invoking the master clock-period calibration routine during `DevInit` (`SequenceCtrl[0]`), three hardware preconditions must hold:

1. **DPLL locked and DFI clock running** (configured by the host before releasing the PMU);
2. **PHY block-enable, clock-shift setup, DQ block reset, and DRAM reset phases completed** (`CSR[0x0200a2]` and `CSR[0x0200f8]` configured, `0x80000404` shift initialized from `CSR[0x0c0080]`, and `Phase 4`/`Phase 5` resets completed; see below);
3. **`CSR[0x020011]` (`ARC 0x90040022`) bits `[6:5]` cleared**: The host pre-training table initializes `CSR[0x020011]` to **`0x006f`**. When `Quickboot == 0` (`byte[0x80000019] == 0`), at the start of `DevInit` Phase 3 the PMU saves the original value (`0x006f`) to DMEM `0x8000042a` and writes `0x006f & 0xff9f = 0x000f` to `CSR[0x020011]` (`0x000f` is also the final value captured in post-training `phypara`).

Once those preconditions hold, the PMU calibrates the clock period and frequency-bucket CSRs using the following sequence:

1. **3-step period-counter strobe with bit 10 falling-edge latch (`CSR[0x0200e5]` and `CSR[0x0200e3]`)**:
   - The period counters at `CSR[0x0300bf]` (`ARC 0x9006017e`) and `CSR[0x0100e0]` (`ARC 0x900201c0`) do **not** latch on a single level write. Simply writing `CSR[0x0200e3] = old_e3 | 0x0c00` without subsequently clearing bit 10 back to `0` leaves both registers at their unlatched reset/pre-training state `0x0240` (`0x0240 & 0x01ff = 0x0040`).
   - Instead, the hardware requires clearing bits `[4:0]` of `CSR[0x0200e5]` (`ARC 0x900401ca`) **before** pulsing bit 10 (`0x0400`) of `CSR[0x0200e3]` (`ARC 0x900401c6`) high and then low while bit 11 (`0x0800`) remains asserted:
     1. Read and save `old_e3 = CSR[0x0200e3]` (`ARC 0x900401c6`) and `old_e5 = CSR[0x0200e5]` (`ARC 0x900401ca`). Neither `0x0200e3` nor `0x0200e5` is in `phypara`, and both are restored in step 4 so their net change is zero.
     2. Write `CSR[0x0200e5] = old_e5 & ~0x001f` (`old_e5 & 0xffe0`, clearing bits `[4:0]`).
     3. Write `CSR[0x0200e3] = old_e3 | 0x0800` (assert bit 11 = counter enable, with bit 10 = `0`).
     4. Write `CSR[0x0200e3] = old_e3 | 0x0c00` (assert bit 10 = `1` while keeping bit 11 = `1`).
2. **Short measurement delay (`delay(10)`) and falling-edge latch**:
   - Invoke the short-delay helper with `n = 10` (`0x0a`).
   - **Zero-RTC-wait scaled delay behavior:** The short-delay helper loads a 1-bit shift from DMEM byte `0x80000404` (`gp + 4`), initialized during Phase 2 from `CSR[0x0c0080]` (`UCCLK_HCLK_ENABLES`, `ARC 0x90180100`) as `shift = 1 & ~(CSR[0x0c0080] >> 2)` (`shift = 1` when `0x0c0080 == 3`, or `shift = 0` when `0x0c0080 == 7`), and computes `scaled = (n + 1) >> shift`. Only when `scaled >= 0x10` (`16`) does the helper poll the 64-bit hardware RTC (`AUX 0x103`–`0x105`). When `scaled < 16` (for `n = 10`, `scaled` is `5` or `11`), it performs **one 16-bit dummy read of `CSR[0x020024]` (`ARC 0x90040048`) and returns immediately** without touching the RTC. (Note that `0x80000404` is distinct from `0x8000045a = 4`, which is used by the ACSM NOP cycle counter.)
   - Immediately after `delay(10)`, write **`CSR[0x0200e3] = old_e3 | 0x0800`** (clearing bit 10 back to `0` while keeping bit 11 = `1`). **This `1 -> 0` falling edge on bit 10 synchronously latches the period counters.**
3. **Read 9-bit period counters and program `CSR[0x02000a + pstate]` and `CSR[0x020017 + pstate]`**:
   - Read CA period counter `T_ca = CSR[0x0300bf] & 0x01ff` (`ARC 0x9006017e`) and write `T_ca` to `CSR[0x02000a + pstate]` (`ARC 0x90040014 + 2 * pstate`).
   - Read DQ period counter `T_dq = CSR[0x0100e0] & 0x01ff` (`ARC 0x900201c0`) and write `T_dq` to `CSR[0x020017 + pstate]` (`ARC 0x9004002e + 2 * pstate`).
   - **Reference values and bit 9 (`0x0200`) interpretation:** The host pre-training table initializes `CSR[0x02000a]`, `CSR[0x020017]`, and `CSR[0x020187]` to `0x0040`. Because the PMU masks `0x0300bf` and `0x0100e0` with `& 0x01ff` (9 bits), bit 9 (`0x0200`) is discarded and never tested; reading raw `0x0240` (`0x0200 | 0x0040`) simply reflects the unlatched pre-training value `0x0040`. On the reference 6400 MT/s board, after the bit 10 falling-edge strobe latches the measurement, `CSR[0x02000a]` reads **`0x0082`** (`130`, or **`0x0086`** = `134` in a `SequenceCtrl = 0x0001` `DevInit`-only capture), whereas `CSR[0x020017]` remains **`0x0040`** (`pre-unchanged`). Thus `CSR[0x02000a] != 0x0040` (`~0x0082`–`0x0086`) is the definitive check that the period counter latched.
4. **Restore counter control registers**:
   - Write `CSR[0x0200e3] = old_e3` and `CSR[0x0200e5] = old_e5`.
5. **Compute frequency buckets from `DRAMFreq` and `DfiFreqRatio` and program `CSR[0x020012]`, `CSR[0x020186]`, `CSR[0x020187]`, and per-rank `CSR[0x01<r>0d9]`**:
   - Contrary to earlier approximations, the frequency-bucket fields in `CSR[0x020012]` (`ARC 0x90040024`) and `CSR[0x020186]` (`ARC 0x9004030c`) are **deterministic functions of the message-block fields `DRAMFreq` (`0x80000006`, 16-bit) and `DfiFreqRatio` (`0x80000008`, 8-bit)**—they do **not** depend on the live counter `T_ca`.
   - Specifically, the PMU computes two scaled frequency values `f_lo` and `f_hi` using unsigned integer division (`divu`):
     ```text
     f    = DRAMFreq / DfiFreqRatio
     f_lo = (DfiFreqRatio == 2) ? (2 * f) : f
     mult = ((DfiFreqRatio != 2) ? 2 : 0) + 2     // 4 when DfiFreqRatio != 2, or 2 when DfiFreqRatio == 2
     f_hi = f * mult
     ```
     *(Note: even if `mult` is evaluated as `6` when `DfiFreqRatio != 2`, for `DRAMFreq = 6400` and `DfiFreqRatio = 4` both `4 * 1600 = 6400` and `6 * 1600 = 9600` exceed the highest threshold `2199`, producing the exact same `fine_hi = 3` and `bucket3 = 7`.)*
   - Both `f_lo` and `f_hi` are classified against the five strict `>` thresholds (`331` = `0x014b`, `531` = `0x0213`, `799` = `0x031f`, `1799` = `0x0707`, `2199` = `0x0897`):
     - `f_lo` determines `coarse` (bits `[7:4]` of `0x020012`) and `fine_lo` (bits `[3:0]` of `0x020012`);
     - `f_hi` determines `bucket3` (bits `[15:12]` of `0x020012` and bits `[7:4]` of `0x020186`) and `fine_hi` (bits `[11:8]` of `0x020012` and bits `[3:0]` of `0x020186`):

     | Interval for `x` (`f_lo` or `f_hi`) | `coarse` (`x = f_lo`, bits `[7:4]`) | `fine_lo` (`x = f_lo`, bits `[3:0]`) | `bucket3` (`x = f_hi`, bits `[15:12]` / `[7:4]`) | `fine_hi` (`x = f_hi`, bits `[11:8]` / `[3:0]`) |
     |---|---:|---:|---:|---:|
     | `x <= 331` (`<= 0x014b`) | `0x20` | `0` | `2` | `0` |
     | `332 <= x <= 531` (`0x014c..0x0213`) | `0x30` | `1` | `3` | `1` |
     | `532 <= x <= 799` (`0x0214..0x031f`) | `0x40` | `2` | `4` | `2` |
     | `800 <= x <= 1799` (`0x0320..0x0707`) | `0x50` | `3` | `5` | `3` |
     | `1800 <= x <= 2199` (`0x0708..0x0897`) | `0x60` | `3` | `6` | `3` |
     | `x > 2199` (`> 0x0897`) | `0x70` | `3` | `7` | `3` |

   - Assemble and write the three P-state-indexed registers (`pstate = byte[0x80000004] & 0x0f`, which is `0` on the reference profile):
     ```text
     CSR[0x020012 + pstate] = (bucket3 << 12) | (fine_hi << 8) | coarse | fine_lo   (ARC 0x90040024, pre-training 0xa1a1)
     CSR[0x020186 + pstate] = (bucket3 << 4)  | fine_hi                             (ARC 0x9004030c, pre-training 0x00a1)
     CSR[0x020187 + pstate] = CSR[0x0100d5]                                         (ARC 0x9004030e, copied unmasked from 0x900201aa)
     ```
   - Immediately after writing `CSR[0x020187 + pstate]`, loop over each active rank `r = 0 .. num_ranks - 1` (`4` ranks across ChA + ChB on the reference board) and write **`0x009c`** (`156`) to `CSR[0x01<r>0d9]` (`ARC 0x900201b2 + r * 0x2000`, which is the third `phypara` halfword `156` observed in the reference dump).
   - **Worked example at 6400 MT/s (`DRAMFreq = 6400`, `DfiFreqRatio = 4`)**:
     - `f = 6400 / 4 = 1600`;
     - `f_lo = 1600` (`800 <= 1600 <= 1799`) → `coarse = 0x50`, `fine_lo = 3`;
     - `f_hi = 6400` (`> 2199`) → `bucket3 = 7`, `fine_hi = 3`;
     - `CSR[0x020012] = (7 << 12) | (3 << 8) | 0x50 | 3 = 0x7353` (exact match to reference `phypara` `0x7353`, updated from pre-training `0xa1a1`);
     - `CSR[0x020186] = (7 << 4) | 3 = 0x0073` (exact match to reference `phypara` `0x0073`, updated from pre-training `0x00a1`);
     - `CSR[0x020187] = CSR[0x0100d5] = 0x0041` (exact match to reference `phypara` `0x0041`, updated from pre-training `0x0040`).

### Complete `DevInit` (`SequenceCtrl[0]`) hardware initialization sequence

While comparing a `SequenceCtrl = 0x0001` capture against `0x0000` shows **68 changed `phypara` entries** (50 in group `0x01`—namely `0x01<r><l>50` and `0x51` across all 36 lanes plus `0x01<r>0a3` and `0x0100d5`—along with 16 in group `0x02` and `0x09080b`/`0x09080c`), the complete `DevInit` stage actually executes **507 CSR write operations** across six ordered hardware phases. Many of those writes target dynamic control, reset, and strobe registers outside the 69 `phypara` serialization segments.

#### Broadcast alias `0x01f0xx` (`ARC 0x9003e0xx`) and configuration commit strobe triad (`0x020028`, `0x02f0a5`, `0x02f077`)

- **All-rank write broadcast (`CSR[0x01f0xx]` / `ARC 0x9003e000 + 2 * xx`)**: Within group `0x01`, rank index `0xf` (`0x01f0xx`, or `0x01f<l>xx`) is a **write-only hardware broadcast alias** that simultaneously updates all four ranks `r = 0..3` (`CSR[0x01<r>0xx]`, `ARC 0x90020000 + r * 0x2000 + 2 * xx`). Because `0x01f0xx` is write-only, the PMU always reads the current value from rank 0 (`CSR[0x0100xx]`), modifies the target bitfield, and writes the result to `CSR[0x01f0xx]`.
- **Configuration commit strobe triad (`0x020028`, `0x02f0a5`, `0x02f077`)**: Whenever the PMU updates PHY control CSRs and needs to latch/commit the configuration into the hardware state machine, it issues this three-write strobe sequence (~40 times across `DevInit`):
  ```text
  CSR[0x020028] = 1    (ARC 0x90040050 = 1)
  CSR[0x02f0a5] = 1    (ARC 0x9005e14a = 1)
  CSR[0x02f077] = 0    (ARC 0x9005e0ee = 0)
  ```
  Note that `CSR[0x020077]` (`ARC 0x900400ee`) is **never written directly by the PMU**; its `1 -> 0` transition observed in `phypara` is driven by the PHY hardware when `CSR[0x02f077] = 0` is strobed.

#### Ordered `DevInit` phases

1. **Phase 1 — PHY block enable (`CSR[0x0200a2]`, `CSR[0x0200f8]`, and `CSR[0x020060]`)**:
   - Read-modify-write `CSR[0x0200a2]` (`ARC 0x90040144`): `CSR[0x0200a2] = (CSR[0x0200a2] & 0xfffc) | 0x0001` (set bits `[1:0] = 1`).
   - Call `delay(4)` (one dummy read of `CSR[0x020024]`), then `delay(0x1f)` (`31` scaled timer units).
   - Write `CSR[0x0200f8] = 1` (`ARC 0x900401f0`).
   - If `Quickboot == 0` (`byte[0x80000019] == 0`) and `MsgMisc[7] == 0` (`(byte[0x80000000] & 0x80) == 0`), write `CSR[0x020060] = 2` (`ARC 0x900400c0`, pre-reset `MemResetL` state).
2. **Phase 2 — Clock-shift initialization and configuration commit**:
   - Read `CSR[0x0c0080]` (`ARC 0x90180100`), compute `shift = 1 & ~(CSR[0x0c0080] >> 2)`, and store `shift` to DMEM byte `0x80000404` (`gp + 4`).
   - Issue the configuration commit strobe triad (`CSR[0x020028] = 1`, `CSR[0x02f0a5] = 1`, `CSR[0x02f077] = 0`).
3. **Phase 3 — `QBPll*` read-back, topology setup, DQ block reset (`Phase 4`), DRAM reset (`Phase 5`), per-rank DQ sweep, and master clock-period calibration**:
   - **Step 3.1 (`QBPll*` read-back)**: If `byte[0x80000008] & 0x20` (or `Pstate & 0x20`) is non-zero, copy the seven `QBPll*` CSRs (`CSR[0x020088]`, `0x020089`, `0x02008a`, `0x02008c`, `0x02004a`, `0x02004b`, `0x02004c`) to message-block offsets `0x80000130`–`0x8000013c` (see [05-message-block.md](05-message-block.md)).
   - **Step 3.2 (`CSR[0x02008b]`, `CSR[0x020072]`, `CSR[0x020073]`)**:
     - Write `CSR[0x02008b]` (`ARC 0x90040116`) = `(byte[0x80000008] & 0x80) ? 0 : (byte[0x80000008] & 1)` (`0` when `DfiFreqRatio == 4`).
     - Write `CSR[0x020072]` (`ARC 0x900400e4`) = `CsPresentChA` (`byte[0x80000025]`, `0x0003` on dual-rank).
     - Write `CSR[0x020073]` (`ARC 0x900400e6`) = `CsPresentChB` (`byte[0x80000040]`, `0x0003` on dual-rank).
   - **Step 3.3 (`CSR[0x020011]` mask and `CSR[0x01<r>0a3]` save/mask)**:
     - If `Quickboot == 0` (`byte[0x80000019] == 0`): save `CSR[0x020011]` (`0x006f`) to DMEM `0x8000042a` and write `CSR[0x020011] &= 0xff9f` (`0x000f`).
     - Earlier in `DevInit`, the four per-rank `CSR[0x01<r>0a3]` values (`0x0b33`, written by the host pre-training table) are saved into DMEM `0x80000430`–`0x80000437`. Here, for each active rank `r = 0 .. num_ranks - 1`, read-modify-write `CSR[0x01<r>0a3]` (`ARC 0x90020146 + r * 0x2000`) `&= 0x000c` (keeping only bits `[3:2]`; the full `0x0b33` values are restored from `0x80000430`–`0x80000437` after `DevInit` calibration).
   - **Step 3.4 (Phase 4 — DQ block reset sequence on `CSR[0x020303]`..`0x020320`)**:
     1. Read and save `u_303 = CSR[0x020303]` (`ARC 0x90040606`);
     2. Write `CSR[0x020310] = 1` (`ARC 0x90040620`);
     3. Write `CSR[0x020303] = (u_303 & 0x00ef) | 0x0010` (set bit 4);
     4. Write `CSR[0x020310] = 0`;
     5. Write `CSR[0x02030d] = 1` (`ARC 0x9004061a`), then `CSR[0x02030d] = 0`;
     6. Write `CSR[0x020311] = 1` (`ARC 0x90040622`);
     7. Wait **`130,000,000` (`0x07bfa480`)** scaled timer units;
     8. Write `CSR[0x020311] = 0`;
     9. Write `CSR[0x020320] = 0` (`ARC 0x90040640`), then `CSR[0x020320] = 1`, then `CSR[0x020320] = 0`;
     10. Restore `CSR[0x020303] = u_303 & 0x00ff` (clear bit 4).
   - **Step 3.5 (Phase 5 — DRAM reset and PHY PLL/reset sequencing)**:
     1. Read `u_0f = CSR[0x01000f + pstate]` (`ARC 0x9002001e`) and write `(u_0f & 0xff78) | 0x0006` to broadcast `CSR[0x01f00f + pstate]` (`ARC 0x9003e01e`).
     2. Write `CSR[0x020061] = 1` (`ARC 0x900400c2`).
     3. Branch on `PllBypassEn` (`byte[0x80000005]`):
        - **Normal PLL mode (`PllBypassEn == 0`)**:
          1. `delay(4)` (dummy read of `CSR[0x020024]`);
          2. Write `CSR[0x020049] = 0x01f0` (`ARC 0x90040092 = 496`);
          3. Write `CSR[0x020061] = 0` (`ARC 0x900400c2`);
          4. Write `CSR[0x0200e1] = 0` (`ARC 0x900401c2`);
          5. Write `CSR[0x020310] = 0` (`ARC 0x90040620`);
          6. Write `CSR[0x0200e0] = 0` (`ARC 0x900401c0`);
          7. Write `CSR[0x0200c3] = 0` (`ARC 0x90040186`);
          8. Write `CSR[0x0200e2] = 3` (`ARC 0x900401c4`), then wait **`1,000,000` (`0x000f4240`)** scaled timer units;
          9. Write `CSR[0x0200e2] = 2`, then wait **`500,000` (`0x0007a120`)** scaled timer units;
          10. Write `CSR[0x0200e2] = 0`, then wait **`10,000,000` (`0x00989680`)** scaled timer units;
          11. Write `CSR[0x020078] = 1` (`ARC 0x900400f0`), `CSR[0x020200] = 0x0033` (`ARC 0x90040400`), `CSR[0x0200a2] = 0` (`ARC 0x90040144`), then `delay(6)`;
          12. Write `CSR[0x020078] = 2`, `CSR[0x0200a2] = 2`, then wait **`10,000` (`0x2710`)** short-delay units;
          13. Write `CSR[0x020078] = 0`, `CSR[0x0200a2] = 0`, `CSR[0x020200] = 0`, then `delay(0x20)`;
          14. Write `CSR[0x0200f8] = 0` (`ARC 0x900401f0`) and `CSR[0x0200a2] = 2` (`ARC 0x90040144`).
        - **PLL bypass mode (`PllBypassEn != 0`)**:
          1. `delay(4)`, then write `CSR[0x020061] = 0`, `CSR[0x0200e1] = 1`, `CSR[0x0200e0] = 1`, `CSR[0x020310] = 0`, `CSR[0x0200c3] = 0`, and wait **`10,000,000` (`0x00989680`)** scaled timer units;
          2. Write `CSR[0x020078] = 1`, `CSR[0x020200] = 0x0033`, `CSR[0x0200a2] = 0`, then `delay(0x20)`;
          3. Write `CSR[0x020078] = 2`, `CSR[0x0200a2] = 2`, then `delay(1024)`;
          4. Execute steps 13 and 14 of the normal branch (`0x020078 = 0`, `0x0200a2 = 0`, `0x020200 = 0`, `delay(0x20)`, `0x0200f8 = 0`, `0x0200a2 = 2`).
   - **Step 3.6 (Per-rank DQ calibration loop when `DRAMFreq >= 1600`)**:
     - If `DRAMFreq < 1600` (`halfword[0x80000006] < 0x0640`), skip directly to Step 3.7.
     - Otherwise, write `CSR[0x01f0d9] = 0x1000` (`ARC 0x9003e1b2`), write `CSR[0x01f00f + pstate] = (CSR[0x01000f + pstate] & 0xfff8) | 0x0006` (`ARC 0x9003e01e`), write `CSR[0x01f0ac + pstate] = CSR[0x0100ac + pstate] & 0xfffe` (`ARC 0x9003e158`), wait `delay(0x28)`, and run the per-rank DQ calibration loop across `r = 0 .. num_ranks - 1` before restoring `CSR[0x01f0ac + pstate] |= 1`.
   - **Step 3.7 (Master clock-period calibration)**:
     - Execute the **master clock-period calibration** sequence detailed above (3-step `0x0200e5`/`0x0200e3` strobe with bit 10 falling edge → `CSR[0x02000a]`, `CSR[0x020017]`, `CSR[0x020012] = 0x7353`, `CSR[0x020186] = 0x0073`, `CSR[0x020187] = 0x0041`, and per-rank `CSR[0x01<r>0d9] = 0x009c`).
4. **Phase 6 — ACSM sequencer reset, block disable, `MemResetL` release (`CSR[0x020060]`), and initial LPDDR5 MRW/MPC sequence**:
   - **Step 6.1 (Initial ACSM NOP flush and block disable)**:
     - Reset the ACSM instruction pointers (`CSR[0x020022 + pstate] = 0` at `ARC 0x90040044`, `CSR[0x020023 + pstate] = 0` at `ARC 0x90040046`, DMEM `0x80000458 = 0`, `0x8000045e = 0`), queue a 4-cycle NOP instruction in ACSM SRAM (`0x41000`), and execute it via `CSR[0x020020] = 1` → poll `CSR[0x020021] == 0` → `CSR[0x020020] = 0` (see the ACSM sequencer subsection below).
     - Wait `delay(0x1f)` then `delay(4)`, write `CSR[0x0200f8] = 0` (`ARC 0x900401f0`), and write `CSR[0x0200a2] = (CSR[0x0200a2] & 0xfffc) | 0x0002` (`ARC 0x90040144`).
     - Write `CSR[0x02002a] = 1` (`ARC 0x90040054`).
   - **Step 6.2 (`MemResetL` release and LPDDR5 mode-register initialization)**:
     - When `Quickboot == 0` and `MsgMisc[7] == 0`, enter the DRAM mode-register initialization routine:
       1. Write `CSR[0x020041 + pstate] = 0` (`ARC 0x90040082`) and `CSR[0x02002a] = 1` (`ARC 0x90040054`);
       2. Wait for `tINIT3` (`2,000,000` = `0x001e8480` scaled units when `MsgMisc[5]` fast-init is set, or `200,000,000` = `0x0bebc200` scaled units otherwise);
       3. Release `MemResetL`: write **`CSR[0x020060] = 3`** (`ARC 0x900400c0`), wait `tINIT4` (`2,125` = `0x084d` scaled units), and if the post-reset state bit is active, write **`CSR[0x020060] = 1`** (`1` is the final value captured in `phypara` on the reference board, or `3` in a minimal `DevInit`-only capture);
       4. Issue the initial LPDDR5 mode-register writes (`MR19`, `MR18`, `MR1`, `MR2`, `MR3`, `MR10`, `MR11`, `MR17`, `MR12`, `MR14`, `MR15`, `MR16`, `MR20`, `MR22`, `MR24`, `MR37`, `MR40`, `MR41`) and ZQ/MPC commands across the populated ranks through the **ACSM command sequencer** (`0x41000` / `ARC 0x90082000` and `CSR[0x020020]`–`0x020027`).
   - **Step 6.3 (Post-`DevInit` lane state, `CSR[0x0000a4]`, and `CSR[0x0c0086]`)**:
     - Program the active channel/rank masks (`CSR[0x02007a]`, `CSR[0x02007e]`), `CSR[0x020042] = byte[0x80000b68] | 0x0100` (`ARC 0x90040084`), `CSR[0x020044]` (`ARC 0x90040088`), and per-rank/lane timing registers (including restoring `CSR[0x01<r>0a3]` from `0x80000430`–`0x80000437` and updating `CSR[0x01<r><l>50]`/`0x51` when applicable; in a full training run with `DisableTrainingLoop[1] == 0`, `0x50`/`0x51` end at `0x0035`).
     - Pulse `CSR[0x0000a4] = 1` (`ARC 0x90000148`) and update status register `CSR[0x0c0086] = 3` (`ARC 0x9018010c`).

### ACSM command sequencer (`0x41000` / `ARC 0x90082000` and `0x020020`–`0x020027`)

All DRAM mode-register writes (`MRW`), multi-purpose commands (`MPC`), and inter-command NOP delays during `DevInit` and subsequent training stages are transmitted by loading 4-halfword instruction packets into the **Address/Command State Machine (ACSM) SRAM** at host CSR index `0x41000`–`0x41fff` (`ARC 0x90082000`–`0x90083ffe`) and triggering the sequencer via group `0x02` control registers `CSR[0x020020]`–`0x020027` (`ARC 0x90040040`–`0x9004004e`):

1. **ACSM sequencer control and status registers (group `0x02`)**:
   - `CSR[0x020022 + pstate]` (`ARC 0x90040044`): ACSM start instruction index (`4`-halfword units; written `0` before loading a command sequence).
   - `CSR[0x020023 + pstate]` (`ARC 0x90040046`): ACSM stop/end instruction index (`4`-halfword units), set to `(halfword_cursor >> 2) - 1` before starting the sequencer.
   - `CSR[0x020027]` (`ARC 0x9004004e`): Sequencer mode/config register (written `0` before starting a command burst).
   - `CSR[0x020020]` (`ARC 0x90040040`): Sequencer run control (`1` = start execution from `CSR[0x020022]` through `CSR[0x020023]`; `0` = stop/clear after completion).
   - `CSR[0x020021]` (`ARC 0x90040042`): Sequencer busy status (polled after writing `CSR[0x020020] = 1` until `CSR[0x020021] == 0`, with a timeout of `25,000` iterations, before writing `CSR[0x020020] = 0`).
   - DMEM `0x80000458` (`16-bit`) and `0x8000045e` (`16-bit`): Current ACSM SRAM halfword write cursor (incremented by `8` halfwords per command/delay pair; reset to `0` at the start of a new sequence).

2. **ACSM 4-halfword instruction format (`hw[0..3]` at `0x90082000 + 2 * cursor`)**:
   - Each ACSM instruction occupies **4 consecutive 16-bit halfwords** (`hw[0..3]`), and the helper routines emit instructions in **8-halfword pairs** (`2 instructions = 16 bytes`).
   - **LPDDR5 CA command entry (`MRW-1` + `MRW-2`)**:
     - An LPDDR5 mode-register write (`MRW`) consists of two back-to-back 4-halfword instructions (`Entry 2k` for `MRW-1` and `Entry 2k+1` for `MRW-2`), where `hw[1] = hw[2] = hw[3] = 0x0000` and `hw[0]` packs the two 7-bit LPDDR5 CA phases (`CA[6:0]` on rising/falling CK edges) plus a 2-bit chip-select mask:
       - **Bits `[6:0]` (rising-edge CA0..CA6)**: `0x58` (`0b1011000`) for `MRW-1`; `0x08 | ((OP & 0x80) >> 1)` (`0x08` when `OP[7] == 0`, or `0x48` when `OP[7] == 1`) for `MRW-2`;
       - **Bits `[13:7]` (falling-edge CA0..CA6)**: `MA[6:0]` (7-bit mode-register address) for `MRW-1`; `OP[6:0]` (low 7 bits of mode-register data) for `MRW-2`;
       - **Bits `[15:14]` (chip-select mask)**: `0b01` (`0x4000`) = rank 0 only, `0b10` (`0x8000`) = rank 1 only, `0b11` (`0xc000`) = broadcast to both ranks.
       ```text
       hw0_mrw1(cs_mask, MA) = ((cs_mask & 3) << 14) | ((MA & 0x7f) << 7) | 0x0058
       hw0_mrw2(cs_mask, OP) = ((cs_mask & 3) << 14) | ((OP & 0x7f) << 7) | ((OP & 0x80) >> 1) | 0x0008
       ```
   - **Inter-command NOP / cycle-delay entry**:
     - Following each command pair, the NOP delay helper writes a 4-halfword delay entry where `hw[0] = hw[1] = hw[2] = 0x0000` and `hw[3] = ((cycles & 7) << 12) | 0x0b00` (for example `0x2b00` for a 2-unit delay, where the cycle count is divided by the 16-bit divisor at DMEM `0x8000045a = 4`).
   - **Verification against the reference ACSM capture (`training_envelope.bin`)**:
     - Decoding the first non-zero halfwords of the reference 6400 MT/s ACSM snapshot reproduces the exact initial message-block mode-register values:

     | Halfword offset (`0x41000 + off`) | Raw halfwords (`hw[0..3]`) | Decoded ACSM instruction |
     |---:|---|---|
     | `0..7` | `0000 0000 0000 0000` × 2 | Initial 2-instruction NOP slot (`cursor = 0..7`) |
     | `8..11` | `c9d8 0000 0000 0000` | `MRW-1`: `cs = 3` (both ranks), `MA = 0x13` (**`MR19`**) (`51672` decimal in reference sample) |
     | `12..15` | `c808 0000 0000 0000` | `MRW-2`: `cs = 3`, **`OP = 0x10`** (`MR19 = 0x10`) |
     | `16..23` | `0000 0000 0000 2b00` + `0000...` | Inter-command NOP delay (`2` units) |
     | `24..31` | `c958 ...` / `c688 ...` | `cs = 3`, **`MR18 = 0x0d`** (`MA = 0x12`, `OP = 0x0d`) |
     | `40..47` | `c0d8 ...` / `d848 ...` | `cs = 3`, **`MR1 = 0xb0`** (`MA = 0x01`, `OP = 0xb0`) |
     | `56..63` | `c158 ...` / `ddc8 ...` | `cs = 3`, **`MR2 = 0xbb`** (`MA = 0x02`, `OP = 0xbb`) |
     | `72..79` | `c1d8 ...` / `c088 ...` | `cs = 3`, **`MR3 = 0x01`** (`MA = 0x03`, `OP = 0x01`) |
     | `88..95` | `c558 ...` / `ec08 ...` | `cs = 3`, **`MR10 = 0x58`** (`MA = 0x0a`, `OP = 0x58`) |
     | `104..111` | `c5d8 ...` / `f188 ...` | `cs = 3`, **`MR11 = 0x63`** (`MA = 0x0b`, `OP = 0x63`) |
     | `120..127` | `48d8 ...` / `5748 ...` | `cs = 1` (rank 0), **`MR17 = 0xae`** (`MA = 0x11`, `OP = 0xae`) |

#### Summary of `DevInit`-written CSRs outside `phypara`

The following CSRs are written by `DevInit` to sequence hardware state machines, resets, and the ACSM sequencer, and are **not** part of the 5,170-entry `phypara` table:

- **Group `0x00`**: `0x0000a4` (`ARC 0x90000148`);
- **Group `0x01` broadcast (`0x01fxxx`)**: `0x01f00a` (`0x9003e014`), `0x01f00f` (`0x9003e01e`), `0x01f0ac` (`0x9003e158`), `0x01f0d9` (`0x9003e1b2`);
- **Group `0x02` control/reset/ACSM**: `0x020020`–`0x020023`, `0x020027`, `0x020028`, `0x02002a`, `0x020041`, `0x020042`, `0x020044`, `0x020049`, `0x020053`, `0x020060` (`MemResetL`), `0x020061`, `0x020070`–`0x020073`, `0x020078`, `0x02007a`, `0x02007e`, `0x02008b`, `0x0200a2`, `0x0200c3`, `0x0200e0`–`0x0200e3`, `0x0200e5`, `0x0200f8`, `0x02011c` (`0x90040238` terminal status pulse), `0x020200`, `0x020303`, `0x02030d`, `0x020310`, `0x020311`, `0x020320`, `0x02f077`, `0x02f0a0`, `0x02f0a5`;
- **Group `0x0c`**: `0x0c0080` (`UCCLK_HCLK_ENABLES`).

By contrast, in the `0x0200c0`–`0x0200cf` range, `CSR[0x0200c3]` (`0x0001`), `CSR[0x0200c6]` (`0x0010`), `CSR[0x0200cd]` (`0x0001`), and `CSR[0x0200cf]` (`0x001f`) hold hardware power-on reset values that are never read by the PMU (`0x0200c5`, `0x0200c7`, `0x0200ca`, and `0x0200cc` are unimplemented and read as `0`). Similarly, `CSR[0x09080b]` and `CSR[0x09080c]` (which change from `0x002f`/`0x006f` in the pre-training table to `0x000f`/`0x000f` in the reference post-training `phypara`) are updated by internal PHY/ACSM hardware state transitions rather than direct PMU register writes. In groups `0x0c` and `0x0d`, the only registers accessed by the PMU outside the four mailbox registers are `CSR[0x0c0080]` (`ARC 0x90180100`) and `CSR[0x0c0086]` (`ARC 0x9018010c`); neither `CSR[0x0d0036]` nor P-state SRAM (`0x0a0000`) is accessed during `DevInit`.

## Candidate evaluation and eye construction

A behavioral training stage performs the following operations:

1. Select a candidate delay, VREF, DCA, DFE, or latency configuration.
2. Apply it to the correct channel/rank/lane or group.
3. Transfer the selected training pattern and compare the returned data.
4. Repeat comparisons for the configured number of bit times.
5. Classify the candidate and construct passing/failing regions.
6. Select an interior operating point rather than simply the first passing point.
7. Program the selected value and publish applicable result/margin fields.

`BitTimeControl[2:0]=n` selects `2^n` times the base comparison count per candidate. The base count, mismatch tolerance, PRBS/pattern lengths, and whether any mismatches are allowed are unknown. Thus, this document cannot define a bit-error-rate threshold for reference compatibility.

For 2D training, candidates span delay and voltage. The selected point optimizes a combined margin:

```text
score = delay_margin * delay_weight + voltage_margin * voltage_weight
```

Separate RX and TX weight pairs occupy message offsets `0x20/0x21` and `0x17/0x18`. Each weight is 0–31. One may be zero; if both are zero, each is treated as 1.

`Train2DMisc` additionally controls iteration count, seeds, contour output, optimization method, and RX VREF runtime range selection. The exact optimizer need not be copied to satisfy the external contract, but its resulting CSR/MR state and reported margins must be coherent.

## Margins and result interpretation

Each populated channel/rank reports four one-byte margins:

| Field | Meaning | Unit |
|---|---|---|
| `RxClkDly_Margin` | Selected RX timing center to nearest failing region | PHY DLL/tap steps |
| `VrefDac_Margin` | Selected RX VREF center to nearest failing region | PHY DAC steps |
| `TxDqDly_Margin` | Selected TX DQ timing center to nearest failing region | PHY DLL/tap steps |
| `DeviceVref_Margin` | Selected device VREF center to nearest failing region | DRAM DAC steps |

Each is the **minimum across all eyes in that timing group**, not an average and not the full eye width. For one dimension with selected coordinate `c`, nearest failing coordinates `L < c` and `R > c`, the conceptual value is:

```text
margin = min(c - L, R - c)
```

The exact treatment of scan boundaries, unobserved failures, truncated eyes, rounding, and saturated values is not established. Do not infer `2 * margin` as a standards-compliant eye width without resolving those cases.

`DFIMRLMargin` is different: it is an input safety increment in DFI clocks added to the smallest passing maximum-read-latency result. It must account for maximum expected positive `tDQSCK` drift over operating voltage and temperature; the interface recommends at least 1. This is not one of the four output eye margins.

Trained VREF/DFE/DCA outputs are copied to MR12/14/15/24/30 inputs for quickboot as specified in `05-message-block.md`. The final live CSR and MR state, not the streamed debug text alone, is authoritative.

## Observability and diagnostics

Streaming messages expose internal measurements but do not replace result collection. A reference full-training capture across two controllers contained 9,153 streamed messages and 124 distinct IDs. Confirmed framing is specified in `08-mailbox-and-streaming.md`.

Useful but incomplete observations include:

- ID `0x007e`, payload length 31: a packed per-bit DQ-related array; older notes describing “32 values” should not be read as 32 payload words.
- ID `0x0104`, length 1: emitted in loops over a 128-byte array; 1,024 occurrences were captured.
- IDs `0x00b1` and `0x011a`: pack eight bit results into a byte.
- ID `0x0042`, length 7: per-lane DQ report with an index and VREF/delay-related values.
- IDs `0x01e0` and `0x01e9`: frequency/PLL-related reports containing `0x1900` (6400).
- IDs `0x0076` (length 4) and `0x0077` (length 3), emitted when `Train2DMisc[0] = 1` and `HdtCtrl = 0x05`: 2D eye contour header (`0x00760004 <ctrl> <rank> <sub> <lane>`, ordered with `lane = 0..8` varying fastest, then `sub = 0..17`, `rank = 0..3`, `ctrl = 0..1`) followed by 64 row-slice messages (`0x00770003 <left_edge> <right_edge> <vref_step>` for `vref_step = 0..63`, where consecutive step pairs repeat across 32 distinct voltage levels, and `<left_edge>` / `<right_edge>` are 7-bit delay bounds `0..127`). Across two controllers on the reference 6400 MT/s board, a full run emits **1,296 contours** (`2 controllers × 4 ranks × 9 lanes × 18 sub-sweeps`) with:
  - horizontal eye opening `right - left` between **70 and 124 delay taps** (median **102 taps**);
  - vertical eye opening between **21 and 32 voltage levels** out of 32 (median **27 levels**);
  - horizontal contour midpoint `(left + right) / 2` between **37.4 and 77.9 taps** (median **62.5 taps**). Because this 2D contour sweeps the DQS/gate axis (`~62.5` taps) whereas `CSR[0x01<r><l>4e]` programs the final per-lane DQ read delay (`0x2d`–`0x4a`, i.e. 45–74 taps), the `0x0076`/`0x0077` contour midpoint is a diagnostic view of the 2D scan rather than the literal integer written into `CSR[0x4e]`. Note also that reducing `Train2DMisc[7:6]` from 2 seeds (`0`) to 1 seed (`1`) reduces training time by ~342 ms while producing identical `CSR[0x4e]`/`0x4f` centers within run-to-run noise.

Payload schemas for most IDs remain unknown. Do not use these inferred meanings as acceptance inputs.

Observed firmware safeguards include a PHY signature/version check, a diagnostic when a useful window is smaller than 12 (not necessarily fatal), and a diagnostic after 64 intervals without valid samples. Their presence does not establish a universal margin threshold or retry policy.

## Completion, failure, and acceptance

### Observed reference behavior

- Enabled stages are selected by `SequenceCtrl`; the PMU emits major `0x07` on normal completion and `0xff` on its fatal path.
- `CsTestFail=0` means no rank failure was reported; nonzero means at least one rank failed.
- The host collects message, PHY-parameter, and ACSM results after the PMU phase.
- The exact rule by which a candidate or complete stage is accepted is unknown. In particular, no minimum values for the four output margins, no universal mismatch allowance, and no complete convergence/retry threshold have been recovered.
- The reference error-return path is inconsistent and is not reliable acceptance evidence; see `08-mailbox-and-streaming.md`.

### Recommended acceptance for a new implementation

This is new policy, not recovered reference behavior:

1. Require normal completion, no PMU timeout/protocol error, and `CsTestFail=0` for every populated rank.
2. Verify that every enabled stage produced all required live CSR, MR, message, PHY-parameter, and ACSM results.
3. Choose explicit per-margin minima from PHY/DRAM documentation and board signal-integrity validation. Record the actual margins with the result. Until those minima are established, report margins as diagnostic data rather than inventing a numeric pass threshold.
4. Bound retraining by iteration count and time. Fail if no passing region exists, results do not converge, or a platform margin minimum is not met.
5. Revalidate when data rate, topology, electrical parameters, firmware/result format, or operating corners change.
6. Run the post-activation memory smoke test independently. A wide reported eye does not prove DDRC address mapping is correct; a smoke-test pass does not prove adequate signal margin.

### Concrete three-level eye acceptance model (recommended design)

Because the reference firmware's internal numeric thresholds are not part of the external hardware contract, a clean-room PMU training implementation should structure candidate and eye evaluation into three explicit levels:

1. **Level 1 — Single-point evaluation `(delay, vref)`**:
   - Program the candidate delay tap and VREF code on the target channel/rank/lane and wait for DAC/DLL settling.
   - Execute `N = base_samples * 2^(BitTimeControl & 0x7)` transfers using patterns that stress inter-symbol interference and simultaneous switching output (for example alternating `0xaa`/`0x55`, walking-1, and walking-0 patterns, with `base_samples >= 64`).
   - Require **zero bit errors** across all `N` comparisons for a candidate point to be classified as `PASS`.
2. **Level 2 — Contiguous passing window and center selection**:
   - Along a 1D delay scan at fixed VREF, identify the longest contiguous run of `PASS` taps `[d_left, d_right]`, discarding isolated 1–2 tap islands caused by noise.
   - Require a minimum window width `W = d_right - d_left + 1 >= W_min` (for example `W_min >= 8` delay taps for DQ and `>= 6` taps for CA/CS) and a minimum vertical opening `H >= H_min` (for example `>= 5` VREF steps in 2D scans). Note that the reference firmware emits a non-fatal warning when a window is narrower than 12 steps.
   - Select the midpoint `d_center = (d_left + d_right) / 2` in 1D, or the point maximizing `delay_margin * delay_weight + voltage_margin * voltage_weight` inside the passing interior in 2D.
3. **Level 3 — Final group margin verification and bounded retry**:
   - Compute the four per-rank minimum margins (`RxClkDly_Margin`, `VrefDac_Margin`, `TxDqDly_Margin`, `DeviceVref_Margin`) as `min_lane min(c - L, R - c)`.
   - Verify that every populated rank satisfies `RxClkDly_Margin >= M_rx_min`, `TxDqDly_Margin >= M_tx_min`, `VrefDac_Margin >= M_vref_min`, and `DeviceVref_Margin >= M_dvref_min` (starting with conservative floors such as `>= 4` delay steps and `>= 2` VREF steps, refined by thermal/voltage corner testing).
   - If any rank/lane fails Level 2 or Level 3, allow at most `R_max` retraining iterations (for example `R_max = 3`, unless `DisableTrainingLoop` is set). If still failing, set the rank's bit in `CsTestFail` (`0x0f`), pulse `0x90040238` (`4 -> 0`), and emit fatal mailbox code `0x000000ff`.

## Unresolved details

A fully cycle-compatible independent trainer still requires:

- exact pass/fail comparison statistics and base bit-time count;
- scan ordering, step rounding, truncated-eye handling, and center-selection details for each stage;
- numeric margin/acceptance thresholds and convergence/retry criteria;
- full streamed payload schemas and major-stage-to-algorithm mapping;
- precise tap/DAC units and CSR programming sequences for every result.
