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
| `0x0001` | `DevInit` | 15 | 68 (507 total CSR write ops) | Executes 6-phase PHY/DRAM initialization (507 CSR writes across groups `0x00`–`0x09`), calibrates group `0x02` master clock period (`0x02000a`, `0x020017`, `0x020012`, `0x020186`, `0x020187`, `0x020011`), reads back `QBPll*` (`0x80000130`–`0x8000013c`), masks `0x09080b`/`0x09080c` (`& 0xff9f` → `0x000f`), and transitions hardware status `0x0c0086` (`0x0003`) |
| `0x0007` | `+ WrLvl + RxEn` | 28 | 165 | Writes per-lane write-leveling and receive-enable coarse/fine CSRs (`0x01000c/0d = 0x009a/0x009b`, `0x010026 = 0x00ba`, etc.) |
| `0x000f` | `+ RdDQS` | 3,567 | 559 | **63% of total runtime (~3.54 s):** trains `CSR[0x4e]` and `CSR[0x4f]` across all 36 lanes (4 ranks × 9 lanes) and populates `RxClkDly_Margin` / `VrefDac_Margin` |
| `0x001f` | `+ WrDq` | 4,413 | 598 | **~846 ms:** trains write DQ eye delays (`0x01<r><l>10..13`, `0x01<r><l>be`) and populates `TxDqDly_Margin`, `DeviceVref_Margin`, and trained `MR14`/`MR15` |
| `0x005f` | `+ DRAM DCA` | 5,611 | 484 | **~1.20 s:** adjusts per-bit transmit delays (`0x01<r><l>68..6f` shifted by ~+4 taps) and updates `MR30` |
| `0x025f` | `+ MxRdLat` | 5,613 | 503 | **~2 ms:** updates DFI maximum read latency CSRs (`0x070011`, `0x070060..63`) and `0x090813` |
| `0x125f` | `+ LPCA` (full mask) | 5,624 | 587 | **~11 ms:** trains CA VREF (`MR12`) and LP5 CS timing (`0x030001 = 0x0004`) |

Note that running `SequenceCtrl = 0x0009` (`DevInit + RdDQS` only, skipping `WrLvl` and `RxEn` because the pre-training tables already hold nominal leveling values) completes in ~3.55 s and produces `CSR[0x4e]`/`CSR[0x4f]` values within 1–2 taps of a full `0x125f` run.

### `DevInit` master clock-period calibration (`0x02xxxx`)

Before invoking the master clock-period calibration routine during `DevInit` (`SequenceCtrl[0]`), four hardware preconditions must hold:

1. **DPLL locked and DFI clock running** (configured by the host before releasing the PMU);
2. **Pre-`DevInit` PMU startup, hardware IP version decode (`CSR[0x0c00ee]`), 6-byte descriptor table (`0x800004cc`), and `0x80000400`–`0x800009a3` parameter tables preserved**: Before Phase 1 begins, top-level PMU startup initializes `CSR[0x020090] = 1` (`ARC 0x90040120`), decodes the hardware PHY IP version from `CSR[0x0c00ee]` (`ARC 0x901801dc`) into `byte[0x80000403]` (`8`–`10` on K3 hardware), verifies the 32-bit PHY signature canary `0x00f26745` at 16-bit registers `CSR[0x09002e]`/`0x09002f` against `halfword[0x80000424]` (`0x6745`) and `halfword[0x80000426]` (`0x00f2`), initializes `word[0x80000410] = 2`, `word[0x8000040c] = 4`, `byte[0x80000a80] = 1`, `byte[0x80000b68] = (DRAMFreq + DfiFreqRatio * 220 - 1) / (DfiFreqRatio * 220)` (`8` at 6400 MT/s with `DfiFreqRatio = 4`), `byte[0x80000b69] = 4`, and active rank bounds `byte[0x80000b6e..0x80000b71]`, and at the start of Phase 3 applies the three 6-byte `[uint32_t csr_idx, uint16_t val]` descriptors pre-loaded at DMEM `0x800004cc`–`0x800004dd` (`CSR[0x02006e] = 0x0001`, `CSR[0x070060] = 0x0000`, `CSR[0x070011] = 0x0000`);
3. **PHY block-enable, clock-shift setup, DQ block reset, DRAM/PLL reset (`400 µs` stabilization wait), and per-rank DQ calibration (`Step 3.6`) completed** (`CSR[0x0200a2]` and `CSR[0x0200f8]` configured, `0x80000404` shift initialized from `CSR[0x0c0080]`, and `Step 3.4`–`Step 3.6` completed; see below);
4. **`CSR[0x020011]` (`ARC 0x90040022`) bits `[6:5]` cleared during `DevInit` and restored at `Step 6.3`**: The host pre-training table initializes `CSR[0x020011]` to **`0x006f`**. When `Quickboot == 0` (`byte[0x80000019] == 0`), at `Step 3.3` the PMU saves the original value (`0x006f`) to DMEM `0x8000042a` and writes `0x006f & 0xff9f = 0x000f` to `CSR[0x020011]`. Later, at the end of `DevInit` (`Step 6.3`, also gated on `Quickboot == 0`), the PMU restores `CSR[0x020011]` from DMEM `0x8000042a` (`0x006f` across all clean USB/FDL full-training captures; see `Step 6.3` and [14-validation-and-current-status.md](14-validation-and-current-status.md) for why the factory SPI envelope holds `0x000f` via an earlier/quickboot snapshot without that restore active).

Once those preconditions hold, the PMU calibrates the clock period and frequency-bucket CSRs using the following sequence:

1. **3-step period-counter strobe with bit 10 falling-edge latch (`CSR[0x0200e5]` and `CSR[0x0200e3]`)**:
   - The period counters at `CSR[0x0300bf]` (`ARC 0x9006017e`) and `CSR[0x0100e0]` (`ARC 0x900201c0`) do **not** latch on a single level write. Simply writing `CSR[0x0200e3] = old_e3 | 0x0c00` without subsequently clearing bit 10 back to `0` leaves both registers at their unlatched reset/pre-training state `0x0240` (`0x0240 & 0x01ff = 0x0040`).
   - Instead, the hardware requires clearing bits `[4:0]` of `CSR[0x0200e5]` (`ARC 0x900401ca`) **before** pulsing bit 10 (`0x0400`) of `CSR[0x0200e3]` (`ARC 0x900401c6`) high and then low while bit 11 (`0x0800`) remains asserted:
     1. Read and save `old_e3 = CSR[0x0200e3]` (`ARC 0x900401c6`) and `old_e5 = CSR[0x0200e5]` (`ARC 0x900401ca`). Neither `0x0200e3` nor `0x0200e5` is in `phypara`, and both are restored in step 4 so their net change is zero.
     2. Write `CSR[0x0200e5] = old_e5 & ~0x001f` (`old_e5 & 0xffe0`, clearing bits `[4:0]`).
     3. Write `CSR[0x0200e3] = old_e3 | 0x0800` (assert bit 11 = counter enable, with bit 10 = `0`).
     4. Write `CSR[0x0200e3] = old_e3 | 0x0c00` (assert bit 10 = `1` while keeping bit 11 = `1`).
2. **Short measurement delay (`delay_dfi_clks(10)`) and falling-edge latch**:
   - Invoke the direct DFI-clock delay helper `delay_dfi_clks(10)` (`n = 0x0a`).
   - **Zero-RTC-wait scaled delay behavior:** `delay_dfi_clks(n)` loads a 1-bit shift from DMEM byte `0x80000404` (`gp + 4`), initialized during Phase 2 from `CSR[0x0c0080]` (`UCCLK_HCLK_ENABLES`, `ARC 0x90180100`) as `shift = 1 & ~(CSR[0x0c0080] >> 2)` (`shift = 1` when `0x0c0080 == 3`, or `shift = 0` when `0x0c0080 == 7`), and computes `rtc_ticks = (n + 1) >> shift`. Only when `rtc_ticks >= 0x10` (`16`) does the helper poll the 64-bit hardware RTC (`AUX 0x103`–`0x105`). When `rtc_ticks < 16` (for `n = 10`, `rtc_ticks` is `5` or `11`), it performs **one 16-bit dummy read of `CSR[0x020024]` (`ARC 0x90040048`) and returns immediately** without touching the RTC. (Note that `0x80000404` is distinct from `0x8000045a = 4`, which is used by the ACSM NOP cycle counter.)
   - Immediately after `delay_dfi_clks(10)`, write **`CSR[0x0200e3] = old_e3 | 0x0800`** (clearing bit 10 back to `0` while keeping bit 11 = `1`). **This `1 -> 0` falling edge on bit 10 synchronously latches the period counters.**
3. **Read 9-bit period counters and program `CSR[0x02000a + pstate]` and `CSR[0x020017 + pstate]`**:
   - Read CA period counter `T_ca = CSR[0x0300bf] & 0x01ff` (`ARC 0x9006017e`) and write `T_ca` to `CSR[0x02000a + pstate]` (`ARC 0x90040014 + 2 * pstate`).
   - Read DQ period counter `T_dq = CSR[0x0100e0] & 0x01ff` (`ARC 0x900201c0`) and write `T_dq` to `CSR[0x020017 + pstate]` (`ARC 0x9004002e + 2 * pstate`).
   - **Reference values, bit 9 (`0x0200`), and saturation (`0x17ff` / `0x01ff`) interpretation:** The host pre-training table initializes `CSR[0x02000a]`, `CSR[0x020017]`, and `CSR[0x020187]` to `0x0040`. Because the PMU masks `0x0300bf` and `0x0100e0` with `& 0x01ff` (9 bits), upper status bits (`0x0200`, `0x1000`, `0x0400`) are discarded:
     - Reading raw **`0x0240`** (`0x0240 & 0x01ff = 0x0040`) indicates the counter **never latched** (missing bit 10 falling edge on `CSR[0x0200e3]`).
     - Reading raw **`0x17ff`** (`0x17ff & 0x01ff = 0x01ff = 511`) indicates the counter **enabled and latched**, but **saturated at full scale (`0x01ff`)** because the target clock was stopped or unstable during measurement (for example, when the `0x800004cc` descriptor table was skipped, the `400 µs` PLL stabilization wait in `Step 3.5.12` was shortened to `12.5 µs` because `byte[0x80000a80]` / `0x80000b68` / `0x80000b69` were not initialized, or `Step 3.6` was omitted).
     - On the reference 6400 MT/s board, when the PHY PLL and `Step 3.5`–`3.6` complete normally, `CSR[0x02000a]` latches **`0x0082`** (`130`, or **`0x0086`** = `134` in a `SequenceCtrl = 0x0001` `DevInit`-only capture), whereas `CSR[0x020017]` remains **`0x0040`** (`pre-unchanged`).
