# GB 46750-2025 — Broadcast Remote ID Protocol

Civil Unmanned Aircraft System Operational Identification
(中华人民共和国民用无人驾驶航空器系统运营识别规范)

Published 2025-10-31, effective 2026-05-01.

This library implements the **broadcast mode** (广播模式) defined in GB 46750-2025. It is
independent of ASTM F3411 — GB 46750-2025 uses its own wire format, field set, and
transmission rules. See the [Differences from ASTM F3411](#differences-from-astm-f3411)
section below.

## 1. Packet Format

Each broadcast-mode RID packet uses a **flag-bitmask chain** structure (not TLV tags):

```
┌──────────┬──────────┬──────────┬───────────────────┬──────────────────────┐
│  Type    │ Version  │ DataLen  │   Flag Bytes      │     Data             │
│  1 byte  │ 1 byte   │ 1 byte   │   1–3 bytes       │   N bytes (≤200)     │
│  0xFF    │          │  1–200   │                   │                      │
└──────────┴──────────┴──────────┴───────────────────┴──────────────────────┘
```

| Byte offset | Field    | Description |
|-------------|----------|-------------|
| 0           | Type     | Always `0xFF` — distinguishes GB 46750 packets from other beacon payload. |
| 1           | Version  | Bits 7–5: protocol major version, always `001` (V1). Bits 4–0: sub-version, 0–63 (rendered as `V1.X`). |
| 2           | DataLen  | Total byte count of the Flag Bytes + Data region (1–200). |
| 3 … 3+N-1   | Flags    | 1–3 flag bytes, chained: bit 0 of each byte is the *extension flag* — `1` = another flag byte follows, `0` = last flag byte. Bits 7–1 signal the presence of protocol fields (see Section 2). |
| 3+N …       | Data     | Concatenated field values, each fixed-length per the protocol. Fields appear in ascending field-ID order (001 → 021). Omitted fields (flag bit = 0) consume zero bytes here. |

**Flag chain rules:**
- A minimum of 1 flag byte is always present.
- A maximum of 3 flag bytes are defined; bit 0 of the 3rd byte is reserved (should be `0`).
- The receiver walks the chain to compute the total flag-region size, then subtracts it
  from `DataLen` to determine the data-region size.

## 2. Flag Bit Mapping

Each bit in the flag region (bits 7–1 of each flag byte) signals the presence of one
field. Fields marked **M** (Mandatory) are always transmitted; fields marked **O**
(Optional) are conditionally included.

| Flag Byte | Bit | Hex Mask | Field ID | Field Name            | M/O | Description                     |
|-----------|-----|----------|----------|-----------------------|-----|---------------------------------|
| Flag1     | 7   | `0x80`   | 001      | SN                    | M   | Product serial number           |
| Flag1     | 6   | `0x40`   | 002      | UIN                   | M   | Registration mark (last 8)      |
| Flag1     | 5   | `0x20`   | 003      | Operation Category    | O   | Open / Specific / Certified     |
| Flag1     | 4   | `0x10`   | 004      | Drone Class           | M   | Micro / Light / Small / …       |
| Flag1     | 3   | `0x08`   | 005      | GCS Position Type     | M   | Takeoff point or GCS location   |
| Flag1     | 2   | `0x04`   | 006      | GCS Latitude/Longitude| M   | 2 × int32 at 1e-7 degrees       |
| Flag1     | 1   | `0x02`   | 007      | GCS Geodetic Altitude | M   | uint16, 0.5 m resolution        |
| Flag1     | 0   | `0x01`   | —        | *(extension flag)*    | —   | `1` = Flag2 follows             |
| Flag2     | 7   | `0x80`   | 008      | UAV Latitude/Longitude| M   | 2 × int32 at 1e-7 degrees       |
| Flag2     | 6   | `0x40`   | 009      | Track Angle           | M   | uint16, 0.1° resolution         |
| Flag2     | 5   | `0x20`   | 010      | Ground Speed          | M   | uint16, 0.1 m/s resolution      |
| Flag2     | 4   | `0x10`   | 011      | Relative Altitude     | O   | Height above takeoff point      |
| Flag2     | 3   | `0x08`   | 012      | Vertical Speed        | O   | Signed 7-bit, 0.5 m/s steps     |
| Flag2     | 2   | `0x04`   | 013      | UAV Geodetic Altitude | M   | uint16, 0.5 m resolution        |
| Flag2     | 1   | `0x02`   | 014      | Barometric Altitude   | O   | uint16, 0.5 m resolution        |
| Flag2     | 0   | `0x01`   | —        | *(extension flag)*    | —   | `1` = Flag3 follows             |
| Flag3     | 7   | `0x80`   | 015      | Operation Status      | M   | Ground / Air / Emergency / …    |
| Flag3     | 6   | `0x40`   | 016      | Coordinate System     | M   | WGS-84 or CGCS2000              |
| Flag3     | 5   | `0x20`   | 017      | Horizontal Accuracy   | M   | GNSS NACp category (0–15)       |
| Flag3     | 4   | `0x10`   | 018      | Vertical Accuracy     | M   | GNSS GVA category (0–15)        |
| Flag3     | 3   | `0x08`   | 019      | Speed Accuracy        | M   | GNSS NACv category (0–15)       |
| Flag3     | 2   | `0x04`   | 020      | Timestamp             | M   | Unix ms, 6 bytes                |
| Flag3     | 1   | `0x02`   | 021      | Timestamp Accuracy    | M   | Category (0–15)                 |
| Flag3     | 0   | `0x01`   | —        | *(reserved)*          | —   | Should be `0`                   |

> Fields marked **M** (Mandatory) are always transmitted per the standard.
> Fields marked **O** (Optional) are transmitted only when the data is available.

## 3. Field Definitions

### Field Encoding Table

| ID  | Field Name           | Bytes | Encoding & Range                                          | Invalid / Absent | M/O |
|-----|----------------------|-------|-----------------------------------------------------------|------------------|-----|
| 001 | SN                   | 20    | ASCII per GB/T 41300, padded with `\0`. Max 20 chars.     | —                | M   |
| 002 | UIN                  | 8     | ASCII, last 8 chars of real-name registration mark, `\0`-padded. | —          | M   |
| 003 | Operation Category   | 1     | 0=Undefined, 1=Open, 2=Specific, 3=Certified, 4–15=Reserved. | —            | O   |
| 004 | Drone Class          | 1     | 0=Micro, 1=Light, 2=Small, 3=Medium, 4=Large, 5–15=Reserved. | —            | M   |
| 005 | GCS Position Type    | 1     | 0=Takeoff point, 1=GCS location, 2–15=Reserved.           | —                | M   |
| 006 | GCS Lat/Lon          | 8     | Two int32 LE, ×1e-7 degrees. Positive=N/positive=E.       | `0xFFFFFFFF` for each axis | M   |
| 007 | GCS Geodetic Altitude| 2     | uint16 LE, `(alt_m + 1000) × 2`, 0.5 m resolution.        | `0x0000`         | M   |
| 008 | UAV Lat/Lon          | 8     | Two int32 LE, ×1e-7 degrees. Positive=N/positive=E.       | `0xFFFFFFFF` for each axis | M   |
| 009 | Track Angle          | 2     | uint16 LE, `angle_deg × 10`, range 0–3599 (0.1° steps).   | `0xFFFF`         | M   |
| 010 | Ground Speed         | 2     | uint16 LE, `speed_mps × 10`, 0.1 m/s resolution.          | `0xFFFF`         | M   |
| 011 | Relative Altitude    | 2     | uint16 LE, `(alt_m + 9000) × 2`, 0.5 m resolution. Offset to takeoff point. | `0x0000` | O   |
| 012 | Vertical Speed       | 1     | Bits 6–0: magnitude × 0.5 m/s (0–127). Bit 7: sign (`0`=positive/ascending, `1`=negative/descending). `0` with sign bit `0` means `0 m/s`. | `0xFF` | O   |
| 013 | UAV Geodetic Altitude| 2     | uint16 LE, `(alt_m + 1000) × 2`, 0.5 m resolution.        | `0x0000`         | M   |
| 014 | Barometric Altitude  | 2     | uint16 LE, `(alt_m + 1000) × 2`, 0.5 m resolution. Ref: 101.325 kPa. | `0x0000` | O   |
| 015 | Operation Status     | 1     | 0=Unknown, 1=Ground, 2=Air, 3=Emergency, 4=RID failure (non-emergency), 5=RID failure (emergency), 6–15=Reserved. | — | M   |
| 016 | Coordinate System    | 1     | 0=WGS-84, 1=CGCS2000, 2–15=Reserved.                     | —                | M   |
| 017 | Horizontal Accuracy  | 1     | GNSS NACp: 0=>18.52 km, 1=<18.52 km, 2=<7.41 km, 3=<3.70 km, 4=<1852 m, 5=<926 m, 6=<556 m, 7=<185 m, 8=<92.6 m, 9=<30 m, 10=<10 m, 11=<3 m, 12=<1 m, 13–15=Reserved. 95% confidence. | — | M |
| 018 | Vertical Accuracy    | 1     | GNSS GVA: 0=>150 m, 1=<150 m, 2=<45 m, 3=<25 m, 4=<10 m, 5=<3 m, 6=<1 m, 7–15=Reserved. 95% confidence. | — | M   |
| 019 | Speed Accuracy       | 1     | GNSS NACv: 0=>10 m/s, 1=<10 m/s, 2=<3 m/s, 3=<1 m/s, 4=<0.3 m/s, 5–15=Reserved. 95% confidence. | — | M |
| 020 | Timestamp            | 6     | uint48 LE, Unix timestamp in milliseconds since epoch.     | `0x000000000000` | M   |
| 021 | Timestamp Accuracy   | 1     | 0=>0.5 s, 1=<0.5 s, 2=<0.4 s, 3=<0.3 s, 4=<0.2 s, 5=<0.1 s, 6=<50 ms, 7=<20 ms, 8=<10 ms, 9–15=Reserved. | — | M |

### Encoding Formulas — Quick Reference

| Field Group             | Wire Value                                  | Decode Formula                  |
|-------------------------|---------------------------------------------|---------------------------------|
| Latitude / Longitude    | `int32_le`                                  | `degrees = wire_val × 1e-7`     |
| Altitude (drone, GCS, barometric) | `uint16_le`                        | `meters = wire_val × 0.5 − 1000` |
| Relative Altitude       | `uint16_le`                                 | `meters = wire_val × 0.5 − 9000` |
| Track Angle             | `uint16_le`                                 | `degrees = wire_val × 0.1`      |
| Ground Speed            | `uint16_le`                                 | `m/s = wire_val × 0.1`          |
| Vertical Speed          | `(sign_bit << 7) \| (magnitude & 0x7F)`    | `m/s = magnitude × 0.5`, negative if sign bit = 1 |
| Timestamp               | `uint48_le` (6 bytes)                       | `ms = wire_val`                 |

All multi-byte integers are **little-endian**. The library uses byte-shift helpers
(`le16_to_h` / `h_to_le16`, etc.) rather than `<endian.h>`, so the code is
endian-agnostic at compile time.

## 4. RF / Broadcast Requirements

Broadcast mode supports the following transmission methods:

- **Bluetooth** — 5.0 and above
- **Wi-Fi** — 2.4 GHz / 5.8 GHz

## Differences from ASTM F3411

GB 46750-2025 is **not wire-compatible** with ASTM F3411. Key differences:

| Aspect                | ASTM F3411                          | GB 46750-2025                      |
|-----------------------|-------------------------------------|------------------------------------|
| Wire format           | 6 message types, 25-byte fixed      | Single variable-length packet, flag-bitmask chain |
| Identity              | Serial number **or** Session ID     | SN **and** UIN (registration ID) both mandatory |
| Authentication        | Auth message with signature pages   | Not defined                        |
| Operator location     | Optional                            | Mandatory (GCS position + type + altitude) |
| Accuracy reporting    | Not defined                         | 4 accuracy levels (horizontal / vertical / speed / timestamp) |
| Broadcast transport   | BT4 + BT5 + Wi-Fi NaN + Wi-Fi Beacon| Bluetooth 5.0+ + Wi-Fi (2.4 GHz / 5.8 GHz) |
| Coordinate system     | WGS-84 only                         | WGS-84 or CGCS2000                 |
| Drone classification  | Not defined                         | Mandatory (Micro → Large)          |
| Operation category    | EU Category & Class (optional)      | Open / Specific / Certified (optional) |
