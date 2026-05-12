---
layout: default
title: "ADR 2026-06-06-001: provisional and migrate playbook"
parent: "Decisions (ADRs)"
render_with_liquid: false
date: 2026-06-06
status: draft
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-06-06
  status: draft
  decision: "Codify the 'provisional-and-migrate' coordination pattern as a documented org-os playbook, with three worked examples from prior loops"
-->
# ADR 2026-06-06-001: Provisional-and-Migrate Coordination Playbook

## Context

The "provisional-and-migrate" pattern has now been used three times in three consecutive loops:

1. **2026-05-30:** AE built three role files (`org-os/roles/{EM,CEO,IC}.md`) in parallel with the board's enum-extension ratification (`ADR-2026-05-23-001`). AE filed the files with `type: rfc` provisionally (the workaround type, because `type: role` wasn't admitted yet) plus an in-body migration note pointing at the same-loop enum extension. Once the board's enum extension landed `active` mid-loop, the parent agent migrated the frontmatter `rfc → role`.

2. **2026-06-06 (this loop):** Board filed the first `board/actions/<date>-<slug>.md` ticket with `type: action` provisionally (the type isn't in the conventions enum yet) plus an in-body note about the 2026-06-13 enum-extension migration (P2 from this loop's retro + P3 admit-`action`).

3. **2026-06-06 → 2026-06-13 (pending):** Sim-farm inlined the schema-JSON shape in its verdict contract this loop; DE accepted the in-loop request and committed to canonicalize the spec under `teams/platform/data-engineering/conventions/dim-schema-json.md` next loop. The provisional spec lives in sim-farm's contract; the migration moves it to DE's `conventions/` tree.

AE's status.md (refreshed at 2026-05-30) names the pattern as a standing practice. The pattern is graduating from one-off coordination move to a reproducible playbook; without an explicit document, future agents must rediscover it from prior loops.

## Decision

(Placeholder — to be expanded loop 2026-05-11-2153.) Author a new playbook at `org-os/playbooks/provisional-and-migrate.md` with `type: playbook` codifying the pattern:

- **When to use:** When team A's work depends on team B's same-loop or future-loop output (typically a conventions change, contract change, enum extension, or new convention admission). Team A's deliverable would be malformed under current rules but well-formed under the upcoming change.

- **Shape:** Team A files the artifact with provisional frontmatter (existing workaround value, or new type with explicit in-body provisional note). The in-body note states: (1) the canonical type/shape it WILL have post-migration, (2) which board work is shipping the canonical change, (3) when the migration will happen. Team B ratifies the canonical change in their parallel work stream. Parent/board housekeeping (or team A themselves, if same-loop) migrates the frontmatter once the canonical change lands `active`.

- **Worked examples (cite from prior loops):**
  - AE rfc → role at 2026-05-30 (mid-loop migration)
  - Board `type: action` provisional at 2026-06-06 (next-loop migration)
  - Sim-farm schema-JSON inline at 2026-06-06 → DE canonicalization at 2026-06-13 (next-loop content migration)

- **Anti-pattern:** Do not use provisional-and-migrate when the canonical change is more than one loop away — the provisional artifact then accrues edit risk across multiple loops with no clear migration boundary. For >1-loop-out work, use the standard request mechanism with `status: accepted-deferred` instead (per ADR-2026-06-06-002, also drafted this loop).

## Alternatives considered

- **A: Status quo (no playbook, pattern reused by precedent).** Rejected because the pattern is now 3-uses-in-3-loops; without a playbook, future ICs (especially new agents) won't recognize when to apply it and may file blocking-instead-of-provisional artifacts.
- **B: Codify only in `org-os/conventions.md` (one section).** Rejected because the pattern is operational (when/how) not just a definitional convention; a playbook with worked examples is the right format.
- **C: Extend the request-mechanism playbook to cover this case.** Rejected because the request mechanism is for cross-team asks via the `requests/` directory; provisional-and-migrate is about same-team artifact filings under upcoming conventions. Distinct enough to warrant its own playbook.

## Consequences

- **Positive:** Future agents can apply the pattern from cold rather than rediscovering it. Cross-team coordination work that depends on board enum/contract extensions becomes a 1-action playbook lookup. The pattern's reuse rate has been 3-in-3-loops; codification at 4+ loops would have been delayed too long.
- **Negative / costs:** One more playbook for new agents to read. The playbook is small (~50-80 lines expected); the cost is bounded.
- **Follow-ups required:** Author `org-os/playbooks/provisional-and-migrate.md`. Cite three worked examples by file path. Cross-link from `org-os/conventions.md` "frontmatter" section.

## Links

- Triggering retro: [board/retros/2026-05-11-1631-ceo-retro.md](../retros/2026-05-11-1631-ceo-retro.md) — P2.
- Sister ADR (drafted same loop): [2026-06-06-002-paused-team-request-acceptance](2026-06-06-002-paused-team-request-acceptance.md) — same retro, different pattern.
- Prior worked examples: AE `rfc → role` at [board/exec-summaries/2026-05-11-1302.md](../exec-summaries/2026-05-11-1302.md); board `type: action` at [board/actions/2026-06-06-001-create-resink-core-github-remote.md](../actions/2026-06-06-001-create-resink-core-github-remote.md).
