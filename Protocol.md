# FT8 plus Protocol

**Version**: 1.0  
**Date**: 2026-09-16  
**Author**: BI3BJU  

---

## 1. Physical Layer Frame Structure

### 1.1 77-bit Raw Data (bits77)

| Field | Length (bits) | Description |
|------|-----------|------|
| `b72` | 72 | User data area, 9 bytes, UTF-8 encoded byte sequence |
| `f2` | 2 | Frame type identifier, values 0–3 |
| `i3` | 3 | Fixed to `6`, indicating FT8 plus type |

**Assembly formula**:

```text
bits77 = (b72 << 5) | (f2 << 3) | 6
```

### 1.2 Physical Layer Coding Format

Compatible with the standard FT8 physical layer coding format:

```text
bits77
  + 14-bit CRC (polynomial 0x2757) => bits91
  + LDPC encoding (rate 91/174) => 174 bits
  + map every 3 bits to 1 FT8 symbol (0–7)
  + add fixed Costas sync sequence
  => 79 symbols
```

### 1.3 Receiver-side Field Extraction

After CRC verification passes and `bits77` is recovered:

```text
i3 = bits77 & 0b111
if i3 == 6:
    bits74 = bits77 >> 3
    f2  = bits74 & 0b11
    b72 = bits74 >> 2
```

---

## 2. Upper Layer Protocol Frame Types (f2)

| f2 value | Type | Purpose | Processing requirement |
|-------|------|------|----------|
| `0` | Single-frame mode | Short message, ≤9 bytes | Decode and output directly |
| `1` | Continuous-frame mode | Long message, streamed broadcast framing | Group by frequency and reassemble |
| `2` | Reserved | Reserved for reliable transport ARQ | Silently discard |
| `3` | Reserved | Reserved for reliable transport ARQ | Silently discard |

---

## 3. `b72` Composition Structure

`b72` is 72 bits total, i.e., 9 bytes. All user data is a UTF-8 encoded byte sequence.

| f2 | Mode | b72 composition (9 bytes) | Description |
|----|------|---------------------|------|
| `0` | Single-frame | `[ UTF-8 data (1–9 bytes) ][ 0x00 padding ]` | Valid data left-aligned, remaining bytes padded with `0x00` |
| `1` | Continuous-frame | `[ UTF-8 data fragment (9 bytes) ]` | All 9 bytes are payload fragments; the last frame is right-padded with `0x00` if shorter than 9 bytes |
| `2/3` | Reserved | Undefined | Ignore |

---

## 4. Single-frame Mode (f2=0)

```text
b72 = [ UTF-8 data (1–9 bytes) ][ 0x00 padding to 9 bytes ]
```

- No CRC or EOT is added.
- Valid data is left-aligned, padded with `0x00` on the right.
- On reception, strip trailing `0x00` and decode as UTF-8.

---

## 5. Continuous-frame Mode (f2=1)

### 5.1 Payload Format

```text
payload = UTF-8 raw text bytes + CRC8 (1 byte) + EOT (1 byte, fixed 0x04)
```

- Maximum raw text length: `574` bytes.
- Maximum total payload length: `576` bytes.
- Fragmentation: split sequentially into 9-byte chunks; each frame has `b72 = that fragment`, `f2 = 1`.
- The last frame is right-padded with `0x00` if shorter than 9 bytes.
- Maximum of 64 frames.

### 5.2 CRC8 Format

| Item | Value |
|------|----|
| Polynomial | `0x07` |
| Initial value | `0x00` |
| Final XOR | None |
| Coverage | Only the UTF-8 encoded bytes of the raw text, excluding CRC and EOT |

### 5.3 EOT Format

| Field | Length | Value |
|------|----|----|
| EOT | 1 byte | Fixed `0x04` |

### 5.4 Reassembly and Parsing

Concatenate all `b72` fragments in the same frequency group in reception order:

```text
raw = concatenated byte string
raw = raw.rstrip(b'\x00')   # remove trailing physical padding
```

After removing padding, parse from the end:

```text
Last 1 byte        = EOT, fixed 0x04
Second-to-last byte = received CRC8
Remaining part      = candidate UTF-8 encoded data
```

If the byte string after removing padding is less than 2 bytes long, treat it as invalid data.

---

## 6. Encoding Requirements

- All user data (the `b72` in both single-frame and continuous-frame modes) uses UTF-8 encoding.
- The receiver must decode as UTF-8.
- Invalid UTF-8 byte sequences are replaced with `�` (U+FFFD).

---


## 7. Callsign Validation Rules

### 7.1 Scope of Application

Applies to the **transmitting callsign** and **target callsign** in all application layer messages. The sender must validate; the receiver should validate and discard the message if validation fails.

### 7.2 Character Set and Case

- Only **uppercase English letters `A`–`Z`** and **digits `0`–`9`** are allowed.
- Lowercase letters, spaces, slash `/`, hyphen `-`, and other characters are not allowed.
- Lowercase input letters shall be converted to uppercase before validation and transmission.
- Callsigns with a slash (e.g., `BI3BJU/P`) are not supported; use the base callsign.

### 7.3 Length

The total callsign length must be **3 to 6 characters**.

### 7.4 Structure

```text
prefix + area digit + suffix
```

| Part | Length | Character set | Description |
|------|------|--------|------|
| Prefix | 1–2 characters | `A`–`Z`, `0`–`9` | Must contain at least one letter; if the first character is a digit, the second character must be a letter |
| Area digit | 1 character | `0`–`9` | Exactly one digit |
| Suffix | 1–4 characters | `A`–`Z` | Uppercase letters only |

### 7.5 Regular Expression

```regex
^(?=.{3,6}$)(?:[A-Z][A-Z0-9]?|[0-9][A-Z])[0-9][A-Z]{1,4}$
```

### 7.6 Validation Procedure

1. Convert the callsign to uppercase.
2. Match against the regular expression in Section 8.5.
3. Match fails: the sender refuses to transmit, and the receiver discards the message.
4. Match succeeds: continue processing.

### 7.7 Examples

| Callsign | Valid | Description |
|------|------|------|
| `BI3BJU` | ✅ | Prefix `BI`, area `3`, suffix `BJU` |
| `K1ABC` | ✅ | Prefix `K`, area `1`, suffix `ABC` |
| `2E0ABC` | ✅ | Prefix `2E`, area `0`, suffix `ABC` |
| `9A1A` | ✅ | Prefix `9A`, area `1`, suffix `A` |
| `B3BJU` | ✅ | Prefix `B`, area `3`, suffix `BJU` |
| `A1B` | ✅ | Prefix `A`, area `1`, suffix `B` |
| `12ABC` | ❌ | Prefix is all digits |
| `A1` | ❌ | Total length less than 3 |
| `BI3BJUU` | ❌ | Total length exceeds 6 |
| `BI3BJU/P` | ❌ | Contains a slash |
| `bi3bju` | ✅ (after conversion) | Valid after conversion to `BI3BJU` |

---

