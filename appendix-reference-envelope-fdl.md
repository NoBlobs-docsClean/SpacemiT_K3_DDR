<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Appendix: Reference Envelope (SPI)](appendix-reference-envelope.md) | [Index](README.md#reading-order-and-contents) | [Next: `E1` Dump →](appendix-reference-envelope-fdl-e1.md)

# Appendix: Reference 6400 MT/s USB/FDL Training Envelopes (`E1`–`E3`, `G1`–`G3`, `K1`–`K3`)

## Scope and relationship to the SPI reference envelope

This appendix indexes **nine** complete **observed** full-training `DDRT` envelopes captured through the **USB/FDL staging path** on the reference SpacemiT K3 Pico-ITX board (`MT62F2G32D4DS`, 2 ranks, x16, `6400 MT/s`, `SequenceCtrl = 0x125f`, default PHY RX ODT `CSR[0x010048/4a/4c/4d] = 0x0c00` = `60 Ω`) across three controlled thermal conditions (`E1`–`E3` forced-air ambient, `G1`–`G3` heated, `K1`–`K3` max heater). Each individual appendix uses the exact same plain-text hexadecimal format as [appendix-reference-envelope.md](appendix-reference-envelope.md).

Common properties across all 9 clean `60 Ω` USB/FDL captures:
- **Matching read-eye delay centers with SPI (`60 Ω` RX ODT)**: All 9 runs converge to `0x4e`/`0x4f` medians of **`61.0`–`63.0` taps** (`mean 62.04–62.83` on `DDR0`, `61.17–62.61` on `DDR1`), matching the factory SPI envelope (`62.5` / `62.0` taps) and confirming negligible thermal drift (`<= ~1` tap) between ambient cooling and maximum heater (`K1`–`K3`). (Earlier runs `C1`–`C3` and `H1`–`H3` showed `~81`–`82` taps because of a `120 Ω` `CSR[0x010048] = 0x0800` override; see [14-validation-and-current-status.md](14-validation-and-current-status.md).)
- **Uniform USB/FDL full-training markers**: Every run records `CSR[0x020011] = 0x006f` (restored at `Step 6.3`), `TrainedVREFCA_A0..B1 = 0x3f` (`63`, 6-bit DAC upper rail), `CSR[0x0c0086] = 0x0001`, `CSR[0x030001] = 0x0004` (`DDR0`) / `0x0000` (`DDR1`) with `0x030101`–`0x031901 = 0x0000`, `3874` / `3875` nonzero `phypara` halfwords, and `1845` / `1845` nonzero `ACSM` halfwords.
- **Valid `0x9800`-byte layout**: Bytes `0x93c8..0x97ff` (`1,080` bytes) are all `0x00` in every capture; concatenating header (`0x0080`) + `DDR0` (`0x49a4`) + `DDR1` (`0x49a4`) + `1,080` zero bytes yields the exact binary whose `zlib.crc32(buf[0x08:0x9800])` matches the header `crc32` field.

## Published USB/FDL captures

| Run | Condition | Duration (ms) | `crc32` | Nonzero `phypara` (`DDR0`/`1`) | `CSR[0x02000a]` (`DDR0`/`1`) | `CSR[0x020187]` (`DDR0`/`1`) | `0x4e/0x4f` median (`DDR0`/`1`) | `0x4e/0x4f` mean (`DDR0`/`1`) | `TrainedVREFDQ_A0` (`DDR0`/`1`) | `RxClkDly_Margin_A0` (`DDR0`/`1`) |
|---|---|---:|---|---|---|---|---:|---:|---|---|
| [**`E1`**](appendix-reference-envelope-fdl-e1.md) | Ambient fan-cooled | `5,622` | `0xfa1e9f98` | `3874` / `3875` | `135` / `133` | `67` / `69` | **`63.0` / `61.5`** | `62.25` / `61.19` | `0x27` / `0x26` | `0x16` / `0x16` |
| [**`E2`**](appendix-reference-envelope-fdl-e2.md) | Ambient fan-cooled | `5,623` | `0x7dcd90a7` | `3874` / `3875` | `135` / `133` | `67` / `69` | **`63.0` / `61.0`** | `62.04` / `61.17` | `0x28` / `0x26` | `0x15` / `0x15` |
| [**`E3`**](appendix-reference-envelope-fdl-e3.md) | Ambient fan-cooled | `5,627` | `0x300b4b95` | `3874` / `3875` | `135` / `133` | `68` / `69` | **`63.0` / `61.5`** | `62.19` / `61.36` | `0x26` / `0x26` | `0x16` / `0x16` |
| [**`G1`**](appendix-reference-envelope-fdl-g1.md) | External heater | `5,626` | `0x22665f20` | `3874` / `3875` | `134` / `132` | `67` / `69` | **`63.0` / `62.0`** | `62.83` / `61.38` | `0x27` / `0x27` | `0x17` / `0x16` |
| [**`G2`**](appendix-reference-envelope-fdl-g2.md) | External heater | `5,624` | `0x9e4b0160` | `3874` / `3875` | `133` / `132` | `67` / `67` | **`63.0` / `63.0`** | `62.58` / `61.94` | `0x26` / `0x27` | `0x15` / `0x15` |
| [**`G3`**](appendix-reference-envelope-fdl-g3.md) | External heater | `5,628` | `0x16ddbe38` | `3874` / `3875` | `133` / `133` | `67` / `68` | **`62.5` / `62.0`** | `62.32` / `61.82` | `0x28` / `0x26` | `0x16` / `0x15` |
| [**`K1`**](appendix-reference-envelope-fdl-k1.md) | Max external heater | `5,627` | `0xa8ff32bf` | `3874` / `3875` | `133` / `131` | `67` / `67` | **`63.0` / `63.0`** | `62.14` / `62.47` | `0x27` / `0x26` | `0x15` / `0x14` |
| [**`K2`**](appendix-reference-envelope-fdl-k2.md) | Max external heater | `5,628` | `0xdc9528ba` | `3874` / `3875` | `133` / `131` | `67` / `67` | **`63.0` / `63.0`** | `62.75` / `62.39` | `0x26` / `0x25` | `0x15` / `0x16` |
| [**`K3`**](appendix-reference-envelope-fdl-k3.md) | Max external heater | `5,628` | `0xde7aa2be` | `3874` / `3875` | `132` / `131` | `66` / `67` | **`63.0` / `63.0`** | `62.60` / `62.61` | `0x26` / `0x27` | `0x16` / `0x15` |
