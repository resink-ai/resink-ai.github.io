---
layout: default
title: Retro
nav_order: 3
parent: "Loop 2026-05-10-2227-002"
grand_parent: "Loops"
render_with_liquid: false
date: 2026-05-10
status: active
type: retro
loop: 2026-05-10-2227-002
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-10
  status: active
  loop: 2026-05-10
  links: parent: board/exec-summaries/2026-05-10-2227-002.md
-->
# Resink.ai CEO Retro — 2026-05-10

## What worked

- **Re-baselining as an explicit CEO framing.** The brief named "this is a re-baseline loop; absorb the pivot, do not start building" as a first-class decision. Four teams (DevOps, SRE, resink-core, sim-farm) stayed plan-only without re-litigating scope; the two teams with build deliverables (DE, AE) hit them all. Reproducibility: when a vision update arrives mid-cadence, the next loop's brief should explicitly name "plan-only" vs "build" framing per team rather than letting each team negotiate it during planning.
- **Pre-deciding ADR fate at brief time.** The brief named ADR-002's fate (archive + supersede) and ADR-003's fate (preserve) before team planning began. Every team's OKR operated against a settled ADR landscape; no team had to guess whether to cite a prior ADR or invent its own re-aim. Compare 2026-05-08→09, where the engine-choice ADR was "to be decided in team-planning" and downstream artifacts were partial. Reproducibility: when a CEO-level pivot conflicts with prior ADRs, name each affected ADR's fate in the brief's context section, not as a team OKR decision.
- **Pre-deciding team ownership of new sub-projects at brief time.** The brief assigned the five nanofab sub-projects to teams (with #5 Product UX deferred) before team planning began. The alternative — letting teams negotiate ownership during their own planning — would have multiplied coordination cost on the loop's single largest organizational decision. Reproducibility: when a vision update introduces new sub-projects, the brief assigns ownership; teams' planning is decomposition, not negotiation.
- **CEO answered the standing manifest-tool ask in the brief.** DevOps's 2026-05-09 ask (Helm vs. Kustomize vs. raw YAML) had been open across one loop boundary. Answering it in this loop's brief context let DevOps plan against a settled answer; the Helm-fit walk of the serving spec found no blocker, so the answer survived. Reproducibility: when a team carries a CEO-direct ask across a loop boundary, the next brief should answer it in the context section — not defer it to team-planning, where it becomes a dependency the team has to work around.
- **Single-document multi-consumer hand-offs replicated.** DE's Kafka ingress contract is a worked example of P3 from the 2026-05-09 retro: one document, eight named sections, four named consumers (resink-core, sim-farm, SRE, serving-deployment). Resink-core, sim-farm, and SRE all reference it from their summaries. Two-loop track record — the pattern is now reliably reproducible, not just observed once.
- **Surface constraints early, write the contract second.** Resink-core wrote consumer-side Kafka constraints into its OKR's named hand-off section before DE drafted the contract; DE absorbed them. Same pattern as the 2026-05-09 ADR-002 / resink-core dynamic. Two-loop track record. Reproducibility: when a contract crosses a team boundary, the consumer's OKR carries surfaced constraints first; the producer writes the contract second, citing those constraints.

## What didn't

- **"Reserved on master" was a false state-claim at loop open.** The 2026-05-09 exec summary recorded that ADR `2026-05-09-001-org-os-bottom-up-flow` was "merged to master via the design branch's PR" with "the slot is reserved on master." Neither was true entering loop 2026-05-10-2227-002: the file was on no branch's `board/decisions/`, and the design branch's PR had not landed. AE flagged the absence as a hard blocker for Bundle A; the board wrote the ADR inline mid-loop to open the gate. **Root cause:** the exec-summary ritual today accepts textual "shipped" claims without a verification step. The next loop's dependent teams pay for any false claim. Pattern risk: any unverified "shipped" or "merged" claim in an exec summary becomes a hidden blocker for the next loop. (Highest-priority retro item.)
- **Source materials lived on a branch the dependent loop did not see.** Related root cause: the bottom-up flow design spec (`docs/superpowers/specs/2026-05-09-org-os-bottom-up-flow-design.md`) and implementation plan (`docs/superpowers/plans/2026-05-09-org-os-bottom-up-flow.md`) were on the `org-os-bottom-up-flow-design` branch, not on the loop-2026-05-09 working branch. They were cherry-picked mid-loop. **Root cause:** cross-branch coordination has no protocol; a team's loop floor depends on artifacts living on its working branch, but the brief authoring step does not verify this. Pattern risk: same as #1 — any cross-branch dependency becomes a discoverability gap.
- **Resink-core's two-sub-project stretch is unsized for next-loop implementation.** This loop's plan-only framing avoids the immediate problem, but next loop the team must size implementation across both #1 Runtime and #2 Training Pipeline. The CEO brief acknowledged the risk under "Risks" but did not constrain the next loop's planning shape — for example, by naming which sub-project's first-slice gets priority. **Root cause:** the brief's risk section is descriptive ("this is risky"), not prescriptive ("here is the constraint that mitigates it"). When a multi-loop stretch is named as a risk, the brief should either (a) constrain the next loop's planning shape, or (b) defer scope sizing to a brief-time decision in the next loop. Today it does neither.

## Evolution proposals

### P1: Verify state-claims at ritual transitions (class: **org-os**)

- **Problem it solves:** "What didn't" #1 and #2. The exec-summary and ceo-brief rituals today accept textual claims about merge state and artifact location without verification. False or stale claims become hidden blockers in the next loop.
- **Proposed change:** Update two rituals.
  - `org-os/rituals/exec-summary.md`: any item claimed as "shipped to master," "merged," or "reserved on master" must carry a verifiable git location (`verified-at: <ref/sha>` or `on-branch: <name>`). Items lacking verification are recorded as "shipped to branch `<X>`, awaiting merge."
  - `org-os/rituals/ceo-brief.md`: a sub-step in step 2 ("Score the gap") that verifies every artifact referenced as an input is present on the brief's working branch; missing artifacts surface as a named "merge/cherry-pick action" in the brief context, with an owner.
  - Optionally, `org-os/rituals/ceo-consolidation.md` step 4 (or a new step) walks per-team summaries and flags any unverified "shipped" claim before publishing the company summary.
- **Review path:** ADR mandatory for the `org-os` portions. Draft placeholder created at [`board/decisions/2026-05-10-003-verify-state-claims-at-ritual-transitions.md`](../decisions/2026-05-10-003-verify-state-claims-at-ritual-transitions.md).
- **Owner:** board.
- **Timing:** **deferred to loop 2026-05-11-0958.** Same-loop bandwidth conflicts with AE's Bundle B (rituals editing); stacking two retro-class ritual changes on the same files would multiply merge friction. P1 picks up in the loop after Bundle B lands.

### P2: Add a `contract` (or unified hand-off-document) type to `org-os/conventions.md` (class: **org-os**)

- **Problem it solves:** DE's surfaced ask during build. DE's Kafka ingress contract is a hand-off document with named consumer sections — the natural type. The current conventions.md enum does not include `contract`, so the document was filed as `rfc` as a workaround. As more contracts land (Helm chart shape, supervisor `--mode=sim` interface, sim-farm verdict format), the workaround compounds.
- **Proposed change:** Add `contract` (or a unified hand-off-document type — `handoff`, `interface`, name TBD) to the type enum in `org-os/conventions.md`. Add an "Additional fields per type" row: `consumers: [<list of team paths>]`, `producers: [<team path>]`. Pairs naturally with the named-hand-off-sections convention (P3 from 2026-05-09 retro).
- **Review path:** ADR mandatory for the `org-os` portion. Draft placeholder created at [`board/decisions/2026-05-10-004-contract-artifact-type.md`](../decisions/2026-05-10-004-contract-artifact-type.md).
- **Owner:** board (with input from DE on the additional-fields shape).
- **Timing:** **deferred to loop 2026-05-11-1113.** Pairs naturally with ADR-005 (carryover-load section in brief) and ADR-006 (out-of-retro routing path) — all three are small conventions/ritual edits and land more cleanly batched.

### P3: Next-loop brief picks resink-core's first-slice sub-project explicitly (class: **product**)

- **Problem it solves:** "What didn't" #3. Resink-core's two-sub-project stretch needs a brief-time constraint, not a per-team negotiation, so next-loop planning has a settled scope. The CEO's first-slice choice has natural downstream implications (sim-farm's `--mode=sim` interface needs a supervisor binary first; DevOps's Helm chart wants a deployable artifact first; both point to #1 Runtime).
- **Proposed change:** The 2026-05-16 CEO brief explicitly names whether resink-core's next first-slice prioritizes #1 (Runtime) or #2 (Training Pipeline), naming the other as next-after. Default direction: **#1 Runtime first** — the supervisor binary is on the critical path for sim-farm's mode-A integration and DevOps's chart skeleton; #2 Training Pipeline can stub the manifest output.
- **Review path:** Product change; lives inside next CEO brief authoring. No ADR required.
- **Owner:** board (next-loop brief authoring).
- **Timing:** **picked up loop 2026-05-11-0958** (brief authoring step).

### P4: Stand up a Product UX team in a future loop (class: **tenant**)

- **Problem it solves:** Cross-cutting blocker — sub-project #5 (Product UX) has no owner per ADR `2026-05-10-002`. Customer-facing surface work cannot begin until a team is named. The spec exists; only ownership is missing.
- **Proposed change:** A future CEO brief (loop 2026-05-11-0958 or 2026-05-23) includes an objective to charter a Product UX team. Use `org-os/playbooks/onboard-application-team.md` for the application-team standup. The team's first product-shaped OKR follows the same pattern resink-core and sim-farm did when they joined.
- **Review path:** Tenant decision (charter shape + team name within this tenant). No ADR required — uses an existing org-os playbook.
- **Owner:** board (next-or-after-next CEO brief authoring).
- **Timing:** **deferred — not committed to a specific loop yet.** Decision pending: do we have capacity to onboard another team while resink-core stretches over two sub-projects? Revisit when resink-core has shipped its first-slice and demonstrated capacity headroom.

## Decisions to record

- **ADR-003 (P1)** "Verify state-claims at ritual transitions." Draft placeholder at [`board/decisions/2026-05-10-003-verify-state-claims-at-ritual-transitions.md`](../decisions/2026-05-10-003-verify-state-claims-at-ritual-transitions.md). Status: `draft`. Owner: board. Picked up loop 2026-05-11-0958.
- **ADR-004 (P2)** "Contract (or unified hand-off-document) artifact type." Draft placeholder at [`board/decisions/2026-05-10-004-contract-artifact-type.md`](../decisions/2026-05-10-004-contract-artifact-type.md). Status: `draft`. Owner: board. Picked up loop 2026-05-11-1113.
- (P3 needs no ADR — product-class, lives in next CEO brief authoring.)
- (P4 needs no ADR — tenant-class, uses existing onboard-application-team playbook.)

## Carryover ADRs still on the books

- **ADR-005 (P1 from 2026-05-09 retro):** "Carryover load tally added to CEO brief authoring step." Draft. Picked up loop 2026-05-11-1113.
- **ADR-006 (P2 from 2026-05-09 retro):** "Out-of-retro routing path for org-os change proposals." Draft. Picked up loop 2026-05-11-1113.

ADR-005 + ADR-006 + this retro's ADR-004 batch cleanly at loop 2026-05-11-1113 (three conventions/ritual edits). This loop's ADR-003 lands earlier (2026-05-16) because it is the highest-priority retro item — it directly addresses the gate-slip pattern that bit AE this loop.

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: **passed.** AE ran `grep -rEi "resink|nanofab|acme\.ai" org-os/` after Bundle A landed; no tenant or product names appear inside `org-os/` outside placeholder contexts. The two new template files (`request.md`, `team-proposals.md`) and the conventions changes were inspected as part of the dry-run.
