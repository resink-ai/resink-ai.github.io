---
layout: default
title: Brief
nav_order: 1
parent: "Loop 2026-05-09"
grand_parent: "Loops"
render_with_liquid: false
date: 2026-05-09
status: active
type: okr
loop: 2026-05-09
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-09
  status: active
  loop: 2026-05-09
  links: parent: ""
-->
# Resink.ai CEO Brief — 2026-05-09

## Context

Loop 2026-05-08 bootstrapped the org-OS itself: four platform teams produced charters, OKRs, and exec summaries; two application teams (resink-core, sim-farm) joined as charter-only. The 2026-05-08 retro classified three evolution proposals (P1 frontmatter linter, P2 application-team onboarding playbook, P3 weekly-cadence automation) and ADR-001 (frontmatter validation) landed.

Three things are blocking forward motion: (a) the engine-choice ADR is partial — DE drafted a Spark vs. Flink comparison but no recommendation is committed; (b) the deployment-target ADR is partial — DevOps enumerated options but did not commit; (c) two CEO-level asks are open from 2026-05-08 — the first demo target and the first runbook scope. The biggest gap from the vision in `ORG.md` is concrete: there is no end-to-end demo pipeline yet, and resink-core cannot start drafting one until the two ADRs land. This loop's job is to finish unblocking and to pull the application teams into product-shaped planning.

**CEO answers to open asks from 2026-05-08:**

- **First demo target — confirmed: `fact_sign_up.parquet`.** This was the proposal in `board/charter.md` and is consistent with `ORG.md`'s enumeration. Both DE's shared-primitive work and the engine-choice ADR's recommendation should be shaped against this single fact-table type for this loop and the next.
- **First runbook scope — confirmed: "demo pipeline failed validation."** SRE has the candidate identified; draft begins this loop and lands by next loop's exec summary at the latest.

A separate org-os refactor (the bottom-up flow design at `docs/superpowers/specs/2026-05-09-org-os-bottom-up-flow-design.md`) is in flight as an ADR-gated change to `org-os/`. This loop **ratifies** that design — confirms direction and schedules execution — but defers the 17-task implementation to loop 2026-05-16 so this loop's capacity stays on the demo-unblock and application-onboarding work.

## Objectives

### O1: Unblock resink-core's start by landing the two carried-over ADRs

Why it matters: Resink-core cannot produce its first product-shaped OKR until the engine-choice and deployment-target decisions are committed. Both ADRs were drafted at 2026-05-08 but neither was finalized. Without finalization, the entire loop slips into another bootstrap-shaped cycle. Closing these two decisions is the single highest-leverage move available this loop.

**Key results**
- KR1.1: A streaming engine choice ADR exists in `board/decisions/` with `status: active` and a one-paragraph recommendation between Spark Structured Streaming and Flink, grounded in `fact_sign_up.parquet` constraints.
- KR1.2: A deployment-target ADR exists in `board/decisions/` with `status: active` selecting one of {local Docker, minikube, cloud} for the first demo, with a stated migration path to whichever target the production demo will use.
- KR1.3: Both CEO-level open asks from 2026-05-08 are answered in this brief (above) and acknowledged in DE's and SRE's team OKRs.

**Tasks**
- [ ] Land the engine-choice ADR — owner: teams/platform/data-engineering
- [ ] Land the deployment-target ADR — owner: teams/platform/devops

### O2: Application teams produce their first product-shaped OKRs

Why it matters: The application teams (resink-core, sim-farm) joined the bootstrap loop via charter only. P2 from the 2026-05-08 retro explicitly scheduled their first product-shaped OKRs for this loop. Without this step, the platform teams continue to build infrastructure with no concrete consumer, and the demo target stays abstract.

**Key results**
- KR2.1: `teams/application/resink-core/okrs/2026-05-09-team-okr.md` exists with `status: active` and contains a concrete `fact_sign_up.parquet` Training → Serving plan, scoped to one fact-table type and respecting the engine-choice ADR landing this loop.
- KR2.2: `teams/application/sim-farm/okrs/2026-05-09-team-okr.md` exists with `status: active` and is scoped to validating the resink-core demo pipeline — generation breadth is explicitly out of scope this loop.