4. **Restore counter control registers**:
   - Write `CSR[0x0200e3] = old_e3` and `CSR[0x0200e5] = old_e5`.
5. **Compute frequency buckets from `DRAMFreq` and `DfiFreqRatio` and program `CSR[0x020012]`, `CSR[0x020186]`, `CSR[0x020187]`, and per-rank `CSR[0x01<r>0d9]`**:
   - The frequency-bucket fields in `CSR[0x020012]` (`ARC 0x90040024`) and `CSR[0x020186]` (`ARC 0x9004030c`) are **deterministic functions of the message-block fields `DRAMFreq` (`0x80000006`, 16-bit) and `DfiFreqRatio` (`0x80000008`, 8-bit)**—they do **not** depend on the live counter `T_ca`.
   - Specifically, the PMU computes two scaled frequency values `f_lo` and `f_hi` using unsigned integer division (`divu`):
     ```text
     f    = DRAMFreq / DfiFreqRatio
     f_lo = (DfiFreqRatio == 2) ? (2 * f) : f
     f_hi = f * 6
     ```
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
     - `f_hi = 1600 * 6 = 9600` (`> 2199`) → `bucket3 = 7`, `fine_hi = 3`;
     - `CSR[0x020012] = (7 << 12) | (3 << 8) | 0x50 | 3 = 0x7353` (exact match to reference `phypara` `0x7353`, updated from pre-training `0xa1a1`);
     - `CSR[0x020186] = (7 << 4) | 3 = 0x0073` (exact match to reference `phypara` `0x0073`, updated from pre-training `0x00a1`);
     - `CSR[0x020187] = CSR[0x0100d5] = 0x0041` (exact match to reference `phypara` `0x0041`, updated from pre-training `0x0040`).
   - *(Diagnostic note on `0x3221` / `0x4332` artifacts: If a custom implementation mistakenly classifies `2 * T_ca` instead of `f = DRAMFreq / DfiFreqRatio` for both `f_lo` and `f_hi`, then whenever `T_ca` hovers around `399`–`400` (`0x018f`–`0x0190`), `2 * T_ca = 798..800` straddles the single common threshold `799`, causing all four nibbles to jump by `+1` simultaneously between `0x4242` / `0x5353` or `0x3221` / `0x4332` (`0x4332 - 0x3221 = 0x1111`). Using `DRAMFreq / DfiFreqRatio` eliminates that sensitivity completely.)*

### Two-tier PMU hardware delay model (`delay_dfi_clks` vs. `delay_ps_or_clks`)

The PMU firmware implements **two** delay entry points on top of the 64-bit ARCv2 auxiliary RTC counter (`AUX 0x103`–`0x105`, clocked at **`400 MHz` = `2.5 ns/tick`**):

1. **`delay_dfi_clks(n)` — Direct DFI-clock delay**:
   - Takes `n` in **DFI clock cycles** (`800 MHz` = `1.25 ns/cycle` at 6400 MT/s with `DfiFreqRatio = 4`).
   - Loads `shift = byte[0x80000404]` (`1 & ~(CSR[0x0c0080] >> 2)`, which is **`1`** when `CSR[0x0c0080] == 3`, or `0` when `0x0c0080 == 7`) and computes:
     ```text
     rtc_ticks = (n + 1) >> shift
     ```
   - **Short-delay fast path (`rtc_ticks < 16`)**: Performs **one 16-bit dummy read of `CSR[0x020024]` (`ARC 0x90040048`)** and returns immediately without touching `AUX 0x103`–`0x105`.
   - **Hardware RTC polling path (`rtc_ticks >= 16`)**: Checks `AUX 0x103` (if bit 0 is `0`, writes `2` then `1` to `AUX 0x103`), reads the 64-bit start count from `AUX 0x104` (low) and `AUX 0x105` (high), and polls until **`rtc_ticks` raw 400 MHz RTC ticks** (`1` RTC tick = `2.5 ns` per scaled unit) have elapsed. There is no secondary multiplier or divisor inside `delay_dfi_clks(n)`.

2. **`delay_ps_or_clks(ps, min_clks)` — Picosecond and minimum-clock conversion wrapper**:
   - Takes a physical duration `ps` in **picoseconds** (`10^-12 s`) and a minimum cycle floor `min_clks`, converts both into DFI clock cycles, and delegates to `delay_dfi_clks(n)`:
     1. Reads `DRAMFreq` (`halfword[0x80000006]` = `6400`) and `DfiFreqRatio` (`byte[0x80000008]` = `4`), converts `ps` into CK cycles `v = (ps * DRAMFreq) / 2,000,000` (using 64-bit fixed-point reciprocal multiplication `0x431bde83 >> 51` when `ps <= 200,000`, or `((ps >> 5) * 0x0a7c5ac5 >> 39) * DRAMFreq / 20` when `ps > 200,000` to prevent 32-bit overflow), and divides by `DfiFreqRatio`:
        ```text
        dfi_clks = (v + DfiFreqRatio + 1) / DfiFreqRatio
        ```
     2. Converts `min_clks` into DFI clock cycles `c`: when `byte[0x80000a80] != 0` (set to `1` at PMU startup), `(byte[0x80000001] & 8) == 0`, and `byte[0x80000b68] != 0`, it scales `min_clks` by `byte[0x80000b68] * byte[0x80000b69]` (`8 * 4 = 32` at 6400 MT/s):
        ```text
        c = min_clks * byte[0x80000b68] * byte[0x80000b69]     // 32 * min_clks when 0x80000a80 == 1
        ```
        Otherwise (if `byte[0x80000a80] == 0`), `c = min_clks + 1`.
     3. Calls `delay_dfi_clks(max(dfi_clks, c))`.

This two-tier structure explains why large integer arguments such as `130,000,000` (`130,000,000 ps = 130 µs`) and `10,000,000` (`10,000,000 ps = 10 µs`) complete in microseconds rather than hundreds of milliseconds, while `delay_ps_or_clks(0, 10000)` in `Step 3.5.12` waits `10,000 * 32 = 320,000` DFI clocks (`160,000` RTC ticks = **`400 µs`**) for PHY PLL stabilization:

| `DevInit` call site | Helper and arguments | `dfi_clks` (`n`) at 6400/4 | `rtc_ticks` (`shift = 1`) | Wall-clock duration (`400 MHz` RTC) |
|---|---|---:|---:|---:|
| Phase 1 enable wait | `delay_dfi_clks(4)` then `delay_dfi_clks(0x1f)` | `4`, `31` | `2` (dummy read), `16` | `< 10 ns`, **`40 ns`** |
| Step 3.4 DQ block reset (`Phase 4`) | `delay_ps_or_clks(130000000, 0)` (`130 ms` in ps = `130 µs`) | `104,001` | `52,001` | **`130.0 µs`** |
| Step 3.5.1 pre-PLL delay | `delay_dfi_clks(4)` | `4` | `2` (dummy read) | `< 10 ns` |
| Step 3.5.8 `0x0200e2 = 3` hold | `delay_ps_or_clks(1000000, 0)` (`1,000,000 ps`) | `801` | `401` | **`1.0 µs`** |
| Step 3.5.9 `0x0200e2 = 2` hold | `delay_ps_or_clks(500000, 0)` (`500,000 ps`) | `401` | `201` | **`0.5 µs`** |
| Step 3.5.10 `0x0200e2 = 0` hold | `delay_ps_or_clks(10000000, 0)` (`10,000,000 ps`) | `8,001` | `4,001` | **`10.0 µs`** |
| Step 3.5.11 `0x020078 = 1` reset assert | `delay_dfi_clks(6)` | `6` | `3` (dummy read) | `< 10 ns` |
| Step 3.5.12 `0x020078 = 2`, `0x0200a2 = 2` **PLL lock wait** | `delay_ps_or_clks(0, 10000)` (`0x80000a80 == 1`) | `320,000` | `160,000` | **`400.0 µs`** (`12.5 µs` if `0x80000a80 == 0`) |
| Step 3.5.13 `0x020078 = 0` settle | `delay_ps_or_clks(0, 0x20)` (`0x80000a80 == 1`) | `1,024` | `512` | **`1.28 µs`** (`42.5 ns` if `0x80000a80 == 0`) |
| Step 3.6 per-rank DQ calibration setup | `delay_dfi_clks(0x3000)` | `12,288` | `6,144` | **`15.36 µs`** |
| Step 3.7 period-counter gate window | `delay_dfi_clks(10)` | `10` | `5` (dummy read) | `< 10 ns` |
| Step 6.2 `tINIT3` before `0x020060 = 3` (`byte[0x80000001] & 2` set / clear) | `delay_ps_or_clks(2000000, 0)` / `(200000000, 0)` | `1,601` / `160,001` | `801` / `80,001` | **`2.0 µs`** / **`200.0 µs`** |
| Step 6.2 `tINIT4`/`tINIT5` after `0x020060 = 3` (`byte[0x80000001] & 2` set / clear) | `delay_ps_or_clks(2000000, 0)` / `(2000000000, 0)` | `1,601` / `1,600,001` | `801` / `800,001` | **`2.0 µs`** / **`2.0 ms`** |
| Step 6.2 quickboot/skip-MRW hold | `delay_ps_or_clks(2125, 4)` | `128` | `64` | **`160 ns`** |
| Step 6.2 post-ZQ `MPC` settle | `delay_ps_or_clks(30000, 0x1e)` | `960` | `480` | **`1.20 µs`** |

### Complete `DevInit` (`SequenceCtrl[0]`) hardware initialization sequence

While comparing a `SequenceCtrl = 0x0001` capture against `0x0000` shows **68 changed `phypara` entries** (50 in group `0x01`—namely `0x01<r><l>50` and `0x51` across all 36 lanes plus `0x01<r>0a3` and `0x0100d5`—along with 16 in group `0x02` and `0x09080b`/`0x09080c`), the complete `DevInit` stage actually executes **507 CSR write operations** across six ordered hardware phases (`Phase 1`: 3 writes; `Phase 2`: 3 writes; `Phase 3` + `Step 3.6` + `Step 3.7`: 77 writes; `Phase 4` / `Step 3.4`: 10 writes; `Phase 5` / `Step 3.5`: 56 writes; `Phase 6`: 218 writes; plus ~40 three-write commit strobe triads). Many of those writes target dynamic control, reset, and strobe registers outside the 69 `phypara` serialization segments.

