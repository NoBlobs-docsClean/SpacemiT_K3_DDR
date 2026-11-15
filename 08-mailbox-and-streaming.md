<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: PHY Training Behavior](07-training-behavior.md) | [Index](README.md#reading-order-and-contents) | [Next: Training Results →](09-training-results.md)

# PMU mailbox and streaming protocol

## Scope

The PMU mailbox is the control and debug channel used while the PHY PMU executes initialization/training. The host must service it continuously: the PMU waits for acknowledgements and can stall if the host stops consuming messages.

This document separates the **observed reference behavior** from **recommended behavior for a new implementation**, especially for fatal error `0xff` and return-value propagation.

## Register views

### Host PHY CSR indices

Host byte address is `DPHY_BASE + 4 * index`.

| Index | Name | Role |
|---:|---|---|
| `0xd0000` | `MICRO_CONT_MUX_SEL` | 1=PMU owns PHY; 0=host/APB access. |
| `0xd0004` | `UCT_SHADOW` | Bit 0 mailbox synchronization/status. |
| `0xd0031` | `DCT_WRITE_PROT` | Host acknowledgement sequencing. |
| `0xd0032` | `UCT_WRITE_ONLY_SHADOW` | Major code, streaming low halfword, or streaming count. |
| `0xd0034` | `UCT_DATA_WRITE_ONLY_SHADOW` | Streaming high halfword. |
| `0xd0099` | `MICRO_RESET` | Start with 9,1,0; stop/stall with 1. |
| `0x200c9` | PLL status resource | Read by the host after major `0xff` as diagnostic data (not accessed by the PMU firmware). |

### PMU ARC addresses

| Address | Host CSR Index | Role |
|---:|---:|---|
| `0x90180008` | `0x0c0004` | Mailbox status, bit 0 |
| `0x90180064` | `0x0c0032` | Major/command or low 16 bits of a data word |
| `0x90180066` | `0x0c0033` | Doorbell/acknowledgement control |
| `0x90180068` | `0x0c0034` | High 16 bits of a data word |

The host shadow registers and PMU registers are two views of the same producer/consumer mechanism, not independent mailboxes. All PMU accesses to these registers use 16-bit halfword loads and stores (`ldh`/`sth`).

## Single-item handshake

### Host-side consumption cycle

Before consuming an item, the host waits for `UCT_SHADOW.bit0 == 0`.

For a normal item, acknowledgement is exactly:

1. Write `0` to `DCT_WRITE_PROT` (`0xd0031`).
2. Wait until `UCT_SHADOW.bit0 == 1`.
3. Write `1` to `DCT_WRITE_PROT`.
4. Before reading the next item, wait until `UCT_SHADOW.bit0 == 0`.

### PMU-side transmission cycle

On the PMU side, every single-word transmission (a 16-bit major code, the `0x08` streaming introducer, the 32-bit streaming header `(id << 16) | len`, or one 32-bit streamed payload word split across `0x90180064` and `0x90180068`) follows this exact 5-step ordering:

1. Write the 16-bit low halfword (or major code) to `0x90180064` (`CSR[0x0c0032]`), and for a 32-bit streaming word also write the 16-bit high halfword to `0x90180068` (`CSR[0x0c0034]`).
2. Write `0` to the doorbell register `0x90180066` (`CSR[0x0c0033]`).
3. Poll status `0x90180008` (`CSR[0x0c0004]`) until `bit 0 == 0` (mailbox ready for new transfer).
4. Write `1` to the doorbell register `0x90180066` (`CSR[0x0c0033]`) to signal the host that new data is valid.
5. Poll status `0x90180008` (`CSR[0x0c0004]`) until `bit 0 == 1` (host has acknowledged consumption).

> **Important doorbell ordering rule:** Do **not** clear `0x90180066` back to `0` immediately after step 5. Writing `0` to `0x90180066` belongs at the start of the *next* transfer cycle (step 2), where it is immediately followed by waiting for `0x90180008[0] == 0`. Clearing `0x90180066` to `0` at the end of a transfer leaves `0x90180008[0]` at `1` while the doorbell is already `0`, which can desynchronize the next word transfer. (Equivalently, a loop that begins with `wait 0x90180008[0] == 1` → write `0x90180064`/`68` → write `0x90180066 = 0` → `wait 0x90180008[0] == 0` → write `0x90180066 = 1` implements the exact same state machine shifted by one step, because `0x90180008[0]` is `1` at reset and after every completed host ACK.)

Acknowledge each major item, the streaming introducer, and **every streamed 32-bit word** (the 32-bit header `(id << 16) | len` plus the `len` payload words) unless the terminal `0xff` exception below applies.

## Major messages

After the ready wait, the host reads `0xd0032` and classifies it as follows:

| Value tested | Meaning | Reference action |
|---|---|---|
| Low byte `0x07` | Normal completion | ACK and return success (0). |
| Low byte `0x08` | Streaming introducer | ACK, then read count and streamed words. |
| Full value `0x000000ff` | Fatal/error notification | Special terminal path **without ACK**; see below. |
| Any other low byte | Major stage/progress item | Report optionally, ACK, continue. |

The error test is the full value `0xff`, not merely any item whose low byte is `0xff`.

A full-training capture on two controllers produced these 22 major bytes, including the final completion byte of the first controller:

```text
0d 00 01 02 fd 06 01 02 fd 04 03 09 07
0d 00 01 02 fd 06 01 02 fd
```

On a complete 13-byte controller run (`0d 00 01 02 fd 06 01 02 fd 04 03 09 07`), the major bytes follow a consistent milestone pattern across the enabled `SequenceCtrl = 0x125f` stages:

| Sequence slice | Observed role in the training flow |
|---|---|
| `0x0d` | Initial training framework / `DevInit` entry |
| `0x00`, `0x01`, `0x02`, `0xfd` | First leveling and receive-gate pass (`WrLvl` / `RxEn` / initial DQ setup; `0xfd` marks the end of the sub-loop pass) |
| `0x06`, `0x01`, `0x02`, `0xfd` | Retraining / refinement pass across `RxEn` and write leveling (`DisableTrainingLoop == 0`) |
| `0x04`, `0x03`, `0x09` | Final read/write 2D eye optimization (`RdDQS`, `WrDq`), `DRAM DCA`, `MxRdLat`, and `LPCA` stages |
| `0x07` | Normal training completion |

The second controller's capture ended in the truncation shown because the host trace buffer stopped after 22 major bytes; it does not establish that completion is optional. Because major stage bytes (other than `0x07`, `0x08`, and `0x000000ff`) are emitted with variable runtime stage numbers, a host consumer must treat all non-terminal major codes as informational progress items rather than branching on specific intermediate byte values.

The PMU normal-completion path emits `0x07` and then remains stopped. Its fatal path emits `0xff` and remains in a terminal break/loop. Completion and error also cause different PMU-side PHY control pulses at ARC CSR `0x90040238` (normal completion pulses `1 -> 0`; fatal error pulses `4 -> 0`), reinforcing that `0xff` is terminal rather than an ordinary item requiring the host to continue.

## Streaming transport

### Wire sequence

On the wire, a streaming message consists of the 16-bit major introducer `0x08` followed by **`len + 1` 32-bit words** (`word 0` is the 32-bit header `(id << 16) | len`, and `words 1..len` are the `len` payload words). The PMU does **not** transmit a separate 16-bit count word before the 32-bit header:

1. **PMU side**:
   - Send 16-bit major code `0x08` (`0x90180064 = 0x0008`) and complete its 5-step handshake.
   - Immediately load the 32-bit header `(id << 16) | len` (`0x90180064 = len`, `0x90180068 = id`) and assert the doorbell (`0x90180066 = 0` → wait `0x90180008[0] == 0` → `0x90180066 = 1` → wait `0x90180008[0] == 1`).
   - For each of the `len` payload words (`k = 0 .. len - 1`), load `0x90180064 = payload[k] & 0xffff`, `0x90180068 = payload[k] >> 16`, and complete one 5-step handshake.
2. **Host side**:
   - After receiving and acknowledging major/command `0x08`, wait for ready (`UCT_SHADOW.bit0 == 0`). At that instant, the PMU has already placed the 32-bit header `(id << 16) | len` into `0xd0034:0xd0032`.
   - Read `j = UCT_WRITE_ONLY_SHADOW` (`0xd0032`) **without issuing an ACK**. Because the header is already loaded in the shadow registers, `j` is simply `len` (the low 16 bits of the header), so **`j + 1` (`len + 1`) 32-bit words** (the header currently in the registers plus `len` subsequent payload words) are consumed in the loop that follows.
   - Loop `j + 1` times (`i = 0 .. j`):
     - read low 16 bits from `0xd0032` (on `i = 0`, this re-reads `len` from the header);
     - read high 16 bits from `0xd0034` (on `i = 0`, this reads `id` from the header);
     - reconstruct `word = (high << 16) | low`;
     - ACK that word;
     - wait for status bit 0 to return to 0 before the next word.
   - Resume major-message processing using the next `0xd0032` value.

The `0x08` item is not itself the 32-bit header and its acknowledgement does not replace any data-word ACK. Conversely, a PMU implementation must **never** insert an extra standalone `len` transfer between `0x08` and `(id << 16) | len`, because doing so would shift the host's `j + 1` loop by one word.

### Message framing

The first streamed word is a header:

```text
header = (id << 16) | len
payload words = exactly len
total words after 0x08 = len + 1 = j + 1
```

The following `len` words are payload. IDs are 16 bits in this framing; observed IDs were `0x0000`–`0x02ff`, and observed payload lengths were 0–31 words. Those observed ranges are capture facts, not a new implementation's license to omit bounds checks.

Example with two payload words:

```text
major 0x08, ACK
word 0: 0x00420002    -> id=0x0042, len=2
word 1: payload[0]
word 2: payload[1]
next major item
```

### Reference capture behavior and limits

The unmodified SPL consumes framing but does not decode payloads: its translation routine is empty, normal debug logging is disabled, and its optional capture array holds only 50 words. The instrumented capture printed raw words and produced approximately 1.4 MiB of UART output, greatly slowing boot.

One two-controller full-training capture with `HdtCtrl = 0x04` and `Train2DMisc = 0x0004` contained **9,153 streaming messages and 124 distinct IDs**. Most payload schemas are unknown. Confirmed framing must not be confused with inferred opcode meanings.

When `Train2DMisc[0] = 1` and `HdtCtrl = 0x05`, the PMU additionally streams **2D eye contours** using two message IDs:

- `0x0076` (`len = 4`): contour header `0x00760004 <ctrl> <rank> <sub> <lane>`, where `<ctrl>` is `0` or `1`, `<rank>` is `0..3` (`A0, A1, B0, B1`), `<sub>` is `0..17`, and `<lane>` is `0..8`.
- `0x0077` (`len = 3`): 64 consecutive row slices `0x00770003 <left_edge> <right_edge> <vref_step>` for `vref_step = 0..63`, where `<left_edge>` and `<right_edge>` are horizontal delay-tap bounds at that VREF step (`left = right = 0x42` indicates a closed/unopened row).

**Host UART capture optimization:** When capturing streaming messages over UART (especially with `Train2DMisc[0] = 1`, which emits `1,296 × 65 = 84,240` contour frames), the host receiver must print only the **`len + 1` actual words** of each frame (`header` + `len` payload words) rather than dumping a fixed 15- or 16-word buffer. Printing only `len + 1` words reduces UART traffic by ~4× (~690 KB vs. >2.7 MB) and avoids multi-minute mailbox stalls.

### Filtering

`HdtCtrl` is a threshold: a streamed message is emitted when its level is **less than** `HdtCtrl`. Increasing the threshold therefore suppresses lower-priority detail. Named levels are:

| Level | Class |
|---:|---|
| `0x05` | Detailed debug such as eye delays |
| `0x0a` | Coarse/rank debug |
| `0xc8` | Stage completion |
| `0xc9` | Assertions |
| `0xff` | Firmware completion only |

`MsgMisc[4]` suppresses streaming, including streamed assertions, regardless of `HdtCtrl`. Major completion/error traffic is separate. The reference board's persisted `HdtCtrl=0x04` explains the detailed capture; it is not a required production value.

## Fatal `0xff`: observed behavior

The reference SPL handles full-word `0xff` specially:

1. It does **not** call the normal ACK routine for `0xff`.
2. It writes `MICRO_RESET=1`.
3. It performs a short busy wait.
4. It selects host/APB access with `MICRO_CONT_MUX_SEL=0`.
5. It reads `0x200c9` as `plllockstatus` and logs it when logging is enabled.
6. It returns that raw PLL-status value from the mailbox routine.

This creates inconsistent propagation:

- The normal LPDDR5 caller checks whether the mailbox return value **equals `0xff`**. Since the routine returns PLL status, an actual PMU `0xff` can escape detection when PLL status is not `0xff`.
- The quickboot caller ignores the mailbox return value entirely.
- Normal completion returns 0 and is unambiguous.

Therefore, the reference's `0xff` handling is **not a dependable error-detection contract**. PLL lock status is diagnostic metadata and must not be substituted for a terminal status.

The lack of ACK is deliberate observed behavior for this terminal path, not an omission to “fix” by blindly applying the normal ACK sequence to an already terminated PMU.

## Recommended new implementation

The following policy is recommended and intentionally differs from the reference's defective propagation:

1. Represent receive outcomes explicitly, for example `PROGRESS`, `COMPLETE`, `PMU_ERROR`, `TIMEOUT`, and `PROTOCOL_ERROR`; do not encode them in raw PLL status.
2. On full-word `0xff`, latch `PMU_ERROR`, preserve the last major/streaming diagnostics and PLL status, perform bounded PMU-stop/host-mux cleanup, and propagate failure to every caller, including quickboot.
3. Do not issue the normal ACK for `0xff` when implementing the observed terminal-peer protocol. Do not wait for another item afterward.
4. ACK all normal completion, major, streaming-introducer, header, and payload items exactly once and in order.
5. Add independent bounded waits to every ready/acknowledgement poll. On timeout, stop the PMU if possible, restore host ownership, keep DDRC ports disabled, and return a distinct error.
6. Validate `len` against the consumer's buffer/stream-processing limit before reading payload. Even when payload is discarded, continue to read and ACK exactly `len` words to preserve transport synchronization; otherwise abort with an explicit protocol error and cleanup.
7. Treat unknown major IDs as progress and unknown streaming IDs as opaque framed payloads. Reserve fatal handling for `0xff`, explicit local validation failures, and timeouts.
8. Maintain a small host-side circular ring buffer (for example the last 16–32 major codes and streaming headers `(id << 16) | len`) in RAM without printing them over UART during normal boot, and dump that ring buffer only if `0xff` or a timeout occurs. This provides post-mortem visibility without incurring the ~1.4 MiB UART bottleneck.
9. Require both normal completion and valid training outputs before result publication or controller activation.

## Receiver state summary

A compatible normal-path receiver can be understood as four states:

```text
WAIT_MAJOR
  -> ACK major, WAIT_MAJOR
  -> 0x08: ACK, READ_COUNT
  -> 0x07: ACK, COMPLETE
  -> 0xff: no ACK, PMU_ERROR

READ_COUNT
  -> expect count+1 words, READ_WORD

READ_WORD
  -> read low/high, ACK, wait ready
  -> remaining words: READ_WORD
  -> frame complete: WAIT_MAJOR
```

Every wait in this state model needs a timeout in a new implementation. The terminal states must carry status independently of diagnostic register values.
