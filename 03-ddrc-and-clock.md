<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Addressing & Access Rules](02-addressing-and-access.md) | [Index](README.md#reading-order-and-contents) | [Next: Parts, Topologies & Timings →](04-parts-topologies-and-timings.md)

# DDRC, DPLL, and host-port control

## Scope and evidence

This chapter covers the host-programmed clock and controller path for each of the two K3 LPDDR5 instances. **Interface** facts come from loader behavior and extracted configuration outputs; **observations** come from board execution. Phase interpretation is marked **inference**, and missing hardware definitions are **unknown**.

All offsets in this chapter are **byte offsets**, not DPHY word indices. Use DDRC bases `0xcb000000` and `0xcc000000`; CFG bases are respectively `0xcb600000` and `0xcc600000`.

## DPLL and clock selection

**Interface.** The supported LPDDR5 paths program the following values:

| DRAM data rate | DPLL setting | Source selection in `CFG+0x18` | PMU `DRAMFreq` |
|---:|---:|---:|---:|
| 4800 MT/s | 3000 MHz | `7 << 19` | `0x12c0` |
| 6400 MT/s | 3200 MHz | `2 << 19` | `0x1900` |

`DRAMFreq` is the rate in MT/s represented in hexadecimal (`4800 = 0x12c0`, `6400 = 0x1900`), not the DPLL frequency in MHz.

| CFG byte offset | Established role |
|---:|---|
| `0x08` | Part of the 3000 MHz DPLL programming path |
| `0x0c` | DPLL configuration; 3200 MHz is programmed here on the default 6400 path |
| `0x10` | Additional 3000 MHz DPLL programming |
| `0x18` | Divisor/source selection, clock-switch kick, DCLK gate, and final enable |
| `0x1c` | Bit 0 polled for DPLL lock |

The observed sequence is:

1. program the DPLL;
2. wait for `CFG+0x1c[0]` to indicate lock;
3. clear `CFG+0x18[21:16]` and install the rate-specific source selection;
4. set bit 25 to start the switch and wait for it to clear;
5. set bit 0 for the final enable;
6. before PHY execution, set bit 22 to release DCLK.

**Unknown.** The DPLL multiplier/divider field meanings and the relationship between every internal clock are not fully documented here. In particular, `DfiFreqRatio` must be preserved as the interface value expected by the PMU; the sources do not justify redefining it as a generic WCK/MemClk ratio. Rates other than 4800 and 6400 MT/s are rejected by this LPDDR5 path rather than approximated.

## DDRC configuration stages

**Interface.** Before training, the loader configures:

- DRAM timing sets for the selected part, rate, rank count, and x8/x16 organization;
- DFI frequency, update, PHY-master, and low-power timing controls;
- refresh and derating;
- ZQ calibration controls;
- rank presence and CS/address mapping;
- power and initialization state.

After training, it collects PHY state, transitions DFI to normal operation, waits for DDRC operational status, adjusts refresh/ZQ state, exits self-refresh, and only then enables host ports.

The `swctl`/`swstat` handshake brackets applicable configuration updates. `stat[1:0] == 01` is the final operational-state condition used by the loader.

## Principal DDRC registers

The table records roles established by the interface. It does not assign undocumented bitfields.

| Register/group | Byte offset(s) | Role in this flow |
|---|---|---|
| DRAM timing sets | `0x00000`–`0x00080` | Rate/topology-specific DRAM timing values |
| DFI timing/update | `0x00580`–`0x00594`, `0x005a4`–`0x005b8` | DFI frequency, low-power, and update timings |
| Refresh | `0x00600`–`0x00608`, `0x00650` | Refresh timing and related controls |
| ZQ timing/configuration | `0x00800`–`0x00804` | Rate/topology-specific ZQ values in the timing output |
| Derating | `0x00b04`–`0x00b08` | Temperature/rate-related derating configuration |
| Rank timing/config | `0x00d00`, `0x00d04`, `0x00d08` | Timing configuration and rank-dependent values |
| `mstr0` | `0x10000` | Master DDR configuration, including rank-dependent state |
| `deratectl1/2` | `0x10104`, `0x10108` | Derating controls |
| `pwrctl` | `0x10180` | Power/self-refresh sequencing |
| `rfshmod0` | `0x10200` | Refresh mode, with rank-dependent values |
| `sched0` | `0x10380` | Scheduler configuration |
| `dfiupd0` | `0x10508` | DFI update control |
| `dfimisc` | `0x10510` | DFI initialization and operational transitions |
| `dfiphymstr` | `0x10518` | DFI/PHY-master interface |
| `opctrl1` | `0x10b84` | Operational control during initialization |
| `inittmg0` | `0x10d00` | Initialization timing |
| `initmr0` | `0x00500` | Initial mode-register configuration |
| `rfshctl0` | `0x10208` | Refresh control (`1` during init/training, `0` in normal operation) |
| `zqctl0`, `zqctl2` | `0x10280`, `0x10288` | ZQ calibration control (`0x80000000` / `1` during init, `0` / `0` in normal operation) |
| `swctl`, `swstat` | `0x10c80`, `0x10c84` | Quasi-static register update transaction (`swctl = 0/1`, poll `swstat[0]`) |
| `dfistat` | `0x10514` | DFI status (`bit 0` polled during `dfimisc` transition) |
| `stat` | `0x10014` | Controller status; final wait for bits `[1:0] == 01` |

**Unknown.** The sources do not provide a complete authoritative register-name map proving how every low-offset ZQ timing register (`0x00800`–`0x00804`) and high-offset ZQ control register (`0x10280`, `0x10288`) corresponds to Synopsys architectural names. Preserve their offsets and sequence instead of treating them as aliases.

## Reference DDRC timing table across supported topologies

**Interface.** The table below lists the 47 DDRC timing and configuration registers programmed for the reference LPDDR5 part (`MT62F2G32D4DS`) across the four supported topology variants (`rate / ranks / x8_mode`). These values represent the JEDEC LPDDR5 timing parameters converted to clock cycles (`ceil(time_ns * data_rate_MT_per_s / 2000)`) and packed into the controller's hardware bitfields. For a different memory device, recompute the timing fields from its datasheet while respecting controller field minimums.

| Byte offset | Register | `6400 / 2R / x16` (`x8=0`) | `6400 / 1R / x16` (`x8=0`) | `4800 / 2R / x16` (`x8=0`) | `6400 / 2R / x8` (`x8=1`) |
|---:|---|---:|---:|---:|---:|
| `0x00000` | `dramset1tmg0` | `0x28103622` | `0x28103622` | `0x1f0c291a` | `0x29103622` |
| `0x00004` | `dramset1tmg1` | `0x00100630` | `0x00100630` | `0x000c0524` | `0x00100630` |
| `0x00008` | `dramset1tmg2` | `0x09111117` | `0x09111117` | `0x070d0f13` | `0x09121219` |
| `0x0000c` | `dramset1tmg3` | `0x000c212f` | `0x000c212f` | `0x000a1a25` | `0x000c2230` |
| `0x00010` | `dramset1tmg4` | `0x0f04040f` | `0x0f04040f` | `0x0b04030b` | `0x0f04040f` |
| `0x00014` | `dramset1tmg5` | `0x02040c09` | `0x02040c09` | `0x02030907` | `0x02040c09` |
| `0x00018` | `dramset1tmg6` | `0x00000012` | `0x00000012` | `0x0000000e` | `0x00000012` |
| `0x0001c` | `dramset1tmg7` | `0x00c90003` | `0x00c90003` | `0x00970002` | `0x00c90003` |
| `0x00024` | `dramset1tmg9` | `0x00020410` | `0x00020410` | `0x0002030d` | `0x00020412` |
| `0x00030` | `dramset1tmg12` | `0x00030000` | `0x00030000` | `0x00030000` | `0x00030000` |
| `0x00034` | `dramset1tmg13` | `0x0c100002` | `0x0c100002` | `0x0a100002` | `0x0c100002` |
| `0x00038` | `dramset1tmg14` | `0x002000e6` | `0x002000e6` | `0x001800ad` | `0x002000e6` |
| `0x0005c` | `dramset1tmg23` | `0x009d0009` | `0x009d0009` | `0x00760007` | `0x009d0009` |
| `0x00060` | `dramset1tmg24` | `0x000f160e` | `0x000f160e` | `0x000d120c` | `0x0010170e` |
| `0x00064` | `dramset1tmg25` | `0x00002806` | `0x00002806` | `0x00001f05` | `0x00002906` |
| `0x00078` | `dramset1tmg30` | `0x00191318` | `0x00191318` | `0x00141114` | `0x001a1419` |
| `0x00580` | `dfitmg0` | `0x033f021f` | `0x033f021f` | `0x032f0217` | `0x0343021f` |
| `0x00584` | `dfitmg1` | `0x00080303` | `0x00080303` | `0x00080303` | `0x00080303` |
| `0x00588` | `dfitmg2` | `0x00183f1f` | `0x00183f1f` | `0x00182f17` | `0x0018431f` |
| `0x00590` | `dfitmg4` | `0x180c0403` | `0x180c0403` | `0x10080403` | `0x1c0c0403` |
| `0x00594` | `dfitmg5` | `0x0410000f` | `0x0410000f` | `0x040c000f` | `0x0410000f` |
| `0x005a4` | `dfilptmg1` | `0x00000300` | `0x00000300` | `0x00000300` | `0x00000300` |
| `0x005a8` | `dfiupdtmg0` | `0x71a4000d` | `0x71a4000d` | `0x7140000d` | `0x71a4000d` |
| `0x005b4` | `dfiupdtmg2` | `0xe000012c` | `0xe000012c` | `0xe000012c` | `0xe000012c` |
| `0x005b8` | `dfiupdtmg3` | `0x12c00147` | `0x12c00147` | `0x12c000e7` | `0x12c00147` |
| `0x00600` | `rfshset1tmg0` | `0xc03f0c34` | `0xc03f0c34` | `0xc03f092c` | `0xc03f0c34` |
| `0x00604` | `rfshset1tmg1` | `0x00e00070` | `0x00e00070` | `0x00a80054` | `0x00e00070` |
| `0x00608` | `rfshset1tmg2` | `0x06480000` | `0x06480000` | `0x05360000` | `0x06480000` |
| `0x00650` | `rfmset1tmg0` | `0x00000098` | `0x00000098` | `0x00000072` | `0x00000098` |
| `0x00800` | `zqset1tmg0` | `0x001804d7` | `0x001804d7` | `0x001203ab` | `0x001804d7` |
| `0x00804` | `zqset1tmg1` | `0x02800100` | `0x02800100` | `0x01e00100` | `0x02800100` |
| `0x00b04` | `derateval0` | `0x1024100a` | `0x1024100a` | `0x0c1b0c08` | `0x1024100a` |
| `0x00b08` | `derateval1` | `0x00000033` | `0x00000033` | `0x00000027` | `0x00000033` |
| `0x00d00` | `tmgcfg` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` |
| `0x00d04` | `ranktmg0` | `0x00000e12` | `0x00000e12` | `0x00000d10` | `0x00000e12` |
| `0x00d08` | `ranktmg1` | `0x0000150b` | `0x0000150b` | `0x0000120b` | `0x0000160a` |
| `0x10000` | `mstr0` | `0x03080008` | `0x01080008` | `0x03080008` | `0x03080008` |
| `0x10104` | `deratectl1` | `0x00000005` | `0x00000005` | `0x00000005` | `0x0000000f` |
| `0x10108` | `deratectl2` | `0x00000005` | `0x00000005` | `0x00000005` | `0x0000000f` |
| `0x10180` | `pwrctl` | `0x00000011` | `0x00000011` | `0x00000011` | `0x00000011` |
| `0x10200` | `rfshmod0` | `0x00000361` | `0x010003f3` | `0x00000361` | `0x00000361` |
| `0x10380` | `sched0` | `0x80012010` | `0x80012010` | `0x80012010` | `0x80012010` |
| `0x10508` | `dfiupd0` | `0xc0000000` | `0xc0000000` | `0xc0000000` | `0xc0000000` |
| `0x10510` | `dfimisc` | `0x00010005` | `0x00010005` | `0x00010005` | `0x00010005` |
| `0x10518` | `dfiphymstr` | `0x70000000` | `0x70000000` | `0x70000000` | `0x70000000` |
| `0x10b84` | `opctrl1` | `0x00000003` | `0x00000003` | `0x00000003` | `0x00000003` |
| `0x10d00` | `inittmg0` | `0x00020002` | `0x00020002` | `0x00020002` | `0x00020002` |

### Additional DDRC registers programmed outside the timing table

During controller preparation and post-training activation, the host also writes:

| Byte offset | Register | Value(s) | Role |
|---:|---|---|---|
| `0x10b84` | `opctrl1` | `0x00000002` (prep), `0x00000000` (active) | Initial operational mode vs. normal operation |
| `0x10d00` | `inittmg0` | `0xc0020002` | Initialization timing control |
| `0x10180` | `pwrctl` | `0x00000800` / `0x00000811` (prep), `0x00000010` (active) | Power / self-refresh sequencing (`bit 11` releases PHY) |
| `0x10208` | `rfshctl0` | `0x00000001` (prep), `0x00000000` (active) | Disables auto-refresh during init/training, enables after |
| `0x10280` | `zqctl0` | `0x80000000` (prep), `0x00000000` (active) | Triggers ZQ calibration during init |
| `0x10288` | `zqctl2` | `0x00000001` (prep), `0x00000000` (active) | ZQ calibration control |
| `0x10c80` | `swctl` | `0` (unlock), `1` (apply) | Brackets quasi-static register writes |
| `0x10c84` | `swstat` | Poll `bit 0 == 1` | Confirms quasi-static update completion |
| `0x10510` | `dfimisc` | `0x00010014` (pre-PMU), `0x00010034` -> `0x00010015` (post-PMU) | DFI initialization and operational transitions |
| `0x20090`–`0x24090` | `pctrl0`–`pctrl4` | `1` | Enables host ports 0–4 after `stat[1:0] == 01` |

## DFI and power-state values are phase-dependent

**Interface.** Different snapshots contain different values for the same register. For example:

- `dfimisc` has an extracted timing-stage value of `0x00010005`, while the wider initialization/activation flow also uses values ending in `0x14`, `0x34`, and `0x15`.
- `opctrl1` is `3` in the extracted timing output; another preparation description records `2` at a different point.
- `pwrctl` is `0x11` in the extracted timing output, while activation uses the documented `0x811`/`0x800` sequence, bit 11 handling, and final `0x10` self-refresh exit.

**Inference.** These are stage differences, not evidence that one arbitrary value can replace the whole sequence. A reimplementation should place writes in their original phase and perform the associated polling. **Unknown.** The available reconciled sources do not establish a complete cycle-accurate ordering for every intermediate write, so the numeric examples must not be collapsed into a single flat register image.

## Address map

**Interface.** Four system-address regions are programmed with `SARBASE0..3` and `SARSIZE0..3` at `0x200c0`–`0x200dc`. They are aligned to 256 MiB and based at 2 GiB. `ADDRMAP1` at `0x30004` differs between single- and dual-rank configurations, and `ADDRMAP3..12` at `0x3000c`–`0x30030` complete the implemented map.

**Unknown.** The sources do not completely specify channel interleaving or every row/bank/column bit. Capacity and address mapping must therefore be derived and validated for the actual part, not inferred only from the rank field.

## Five host ports

Each DDRC instance has five port-control registers in the analyzed flow:

| Port | Byte offset | Final enable value |
|---:|---:|---:|
| 0 | `0x20090` | `1` |
| 1 | `0x21090` | `1` |
| 2 | `0x22090` | `1` |
| 3 | `0x23090` | `1` |
| 4 | `0x24090` | `1` |

**Interface.** These writes occur after the controller reaches its operational state and leaves self-refresh. Enabling a port earlier would expose traffic to an uninitialized or training-owned interface.

**Unknown.** The evidence establishes five software-controlled ports and their enable sequence, but does not identify each port's SoC master, QoS capabilities, or bandwidth allocation. Those should not be inferred from port number alone.
