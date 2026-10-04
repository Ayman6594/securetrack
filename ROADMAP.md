# SecureTrack Roadmap

Each phase ends with **exit criteria**: concrete, testable, demonstrable. Don't start the next phase
until they're met. After each phase, update `docs/THREAT_MODEL.md` and write at least one ADR.

Legend: ⬜ not started · 🟦 in progress · ✅ done

---

## Phase 0: Foundations ⬜
**Goal:** a clean, reproducible workspace and a first threat model.
- [ ] Linux dev environment (WSL2 Ubuntu or VM), Git, SSH keys, Python 3.12+, Docker
- [ ] Repo scaffold, `.gitignore`, pre-commit hooks (ruff, gitleaks)
- [ ] Minimal CI (lint + secret scan)
- [ ] Threat model v1 and wire contract v1 reviewed
- [ ] ADR-0001 (Ed25519 over shared HMAC)

**Exit:** a commit to `main` triggers green CI; a planted fake secret is blocked by pre-commit.
**You learn:** Linux, Git workflow, security thinking, documentation habits.

## Phase 1: Simulator + Gateway ⬜
**Goal:** a believable tracker and radio, with no backend yet.
- [ ] Python library implementing the frame encoder/decoder from the wire contract
- [ ] Tracker simulator: Ed25519 keypair, counter, battery drain model, status machine, GPS-like paths
- [ ] `Transport` interface + `SimTransport` with MTU enforcement, loss, duplication, reordering, latency
- [ ] Gateway: decode, validate shape, buffer offline, batch, retry with backoff
- [ ] Unit tests including **golden test vectors** shared with future firmware

**Exit:** a 1-hour simulated run produces correct frames despite 20% loss; gateway never crashes on malformed input (fuzzed).
**You learn:** Python packaging, BLE constraints, binary protocols, networking fundamentals, testing.

## Phase 2: Backend + Database ⬜
**Goal:** authenticated, validated ingestion.
- [ ] FastAPI app, PostgreSQL, Alembic migrations
- [ ] User registration/login (Argon2, JWT access + refresh)
- [ ] Gateway tokens (hashed at rest, scoped)
- [ ] `POST /telemetry` with signature verification and three-layer replay protection
- [ ] Authorization tests: users can never read another user's devices (IDOR tests)

**Exit:** replayed, tampered, and unsigned frames are all rejected with tests proving it.
**You learn:** API design, SQL, authN vs authZ, applied crypto.

## Phase 3: Provisioning + Revocation ⬜
**Goal:** secure device onboarding.
- [ ] One-time provisioning tokens (hashed, short TTL, single use)
- [ ] Public-key registration and owner binding
- [ ] Suspend / revoke / re-provision flows, all audited
- [ ] Immutable-style audit log (append-only table, no UPDATE/DELETE grants)

**Exit:** token reuse, expired token, and revoked-device telemetry all fail; audit log shows every attempt.
**You learn:** key lifecycle, trust-on-first-use pitfalls, audit design.

## Phase 4: Containerize + CI/CD ⬜
**Goal:** one-command run and a security-gated pipeline.
- [ ] Multi-stage, non-root Dockerfiles; `docker compose up` full stack with Caddy TLS
- [ ] GitHub Actions: ruff, mypy, pytest + coverage, Bandit, pip-audit, Trivy, gitleaks
- [ ] Pipeline fails on high/critical findings; branch protection on `main`

**Exit:** a PR with a vulnerable dependency or committed secret is blocked.
**You learn:** Docker hardening, CI/CD, DevSecOps.

## Phase 5: Observability ⬜
**Goal:** see what the system is doing, and when someone attacks it.
- [ ] Prometheus metrics (ingest rate, rejects by reason, auth failures, device heartbeat age)
- [ ] Structured JSON logs to Loki; correlation IDs
- [ ] Grafana dashboards + alert rules
- [ ] **No secrets or precise locations in logs** (tested)

**Exit:** each rejection reason is visible on a dashboard within 15 seconds.
**You learn:** metrics vs logs, alert design, avoiding alert fatigue.

## Phase 6: Cloud + Infrastructure as Code ⬜
**Goal:** deploy reproducibly.
- [ ] Terraform: VM or container service, managed Postgres, secrets manager, network rules
- [ ] Real DNS and TLS certificates; least-privilege IAM
- [ ] Remote Terraform state, `terraform plan` in CI, cost guardrails
- [ ] Teardown script (avoid surprise bills)

**Exit:** `terraform apply` from scratch produces a working HTTPS deployment; `destroy` removes everything.
**You learn:** cloud networking, IAM, secrets, IaC, cost awareness.

## Phase 7: Security Validation + Incident Drills ⬜
**Goal:** attack your own system and write it up.
- [ ] Scripted drills in `scripts/`: replay, rogue gateway, credential stuffing, silent device
- [ ] OWASP ZAP baseline scan, API fuzzing (e.g. Schemathesis)
- [ ] 4+ incident reports in `docs/incidents/` (timeline, detection, root cause, fix, lessons)
- [ ] Threat model v2 with residual-risk assessment

**Exit:** every drill is detectable in Grafana/Loki and has a runbook.
**You learn:** offensive thinking, detection engineering, post-mortem writing.

## Phase 8: Real Hardware ⬜
**Goal:** swap the simulator for an ESP32 (roughly $5 to 10).
- [ ] Firmware generating/storing the Ed25519 key (flash encryption, secure boot reviewed)
- [ ] Pass the golden test vectors from Phase 1
- [ ] `BleTransport` (e.g. Bleak) implementing the same interface
- [ ] Document what the simulator got wrong

**Exit:** a real board's frames are accepted by the unchanged backend.
**You learn:** embedded constraints, BLE GATT, hardware key storage.

## Phase 9 (stretch): Finder Network ⬜
- [ ] Rotating identifiers derived from a shared secret and time epoch
- [ ] End-to-end encrypted location reports (finder can't read; server can't read)
- [ ] Anti-stalking simulation: "unknown tracker travelling with you" alert
- [ ] ADR on privacy/abuse trade-offs

---

## Portfolio definition of done
- `docker compose up` gives a working demo
- Green CI with security gates
- Threat model, architecture diagram, ≥ 3 incident reports
- Tests covering all crypto and auth paths
- README with demo GIF and honest limitations

## Suggested pace (about 6 to 8 h/week)
Phases 0 to 1: weeks 1 to 3 · Phases 2 to 3: weeks 4 to 7 · Phases 4 to 5: weeks 8 to 11 ·
Phases 6 to 7: weeks 12 to 16 · Phase 8+: when hardware arrives.
