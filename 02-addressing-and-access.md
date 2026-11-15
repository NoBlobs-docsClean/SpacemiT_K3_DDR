<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: System Architecture](01-system-architecture.md) | [Index](README.md#reading-order-and-contents) | [Next: DDRC, DPLL & Host Ports →](03-ddrc-and-clock.md)

# Addressing and access rules

## Evidence and notation

**Interface** identifies loader/device-tree behavior, **hardware / ABI** identifies PMU processor, bus-width, and memory-map constraints, and **observation** identifies captured board or saved-image data. Conclusions without independent hardware confirmation are marked **inference**; unresolved behavior is **unknown**.

There are three distinct address views:

1. host DDRC/CFG byte offsets;
2. host DPHY 32-bit word indices;
3. ARC virtual addresses for DMEM and PMU-visible peripherals.

They are not interchangeable.

## Host physical bases

**Interface.** The board's DDRC addresses are confirmed independently by the SPL DTB and the DTB embedded in the reference FSBL:

| Resource | Instance 0 | Instance 1 | Address rule |
|---|---:|---:|---|
| DDRC | `0xcb000000` | `0xcc000000` | 4 MiB declared region |
| CFG/DPLL | `0xcb600000` | `0xcc600000` | `DDRC_BASE + 0x600000` |
| DPHY | `0xcb800000` | `0xcc800000` | `DDRC_BASE + 0x800000` |

The generic `0xc0000000` node is overridden for this board. The older `0xc0800000` DPHY derivation is likewise superseded by the confirmed `0xcb800000` base.

## DDRC and CFG: byte offsets

**Interface.** DDRC and CFG constants are byte offsets, even when implementation code converts them to a 32-bit array subscript:

```text
DDRC address = DDRC_BASE + byte_offset
CFG address  = DDRC_BASE + 0x600000 + byte_offset
```

Thus DDRC `mstr0` at byte offset `0x10000` is at `0xcb010000` for instance 0 and `0xcc010000` for instance 1. Host accesses to these registers are 32-bit.

## DPHY: host word indices

**Interface.** A DPHY constant `i` is an index into a 32-bit host pointer, not a byte offset:

```text
DPHY address = DPHY_BASE + 4 * i
```

A normal host transaction is 32 bits wide and addresses one 16-bit PHY CSR value in that word. Tables store 16-bit values even though the host pointer and bus transaction are 32-bit. This is a host-window convention; it does not make the PMU's own peripheral accesses 32-bit.

Examples:

| Host index | Purpose | Instance 0 address | Instance 1 address |
|---:|---|---:|---:|
| `0x41000` | ACSM SRAM window | `0xcb940000` | `0xcc940000` |
| `0x50000` | PMU IMEM | `0xcbc00000` | `0xccc00000` |
| `0x58000` | DMEM/message block | `0xcbe00000` | `0xcce00000` |
| `0x58009` | `HdtCtrl` | `0xcbe00024` | `0xcce00024` |
| `0xd0000` | Host/PMU control mux | `0xcbc40000` | `0xccc40000` |
| `0xd0099` | `MICRO_RESET` | `0xcbc40264` | `0xccc40264` |

**Unknown.** The source interface establishes the low 16-bit CSR payload convention, but it is not a complete electrical specification for upper-lane behavior or for arbitrary unaligned/16-bit host bus writes. Reimplementations should preserve the demonstrated 32-bit host access pattern rather than assume other host widths are equivalent.

## DPHY memory windows and ARC DMEM mapping

Each host DPHY index in the DMEM window (`0x58000`–`0x5ffff`) addresses one 16-bit halfword. On the ARC PMU side, DMEM is byte-addressed starting at `0x80000000`:

```text
ARC_DMEM_address = 0x80000000 + 2 * (host_dphy_index - 0x58000)
Host_MMIO_address = DPHY_BASE + 4 * host_dphy_index
```

| Start index (Host DPHY) | ARC Address | Window | Documented use |
|---:|---:|---|---|
| `0x41000` | *(via PHY CSRs)* | ACSM SRAM | 4096 halfwords (`0x2000` bytes) saved and restored |
| `0x50000` | `0x00000000` | IMEM | Normal or quickboot PMU program (up to 64 KiB), transferred as halfwords |
| `0x58000` | `0x80000000` | DMEM message block | `0xa0` useful message halfwords (`0x140` bytes); `0x200` halfwords (`0x400` bytes) reserved |
| `0x580a0` | `0x80000140` | Message block padding | Reserved tail of the `0x200`-halfword message reservation (through `0x800003ff`) |
| `0x58200` | `0x80000400` | `phypara` / parameter area | Full training: pre-training DMEM tables (`0x58200`–`0x584d1`). Quickboot: 5170 serialized `phypara` halfwords |
| `0x584d2` | `0x800009a4` | Full-training DMEM clear start | Cleared by host through `0x5ffff` (`0x8000ffff`) before full training; available for firmware `.bss` and stack |
| `0x59622` | `0x80002c44` | Quickboot additional DMEM | `0x58000 + 0x200 + 5170`; holds the 1,420-byte (710-halfword) quickboot DMEM parameter table |
| `0x5aa00` | `0x80005400` | Quickboot partial clear limit | The host zeroes the range below this index before loading the extra DMEM image, then zeroes the remainder through `0x5ffff` after loading it |
| `0x60000` | `0x80010000` | DMEM limit / stack top | Upper limit of the 64 KiB DMEM window; initial ARC stack pointer (`sp = 0x80010000`, grows downward) |
| `0xa0000` | *(via PHY CSRs)* | P-state SRAM | 16384-halfword resource; restoration is not active in the current flow |

The 5170-halfword PHY image consists of 69 ordered contiguous or sparse segments. Its bases and relative offsets are host DPHY indices (for the three sequential segments, the relative range starts at a non-zero `start_offset`: `46` for segment 48 at `0x90000`, `292` for segment 54 at `0x20000`, and `395` for segment 55 at `0x20000`, making all 5170 CSR indices unique). Segment 48 is the largest single contiguous run: 1536 halfwords at `0x9002e`–`0x9062d` (base `0x90000`, `start_offset = 46`).

## ARC-side addresses and hardware access widths

**Hardware / ABI.** The PMU sees its own little-endian ARC address space. Do not add these addresses to a host DPHY base.

For any host DPHY CSR index `i`, the corresponding 16-bit register address in the PMU's `0x90xxxxxx` peripheral window is:

```text
Host MMIO address = DPHY_BASE + 4 * i
ARC CSR address   = 0x90000000 + 2 * i
```

This exact 2× index-to-ARC mapping is confirmed across multiple shared registers:

| Host DPHY index `i` | Host MMIO (`DPHY0 + 4*i`) | ARC CSR (`0x90000000 + 2*i`) | Hardware role |
|---:|---:|---:|---|
| `0x1004e` | `0xcb840138` | `0x9002009c` | Rank 0, Lane 0 trained read eye center 1 |
| `0x1004f` | `0xcb84013c` | `0x9002009e` | Rank 0, Lane 0 trained read eye center 2 |
| `0x100a3` | `0xcb84028c` | `0x90020146` | Rank 0, Lane 0 `DevInit` status/config (`0x0b33`) |
| `0x20012` | `0xcb880048` | `0x90040024` | `DevInit` calibrated clock-period bucket register |
| `0xc0000` | `0xcbb00000` | `0x90180000` | Mailbox register block base |
| `0xc0032` | `0xcbb000c8` | `0x90180064` | `UCT_WRITE_ONLY` (mailbox low 16-bit data / major code) |

| ARC range/address | Purpose | Hardware / ABI access rule |
|---|---|---|
| `0x00000000`–`0x0000ffff` | PMU IMEM (64 KiB) | Reset vector at `0x0` branches to effective startup at `0x1fc` |
| `0x80000000`–`0x8000013f` | Message block (320 bytes) | 16-bit `ldh`/`sth` for halfword fields; 8-bit `ldb`/`stb` for single-byte fields |
| `0x80000400` (`0x58200`) | Host-supplied `phypara` / parameter base | Live input state populated by host SPL; must not be zeroed by firmware `.bss` startup |
| `0x800009a4`–`0x8000ffff` | Zeroed DMEM workspace (full training) | Cleared by host before PMU release; used for firmware `.bss`/globals (e.g. `0x80004000`) and downward stack |
| `0x8000e000`–`0x8000e0ff` | Transient firmware-to-host message scratch | Host may consume/clear this range; do not use for persistent state |
| `0x80010000` | Initial stack top | Stack grows downward |
| `0x90000000` | PHY CSR read descriptor window | 16-bit (`0x90000000 + 2 * index`) |
| `0x9001e000` | PHY CSR write descriptor window | 16-bit (`0x9001e000 + 2 * index`) |
| `0x90082000` | Packed command sequence window / ACSM (`0x41000`) | 16-bit (`0x90082000 + 2 * index`) |
| `0x90180008` | Mailbox status/ACK | 16-bit (`bit 0` = busy / ACK state) |
| `0x90180064` | Mailbox command or low halfword | 16-bit |
| `0x90180066` | Mailbox doorbell/ACK | 16-bit |
| `0x90180068` | Mailbox high halfword | 16-bit |
| `0x90xxxxxx` | PMU-visible PHY/training peripherals | 16-bit halfword accesses (`ARC_CSR = 0x90000000 \| (2 * csr_index)`) |

**Hardware / ABI.** Using a 32-bit load/store (`ld`/`st` / `uint32_t`) on a DMEM halfword field or `0x90xxxxxx` peripheral register violates the 16-bit peripheral interface and reads zero, corrupts the adjacent halfword, or stalls the core. Furthermore, all C pointers and casts targeting `0x90xxxxxx` or shared `0x8000xxxx` DMEM must be qualified **`volatile`** so the compiler never coalesces or elides multi-write hardware pulse sequences. Packed structures whose element stride is not divisible by four (such as 6-byte descriptor records) must be accessed bytewise (`ldb`/`uint8_t *`); a 32-bit unaligned access faults and hangs the ARCv2 EM core.

The ARC timer at AUX registers `0x103`, `0x104`, and `0x105` is accessed with ARC `sr`/`lr` instructions, not as DMEM or a `0x90xxxxxx` peripheral. Its low/high counter reads support firmware delays (with very short scaled delays `< 16` reading CSR `0x90040048`).

### DPHY CSR group, rank, lane, and offset decomposition

Within the DPHY CSR space, a host word index `i` decomposes into four bitfields:

```text
group  = i >> 16             (ARC base: 0x90000000 | (group << 17))
rank   = (i >> 12) & 0xF     (host stride 0x1000; ARC stride 0x2000 = rank << 13)
lane   = (i >>  8) & 0xF     (host stride 0x100;  ARC stride 0x200  = lane << 9; lanes 0..8 used in x16)
offset = i & 0xFF            (host stride 1;      ARC stride 2      = offset << 1)
```

Principal DPHY CSR groups:

| Group (`i >> 16`) | Host index base | ARC address base | Role |
|---:|---:|---:|---|
| `0x01` | `0x10000` | `0x90020000` | Per-rank (`0..3`) and per-lane (`0..8`) DQ timing, VREF, ODT, and eye-center CSRs |
| `0x02` | `0x20000` | `0x90040000` | PHY engine control, clock-period calibration (`DevInit`), and MemReset control |
| `0x03` | `0x30000` | `0x90060000` | Command/Address (CA) training (`LPCA`) and CA TX drive strength |
| `0x04` | `0x41000` | `0x90082000` | ACSM SRAM (4096 halfwords, 8192 bytes) |
| `0x05` | `0x50000` / `0x58000` | `0x900a0000` / `0x900b0000` | Host-side IMEM (`0x50000`) and DMEM (`0x58000`) windows |
| `0x07` | `0x70000` | `0x900e0000` | Maximum read latency (`MxRdLat`) registers |
| `0x09` | `0x90000` | `0x90120000` | PHY/P-state configuration block (includes 1536-entry sequential segment at `0x9002e`) |
| `0x0a` | `0xa0000` | `0x90140000` | P-state SRAM (16384 halfwords) |
| `0x0c`, `0x0d` | `0xc0000`, `0xd0000` | `0x90180000`, `0x901a0000` | Mailbox (`0x0c0004`, `0x0c0032`–`0x0c0034`), clock enables (`0x0c0080`), status (`0x0c0086`), mux (`0x0d0000`), and PMU reset (`0x0d0099`) |
| `0x11` | `0x110000` | `0x90220000` | Read-only/power-on default DQ mirror block (1,024 non-zero hardware power-on reset registers across 4 ranks × 9 lanes; never written by host tables or PMU firmware, and not part of `phypara`) |
| `0x12`, `0x13`, `0x19` | `0x120000`, `0x130000`, `0x190000` | `0x90240000`, `0x90260000`, `0x90320000` | Secondary/mirror PHY control, CA, and lanes 8–9 extension blocks |

Within the DQ block (`group = 0x01`, base `0x10000`), not every offset is replicated across all 9 lanes:

- **Per-lane offsets (replicated across lanes `0..8`, stride `0x100`)**: `0x10`–`0x13`, `0x1c`–`0x1f`, `0x24`–`0x27`, `0x4e`–`0x51` (`0x4e`/`0x4f` = trained read eye centers; `0x50`/`0x51` = seed/edge registers), `0x68`–`0x6f` (write DQ delays adjusted by `DCA`), `0xaa`, `0xb5`, `0xbe`.
- **Rank-global offsets (exist only at `lane = 0`, i.e. `0x100xx + rank * 0x1000`)**:
  - `0x40`, `0x42`, `0x43`, `0x44`, `0x45` — rank TX drive strength (`io_override` target; writing `0x10140` is invalid because lane > 0 copies do not exist);
  - `0x48`, `0x4a`, `0x4b`, `0x4c`, `0x4d` — rank RX ODT termination controls.

Many per-lane and per-rank PHY CSR accesses compute their effective address by combining the channel/rank selector (`rank << 12` in host index or `rank << 13` in ARC address), lane selector (`lane << 8` in host index or `lane << 9` in ARC address), and a family-specific aligned base constant using **bitwise OR** (`|`), never addition (`+`):

```text
host_index  = base_index | (rank << 12) | (lane << 8) | offset
arc_address = base_constant | (rank << 13) | (lane << 9) | (offset << 1)
```

Hardware CSR family base constants include `0x90020000`, `0x90020158`, `0x9002016e`, `0x9003e000`, `0x9003e126`, `0x90040000`, `0x90060002`, and `0x9007e002`. An implementation must preserve bitwise OR composition so that selector bits never carry into adjacent address fields.

## Host PMU control, MemReset, and auxiliary CSRs

In addition to the mailbox indices below, the host controls PMU execution, clocks, MemReset, and capture modes through the following DPHY word indices:

| Host DPHY index | Interface role | Observed values / behavior |
|---:|---|---|
| `0xc0032` | `UCT_WRITE_ONLY` | Defined in interface header (unused in main host polling loop) |
| `0xc0033` | `UCT_WRITE_PROT` | Defined in interface header (unused in main host polling loop) |
| `0xc0080` | `UCCLK_HCLK_ENABLES` | Clock enables (`1`, `2`, `3`, `7` depending on stage) |
| `0xc0086` | ECC control | Written `1` in quickboot to disable ECC |
| `0xd0000` | `MICRO_CONT_MUX_SEL` | `0` = host/APB owns PHY CSRs; `1` = PMU owns PHY CSRs |
| `0xd0003` | Training phase auxiliary | Written `0` at the end of the post-run training table |
| `0xd0030` | `DCT_WRITE_ONLY` | Defined in interface header (unused in main host polling loop) |
| `0xd0036` | APB access mode | Written `0` in quickboot (APB-only mode) |
| `0xd0099` | `MICRO_RESET` | `9 -> 1 -> 0` releases PMU execution; `1` stalls/resets PMU |
| `0xd00e7` | Quickboot finalization | Written `0x0600` at the end of quickboot |
| `0x20037` | ACSM capture toggle (`csrACSMWckWriteToggleDelayReserved[0]`) | Bit 6 set before ACSM collection and cleared after stopping PMU |
| `0x20060`, `0x20090` | Quickboot MemReset control | Used during the quickboot MemReset sequence |
| `0x200c9` | PLL lock status (`plllockstatus`) | Read as diagnostic data after fatal message `0xff` |
| `0x3f042`, `0x3f043` | Quickboot MemReset timing | Written `0x0f0f` during quickboot MemReset |
| `0x3f0a2`, `0x3f0a3` | Quickboot MemReset timing | Written `0x0f00` and `0x0800` respectively during quickboot MemReset |

## Mailbox views

The host and PMU use different addresses for the same protocol:

| Function | Host DPHY index | PMU ARC address |
|---|---:|---:|
| Status/busy (`UCT_SHADOW`) | `0xd0004` | `0x90180008` |
| Acknowledge/protection (`DCT_WRITE_PROT`) | `0xd0031` | `0x90180066` |
| Major message or streaming low halfword/count (`UCT_WRITE_ONLY_SHADOW`) | `0xd0032` | `0x90180064` |
| Streaming high halfword (`UCT_DATA_WRITE_ONLY_SHADOW`) | `0xd0034` | `0x90180068` |

**Interface + hardware.** The PMU waits for a free mailbox, writes message halves, and signals the doorbell. The host acknowledges by writing 0, waiting for the consumed state, and writing 1. Before the next item it waits for the free state. Major value `0x07` denotes completion, `0x08` introduces streaming data, and `0xff` enters the documented error diagnostic path. These polls have no timeout in the analyzed interface.

A streaming header contains an opcode and payload length followed by that many 32-bit words, transported as halfword pairs. **Unknown.** The format is established, but most opcode semantics are not.

## DDRC address mapping and ports

**Interface.** The DDRC address map is separate from its MMIO address:

- `SARBASE0..3` and `SARSIZE0..3`, at byte offsets `0x200c0`–`0x200dc`, describe four 256 MiB-aligned regions beginning at the 2 GiB DRAM base.
- `ADDRMAP1` at `0x30004` controls rank/CS mapping; the single-rank configuration uses the documented inactive value `0x3f`.
- `ADDRMAP3..12` occupy `0x3000c`–`0x30030`.
- Five ports are enabled through `pctrl` at `0x20090`, `0x21090`, `0x22090`, `0x23090`, and `0x24090`.

**Unknown.** The available evidence does not fully decode every bank, row, column, channel-interleave, or system-address bit. Do not derive a complete physical-to-DRAM mapping solely from the region registers.

## Access-safety conclusion

**Inference supported by observations.** Direct post-boot DPHY access is not a safe debugging shortcut. One older attempted address was also based on the wrong DPHY base, which invalidates any explanation that attributed the resulting stall solely to PMU power-down. Even with corrected bases, access should remain within the demonstrated SPL ownership and clock/mux phases; concurrent host access while the PMU owns the PHY is outside the contract.
