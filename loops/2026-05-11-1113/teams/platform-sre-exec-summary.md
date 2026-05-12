---
layout: default
title: platform-sre Exec Summary — 2026-05-23
nav_exclude: true
render_with_liquid: false
date: 2026-05-23
status: active
type: exec-summary
loop: 2026-05-11-1113
owner: teams/platform/sre
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/sre
  date: 2026-05-23
  status: active
  loop: 2026-05-23
  links: parent: teams/platform/sre/okrs/2026-05-11-1113-team-okr.md
-->
# SRE Exec Summary — 2026-05-23

## What we shipped

First build-mode loop in four weeks. One artifact, planned and shipped end-to-end.

### O1: Ship the nanofab-supervisor-failed-validation runbook

- KR1.1 + KR1.3 — Runbook shipped at `status: active` with `type: runbook` frontmatter and a top-of-document § Severity criteria committing the S1/S2/S3 scale verbatim from the OKR: [`teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md`](../runbooks/nanofab-supervisor-failed-validation.md). One document, 13 named failure-mode subsections (7 MVP + 6 post-MVP stubs) per the CEO's "one runbook" granularity decision.
- KR1.1 + KR1.2 + KR1.5a — Four sim-farm Mode-A MVP subsections (`Symptom` / `Diagnosis` / `Mitigation` / `Severity` per subsection): § 3.1 `missing`, § 3.2 `extra`, § 3.3 `diverged`, § 3.4 `engine_failure` (engine exit code 2). Each cross-links the 2026-05-16 verdict contract (`type: contract`, `engine_version: 0.2.0`); enum values `"missing"`/`"extra"`/`"diverged"` and exit codes `0`/`1`/`2` referenced verbatim from the contract. Covers both the single-dim `0.1.0` shape and the multi-dim `0.2.0` shape.
- KR1.1 + KR1.2 — Three supervisor-side MVP subsections grounded in `crates/nanofab-supervisor/src/main.rs`: § 4.1 `panic::catch_unwind` (cites lines 167-197), § 4.2 manifest-validation failure (cites lines 93-125; six distinct stderr-line variants cataloged), § 4.3 cargo-build / stage-1 gate failure (Option-A static linking context). Line refs flagged as commit-drift-sensitive in § 7.
- KR1.4 — Six post-MVP stubs landed in § 6, one paragraph each with a "Full coverage when …" marker and a spec back-pointer: § 6.1 sim-worker pod crash / `INFRA_FAILURE` (spec §6.1), § 6.2 Mode-B sidecar pair-orphan `orphan_count > 1%` (spec §6.4), § 6.3 write-trace socket overrun / `trace_dropped` (spec §6.5), § 6.4 Mode-A coordinator crash / restart (spec §6.6), § 6.5 DuckDB diff failures beyond MVP exit-2 (spec §6.3), § 6.6 supervisor `panic_event` trace-record (spec §6.2).
- KR1.5a — sim-farm verdict-layer enum names + exit codes landed in-loop (clean). Confirmed against `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` (`status: active`, `engine_version: 0.2.0`); both single-dim (`pass`) and multi-dim (`overall_pass` + `verdicts[]`) shapes referenced.
- KR1.5c — DevOps `tenant`-label spelling landed in-loop (clean). Confirmed against `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/templates/_helpers.tpl`; canonical filter is bare `tenant=<id>`, sourced from required `.Values.tenant`. Full label set documented in § Cross-cutting note 1.
- KR2.1 + KR1.6 — `teams/platform/sre/status.md` refreshed in place: date bumped to 2026-05-23, Recent shipments anchored to the runbook, Carrying-into-next-loop enumerates SLO authoring / on-call rotation / alert paging as deferred-not-dropped plus the frontmatter-type ADR follow-up.

### O2: Housekeeping — close the 2026-05-10 plan-only carryovers

- KR2.2 — Runbook § 5 ("Out of scope (this revision)") names the deferred runbook classes verbatim from the 2026-05-10 § Out-of-scope list (KV cluster failures, Kafka broker loss, Iceberg snapshot replication lag, region failover) plus SLO-per-pod, alerting/paging wiring, runtime coordinator hot-swap state machine, and sub-project #5. Same enumeration as last loop's deferral, kept stable across loops.

## What we didn't ship and why

