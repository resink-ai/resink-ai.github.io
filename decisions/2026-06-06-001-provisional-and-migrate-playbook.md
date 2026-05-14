---
layout: default
title: "ADR 2026-06-06-001: provisional and migrate playbook"
date: 2026-06-06
status: active
type: adr
owner: board
parent: Decisions (ADRs)
nav_order: 3
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-06-06
  status: active
  decision: "Codify the 'provisional-and-migrate' coordination pattern as a documented org-os playbook, with three worked examples from prior loops"
-->
{% raw %}

# ADR 2026-06-06-001: Provisional-and-Migrate Coordination Playbook

## Context

The "provisional-and-migrate" pattern has now been used three times in three consecutive loops:

1. **2026-05-30:** AE built three role files (`org-os/roles/{EM,CEO,IC}.md`) in parallel with the board's enum-extension ratification (`ADR-2026-05-23-001`). AE filed the files with `type: rfc` provisionally (the workaround type, because `type: role` wasn't admitted yet) plus an in-body migration note pointing at the same-loop enum extension. Once the board's enum extension landed `active` mid-loop, the parent agent migrated the frontmatter `rfc → role`.

2. **2026-06-06 (this loop):** Board filed the first `board/actions/<date>-<slug>.md` ticket with `type: action` provisionally (the type isn't in the conventions enum yet) plus an in-body note about the 2026-06-13 enum-extension migration (P2 from this loop's retro + P3 admit-`action`).

3. **2026-06-06 → 2026-06-13 (pending):** Sim-farm inlined the schema-JSON shape in its verdict contract this loop; DE accepted the in-loop request and committed to canonicalize the spec under `teams/platform/data-engineering/conventions/dim-schema-json.md` next loop. The provisional spec lives in sim-farm's contract; the migration moves it to DE's `conventions/` tree.

AE's status.md (refreshed at 2026-05-30) names the pattern as a standing practice. The pattern is graduating from one-off coordination move to a reproducible playbook; without an explicit document, future agents must rediscover it from prior loops.

## Decision

Codify the provisional-and-migrate pattern as a first-class org-os playbook at `org-os/playbooks/provisional-and-migrate.md` (`type: playbook`) so future ICs can apply it from cold rather than rediscover it from prior loops. The playbook documents:

- **When to use** — team A's work depends on team B's same-loop or one-loop-out output (typically a conventions change, contract change, enum extension, or new convention admission). Team A's deliverable would be malformed under current rules but well-formed under the upcoming change.
- **Shape** — team A files with provisional frontmatter + an explicit in-body provisional note naming (1) the canonical type/shape post-migration, (2) which board work ships the canonical change, (3) when the migration will happen. Team B ratifies the canonical change. Parent/board housekeeping (or team A themselves, if same-loop) migrates frontmatter and removes the provisional note once the canonical change lands `active`.
- **Worked examples** — three from the prior three loops (AE `rfc → role` at 2026-05-30; board `type: action` provisional at 2026-06-06; sim-farm schema-JSON spec inline at 2026-06-06 → DE canonicalization at 2026-06-13).
- **Anti-pattern** — do not use when the canonical change is more than one loop away; instead use the standard request mechanism with the `deferred_to_loop` field per [ADR-2026-06-06-002](2026-06-06-002-paused-team-request-acceptance.md).

Additionally, cross-link the new playbook from `org-os/conventions.md` near the frontmatter section so a reader looking at the type enum has a one-hop path to "what to do when your artifact's canonical type isn't admitted yet."

The pattern's reuse rate (3-in-3-loops) is the load-bearing argument for codification this loop rather than waiting for a 4th instance.

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
{% endraw %}
