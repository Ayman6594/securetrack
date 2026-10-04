# ADR-0001: Per-device Ed25519 signatures instead of shared HMAC secrets

- **Status:** Accepted
- **Date:** 2026-10-04

## Context
The backend must be convinced that a telemetry frame came from a specific registered device and was
not altered. Two mainstream options: a shared secret (HMAC) or asymmetric signatures.

## Options considered
| Option | Pros | Cons |
|---|---|---|
| HMAC-SHA256 with per-device shared secret | Small tag (can truncate to 8 to 16 B, fits legacy BLE), fast, simple | Server must store the secret: DB leak ⇒ attacker can forge any device; server can forge too (no non-repudiation) |
| **Ed25519 per-device keypair** | Server stores public key only; DB leak doesn't enable forgery; fast; supported by ESP32 libraries | 64-byte signature doesn't fit legacy advertising; slightly more compute |
| ECDSA P-256 | Hardware-accelerated on some chips | Nonce pitfalls, bigger/complex implementations |

## Decision
Use **Ed25519**, keypair generated on the device, private key never exported. Telemetry that carries a
signature travels over GATT or extended advertising. Legacy-advertising beacons (Phase 9) use rotating IDs instead.

## Consequences
- (+) Strong compromise containment (see Threat Model T1, T9).
- (+) Clean provisioning story: register a public key.
- (−) Frame is 89 B, so legacy-only BLE 4.x phones can't relay it as an advertisement.
- (−) Need golden test vectors so firmware matches the simulator exactly.

## Revisit if
Battery or latency measurements on real hardware make signing impractical.
