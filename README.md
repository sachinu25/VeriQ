# VeriQ

**VeriQ** is a research-driven, diagnosis-informed home network QoS controller. The goal is to detect likely congestion causes, make bounded and auditable changes, verify their effects, and fall back safely when evidence is insufficient.

> **Development status:** foundation phase. This repository is not yet a working VeriQ prototype. The existing `qosmate-main/` directory is retained as a reference/comparison snapshot; it must not be treated as the VeriQ controller or as proof that VeriQ requirements are implemented.

## Design principles

- **Diagnose before acting:** increased latency alone is not sufficient evidence that the local WAN queue is the cause.
- **Safety first:** every proposed change must pass a pure action checker before an enforcer can apply it.
- **Bounded control:** rate changes must have explicit limits, freshness requirements, and fallback budgets.
- **Verify after applying:** read back owned kernel state and record whether an action improved, did not improve, or produced inconclusive evidence.
- **Fail safely:** loss of the controller must not silently leave an unverified control loop running.
- **Reproducible evidence:** performance claims require raw measurements, environment details, repeatable scripts, and fair baselines.
- **No live-router experimentation during early development:** initial work belongs in an isolated Linux network-namespace lab.

## Repository map

- `qosmate-main/` — existing QoSmate reference snapshot. Preserve its provenance and licensing; do not blindly copy or rebrand it.
- `cmd/veriqd/` — VeriQ daemon entry point (currently a foundation skeleton).
- `internal/model/` — typed domain models for observations (currently minimal).
- `docs/VERIQ_IMPLEMENTATION_PLAN.md` — staged implementation plan and acceptance gates.
- `lab/` — reproducible network experiments (to be built next).

## Build the current foundation

Requires Go 1.23 or later.

```bash
go test ./...
go run ./cmd/veriqd
```

The current command is deliberately a no-op foundation: it does not inspect or modify host networking, run `tc`/`nft`, or change router settings.

## Before calling this a prototype

The next gates are: recover the authoritative VeriQ design/specification into the repository; create and test the isolated lab; establish a reproducible static bidirectional CAKE baseline; then implement observations, diagnosis, the pure action checker, bounded control, readback/reconciliation, watchdog/SAFE behavior, and evidence receipts. Do not connect adaptive decisions to live kernel enforcement until the relevant lab and safety tests pass.