#### Broadcast alias `0x01f0xx` (`ARC 0x9003e0xx`), configuration commit strobe triad (`0x020028`, `0x02f0a5`, `0x02f077`), and channel-commit pulse (`0x02f0a0`)

- **All-rank write broadcast (`CSR[0x01f0xx]` / `ARC 0x9003e000 + 2 * xx`)**: Within group `0x01`, rank index `0xf` (`0x01f0xx`, or `0x01f<l>xx`) is a **write-only hardware broadcast alias** that simultaneously updates all four ranks `r = 0..3` (`CSR[0x01<r>0xx]`, `ARC 0x90020000 + r * 0x2000 + 2 * xx`). Because `0x01f0xx` is write-only, the PMU always reads the current value from rank 0 (`CSR[0x0100xx]`) or rank `r` (`CSR[0x0100xx + r]`), modifies the target bitfield, and writes the result to `CSR[0x01f0xx]`.
- **Configuration commit strobe triad (`0x020028`, `0x02f0a5`, `0x02f077`)**: Whenever the PMU finishes configuring a logical initialization step or functional CSR block and needs to latch/commit the configuration into the hardware state machine, it issues this three-write strobe sequence (~40 times across `DevInit`, always at the end of a logical step—not per individual serialization segment):
  ```text
  CSR[0x020028] = 1    (ARC 0x90040050 = 1)
  CSR[0x02f0a5] = 1    (ARC 0x9005e14a = 1)
  CSR[0x02f077] = 0    (ARC 0x9005e0ee = 0)
  ```
  The write order **`0x020028` → `0x02f0a5` → `0x02f077`** is strict, with **no delay between the three consecutive stores** (any required settle `delay()` runs *after* the triad completes). Note that `CSR[0x020077]` (`ARC 0x900400ee`) is **never written directly by the PMU**; its `1 -> 0` transition observed in `phypara` is driven by the PHY hardware when `CSR[0x02f077] = 0` is strobed.
- **Channel/rank timing commit pulse (`CSR[0x02f0a0]` / `ARC 0x9005e140`)**: Whenever the PMU finishes updating per-channel/per-rank ACSM muxing or lane timing registers (called from 12 sites across `Step 6.2`, `Step 6.3`, and subsequent training stages), it pulses `CSR[0x02f0a0]`:
  ```text
  CSR[0x02f0a0] = 1    (ARC 0x9005e140 = 1)
  delay_dfi_clks(0x20)
  CSR[0x02f0a0] = 0    (ARC 0x9005e140 = 0)
  ```

#### Ordered `DevInit` phases

0. **Phase 0 — Pre-`DevInit` top-level PMU startup**:
   - Write `CSR[0x020090] = 1` (`ARC 0x90040120`).
   - Test bit 0 of **`MsgMisc` (`byte[0x80000001] & 1`)** (note that message-block offset `0x00` at `byte[0x80000000]` is `Reserved00 = 0x00`, whereas **offset `0x01` at `byte[0x80000001]` is `MsgMisc = 0x01`** on the reference profile, matching [05-message-block.md](05-message-block.md)). Because `(byte[0x80000001] & 1) != 0` (`MsgMisc[0] == 1`, `MTESTEnable`), this branch **is taken on the reference board**: write `0` to `CSR[0x03f01a]` (`ARC 0x9007e034`), `0` to `CSR[0x01f01a]` (`ARC 0x9003e034`), `0` to `CSR[0x02001a]` (`ARC 0x90040034`), `0` to `CSR[0x0200aa]` (`ARC 0x90040154`), and `3` to `CSR[0x02001a]` (`ARC 0x90040034`).
   - Issue the configuration commit strobe triad (`CSR[0x020028] = 1`, `CSR[0x02f0a5] = 1`, `CSR[0x02f077] = 0`); because `(byte[0x80000001] & 1) != 0` (`MsgMisc[0] == 1`), also pulse `CSR[0x02f052]` (`ARC 0x9005e0a4`: `1 -> delay_dfi_clks(10) -> 0`).
   - **Decode hardware PHY IP version from `CSR[0x0c00ee]` (`ARC 0x901801dc`) into `byte[0x80000403]`**: Read `CSR[0x0c00ee]` (`ARC 0x901801dc`) and `byte[0x80000002]` (`0` before `PmuRevision` is written). Because `byte[0x80000002] == 0`, decode `ver = CSR[0x0c00ee] & 0x0fff` (`0x0101..0x0105` → `2..6`, `0x0110` → `7`, `0x0200` → `8`, `0x0240` → `9`, else `10`) and write the resulting version code (**`8`–`10` on K3 hardware**) to **`byte[0x80000403]` (`gp + 3`)**.
   - **Verify the 32-bit PHY signature canary (`0x00f26745`)**: Because `byte[0x80000403]` is now non-zero (`8`–`10`), the PMU **always selects the 16-bit register pair `CSR[0x09002e]` and `CSR[0x09002f]` (`ARC 0x9012005c` and `0x9012005e`, at the start of `phypara` Segment 48)**—never `CSR[0x090029]`/`0x09002a` (`ARC 0x90120052`/`0x90120054`, which is a 9-bit register that truncates `0x6745` to `0x6745 & 0x01ff = 0x0145` if `byte[0x80000403]` is left at `0`). It writes `0x6745` to `CSR[0x09002e]` and `0x00f2` to `CSR[0x09002f]`, reads them back, and compares them against the 32-bit canary comparison words `halfword[0x80000424]` (`0x6745`) and `halfword[0x80000426]` (`0x00f2`) pre-loaded by the host in DMEM (`0x80000424` is strictly the canary comparison word, not an ODT setting); if either halfword mismatches, it aborts with fatal mailbox code `0x000000ff`.
   - Write `PmuRevision = 0x2409` to message-block offset `0x80000002`.
1. **Phase 1 — PHY block enable (`CSR[0x0200a2]`, `CSR[0x0200f8]`, and `CSR[0x020060]`)**:
   - Read-modify-write `CSR[0x0200a2]` (`ARC 0x90040144`): `CSR[0x0200a2] = (CSR[0x0200a2] & 0xfffc) | 0x0001` (set clock-select bits `[1:0] = 1`, selecting the reference/bypass clock before PLL bring-up).
   - Call `delay_dfi_clks(4)` (one dummy read of `CSR[0x020024]`), then `delay_dfi_clks(0x1f)` (`16` RTC ticks = `40 ns`).
   - Write `CSR[0x0200f8] = 1` (`ARC 0x900401f0`).
   - If `Quickboot == 0` (`byte[0x80000019] == 0`) and `MsgMisc[7] == 0` (`(byte[0x80000001] & 0x80) == 0`), write `CSR[0x020060] = 2` (`ARC 0x900400c0`, pre-reset `MemResetL` state).
2. **Phase 2 — Topology counts (`0x8000040c`), clock-shift initialization (`0x80000404`), and rank bounds (`0x80000b6e..0x80000b71`)**:
   - Compute the channel and rank counts in DMEM (`gp = 0x80000400`): set `word[0x80000410] = u7` (`2` channels) and `word[0x8000040c] = u7 * u0 = 4` (`4` total ranks across ChA + ChB when `CsPresentChB != 0`, or halved to `2` when `CsPresentChB == 0`), and set `byte[0x80000a80] = 1`.
   - Read `CSR[0x0c0080]` (`ARC 0x90180100`), compute `shift = 1 & ~(CSR[0x0c0080] >> 2)`, and store `shift` to DMEM byte `0x80000404` (`gp + 4`). *(Note: this single `ldh` is the **only** access to `CSR[0x0c0080]` in the entire PMU firmware; the PMU never writes `0x0c0080`. The `2 -> 3 -> 7` clock-enable sequence on `0x0c0080` is executed exclusively by the host SPL.)*
   - Issue the configuration commit strobe triad (`CSR[0x020028] = 1`, `CSR[0x02f0a5] = 1`, `CSR[0x02f077] = 0`).
   - Initialize the active lane range (`byte[0x80000b6c] = 0`, `byte[0x80000b6d] = 8`), the frequency-derived cycle multipliers `byte[0x80000b68] = (DRAMFreq + DfiFreqRatio * 220 - 1) / (DfiFreqRatio * 220)` (`8` at 6400/4) and `byte[0x80000b69] = 4`, and the active rank index ranges `byte[0x80000b6e..0x80000b71]` from `EnabledDQsChA` (`byte[0x80000024]` = `16`), `EnabledDQsChB` (`byte[0x8000003f]` = `16`), `CsPresentChB` (`byte[0x80000040]` = `3`), and `word[0x8000040c]` (`4`):
     ```text
     nA = (EnabledDQsChA + 7) >> 3                       // (16 + 7) >> 3 = 2 on x16
     byte[0x80000b6e] = 0                                // ChA first rank = 0
     byte[0x80000b6f] = nA - 1                           // ChA last rank  = 1
     if (CsPresentChB == 0) {
         byte[0x80000b70] = nA                           // 2 (empty range: start > end)
         byte[0x80000b71] = nA - 1                       // 1
     } else {
         u = word[0x8000040c] >> 1                       // 4 >> 1 = 2
         byte[0x80000b70] = u                            // ChB first rank = 2
         byte[0x80000b71] = ((EnabledDQsChB + 7) >> 3) + u - 1   // ChB last rank = 3
     }
     ```
