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
2. **PHY block-enable and channel-release phases completed** (`CSR[0x0200a2]` and `CSR[0x0200f8]` asserted, and channel resets released; see Phase 1 and Phase 2 below);
3. **`CSR[0x020011]` (`ARC 0x90040022`) bits `[6:5]` cleared**: The host pre-training table initializes `CSR[0x020011]` to **`0x006f`**. At the start of `DevInit` Phase 3, the PMU saves the original value (`0x006f`) to DMEM `0x8000042a` and writes `0x006f & 0xff9f = 0x000f` to `CSR[0x020011]` (`0x000f` is also the final value captured in post-training `phypara`).

Once those preconditions hold, the PMU calibrates the clock period and frequency-bucket CSRs using the following sequence:

1. **Enable period counters (`CSR[0x0200e3]` and `CSR[0x0200e5]`)**:
   - Read and save `u_e3 = CSR[0x0200e3]` (`ARC 0x900401c6`), then write `CSR[0x0200e3] = u_e3 | 0x0c00` (setting **bits `[11:10]`**; writing `0x0001` does **not** enable the counters and leaves `0x0300bf` and `0x0100e0` at `0`).
   - Read and save `u_e5 = CSR[0x0200e5]` (`ARC 0x900401ca`), then write `CSR[0x0200e5] = u_e5 & 0xffe0` (clearing bits `[4:0]`).
2. **Wait for measurement accumulation**:
   - Delay for a **fixed 10 hardware RTC ticks** (no busy bit is polled).
3. **Read period counters and program `CSR[0x02000a]` and `CSR[0x020017]`**:
   - Read CA period counter `T_ca = CSR[0x0300bf] & 0x01ff` (`ARC 0x9006017e`) and write `T_ca` to `CSR[0x02000a]` (`ARC 0x90040014`, typically `0x018f`–`0x0190` = 399–400 at 6400 MT/s).
   - Read DQ period counter `T_dq = CSR[0x0100e0] & 0x01ff` (`ARC 0x900201c0`) and write `T_dq` to `CSR[0x020017]` (`ARC 0x9004002e`, `0x0100` = 256 at 6400 MT/s).
4. **Restore counter control registers**:
   - Write `CSR[0x0200e3] = u_e3` and `CSR[0x0200e5] = u_e5`.
5. **Compute frequency bucket and program `CSR[0x020012]`, `CSR[0x020186]`, and `CSR[0x020187]`**:
   - Compute `val = DRAMFreq / DfiFreqRatio` from message-block fields `DRAMFreq` (`0x80000006`, 16-bit) and `DfiFreqRatio` (`0x80000008`, 8-bit). If `DfiFreqRatio == 2` (`byte[0x008] == 2`), double `val = 2 * val` (note that this check tests `DfiFreqRatio == 2`, not `Pstate == 0`).
   - Classify `val` across the six intervals defined by the five thresholds (`0x014b` = 331, `0x0213` = 531, `0x031f` = 799, `0x0707` = 1799, `0x0897` = 2199) to obtain `coarse` (bits `[7:4]`) and `fine` (bits `[3:0]` and `[11:8]`):

     | Interval for `val` | `coarse` (`[7:4]`) | `fine` (`[3:0]`, `[11:8]`) |
     |---|---:|---:|
     | `val < 0x014b` (`< 331`) | `0x20` | `1` |
     | `0x014b <= val <= 0x0213` (`331..531`) | `0x30` | `0` |
     | `0x0213 < val <= 0x031f` (`532..799`) | `0x40` | `2` |
     | `0x031f < val <= 0x0707` (`800..1799`) | `0x50` | `3` |
     | `0x0707 < val <= 0x0897` (`1800..2199`) | `0x60` | `3` |
     | `val > 0x0897` (`> 2199`) | `0x70` | `3` |

   - Combine `coarse` and `fine` with the secondary period bucket `bucket3` (`2..7`, derived from the measured period counter; on the reference 6400 MT/s board `2 * T_ca ≈ 798..800` sits on the `799` boundary):
     - Write `CSR[0x020012]` (`ARC 0x90040024`) = `(bucket3 << 12) | (fine << 8) | coarse | fine` (yielding `0x3221` when below the `799` threshold or `0x4332` when at/above `800`);
     - Write `CSR[0x020186]` (`ARC 0x9004030c`) = `(bucket3 << 4) | fine` (yielding `0x0201` or `0x0302`);
     - Copy `CSR[0x020187]` (`ARC 0x9004030e`) = `CSR[0x0100d5]` (`ARC 0x900201aa`, `0x0201` or `0x0302`).

### Complete `DevInit` (`SequenceCtrl[0]`) hardware initialization sequence

