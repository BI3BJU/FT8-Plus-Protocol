# FT8 plus Protocol

**Version**: 1.0 
**Date**: 2026-10-02  
**Author**: BI3BJU

---

## 1. Physical Layer Frame Structure

### 1.1 77-bit Raw Data (bits77)

| Field | Length (bits) | Description |
|-------|---------------|-------------|
| `b72` | 72 | User data field, 9 bytes, UTF-8 encoded byte sequence |
| `f2` | 2 | Frame type identifier, values 0–3 |
| `i3` | 3 | Fixed to `6`, indicating FT8 plus type |

**Assembly formula**:

```text
bits77 = (b72 << 5) | (f2 << 3) | 6
```

### 1.2 Physical Layer Encoding Format

Compatible with the standard FT8 physical layer encoding format:

```text
bits77
  + 14-bit CRC (polynomial 0x2757) => bits91
  + LDPC encoding (code rate 91/174) => 174 bits
  + map every 3 bits to 1 FT8 symbol (0–7)
  + add fixed Costas synchronization sequence
  => 79 symbols
```

### 1.3 Receiver-Side Field Extraction

After CRC verification passes and `bits77` is recovered:

```text
i3 = bits77 & 0b111
if i3 == 6:
    bits74 = bits77 >> 3
    f2  = bits74 & 0b11
    b72 = bits74 >> 2
else:
    Do not parse as FT8 plus; fall back to the standard FT8 process.
```

That is: the Plus side only processes frames with `i3 == 6`; standard frames with `i3 = 0,1`, etc. are parsed according to the original standard FT8 process and will not be misidentified as FT8 plus frames.

---

## 2. Upper-Layer Protocol Frame Types (f2)

| f2 value | Type | Purpose | Processing requirement |
|----------|------|---------|------------------------|
| `0` | Single-frame mode | Short message, ≤9 bytes | Decode and output directly |
| `1` | Continuous-frame mode | Long message, streamed broadcast fragmentation | Reassemble by frequency group; frame loss allowed |
| `2` | Reserved | Reserved for reliable transmission ARQ | Silently discard |
| `3` | Reserved | Reserved for reliable transmission ARQ | Silently discard |

---

## 3. `b72` Composition

`b72` is 72 bits in total, i.e., 9 bytes. All user data is a UTF-8 encoded byte sequence.

| f2 | Mode | b72 composition (9 bytes) | Description |
|----|------|----------------------------|-------------|
| `0` | Single-frame | `[ UTF-8 data (1–9 bytes) ][ 0x00 padding ]` | Valid data is left-aligned; the remainder is padded with `0x00` |
| `1` | Continuous-frame | `[ UTF-8 data fragment (9 bytes) ]` | All 9 bytes are payload fragments; if the last frame is less than 9 bytes, pad right with `0x00` |
| `2/3` | Reserved | Undefined | Ignore |

---

## 4. Single-Frame Mode (f2=0)

```text
b72 = [ UTF-8 data (1–9 bytes) ][ 0x00 padding to 9 bytes ]
```

- No CRC or EOT is added.
- Valid data is left-aligned and padded with `0x00` on the right.
- On reception, remove trailing `0x00` and decode as UTF-8.
- A single frame can be independently identified, decoded, and displayed; damage to or loss of a single frame does not affect other frames.

---

## 5. Continuous-Frame Mode (f2=1)

### 5.1 Payload Format

```text
payload = UTF-8 original text bytes + CRC8 (1 byte) + EOT (1 byte, fixed 0x04)
```

- Maximum original text length: `574` bytes.
- Maximum total payload length: `576` bytes.
- Fragmentation: split sequentially into 9-byte pieces; each frame has `b72 = that fragment`, `f2 = 1`.
- If the last frame is less than 9 bytes, pad right with `0x00`.
- Maximum of 64 frames.

### 5.2 CRC8 Format

| Item | Value |
|------|-------|
| Polynomial | `0x07` |
| Initial value | `0x00` |
| Final XOR | None |
| Coverage | Only the UTF-8 encoded bytes of the original text; excludes CRC and EOT |

### 5.3 EOT Format

| Field | Length | Value |
|-------|--------|-------|
| EOT | 1 byte | Fixed `0x04` |

### 5.4 Reassembly and Parsing

Concatenate all `b72` fragments within the same frequency group in reception order:

```text
raw = concatenated byte string
raw = raw.rstrip(b'\x00')   # remove trailing physical padding
```

After removing padding, parse from the end:

```text
last 1 byte        = EOT, fixed 0x04
second-to-last byte = received CRC8
remaining part      = candidate UTF-8 encoded data
```

If the byte string after removing padding is less than 2 bytes, it is considered invalid data.

### 5.5 Continuous-Frame Loss Tolerance and Arbitrary Frame Start

1. **Frame loss allowed**: Do not discard the entire message because of missing frames.
2. **Support starting from any frame**: The receiver may start a continuous-frame buffer from any `f2=1` frame within the same frequency group; it is not required to receive a specific first frame.
3. **Continuous-frame loss allowed**: Subsequent `f2=1` frames in the same frequency group are appended in reception order; after intermediate frame loss, the received fragments are still concatenated in reception order.
4. **Complete reception**: If the continuous frames are completely received and EOT and CRC8 can be parsed, then:
   - Perform UTF-8 decoding on the candidate UTF-8 data;
   - Calculate CRC8 on the candidate data and compare it with the extracted value;
   - If CRC8 verification passes: display according to UTF-8 decoding and mark as complete;
   - If CRC8 verification fails: still display the decoded content and mark "integrity check failed".
5. **Unable to parse completely**: If EOT cannot be parsed, or the length is less than 2, then for each received fragment, independently remove trailing `0x00` and attempt UTF-8 decoding; if successful, display it; if failed, discard that fragment.
6. Damage to or loss of a single frame does not affect other frames.

---

## 6. Encoding Requirements

- All user data (the `b72` in both single-frame and continuous-frame modes) uses UTF-8 encoding.
- The receiver must decode according to UTF-8.
- Illegal UTF-8 byte sequences are replaced with `�` (U+FFFD).

---

## 7. Compatibility and No Misidentification

### 7.1 Standard End Receives Plus Frames

When a standard FT8 decoder encounters `i3 = 6`, it treats it as a non-standard/reserved type and automatically silently discards it.

- Does not display garbled text;
- Does not crash;
- Does not enter standard FT8 message parsing.

### 7.2 Plus End Receives Standard Frames

When the Plus end receives, it extracts `i3`:

- If `i3 == 6`: parse as FT8 plus and process according to `f2`;
- If `i3 != 6`, e.g., `i3 = 0,1`: do not enter FT8 plus parsing; fall back to the standard FT8 process.

Therefore, standard frames with `i3 = 0,1` will not be misidentified as FT8 plus frames.

### 7.3 No Misidentification

Since `i3 = 6` is the dedicated identifier for FT8 plus:

- Standard FT8 frames do not use `i3 = 6`;
- FT8 plus frames must use `i3 = 6`.

Therefore:

- The Plus end will not misidentify standard frames as Plus frames;
- The standard end will not misidentify Plus frames as standard messages, but will automatically discard them;
- The current scheme has no misidentification.

---

