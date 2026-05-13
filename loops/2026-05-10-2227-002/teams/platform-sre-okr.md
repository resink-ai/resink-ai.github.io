---
layout: default
title: platform-sre OKR — 2026-05-10-2227-002
date: 2026-05-10
status: active
type: okr
loop: 2026-05-10-2227-002
owner: teams/platform/sre
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/sre
  date: 2026-05-10
  status: active
  loop: 2026-05-10-2227-002
  links: parent: board/okrs/2026-05-10-2227-002-ceo-brief.md
-->
{% raw %}

# SRE OKR — 2026-05-10

## Context

The CEO brief ratifies the nanofab pivot via ADR-002 supersession (O1) and seats SRE on sub-project #4 (Nanofab Serving & Deployment) for operational shape, runbooks, on-call, and capacity surfaces (O3). SRE's deferred 2026-05-09 runbook ("demo pipeline failed validation") was Spark-shaped and does not survive the pivot. Its replacement target is the nanofab supervisor in `--mode=sim` (per [Sim Farm spec](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md) §1.3) and the runtime hot-swap state machine (per [runtime spec](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) §6.2-6.3, as referenced from the [serving spec](../../../../docs/superpowers/specs/2026-05-10-nanofab-serving-deployment-design.md) §6.1).

The cross-team inputs SRE was waiting on last loop (sim-farm's failure-mode list, resink-core's deployment shape) landed in 2026-05-09 — so this loop's planning is well-grounded. Per CEO brief O3 KR3.4 this OKR is **plan-only** — no runbook draft this loop; the runbook lands next loop, against a scope explicitly named here.

## Objectives

### O1: Claim sub-project #4 operational surfaces (charter-touch + planning)

Why it matters: Per CEO brief [O3 KR3.2](../../../../board/okrs/2026-05-10-2227-002-ceo-brief.md), SRE now owns sub-project #4's operational shape, runbooks, on-call, and capacity surfaces. Updating the charter this loop makes the ownership boundary explicit before SRE starts producing artifacts against the serving spec.

**Key results**
- KR1.1: `teams/platform/sre/charter.md` updated to declare ownership of sub-project #4's operational shape, runbooks, on-call, and capacity surfaces, with explicit references to the serving spec (§4 multi-tenancy, §5 region & availability, §7 secrets, §8 cost guardrails) and runtime spec (§6 hot swap). Existing v1 charter content (mission, SLO surface, runbook ownership, decision rights) preserved.
- KR1.2: Charter delta calls out the boundary against DevOps (deployment topology, IaC, CI/CD gate, secrets storage) so future cross-team work has an unambiguous seam.

**Tasks**
- [x] Update `teams/platform/sre/charter.md` per KR1.1/KR1.2 — owner: teams/platform/sre

### O2: Plan the first sub-project-#4 product-shaped slice — runbook for nanofab supervisor failure modes

Why it matters: Per CEO brief [O3 KR3.3](../../../../board/okrs/2026-05-10-2227-002-ceo-brief.md), SRE produces a team OKR this loop that names its first product-shaped slice under sub-project #4. The suggested re-aim is the deferred "demo pipeline failed validation" runbook, now re-pointed at the nanofab supervisor's seven Sim-Farm-enumerated failure modes against the new runtime. Per [O3 KR3.4](../../../../board/okrs/2026-05-10-2227-002-ceo-brief.md), this is **plan-only this loop**: the runbook lands next loop. The three carryovers from `status.md` (storage convention; runbook draft; cross-link) are absorbed here — the storage-convention decision lands *this* loop so next-loop drafting is unblocked.

