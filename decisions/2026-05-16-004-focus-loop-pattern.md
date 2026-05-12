---
layout: default
title: "ADR 2026-05-16-004: focus loop pattern"
date: 2026-05-16
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-16
  status: active
  decision: "Codify the focus-loop pattern AND its consolidation-loop sibling as a single playbook at org-os/playbooks/focus-and-consolidation-loops.md; both patterns are CEO-invocable, both have explicit preconditions, both have explicit exit criteria, and they form a paired rhythm (focus loops produce structural deviations; consolidation loops close them)"
-->
# ADR 2026-05-16-004: Codify the Focus-Loop Pattern (Extended with Consolidation-Loop Sibling)

## Context

Two recent loops have exhibited reproducible CEO-invoked loop shapes that fall outside the steady-state "all teams active with their own OKRs" pattern documented in the charter:

- **Loop 2026-05-16** was a CEO-called **focus loop**: four activated teams (resink-core, sim-farm, DE, AE), two paused teams (DevOps, SRE), pre-committed retro items deferred (Bundle B + ADR-003), single closing seam (`make mvp-loop` exits 0 with `verdict=pass`). The loop produced more architectural learning than the prior two loops combined — surfaced the ABI Option A gap, the `--bare` auth deviation, the `_process` C-ABI gap, the `pytz` transitive dep, the predicted-but-nonexistent `claude --skill` flag — and closed its primary deliverable on the first end-to-end attempt with real LLM codegen.
- **Loop 2026-05-23** was a CEO-called **consolidation loop**: all six teams active, every team shipping a carryover from prior loops (resink-core widened the MVP fixture; sim-farm closed the Mode-A multi-dim extension; AE shipped Bundle B after a 3-loop slip; DE flipped contract types from `rfc → contract`; DevOps + SRE returned to build mode after 4 weeks). One closing seam (the widened verdict) anchored integration risk while six independent work streams ran in parallel. The 2026-05-23 retro P5 explicitly named the consolidation-loop pattern as the focus-loop's sibling and recommended codification.

The two patterns are reproducible but un-named. Future CEOs (human or agent) reading the rituals shouldn't have to re-derive the framing from scratch each time the company drifts into one of these shapes. More important: the two patterns form a rhythm — focus loops deliberately introduce structural deviations (Option A, the `--bare` drop, the pytz workaround) in service of a single closing seam; consolidation loops are how those deviations get absorbed back into the shared surface (contract types ratified, ABI restoration scheduled, retro items closed). Documenting them as a pair makes the rhythm legible.

## Decision

