# VeriQ Implementation Plan

**Status:** working plan derived from the repository audit. This is not a replacement for the authoritative research/design document.

## Source-of-truth rule

The research handbook identifies `docs/11_VeriQ_v5_Final_Design.md` as the final design source and Appendix D as the place to preserve v4.1 contracts unless v5 explicitly changes them. Those source documents were not present in the audited public repository. Recover the originals from the research PDF and commit them before claiming full design traceability. Do not invent missing requirements or silently reinterpret the handbook.

## Current baseline

The public repository currently contains a `qosmate-main/` snapshot. It has useful QoS implementation material for study and comparison, but it does not provide the VeriQ diagnosis-first controller, action checker, evidence receipts, safety supervisor, reproducible lab, or VeriQ package. Keep it intact while VeriQ-specific components are built alongside it.

## Product contract

VeriQ should progress toward this loop:

1. Collect timestamped observations from independently injectable sources.
2. Validate freshness, quality, and provenance before using observations.
3. Estimate congestion/capacity without treating application-limited throughput as a capacity measurement.
4. Diagnose a likely cause (for example `OWN_QUEUE`, `WAN_LIKELY`, `REMOTE`, `WIFI`, `CPU`, `PROBE`, `MIXED`, or `UNKNOWN`) and confidence/level.
5. Produce a bounded proposed action.
6. Pass the proposal through a pure, deterministic Action Checker.
7. Let the enforcer apply only checker-approved actions with explicit ownership and expiry.
8. Read back kernel state and verify the effect.
9. Emit an auditable receipt: `IMPROVED`, `NOT_IMPROVED`, or `INCONCLUSIVE`.
10. Enter a defined safe/degraded state when evidence, connectivity, or enforcement health is inadequate.

Latency alone must never be treated as proof of local WAN congestion. An unknown diagnosis must not trigger an aggressive rate cut.

## State model to validate against the final design

The audit identified these candidate states: `INIT`, `SHADOW`, `LIVE`, `DEGRADED`, `HOLD_LINKDOWN`, `SAFE`, and `DISABLED`. Before implementation, compare this list with the recovered v5 design and preserve the exact required names/transitions.

## Phased roadmap and exit criteria

### Phase 0 — Design and build foundation
- [x] Preserve the existing QoSmate snapshot.
- [x] Add a minimal Go module, daemon entry point, domain-model starting point, and tests.
- [ ] Recover and commit the authoritative research/specification documents.
- [ ] Add CI for formatting, unit tests, and static checks.
- **Exit:** source requirements are traceable to the committed design; `go test ./...` passes in CI.

### Phase 1 — Reproducible isolated lab
- [ ] Add `lab/README.md`, a capability preflight, experiment runner, and summary generator.
- [ ] Run only in an isolated Linux network-namespace lab; fail closed if prerequisites are missing.
- [ ] Capture raw ping, iperf3, `tc` state/counters, environment, kernel version, and applied base RTT.
- [ ] Compare FIFO and static CAKE in both directions; do not claim a result until raw evidence regenerates the summary.
- **Exit:** another person can reproduce the experiment from documented commands and regenerate the same summary from raw data.

### Phase 2 — Static datapath
- [ ] Implement static bidirectional CAKE in the lab, including WAN egress and IFB ingress.
- [ ] Verify IPv4 and IPv6 behavior, CAKE tins/counters, kernel capabilities, cleanup, and readback.
- [ ] Keep the adaptive controller in SHADOW mode.
- **Exit:** tests prove the intended qdiscs are installed, observed, and cleaned up without touching unrelated interfaces or qdiscs.

### Phase 3 — Observation model and probes
- [ ] Define timestamped snapshots with explicit validity, source, and units.
- [ ] Add injectable probe/counter sources and a fake clock for deterministic tests.
- [ ] Implement median/quorum handling, outlier/loss treatment, route-change invalidation, and low-load baseline handling.
- **Exit:** deterministic tests cover stale samples, probe failure, outliers, route changes, and insufficient evidence.

### Phase 4 — Estimation and diagnosis
- [ ] Implement the final-design estimator and explicitly represent censored/lower-bound samples.
- [ ] Implement diagnosis labels and levels exactly as specified by the recovered design.
- [ ] Test remote delay, reflector faults, Wi-Fi retries, CPU/softirq pressure, and probe loss.
- **Exit:** injected non-WAN faults do not automatically become local-WAN congestion diagnoses.

### Phase 5 — Controller and Action Checker
- [ ] Implement the controller as a pure state transition from snapshot + state to a proposed action.
- [ ] Implement a pure `Check` function with explicit refusal codes and bounded action validation.
- [ ] Enforce freshness, minimum evidence, rate bounds, cooldowns, and fallback budgets from the final design.
- [ ] Replay historical/synthetic traces before enabling any action.
- **Exit:** all actions are rejected by default unless they satisfy the checker; tests cover every refusal code and state transition.

### Phase 6 — Enforcement, reconciliation, and SAFE
- [ ] Enforcer accepts only checker-approved, scoped actions.
- [ ] Own and reconcile only VeriQ-created objects; read back kernel state after changes.
- [ ] Stop if an unexpected/foreign qdisc or incompatible kernel capability is detected.
- [ ] Add an independent watchdog and tested SAFE behavior; controller failure must not interrupt ordinary forwarding.
- **Exit:** fault-injection tests show bounded recovery, safe cleanup, and no interference with foreign objects.

### Phase 7 — Classification and leases
- [ ] Implement the exact final-design mark layout and owned nftables table.
- [ ] Preserve unrelated mark bits and verify connection-mark/DSCP behavior.
- [ ] Start with explicit classification rules plus a best-effort default; defer ML-based classification until the baseline is validated.
- [ ] Add bounded, expiring leases and admission/timeout behavior if required by the final spec.
- **Exit:** packet classification and per-tin counters are verified in the lab for IPv4 and IPv6.

### Phase 8 — Receipts, API, and product integration
- [ ] Emit structured, timestamped decision/action/result receipts.
- [ ] Add minimal local API/telemetry with privacy-conscious defaults.
- [ ] Add OpenWrt packaging, UCI/procd integration, ACLs, and UI only after the core controller is safe and testable.
- **Exit:** a demo can explain why an action was proposed, why it was accepted/rejected, what changed, and what evidence followed.

### Phase 9 — Evaluation and release readiness
- [ ] Compare against tuned static SQM/CAKE and pinned, documented alternatives such as cake-autorate and QoSmate.
- [ ] Use repeated runs, raw evidence, confidence intervals, both IP families, and injected fault scenarios.
- [ ] Track inappropriate interventions, p95 added delay, goodput, fairness, recovery time, CPU/RSS, checker refusals, and SAFE timing.
- [ ] Resolve third-party source/binary provenance and licensing before packaging.
- **Exit:** publish claims only for results actually reproduced; distinguish measured results from targets and hypotheses.

## Safety gates

- Never test adaptive enforcement on the user's live router during early development.
- Never run multiple QoS enforcers simultaneously in one experiment.
- Never connect the controller to live `tc`/nft enforcement before the static datapath and checker pass their tests.
- Do not describe design-only features as implemented.
- Do not describe a one-off preliminary run as a verified performance comparison.
- If a requirement conflicts with the recovered final design, stop and document the conflict rather than guessing.

## Immediate next task

Build Phase 1's isolated lab harness first. It should detect missing dependencies, use namespaces, record the exact environment and applied base RTT, save raw results, and regenerate `summary.csv`. The harness must not alter host/root networking when preflight or namespace setup fails.
