---
layout: default
title: platform-sre OKR — 2026-05-23
nav_exclude: true
render_with_liquid: false
date: 2026-05-23
status: active
type: okr
loop: 2026-05-23
owner: teams/platform/sre
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/sre
  date: 2026-05-23
  status: active
  loop: 2026-05-23
  links: parent: board/okrs/2026-05-23-ceo-brief.md
-->
# SRE OKR — 2026-05-23

## Context

First build-mode loop since 2026-05-09. The 2026-05-10 loop was plan-only — charter-touch + runbook scope authored, no prose shipped. Between then and now, the supervisor binary at `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs` landed with a real `panic::catch_unwind` path (runtime spec §7.4); sim-farm's diff engine + Mode-A verdict contract (`teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`) stabilized with named `mismatches[].type` enum values and a 3-exit-code semantics; the closed loop ran green end-to-end on the single-dim fixture.

The runbook now has a real failure surface to catalog. Per CEO brief [O5](../../../../board/okrs/2026-05-23-ceo-brief.md), this loop transitions SRE from plan to build: ship `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` as `status: active` with named subsections per failure mode, against the CEO's standing-answer **"one runbook"** granularity decision. Modes B/C and the broader sim-farm spec §6 failure modes remain stubbed pending those modes shipping.

## Objectives

### O1: Ship the nanofab-supervisor-failed-validation runbook

source: ceo-brief

Why it matters: SRE's 2026-05-10 OKR named the runbook as next-loop work and named three cross-team verification points as preconditions. All three teams (sim-farm, resink-core, DevOps) are now active this loop, so the verifications land in-loop instead of as forward-pointers. The runbook is also the team's primary owned-surface artifact through 2026-05-30 — until on-call rotation and SLO authoring start (future loops), this is what "SRE owns sub-project #4 operational shape" cashes out to.

**Key results**