Author `org-os/playbooks/focus-and-consolidation-loops.md` as a single playbook naming both patterns side-by-side. Frontmatter: `type: playbook` (legal as of ADR-2026-05-23-001's ratification this same loop). The playbook body names, for each pattern:

**Focus loop.**
- **Precondition (invocation trigger):** the company has shipped two consecutive plan-only loops, OR team OKRs depend on cross-team contracts that have not yet been exercised in code, OR the CEO has identified a single closing-seam deliverable that all active teams' work must converge on.
- **Shape:**
  1. **Activate only teams on the critical path** of one specific closing seam. The brief names the seam explicitly (one observable artifact, one exit criterion — a green test, a passing verdict, a shipped contract — not "a plan is written").
  2. **Pause other teams explicitly** with dated resumption commitments. Paused teams' status files do not change; their carryovers carry intact.
  3. **Defer pre-committed retro items** if they conflict with the focus. The deferral is named in the brief's context section with a forward-pointer to the new target loop. ADR placeholders are not closed; their `status: draft` carries forward.
  4. **Gate on a single mechanical exit criterion** named in the brief and verified at consolidation time.
- **Worked example:** loop 2026-05-16 (MVP-focus loop; closing seam = `make mvp-loop` exits 0 with `verdict=pass mismatches=0`).
- **Exit criterion:** the mechanical artifact named in the brief is green. If green, the loop succeeded regardless of how many other items deferred.

**Consolidation loop.**
- **Precondition (invocation trigger):** the prior loop was a focus loop that introduced named deviations (architectural compromises, deferred retro items, paused teams' carryovers), AND those deviations have accumulated enough mass that addressing them in a parallel pass is cheaper than addressing them one per loop.
- **Shape:**
  1. **Activate every team in parallel.** Each team is asked to ship one carryover or close one named seam (not "ship a new feature stack"); the work is breadth-first across teams, not depth-first within one team.
  2. **Name one closing seam** that anchors integration risk while parallel work runs (e.g., the widened verdict in loop 2026-05-23). The seam validates that parallel work has not drifted apart.
  3. **Ratify deferred org-os evolutions in a coordinated batch.** ADR placeholders that have been carrying their `status: draft` across the focus-loop window flip `→ active` together; contracts under workaround types migrate to their canonical types together. Batching is cheaper than per-loop ratification when more than one item is pending.
  4. **Resume paused teams** with their carryovers intact — no re-planning required.
- **Worked example:** loop 2026-05-23 (six teams active; six-ADR batch ratified; three contracts migrated `type: rfc → type: contract`; AE Bundle B shipped after 3-loop slip; DevOps + SRE returned to build).
- **Exit criterion:** every team produces an exec summary (paused-team format is not used); the named closing seam is green; the ratified batch is in `status: active`; per-team carryovers are closed or explicitly re-deferred with a named target loop.

**The rhythm.** Focus loops produce structural deviations. Consolidation loops close them. The two patterns together form an explicit cadence the CEO uses to manage the company's plan-vs-build balance: focus when a closing seam is overdue or when plans have drifted from code; consolidate when deviations have piled up. Neither pattern is the steady state — the steady state is still "all teams active with their own OKRs" — but both are legitimate CEO tools to invoke when the conditions match.

**Deferral discipline (applies to both patterns).** Deferred items are named with a target loop and a one-line rationale; they do not disappear. The CEO brief's "Carryover load by team" table tracks deferred items across loops. A deferral that slips a third consecutive loop is automatically a retro flag.

The single-playbook framing (both patterns in one document) is deliberate: the patterns are paired, not independent. A reader who finds the focus-loop pattern should immediately see the consolidation-loop pattern as its sibling and understand when each applies.

## Alternatives considered

- **A: Leave the pattern un-named.** Rejected because two CEO-called focus loops (2026-05-10 the pivot-absorption loop; 2026-05-16 the MVP-focus loop) and one consolidation loop (2026-05-23) within four weeks suggest the patterns are recurring; future CEOs deserve a documented framing.
- **B: Document as a hard requirement, not an optional MAY.** Rejected because both patterns are tools, not the default; weekly cadence with all teams active and their own OKRs remains the steady-state shape per the charter.
- **C: Document as a `rituals/` change (modify ceo-brief.md) rather than a new `playbooks/` entry.** Rejected because both patterns are invocation-shaped (CEO decides whether to use), not a step in every brief.
- **D: Two separate playbooks (one for focus, one for consolidation).** Rejected because the rhythm framing — focus produces deviations; consolidation closes them — is the value-add of pairing them in one document. A reader who lands on one pattern would miss half the framing.
- **E: Defer codification of the consolidation-loop sibling to a separate ADR.** Rejected because loop 2026-05-23's retro P5 explicitly named the pairing; codifying both at once costs less authoring than two ADR rounds and keeps the playbook coherent.

## Consequences

- **Positive:** Future CEOs have a documented framing for two recurring loop shapes. The deferral discipline (don't drop pre-committed retro items silently; date them forward; third-loop slip is a retro flag) is encoded. The "mechanical exit criterion" requirement prevents focus-loops from collapsing back into plan-only work. The rhythm framing makes the focus → consolidation cadence legible and reproducible.
- **Negative / costs:** Risk of overuse — if every loop becomes a focus-loop, paused teams' carryovers accumulate; if every loop becomes a consolidation-loop, momentum on new work stalls. Mitigation: the playbook entry names invocation preconditions, not "the CEO feels like it." Three consecutive non-steady-state loops would themselves be a retro flag.
- **Follow-ups required:**
  - Author `org-os/playbooks/focus-and-consolidation-loops.md` (lands alongside this ratification).
  - Cross-reference from `org-os/rituals/ceo-brief.md` to the playbook as one of the "loop shapes the CEO may select."

## Links

- Triggering retros: [board/retros/2026-05-16-ceo-retro.md](../retros/2026-05-16-ceo-retro.md) — P5; [board/retros/2026-05-23-ceo-retro.md](../retros/2026-05-23-ceo-retro.md) — P5 (consolidation-loop sibling).
- Worked example (focus): [board/okrs/2026-05-16-ceo-brief.md](../okrs/2026-05-16-ceo-brief.md), [board/exec-summaries/2026-05-16.md](../exec-summaries/2026-05-16.md).
- Worked example (consolidation): [board/okrs/2026-05-23-ceo-brief.md](../okrs/2026-05-23-ceo-brief.md), [board/exec-summaries/2026-05-23.md](../exec-summaries/2026-05-23.md).
- Companion ADR ratified same loop: [2026-05-16-003-contract-environment-verification](2026-05-16-003-contract-environment-verification.md), [2026-05-23-001-conventions-enum-extension](2026-05-23-001-conventions-enum-extension.md).
- Playbook landed alongside this ratification: [org-os/playbooks/focus-and-consolidation-loops.md](../../org-os/playbooks/focus-and-consolidation-loops.md).