While comparing a `SequenceCtrl = 0x0001` capture against `0x0000` shows **68 changed `phypara` entries** (50 in group `0x01`—namely `0x01<r><l>50` and `0x51` across all 36 lanes plus `0x01<r>0a3` and `0x0100d5`—along with 16 in group `0x02` and `0x09080b`/`0x09080c`), the complete `DevInit` stage actually executes **507 CSR write operations** across six ordered hardware phases. Many of those writes target dynamic control, reset, and strobe registers outside the 69 `phypara` serialization segments.

#### Configuration commit strobe triad (`0x020028`, `0x02f0a5`, `0x02f077`)

Throughout `DevInit`, whenever the PMU updates a group of PHY control CSRs and needs to latch/commit the configuration into the hardware state machine, it issues this three-write strobe sequence (~40 times across `DevInit`):

```text
CSR[0x020028] = 1    (ARC 0x90040050 = 1)
CSR[0x02f0a5] = 1    (ARC 0x9005e14a = 1)
CSR[0x02f077] = 0    (ARC 0x9005e0ee = 0)
```

#### Ordered `DevInit` phases

1. **Phase 1 — PHY block enable (`CSR[0x0200a2]` and `CSR[0x0200f8]`)**:
   - Read-modify-write `CSR[0x0200a2]` (`ARC 0x90040144`) `|= 0x0100` (set bit 8).
   - Read-modify-write `CSR[0x0200f8]` (`ARC 0x900401f0`) `|= 0x0001` (set bit 0).
2. **Phase 2 — Channel release**:
   - For each channel `ch = 0, 1`:
     - Write `CSR[0x020028] = 1`, `CSR[0x02f077] = 0`, `CSR[0x02f0a0] = 0` (`ARC 0x9005e140`), and `CSR[0x020027] = 0x8000` (`ARC 0x9004004e`).
     - Wait **23 hardware RTC ticks**.
     - Write `CSR[0x02f0a0] = 1`, wait **23 RTC ticks**, and write `CSR[0x02f0a0] = 0`.
3. **Phase 3 — Topology evaluation, `QBPll*` read-back, DQ block reset, and clock calibration**:
   - Save `CSR[0x020011]` (`0x006f`) to DMEM `0x8000042a` and write `CSR[0x020011] &= 0xff9f` (`0x000f`).
   - Clear `CSR[0x0000a4] = 0` (`ARC 0x90000148`).
   - Write `CSR[0x020073]` (`ARC 0x900400e6`): `1` if `PllBypassEn != 0` (`0x80000005`), else `0`.
   - Write `CSR[0x020072]` (`ARC 0x900400e4`): `(Pstate & 0x0f)` (`0x80000004`).
   - Evaluate `CsPresentChA` (`0x80000025`) and `CsPresentChB` (`0x80000040`): if no ranks are populated on either channel, abort immediately through the fatal `0x000000ff` mailbox path.
   - Write `CSR[0x02007a]` (`ARC 0x900400f4`) with the active channel mask (`1` for single-channel, `3` when both channels are populated) and `CSR[0x02007e]` (`ARC 0x900400fc`) with the rank-enable mask (`(CsPresentChB << 2) | CsPresentChA`, i.e. `0x000f` on dual-rank ChA+ChB).
   - Write `CSR[0x0c0080] = 3` (`ARC 0x90180100`, enabling PMU/HCLK clocks).
   - If `Pstate & 0x20` is non-zero, copy the seven `QBPll*` CSRs (`0x020088`–`0x02008c`, `0x02004a`–`0x02004c`) to message-block offsets `0x80000130`–`0x8000013c` (see [05-message-block.md](05-message-block.md)).
   - Assert DQ block reset on the active channel selection (`CSR[0x01f00f] = 0` at `ARC 0x9003e01e`, `CSR[0x01f0d9] = 1` at `ARC 0x9003e1b2`), wait **10 RTC ticks**, then release (`CSR[0x01f0d9] = 0`, `CSR[0x01f00f] = 1`, `CSR[0x01f0ac] = 0x00ff` at `ARC 0x9003e158`).
   - Execute the **master clock-period calibration** sequence above (`0x02000a`, `0x020017`, `0x020012`, `0x020186`, `0x020187`).
   - Write `CSR[0x02008b] = 0` (`ARC 0x90040116`), `CSR[0x020049] = 1` (`ARC 0x90040092`), and `CSR[0x020061] = 2` (`ARC 0x900400c2`).
