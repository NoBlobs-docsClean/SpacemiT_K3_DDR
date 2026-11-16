<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: DDRC, DPLL & Host Ports](03-ddrc-and-clock.md) | [Index](README.md#reading-order-and-contents) | [Next: PMU Message Block →](05-message-block.md)

# Supported parts, topologies, and timing scope

## Evidence and scope

**Interface** facts describe the loader's tables and accepted inputs. **Observation** identifies the reference board and captured results. **Inference** and **unknown** are used where coverage or interpretation is not proven. This chapter is primarily about LPDDR5; LPDDR4X entries elsewhere in the tree do not extend the LPDDR5 timing or PMU contract described here.

## Supported LPDDR5 part table

| Part number | CRC-32 used for lookup | Ranks | Device organization flag | Listed density |
|---|---|---:|---|---:|
| `MT62F1G32D2DS` | `0x0fd38dd9` | 1 | x16 (`x8_mode = 0`) | 4096 MiB |
| `MT62F2G32D4DS` | `0x85d1f688` | 2 | x16 (`x8_mode = 0`) | 8192 MiB |
| `RS3G32LG5D8FDB` | `0xf74c6bfc` | 2 | x8 (`x8_mode = 1`) | 12288 MiB |
| `MT62F4G32D8DV` | `0x3acef2e4` | 2 | x8 (`x8_mode = 1`) | 16384 MiB |

These are table capacities, not proof of a particular board's total capacity across both DDRC instances. Physical channel population and system address interleaving are not fully established by the part table.

### LPDDR4X part table (for lookup reference only)

The same part-lookup table also lists three LPDDR4X entries so the loader can identify generation (`type = 0` for LPDDR4X, `type = 1` for LPDDR5):

| Part number | CRC-32 used for lookup | Ranks | Device organization flag | Listed density |
|---|---|---:|---|---:|
| `MT53E1G32D2FW` | `0x75251ab8` | 1 | x16 (`x8_mode = 0`) | 4096 MiB |
| `MT53E2G32D4DE` | `0x3ea87223` | 2 | x16 (`x8_mode = 0`) | 8192 MiB |
| `MT53E4G32D8CY` | `0xaa9d4848` | 2 | x8 (`x8_mode = 1`) | 16384 MiB |

**Interface.** The part string is obtained from board identification data (specifically the I2C EEPROM TLV record `TLV_CODE_DDR_PARTNUMBER`, surfaced as the `ddr_partnumber` environment variable), with a device-tree property as fallback. Lookup uses CRC-32 as a filter and requires an exact string match. If no entry matches, the current selection behavior falls back to the first table entry. That fallback is not automatic detection of an unknown memory device; a product may instead choose to reject it.

**Observation.** The reference board uses `MT62F2G32D4DS`, two ranks, x16 organization, and 6400 MT/s. Its persisted training envelope agrees with those settings.

## Supported rates and topology matrix

The LPDDR5 initialization path accepts only:

- **4800 MT/s**, with 3000 MHz DPLL programming, CFG selector `7 << 19`, and `DRAMFreq = 0x12c0`;
- **6400 MT/s**, with 3200 MHz DPLL programming, CFG selector `2 << 19`, and `DRAMFreq = 0x1900`.

**Interface.** The pre-training base table is for dual-rank x8 at 6400 MT/s. Overrides cover:

| Topology | 4800 MT/s | 6400 MT/s |
|---|---|---|
| Single-rank x16 | PHY override present | PHY override present |
| Dual-rank x16 | PHY override present | PHY override present; reference timing configuration available |
| Dual-rank x8 | PHY override present | Base PHY configuration; reference timing configuration available |

Concrete examples of message-block and PHY CSR overrides applied by these topology tables include:

- **4800 MT/s override**: sets DMEM index `0x58003` (`DRAMFreq`) to `0x12c0` (4800 MT/s).
- **Single-rank x16 override**: adjusts the `X8Mode`, enabled-DQ, and CS-presence message fields through DMEM indices `0x5800e`, `0x58012`, and `0x58020`. Entries that must not be written use the `0xdeadbeef` skip sentinel.

The message block carries `EnabledDQsChA/B` and `CsPresentChA/B` masks. These masks, the x8/x16 overrides, and DDRC rank/address-map settings must describe the same topology. A part match alone is not sufficient.

**Unknown.** The existence of table entries and overrides is not equivalent to board-level validation of every part at both rates. No universal training-result compatibility should be assumed across topology, rate, part, or PMU-program changes.

## PHY configuration precedence

For a given PHY CSR write, priority from lowest to highest is:

1. base pre-training/training table value;
2. rate/topology override;
3. I/O override.

A topology override value of `0xdeadbeef` suppresses that write. The host table representation carries 16-bit CSR values even though it addresses the DPHY through 32-bit words.

Default LPDDR5 I/O policy in the interface is:

| Setting | Configured policy | PHY / DRAM encoding | Target CSRs / Message fields |
|---|---|---|---|
| PHY write drive (TX DS) | R30 (`30 Ω`) | `0x0f` (`v \| (v << 8)` = `0x0f0f`) | DQ rank-global `0x10040, 42..45 + rank*0x1000`; CA `0x30040..43 + rank*0x1000` |
| PHY receive ODT (RX ODT) | R60 (`60 Ω`) | `0x0c` (`v << 8` = `0x0c00`) | DQ rank-global `0x10048, 4a..4d + rank*0x1000` |
| DRAM DQ ODT | R60 (`60 Ω`) | `4` (JEDEC MR11 code) | Message block `MR11` (`0x58035`–`0x58036`) |
| DRAM CA ODT | R80 (`80 Ω`) | `3` (JEDEC MR11 code) | Message block `MR11` (`0x58035`–`0x58036`) |
| DRAM NT/SoC ODT | Disabled (`R_OFF`) | `0` | Message block `MR17` / `MR41` |
| DRAM PDDS | R40 (`40 Ω`) | `6` (JEDEC MR3 code) | Message block `MR3` (`0x58031`–`0x58032`) |
| 2D training | Enabled (`1`) | `Disable2D = 0` | Message block `0x58067` |

Supported PHY impedance codes are `OFF = 0x00`, `120 Ω = 0x08`, `60 Ω = 0x0c`, `40 Ω = 0x0e`, and `30 Ω = 0x0f`; supported DRAM impedance codes are `R_OFF = 0`, `R_240 = 1`, `R_120 = 2`, `R_80 = 3`, `R_60 = 4`, `R_48 = 5`, and `R_40 = 6`.

### Measured hardware sensitivity to PHY TX drive strength and RX ODT

**Observation.** Sweeping PHY TX drive strength and RX ODT across all five impedance codes on the 6400 MT/s dual-rank reference board established two distinct electrical behaviors:

1. **PHY TX Drive Strength (`0x10040, 42..45 + rank*0x1000`, `v | (v << 8)`) — step-function threshold**:
   | TX Drive Strength | Code (`0x10040`) | Training Duration | Measured Outcome |
   |---|---:|---:|---|
   | `30 Ω` (default) | `0x0f0f` | 5,624 ms | Full training succeeds |
   | `40 Ω` | `0x0e0e` | 5,619 ms | Full training succeeds; trained `0x4e/0x4f` identical to `30 Ω` within run-to-run noise |
   | `60 Ω` | `0x0c0c` | **41 ms** | **Aborts** before `DevInit` completes (`0x100a3 = 0x0000`, `0x4e/0x4f = 0x35`) |
   | `120 Ω` | `0x0808` | **18 ms** | **Aborts** early |
   | `OFF` | `0x0000` | **12 ms** | **Aborts** immediately |

   The electrical cliff lies between **40 Ω and 60 Ω**, and `0x0f` (`30 Ω`) is the strongest setting supported by the PHY encoding. Consequently, TX drive strength is already at its ceiling and cannot be increased further for additional margin; conversely, an early training abort (`12`–`41 ms` with `0x100a3 = 0`) is a clear diagnostic signature of insufficient drive or broken signal closure.

2. **PHY RX ODT (`0x10048, 4a..4d + rank*0x1000`, `v << 8`) — monotonic delay curve**:
   | RX ODT | Code (`0x10048`) | Median `0x4e/0x4f` (taps) | Observed Range (hex) | Training Duration |
   |---|---:|---:|---|---:|
   | `OFF` | `0x0000` | **106** | `0x65`–`0x6e` | 4,434 ms |
   | `120 Ω` | `0x0800` | **82** | `0x45`–`0x5f` | 5,626 ms |
   | `60 Ω` (default) | `0x0c00` | **62** | `0x2d`–`0x4a` | 5,624 ms |
   | `40 Ω` | `0x0e00` | **50** | — | 5,602 ms |
   | `30 Ω` | `0x0f00` | **42** | `0x17`–`0x37` | 5,577 ms |

   Stronger receive termination monotonically reduces the trained read-eye center across a **~64-tap span** (more than 20× the ±3-tap run-to-run jitter), and no setting aborts training. Because the default `60 Ω` sits in the middle of the usable range (`30`–`120 Ω`), RX ODT provides a genuine tuning lever for receive eye optimization.

### Host `io_override` group-matching caveat

**Interface + observation.** When the reference loader applies `io_override` entries on top of a base table, its table-walk filter checks whether `(override_offset & ~0x7fff)` matches the current base table's starting address. Because the `io_override` list only emits rank-global `lane 0` CSRs and specific message-block offsets, any override entry whose `(offset & ~0x7fff)` group does not match the active table pass is silently skipped. On the reference board, this causes `MR17` (`0x58041`, byte `0x082`) in the captured envelope to retain the base-table value **`0x2e`** rather than the `0x28` value computed by the I/O override helper. An independent host implementation should apply I/O overrides deterministically after all base and topology tables have been written, or preserve the exact `0x2e` `MR17` value expected by the reference profile.

### Host pre-training `CSR[0x01<r>00f]` initialization

**Interface + observation.** The reference pre-training table initializes all four per-rank CSRs `CSR[0x01<r>00f]` (`r = 0..3`: `0x01000f`, `0x01100f`, `0x01200f`, `0x01300f`) to **`0x1086`** (with bit 7 = `0x0080` set) at 6400 MT/s. Leaving bit 7 clear (`0x1006`) alters the initial value read before training (`0x01<r>00f` becomes `0x1004` rather than `0x1084` before host post-training writes `0x1089`). An independent host loader should write `0x1086` (bit 7 set) to all four `0x01<r>00f` entries prior to PMU start.

## Host DDRC timing dependencies

Host-side DDRC timing selection is not a simple frequency scaling exercise:

- At 6400 MT/s, changing from one to two x16 ranks changes DRAM timing entries, DFI values, rank timing, `mstr0` (`0x01080008` for 1 rank vs. `0x03080008` for 2 ranks), `rfshmod0` (`0x010003f3` vs. `0x00000361`), and derating controls.
- At two x16 ranks, changing from 6400 to 4800 MT/s changes rate-dependent timing and refresh/ZQ values.
- At 6400 MT/s and two ranks, changing x16 to x8 changes rank and operational timing fields.

**Interface.** A nanosecond-to-cycle conversion commonly needs upward rounding, representable as `ceil(time_ns * data_rate_MT_per_s / 2000)`. **Unknown.** That arithmetic alone is insufficient to reconstruct the full host DDRC timing table: register-specific rounding, minimum/maximum constraints, mode-register rules, rank interactions, and DFI/PHY timing requirements also apply.

## Persistence and timing compatibility

Each controller's quickboot result contains a 320-byte message block, 5170 PHY halfwords, and 4096 ACSM halfwords. The 5170-halfword PHY serialization has a fixed 69-segment order; contiguous and sparse segments must remain in that exact order.

**Interface.** Quickboot acceptance checks the envelope identity, DDR type, ranks, data rate, and CRC. This ties a result to selected configuration properties but does not turn training output into a universal timing database. P-state SRAM restoration is not active in the documented quickboot path.

## Coverage gaps to preserve

- Full DDRC bitfield definitions and all initialization-phase transitions are not established.
- PHY lane timing, trained delay values, and most streaming-opcode meanings are outside host DDRC timing configuration.
- Support in a lookup or override table does not establish electrical validation for another board or memory package.

## PHY table descriptor formats

PHY configuration tables applied by the host use two descriptor representations:

- **Discrete table**: an ordered list of explicit `(csr_index, halfword_value)` pairs.
- **Linear table**: a descriptor specifying `base` (starting CSR word index), `count` (number of entries), `linear_increase` (`1` when indices increment by 1), and a pointer to the contiguous halfword values.

## Summary of size and sentinel constants

| Constant | Value | Scope / Usage |
|---|---|---|
| Message block useful / reserved size | `0xa0` / `0x200` halfwords (`0x140` / `0x400` bytes) | DMEM at index `0x58000` |
| Serialized `phypara` length | `5170` halfwords (`0x2864` bytes across 69 segments) | Result collection & quickboot (`0x58200`) |
| ACSM SRAM snapshot length | `4096` halfwords (`0x2000` bytes) | Result collection & quickboot (`0x41000`) |
| P-state SRAM size | `16384` halfwords (`1 << 14`) | Index `0xa0000` (optional / inactive in current quickboot) |
| Per-controller saved record | `0x49a4` bytes | Persistent envelope (`0x140 + 0x2864 + 0x2000`) |
| Reserved two-controller envelope | `0x9800` bytes | Persistent storage (`bootinfo` offset `0x10000`) |
| Host memory smoke-test block | `0x4000` bytes | Post-activation verification |
| Envelope magic | `0x54524444` | ASCII `"DDRT"` |
| Topology override skip sentinel | `0xdeadbeef` | Suppresses the corresponding CSR write |
| Mailbox major completion | `0x07` | Normal PMU training/quickboot completion |
| Mailbox streaming introducer | `0x08` | Followed by word count and 32-bit framed words |
| Mailbox fatal error | `0x000000ff` | Terminal PMU error notification |
