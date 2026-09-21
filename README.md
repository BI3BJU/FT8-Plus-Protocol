# FT8 plus Protocol

A lightweight extension compatible with the standard FT8 physical layer, enabling streaming and loss-tolerant transmission of long Unicode text messages.

Version 1.0 · 2026-09-16 · BI3BJU

## Physical Layer

Compatible with FT8: `bits77 + CRC14(0x2757) + LDPC(91/174) + Costas => 79 symbols`

## Frame Structure

`bits77 = (b72 << 5) | (f2 << 3) | 6`

| Field | Bits | Description |
|---|---|---|
| b72 | 72 | 9-byte UTF-8 data |
| f2 | 2 | Subframe type |
| i3 | 3 | Frame type extension, fixed to 6 |

Subframe (f2) types: 0 single frame (≤9 bytes); 1 continuous frame (≤574 bytes, CRC8+EOT, up to 64 frames); 2/3 reserved.

## Application Layer

- Beacon: `CALLSIGN GRID`, e.g. `BI3BJU OM89EW`
- Broadcast: `CALLSIGN:message`, e.g. `BI3BJU:Hello everyone`
- Directed: `CALLSIGN@TARGET message`, e.g. `BI3BJU@K1ABC Hello`

Callsign: uppercase `A-Z`, `0-9`, length 3–6.

## Saved Files

UTF-8 without BOM, located in the working directory:

- `config.txt`: configuration
- `contact.txt`: contacts, up to 100 entries
- `history.txt`: history, up to 100 lines
- `status.txt`: status, up to 100 lines