**Key results**
- KR2.1: A runbook storage convention is decided and documented in `teams/platform/sre/charter.md` (or a one-line note under `teams/platform/sre/runbooks/README.md`) so the next-loop draft has a known home. Default convention: `teams/platform/sre/runbooks/<slug>.md`, one runbook per file, frontmatter `type: runbook`, `owner: teams/platform/sre`, `status: draft|active`.
- KR2.2: A written planning note (this OKR body, § Runbook plan below) names the next-loop runbook's scope: one runbook covering the seven failure modes from `teams/application/sim-farm/okrs/2026-05-10-2227-001-team-okr.md` § Validation contract KR1.1.b (schema drift, duplicate `event_id`, late data, missing `user_id`, out-of-range `timestamp`, null in required field, unknown `country_code`) — re-mapped from the Spark-pipeline framing onto the nanofab supervisor's signals in `--mode=sim` (panic events in `trace.jsonl`, sim-farm verdict layers, runtime coordinator's hot-swap state-machine transitions).
- KR2.3: The runbook plan names the operational signals SRE will key on per failure mode, drawn from the runtime/serving specs: (a) supervisor panic events captured per `panic::catch_unwind` (runtime spec §7.4 / sim-farm spec §6.2), (b) Sim Farm verdict layers (`node_coverage`, `diff`, `INFRA_FAILURE`, `DIFF_ENGINE_FAILURE` — sim-farm spec §6.1-6.3), (c) hot-swap state-machine transitions (runtime spec §6.2-6.3, blocked-on-INFRA_FAILURE behavior per sim-farm spec §6.1), (d) per-tenant metric labels (serving spec §4.1 — `tenant_id` required on every metric/log).
- KR2.4: The plan names the three cross-team verification points that need to be true *before* the next-loop runbook can be considered complete (see § Cross-team asks).

**Tasks**
- [x] Decide and document the runbook storage convention (KR2.1) — owner: teams/platform/sre
- [x] Write the runbook plan section below (KR2.2/KR2.3/KR2.4) — owner: teams/platform/sre
- [x] Cross-link this OKR from `teams/platform/sre/status.md` "Recent shipments" once it lands — owner: teams/platform/sre

## Runbook plan (KR2.2-KR2.4): "nanofab supervisor failed validation"

**Target artifact (next loop):** `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md`.

**Scope.** One runbook covering the seven failure modes enumerated by sim-farm in 2026-05-09 (Validation contract KR1.1.b). The runbook treats them as a single operational class — "a sim-farm verdict failed against a supervisor in `--mode=sim`, blocking a hot swap or a training stage-3 release" — and structures triage by *which verdict layer* failed, since that determines the next action.

**Failure-mode → signal mapping (draft, to be refined next loop):**

| # | Failure mode (from sim-farm 2026-05-09 KR1.1.b) | Primary signal | Detection surface | Triage path |
|---|---|---|---|---|
| 1 | Schema drift | Sim Farm verdict, schema-diff layer FAIL | sim-farm spec §6.3 — structural diff in DuckDB | Quarantine release; escalate to resink-core (DAG author) |
| 2 | Duplicate `event_id` | Sim Farm verdict, idempotency/diff layer FAIL | per-table diff vs reference | Inspect dedup node in `trace.jsonl`; quarantine |
| 3 | Late data | Sim Farm verdict, watermark layer FAIL | trace.jsonl watermark events | Confirm vs declared watermark policy; route to DE if ingress contract mismatch |
| 4 | Missing `user_id` | Sim Farm verdict, referential layer FAIL | dead-letter row counts | Confirm dead-letter wiring; quarantine if dead-letter missing |
| 5 | Out-of-range `timestamp` | Sim Farm verdict, sanity layer FAIL | trace.jsonl + per-table diff | Quarantine; escalate to DE for ingress sanity check |
| 6 | Null in required field | Sim Farm verdict, schema layer FAIL | per-table NULL counts | Quarantine; escalate to resink-core |
| 7 | Unknown `country_code` | Sim Farm verdict, value-domain layer FAIL | trace.jsonl + dictionary lookup | Quarantine; escalate to DE if dictionary stale |

**Cross-cutting runbook sections (all seven modes share these):**
- *Detection.* Where the alert fires — sim-farm `simfarm.verdicts` Iceberg table watch, supervisor panic-event stream, runtime coordinator hot-swap state-machine `BLOCKED` transitions. Per-tenant `tenant_id` label is required on every alert.
- *Initial triage.* Pull the trace.jsonl for the failing sim run; read the verdict layers; identify whether the failure is a *supervisor* problem (panic, infra) vs a *content* problem (diff, schema, value-domain).
- *Containment.* For hot-swap (mode B/C) failures: the runtime coordinator already blocks the swap on `INFRA_FAILURE` (sim-farm spec §6.1) — SRE confirms the block held; no manual rollback needed. For training (mode A) failures: training validator retries up to 2× before escalating (sim-farm spec §6.1); SRE's role is to confirm the retry path completed.
- *Rollback / quarantine.* For training stage-3 failures: training repo PR stays unmerged; no production impact. For hot-swap blue/green failures: blue/green candidate fleet drained, prior fleet remains; SRE verifies traffic still on prior fleet via Grafana per-tenant dashboard (serving spec §4.1).
- *Escalation criteria.* When SRE pages an IC: (a) hot-swap blocked > 30 min with no clear path forward, (b) cross-tenant impact suspected (defense-in-depth check failed per serving spec §4.1), (c) `DIFF_ENGINE_FAILURE` or repeated `INFRA_FAILURE` (sim-farm itself is the suspect).
- *Per-tenant capacity surface.* Linked from this runbook (planned, not authored this loop): a brief on the per-tenant `LimitRange` / `ResourceQuota` shape (serving spec §4.1) and the cost-guardrail anomaly suspension (§8.3), since "capacity exceeded" can manifest as a sim-farm `INFRA_FAILURE` and the runbook should distinguish "real failure" from "we throttled them."

**Verification points (KR2.4) — these must hold before the next-loop runbook is "done":**
1. The verdict-layer names above match what sim-farm actually emits — confirmed against sim-farm's next-loop implementation OKR (still pending; treat the table above as draft until sim-farm signs off).
2. The runtime coordinator's hot-swap state machine surfaces `BLOCKED` transitions in a way SRE can alert on — confirmed against runtime spec §6.2-6.3 and any next-loop runtime work.
3. Per-tenant `tenant_id` labels exist on the relevant alerts/metrics — confirmed against DevOps's serving topology / Prometheus wiring (sub-project #4 deployment-side work).

## Cross-team asks

- **From teams/application/sim-farm, by mid-next-loop:** confirmation of the precise verdict-layer names and shape (the table above is SRE's best read of the spec + 2026-05-09 contract; sim-farm owns the actual emit). Without this, the runbook covers near-but-not-exact signals.
- **From teams/application/resink-core, by mid-next-loop:** the runtime coordinator's hot-swap state-machine surface for `BLOCKED` / `PROCEED_WITH_CAUTION` transitions, ideally a metric or structured-log shape SRE can alert on. Per the runtime/sim-farm specs the behavior is defined; SRE needs the observable handle.
- **From teams/platform/devops, by mid-next-loop:** confirmation that per-tenant `tenant_id` label injection (serving spec §4.1) is in DevOps's first sub-project-#4 slice or named as a near-term follow-up. SRE's runbook keys on this label being present; if DevOps's first slice is supervisor-deploy-shaped and the metric wiring slips later, SRE adjusts scope.
- **From the CEO, this loop or next:** confirmation that "one runbook covering seven failure modes" is the right granularity (vs seven separate runbooks). SRE's read of the brief is yes (the seven are operationally a single class), but a separate-files convention may be preferred for searchability — easy to refactor next loop if so.

## Risks

- **Spec drift between plan and implementation.** The verdict-layer names and hot-swap state-machine surface are read from specs, not running code. Next-loop runbook drafting may need a revision pass once sim-farm and resink-core start implementing. Mitigation: the runbook plan above lists three verification points (KR2.4) — drafting starts only after those are confirmed, OR the draft ships with explicit "TODO: verify against impl" markers on the affected sections.
- **Capacity / on-call surface still implicit.** This OKR explicitly does *not* author SLOs, on-call rotation, or capacity playbooks for sub-project #4 — only the runbook plan. The broader operational shape (SLOs per supervisor pod, per-tenant capacity dashboards, on-call escalation tree) is named in the charter but unscheduled. Mitigation: surface this as a future-loop scoping ask in the next exec summary; don't pretend it's covered here.
- **Cross-team asks land late.** Three of the four asks above depend on sim-farm / resink-core / DevOps next-loop work that may itself slip. Mitigation: the runbook draft can ship with placeholder verdict-layer names referenced from the spec sections, and the verification pass becomes a follow-up loop task. Better an imperfect runbook than no runbook.

## Out of scope this loop

- Drafting the runbook itself (next loop, per CEO brief O3 KR3.4 plan-only framing).
- Authoring SLOs, on-call rotation policy, or capacity dashboards for sub-project #4 (future loops; surfaced in charter but not scheduled here).
- Runbooks for failure modes outside the sim-farm seven (e.g., KV cluster failures, Kafka broker loss, Iceberg snapshot replication lag) — those land as separate runbooks in future loops; this loop scopes only the validation-failure class.
- Wiring alerts to a paging system (no production sub-project #4 deployment yet; DevOps's first slice is Helm-chart-shaped, not full prod).
- Any work on sub-project #5 (Product UX) operational surfaces — ownership for #5 is deferred per CEO brief.
{% endraw %}