3. **Phase 3 — `QBPll*` read-back, `0x800004cc` descriptor table, topology setup, DQ block reset (`Phase 4`), DRAM reset (`Phase 5`), per-rank DQ sweep, and master clock-period calibration**:
   - **Step 3.1 (`QBPll*` read-back and `0x800004cc` descriptor application)**:
     - Test bit 5 of **`Pstate` (`byte[0x80000004] & 0x20`)**—not `DfiFreqRatio` (`0x80000008`). When `(byte[0x80000004] & 0x20) != 0` (`Pstate == 0x30` on the reference profile), copy the seven `QBPll*` CSRs (`CSR[0x020088]`, `0x020089`, `0x02008a`, `0x02008c`, `0x02004a`, `0x02004b`, `0x02004c`) to message-block offsets `0x80000130`–`0x8000013c` (see [05-message-block.md](05-message-block.md)).
     - Apply the three 6-byte `[uint32_t csr_idx, uint16_t val]` descriptors stored at DMEM `0x800004cc`–`0x800004dd` (reading byte-by-byte to avoid unaligned 32-bit loads at `+6`), which write:
       ```text
       CSR[0x02006e] = 0x0001    (ARC 0x900400dc = 1)
       CSR[0x070060] = 0x0000    (ARC 0x900e00c0 = 0)
       CSR[0x070011] = 0x0000    (ARC 0x900e0022 = 0)
       ```
   - **Step 3.2 (`CSR[0x02008b]`, `CSR[0x020072]`, `CSR[0x020073]`)**:
     - Read **`Pstate` (`byte[0x80000004]`)** and write `CSR[0x02008b]` (`ARC 0x90040116`) = `(byte[0x80000004] & 0x80) ? 0 : (byte[0x80000004] & 1)` (`0` when `Pstate == 0x30`).
     - Write `CSR[0x020072]` (`ARC 0x900400e4`) = `CsPresentChA` (`byte[0x80000025]`, `0x0003` on dual-rank).
     - Write `CSR[0x020073]` (`ARC 0x900400e6`) = `CsPresentChB` (`byte[0x80000040]`, `0x0003` on dual-rank).
   - **Step 3.3 (`CSR[0x020011]` mask and `CSR[0x01<r>0a3]` save/mask)**:
     - If `Quickboot == 0` (`byte[0x80000019] == 0`):
       1. Save `CSR[0x020011]` (`0x006f`) to DMEM `0x8000042a` and write `CSR[0x020011] &= 0xff9f` (`0x000f`).
       2. Save each populated rank's `CSR[0x01<r>0a3]` (`ARC 0x90020146 + r * 0x2000`, initialized to `0x0b33` by the host pre-training table) into DMEM **`0x80000430 + 2 * r`** (`0x80000430`–`0x80000437` across `r = 0..3`).
       3. Immediately after saving, loop across `r = 0 .. word[0x8000040c] - 1` (`4` ranks) and mask `CSR[0x01<r>0a3] &= 0x000c` (keeping only bits `[3:2]`; the saved `0x0b33` values are restored from `0x80000430 + 2 * r` in Phase 6 `Step 6.3`).
   - **Step 3.4 (Phase 4 — DQ block reset sequence on `CSR[0x020303]`..`0x020320`)**:
     1. Read and save `u_303 = CSR[0x020303]` (`ARC 0x90040606`);
     2. Write `CSR[0x020310] = 1` (`ARC 0x90040620`);
     3. Write `CSR[0x020303] = (u_303 & 0x00ef) | 0x0010` (set bit 4);
     4. Write `CSR[0x020310] = 0`;
     5. Write `CSR[0x02030d] = 1` (`ARC 0x9004061a`), then `CSR[0x02030d] = 0`;
     6. Write `CSR[0x020311] = 1` (`ARC 0x90040622`);
     7. Call **`delay_ps_or_clks(130000000, 0)`** (`130,000,000 ps = 130.0 µs`, i.e. `104,001` DFI clocks = `52,001` RTC ticks at 6400/4);
     8. Write `CSR[0x020311] = 0`;
     9. Write `CSR[0x020320] = 0` (`ARC 0x90040640`), then `CSR[0x020320] = 1`, then `CSR[0x020320] = 0`;
     10. Restore `CSR[0x020303] = u_303 & 0x00ff` (clear bit 4).
   - **Step 3.5 (Phase 5 — DRAM reset and PHY PLL/reset sequencing)**:
     - **Register roles in `Step 3.5`**: `CSR[0x0200a2]` (`ARC 0x90040144`) bits `[1:0]` select the active PHY clock mux (`0` = clock gated/off during reset transitions, `1` = reference/bypass clock, `2` = PHY PLL clock), and `CSR[0x020078]` (`ARC 0x900400f0`) controls the PHY PLL reset state (`1` = assert PLL reset while clocks are gated, `2` = release PLL reset and allow lock during the `400 µs` stabilization wait, `0` = normal operating state). The PMU firmware **never** polls `CSR[0x0200c9]` (`0` references in the PMU firmware) or any other PLL lock status register; it relies exclusively on the timed wait in step 12 below.
     1. Read `u_0f = CSR[0x01000f + pstate]` (`ARC 0x9002001e`) and write `(u_0f & 0xff78) | 0x0006` to broadcast `CSR[0x01f00f + pstate]` (`ARC 0x9003e01e`).
     2. Write `CSR[0x020061] = 1` (`ARC 0x900400c2`).
     3. Branch on `PllBypassEn` (`byte[0x80000005]`):
        - **Normal PLL mode (`PllBypassEn == 0`)**:
          1. `delay_dfi_clks(4)` (dummy read of `CSR[0x020024]`);
          2. Write `CSR[0x020049] = 0x01f0` (`ARC 0x90040092 = 496`);
          3. Write `CSR[0x020061] = 0` (`ARC 0x900400c2`);
          4. Write `CSR[0x0200e1] = 0` (`ARC 0x900401c2`);
          5. Write `CSR[0x020310] = 0` (`ARC 0x90040620`);
          6. Write `CSR[0x0200e0] = 0` (`ARC 0x900401c0`);
          7. Write `CSR[0x0200c3] = 0` (`ARC 0x90040186`);
          8. Write `CSR[0x0200e2] = 3` (`ARC 0x900401c4`), then call **`delay_ps_or_clks(1000000, 0)`** (`1.0 µs` = `401` RTC ticks);
          9. Write `CSR[0x0200e2] = 2`, then call **`delay_ps_or_clks(500000, 0)`** (`0.5 µs` = `201` RTC ticks);
          10. Write `CSR[0x0200e2] = 0`, then call **`delay_ps_or_clks(10000000, 0)`** (`10.0 µs` = `4,001` RTC ticks);
          11. Write `CSR[0x020078] = 1` (`ARC 0x900400f0`), `CSR[0x020200] = 0x0033` (`ARC 0x90040400`), `CSR[0x0200a2] = 0` (`ARC 0x90040144`), then `delay_dfi_clks(6)`;
          12. Write `CSR[0x020078] = 2`, `CSR[0x0200a2] = 2`, then call **`delay_ps_or_clks(0, 10000)`** (which scales `10,000 * 8 * 4 = 320,000` DFI clocks = `160,000` RTC ticks = **`400.0 µs`** when `byte[0x80000a80] == 1`);
          13. Write `CSR[0x020078] = 0`, `CSR[0x0200a2] = 0`, `CSR[0x020200] = 0`, then call **`delay_ps_or_clks(0, 0x20)`** (`1,024` DFI clocks = `512` RTC ticks = **`1.28 µs`**);
          14. Write `CSR[0x0200f8] = 0` (`ARC 0x900401f0`) and `CSR[0x0200a2] = 2` (`ARC 0x90040144`).
        - **PLL bypass mode (`PllBypassEn != 0`)**:
          1. `delay_dfi_clks(4)`, then write `CSR[0x020061] = 0`, `CSR[0x0200e1] = 1`, `CSR[0x0200e0] = 1`, `CSR[0x020310] = 0`, `CSR[0x0200c3] = 0`, and call `delay_ps_or_clks(10000000, 0)` (`10.0 µs`);
          2. Write `CSR[0x020078] = 1`, `CSR[0x020200] = 0x0033`, `CSR[0x0200a2] = 0`, then `delay_ps_or_clks(0, 0x20)`;
          3. Write `CSR[0x020078] = 2`, `CSR[0x0200a2] = 2`, then `delay_ps_or_clks(0, 1024)`;
          4. Execute steps 13 and 14 of the normal branch (`0x020078 = 0`, `0x0200a2 = 0`, `0x020200 = 0`, `delay_ps_or_clks(0, 0x20)`, `0x0200f8 = 0`, `0x0200a2 = 2`).
   - **Step 3.6 (Per-rank DQ calibration loop when `DRAMFreq >= 1600`)**:
     - If `DRAMFreq < 1600` (`halfword[0x80000006] < 0x0640`), skip directly to Step 3.7.
     - **Important — Per-rank only (`r = 0..3`), NO lane loop (`l = 0..8`)**: `Step 3.6` iterates **exclusively over the active ranks `r`** (`byte[0x80000b6e]..byte[0x80000b6f]` = `0..1` on Channel A, then `byte[0x80000b70]..byte[0x80000b71]` = `2..3` on Channel B) and **never loops over lanes `l = 0..8`**. The step-capture registers `CSR[0x01<r>0d0..d5]` (`ARC 0x900201a0..0x900201aa + r * 0x2000`), the latch strobe `CSR[0x01<r>0ac]` (`ARC 0x90020158 + r * 0x2000`), the step-selection result register `CSR[0x01<r>0ad]` (`ARC 0x9002015a + r * 0x2000`), and the companion calibration copy `CSR[0x01<r>0d6]` → `CSR[0x01<r>0af]` (`ARC 0x900201ac` → `0x9002015e + r * 0x2000`) are **per-rank registers located only at sub-block `0` of each rank `r`**. If a custom implementation mistakenly adds an inner lane loop `l = 0..8` at `+ l * 0x200`, `l = 0` reads `d[0..4] = {0, 0, 0x41, 0x85, 0xca}` and selects `k = 2`, and then `l = 1..8` read unimplemented zeros (`d[0..4] = {0, 0, 0, 0, 0}`) and overwrite `CSR[0x01<r>0ad]` with the fallback `k = 5`.
     - **Three-stage `Step 3.6` register sequence**:
       1. **Broadcast setup before the rank loop**:
          - Select the rank-calibration gate register and bit mask from `byte[0x80000403]` (set to `8`–`10` from `CSR[0x0c00ee]` at startup): when `byte[0x80000403] >= 4`, the gate register offset is **`0x0057 + pstate`** (`CSR[0x01<r>057 + pstate]` at `ARC 0x900200ae + r * 0x2000` / broadcast `CSR[0x01f057 + pstate]` at `ARC 0x9003e0ae`) and `bit_mask = 1` (bit 0) (for `< 2`, offset `0x0014 + pstate` with `bit_mask = 2`; for `2..3`, offset `0x0056` with `bit_mask = 2`).
          - Write `CSR[0x01f0d9] = 0x1000` (`ARC 0x9003e1b2`).
          - Read `CSR[0x01000f + pstate]` (`ARC 0x9002001e`) and write `(CSR[0x01000f + pstate] & 0xfff8) | 0x0006` to broadcast `CSR[0x01f00f + pstate]` (`ARC 0x9003e01e`).
          - Read `CSR[0x010057 + pstate]` (`ARC 0x900200ae`) and write `CSR[0x010057 + pstate] & ~0x0001` to broadcast **`CSR[0x01f057 + pstate]` (`ARC 0x9003e0ae`)**.
          - Read `CSR[0x0100ac]` (`ARC 0x90020158`) and write `CSR[0x0100ac] & ~0x0001` to broadcast `CSR[0x01f0ac]` (`ARC 0x9003e158`).
          - Write `CSR[0x01f00f + pstate] = CSR[0x01000f + pstate] | 0x0005`, wait **`delay_dfi_clks(0x3000)`** (`12,288` DFI clocks = `6,144` RTC ticks = **`15.36 µs`**), and write `CSR[0x01f00f + pstate] = (CSR[0x01000f + pstate] & ~0x0005) | 0x0004`.
       2. **Per-rank calibration loop (`r = 0, 1` on Channel A, then `r = 2, 3` on Channel B — executed once per rank)**:
          - Read the five rank-level step capture registers **`d[0..4] = CSR[0x01<r>0d0..d4]`** (`ARC 0x900201a0..0x900201a8 + r * 0x2000`), with sentinel `d[5] = 0x03ff`.
          - Pulse **`CSR[0x01<r>0ac]`** (`ARC 0x90020158 + r * 0x2000`) bit 0 (`write old | 1` → `write old & ~1`).
          - **After** pulsing `0x01<r>0ac`, read the rank-level calibration reference **`d5 = CSR[0x01<r>0d5]`** (`ARC 0x900201aa + r * 0x2000`). Reading `d5` after the `0x01<r>0ac` pulse is why `CSR[0x0100d5]` transitions from its pre-pulse value (`0x019a`–`0x01bf` = `410`–`447`) to **`0x0041`–`0x0043` (`65`–`67`)** during `Step 3.6`, which `Step 3.7` then copies into `CSR[0x020187]`.
          - Compute the **14% acceptance tolerance** `tol = (d5 * 14) / 100` (`9` when `d5 = 0x0041..0x0043`). Across `k = 0..4` (with `d[5] = 0x03ff`), evaluate the adjacent step delta `delta_k = d[k + 1] - d[k]`: select the first `k` where `2 * d[k] >= d5` (non-zero step floor) and `|delta_k - d5| <= tol`. If no step satisfies both conditions, use the fallback value **`k = 5`**.
          - *(Worked example on the reference board: for `d[0..4] = {0x0000, 0x0000, 0x0041, 0x0085, 0x00ca}` and `d5 = 0x0041..0x0043` (`65..67`, `tol = 9`), `k = 0` and `k = 1` fail `2 * d[k] >= d5` (`d[0] = d[1] = 0`), and **`k = 2`** (`2 * 0x0041 = 130 >= 67`, `delta_2 = 0x0085 - 0x0041 = 68`, `|68 - d5| <= 9`) is selected on all four ranks `r = 0..3`.)*
          - Write the selected code `k` (`2` on the reference board, or `5` on fallback) to **`CSR[0x01<r>0ad]`** (`ARC 0x9002015a + r * 0x2000`, serialized in `phypara` Segment 0 as `CSR[0x0100ad] = 0x0002`).
          - Latch the per-rank calibration result and copy `CSR[0x01<r>0d6]` to `CSR[0x01<r>0af]`:
            1. Write `CSR[0x01<r>057 + pstate] |= 0x0001` (`ARC 0x900200ae + r * 0x2000`);
            2. Write `CSR[0x01<r>00f + pstate] = (CSR[0x01<r>00f + pstate] & ~0x0003) | 0x0001` (`ARC 0x9002001e + r * 0x2000`);
            3. Wait `delay_dfi_clks(((CSR[0x01<r>014 + pstate] << 1) & 0x01fe) * 160 + 160)`:
               - **Important — `pstate` is the P-state index `0` (`word[0x8000041c] = 0`), NOT the `Pstate` byte `0x30`**: The register read here is `CSR[0x01<r>014]` (`ARC 0x90020028 + r * 0x2000`), which the host pre-training table (`training_table_49`) initializes to **`0x3232`** across all four ranks `r = 0..3` on both controllers (adding `0x30` would erroneously read `CSR[0x01<r>044] = 0x0f0f`).
               - With `CSR[0x01<r>014] = 0x3232`, `((0x3232 << 1) & 0x01fe) = 0x0064 = 100`, giving **`100 * 160 + 160 = 16,160` DFI clocks** (`8,080` RTC ticks = `20.2 µs` when `shift = 1` at `400 MHz` RTC, or `10.1 µs` at `1600 MHz` DFI equivalent);
            4. Write `CSR[0x01<r>057 + pstate] &= ~0x0001` and `CSR[0x01<r>00f + pstate] &= ~0x0001`;
            5. Pulse `CSR[0x01<r>0ac]` bit 0 (`write old | 1` → `write old & ~1`);
            6. Copy **`CSR[0x01<r>0d6]`** (`ARC 0x900201ac + r * 0x2000`) to **`CSR[0x01<r>0af]`** (`ARC 0x9002015e + r * 0x2000`, serialized in `phypara` Segments 14–17: `0x0039`–`0x003b` on `DDR0` and `0x0038`–`0x003b` on `DDR1` in the SPI envelope, or `0x003a`–`0x0040` in USB/FDL runs).
       3. **Broadcast restore after the rank loop and `CSR[0x01<r>00f]` lifecycle (`0x1086 -> 0x1006 -> 0x1004 -> 0x1089`)**:
          - Write `CSR[0x01f057 + pstate] = CSR[0x010057 + pstate] | 0x0001` (`ARC 0x9003e0ae`);
          - Write `CSR[0x01f00f + pstate] = (CSR[0x01000f + pstate] & ~0x0003) | 0x0001` (`ARC 0x9003e01e`);
          - Wait `delay_dfi_clks(((CSR[0x010014 + pstate] << 1) & 0x01fe) * 160 + 160)` (**`16,160` DFI clocks** for `CSR[0x010014] = 0x3232`);
          - Write `CSR[0x01f057 + pstate] = CSR[0x010057 + pstate] & ~0x0001`;
          - Write `CSR[0x01f00f + pstate] = CSR[0x01000f + pstate] & ~0x0001`.
          - **Lifecycle of `CSR[0x01<r>00f]` (`0x1086 -> 0x1006 -> 0x1004 -> 0x1089`) and `CSR[0x01<r>057]`**:
            - The host pre-training table initializes `CSR[0x01<r>00f]` (`r = 0..3`) to **`0x1086`** at 6400 MT/s (`0x0c86` at 4800 MT/s).
            - `Step 3.5` masks `u_0f & 0xff78 | 0x0006` (which intentionally clears bits `[2:0]` and bit 7 = `0x0080`), turning `0x1086` into **`0x1006`**. `Step 3.6` then modifies `[1:0]` and leaves bit 1 cleared, leaving `CSR[0x01<r>00f]` at **`0x1004`** (`bits [2:0] = 0b100`, bit 7 clear) when the PMU finishes `DevInit`. No PMU step re-sets bit 7.
            - Then, **after the PMU halts and before `phypara` is serialized**, the **host post-training table** explicitly writes **`CSR[0x01<r>057] = 0x0001`** and **`CSR[0x01<r>00f] = 0x1089`** across all four ranks `r = 0..3` (`0x0c89` at 4800 MT/s). That host post-training write is why `phypara` records `0x1089`.
          - *(Note: the per-rank channel-bit clear on `CSR[0x01f002 + rank]` at `ARC 0x9003e004 + 2 * rank`—clearing bit 0 for Channel A ranks `0..1` and bit 1 for Channel B ranks `2..3`—and its matching restore `CSR[0x01f002 + rank] = CSR[0x010002 + rank] | chan_mask` are performed by the rank-mux wrapper used around per-rank command/training sweeps.)*
   - **Step 3.7 (Master clock-period calibration)**:
     - Execute the **master clock-period calibration** sequence detailed above (3-step `0x0200e5`/`0x0200e3` strobe with bit 10 falling edge → `CSR[0x02000a]`, `CSR[0x020017]`, `CSR[0x020012] = 0x7353`, `CSR[0x020186] = 0x0073`, `CSR[0x020187] = 0x0041`, and per-rank `CSR[0x01<r>0d9] = 0x009c`).