- KR1.1: `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` lands with `status: active`, frontmatter per charter convention (`type: runbook`, `owner: teams/platform/sre`), and named subsections covering the seven failure modes named in CEO brief O5 KR5.1:
  - Sim-farm MVP cases (Mode A verdict-layer): `missing`, `extra`, `diverged`, `engine_failure` (exit code 2).
  - Supervisor `panic::catch_unwind` path (codegen-generated `process()` panic).
  - Manifest-validation failure (coordinator / supervisor rejects malformed seal — read/parse / DAG-count / node-count / unknown-op failures).
  - Cargo-build failure (codegen output that doesn't compile — stage-1 gate, surfaces before supervisor ever boots).
- KR1.2: Each subsection follows the common shape: **symptom** (what the operator sees — stdout/stderr lines, exit codes, missing/wrong artifacts on disk), **diagnosis** (grep targets in `trace.jsonl` / `workspace/verdict.json` / supervisor stderr / cargo build log), **mitigation** (rollback / pause / re-dispatch path; "the loop never silently passed" framing per sim-farm verdict contract), and **severity tier** (S1/S2/S3 per KR1.3).
- KR1.3: A 3-tier severity scale is committed at the top of the runbook and applied per subsection:
  - **S1 — supervisor stuck or wrong-data risk to downstream**: any case where a non-zero exit slips past the driver, where the supervisor hangs (no progress for > 5× expected fixture duration), or where a panic is silently swallowed. The four MVP mismatch types (`missing`/`extra`/`diverged`/`engine_failure`) are S1 when they reach a hot-swap/training-stage-3 gate; S2 in pure MVP dev-loop context because the driver gates correctly.
  - **S2 — failure surfaces cleanly and stops the loop, but operator must investigate root cause**: panic with non-zero exit, manifest rejection, cargo-build failure. The supervisor refused to proceed; no bad data downstream; operator triages the offending DAG/codegen.
  - **S3 — cosmetic, observability, or contract-version gap**: e.g., a `verdict.json` field that has been renamed but the runbook hasn't been updated; a trace-log line missing a tenant label; verdict-layer name drift.
- KR1.4: A stubs section names post-MVP failure modes from sim-farm spec [§6.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md) (sidecar mid-shadow failure), §6.5 (write-trace socket overrun), §6.6 (coordinator crash), §6.3 (DuckDB diff engine failure beyond the MVP exit-code-2 case), §6.1 (sim-worker pod crash / `INFRA_FAILURE`), and Mode-B's `orphan_count > 1%` containment behavior. Each gets a one-paragraph stub with a "full coverage when Mode B / Mode C ships" marker and a forward-pointer to the spec section.
- KR1.5: Three cross-team verification points either land in-loop or carry an explicit `TODO: verify against impl` marker (per CEO brief O5 KR5.3 acceptance):
  - **(a) sim-farm verdict-layer names** — runbook references the exact `Verdict.mismatches[].type` enum values `"missing"` / `"extra"` / `"diverged"` and the three exit codes (`0`/`1`/`2`) from `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`. **Expected to land in-loop** — the contract is already `status: active`.
  - **(b) runtime `BLOCKED` observable** — runbook documents how the supervisor exposes its "blocked because sim verdict failed" state. Read of `main.rs` says: in the MVP, supervisor `run()` returns a `Result<(), String>`, `main()` prints `supervisor: error: <msg>` to stderr, and `std::process::exit(1)`. There is no distinct `BLOCKED` state separate from any other non-zero exit. **Expected to land with `TODO: verify against impl`** — the runbook records the MVP observable (stderr line + exit code 1) and forward-points to the loop in which a separate `BLOCKED` state-machine surface lands (sub-project #4 hot-swap work, post-Mode-B).
  - **(c) DevOps `tenant_id` label injection** — runbook's "filter logs by tenant" guidance references the `tenant` parameter in `values.yaml` (CEO brief O4 KR4.2) which becomes a Kubernetes label on the Deployment. **Expected to land in-loop** — DevOps's O4 commits to the values.yaml shape and the Helm chart smoke this loop; SRE coordinates mid-loop to confirm the label name spelling.
- KR1.6: `teams/platform/sre/status.md` updated in place with a "Recent shipments" entry pointing at the runbook (charter convention; carryover #3 from 2026-05-09 status closure pattern). Tenant-isolation check is N/A for the runbook (it lives under `teams/`, not `org-os/`).

**Tasks**

- [x] Draft runbook frontmatter + top-level severity-tier section (KR1.3) — owner: teams/platform/sre.
  - Shipped: `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` opens with `type: runbook` frontmatter and a top-of-document § Severity criteria block that commits the S1/S2/S3 definitions verbatim from KR1.3. Frontmatter-type workaround documented inline at the top of the body: `type: runbook` is the SRE-charter-established type but not yet in `org-os/conventions.md` (followed up as a carryover in status.md).
- [x] Author the four sim-farm MVP mismatch subsections (`missing`/`extra`/`diverged`/`engine_failure`); cross-link the verdict contract's exit-code table and `Mismatch` object schema (KR1.1, KR1.2, KR1.5a) — owner: teams/platform/sre.
  - Shipped: § 3.1 `missing`, § 3.2 `extra`, § 3.3 `diverged`, § 3.4 `engine_failure` (exit code 2). Each uses the Symptom / Diagnosis / Mitigation / Severity shape; cross-links the 2026-05-16 verdict contract (`type: contract`, `engine_version: 0.2.0`) for both single-dim and multi-dim shapes. KR1.5a closed: enum values referenced verbatim from the contract.
- [x] Author the supervisor-side subsections: `panic::catch_unwind` path (KR1.1), manifest-validation failure (KR1.1), cargo-build failure (KR1.1). Reference `main.rs` line numbers / error strings where helpful (KR1.2) — owner: teams/platform/sre.
  - Shipped: § 4.1 `panic::catch_unwind` (cites `main.rs` lines 167-197), § 4.2 manifest-validation failure (cites lines 93-125 + the unknown-op path at line 103), § 4.3 cargo-build / stage-1 gate failure. Each stderr-line catalog is grounded in the actual `eprintln!` / `Err(format!(...))` strings in `main.rs`. Line references are flagged as commit-drift-sensitive in § 7.
- [x] Mid-loop sync with sim-farm to confirm verdict-layer names are byte-identical to the contract (KR1.5a) — owner: teams/platform/sre + teams/application/sim-farm.
  - Confirmed against `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` (`status: active`, `engine_version: 0.2.0`). `Mismatch.type` enum values `"missing"` / `"extra"` / `"diverged"` are used verbatim; the three exit codes (`0` / `1` / `2`) and their final-line stderr/stdout formats are referenced verbatim. Both the single-dim shape (`pass`) and the multi-dim shape (`overall_pass` + `verdicts[]`) are covered.
- [x] Mid-loop sync with resink-core for the `BLOCKED` observable; expected outcome is "MVP exit-code-1 + stderr line is the current observable; richer surface follows hot-swap"; record as `TODO: verify against impl` if no richer answer (KR1.5b) — owner: teams/platform/sre + teams/application/resink-core.
  - Landed with the predicted `TODO: verify against impl` marker. Read of `crates/nanofab-supervisor/src/main.rs` confirmed that the MVP supervisor has no distinct `BLOCKED` surface — `run()` returns `Result<(), String>`, `main()` either prints `supervisor: ok` and exits 0, or prints `supervisor: error: <msg>` and `std::process::exit(1)`. The runtime coordinator that would surface a richer `BLOCKED` state is not in MVP scope. Documented in the runbook as § "Cross-cutting note 2 — `BLOCKED` observable is a known gap" with a forward-pointer to the post-Mode-B / post-`dlopen`-restoration loop.
- [x] Mid-loop sync with DevOps for the `tenant_id` Kubernetes label name spelling in `values.yaml` (KR1.5c) — owner: teams/platform/sre + teams/platform/devops.
  - Confirmed against `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/templates/_helpers.tpl`. The canonical label key is bare `tenant` (not `tenant_id`), sourced from required `.Values.tenant`. The chart's `nanofab-supervisor.tenantLabels` helper emits the full label set, including `tenant`, `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`, `nanofab.resink.ai/dag-version`, and `nanofab.resink.ai/fleet-color`. Documented in the runbook as § "Cross-cutting note 1 — Filter logs by tenant" with the canonical filter `tenant=<id>` and the full label-set listing.
- [x] Author the post-MVP stubs section covering sim-farm spec §6.1, §6.3, §6.4, §6.5, §6.6 and Mode-B orphan-count handling (KR1.4) — owner: teams/platform/sre.
  - Shipped: § 6.1 sim-worker pod crash / `INFRA_FAILURE` (spec §6.1), § 6.2 Mode-B sidecar pair-orphan `orphan_count > 1%` (spec §6.4), § 6.3 supervisor write-trace socket overrun / `trace_dropped` (spec §6.5), § 6.4 Mode-A coordinator crash / restart (spec §6.6), § 6.5 DuckDB diff failures beyond MVP exit-code-2 (spec §6.3), § 6.6 supervisor `panic_event` trace-record (spec §6.2). Each is one paragraph with a "Full coverage when ..." marker and a spec back-pointer.
- [x] Flip runbook `status: draft → active` once verification points (a) and (c) confirm and (b) either confirms or carries the named TODO; update `status.md` recent-shipments (KR1.6) — owner: teams/platform/sre.
  - Runbook shipped at `status: active` directly (no draft intermediate; all three verification points either landed or carry the planned `TODO` marker). `teams/platform/sre/status.md` refreshed: Last-updated date bumped to 2026-05-23, Recent shipments lists the runbook with anchor link, Carrying-into-next-loop enumerates SLO / on-call / paging deferrals + the frontmatter-type ADR follow-up.

### O2: Housekeeping — close the 2026-05-10 plan-only carryovers

source: ceo-brief

Why it matters: The 2026-05-10 OKR named four carryovers in its § Out of scope this loop (drafting the runbook itself; SLO/on-call/capacity authoring; runbooks outside sim-farm-seven; alert wiring). O1 closes the first. The remaining three stay deferred per CEO brief O5 framing (post-Mode-B work and on-call rotation policy are explicitly out-of-scope this loop), but the deferral is recorded here so retro can read it cleanly. Small; folds into O1 cleanup.

**Key results**

- KR2.1: `status.md` is refreshed in-loop with (a) the runbook backlink (KR1.6), (b) an explicit "Carrying into next loop" enumeration that names SLO authoring, on-call rotation policy, and alert paging as deferred-not-dropped with a near-term loop target (likely 2026-05-30 or 2026-06-06 — SRE picks during execution, depending on which sub-project #4 surface activates first).
- KR2.2: A one-line note in the runbook's "Out of scope" section names the failure-mode classes deferred to future runbooks (KV cluster failures, Kafka broker loss, Iceberg snapshot replication lag, region failover) — same enumeration as the 2026-05-10 OKR § Out of scope to keep the deferral list stable across loops.

**Tasks**

- [x] Refresh `teams/platform/sre/status.md` (KR2.1) — owner: teams/platform/sre.
  - Status.md now lists the runbook under Recent shipments and names SLO authoring, on-call rotation policy, and alert paging wiring under Carrying-into-next-loop with explicit "deferred-not-dropped" framing and a near-term loop target. Also adds a follow-up for the frontmatter-type `runbook` ADR alignment.
- [x] Add the "Out of scope" section to the runbook listing future-loop runbook classes (KR2.2) — owner: teams/platform/sre.
  - Runbook § 5 ("Out of scope (this revision)") names KV cluster failures, Kafka broker loss, Iceberg snapshot replication lag, region failover, SLO-per-pod, alerting/paging wiring, runtime coordinator hot-swap state machine, and sub-project #5. Same enumeration as the 2026-05-10 OKR Out-of-scope list to keep the deferral stable across loops.

## Cross-team asks

- **From teams/application/sim-farm, by mid-loop (Wave 1):** confirmation that the runbook may reference the verdict contract's three `Mismatch.type` enum values (`"missing"`, `"extra"`, `"diverged"`) and three exit codes (`0` PASS / `1` FAIL / `2` engine-failure) verbatim, and that no rename is planned this loop. Contract is `status: active` — expected to be a one-line "confirmed" reply. Closes KR1.5a.
- **From teams/application/resink-core, by mid-loop:** the supervisor's "blocked because sim verdict failed" observable handle in MVP scope. SRE's read of `crates/nanofab-supervisor/src/main.rs` is that there is currently only the generic `supervisor: error: <msg>` stderr line + exit code 1; no distinct `BLOCKED` state-machine surface exists yet. If resink-core confirms this is the MVP observable and the richer surface lands post-Mode-B, the runbook records that explicitly with a forward-pointer and a `TODO: verify against impl`. Closes KR1.5b.
- **From teams/platform/devops, by mid-loop:** the precise spelling of the Kubernetes label injected from the `tenant` `values.yaml` parameter (CEO brief O4 KR4.2). SRE expects something like `resink.ai/tenant=<value>` but defers the bikeshed to DevOps's chart authoring. If DevOps's `values.yaml` slips to late-loop, SRE writes the runbook against the parameter name (`{{ .Values.tenant }}`) and follows up next loop. Closes KR1.5c.
- **From CEO, this loop, already answered:** "one runbook" granularity confirmed (CEO brief "Standing CEO answers"). No further ask; recorded here for traceability.

## Runbook structure

The runbook is one document; the named subsections inside it are:

1. **Header & severity scale.** Frontmatter + top-of-document explanation of the S1/S2/S3 tiers (KR1.3) so every subsection's tier is anchored to one definition.
2. **Detection surfaces (cross-cutting).** Where alerts originate per CEO brief O5 KR5.3 verification points: sim-farm `workspace/verdict.json` + stdout/stderr from the engine; supervisor stderr + exit code; cargo build log; (post-MVP) the runtime coordinator's hot-swap state-machine `BLOCKED` transitions per sim-farm spec §6.1.
3. **Sim-farm MVP verdict failures (Mode A).** One subsection per `Mismatch.type` value plus the engine-failure exit-code-2 path. Cross-link to the verdict contract's worked examples.
   - 3.1 `missing` rows
   - 3.2 `extra` rows
   - 3.3 `diverged` rows
   - 3.4 `engine_failure` (exit code 2)
4. **Supervisor-side failures.**
   - 4.1 `panic::catch_unwind` path (codegen `process()` panic — `main.rs` lines 172-197 — currently exits 1 with stderr `panic processing event <event_id>`)
   - 4.2 Manifest-validation failure (read/parse error; DAG count ≠ 1; node count ≠ 1; unknown `op` — `main.rs` lines 93-125)
   - 4.3 Cargo-build failure (stage-1 gate — codegen-output crate at `<workspace>/nodes/<table>_scd2/` fails to compile under Option A static linking; supervisor binary fails to link, never boots)
5. **Out of scope (this revision).** Named future-loop runbook classes (KR2.2) and the post-MVP stub forward-pointers (KR1.4).
6. **Post-MVP stubs.** One paragraph each for sim-farm spec §6.1, §6.3, §6.4, §6.5, §6.6, and Mode-B `orphan_count > 1%`. Each carries the "full coverage when Mode B / Mode C ships" marker.
7. **Cross-references.** Verdict contract, sim-farm spec §6, runtime spec §7.4 + §8.4, serving spec §4.1, SRE charter.

## Risks

- **Verification point (b) `BLOCKED` observable is genuinely not in MVP scope.** Read of `main.rs` strongly suggests there is no distinct `BLOCKED` state-machine surface — the supervisor either exits 0 ("supervisor: ok") or 1 ("supervisor: error: …"). Mitigation: the OKR commits up front to landing this verification as `TODO: verify against impl` with a forward-pointer rather than as a hard block; the runbook documents the MVP observable (exit code + stderr) and is honest about what's not there yet. Retro can decide whether the gap is acceptable post-Mode-B.
- **DevOps `tenant_id` label name may slip past mid-loop.** O4 KR4.2 commits to the parameter but the chart authoring is concurrent with this runbook. Mitigation: write the runbook against the parameter spelling (`{{ .Values.tenant }}`) and update the rendered label name as a small follow-up if needed; not a blocker.
- **Multi-dim widening (CEO brief O1) may surface manifest-validation paths the MVP supervisor's exact-1-DAG / exact-1-node checks reject.** O1 KR1.2 says the orchestrator now writes a multi-node manifest; the supervisor's `main.rs` lines 114-124 currently `return Err(...)` on `dags.len() != 1` and `nodes.len() != 1`. The MVP's "manifest rejection" subsection of the runbook may rapidly become wrong (or, more precisely, the *current* rejection conditions become the historical scope). Mitigation: write the runbook against what's in `main.rs` *today*; if resink-core lands the multi-node supervisor mid-loop, refresh the rejection-conditions list in 4.2 before flipping `status: active`. Both `main.rs` line refs in the runbook are pinned to a commit SHA in the cross-references section.
- **Drafting the runbook against a moving target.** Three teams ship in parallel this loop (sim-farm verdict extension for multi-dim, resink-core multi-node supervisor, DevOps Helm chart). Mitigation: SRE drafts against the *2026-05-16 verdict contract + 2026-05-23 main.rs* as the canonical reference; refresh pass mid-loop after the three sync points; flip `status: active` last.
- **Modes B/C stubs may be wrong by the time those modes ship.** The §6 spec sections are stable but the implementation may diverge. Mitigation: stubs are one paragraph each, forward-pointed to the spec; refresh expected on the loops Modes B and C ship. Stubs are explicitly named as stubs in the runbook header.

## Out of scope this loop

- Mode B (per-node shadow) and Mode C (DAG blue/green warmup) deep-dive runbook sections — stubs only this loop per KR1.4; full coverage when those modes ship.
- On-call rotation policy authoring (named in 2026-05-10 § Out of scope; remains deferred; surfaced in `status.md` per KR2.1).
- Capacity planning for sub-project #4 — per-tenant `LimitRange` / `ResourceQuota` dashboards, cost-guardrail anomaly playbook (serving spec §4.1, §8.3). Charter-owned, not loop-scheduled.
- Alerting rules / paging wiring — no production sub-project #4 deployment yet; DevOps's first slice is Helm-chart-shaped (CEO brief O4), not full prod.
- SLOs per supervisor pod (named in 2026-05-10 § Out of scope; remains deferred).
- Region & availability operational response, DR drills (charter-owned, charter §5; not loop-scheduled).
- Secrets-rotation surfacing (charter-owned, charter §7; not loop-scheduled; depends on DevOps's rotation Lambdas which are post-MVP).
- Runbooks for failure-mode classes outside the supervisor-failed-validation umbrella — KV cluster failures, Kafka broker loss, Iceberg snapshot replication lag, region failover — these land as separate runbooks in future loops (KR2.2 records the deferral list in the runbook itself).
- The one-vs-seven runbook granularity question — CEO answered "one runbook" (brief "Standing CEO answers"); no further deliberation.
- Any sub-project #5 (Product UX) operational surfaces — ownership deferred per CEO brief.
