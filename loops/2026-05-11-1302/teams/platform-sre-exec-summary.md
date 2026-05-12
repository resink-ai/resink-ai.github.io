---
layout: default
title: platform-sre Exec Summary — 2026-05-11-1302
date: 2026-05-11
status: active
type: exec-summary
loop: 2026-05-11-1302
owner: teams/platform/sre
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/sre
  date: 2026-05-11
  status: active
  loop: 2026-05-11-1302
  links: parent: board/okrs/2026-05-11-1302-ceo-brief.md
-->
# SRE Exec Summary — 2026-05-30 (paused team — review-ack)

**Loop status:** paused per CEO brief 2026-05-30. No build deliverables this loop; single light "review the resink-core docs PR" ask.

## Review-ack

Read `repos/resink-ai/resink-core/CLAUDE.md` + `docs/{architecture,concepts,user-guide,module-catalog}.md`. SRE's surface is **accurately represented.** Specifics checked:

- **`CLAUDE.md` § "Where to find more"** cross-links the SRE runbook explicitly: `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` (in parent newbase). Pointer is correct and the runbook now carries `type: runbook` (canonical after the board's enum extension this loop per ADR-2026-05-23-001 extended ratification adding `runbook` to `org-os/conventions.md`).
- **`concepts.md` § "trace"** correctly names the SRE runbook as the triage entry-point: "Per runtime spec §8.4, the trace is the supervisor's contract with the SRE observability surface; when a `panic::catch_unwind` fires, the trace carries the panic message and the SRE runbook at `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` is the triage entry-point." Accurate characterization of the trace → runbook contract.
- **`user-guide.md` § "Troubleshooting"** cross-links the SRE runbook for "Supervisor terminal failure / panic": "The supervisor uses `panic::catch_unwind` and emits the panic message into `trace.jsonl`; the runbook is the triage entry-point for any non-success terminal state." Correct — matches the runbook's MVP § 5 (supervisor `panic::catch_unwind`) coverage.
- **`module-catalog.md` § `crates/nanofab-supervisor/`** correctly names SRE as the owner of the failed-validation runbook with the exact runbook path.
- **`module-catalog.md` § `crates/nanofab-node-abi/`** correctly notes "the SRE runbook references the failure-mode shapes (`NodeError` variants) for triage." Accurate — the runbook does cross-reference the ABI error variants in its supervisor-side subsections.
- **`module-catalog.md` § `deploy/charts/nanofab-supervisor/`** correctly notes "SRE consumes the deployment surface (signal handling, env-var convention, port convention) for runbook authoring" — accurate forward-pointer; signal handling and port convention surface in the runbook when the supervisor-side hand-off items land.
- **`architecture.md`** correctly omits SLO authoring, on-call rotation, and alert paging from the "today" surface — these are deferred-not-dropped per SRE's status.md and are correctly absent from a "what works today" doc.

No corrections needed. The runbook's MVP scope (7 named failure-mode subsections — 4 sim-farm verdict mismatches + 3 supervisor-side paths) and the post-MVP § 6 stubs (Modes B / C failure-mode expansion) are correctly characterized by their absence from resink-core's "what works today" docs and by their explicit forward-pointing in the runbook itself.

## Carryover unchanged

- **BLOCKED observable verification** — deferred-not-dropped, forward-pointed in the runbook's § Cross-cutting note 2 as `TODO: verify against impl`. Closes when resink-core surfaces a structured `BLOCKED` state from the runtime coordinator (verdict id, mismatch count, types, tenant). **Forward-pointed to post-dlopen-restoration** per ADR-2026-05-16-001 (resink-core supervisor swap targets loop 2026-05-11-1631; the structured BLOCKED surface is downstream of that work). Carries.
- **Modes B / C failure-mode expansion** — § 6 stubs in the runbook (sim-worker pod crash / `INFRA_FAILURE`, Mode-B sidecar orphan-count, write-trace socket overrun, Mode-A coordinator restart, DuckDB retry-once, supervisor `panic_event` trace record). **Full prose lands the loop those modes ship in CI** — i.e., when sim-farm Modes B (per-node shadow) and C (DAG blue/green warmup) land per Sim Farm spec §5.2 / §5.3. Carries; bound to sim-farm's Modes B/C scoping carryover.
- **Frontmatter-type `runbook` ADR** — **closed this loop** as part of the board's ADR-2026-05-23-001 extended ratification, which adds `runbook` (plus `convention`, `playbook`, `report`) to `org-os/conventions.md`. SRE's runbook frontmatter migrates `type: rfc → type: runbook` in the same wave. No longer a carryover.
- **SLO authoring (per-supervisor-pod)** — deferred-not-dropped; near-term target shifts to loop 2026-05-11-1631 or later, depending on when DevOps's first prod slice lands (which is itself gated on the minikube smoke and the post-dlopen supervisor-side hand-off items). Carries.
- **On-call rotation policy** — deferred-not-dropped; draft surfaces when the runbook catalog grows beyond one entry. Depends on staffing decisions. Carries.
- **Alert paging wiring** — deferred-not-dropped; depends on metric-emission seam being wired live by DevOps (Prometheus / log routing). Carries.
- **Refresh pass on the runbook** — pin `main.rs` line refs to a commit SHA in § 7; refresh § 4.2's rejection-conditions list once the multi-node supervisor lands (already landed 2026-05-23 — refresh due). Carries with mild priority bump.

## Asks

None this loop. The `type: runbook` ADR carryover closes via the board's ratification wave (no SRE action required beyond confirming the frontmatter flip lands when the migration runs).
