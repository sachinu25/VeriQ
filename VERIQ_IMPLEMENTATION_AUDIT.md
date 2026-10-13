# VeriQ implementation audit

**Audit date:** 13 October 2026 (IST)  
**Repository:** https://github.com/sachinu25/VeriQ, main at 0ef6ebd (first commit)  
**Specification:** local VeriQ_Handbook_v5.pdf, 361 pages; Appendix C, pp. 327–354, is authoritative; Appendix D, pp. 355–357, retains v4.1 contracts except where Appendix C changes them.  
**Scope:** read-only inspection and shell syntax checks. No router configuration was changed. This report is the only new repository file.

## 1. Executive verdict

**The research direction is sound, but the public repository is not yet a VeriQ prototype.** It contains a QoSmate package snapshot under qosmate-main. There is no Go daemon, VeriQ CLI, controlled test lab, VeriQ nft/tc renderer, Action Checker, evidence pipeline, OpenWrt VeriQ package, or UI. No VeriQ action can currently be built or demonstrated from this repository. The existing QoSmate code is a useful comparative reference and a source for understanding OpenWrt plumbing, subject to its licence; it is not evidence that the VeriQ contracts work.

The essential innovation proposed in Appendix C §§0–9 is **diagnosis before changing a rate, with bounded, checked, auditable actions and a static-CAKE safe state**. CAKE, nftables classification, an IFB, and delay-based autorate already exist elsewhere. The differentiator will have to be measured against tuned static SQM, cake-autorate, and QoSmate, especially in conditions where delay is outside the access bottleneck. The claim “never worse than static SQM” is a design invariant and test goal, not a result.

**Largest technical risk:** false diagnosis or a faulty enforcement path that cuts capacity for remote/Wi-Fi/CPU/probe faults, or damages connectivity on a failed/partial kernel update. The second immediate risk is the missing reproducible lab: the handbook refers to lab/ and evidence/ files that are absent from this GitHub repository.

## 2. Evidence and limits

I cloned the public main branch and inspected its complete 15-file tree and sole commit. I read the checked-in shell, UCI, Makefile and file-fixup files; traced startup, qdisc/classification/autorate paths; and ran syntax-only checks. I extracted the full handbook text and checked the authoritative Appendix C, Appendix D, Day-0 findings (handbook §4.9), delivery plan (§18), implementation and evaluation sections. I did not run any script that changes networking. The GitHub app connector returned USER_NOT_LOGGED_IN; public GitHub clone succeeded. This audit is therefore of the public main commit, not private branches or unpushed work.

**Status vocabulary:** BUILT AND TESTED requires a meaningful runtime test; IMPLEMENTED BUT UNVERIFIED means code exists but no runtime proof; PARTIALLY IMPLEMENTED means only a subset or a different design exists; DESIGNED ONLY means specified in the handbook without repository code; MISSING means neither the specified repository artifact nor implementation is present. Shell syntax passing is not a runtime test.

### Repository inventory

