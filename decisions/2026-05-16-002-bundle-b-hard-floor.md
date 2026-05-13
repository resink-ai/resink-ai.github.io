---
layout: default
title: "ADR 2026-05-16-002: bundle b hard floor"
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
  decision: "AE's Bundle B (7 org-os rituals tasks) becomes the hard floor for loop 2026-05-11-1113; AE refuses other loop-floor work until Bundle B ships or is explicitly blocked"
-->
{% raw %}

# ADR 2026-05-16-002: Bundle B Hard Floor at Loop 2026-05-11-1113

## Context

AE's Bundle B (T6 `team-intake.md` + edits to T7 `executive-loop.md`, T8 `ceo-brief.md`, T9 `team-planning.md`, T10 `exec-summary.md`, T11 `ceo-consolidation.md`, T12 `retro.md`) was scheduled as a loop floor for 2026-05-10 (slipped — mid-loop ADR-gate slip on ADR `2026-05-09-001`) and again for 2026-05-16 (deferred — CEO brief explicit MVP-focus decision). Different causes, same surface effect: B has slipped two consecutive loops. AE flagged this in their 2026-05-16 exec summary as a pattern, not a one-off, with the explicit observation that "the status quo of single-loop slips is no longer acceptable as an implicit answer."

## Decision

The 2026-05-23 CEO brief activates Bundle B as AE's **hard floor** for the loop. AE refuses all non-Bundle-B floor pulls until the 7 Bundle B tasks ship OR are explicitly blocked on a non-AE dependency. The refusal applies to every category of pull AE would otherwise be eligible for: no codegen pattern expansion (no `scd1_first_event`, no `window_stats_with_decrement`); no MVP-shaped deliverables; no DISPATCH.md addenda (including the `--bare` addendum carried from 2026-05-16, which defers to 2026-05-30 alongside Bundle C); no charter work beyond housekeeping. The seven tasks are atomic:

- T6: create `org-os/rituals/team-intake.md`.
- T7: add a team-intake step to `org-os/rituals/executive-loop.md`.
- T8: add a ratification step to `org-os/rituals/ceo-brief.md`.
- T9: add three-source decomposition (brief / cross-team-request / team-initiated) to `org-os/rituals/team-planning.md`.
- T10: add request-lifecycle closure to `org-os/rituals/exec-summary.md`.
- T11: add request roll-ups to `org-os/rituals/ceo-consolidation.md`.
- T12: add a one-line request closure note to `org-os/rituals/retro.md`.

Definition of done per task: the ritual file reflects the bottom-up flow design from ADR `2026-05-09-001`; the tenant-isolation dry-run still passes after each edit; cross-cutting consistency holds (cross-file references resolve). Bundle B is shipped when all 7 task files have landed; partial completion does not count.

Coordination: ADR-2026-05-10-003 (verify-state-claims) also ratifies this loop and edits `org-os/rituals/ceo-brief.md`, `exec-summary.md`, and `ceo-consolidation.md`. AE coordinates merge ordering with the board in two waves: Wave 1 (T6, T7, T9, T10, T11, T12) ships without dependency on ADR-003; Wave 2 (T8 `ceo-brief.md`) lands after the board's ADR-003 and ADR-005 edits to `ceo-brief.md` so the same file is edited once with all changes folded in.

## Alternatives considered

- **A: Re-slice Bundle B into 2-3 smaller chunks that absorb alongside MVP-shaped work in subsequent loops.** Rejected because (a) re-slicing is itself work that AE absorbs from the same bandwidth pool; (b) the existing 7-task bundle is already small (one ritual file per task — atomic edits); (c) two consecutive loops without B means the cost of further slip exceeds the cost of explicit prioritization.
- **B: Status quo (B keeps slipping a loop at a time until "the right loop").** Rejected because the pattern is now visible: B loses to any CEO-priority pull because it has no external consumer pressure. Three loops without B will not differ from two.
- **C: Move Bundle B to a different owner.** Rejected because B is AE's core competence (org-os ritual authoring) and AE is the team that drafted the 17-task plan; transfer would re-introduce context cost.

## Consequences

- **Positive:** Bundle B ships in a known loop; the request lifecycle becomes wired end-to-end; team-intake gains a ritual that lets bottom-up proposals surface mechanically. AE's loop becomes predictable (one objective, one floor) which reduces planning friction. Pattern of single-loop slips broken.
- **Negative / costs:** AE cannot take MVP-shaped pulls for one loop. If a critical bug emerges in the codegen path during 2026-05-23, AE either (a) breaks the floor or (b) routes the fix through a workaround owned by resink-core. The CEO brief at 2026-05-23 must explicitly avoid scheduling AE on codegen work.
- **Follow-ups required:** 2026-05-23 brief's "team assignments" section names AE only against Bundle B. 2026-05-23 OKR for AE is single-objective. ADR-003 (verify-state-claims) lands in the same loop, also touching `org-os/rituals/` — coordinate file ordering to avoid merge conflicts.

## Links

- Triggering retro: [board/retros/2026-05-11-0958-ceo-retro.md](../retros/2026-05-11-0958-ceo-retro.md) — P2.
- Related ADRs: [2026-05-09-001-org-os-bottom-up-flow](2026-05-09-001-org-os-bottom-up-flow.md) (the 17-task plan B implements); [2026-05-10-003-verify-state-claims-at-ritual-transitions](2026-05-10-003-verify-state-claims-at-ritual-transitions.md) (sibling 2026-05-23 ritual edit).
- AE's status carrying Bundle B context: [`teams/platform/agent-engineering/status.md`](../../teams/platform/agent-engineering/status.md).
- AE's 2026-05-16 exec summary flagging the pattern: [`teams/platform/agent-engineering/exec-summaries/2026-05-11-0958.md`](../../teams/platform/agent-engineering/exec-summaries/2026-05-11-0958.md).
{% endraw %}