4. **Phase 6 — ACSM clock divider init, one-shot NOP flush, `MemResetL` release (`CSR[0x020060]`), and initial LPDDR5 MRW + ZQ `MPC` bursts**:
   - **Step 6.1 (ACSM clock divider setup from read-only `CSR[0x020314]`/`0x020315` and one-shot NOP flush)**:
     - **Read-only hardware calibration inputs `CSR[0x020314]` and `CSR[0x020315]`**: Neither the host pre-training tables nor the PMU firmware ever writes to `CSR[0x020314]` (`ARC 0x90040628`) or `CSR[0x020315]` (`ARC 0x9004062a`) (`0` writes in the PMU firmware and `0` entries in the host tables). They are **read-only PHY hardware calibration output registers** in the `0x020300`–`0x020320` block: before `Step 3.4` and `Step 3.5` run, they read `0x0000`, and once the DQ block reset (`Step 3.4`) and PHY PLL lock (`Step 3.5`) complete, the hardware automatically populates them (`0x0036`–`0x003b` and `0x007b`–`0x0083` at 6400 MT/s).
     - When `byte[0x80000403] >= 8` (set from `CSR[0x0c00ee]` at startup), read `CSR[0x020314]` and `CSR[0x020315]` and initialize the ACSM clock/control registers: write `CSR[0x090808] = (CSR[0x020314] << 8) | 1` (`0x3601`–`0x3b01` at `ARC 0x90121010`), write `CSR[0x090813] = (CSR[0x020315] << 8) | 1` (`0x7b01`–`0x8301` at `ARC 0x90121026`), and mask `CSR[0x09080b] &= 0xff9f` and `CSR[0x09080c] &= 0xff9f` (`ARC 0x90121016`/`0x90121018` from `0x002f`/`0x006f` to `0x000f`/`0x000f`).
     - Reset the ACSM instruction pointers (`CSR[0x020022 + pstate] = 0` at `ARC 0x90040044`, `CSR[0x020023 + pstate] = 0` at `ARC 0x90040046`, DMEM `0x80000458 = 0`, `0x8000045e = 0`, `byte[0x8000046a] = 0`), queue a 4-cycle NOP instruction in ACSM SRAM (`0x41000`), and execute it in **one-shot mode** (`CSR[0x020031]` remains `0`): write `CSR[0x020027] = 0`, `delay_dfi_clks(6)`, `CSR[0x020023 + pstate] = (halfword[0x80000458] >> 2) - 1`, `CSR[0x020020] = 1` → **poll `while (CSR[0x020021] == 0)` (exiting as soon as `CSR[0x020021] != 0`, since `1` = `ACSM_DONE`)** → `CSR[0x020020] = 0` (see the ACSM sequencer subsection below).
     - Wait `delay_dfi_clks(0x1f)` then `delay_dfi_clks(4)`, write `CSR[0x0200f8] = 0` (`ARC 0x900401f0`), and write `CSR[0x0200a2] = (CSR[0x0200a2] & 0xfffc) | 0x0002` (`ARC 0x90040144`).
     - Write `CSR[0x02002a] = 1` (`ARC 0x90040054`).
   - **Step 6.2 (`MemResetL` release, LPDDR5 mode-register initialization burst, and ZQ calibration `MPC` burst)**:
     - Assemble the per-rank mode-register table in DMEM from the **message block** (`0x8000005a`–`0x800000cd`: `MR1`..`MR41` for `A0`, `A1`, `B0`, `B1`).
     - When `Quickboot == 0` (`byte[0x80000019] == 0`) and `MsgMisc[7] == 0` (`(byte[0x80000001] & 0x80) == 0`), execute the DRAM reset-release and mode-register initialization flow:
       1. Write `CSR[0x020041 + pstate] = 0` (`ARC 0x90040082`) and `CSR[0x02002a] = 1` (`ARC 0x90040054`);
       2. Wait for `tINIT3`: test bit 1 of **`MsgMisc` (`byte[0x80000001] & 2`)**, calling **`delay_ps_or_clks(2000000, 0)`** (`2.0 µs` = `801` RTC ticks) if set, or **`delay_ps_or_clks(200000000, 0)`** (`200.0 µs` = `80,001` RTC ticks) if clear. Because `MsgMisc = 0x01` on the reference profile (`0x01 & 2 == 0`), the fast-`tINIT` branch is **not** taken and the reference run waits the full **`200.0 µs`**;
       3. Release `MemResetL`: write **`CSR[0x020060] = 3`** (`ARC 0x900400c0`), wait `tINIT4`/`tINIT5` via **`delay_ps_or_clks(2000000, 0)`** (`2.0 µs` if `(byte[0x80000001] & 2) != 0`) or **`delay_ps_or_clks(2000000000, 0)`** (`2,000,000,000 ps` = **`2.0 ms`** = `800,001` RTC ticks when `(byte[0x80000001] & 2) == 0`, as on the reference profile), and then test **`Misc[0]` (`byte[0x8000000d] & 1`)** in the message block: only if `Misc[0] == 1` does the PMU write `CSR[0x020060] = 1`. **Because `Misc = 0x00` (`Misc[0] == 0`) on the reference profile, `CSR[0x020060] = 1` is never executed by the PMU and no later stage touches `CSR[0x020060]`, so the final value captured in `phypara` (both SPI and FDL) is `0x0003` (`pre = 0x0002`, `trained = 0x0003`)**;
       4. Prepare the ACSM routing registers in exact order (`CSR[0x03f084] = 0` at `ARC 0x9007e108`, `CSR[0x01f00a] = 0` at `ARC 0x9003e014`, `CSR[0x02002a] = 1` at `ARC 0x90040054`, `CSR[0x020041 + pstate] = 1` at `ARC 0x90040082`), reset the ACSM cursor (`0x80000458 = 0`), queue the entire initial LPDDR5 mode-register write burst (`MR19`, `MR18`, `MR1`, `MR2`, `MR3`, `MR10`, `MR11`, `MR17`, `MR12`, `MR14`, `MR15`, `MR16`, `MR20`, `MR22`, `MR24`, `MR37`, `MR40`, `MR41`, each followed by a `2`-unit `0x2b00` NOP delay) into ACSM SRAM (`0x41000` / `ARC 0x90082000`), and trigger the **ACSM command sequencer once in one-shot mode** (`CSR[0x020027] = 0`, `delay_dfi_clks(6)`, `CSR[0x020023 + pstate] = (cursor >> 2) - 1`, `CSR[0x020020] = 1` → poll `while (CSR[0x020021] == 0)` → `CSR[0x020020] = 0`);
       5. Immediately after the MRW burst, reset the ACSM cursor (`0x80000458 = 0`), queue the **LPDDR5 ZQ calibration (`MPC`) burst** (on the reference board where `byte[0x80000a80] == 1` and `(byte[0x80000001] & 8) == 0`, this is the 48-halfword Variant 1 sequence: a `5`-unit NOP, `cmd = 0x25` + `cmd = 0x26` with `OP = 0x86` and `cs = 0x0f` / `0`, two `16`-unit NOPs, a `1`-unit NOP, and a `10`-unit NOP; see exact halfwords below), execute it via a one-shot ACSM run, restore `CSR[0x020041 + pstate] = 0` and `CSR[0x01f00a] = 0`, and pulse the channel-commit strobe `CSR[0x02f0a0]` (`1 -> delay_dfi_clks(0x20) -> 0`).
   - **Step 6.3 (Post-`DevInit` restore of `CSR[0x020011]` and `CSR[0x01<r>0a3]`, lane state, and `CSR[0x0000a4]`)**:
     - Program the active channel/rank masks (`CSR[0x02007a]` and `CSR[0x02007e]`), `CSR[0x020042] = byte[0x80000b68] | 0x0100` (`ARC 0x90040084 = 0x0108`), `CSR[0x020044] = (DfiFreqRatio == 2) ? 3 : 2` (`ARC 0x90040088 = 2` during `DevInit`, later updated to `byte[0x80000b68] | 0x0100 = 0x0108` during post-`DevInit` stage setup), pulse `CSR[0x02f0a0]` (`1 -> delay_dfi_clks(0x20) -> 0`), write `CSR[0x02006e] = 0` (`ARC 0x900400dc = 0`), and when `Quickboot == 0` (`byte[0x80000019] == 0`):
       - **Restore `CSR[0x020011 + pstate]` (`ARC 0x90040022`) from DMEM `halfword[0x8000042a]` (`0x006f`)** and **restore `CSR[0x01<r>0a3]` (`0x0b33` / `0x0b3f`) from DMEM `0x80000430`–`0x80000437`**, write `CSR[0x020072] = CsPresentChA` (`0x0003`) and `CSR[0x020073] = CsPresentChB` (`0x0003`), and refresh `CSR[0x090808] = (CSR[0x020314] << 8) | 1` and `CSR[0x090813] = (CSR[0x020315] << 8) | 1`. *(Because this `Step 6.3` restore writes `0x006f` from `0x8000042a` back to `CSR[0x020011]`, every clean full-training run staged via USB/FDL (`Quickboot == 0`) finishes with `CSR[0x020011] = 0x006f`; the factory SPI envelope's `0x000f` reflects an earlier/quickboot snapshot where this restore was skipped — see [14-validation-and-current-status.md](14-validation-and-current-status.md).)*
       - Save and restore `CSR[0x01<r><l>50]` and `CSR[0x01<r><l>51]` via the per-lane context structure (`+0xc4`/`+0xc6`); because their initial value comes from the host pre-training table (`0x0035`), they remain `0x0035` after training when `DisableTrainingLoop[1] == 0`.
     - Write `CSR[0x0000a4] = 1` (`ARC 0x90000148`). *(Note: `CSR[0x0c0086]` at `ARC 0x9018010c` has `0` references—neither read nor written—in the full-training PMU firmware, though the quickboot PMU firmware restores `0x0c0086` directly from `phypara` in Pass 1. Because the host pre-training table writes `CSR[0x0c0086] = 0x0001` before starting full training, reading `0x0c0086` from the PMU at the end of `DevInit` simply returns `0x0001`; it cannot be used as a PMU-side `DevInit` self-check.)*

### ACSM command sequencer (`0x41000` / `ARC 0x90082000` and `0x020020`–`0x020031`)

All DRAM mode-register writes (`MRW`), multi-purpose commands (`MPC`), and inter-command NOP delays during `DevInit` and subsequent training stages are transmitted by loading 4-halfword instruction packets into the **Address/Command State Machine (ACSM) SRAM** at host CSR index `0x41000`–`0x41fff` (`ARC 0x90082000`–`0x90083ffe`, with complex training patterns staged through DMEM `0x800009a8`) and triggering the sequencer via group `0x02` control registers `CSR[0x020020]`–`0x020031` (`ARC 0x90040040`–`0x90040062`). **The PMU firmware never issues LPDDR5 Mode Register Reads (`MRR`)** and contains no `MRR` readback CSR path; only `MRW` and `MPC` bursts are used.

1. **ACSM sequencer control, `CSR[0x020021]` (`ACSM_DONE`) active-high polarity, and one-shot vs. continuous-loop (`CSR[0x020031]`) modes**:
   - `CSR[0x090808]` (`ARC 0x90121010`) and `CSR[0x090813]` (`ARC 0x90121026`): ACSM clock/timing dividers initialized in `Step 6.1` (when `byte[0x80000403] >= 8`) from the read-only hardware calibration registers `CSR[0x020314]` (`0x0036`–`0x003b`) and `CSR[0x020315]` (`0x007b`–`0x0083`) as `(val << 8) | 1`, yielding **`0x3b01`** and **`0x8301`** in the reference `phypara`. The adjacent registers `CSR[0x09080b]` (`ARC 0x90121016`, `0x90121010 + 6`) and `CSR[0x09080c]` (`ARC 0x90121018`, `0x90121010 + 8`) are masked with `& 0xff9f` (`0x002f`/`0x006f` → `0x000f`/`0x000f`, matching the `& 0xff9f` mask applied to `CSR[0x020011]`).
   - **`CSR[0x020021]` (`ARC 0x90040042`) — Active-high `ACSM_DONE` status (`1` = done/idle/ready, `0` = busy/running)**:
     - **`CSR[0x020021]` is an active-high completion/ready flag (`1` = done/idle, `0` = busy/running), NOT an active-high busy flag.**
     - Both the one-shot completion poll and the continuous-loop stop routine **loop while `CSR[0x020021] == 0` (`busy`) and exit with success (`return 1`) on the first read where `CSR[0x020021] != 0` (`1` = done)**.
     - Because a short one-shot ACSM sequence (such as the 4-cycle NOP in `Step 6.1`, which executes in ~5 ns) finishes before the ARC core executes its first `ldh` of `CSR[0x020021]`, `CSR[0x020021]` already reads **`1` (`DONE`)** on the very first poll iteration. Waiting for `CSR[0x020021] == 0` inverts the hardware polarity and causes a false 25,000-iteration timeout on an already-completed sequence.
   - **Two distinct ACSM execution modes**:
     1. **One-shot burst mode (used for all `DevInit` bursts in `Step 6.1` and `Step 6.2`)**:
        - Does **not** touch `CSR[0x020031]` (`CSR[0x020031]` remains `0`).
        - For MRW/MPC bursts in `Step 6.2`, first configure the channel/rank ACSM routing registers in exact order:
          ```text
          CSR[0x03f084]        = 0    (ARC 0x9007e108 = 0)
          CSR[0x01f00a]        = 0    (ARC 0x9003e014 = 0)
          CSR[0x02002a]        = 1    (ARC 0x90040054 = 1)
          CSR[0x020041+pstate] = 1    (ARC 0x90040082 = 1)
          ```
        - Reset the ACSM pointers (`CSR[0x020022 + pstate] = 0`, `CSR[0x020023 + pstate] = 0`, `halfword[0x80000458] = 0`, `halfword[0x8000045e] = 0`, `byte[0x8000046a] = 0`) and write the 8-halfword instruction pairs to `0x90082000 + 2 * cursor`.
        - Trigger the one-shot burst and poll for completion:
          1. Write `CSR[0x020027] = 0` (`ARC 0x9004004e = 0`);
          2. Call `delay_dfi_clks(6)`;
          3. Write `CSR[0x020023 + pstate] = (halfword[0x80000458] >> 2) - 1` (`ARC 0x90040046`);
          4. Write `CSR[0x020020] = 1` (`ARC 0x90040040 = 1`);
          5. **Poll `CSR[0x020021]` (`ARC 0x90040042`) while `CSR[0x020021] == 0` (up to `25,000` iterations), succeeding as soon as `CSR[0x020021] != 0` (`1` = done)**;
          6. Write `CSR[0x020020] = 0` (`ARC 0x90040040 = 0`).
     2. **Continuous background loop mode (`CSR[0x020031] = 1`, used only in later read/write training stages, never in `DevInit`)**:
        - Start: write `CSR[0x020022 + pstate] = 300` (`0x012c`), `CSR[0x020023 + pstate] = halfword[0x8000047e]`, **`CSR[0x020031] = 1`** (`ARC 0x90040062 = 1`, continuous loop enable), `CSR[0x020027] = 0`, `delay_dfi_clks(6)`, and `CSR[0x020020] = 1` (without polling `CSR[0x020021]`, since the pattern loops continuously in the background while `CSR[0x020031] == 1`), then wait `delay_ps_or_clks(50000, 0x0f)`.
        - Stop: write **`CSR[0x020031] = 0`** (disable continuous loop), **poll `CSR[0x020021]` while `CSR[0x020021] == 0` (until `CSR[0x020021] == 1` = done)**, write `CSR[0x020020] = 0`, and wait `delay_ps_or_clks(5000, 0)`.
   - DMEM `0x80000458` (`16-bit`) and `0x8000045e` (`16-bit`): Current ACSM SRAM halfword write cursor (incremented by `8` halfwords per command/delay pair; reset to `0` at the start of each burst).

2. **ACSM 4-halfword instruction format (`hw[0..3]` at `0x90082000 + 2 * cursor`) and single-burst queuing**:
   - Each ACSM instruction occupies **4 consecutive 16-bit halfwords** (`hw[0..3]`), and the helper routines emit instructions in **8-halfword pairs** (`2 instructions = 16 bytes`).
   - **Single-trigger burst execution**: During `Step 6.2`, the firmware does **not** trigger the ACSM once per `MRW`. Instead, it resets `cursor = 0` (`0x80000458 = 0`), writes the initial 8-halfword NOP slot (`hw[0..7] = 0`), appends all `MRW-1`/`MRW-2` pairs and their trailing `0x2b00` NOP entries sequentially into `0x90082000 + 2 * cursor`, sets `CSR[0x020023] = (cursor >> 2) - 1`, and triggers `CSR[0x020020] = 1` **once for the entire burst**.
   - **Source of mode-register values and chip-select mask (`cs_mask`)**:
     - Every mode-register value in the `Step 6.2` burst is read from the **message block** (`0x8000005a`–`0x800000cd`)—none are hardcoded in the firmware.
     - Channel-common mode registers (`MR19`, `MR18`, `MR1`, `MR2`, `MR3`, `MR10`, `MR11`, `MR16`, `MR20`, `MR22`, `MR37`, `MR40`, `MR41`) are emitted once using **`cs_mask = 3`** (broadcast to both ranks `0` and `1` on the channel, reading the `A0` byte from the message block).
     - Per-rank mode registers (`MR17`, `MR12`, `MR14`, `MR15`, `MR24`, `MR30`) are emitted individually per populated rank `r` using **`cs_mask = 1 << r`** (`cs_mask = 1` for rank 0 from `MR*_A0`, then `cs_mask = 2` for rank 1 from `MR*_A1`).
   - **LPDDR5 CA command entry (`MRW-1` + `MRW-2`)**:
     - An LPDDR5 mode-register write (`MRW`) consists of two back-to-back 4-halfword instructions (`Entry 2k` for `MRW-1` and `Entry 2k+1` for `MRW-2`), where `hw[1] = hw[2] = hw[3] = 0x0000` and `hw[0]` packs the two 7-bit LPDDR5 CA phases (`CA[6:0]` on rising/falling CK edges) plus a 2-bit chip-select mask:
       - **Bits `[6:0]` (rising-edge CA0..CA6)**: `0x58` (`0b1011000`) for `MRW-1`; `0x08 | ((OP & 0x80) >> 1)` (`0x08` when `OP[7] == 0`, or `0x48` when `OP[7] == 1`) for `MRW-2`;
       - **Bits `[13:7]` (falling-edge CA0..CA6)**: `MA[6:0]` (7-bit mode-register address) for `MRW-1`; `OP[6:0]` (low 7 bits of mode-register data) for `MRW-2`;
       - **Bits `[15:14]` (chip-select mask)**: `0b01` (`0x4000`) = rank 0 only, `0b10` (`0x8000`) = rank 1 only, `0b11` (`0xc000`) = broadcast to both ranks.
       ```text
       hw0_mrw1(cs_mask, MA) = ((cs_mask & 3) << 14) | ((MA & 0x7f) << 7) | 0x0058
       hw0_mrw2(cs_mask, OP) = ((cs_mask & 3) << 14) | ((OP & 0x7f) << 7) | ((OP & 0x80) >> 1) | 0x0008
       ```
   - **8-halfword NOP / cycle-delay pair (`hw[0..7]`, 8-bit repeat count split across `hw[3]` and `hw[7]`)**:
     - Every NOP delay entry emitted into ACSM SRAM occupies an **8-halfword pair (`hw[0..7]`, 16 bytes)** and encodes an 8-bit repeat count `r = ceil(cycles / divisor) - 1` (`0..255`), where `divisor = 1` when `raw_mode == 1` (used for the explicit `4`-, `5`-, `10`-, `16`-, and `1`-unit NOPs in `Step 6.1` and `Step 6.2.5`) or `divisor = 2 * halfword[0x8000045a] = 8` when `raw_mode == 0` (used for the `cycles = 17` inter-MRW spacing in `Step 6.2.4`, where `ceil(17 / 8) - 1 = 2`).
     - When `r == 0` (`1` unit total): all 8 halfwords `hw[0..7]` are `0x0000`.
     - When `1 <= r <= 255`: the **low 3 bits `r[2:0]`** are placed in **`hw[3]` bits `[14:12]`** with opcode `0x0b00`, and the **upper 5 bits `r[7:3]`** are placed in **`hw[7]` bits `[14:10]`** (`(r & 0xf8) << 7`):
       ```text
       hw[0..2] = 0x0000
       hw[3]    = ((r & 0x07) << 12) | 0x0b00
       hw[4..6] = 0x0000
       hw[7]    = (r & 0xf8) << 7
       ```
       *(If `r >= 256`, the helper emits maximal 255-unit pairs `hw[3] = 0x7b00, hw[7] = 0x7c00` in a loop, subtracting `255` each iteration.)*
     - Concrete 8-halfword NOP pairs emitted during `Step 6.1` and `Step 6.2`:

       | NOP call site | `cycles` | `raw_mode` (`divisor`) | Repeat count `r` | `hw[3]` | `hw[7]` | Complete 8-halfword pair (`hw[0..7]`) |
       |---|---:|---:|---:|---:|---:|---|
       | `1`-unit NOP (`Step 6.2.5`) | `1` | `1` (`1`) | `0` (`0x00`) | `0x0000` | `0x0000` | `0000 0000 0000 0000 0000 0000 0000 0000` |
       | Inter-MRW spacing (`Step 6.2.4`) | `17` | `0` (`8`) | `2` (`0x02`) | `0x2b00` | `0x0000` | `0000 0000 0000 2b00 0000 0000 0000 0000` |
       | `4`-unit NOP (`Step 6.1` flush) | `4` | `1` (`1`) | `3` (`0x03`) | `0x3b00` | `0x0000` | `0000 0000 0000 3b00 0000 0000 0000 0000` |
       | `5`-unit NOP (`Step 6.2.5` pre-MPC) | `5` | `1` (`1`) | `4` (`0x04`) | `0x4b00` | `0x0000` | `0000 0000 0000 4b00 0000 0000 0000 0000` |
       | `10`-unit NOP (`Step 6.2.5` post-MPC) | `10` | `1` (`1`) | `9` (`0x09`) | `0x1b00` | `0x0400` | `0000 0000 0000 1b00 0000 0000 0000 0400` |
       | `16`-unit NOP (`Step 6.2.5` post-MPC) | `16` | `1` (`1`) | `15` (`0x0f`) | `0x7b00` | `0x0400` | `0000 0000 0000 7b00 0000 0000 0000 0400` |

   - **LPDDR5 ZQ Calibration (`MPC`) burst encoding in `Step 6.2.5`**:
     - **Variant selection condition**: `Step 6.2.5` evaluates `mode = (byte[0x80000a80] != 0) && ((byte[0x80000001] & 8) == 0)`. Because `byte[0x80000a80] == 1` (initialized in Phase 2) and `MsgMisc = byte[0x80000001] = 0x01` (`MsgMisc[3] == 0`), **the reference board always executes Variant 1 (`mode = 1`)**.
     - **How `cs_mask = 0x0f` fits the 2-bit chip-select field**: The ACSM command encoder masks `cs_mask` with **`cs_mask & 3`** when packing bits `[15:14]` of `hw[0]` (`0x0f & 3 = 3` → `0xc000`), and tests bit 1 of `cs_mask` (`(cs_mask & 2) != 0`) to set bit 4 (`0x0010`) of `hw[3]`.
     - **Variant 1 (`mode = 1`, active on the reference board — 48 halfwords = `6 × 8` halfwords, `CSR[0x020023] = (48 >> 2) - 1 = 11`)**:
       - Emits a `5`-unit NOP, followed by an 8-halfword ZQ `MPC` command pair (`cmd = 0x25` with `OP = 0x86`, `cs_mask = 0x0f` in `hw[0..3]`, and `cmd = 0x26` with `OP = 0x86`, `cs_mask = 0` in `hw[4..7]`), followed by two `16`-unit NOPs, a `1`-unit NOP, and a `10`-unit NOP:
         ```text
         hw0_cmd25(cs_mask, OP) = ((cs_mask & 3) << 14) | ((((OP >> 1) & 0x40) | 0x30) << 7) | (((OP >> 1) & 0x40) | 0x30)
                                = 0xc000 | (0x70 << 7) | 0x0070 = 0xf870   (for cs_mask = 0x0f, OP = 0x86)
         hw3_cmd25(cs_mask)     = (cs_mask & 2) ? 0x0010 : 0x0000 = 0x0010
         hw4_cmd26(cs_mask, OP) = ((cs_mask & 3) << 14) | ((OP & 0x7f) << 7) | (OP & 0x7f)
                                = 0x0000 | (0x06 << 7) | 0x0006 = 0x0306   (for cs_mask = 0, OP = 0x86)
         ```
       - Complete 48-halfword ACSM SRAM image written at `0x41000` (`ARC 0x90082000`) for Variant 1:

         | Halfword offset (`0x41000 + off`) | Raw halfwords (`hw[0..7]`) | Decoded ZQ `MPC` burst instruction (`mode = 1`) |
         |---:|---|---|
         | `0..7` | `0000 0000 0000 4b00 0000 0000 0000 0000` | Pre-MPC `5`-unit NOP delay (`r = 4`) |
         | `8..15` | `f870 0000 0000 0010 0306 0000 0000 0000` | LPDDR5 `MPC(ZQCAL LATCH)` (`OP = 0x86`): `cmd = 0x25` (`cs = 0x0f & 3 = 3`, `hw[0] = 0xf870`, `hw[3] = 0x0010`) + `cmd = 0x26` (`cs = 0`, `hw[4] = 0x0306`) |
         | `16..23` | `0000 0000 0000 7b00 0000 0000 0000 0400` | Post-MPC `16`-unit NOP delay (`r = 15 = 0x0f`) |
         | `24..31` | `0000 0000 0000 7b00 0000 0000 0000 0400` | Post-MPC `16`-unit NOP delay (`r = 15 = 0x0f`) |
         | `32..39` | `0000 0000 0000 0000 0000 0000 0000 0000` | Post-MPC `1`-unit NOP delay (`r = 0`) |
         | `40..47` | `0000 0000 0000 1b00 0000 0000 0000 0400` | Post-MPC `10`-unit NOP delay (`r = 9 = 0x09`) |

     - **Variant 0 (`mode = 0`, used only when `byte[0x80000a80] == 0` or `MsgMisc[3] == 1`)**:
       - Emits a single 8-halfword packet (`cmd = 0x19` with `OP = 0x86`, `cs_mask = 3`):
         ```text
         hw[0] = ((cs_mask & 3) << 14) | ((OP & 0x7f) << 7) | ((OP & 0x80) >> 1) | 0x0030 = 0xc370
         hw[3] = (cs_mask & 2) ? 0x0010 : 0x0000 = 0x0010
         ```
         (`c370 0000 0000 0010 0000 0000 0000 0000`, `CSR[0x020023] = 1`), triggers a one-shot ACSM run, and waits `delay_ps_or_clks(30000, 0x1e)` (`1.20 µs`).
   - **Verification against the reference ACSM capture (`training_envelope.bin`)**:
     - Decoding the first non-zero halfwords of the reference 6400 MT/s ACSM snapshot reproduces the exact initial message-block mode-register values:

     | Halfword offset (`0x41000 + off`) | Raw halfwords (`hw[0..3]`) | Message-block source | Decoded ACSM instruction |
     |---:|---|---|---|
     | `0..7` | `0000 0000 0000 0000` × 2 | — | Initial 2-instruction NOP slot (`cursor = 0..7`) |
     | `8..11` | `c9d8 0000 0000 0000` | `MR19_A0` (`0x8a`) | `MRW-1`: `cs = 3` (both ranks), `MA = 0x13` (**`MR19`**) (`51672` decimal in reference sample) |
     | `12..15` | `c808 0000 0000 0000` | `MR19_A0 = 0x10` | `MRW-2`: `cs = 3`, **`OP = 0x10`** (`MR19 = 0x10`) |
     | `16..23` | `0000 0000 0000 2b00` + `0000...` | Fixed (`2` units) | Inter-command NOP delay (`2` units) |
     | `24..31` | `c958 ...` / `c688 ...` | `MR18_A0 = 0x0d` (`0x86`) | `cs = 3`, **`MR18 = 0x0d`** (`MA = 0x12`, `OP = 0x0d`) |
     | `40..47` | `c0d8 ...` / `d848 ...` | `MR1_A0 = 0xb0` (`0x5a`) | `cs = 3`, **`MR1 = 0xb0`** (`MA = 0x01`, `OP = 0xb0`) |
     | `56..63` | `c158 ...` / `ddc8 ...` | `MR2_A0 = 0xbb` (`0x5e`) | `cs = 3`, **`MR2 = 0xbb`** (`MA = 0x02`, `OP = 0xbb`) |
     | `72..79` | `c1d8 ...` / `c088 ...` | `MR3_A0 = 0x01` (`0x62`) | `cs = 3`, **`MR3 = 0x01`** (`MA = 0x03`, `OP = 0x01`) |
     | `88..95` | `c558 ...` / `ec08 ...` | `MR10_A0 = 0x58` (`0x66`) | `cs = 3`, **`MR10 = 0x58`** (`MA = 0x0a`, `OP = 0x58`) |
     | `104..111` | `c5d8 ...` / `f188 ...` | `MR11_A0 = 0x63` (`0x6a`) | `cs = 3`, **`MR11 = 0x63`** (`MA = 0x0b`, `OP = 0x63`) |
     | `120..127` | `48d8 ...` / `5748 ...` | `MR17_A0 = 0xae` (`0x82`) | `cs = 1` (rank 0), **`MR17 = 0xae`** (`MA = 0x11`, `OP = 0xae`) |
     | `136..143` | `88d8 ...` / `9748 ...` | `MR17_A1 = 0xae` (`0x83`) | `cs = 2` (rank 1), **`MR17 = 0xae`** (`MA = 0x11`, `OP = 0xae`) |

#### Summary of `DevInit`-written CSRs outside `phypara`

The following CSRs are written by `DevInit` to sequence hardware state machines, resets, and the ACSM sequencer, and are **not** part of the 5,170-entry `phypara` table:

- **Group `0x00`**: `0x0000a4 = 1` (`ARC 0x90000148`; never written `0`);
- **Group `0x01` broadcast (`0x01fxxx`)**: `0x01f002`–`0x01f005` (`0x9003e004`–`0x9003e00a`, per-rank channel-bit clear/restore), `0x01f00a` (`0x9003e014`), `0x01f00f` (`0x9003e01e`), `0x01f01a` (`0x9003e034`), `0x01f057` (`0x9003e0ae`, `Step 3.6` rank-calibration gate bit 0), `0x01f0ac` (`0x9003e158`), and `0x01f0d9` (`0x9003e1b2`). *(Note: the `Step 3.6` result registers `CSR[0x01<r>0ad]` at `ARC 0x9002015a + r * 0x2000` and `CSR[0x01<r>0af]` at `ARC 0x9002015e + r * 0x2000` **are** part of `phypara` Segment 0, where `CSR[0x0100ad] = 0x0002`.)*
- **Group `0x02` and `0x03` control/reset/ACSM**: `0x02001a`, `0x020020`–`0x020023`, `0x020027` (`= 0`, never `0x8000`), `0x020028`, `0x02002a`, `0x020031` (continuous-loop ACSM enable), `0x020041`, `0x020042`, `0x020044`, `0x020049` (`= 0x01f0`, never `1`), `0x020053`, `0x020060` (`MemResetL`), `0x020061` (`1 -> 0`, never `2`), `0x02006e` (`= 1` via `0x800004cc`), `0x020070`–`0x020073`, `0x020078`, `0x02007a`, `0x02007e`, `0x02008b`, `0x020090` (`= 1`), `0x0200a2`, `0x0200aa`, `0x0200c3`, `0x0200e0`–`0x0200e3`, `0x0200e5`, `0x0200f8`, `0x02011c` (`0x90040238` terminal status pulse), `0x020200`, `0x020303`, `0x02030d`, `0x020310`, `0x020311`, `0x020320`, `0x02f052`, `0x02f077`, `0x02f0a0` (`0x9005e140`: `1 -> delay(0x20) -> 0`), `0x02f0a5`, `0x03f01a` (`0x9007e034`), and `0x03f084` (`0x9007e108`).

By contrast, in the `0x0200c0`–`0x0200cf` range, `CSR[0x0200c3]` (`0x0001`), `CSR[0x0200c6]` (`0x0010`), `CSR[0x0200cd]` (`0x0001`), and `CSR[0x0200cf]` (`0x001f`) hold hardware power-on reset values that are never read by the PMU (`0x0200c5`, `0x0200c7`, `0x0200ca`, and `0x0200cc` are unimplemented and read as `0`), `CSR[0x0200c9]` (`0` references) is never polled by the PMU, and `CSR[0x020314]` / `CSR[0x020315]` (`ARC 0x90040628` / `0x9004062a`) are read-only hardware calibration output registers (`0` writes in host or PMU code) read by `Step 6.1`. In groups `0x0c` and `0x0d`, **the PMU never writes to `CSR[0x0c0080]` (`ARC 0x90180100`, read once in Phase 2), `CSR[0x0c0086]` (`ARC 0x9018010c`, 0 references in PMU firmware), or `CSR[0x0c00ee]` (`ARC 0x901801dc`, read once at startup to populate `byte[0x80000403] = 8..10`)**; the `2 -> 3 -> 7` transitions on `0x0c0080` are performed exclusively by the host SPL, and `0x0c0086` reflects hardware/host status. Neither `CSR[0x0d0036]` nor P-state SRAM (`0x0a0000`) is accessed during `DevInit`.

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