- **KR1.5b — `BLOCKED` observable verification: TODO marker landed in lieu of a clean confirmation.** As predicted in the OKR risks section, the MVP supervisor exposes no distinct `BLOCKED` state-machine surface — read of `main.rs` confirms `run()` returns `Result<(), String>`, `main()` either prints `supervisor: ok` and exits `0` or prints `supervisor: error: <msg>` and `std::process::exit(1)`. Every non-success terminal state collapses into the pair `(exit code 1, stderr line)`. Documented in the runbook as § Cross-cutting note 2 with an explicit `TODO: verify against impl` block and a forward-pointer to the post-Mode-B / post-`dlopen`-restoration loop (earliest 2026-06-06, more likely later).
- **Modes B/C deep-dive runbook sections — stubs only (deferred-by-design).** Per KR1.4 the runbook ships paragraph-level stubs forward-pointed to spec §6; full prose lands the loop those modes ship.
- **On-call rotation policy authoring — deferred-not-dropped.** Out-of-scope this loop per OKR § Out of scope; carried in `status.md` Carrying-into-next-loop. Surfaces a draft when the runbook catalog grows beyond one entry.
- **SLOs per supervisor pod — deferred-not-dropped.** Out-of-scope this loop per OKR; depends on a production deployment, which is contingent on DevOps's first prod slice. Carried in `status.md`.
- **Alerting / paging wiring — deferred-not-dropped.** No production sub-project #4 deployment yet; DevOps's first slice is Helm-chart-shaped (CEO brief O4), not full prod. Carried in `status.md`.

## Surprises

- **DevOps's canonical label key is bare `tenant`, not `tenant_id`.** The 2026-05-10 OKR carried the planning-time placeholder `tenant_id` (echoing serving spec §4.1's `tenant_id` framing). Read of `_helpers.tpl` revealed DevOps's chart settled on bare `tenant` sourced from `.Values.tenant` (with `fail` if unset). Runbook references the canonical key correctly throughout; § Cross-cutting note 1 documents the full label set.
- **Sim-farm verdict contract type was migrated to `type: contract` this loop.** Planning-time references were to a draft of the contract; the board ratified the contract-class convention this loop. Runbook references both the single-dim `0.1.0` shape (byte-identical preserved) and the new multi-dim `0.2.0` shape (`overall_pass` + `verdicts[]`); jq examples cover both via `.verdicts[]? // .`.
- **Runbook frontmatter `type: runbook` is not in `org-os/conventions.md`.** The SRE charter (§ Owned products) establishes `type: runbook` as the SRE-local convention, but the org-os conventions enum does not yet include it. Worked around inline at the top of the runbook body (validators-treat-as-unprocessable should skip; downstream readers read as runbook regardless). Follow-up ADR carried to retro and to `status.md` Carrying-into-next-loop.

## Asks

- **CEO / retro — ADR candidate to add `type: runbook` to `org-os/conventions.md`** (or fold runbooks into the existing `type: contract` family as a sister to ADR-2026-05-10-004). SRE will draft the ADR proposal during retro; classification suggests org-os-class.
- **teams/application/resink-core — surface a distinct `BLOCKED` observable in the supervisor when `dlopen` restoration lands.** Today the supervisor collapses every non-success terminal state into `(exit code 1, stderr line)`; once the runtime coordinator exposes a structured `BLOCKED` state (verdict id, mismatch count, mismatch types, tenant) the runbook's § Cross-cutting note 2 TODO closes and § 3 / General triage can consume the structured surface instead of stderr grepping. Earliest target: post-Mode-B / post-`dlopen`-restoration (≥ 2026-06-06).

## Metrics

- **KR1.1** (runbook lands `status: active` with frontmatter + 7 named MVP subsections per CEO brief O5 KR5.1): **shipped** — `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` § 3.1-3.4, § 4.1-4.3.
- **KR1.2** (each subsection uses common Symptom / Diagnosis / Mitigation / Severity shape): **shipped** — applied verbatim across all seven MVP subsections.
- **KR1.3** (3-tier severity scale committed at top + applied per subsection): **shipped** — § Severity criteria; every subsection carries an explicit Severity tag.
- **KR1.4** (post-MVP stubs section names sim-farm spec §6.1, §6.3, §6.4, §6.5, §6.6 and Mode-B orphan-count): **shipped** — § 6.1-6.6 cover all six.
- **KR1.5a** (sim-farm verdict-layer names verified in-loop): **landed clean** — confirmed against verdict contract (`engine_version: 0.2.0`).
- **KR1.5b** (runtime `BLOCKED` observable verified in-loop): **landed as predicted TODO** — `TODO: verify against impl` recorded in § Cross-cutting note 2 with forward-pointer to post-Mode-B / post-`dlopen`-restoration.
- **KR1.5c** (DevOps `tenant`-label spelling verified in-loop): **landed clean** — confirmed against `_helpers.tpl`; canonical key `tenant` documented.
- **KR1.6** (`status.md` refreshed with Recent-shipments entry): **shipped** — `teams/platform/sre/status.md` refreshed in place, date 2026-05-23.
- **KR2.1** (`status.md` Carrying-into-next-loop enumerates SLO / on-call / paging): **shipped** — all three deferred-not-dropped with near-term loop target; frontmatter-type ADR follow-up added.
- **KR2.2** (runbook § Out of scope lists deferred classes verbatim from 2026-05-10 list): **shipped** — § 5.

**Verification-point tally:** 2 of 3 landed clean (sim-farm enum names, DevOps `tenant` label); 1 landed as the planned `TODO: verify against impl` marker (resink-core `BLOCKED` observable). All three accounted for in-loop. Tenant-isolation invariant: N/A — no `org-os/` edits.
