---
layout: default
title: platform-sre OKR — 2026-05-10-2227-001
date: 2026-05-09
status: active
type: okr
loop: 2026-05-10-2227-001
owner: teams/platform/sre
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/sre
  date: 2026-05-09
  status: active
  loop: 2026-05-10-2227-001
  links: parent: board/okrs/2026-05-10-2227-001-ceo-brief.md
-->
# SRE OKR — 2026-05-09

## Context

The CEO brief confirms "demo pipeline failed validation" as the first runbook scope (a 2026-05-08 SRE ask). The 2026-05-08 metric-list draft for the demo pipeline (3 metrics) is in place; the runbook is the next layer of operational readiness for the demo, even before the pipeline ships. This loop SRE drafts the runbook so it's ready when the pipeline is.

## Objectives

### O1: Draft the "demo pipeline failed validation" runbook

Why it matters: Carryover from 2026-05-08 — runbook needed the demo pipeline to exist before being writable, but the validation runbook can be drafted from sim-farm's failure-mode contract independently. Drafting now means SRE doesn't become the gate when the pipeline lands.

**Key results**
- KR1.1: A first-draft runbook exists at a documented path under `teams/platform/sre/` covering: (a) detection signals for "validation failed," (b) initial triage steps, (c) rollback or quarantine procedure, (d) escalation criteria for paging an IC.
- KR1.2: The runbook references sim-farm's failure-mode list (from `teams/application/sim-farm/okrs/2026-05-10-2227-001-team-okr.md`) so the scenarios it covers match what sim-farm intends to detect.

**Tasks**
- [ ] Decide the runbook storage convention for SRE (e.g., `teams/platform/sre/runbooks/`) — owner: teams/platform/sre
  - > deferred: scoped out of this loop's critical-path build per CEO scope decision. Sim-farm's failure-mode list landed (in `teams/application/sim-farm/okrs/2026-05-10-2227-001-team-okr.md` § Validation contract) so next-loop drafting is well-grounded; resink-core's deployment shape is also captured in their plan addendum.
- [ ] Draft the "demo pipeline failed validation" runbook — owner: teams/platform/sre
  - > deferred: depends on the storage convention above.
- [ ] Cross-link the runbook from the team's `status.md` "Recent shipments" — owner: teams/platform/sre
  - > deferred: depends on the runbook above.

## Cross-team asks

- **From teams/application/sim-farm, by mid-loop:** the enumerated failure modes sim-farm intends to detect on the demo pipeline. Without these, the runbook covers generic patterns rather than the actual scenarios that will fire alerts.
- **From teams/application/resink-core, by mid-loop:** the demo pipeline's deployment / liveness shape (where the pipeline runs, how it signals "alive"), so the runbook's detection section is grounded.

## Risks

- The runbook may end up generic if the cross-team asks above are not answered by mid-loop. Mitigation: ship a generic v1 with named placeholder sections so the next loop's revision is mechanical rather than from-scratch.

## Out of scope this loop

- Wiring alerts to a paging system (no pipeline yet).
- Runbooks for failure modes other than "validation failed."
- Documenting SLOs for the demo pipeline (the bootstrap-loop notes a smaller-than-expected SLO surface; revisit when the pipeline is live).
