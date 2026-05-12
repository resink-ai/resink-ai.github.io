---
layout: default
title: "ADR 2026-05-30-002: multi loop plan relative dating"
date: 2026-05-30
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-30
  status: active
  decision: "Multi-loop plans inside ADRs name dates as `loop+N` relative to the ADR's ratification loop, with an explicit 'subject to team capacity at that loop's brief' qualifier; absolute calendar dates only when external commitments require them"
-->
# ADR 2026-05-30-002: Multi-Loop Plan Relative Dating

## Context

ADR-2026-05-16-001 (Option A → dlopen restoration plan) named absolute dates: AE template extension at 2026-05-30, supervisor swap at 2026-06-06, hot-swap test the loop after. At 2026-05-30, AE's full bandwidth was consumed by Bundle C (ratified by ADR-2026-05-16-002 hard floor); the template extension slipped to 2026-06-06 without a recorded slippage mechanism inside the ADR.

The slippage isn't an ADR re-litigation — the plan is still correct, the dates just don't account for cumulative team-capacity carryover. But absolute-date plans assume linear team capacity; they don't account for the team being held by another commitment in the named window.

## Decision

When an ADR records a multi-loop plan, every step inside that plan names its loop **relative to the ratification loop** (the loop in which the ADR flipped to `status: active`):

- Step 1 = `loop+1`, step 2 = `loop+2`, etc. The ratification loop itself is `loop+0` — used when a step is bundled with the ratification.
- Each named loop step carries an explicit "subject to team capacity at that loop's brief" qualifier. Brief authoring at the named loop confirms the step lands; absence triggers a brief-level slippage record (not an ADR re-litigation).
- Absolute calendar dates (e.g., `2026-06-06`) appear **only** when an external commitment requires them — customer milestones, conference dates, regulatory deadlines. In that case, the ADR explicitly names the external commitment as the date's anchor (e.g., "must land by 2026-09-30 — Q3 customer release"). Without a named external anchor, dates must use loop+N form.

**Worked example (ratified this loop):** ADR-2026-05-16-001's plan rewritten under the convention reads: step 1 (AE template extension) = `loop+1` (which slipped to `loop+2` at the 2026-05-30 brief; recorded in that loop's brief, not in the ADR); step 2 (resink-core supervisor swap) = `loop+2` (re-targeted to `loop+3` per this loop's brief); step 3 (hot-swap correctness test) = `loop+3` (re-targeted to `loop+4`). All slippage records live in the loop briefs; the ADR body stays as authored. The convention is **demonstrated** in the same loop in which it ratifies — this loop's brief uses `loop+1` / `loop+2` notation to describe the next two steps of the ADR-2026-05-16-001 plan.

**Grandfathering:** Existing ADRs with absolute-date multi-loop plans do not need to be re-edited; their plans grandfather under the absolute-date convention. Future ADRs and ADRs that materially revise their multi-loop plan use the loop+N convention.

## Alternatives considered

- **A: Status quo (absolute dates).** Rejected because the pattern just bit ADR-2026-05-16-001 within one loop of its ratification.
- **B: No date naming at all (just "next loop", "loop after").** Rejected because "next loop" is ambiguous when the ADR ratifies mid-loop or when the next loop is a focus loop that defers everything else.
- **C: Date with mandatory carryover-tally check.** Rejected as too heavy; the qualifier ("subject to team capacity") captures the same idea without adding a forcing function.

## Consequences

- **Positive:** Multi-loop plans become resilient to single-loop carryover slips. Readers parsing an ADR understand the dates as "earliest reasonable" rather than "committed schedule." Brief authoring doesn't have to re-litigate the ADR every time a single loop's capacity squeezes. The slippage record lives at the loop boundary (in the brief) where the carryover analysis already happens.
- **Negative / costs:** The convention is lighter than absolute dates; readers wanting a hard timeline have to consult the upcoming loop's brief rather than the ADR. ADRs become less timeline-precise. Mitigated by the carryover-load table per ADR-2026-05-09-005 — each loop's brief makes the actual-vs-planned schedule observable.
- **Follow-ups required:** `org-os/templates/adr.md` gains a "Multi-loop plan note" subsection capturing the loop+N convention (this loop). ADR-2026-05-16-001 grandfathers (no retroactive edit). Future multi-loop ADRs comply.

## Alternatives considered

- **A: Status quo (absolute dates).** Rejected because the pattern just bit ADR-2026-05-16-001 within one loop of its ratification.
- **B: No date naming at all (just "next loop", "loop after").** Rejected because "next loop" is ambiguous when the ADR ratifies mid-loop or when the next loop is a focus loop that defers everything else.
- **C: Date with mandatory carryover-tally check.** Rejected as too heavy; the qualifier ("subject to team capacity") captures the same idea without adding a forcing function.

## Consequences

- **Positive:** Multi-loop plans become resilient to single-loop carryover slips. Readers parsing an ADR understand the dates as "earliest reasonable" rather than "committed schedule." Brief authoring doesn't have to re-litigate the ADR every time a single loop's capacity squeezes.
- **Negative / costs:** The convention is lighter than absolute dates; readers wanting a hard timeline have to consult the upcoming loop's brief rather than the ADR. ADRs become less timeline-precise.
- **Follow-ups required:** Update `org-os/templates/adr.md` with the loop+N convention. Optionally update ADR-2026-05-16-001 retroactively (cite the loop+N convention as a corrigendum) — but grandfathering is also acceptable.

## Links

- Triggering retro: [board/retros/2026-05-11-1302-ceo-retro.md](../retros/2026-05-11-1302-ceo-retro.md) — P4.
- Worked-example slippage: [ADR-2026-05-16-001](2026-05-16-001-abi-option-a-mvp-deviation.md) — AE template extension at named 2026-05-30 slipped one loop to 2026-06-06.
- Sister convention: this ADR + ADR-2026-05-16-003 + ADR-2026-05-30-001 form the "verify-and-honest-about-dates" family.
