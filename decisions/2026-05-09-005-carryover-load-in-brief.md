---
layout: default
title: "ADR 2026-05-09-005: carryover load in brief"
date: 2026-05-09
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-09
  status: active
  decision: Every CEO brief includes a "Carryover load by team" tally so brief commitments visibly account for deferred work entering the loop.
-->
# ADR 2026-05-09-005: Carryover-load tally in the CEO brief

## Context

P1 from the 2026-05-09 CEO retro. Loop 2026-05-09 closed with four teams carrying deferred work into the next loop (DE shared-primitive contract; DevOps frontmatter-lint script; SRE runbook draft; AE bottom-up flow sizing). The deferrals were deliberate CEO scope decisions, but the brief authored at the start of the next loop did not surface cumulative deferred load before committing new objectives. The pattern risk is that each loop's brief silently inherits last loop's overhang while sizing new work as if from a clean slate; over three or four loops this compounds into capacity that is overcommitted on paper and underdelivered in practice. The CEO brief ritual already requires reading every team's most recent exec summary (step 2 "Score the gap"); what is missing is a structured artifact the brief must produce so the carryover is visible at commit time rather than discovered at exec-summary time.

## Decision

Every CEO brief MUST include a "Carryover load by team" subsection inside its Context section, structured as a table with four columns: **Team**, **Carried items** (one-line summaries of deferred or in-flight items entering the loop), **Sized** (S / M / L T-shirt size — L items get re-shape scrutiny), and **Re-shape note** (how the carryover is being folded into this loop's objectives, or "carries again" if it slips further). The table is produced by reading each team's most recent exec summary and `status.md` during step 2 of the CEO-brief ritual. The brief's Risks and Out-of-scope sections must reference this tally when committing new work or deferring further. The 2026-05-23 CEO brief (`board/okrs/2026-05-23-ceo-brief.md`) is the worked example: it carries the full six-team table inline and references it in the per-objective shaping. The ritual file `org-os/rituals/ceo-brief.md` is updated to add this subsection requirement to step 2.

## Alternatives considered

- **A: Free-text carryover paragraph.** Rejected because unstructured prose loses the per-team accountability that drives the tally's value — a missing team in a table is visible; a missing team in a paragraph is invisible.
- **B: A separate `carryover.md` artifact per loop.** Rejected because the brief is already the loop's commitment document; splitting carryover into a sibling file lets the brief commit new work without the carryover constraint visible on the same page.
- **C: Status quo with retro-only catch.** Rejected because retro is end-of-loop; the cost of an over-committed loop is paid before retro can correct it.
- **D: Automated tooling to compute carryover from exec summaries.** Deferred to Phase 2; the manual table is light enough for the CEO to author by hand and the structure is what matters first.

## Consequences

- **Positive:** The CEO brief's commit step now visibly reckons with the loop entry state. Teams whose carryover load is consistently L get re-shape scrutiny in the brief rather than at retro. Cross-loop deferral pattern (the "single-loop slip" pathology that ADR-2026-05-16-002 separately addresses for AE) becomes visible across all teams.
- **Negative / costs:** ~15 minutes of additional authoring per brief. The CEO must read all team exec summaries (the ritual already requires this for "Score the gap" — the marginal cost is the table itself, not the reading).
- **Follow-ups required:** Apply this convention starting with the 2026-05-23 brief (already complete by ratification time). Retro at 2026-05-23 evaluates whether the S/M/L sizing column is useful or noise.

## Links

- Triggering retro: [board/retros/2026-05-09-ceo-retro.md](../retros/2026-05-09-ceo-retro.md) (P1)
- Ritual updated: [org-os/rituals/ceo-brief.md](../../org-os/rituals/ceo-brief.md)
- Worked example: [board/okrs/2026-05-23-ceo-brief.md](../okrs/2026-05-23-ceo-brief.md) § "Carryover load by team"
- Related ADRs: [2026-05-08-001-frontmatter-validation](2026-05-08-001-frontmatter-validation.md), [2026-05-09-006-org-os-change-routing](2026-05-09-006-org-os-change-routing.md)
