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

| `SequenceCtrl` | Enabled stages | Duration (ms) | Incremental CSR writes vs. `0x0000` | Primary CSR / message-block effects |
|---|---|---:|---:|---|
| `0x0000` | None (PMU framework only) | 8 | 0 (baseline) | Enters and exits cleanly without touching `0x02xxxx` calibration CSRs |
| `0x0001` | `DevInit` | 15 | 68 | Calibrates group `0x02` master clock period (`0x02000a`, `0x020017`, `0x020012`, `0x020186`, `0x020187`), initializes ACSM (`0x040020`, `0x0400fd`, `0x040100`), updates `0x0c0086` (`0x0003`) |
| `0x0007` | `+ WrLvl + RxEn` | 28 | 165 | Writes per-lane write-leveling and receive-enable coarse/fine CSRs (`0x01000c/0d = 0x009a/0x009b`, `0x010026 = 0x00ba`, etc.) |
| `0x000f` | `+ RdDQS` | 3,567 | 559 | **63% of total runtime (~3.54 s):** trains `CSR[0x4e]` and `CSR[0x4f]` across all 36 lanes (4 ranks × 9 lanes) and populates `RxClkDly_Margin` / `VrefDac_Margin` |
| `0x001f` | `+ WrDq` | 4,413 | 598 | **~846 ms:** trains write DQ eye delays (`0x01<r><l>10..13`, `0x01<r><l>be`) and populates `TxDqDly_Margin`, `DeviceVref_Margin`, and trained `MR14`/`MR15` |
| `0x005f` | `+ DRAM DCA` | 5,611 | 484 | **~1.20 s:** adjusts per-bit transmit delays (`0x01<r><l>68..6f` shifted by ~+4 taps) and updates `MR30` |
| `0x025f` | `+ MxRdLat` | 5,613 | 503 | **~2 ms:** updates DFI maximum read latency CSRs (`0x070011`, `0x070060..63`) and `0x090813` |
| `0x125f` | `+ LPCA` (full mask) | 5,624 | 587 | **~11 ms:** trains CA VREF (`MR12`) and LP5 CS timing (`0x030001 = 0x0004`) |

Note that running `SequenceCtrl = 0x0009` (`DevInit + RdDQS` only, skipping `WrLvl` and `RxEn` because the pre-training tables already hold nominal leveling values) completes in ~3.55 s and produces `CSR[0x4e]`/`CSR[0x4f]` values within 1–2 taps of a full `0x125f` run.

### `DevInit` master clock-period calibration (`0x02xxxx`)

During `DevInit` (`SequenceCtrl[0]`), the PMU measures the live clock period from two hardware period counters and programs five group `0x02` CSRs:

1. Save `CSR[0x020011]` (`ARC 0x90040022`), clear bits `[6:5]` (`val & 0xff9f`), and write it back during calibration.
2. Trigger the period measurement at `CSR[0x0200e3]` (`ARC 0x900401c6`), then read:
   - CA period counter `T_ca = CSR[0x0300bf] & 0x01ff` (`ARC 0x9006017e`), stored to `CSR[0x02000a]` (`ARC 0x90040014`, typically `0x0190` = 400 at 6400 MT/s);
   - DQ period counter `T_dq = CSR[0x0100e0] & 0x01ff` (`ARC 0x900201c0`), stored to `CSR[0x020017]` (`ARC 0x9004002e`, `0x0100` = 256).
3. Compute `val = 2 * T_ca` (when `Pstate` parameter = 0, else `T_ca`) and classify `val` against five frequency thresholds (`0x014b`, `0x0213`, `0x031f` = 799, `0x0707`, `0x0897`) to select 4-bit `coarse`, `fine`, and `bucket3` codes:
   - Write `(bucket3 << 12) | (fine << 8) | (fine << 4) | coarse` to `CSR[0x020012]` (`ARC 0x90040024`; on the reference board `val ≈ 800` sits on the `799` boundary, producing `0x3221` when `val <= 799` or `0x4332` when `val >= 800`);
   - Write `(fine << 8) | coarse` to `CSR[0x020186]` and `CSR[0x020187]` (`ARC 0x9004030c`/`0x9004030e`, producing `0x0201` or `0x0302`).
4. Restore the original `CSR[0x020011]` value (`0x0041`) before `DevInit` exits.

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
