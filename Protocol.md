# FT8 Plus Protocol

**Version**: 1.0  
**Date**: 2026-10-03  
**Author**: BI3BJU  

---

## 1. Physical Layer

- `Use standard FT8 physical layer encoding for bits77 and below`
- `bits77 = (b72 << 5) | (f2 << 3) | 6`
- `b72`: 72 bits, i.e., 9 bytes of user data.
- `b72` uses big-endian order: among the 9 bytes of user data, the first byte is in the most significant 8 bits, and the last byte is in the least significant 8 bits.
- `f2`: 2-bit frame type.
- `i3`: 3 bits, fixed to `6`.

---

## 2. Frame Type (f2)

| f2 | Type | Handling |
|---|---|---|
| `0` | Single-frame mode | 
| `1` | Continuous-frame mode | 
| `2` | Reserved ARQ | Discard |
| `3` | Reserved ARQ | Discard |

---

## 3. Data Format (b72)

### 3.1 General Rules

- Input is limited to **printable Unicode BMP characters**, excluding control characters and format characters.
- All data uses **UTF-8** encoding.
- The receiver decodes according to UTF-8; invalid sequences are replaced with `�` (U+FFFD).

---

### 3.2 Single-Frame Mode (f2=0)

Payload format:
```text
b72 = [ UTF-8 text (1~9 bytes) ][ 0x20 padding to 9 bytes ]
```

Rules:
- Text maximum 9 bytes.
- Left-aligned, padded on the right with spaces `0x20`.
- No CRC8 is appended, and no EOT is appended.

---

### 3.3 Continuous-Frame Mode (f2=1)

Payload format:
```text
b72 = [ UTF-8 text ][ 0x20 padding ][ CRC8 ][ EOT 0x04 ]
```

Rules:
- Total length must be a **multiple of 9**.
- Total length maximum `576` bytes, i.e., at most `64` frames.
- Text maximum `574` bytes.
- The last byte is fixed to `0x04`.
- Padding uses spaces `0x20`, located before CRC8.
- CRC8 is located before EOT and covers from the first payload byte to the byte before CRC8.
- CRC-8/ATM: polynomial `0x07`, initial value `0x00`, result XOR `0x00`.

---

## 4. Compatibility

- When a standard FT8 endpoint receives a Plus frame with `i3=6`: silently discard it, do not display garbled text, and do not enter standard parsing.