**Tasks**
- [ ] Draft resink-core's first product-shaped OKR — owner: teams/application/resink-core
- [ ] Draft sim-farm's first product-shaped OKR — owner: teams/application/sim-farm

### O3: Org-os evolution — P2 lands; bottom-up flow ratified for next loop

Why it matters: P2 from 2026-05-08 retro is explicitly scheduled for this loop and supports O2 directly (a worked playbook accelerates application-team onboarding). The bottom-up flow design is a larger structural change that needs CEO ratification this loop so AE can pick up implementation next loop without re-litigating direction. Both keep the org-OS evolving without smuggling changes past the ADR gate.

**Key results**
- KR3.1: `org-os/playbooks/onboard-application-team.md` exists with `status: active`, plus its mandatory ADR in `board/decisions/` with `status: active`.
- KR3.2: The bottom-up flow design ADR (already drafted at `board/decisions/2026-05-09-001-org-os-bottom-up-flow.md` on the design branch) is reviewed and either approved as-is or returned with concrete change requests; if approved, AE's next-loop OKR (drafted in O4) carries the implementation.

**Tasks**
- [ ] Write `org-os/playbooks/onboard-application-team.md` and its ADR — owner: board
- [ ] Review and rule on the bottom-up flow ADR — owner: board

### O4: Operational stewardship — P1 ships; first runbook drafted; AE owns next-loop bottom-up implementation

Why it matters: P1's product portion (the frontmatter-lint CI script) is owed to DevOps from the 2026-05-08 retro and turns the manually-enforced ADR-001 contract into a checked one. SRE's first runbook ("demo pipeline failed validation") was committed against in 2026-05-08 and needs to land before the demo pipeline exists, so the runbook is ready when the pipeline is. AE needs an OKR shape for the bottom-up flow implementation so the next loop can begin execution without a planning gap.

**Key results**
- KR4.1: A frontmatter-lint script wired into CI exists; its location and invocation are documented in `teams/platform/devops/`.
- KR4.2: A first-draft runbook for "demo pipeline failed validation" exists; SRE owns the location.
- KR4.3: AE's team OKR for this loop carries an explicit task to scope the bottom-up flow implementation (size, owner, milestone) so the next loop's brief can hand it to AE for execution.

**Tasks**
- [ ] Land the frontmatter-lint CI script — owner: teams/platform/devops
- [ ] Draft the "demo pipeline failed validation" runbook — owner: teams/platform/sre
- [ ] Scope the bottom-up flow implementation for next loop — owner: teams/platform/agent-engineering

## Risks

- **Two ADRs on the critical path.** If either engine-choice or deployment-target stalls, O2 lands as a half-complete OKR for resink-core. Mitigation: CEO confirmation of `fact_sign_up.parquet` (above) lets resink-core draft the Training half of the demo even if either ADR slips.
- **High artifact count for one loop.** Six teams plus three ADRs (DE, DevOps, P2 playbook) plus a CI script plus a runbook draft is dense. Mitigation: bottom-up flow implementation is review-only this loop, not execution; sim-farm's OKR is scoped narrow on purpose.
- **Bottom-up flow ADR review may surface design changes.** If the CEO returns the bottom-up flow ADR with material changes, AE's KR4.3 (scoping next-loop implementation) is harder to land. Mitigation: review the ADR early in the loop so AE has time to absorb any changes before drafting the scope.
- **Application teams have not run a build phase before.** Their first OKRs may be over- or under-scoped. Mitigation: P2 playbook lands this loop; CEO consolidation calls out scope-calibration as a learning input for the retro.

## Out of scope this loop

- Bottom-up flow implementation (deferred to loop 2026-05-16 per O3 ratification).
- Product code beyond OKR-level plans for application teams; resink-core's pipeline is planned this loop, not built.
- Phase 2 automation work (P3 from 2026-05-08 retro).
- Broader fact-table coverage; only `fact_sign_up.parquet` this loop.