| Path | Actual role and finding |
| --- | --- |
| qosmate-main/Makefile | OpenWrt package named qosmate v0.5.48, declared GPL-3.0-or-later. Build/Prepare and Build/Compile are empty. Install copies only qosmate.sh, qosmate init, an empty hotplug file, and UCI config. It does not install the checked-in autorate scripts/service, tc distributions, binary tc extension, or LuCI frontend. |
| qosmate-main/etc/config/qosmate | QoSmate UCI defaults. WAN eth1, down/up rates 90000/45000 kbit/s, ROOT_QDISC=hfsc, autorate disabled. Appendix C §4.3 makes CAKE the VeriQ default. |
| qosmate-main/etc/qosmate.sh | QoSmate nftables DSCP classification, WAN qdisc setup, IFB ingress redirection, CAKE/HFSC/HTB/hybrid branches and rate limits. It writes table inet dscptag, restores DSCP using tc ctinfo dscp 63 128, and does not implement VeriQ's ct CLASS/PROV/SANCT mark layout or Action Checker. |
| qosmate-main/etc/qosmate-autorate.sh | QoSmate shell autorate loop. It pings three reflectors, averages RTT, uses a lower-only idle baseline, interface-byte rates and latency thresholds, then changes upload/download rates. It has a simple idle-direction guard, but no independent evidence groups, WAN/REMOTE/WIFI/CPU labels, checker, rolling fallback budget, or receipt assessment. |
| qosmate-main/etc/qosmate-autorate-tc.sh | QoSmate tc rate updates; CAKE uses tc qdisc change. A useful reference, but no VeriQ read-back/version/ownership contract. |
| qosmate-main/etc/init.d/qosmate and qosmate-autorate | QoSmate procd/service, updater, integrity, config migration, setup and health functions. VERSION=dev makes start_service call update -f; missing files can also trigger upstream fetch. This is a dependency on upstream network downloads, not a self-contained VeriQ package. |
| qosmate-main/etc/hotplug.d/iface/13-qosmateHotplug | Zero-byte file in Git. The init script generates a hotplug script at runtime. The package installs the empty file. |
| qosmate-main/usr/lib/tc/*.dist and m_xt.so | Five distribution data files and one ELF x86-64 shared object (e_machine 0x3e). No source, licence notice, build recipe, or architecture variants for the binary are checked in. The Makefile does not install them. |
| qosmate-main/file_fixups.txt | QoSmate distribution path mapping, not a VeriQ migration or importer. |

**Absent at this commit:** README, LICENSE text, docs/, lab/, evidence/, go.mod, Go/Python source for VeriQ, tests, CI workflows, lockfiles, OpenWrt VeriQ packaging, LuCI files, and any VeriQ-specific history. The local handbook claims Day-0 files under lab/ and evidence/day0_sandbox_2026-10-11/; those files are not in the audited GitHub repository. The handbook's docs/11_VeriQ_v5_Final_Design.md and docs/09 contract are likewise absent as standalone files, though their contents are embedded in Appendices C and D.

### What was actually validated

* The six shell files, including the empty hotplug file, passed sh -n using Git for Windows sh. This checks syntax only.
* The repository has one commit (0ef6ebd) and a clean main checkout before this report.
* No Go build or go test is possible: no go.mod or Go files. No repository test command or CI exists.
* No OpenWrt SDK/buildroot or safe Linux namespace lab was provided here. QoSmate package build, installation, nft/tc behavior, bidirectional tin mapping, and autorate operation remain **UNVERIFIED**.
* The handbook's Day-0 266.1 ms upload added p95 FIFO result is a **preliminary, single, emulated, IPv4-only run**, with no base RTT applied and no CAKE/fq_codel arm (§4.9). Its raw evidence is not in this repository. Do not treat it as a VeriQ result or a comparison against CAKE.

## 3. Research-to-code traceability

The “next action” column names the smallest concrete artifact or verification step. All paths refer to the audited repo or proposed paths from Appendix C §14.1.

| Requirement and reference | Corresponding repository evidence | Status | Gap / exact next action |
| --- | --- | --- | --- |
| Go veriqd, veriqctl, typed messages and deterministic replay (C §§3, 11.4, 14.2) | No go.mod, cmd/, internal/ | DESIGNED ONLY | Create go.mod, cmd/veriqd, cmd/veriqctl, internal/model and internal/clock; make go test ./... pass with a fake-clock smoke test. |
| UCI config validation, seven operating states, monotonic config version (C §§8.2, 12.3, 14.3) | QoSmate config and init use different schema | DESIGNED ONLY | Add internal/config and internal/state; implement INIT/SHADOW/LIVE/DEGRADED/HOLD_LINKDOWN/SAFE/DISABLED and validation. Do not relabel QoSmate UCI as VeriQ. |
| tc/nft/conntrack/Wi-Fi observations with timestamps (C §§3.1, 6.1, 7.1) | QoSmate script reads limited interface counters; no VeriQ observer | PARTIALLY IMPLEMENTED | Add internal/observe with injectable sources and immutable Snapshot; test stale/missing signals. |
| 10 pps qualified near/far probes, per-reflector baseline and median/quorum (C §§6.2–6.4, C8/C9) | QoSmate autorate averages pings; lower-only baseline (autorate.sh) | PARTIALLY IMPLEMENTED | Build probe manager and 10-min windowed low-load baselines; test route change, loss, outlier and RTT-only direction cases. |
| Capacity estimate μ/σ, censored vs lower-bound samples and memory (C §§6.1, 6.6–6.7; D A.3) | No VeriQ estimate package; QoSmate uses achieved rate | DESIGNED ONLY | Add internal/estimate; never learn capacity from application-limited throughput; golden-vector tests. |
| Diagnosis OWN_QUEUE/WAN_LIKELY/REMOTE/WIFI/CPU/PROBE/MIXED/UNKNOWN, L0–L3 (C §7) | No code; QoSmate has latency threshold plus idle-direction guard | DESIGNED ONLY | Implement pure internal/diagnose with independent evidence groups and permission table; fault tests A6/A7/A17/A32. |
| Rate FSM, fast cut C1, exploration, floor and fallback budget (C §§6.5, 7.3, 8.1; D A.2, A.4, A.7) | QoSmate calculate_new_rate uses 85% down/102% up defaults; no episode budget | PARTIALLY IMPLEMENTED | Implement internal/control with fake clock; L2 fresh G-hat target max(S_min, 0.90×G-hat), otherwise ×0.75; test budget persistence and 100→20 Mbit/s case. |
| Pure Action Checker and ACCEPT action/version token (C §8.1; D A.1) | No checker or token; QoSmate calls tc directly | DESIGNED ONLY | Implement internal/checker.Check as a pure function and refusal-code table tests; enforcer must reject untokened actions. |
| Static CAKE on WAN and IFB, both directions and IPv4/IPv6 (C §§4.1–4.4, 12.6) | qosmate.sh setup_cake and IFB path exist, but default is HFSC; runtime untested | IMPLEMENTED BUT UNVERIFIED (QoSmate only) | Build VeriQ-owned CAKE renderer; prove both qdiscs and per-tin counters on full Linux kernel, in both IP families. |
| nft classification, CLASS/PROV/SANCT bits 24–30, ctinfo cpmark, fwmark tins (C §§4.2, 5.2; D A.8) | QoSmate table dscptag stores DSCP in ct mark and uses ctinfo dscp | PARTIALLY IMPLEMENTED (different plumbing) | Implement table inet veriq; preserve unrelated mark bits; test upload and download marks and CAKE 1–4 tin mapping on target kernel. |
| QoSmate-compatible rules, sets, heuristics, ML abstention, eligibility envelope (C §§5.1–5.7) | QoSmate rules/sets and rate limits; no VeriQ provenance, abstention or sanctions | PARTIALLY IMPLEMENTED | For P0 start with explicit user rules and BE default, then lease/envelope; defer ML to P1 per C §18. Build importer instead of copying/rebranding. |
| Authenticated, expiring intent leases and admission (C §5.8; D A.1) | No lease implementation | DESIGNED ONLY | Add internal/lease and kernel timeout sets; test expiry during daemon outage, reboot/NTP, admission and per-flow envelope. |
| Owned-object reconciliation, read-back, foreign-object handling (C §8.3; D A.5) | QoSmate start/stop deletes qdiscs/table; no VeriQ ownership/version | DESIGNED ONLY | Add desired/observed hashes, own only veriq table/handles/IFB, adopt matching state and stop on foreign qdisc. |
| Separate watchdog and SAFE fallback (C §§8.2–8.4) | QoSmate procd autorate service, but no VeriQ watchdog or heartbeat | DESIGNED ONLY | Add openwrt/veriq watchdog POSIX loop and safe.env; kill daemon in lab and prove SAFE within 12 s while forwarding continues. |
| Receipts, explain API, decision JSONL, telemetry rings (C §§9, 11.4, 14.3) | QoSmate logs/state CSV are not receipts; no VeriQ socket/API | DESIGNED ONLY | Add internal/receipt, internal/api, internal/telemetry; report IMPROVED/NOT_IMPROVED/INCONCLUSIVE from pre/post windows. |
| OpenWrt VeriQ package, UCI/procd/hotplug, ACL, LuCI (C §§11–12) | QoSmate Makefile/init only; LuCI referenced by updater but absent | PARTIALLY IMPLEMENTED (QoSmate reference) | Build openwrt/veriq package and minimal status LuCI after the core; first pass must install offline in x86-64 VM. |
| Reproducible lab, traffic generators, baselines (C §§3.3, 14.4, 15; handbook §4.9) | No lab/ or evidence/ in repository | DESIGNED ONLY | Recover exact Day-0 source/evidence if available; add capability-checked netns lab, raw logs and static CAKE comparison. |
| Unit/integration/fault tests and evidence ledger (C §§10, 14.4–15) | No tests or .github/workflows | DESIGNED ONLY | Add go test/race CI, netns tin self-test, fault injections and claims regeneration; preserve failed runs. |
| Security, privacy and provenance (C §§5.1, 13) | QoSmate metadata GPL; no repo licence file; prebuilt x86-64 .so | PARTIALLY IMPLEMENTED (metadata only) | Establish provenance and licence notices; keep QoSmate comparison separate. Add root-only socket/ACL and data-retention tests before user-facing features. |

No VeriQ requirement in this matrix qualifies as BUILT AND TESTED. The QoSmate rows are not credited as VeriQ implementation.

## 4. Architecture mismatches and decisions

1. **Wrong product boundary.** The repository is a QoSmate snapshot, while C §14.1 specifies a Go monorepo and distinct OpenWrt packages. Preserve qosmate-main as a pinned reference/baseline; implement VeriQ in new directories.
2. **Different classification carrier.** QoSmate stores DSCP and restores it with ctinfo dscp. VeriQ requires CLASS/PROV/SANCT in ct mark bits 24–30 and ctinfo cpmark on download (C §4.2). Tin mapping and the exact cpmark syntax must be self-tested; neither can be assumed from a code line.
3. **Different controller.** The checked-in autorate averages reflector RTT and reduces rates on delay. VeriQ requires median/quorum, independent evidence, per-direction attribution, C1 fast-cut and bounded fallback. Reusing the QoSmate loop as the VeriQ controller would defeat the central hypothesis.
4. **Missing safety boundary.** No checker token, desired-state version, read-back, owned-object reconciler, watchdog or SAFE state. QoSmate's direct qdisc deletion on stop is unsuitable for VeriQ's foreign-object ownership contract.
5. **Not self-contained packaging.** Makefile's installed files omit needed autorate files; init may fetch upstream code on first start. The checked-in hotplug file is empty. No LuCI package, signatures, SDK build, or package tests exist.
6. **Scope tension.** Appendix C §0 describes full QoSmate parity/product readiness, while §18 explicitly places ML, HFSC/HTB, household page, app catalogue and 3D twin in P1. Follow §18 for the Round-2 P0 critical path. State omissions openly rather than calling a narrower demo full parity.
7. **Licence boundary.** GPL-3.0-or-later is declared in the QoSmate Makefile. There is no LICENSE text in this repository and the binary's provenance/build is undocumented. Resolve upstream attribution and redistribution conditions before incorporating any QoSmate code into VeriQ. A separate clean implementation can use the public behavior as a reference.

## 5. Readiness by dimension

| Dimension | Assessment and evidence |
| --- | --- |
| Research/design | Extensive and actionable: Appendix C specifies interfaces, states, contracts, scenarios and priorities. Several kernel-specific assumptions remain unverified; all thresholds are starting parameters. |
| VeriQ implementation | Essentially absent in this public commit. QoSmate shell code supplies a reference, not a VeriQ module. |
| Buildability | Shell syntax passes. QoSmate OpenWrt package build is unverified and package manifest is incomplete. VeriQ cannot be built. |
| Testability | No committed lab, test suite, CI or raw evidence. The handbook's prior Day-0 experiment cannot be reproduced from this checkout. |
| Demo readiness | No end-to-end VeriQ path or repeatable demo. A QoSmate experiment might be assembled externally, but is unverified here. |
| Product readiness | Absent: no VeriQ package, security boundary, migration, signed distribution, hardware soak or lifecycle tests. |

**Usable today:** the v5 design, QoSmate source as a study/comparator, and a syntax-valid shell snapshot. **Not usable today:** any claimed VeriQ adaptive decision, feature parity, performance result, safety guarantee, or release package.

## 6. First five tasks and minimum credible prototype

1. **Recover and commit the missing research artifacts** into docs/ and lab/ (especially docs/11, docs/09 Appendix A, quickstart_bufferbloat.sh, summarize.py, lab/README.md and raw Day-0 evidence). Preserve original history/provenance. If unavailable, implement a new minimal lab and label the old raw data unavailable. Do not copy a PDF into the repo merely as a substitute for source-controlled contracts.
2. **Reproduce the lab on Ubuntu 24.04/full kernel.** Verify sch_cake, sch_netem, ifb and act_ctinfo; create a real 20 ms base RTT, run FIFO versus static CAKE in both directions and both IP families; archive raw logs and environment fingerprint.
3. **Create a buildable Go skeleton** with config schema, immutable Snapshot, fake clock, state model and JSONL decision log; make go test ./... and CI pass before network privileges enter the daemon.
4. **Build static VeriQ-owned CAKE enforcement and tin self-test.** Keep dynamic control in SHADOW. Prove nft marks, WAN/IFB tins, read-back and cleanup ownership. A misclassified download invalidates later controller experiments.
5. **Add pure diagnosis, controller and checker in replay tests**, then connect live actions behind the checker and watchdog. Demonstrate a capacity drop and a remote-delay fault, with accepted/refused actions and receipts.

**Smallest credible end-to-end prototype:** one full-kernel netns router; static bidirectional CAKE; a Go control loop that observes timestamped WAN/IFB counters and healthy far/near probes; labels WAN_LIKELY versus REMOTE/UNKNOWN; a pure checker admitting only bounded cuts; an enforcer with read-back; an independent SAFE watchdog; JSONL decisions/receipts; and a runner that compares against tuned static CAKE and cake-autorate in S2 and S6. Explicit rules and a single temporary lease can follow after that spine. No ML, 3D twin, advanced UI, multiple scheduler backends, public feed or product claim is needed to prove the core idea.

**First coding task after this audit:** recover or create lab/quickstart_bufferbloat.sh plus lab/summarize.py and lab/README.md, with a capability preflight that fails if netem/CAKE/IFB are absent, records the *applied* base RTT, and writes raw ping/iperf3/environment files. The first test should prove FIFO and CAKE arms both run and that summary.csv regenerates from raw files. This is the earliest missing dependency for meaningful VeriQ work.

## 7. Ordered implementation roadmap

All network commands below are for an isolated Ubuntu/OpenWrt lab, **not this Windows machine or a household router**. Effort is an estimate for one engineer familiar with Go but learning kernel QoS; a small team can parallelise after the interfaces are stable. “P0” means the Round-2 critical path from C §18, though the minimum prototype narrows its breadth.

| Phase (priority; effort; depends on) | Exact files/task and expected behavior | Command and verification / definition of done |
| --- | --- | --- |
| 0. Source and CI (P0; 1–2 days; none) | Add README.md, docs/11_VeriQ_v5_Final_Design.md, docs/09 contract, go.mod, cmd/veriqd, cmd/veriqctl, internal/model, internal/clock, .github/workflows/ci.yml. Retain qosmate-main unchanged as a reference. | go test ./...; git status --short; CI runs on a fresh clone. Done when empty daemon/CLI build, tests pass, source contracts are committed and provenance is documented. |
| 1. Full-kernel lab (P0; 2–4 days; 0) | Recover/add lab/quickstart_bufferbloat.sh, summarize.py, README.md, scenario config and raw evidence manifest. Four netns path client→router→ISP→server; capability check before setup and cleanup by owned namespace names only. | sudo modprobe sch_cake sch_netem sch_tbf ifb act_ctinfo; sudo DURATION=15 lab/quickstart_bufferbloat.sh fifo cake. Done when applied 20 ms base RTT, IPv4/IPv6 status, kernel/tools, raw logs and CAKE/FIFO results are recorded and summary regenerates. |
| 2. Static datapath (P0; 4–7 days; 1) | Add internal/enforce/cake and nft renderer, lab/tin_selftest.sh. Install owned WAN CAKE + IFB CAKE with table inet veriq; keep unrelated marks/foreign qdiscs untouched. Test class 1–4 in both directions before adaptive changes. | sudo lab/tin_selftest.sh; tc -s -j qdisc show dev WAN and ifb-veriq; nft -j list table inet veriq. Done when per-tin counters and packet captures agree for IPv4/IPv6, and disabling removes only owned objects. |
| 3. Daemon observations (P0; 3–5 days; 0,2) | Add internal/config, state, observe/tcstats, telemetry; immutable 250 ms Snapshot with sample ages; basic veriqctl status -j and log. | go test ./...; go test -race ./...; run read-only observer in lab. Done when missing/stale stats are explicit and replaying recorded inputs yields identical snapshots. |
| 4. Probe/estimate (P0; 5–8 days; 1,3) | Add internal/probe and estimate; near-hop plus far pool, qualification, median/quorum, low-load windowed baselines, G-hat and censored μ/σ. Treat failed probes as missing. | go test ./internal/probe ./internal/estimate; inject route change, reflector outlier and application-limited load. Done when each case has expected evidence and no false capacity sample. |
| 5. Diagnosis/control/checker (P0; 5–8 days; 3,4) | Add internal/diagnose, control, checker. Implement permission table, C1 target, floor, 1.04 exploration, 5% fallback, 10% rolling budget, action/version tokens. Start in SHADOW. | go test -race ./internal/diagnose ./internal/control ./internal/checker; replay S2/S6/A17/A32 vectors. Done when all refusal codes, state transitions and budget persistence pass, and no unapproved action reaches enforcement. |
| 6. Apply/reconcile/SAFE (P0; 4–7 days; 2,5) | Add internal/reconcile, owned-object read-back, openwrt/veriq watchdog and state files. Apply accepted actions, compensate failed steps, preserve kernel forwarding on daemon exit. | In lab: kill veriqd, delete own nft table, inject foreign qdisc; inspect tc/nft and decision log. Done when SAFE is reached within the 12 s target after kill, owned nft table is restored, foreign objects are not overwritten, and rates remain bounded. |
| 7. Rules and temporary intent (P0 narrow; 4–7 days; 2,5,6) | Add internal/classify and lease, initial explicit rule compiler, admission and timeout sets; enforce BE default and no priority from uncertainty. Defer app catalogue/ML. | Test precedence, expired/rebooted lease, sanctioned bulk flow and both IP families. Done when two-lease/one-device admission and first-packet marking work without daemon packet handling. |
| 8. Receipts/API/minimal UI (P0 narrow; 3–6 days; 3,5–7) | Add internal/receipt, api and minimal luci-app-veriq status/autorate page or a local dashboard for the demo; show state, evidence, action, verdict, read-back and outcome. | veriqctl status -j; veriqctl explain rate down; receipt fixture tests. Done when an accepted cut and a rejected remote-delay cut have complete, non-causal explanations and raw references. |
| 9. Evaluation/CI (P0; 7–12 days; 1–8) | Add lab/scenarios, run.py, summarize.py, claims.yml and pinned baseline configs. S1/S2/S4/S6/S7/A6/A7/A9/A14/A32 first; H0 smoke, then n=5 pilot and pre-set final n. | Run each arm in isolation; regenerate figures; go test -race ./...; CI with privileged netns runner if available. Done when static SQM, cake-autorate, QoSmate and VeriQ configurations/raw data are archived; failed runs retained; confidence intervals use runs, not packets. |
| 10. OpenWrt package/demo (P0; 5–10 days; 2–9) | Add openwrt/veriq Makefile/files, UCI/procd/hotplug and minimal rpcd ACL; package x86-64 VM first, then Pi. Demo script resets, injects fault, shows action/receipt, kills daemon, restores. | OpenWrt SDK package build; install in QEMU; health/selftest; run demo twice. Done when package installs offline with all files, no upstream self-download, and every demo number points to raw evidence. |
| 11. Parity and product expansion (P1/P2; weeks; 10) | P1: importer, QoSmate-compatible rules/sets, HFSC/HTB, ML+conformal, app catalogue, household UI, Hindi, H2/H3, bench, UX, twin. P2: exporter, signed feed/apk, prplOS/RDK-B, hybrid/MQ-CAKE, long pilots. | For each feature add contract test, netns/hardware test, docs and an evaluation claim before calling it complete. P1 exit is feature parity plus measured behavior; P2 exit depends on real deployment and signed release evidence. |

**Dependency rule:** do not integrate the controller with live tc until static bidirectional CAKE and the checker tests pass. The first genuinely end-to-end checkpoint is phases 0–6 plus scenarios S2 and S6; leases/UI deepen the demo but cannot rescue an unverified datapath.

## 8. Experimental and claims plan

1. **Day-0 correction.** Use a full kernel with sch_cake, sch_fq_codel, sch_netem and IFB. Verify the applied 20 ms base RTT from idle ping and tc read-back, not a requested environment variable. Replicate the original 20/5 Mbit/s, 300 ms ISP FIFO condition for historical comparison, then run the four arms (FIFO, short FIFO, fq_codel, CAKE) in both IP families. Store raw ping, iperf3 JSON, tc stats, pcap where needed, environment and failures. Repeat at least 10 runs per arm before quoting a median/CI.
2. **Fair baselines.** Tune static SQM/CAKE, cake-autorate and QoSmate according to pinned versions and their own documentation. Archive configs, kernel, iproute2, OpenWrt image, CPU, NIC/offload settings and seeded fault schedule. Never let two enforcers run together. Include a controller-only comparison on the same CAKE datapath and a whole-system comparison.
3. **Ground-truth faults.** Access capacity drop (S2/A1), remote-only delay (S6/A6), one bad reflector (S7), Wi-Fi retries/low station rate, probe loss/quorum failure, CPU softirq saturation (A17), and ICMP-only delay at the near hop (A32). Label diagnosis from injected fault logs, not from VeriQ output. Test no-cut or bounded fallback where control is futile; test fast confirmation when capacity really drops.
4. **Metrics.** Primary: count, magnitude and duration of inappropriate interventions in control-futile conditions. Co-primary: settling *confirmation* time, target no more than cake-autorate +1 s. Also measure added delay p95 up/down, receiver goodput, WebRTC freezes/concealment, DASH stalls, game RTT p99, page load, Jain fairness, bulk leftover share, recovery, CPU/RSS, checker rejections and SAFE timing. The bulk non-inferiority target is comparator minus 5 percentage points. These are targets, not observed facts.
5. **Statistics and honesty.** The run is the unit. Start with five-run pilots, then choose n for CI width; counterbalance order, use bootstrap CIs, Holm correction for secondary outcomes, and preserve censored/missing/failed runs. Keep labels “measured”, “target”, “starting parameter”, “illustrative”, and “hypothesis” distinct. Receipts show association and concurrent-change caveats; they cannot establish causality alone.

## 9. Risks, assumptions, unresolved questions

* The public repository may not include local branches or private work. This report describes only main at 0ef6ebd. If a fuller repo exists, re-run the matrix there.
* Kernel support for CAKE fwmark values, ctinfo cpmark and nft meter/set features is target-dependent; the Stage-1 self-test is decisive (C §§4.1–4.2, 12.6, O2/O3).
* The full v5 P0 plan is ambitious for a solo engineer before 13 Dec 2026. Keep the non-negotiable core (lab, static CAKE, diagnosis, checker, read-back, SAFE, baselines) and explicitly label deferred parity features. Appendix C O7 permits a Python lab controller if Go ramp-up blocks P0, with the same interfaces/golden tests and a Go port in P1.
* The handbook's “static SQM worst case” goal can be violated by a wrong static rate, kernel incompatibility, offload bypass, CPU saturation or a broken rollback. Only adversarial testing can support a bounded claim.
* Resolve exact source/licence for qosmate-main and m_xt.so, the appropriate VeriQ licence (C O5), and the location of the missing lab/raw evidence before publication.
* A production or superiority claim requires hardware runs, signed packages, upgrade/rollback tests, security/privacy review and repeated head-to-head measurements. None exist in this repository today.
