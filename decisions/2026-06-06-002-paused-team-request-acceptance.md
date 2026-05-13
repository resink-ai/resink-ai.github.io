---
layout: default
title: "ADR 2026-06-06-002: paused team request acceptance"
date: 2026-06-06
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-06-06
  status: active
  decision: "Paused teams accepting in-loop requests record the acceptance canonically in the request file's frontmatter (status field + deferred_to_loop), not in status.md; status.md may reference the request file by path but is not source-of-truth"
-->
{% raw %}

# ADR 2026-06-06-002: Paused-Team Request Acceptance Canonicalization

## Context

Loop 2026-05-11-1631: DE was paused per CEO brief, accepted an in-loop request from sim-farm (`teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md`), and committed to act at loop+1 (= 2026-06-13). The commitment was recorded in three places:

1. DE's exec summary (paragraph form, with deadline)
2. DE's `status.md` § "Carrying into next loop" (one-line)
3. The request file's frontmatter (`status: open` at filing, expected to transition `status: accepted-deferred-to-loop-2026-06-13` at intake)

No canonical source-of-truth. If the three diverge (likely over time as status.md gets refreshed each loop), readers don't know which one to trust. ADR-2026-05-09-006's request-flow lifecycle defines the request states (`open | accepted | declined | fulfilled | dropped | escalated`) but doesn't say where a paused team's "accepted with action in a future specific loop" record lives.

This is a minor bookkeeping friction today, but the pattern is recurring: at 2026-06-06 alone we have one (DE↔sim-farm); at 2026-06-13 with the P1/P2 work landing, we'll have several more. Without a canonical write-down, status.md and request files will drift and the request-flow metrics in CEO consolidation (per ADR-2026-05-09-006 metrics table) become unreliable.

## Decision

Canonicalize the source-of-truth for any request's status (especially for paused teams accepting in-loop with deferred action) in the request file itself. Three concrete changes:

1. **The request file is canonical.** For any request's status field — including a paused team's "accepted with action in a future named loop" record — the request file under `teams/<layer>/<receiver>/requests/<date>-<slug>.md` is source-of-truth. `status.md` and exec summaries MAY reference the request file by path (e.g., "Carrying into next loop: schema-JSON canonicalization per [request 2026-06-06-001](requests/...)"); such references are informational only and grandfather under earlier-loop authoring.

2. **New optional frontmatter field on requests: `deferred_to_loop: <YYYY-MM-DD>`.** Used when a paused team accepts a request in-loop but defers concrete action to a specific named loop. The state transition is the existing `open → accepted` (no new `accepted-deferred` state — the field carries the timing while the state space stays at the seven values defined in `org-os/conventions.md`).

3. **Forcing function at brief-authoring time** (the open question from the draft is decided "yes"). The CEO brief ritual's first read step (per `org-os/rituals/ceo-brief.md` step 1 and ADR-2026-05-09-005's carryover-load discipline) surfaces every `accepted` request whose `deferred_to_loop` matches the loop being authored. Surfacing means: the brief names the request explicitly in its Context section's carryover summary so it cannot be silently absorbed. This is the same shape as the `board/actions/<date>-<slug>.md` `due:` field's forcing function.

Three ritual/template edits land alongside this ratification:

- **`org-os/rituals/team-intake.md`** specifies the request file as canonical source-of-truth, names the new optional `deferred_to_loop` field, and clarifies the `open → accepted` transition for paused-team in-loop acceptance.
- **`org-os/templates/request.md`** documents the new optional `deferred_to_loop` frontmatter field.
- **`org-os/rituals/ceo-brief.md`** step 1 (re-read state) explicitly mentions surfacing any request hitting its `deferred_to_loop` this loop.

**Grandfathering.** Existing request files do not need retroactive `deferred_to_loop` insertion (the field is optional + future-only). The first request to use the field is `teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md` at its 2026-06-13 acceptance bookkeeping; no historical request needs editing.

## Alternatives considered

- **A: Status quo (record in status.md + request file; reader chooses).** Rejected because the records drift over time as status.md is refreshed each loop. The friction is small but recurring; not a one-off.
- **B: Record only in status.md, drop the request file's frontmatter status field.** Rejected because the request file is the canonical artifact of the cross-team ask; dropping its status field breaks ADR-2026-05-09-006's request-flow lifecycle.
- **C: Add a new state `accepted-deferred` to the request enum.** Rejected because the state space stays smaller with `accepted + deferred_to_loop`; the new field is cheaper than a new state.

## Consequences

- **Positive:** Single source-of-truth for request acceptance status. The CEO consolidation metrics in the request-flow table become trustworthy at loop-boundary read time. Brief authoring can verify "any paused-team-accepted request hitting its deferred_to_loop this loop?" as part of its first read step.
- **Negative / costs:** One new optional frontmatter field; rituals update. Existing request files don't need retroactive editing (`deferred_to_loop` is optional + future-only).
- **Follow-ups required:** Update `org-os/rituals/team-intake.md` with the canonicalization rule. Update `org-os/templates/request.md` to mention the optional field. Sister cross-link from `org-os/rituals/ceo-brief.md` step 1 (read state) — surface any request hitting its deferred-loop this loop.

## Alternatives considered (for forcing function — decided this ratification)

- **A1: Forcing function on `deferred_to_loop`** (brief's first read step surfaces unresolved deferrals at the named loop). **Adopted.** Maps the request flow to the loop-boundary discipline; same mechanism as `board/actions/<date>-<slug>.md` `due:` field.
- **A2: No forcing function — soft deadline only.** Rejected because soft deadlines on paused-team work tend to slip indefinitely; the forcing function is the load-bearing piece.

## Links

- Triggering retro: [board/retros/2026-05-11-1631-ceo-retro.md](../retros/2026-05-11-1631-ceo-retro.md) — P3.
- Sister ADR (drafted same loop): [2026-06-06-001-provisional-and-migrate-playbook](2026-06-06-001-provisional-and-migrate-playbook.md) — same retro, different pattern.
- Related: [ADR-2026-05-09-006](2026-05-09-006-org-os-change-routing.md) — request-flow lifecycle this ADR extends.
- Worked example trigger: [teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md](../../teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md) — DE's pending paused-team acceptance.
{% endraw %}
