<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Quickboot](11-quickboot.md) | [Index](README.md#reading-order-and-contents) | [Next: Firmware Behavioral Requirements →](13-firmware-behavioral-requirements.md)

# ARC execution environment

## Scope and evidence

The LPDDR5 training and quickboot programs execute on the PHY's embedded **ARCv2 EM PMU**, not on the host RISC-V CPU. This document specifies the hardware execution environment, memory map, register file constraints, peripheral CSR windows, and ABI expectations required to build and run independent PMU firmware.

Facts confirmed by hardware probing and the host/PMU interface contract are marked **observed**. Design guidance is marked **inferred**, and unresolved behavior is listed separately. No vendor implementation code or internal software layout is reproduced here.

## Observed: processor and image model

- The core is a 32-bit, little-endian **ARCv2 EM** processor supporting 16-bit and 32-bit instruction encodings, two zero-overhead loop registers (`lp_count` with start/end bounds), and decrement-and-branch (`dbnz`).
- It has no MMU or FPU in this environment; all addresses are physical in the PMU's local address space.
- The host loads the firmware image into PMU IMEM at PHY CSR index `0x50000`, transferred as 16-bit halfwords.
- Hardware probing demonstrated a complete **64 KiB IMEM** window: instructions at offsets up to `0xfff8` executed successfully. Note that in the quickboot loader path, the host decompression buffer is bounded to `0x6800` bytes (26,624 bytes), whereas the full-training loader path accommodates up to 64 KiB.
- The host releases the PMU using the `MICRO_RESET = 9, 1, 0` sequence (`0xd0099`) and communicates exclusively through shared DMEM and the mailbox registers. There is no register-based parameter passing between host and PMU at reset.
- **Host re-access to DMEM between PMU runs (without reloading IMEM)**: After a PMU execution completes, the host normally hands the bus mux to the ARC core (`MICRO_CONT_MUX_SEL = 1` at `0xd0000`) and may gate PMU clocks (`UCCLK_HCLK_ENABLES = 0x0002` at `0xc0080`). Simply pulsing `MICRO_RESET = 1 / 9` at `0xd0099` without restoring mux ownership and clocks leaves APB accesses to `0x58000+` isolated (reads return `0`, writes are dropped). To patch the message block and re-run without reloading IMEM:
  1. Write `CSR[0x0d0099] = 0x0001` (`MICRO_RESET` stall), **`CSR[0x0d0000] = 0x0000`** (`MICRO_CONT_MUX_SEL = 0`, APB owns DMEM/CSRs), and **`CSR[0x0c0080] = 0x0007`** (`UCCLK_HCLK_ENABLES`, un-gate `uCclk`/`Hclk`).
  2. Update the target message-block words at `0x58000+` (and restore clean initial values in `0x58200+` if the prior run mutated runtime globals).
  3. Write **`CSR[0x0d0000] = 0x0001`** (`MICRO_CONT_MUX_SEL = 1`, ARC owns DMEM/CSRs) followed by `CSR[0x0d0099] = 9, 1, 0`.
- The initial stack top is **`0x80010000`**, and the stack grows downward.

The host's PHY CSR indexing convention is distinct from ARC addresses: a host `phy_reg[index]` accesses a 16-bit CSR through a 32-bit bus word at `DPHY_BASE + 4 * index`. ARC DMEM addresses (`0x8000xxxx`) and peripheral addresses (`0x90xxxxxx`) are in the PMU's native byte-addressed space, not host CSR indices.

## Observed: effective entry point is `0x1fc`

The image begins with a reset branch at offset `0x0`, but the effective startup entry is **`0x1fc`**. The reset vector at `0x0` branches to `0x1fc`; normal execution must not be linked as a flat function body starting directly at `0x0` without preserving the `0x1fc` entry layout.

Startup code at `0x1fc` must initialize core state, set `sp = 0x80010000`, locate the host-provided parameter area at `0x80000400`, and then enter the training flow. Hardware experiments confirmed that execution reaching `0x1fc` (and optionally continuing at `0x200`) behaves deterministically, while pointing the reset branch at `0x0` to an arbitrary other offset did not complete normal initialization. An independent image should therefore keep a reset branch at `0x0` that targets startup code at `0x1fc`.

## Observed: reduced register file (`-mrf16`)

This K3 PMU core implements the **16-entry reduced ARCv2 register file** (`RF16`) and **does not implement general-purpose registers `r16` through `r25`**. The architectural register set available to firmware is:

- `r0`–`r15` (general-purpose registers, with `r0`–`r7` used for argument passing and `r0` for return values);
- `gp` (`r26`, small-data / global pointer);
- `fp` (`r27`, frame pointer);
- `sp` (`r28`, stack pointer);
- `r30`; and
- `blink` (`r31`, branch link return address).

Executing any instruction that reads or writes `r16`–`r25` (for example a compiler-generated callee-save push such as `st.aw r25, [sp, -4]`) immediately faults or hangs the PMU core.

Consequently, **`-mrf16` is mandatory on every compiler invocation and on the final linker invocation**:

- When Link-Time Optimization (`-flto`) is enabled, the linker invokes `LTRANS` to regenerate machine code across translation units. Passing `-mrf16` only during compilation (`-c`) is insufficient: if `-mrf16` is omitted on the link command line, `LTRANS` reverts to the 32-register ABI and emits `r16`–`r25` instructions.
- Every linked ELF image must be verified with `arc-elf32-objdump -d` to confirm zero instructions reference `r16`–`r25`.

## Observed: `phypara` base (`0x80000400`) vs. firmware `.bss` (`0x80004000`)

Before releasing the PMU, the host loader populates the message block (`0x80000000`–`0x8000013f`, host DPHY `0x58000`–`0x5809f`) and pre-training PHY parameter structures starting at **`0x80000400`** (host DPHY `0x58200`). Every host DPHY index in `0x58000`–`0x5ffff` maps into ARC DMEM via:

```text
ARC_DMEM_address = 0x80000000 + 2 * (host_dphy_index - 0x58000)
```

This low DMEM region is **live input and runtime state**, not zero-initialized scratch memory:

| Region / Purpose | Host DPHY Index | ARC Address | Contract |
|---|---|---|---|
| Message block (`0x140` bytes active, `0x400` reserved) | `0x58000`–`0x5809f` (`0x581ff`) | `0x80000000`–`0x8000013f` (`0x800003ff`) | Host input parameters and PMU output fields |
| Host-provided `phypara` / parameter area | `0x58200`–`0x584d1` (full) / `0x58200`–`0x59631` (QB) | **`0x80000400`–`0x800009a3`** (full) / `0x80000400`–`0x80002c63` (QB) | Live configuration written by SPL plus PMU runtime state; **must not be zeroed** by startup |
| Quickboot extra DMEM parameter table | `0x5aa00`–`0x5acc5` | `0x80005400`–`0x8000598b` | 1,420-byte (710-halfword, `491` non-zero) quickboot table; quickboot sets `gp = 0x80005400` (`[gp, 0x78] = 0x80000400` points to `phypara`) |
| Independent firmware `.bss` / globals (recommended placement) | `0x5a000`+ | **`0x80004000`+** | Safe region for zero-initialized compiler `.bss` and static state |
| Transient firmware/host scratch | `0x5f000`–`0x5f07f` | `0x8000e000`–`0x8000e0ff` | Read and cleared by host SPL; not persistent across phases |
| Initial stack top | `0x60000` | `0x80010000` | Grows downward |

Within `0x80000400`–`0x80000e4e` (relative to `gp = 0x80000400`, where `0x80000400`–`0x800009a3` / host DPHY `0x58200`–`0x584d1` is pre-loaded by the host SPL pre-training table), the PMU firmware reads and writes the following specific configuration tables, canaries, and runtime state offsets during `DevInit` and subsequent stages:

| ARC DMEM Address | Host DPHY Index | Width | Pre-loaded value | Role during PMU execution |
|---:|---:|---|---|---|
| `0x80000403` | `0x58201` (hi) | 8-bit | `0x00` (set to `8`–`10`) | Hardware PHY IP version selector (`[gp, 3]`), populated at startup by decoding `CSR[0x0c00ee] & 0x0fff` (`ARC 0x901801dc`: `0x0101..0x0105` → `2..6`, `0x0110` → `7`, `0x0200` → `8`, `0x0240` → `9`, else `10`); because `byte[0x80000403]` is `8`–`10` (`!= 0`), startup always selects 16-bit `CSR[0x09002e]`/`0x09002f` for the canary check and enables the `CSR[0x090808]`/`0x090813` divider writes in `Step 6.1` |
| `0x80000404` | `0x58202` (lo) | 8-bit | `0x00` | RTC clock shift (`1 & ~(CSR[0x0c0080] >> 2)`: `1` when `0x0c0080 == 3`, `0` when `0x0c0080 == 7`) used by `delay_dfi_clks(n)` to compute `rtc_ticks = (n + 1) >> shift` |
| `0x80000408` | `0x58204` | 16-bit | `0x0500` | Pre-loaded configuration word (`[gp, 8]`) |
| `0x8000040c`, `0x80000410` | `0x58206`, `0x58208` | 2 × 32-bit | `0x00000000` (set to `4`, `2`) | Total active rank count (`0x8000040c = u7 * u0 = 4`, `[gp, 12]`, halved to `2` if `CsPresentChB == 0`) and channel multiplier (`0x80000410 = u7 = 2`, `[gp, 16]`), computed by the PMU in Phase 2 |
| `0x8000041c` | `0x5820e` | 32-bit | `0x00000000` | P-state / channel offset (`[gp, 28]`) |
| **`0x80000424`, `0x80000426`** | **`0x58212`, `0x58213`** | **2 × 16-bit** | **`0x6745`, `0x00f2`** | **Mandatory 32-bit PHY signature canary comparison words (`0x00f26745`, `[gp, 36]`/`[gp, 38]`, NOT an ODT parameter)**: verified at PMU startup against 16-bit `CSR[0x09002e]`/`0x09002f` (`0x9012005c`/`0x9012005e`); if missing or zeroed (or if `byte[0x80000403]` was left at `0` so 9-bit `CSR[0x090029]` truncated `0x6745` to `0x0145`), the PMU aborts immediately with fatal `0x000000ff` |
| `0x8000042a` | `0x58215` | 16-bit | `0x0000` | Saved pre-training value of `CSR[0x020011]` (`0x006f`) before masking bits `[6:5]` to `0x000f` |
| `0x8000042c` | `0x58216` | 16-bit | `0x0000` | Stage execution progress bitmask (`\|= 1`, `\|= 2`, `\|= 4`, `\|= 8`) |
| `0x80000430`–`0x80000437` | `0x58218`–`0x5821b` | 4 × 16-bit | `0x0000` | Saved per-rank values of `CSR[0x01<r>0a3]` (`0x0b33` for `r = 0..3`, saved in `Step 3.3` and restored in `Step 6.3`) |
| `0x80000438`, `0x8000043a` | `0x5821c`, `0x5821d` | 16-bit | `0x0000`, `0x0004` | Saved state word (`0x438`) and pre-loaded loop divisor (`0x43a = 4`) |
| `0x80000458`, `0x8000045a`, `0x8000045e` | `0x5822c`, `0x5822d`, `0x5822f` | 16-bit | `0x0000`, `0x0004`, `0x0000` | ACSM halfword write cursor (`0x80000458` and `0x8000045e`) and NOP cycle-count divisor (`0x8000045a = 4`; a 32-bit store to `0x80000458` clobbers `0x8000045a` and breaks ACSM delay loops) |
| `0x8000045b`–`0x80000495` | `0x5822d`–`0x5824a` | 8/16-bit | Pre-loaded tables | Per-channel / per-rank mode scratch and pre-loaded lane/byte lookup tables (`0x8000047e` supplies the initial `CSR[0x020023]` value during `CSR[0x020031] = 1` continuous-loop ACSM enable) |
| **`0x800004cc`–`0x800004dd`** | **`0x58266`–`0x5826e`** | **3 × 6 bytes** | **3 CSR descriptors** | Three 6-byte `[uint32_t csr_idx, uint16_t val]` descriptors applied at `Step 3.1`: **`CSR[0x02006e] = 0x0001`**, **`CSR[0x070060] = 0x0000`**, **`CSR[0x070011] = 0x0000`** |
| `0x800004de`–`0x800009a3` | `0x5826f`–`0x584d1` | 8/16-bit | Pre-loaded tables | Sub-table descriptors (`0x4de`, `0x4f4`) and frequency-indexed PHY timing/configuration tables (`0x510`–`0x9a3`) |
| `0x800009a8`+ | `0x584d4`+ | 16-bit array | `0x0000` | ACSM instruction staging buffer in DMEM (copied into `0x90082000` ACSM SRAM) |
| `0x80000a80` | `0x58540` (lo) | 8-bit | `0x00` (set to `1`) | Cycle-multiplier enable flag set to `1` at PMU startup; scales `min_clks` in `delay_ps_or_clks(ps, min_clks)` by `byte[0x80000b68] * byte[0x80000b69]` (`32×`) |
| `0x80000b68`, `0x80000b69` | `0x585b4` | 2 × 8-bit | `0x00`, `0x00` (set to `8`, `4`) | Frequency-derived multiplier `byte[0x80000b68] = (DRAMFreq + DfiFreqRatio * 220 - 1) / (DfiFreqRatio * 220)` (`8` at 6400/4, written as `0x0108` to `CSR[0x020042]`) and constant `byte[0x80000b69] = 4` |
| `0x80000b6a`–`0x80000b71` | `0x585b5`–`0x585b8` | 8 × 8-bit | `0x00` | Active lane bounds (`0x80000b6c..0x80000b6d = 0..8`) and rank bounds (`nA = (EnabledDQsChA + 7) >> 3`: `0x80000b6e = 0`, `0x80000b6f = nA - 1 = 1`; when `CsPresentChB != 0`, `u = word[0x8000040c] >> 1 = 2`, `0x80000b70 = u = 2`, `0x80000b71 = ((EnabledDQsChB + 7) >> 3) + u - 1 = 3`) |
| `0x80000e4c`, `0x80000e4e` | `0x58726`, `0x58727` | 2 × 16-bit | `0x0000` | Saved state words during post-`DevInit` stage transitions |

If a C startup routine places `.bss` at `0x80000400` and zeros it on boot, it wipes the SPL-supplied parameter tables before training begins, causing immediate training failure. Furthermore, if the compiler uses the hardware `gp` register (`r26`) for its own small-data section (`.sdata`/`.sbss`), that compiler `gp` must point to the firmware's own data region rather than colliding with the `0x80000400` parameter table.

## Observed: access widths and unaligned table strides

The PMU's DMEM (`0x8000xxxx`) and PHY/peripheral CSR space (`0x90xxxxxx`, including mailbox registers `0x90180008`, `0x90180064`, `0x90180066`, and `0x90180068`) are **16-bit interfaces** on the ARC side.

### ARC CSR address formation

When the PMU accesses a PHY CSR index from the ARC core, the 16-bit halfword address in the `0x90xxxxxx` space is formed by doubling the 16-bit CSR index (combined with any block/channel/rank selector bits via bitwise OR):

```text
ARC_CSR_address = 0x90000000 | (2 * (csr_index | block_or_rank_bits))
```

In addition, three dedicated hardware descriptor/command windows are exposed in the `0x90xxxxxx` space:

- `0x90000000 + 2 * index`: read-side descriptor transfer window;
- `0x9001e000 + 2 * index`: write-side descriptor transfer window;
- `0x90082000 + 2 * index`: packed command-sequence halfword window.

### Hardware PHY CSR windows on the ARC bus (`0x90xxxxxx`)

| ARC Address / Range | Hardware Role |
|---|---|
| `0x90000000 + 2 * idx`, `0x9001e000 + 2 * idx` | Read-side and write-side PHY descriptor transfer windows |
| `0x90020020`/`24`/`32`/`38`/`d0`/`d4`/`d8`, `0x90020100` | Per-rank/lane trained delay and result registers |
| `0x90020146`, `0x90020158`, `0x9002016e`, `0x9002017a`, `0x900201aa`, `0x900201b2` | Per-lane training control and read/write eye CSRs (stride scaled by lane/rank selector bits) |
| `0x9003e000`, `0x9003e012`, `0x9003e014`, `0x9003e034`, `0x9003e126`, `0x9003e172` | Hardware training engine control (start/stop, mask, mode, status, reset) |
| `0x90040034`, `0x90040040`/`42`/`44`/`46`/`48`/`4e`/`62` | PHY engine control/handshake (`0x90040040`/`42` start/config, poll `0x90040062`), P-state parameters (`+ 2 * pstate`), and short-delay read (`0x90040048`) |
| `0x90040084`, `0x90040094`, `0x900400a6`, `0x900400c0`, `0x900400e0`, `0x900400e2` | PHY configuration and mode/timing control registers |
| `0x900400e4`/`e6`/`f4`/`fc` | Active rank / P-state / channel selection |
| `0x90040120`, `0x90040146`, `0x90040154`, `0x900401dc`, `0x9004030e` | Initialization and control pulse registers (`0x90040146` uses `1 -> wait 0x20 -> 0` pulse) |
| `0x90040238` | Terminal status pulse register (`1 -> 0` on normal completion `0x07`; `4 -> 0` on fatal error `0xff`) |
| `0x9005e012`, `0x9005e0a4`, `0x9005e140`, `0x9005e14a` | Training finalization, synchronization, and delayed pulse (`0x9005e140` uses `1 -> wait 0x20 -> 0` pulse) |
| `0x90060002`/`04`, `0x90061002`, `0x90061e02`, `0x9007e002`/`04`/`34` | Per-lane CA/CS and training result/control registers |
| `0x90082000 + 2 * idx` | Command sequence serialization window |
| `0x90180008`, `0x90180064`, `0x90180066`, `0x90180068` | PMU mailbox status (`bit 0`), low 16-bit data/major code, doorbell/ACK, and high 16-bit data |

### Access width rules

- Standard CSR and DMEM word accesses must use **16-bit halfword loads and stores (`ldh`/`sth` / `volatile uint16_t`)**. Accessing `0x8000xxxx` or `0x90xxxxxx` as 32-bit (`ld`/`st` / `uint32_t`) reads zero, corrupts adjacent halfwords, or stalls the internal peripheral bus.
- **Mandatory `volatile` qualification**: Every pointer or cast targeting `0x8000xxxx` (DMEM) or `0x90xxxxxx` (PHY/mailbox CSRs) must be declared `volatile` (`volatile uint16_t *`, `volatile uint8_t *`). Without `volatile`, compiler dead-store elimination and loop-invariant load hoisting silently remove hardware polling loops (such as `0x90180008` mailbox handshakes or `0x90040062` engine completion polls) and intermediate register sequences (such as the `0x020011` `& 0xff9f` save/restore around clock calibration).
- Individual 8-bit message-block fields are accessed via byte loads/stores (`ldb`/`stb` / `volatile uint8_t`) at their exact little-endian byte offset.
- Packed parameter tables whose element stride is **not a multiple of 4 bytes** (for example, 6-byte descriptor records) must be read **byte-by-byte (`ldb` / `uint8_t *`)** and assembled in registers. Performing a 32-bit load at an unaligned address (`base + 6`) causes an unaligned memory access fault/hang on the PMU core.

## Observed: auxiliary registers (`RTC` timer and exceptions)

The PMU implements ARCv2 auxiliary registers accessed exclusively via **`lr` (load from auxiliary register) and `sr` (store to auxiliary register)** instructions—never via memory-mapped loads/stores:

| AUX Address | Register Role | Usage |
|---:|---|---|
| `0x103` | RTC control | `lr`/`sr`; enables/resets the free-running 64-bit hardware counter (writes `2`, then `1`) |
| `0x104` | RTC count low (32-bit) | `lr`; lower 32 bits of the free-running timer |
| `0x105` | RTC count high (32-bit) | `lr`; upper 32 bits of the free-running timer |
| `0x400` | `ERET` (Exception Return) | `lr`; faulting PC on core exception |
| `0x403` | `ECR` (Exception Cause) | `lr`; architectural exception cause code |

### Hardware timer polling

Hardware delays use the 64-bit RTC counter across AUX `0x103`–`0x105` (running at **`400 MHz` = `2.5 ns/tick`**) through two entry points (see [07-training-behavior.md](07-training-behavior.md)):

1. **`delay_dfi_clks(n)`**: takes `n` in DFI clock cycles (`800 MHz` = `1.25 ns/cycle` at 6400 MT/s with `DfiFreqRatio = 4`) and computes `rtc_ticks = (n + 1) >> byte[0x80000404]` (`byte[0x80000404] = 1 & ~(CSR[0x0c0080] >> 2)` = `1` when `0x0c0080 == 3`).
   - When `rtc_ticks < 16` (`0x10`), performs a single dummy 16-bit read from `0x90040048` (`CSR[0x020024]`) and returns immediately without touching the RTC.
   - When `rtc_ticks >= 16`, checks `AUX 0x103` (if bit 0 is `0`, writes `2` then `1` to `0x103`), reads `0x104` (low) and `0x105` (high), and polls until **`rtc_ticks` raw 400 MHz RTC ticks** (`1` tick per scaled unit) elapse while accounting for 32-bit rollover from `0x104` into `0x105`. Modeling `0x103`–`0x105` as memory-mapped addresses causes delay loops to hang indefinitely.
2. **`delay_ps_or_clks(ps, min_clks)`**: converts `ps` (**time in picoseconds**) into DFI clock cycles `dfi_clks = (((ps * DRAMFreq) / 2,000,000) + DfiFreqRatio + 1) / DfiFreqRatio`, computes the cycle floor `c = min_clks * byte[0x80000b68] * byte[0x80000b69]` (`32 * min_clks` when `byte[0x80000a80] == 1`, or `min_clks + 1` otherwise), and calls `delay_dfi_clks(max(dfi_clks, c))`.

## Observed: arithmetic instructions and `libgcc` constraints

Several software arithmetic helpers in standard full-register `arc-elf32` `libgcc` fail or hang on the K3 PMU core:

1. **64-bit multiplication (`__muldi3` / `__umulsi3_highpart`)**: Standard `libgcc` implementations using `rlc`/`adc` sequences hang the PMU core when computing 64-bit products or upper 32-bit products (`(uint64_t)a * b >> 32`). The hardware implements the unsigned 32×32 high-part multiply instruction **`mpymu`** (and `mpy`/`mpyu`). Firmware must pass **`-mmpy-option=wlh5`** (which enables `mpy`/`mpyu`/`mpymu`/`mac` generation directly in GCC) and provide a global `__muldi3` override built around `mpymu`/`mpyu`.
2. **32-bit integer division (`__udivsi3` / `__divsi3` / `__udivmodsi4`)**: Standard `libgcc` division hangs on the small-divisor fast path (when `divisor >= dividend / 2`, such as `5 / 4`). The hardware implements **`divu`** (and `divs`/`remu`/`rems`). Firmware must pass **`-mdiv-rem`** (so GCC emits hardware `divu`/`divs`/`remu` directly) and override `__udivsi3` and `__divsi3` globally using hardware division.
3. **Small-data pointer (`-mno-sdata`)**: Passing **`-mno-sdata`** prevents GCC from using `gp` (`r26`) for compiler-generated `.sdata`/`.sbss` offsets, avoiding any accidental collision with the `0x80000400` parameter table.
4. **Global override preservation under LTO**: When compiling firmware with `-flto`, the translation unit providing global `__muldi3`, `__udivsi3`, and `__divsi3` replacements must be compiled **without `-flto`** (while still passing `-mrf16`) so that `LTRANS` does not discard the custom symbols and fall back to the broken `libgcc` archive implementations.

## Observed: ARCv2 instruction semantics (`dbnz.d`, `j_s.d`, `add1`/`add2`/`add3`)

When writing assembly startup or peripheral loops for the ARCv2 EM core:

- **Branch/loop delay slots (`.d`)**: An instruction in the delay slot of `dbnz.d`, `b.d`, or `j_s.d [blink]` executes unconditionally on the final branch/iteration as well.
- **Scaled arithmetic (`add1`/`add2`/`add3`, `sub1`/`sub2`/`sub3`)**: These architectural instructions shift their second source operand by 1, 2, or 3 bits (multiplying by **2, 4, or 8**) before adding/subtracting.

## Summary of host ↔ PMU runtime contract

### What the PMU expects from the host

1. PHY ownership transferred to PMU (`MICRO_CONT_MUX_SEL = 1` at `0xd0000`).
2. Complete firmware loaded into IMEM (`0x50000`) and any required initial DMEM loaded.
3. Validated 320-byte message block at `0x58000` (`0x80000000`) and `phypara` tables at `0x80000400`.
4. Unused DMEM cleared (`0x584d2`–`0x5ffff` in full training).
5. No concurrent host CSR accesses while the PMU owns the mux.
6. Continuous mailbox consumption and acknowledgement (`0xd0004`, `0xd0031`, `0xd0032`, `0xd0034`).

### What the host expects from the PMU

1. PHY hardware version/signature verification at startup, failing early on mismatch.
2. Progress notifications and optional streaming frames (`0x08`) over the 16-bit mailbox.
3. Trained outputs (CDDs, VREFs, margins, DFE/DCA, quickboot PLL words) written to the message block and PHY/ACSM memories before completion.
4. Terminal major message `0x07` on normal completion or `0x000000ff` on fatal error, followed by a clean halt.
