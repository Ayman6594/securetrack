# Wire Contract v1

The single most important interface in SecureTrack. The simulator, the gateway, the backend, and
the future ESP32 firmware must all agree on these bytes. **If this stays stable, hardware can be
swapped without touching the backend.**

## 1. Why a binary frame, and why two frame types

BLE radio payloads are tiny:

| Channel | Max payload | Notes |
|---|---|---|
| Legacy advertising | 31 bytes (≈ 24 usable after flags + manufacturer header) | Universally supported |
| Extended advertising (BLE 5) | up to 255 bytes per PDU | Needs BLE 5 on both ends |
| GATT notification | ATT MTU − 3 (up to 244 bytes after MTU negotiation) | Connected mode |

An **Ed25519 signature alone is 64 bytes**, so a signed frame **cannot** fit in a legacy
advertisement. This is a real design constraint, and the simulator enforces it:

| Frame type | Channel | Authenticated? | Purpose |
|---|---|---|---|
| **Telemetry frame (v1)** | GATT or extended advertising | Yes, Ed25519 signature | Status, battery, location |
| **Beacon frame** *(reserved, Phase 9)* | Legacy advertising, ≤ 24 B | Via rotating ID, not signature | "I'm nearby" presence |

## 2. Telemetry frame v1 (89 bytes)

All integers are **big-endian**. Python: `struct.Struct("!BBB4sIIiiH")` (25 bytes) + 64-byte signature.

| Offset | Size | Field | Type | Notes |
|---|---|---|---|---|
| 0 | 1 | `version` | u8 | Must be `0x01`. Reject anything else. |
| 1 | 1 | `flags` | u8 | See below |
| 2 | 1 | `battery_pct` | u8 | 0 to 100; values > 100 are invalid |
| 3 | 4 | `device_id` | bytes | Short ID assigned at provisioning |
| 7 | 4 | `counter` | u32 | Strictly increasing per device; never reused |
| 11 | 4 | `timestamp` | u32 | Unix seconds (valid until 2106) |
| 15 | 4 | `lat_e7` | i32 | Degrees × 10⁷ (≈ 1 cm resolution) |
| 19 | 4 | `lon_e7` | i32 | Degrees × 10⁷ |
| 23 | 2 | `accuracy_m` | u16 | Metres; `0xFFFF` = unknown |
| 25 | 64 | `signature` | bytes | Ed25519 over the signing input below |

**Flags byte**

| Bits | Meaning |
|---|---|
| 0 to 1 | Status: `0` idle, `1` moving, `2` low-power, `3` tamper |
| 2 | `location_valid` |
| 3 | `low_battery` |
| 4 to 7 | Reserved, must be `0` (receivers reject non-zero in v1) |

## 3. Signing

```text
signing_input = b"SecureTrack-v1\x00" || frame[0:25]
signature     = Ed25519_sign(device_private_key, signing_input)
```

- The **domain-separation prefix** prevents a signature made for one purpose being valid in another.
- The signature covers **every** field, including `device_id` and `counter`, so none can be altered.
- Verification uses constant-time library routines. Never compare signatures with `==` on secrets you derive yourself.

## 4. Backend JSON view (after the gateway decodes)

```json
{
  "device_id": "7f3a91c2",
  "counter": 10482,
  "timestamp": "2026-10-04T09:30:12Z",
  "battery_pct": 87,
  "status": "moving",
  "location": {"lat": 33.5731000, "lon": -7.5898000, "accuracy_m": 12},
  "frame_b64": "<original 89-byte frame, base64>"
}
```

The backend **always verifies the original frame bytes** (`frame_b64`), never trusting the gateway's
decoded JSON. That is what stops a compromised gateway from forging data.

## 5. Replay and validation rules (server side)

1. `version == 1` and reserved flag bits are zero, otherwise reject.
2. `device_id` exists and its state is `active`.
3. Signature verifies against the stored public key.
4. `counter > last_counter` for that device (then update atomically in the same transaction).
5. `|server_time − timestamp| ≤ 5 minutes` (offline-buffered frames use a separate, longer, configurable window, with the counter rule still enforced).
6. Latitude within ±90°, longitude within ±180°, battery ≤ 100.

Every rejection increments a labelled Prometheus counter and writes an audit/log entry.

## 6. Mandatory tests

- **Golden vectors:** fixed key + fixed fields → fixed 89-byte output. Firmware must reproduce them byte for byte.
- Round-trip encode/decode for random valid frames.
- Fuzz: random bytes of every length 0 to 300 never crash the decoder.
- Bit-flip every byte → signature verification fails.
- Replay of an old counter → rejected.
- Simulator refuses to send a frame exceeding the configured channel limit.

## 7. Open questions (resolve via ADRs)

- Device ID length (4 bytes ≈ collision-prone beyond ~65k devices; fine for a personal project).
- Real ESP32 has no GPS: should `location` come from the gateway instead? (Likely yes. Then the *gateway*
  attaches location in a separate, gateway-signed envelope.)
- Time source on a device without RTC: counter-only trust vs gateway-assisted time.