4. **Phase 4 — Second-channel DQ block reset**:
   - Switch `CSR[0x02007a] = 2` (Channel B selector) and repeat the DQ block reset pulse (`CSR[0x01f00f] = 0`, `CSR[0x01f0d9] = 1` → wait **10 RTC ticks** → `CSR[0x01f0d9] = 0`, `CSR[0x01f00f] = 1`, `CSR[0x01f0ac] = 0x00ff`), then restore `CSR[0x02007a] = 3` (both channels).
5. **Phase 5 — DRAM reset (`MemResetL`) and LPDDR5 `tINIT` timing sequence**:
   - Configure the PHY reset/timing control registers (`CSR[0x020078] = 0x8000` at `ARC 0x900400f0`, `CSR[0x020070]`/`0x020071` at `ARC 0x900400e0`/`0x900400e2`, `CSR[0x020053]` at `ARC 0x900400a6`, `CSR[0x0200e0]`–`0x0200e2` at `ARC 0x900401c0`–`0x900401c4`, `CSR[0x0200c3]` at `ARC 0x90040186`, `CSR[0x020020]` at `ARC 0x90040040`, and `CSR[0x02002a]` at `ARC 0x90040054`) and issue the commit strobe triad (`0x020028 = 1`, `0x02f0a5 = 1`, `0x02f077 = 0`).
   - Sequence `MemResetL` at `CSR[0x020060]` (`ARC 0x900400c0`) across the LPDDR5 initialization intervals (`tINIT1` / `tINIT3` / `tINIT4` / `tINIT5`):
     1. Write `CSR[0x020060] = 2` and wait **10 RTC ticks**;
     2. Write `CSR[0x020060] = 0` (assert DRAM `RESET_n` low) and wait **200,000 scaled ticks** (`tINIT1` / `tINIT2`);
     3. Write `CSR[0x020060] = 1` (deassert DRAM `RESET_n` high) and wait **2,000,000 scaled ticks** (`tINIT3`, ~2 ms);
     4. Issue the commit strobe triad and wait **2,125 scaled ticks** (`tINIT5`).
6. **Phase 6 — Clock-enable transition and per-lane DQ initialization**:
   - Update `CSR[0x0c0080]` (`ARC 0x90180100`): write `2`, then `3`, and after per-lane setup write `7`.
   - Save the four per-rank `CSR[0x01<r>0a3]` values (`0x0b33`, written by the host pre-training table) into DMEM `0x80000430`–`0x80000437` and program the per-lane `0x01<r><l>50`/`0x51` and `0x01f00a` (`ARC 0x9003e014`) initialization state.
   - Update status register `CSR[0x0c0086]` (`ARC 0x9018010c`) from `1` to `3` upon `DevInit` completion.

#### Summary of `DevInit`-written CSRs outside `phypara`

The following CSRs are written by `DevInit` to sequence hardware state machines and resets, and are **not** part of the 5,170-entry `phypara` table:

- **Group `0x00`**: `0x0000a4` (`ARC 0x90000148`);
- **Group `0x01` broadcast (`0x01fxxx`)**: `0x01f00a` (`0x9003e014`), `0x01f00f` (`0x9003e01e`), `0x01f0ac` (`0x9003e158`), `0x01f0d9` (`0x9003e1b2`);
- **Group `0x02` control/reset**: `0x020020`, `0x020027`, `0x020028`, `0x02002a`, `0x020042`, `0x020044`, `0x020049`, `0x020053`, `0x020060` (`MemResetL`), `0x020061`, `0x020070`–`0x020073`, `0x020078`, `0x02007a`, `0x02007e`, `0x02008b`, `0x0200a2`, `0x0200c3`, `0x0200e0`–`0x0200e2`, `0x0200f8`, `0x02011c` (`0x90040238` terminal status pulse), `0x020200`, `0x020303`, `0x02030d`, `0x020310`, `0x020311`, `0x020320`, `0x02f077`, `0x02f0a0`, `0x02f0a5`;
- **Group `0x0c`**: `0x0c0080` (`UCCLK_HCLK_ENABLES`).

By contrast, in the `0x0200c0`–`0x0200cf` range, `CSR[0x0200c3]` (`0x0001`), `CSR[0x0200c6]` (`0x0010`), `CSR[0x0200cd]` (`0x0001`), and `CSR[0x0200cf]` (`0x001f`) hold hardware power-on reset values that are never read by the PMU (`0x0200c5`, `0x0200c7`, `0x0200ca`, and `0x0200cc` are unimplemented and read as `0`). In groups `0x0c` and `0x0d`, the only registers accessed by the PMU outside the four mailbox registers are `CSR[0x0c0080]` (`ARC 0x90180100`) and `CSR[0x0c0086]` (`ARC 0x9018010c`); neither `CSR[0x0d0036]` nor P-state SRAM (`0x0a0000`) is accessed during `DevInit`.

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
