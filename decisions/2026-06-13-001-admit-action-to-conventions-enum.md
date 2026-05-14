---
layout: default
title: "ADR 2026-06-13-001: admit action to conventions enum"
date: 2026-06-13
status: active
type: adr
owner: board
parent: Decisions (ADRs)
nav_order: 1
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-06-13
  status: active
  decision: "Admit type: action to the org-os conventions enum so board-action tickets become canonical artifacts; migrate the two existing 2026-06-06-001 and 2026-06-06-002 board-action tickets from provisional to canonical in the same loop"
-->
{% raw %}

# ADR 2026-06-13-001: Admit `action` to the conventions type enum

## Context

Two `board/actions/<date>-<slug>.md` tickets have been filed under provisional `type: action` frontmatter:

- [`board/actions/2026-06-06-001-create-resink-core-github-remote.md`](../actions/2026-06-06-001-create-resink-core-github-remote.md) — workspace-promotion forcing function.
- [`board/actions/2026-06-06-002-install-minikube-on-dev-machine.md`](../actions/2026-06-06-002-install-minikube-on-dev-machine.md) — minikube install forcing function.

Both ticket bodies carry an explicit "frontmatter `type: action` is provisional" blockquote pointing at this loop's enum-extension migration. Both apply the [provisional-and-migrate playbook](../../org-os/playbooks/provisional-and-migrate.md) (ratified this loop via [ADR-2026-06-06-001](2026-06-06-001-provisional-and-migrate-playbook.md)) — the canonical-change loop is `loop+1` from the tickets' filing.

The pattern's precedent is [ADR-2026-05-23-001](2026-05-23-001-conventions-enum-extension.md) (ratified loop 2026-05-11-1302, after a 2026-05-23 draft), which admitted FIVE artifact types in a single batch (`runbook`, `convention`, `playbook`, `report`, `role`) — including `report` and `role` admitted same-loop as the artifacts that needed them. Same-loop draft-and-ratify of an enum extension is therefore an established board move, not a novel one.

The provisional-and-migrate cycle for the two `board/actions/` tickets needs closure this loop. Without canonical admission, the tickets accrue edit risk loop-over-loop (the in-body provisional notes drift; future readers must rediscover that `action` was admitted somewhere). The enum extension closes the cycle cleanly.

## Decision

Admit `action` to `org-os/conventions.md`'s `type` enum. Add a row to the "Additional fields per type" table for `action`:

| type     | additional required keys                            |
|----------|-----------------------------------------------------|
| `action` | `due: YYYY-MM-DD`, `status: open \| done \| superseded` |

The `due:` field is the forcing-function date — the brief at that loop surfaces the ticket if it has not closed (per [ADR-2026-05-09-005](2026-05-09-005-carryover-load-in-brief.md)'s carryover-load discipline + the ceo-brief ritual's first read step).

The `status:` enum for `action` is narrower than the publication-state enum: only `open | done | superseded`. Action tickets do not draft, do not publish, do not archive — they are filed as `open`, advanced to `done` on resolution, or `superseded` on supersession. Add this to the conventions.md status-values section under a new "`action` — lifecycle status, not publication status" subsection (mirror the existing `request` status subsection's shape).

**Migrate the two existing tickets in the same loop:**

1. `board/actions/2026-06-06-001-create-resink-core-github-remote.md` — remove the in-body "frontmatter `type: action` is provisional" blockquote; preserve the rest of the body. Frontmatter `type: action` is now canonical.
2. `board/actions/2026-06-06-002-install-minikube-on-dev-machine.md` — same migration: remove the provisional blockquote; preserve the rest of the body.

The migration follows the [provisional-and-migrate playbook](../../org-os/playbooks/provisional-and-migrate.md)'s step 4 ("Migrate the frontmatter") — the body reads as if the canonical shape was always in force.

## Alternatives considered

- **A: Status quo (provisional indefinitely).** Rejected because the in-body provisional notes accrue edit risk every loop they linger; the canonical-change boundary becomes ambiguous. The provisional-and-migrate playbook explicitly anti-patterns >1-loop-out provisional artifacts; both tickets are at exactly the 1-loop boundary this loop, so the migration is on schedule.
- **B: One-by-one admission per artifact type as it arises.** Rejected because batching is cheaper (the precedent at ADR-2026-05-23-001 admitted five types in one ratification with finite cost). For `action` specifically there is only one type to admit, so the batch argument is weaker — but the same-loop draft-and-ratify cost is also tiny, and there are no other pending artifact types this loop needing admission.

## Consequences

- **Positive:** The board-action ticket pattern (`board/actions/<date>-<slug>.md` with `due:`-based forcing function) is now first-class. Future board-action tickets file canonically from creation (no provisional-and-migrate needed). The two existing tickets reach canonical state in the same loop the enum admits the type.
- **Negative / costs:** One more value in the type enum (12 → 13 owned-artifact types). One more row in the additional-fields table. The `action`-specific status enum (`open | done | superseded`) is the second narrowed-status enum after `request`'s lifecycle status — a small bookkeeping load but well-contained by the conventions.md "Status values" section.
- **Follow-ups required:**
  - `org-os/conventions.md`: add `action` to the type enum; add the additional-fields row; add the `action` lifecycle-status subsection.
  - Migrate the two existing `board/actions/2026-06-06-001*.md` and `board/actions/2026-06-06-002*.md` files (in-body provisional notes removed; bodies otherwise preserved).
  - Future frontmatter-lint script (DevOps's deferred work) gains an `action`-type-specific required-key check (`due:` + narrowed `status:` enum).

## Links

- Sister precedent (same shape, larger batch): [ADR-2026-05-23-001](2026-05-23-001-conventions-enum-extension.md) — five-type enum extension ratified at 2026-05-30 with `report` and `role` admitted same-loop as the artifacts that needed them.
- Ratifying playbook applied to the migration: [ADR-2026-06-06-001](2026-06-06-001-provisional-and-migrate-playbook.md) (provisional-and-migrate playbook, ratified this loop) + [`org-os/playbooks/provisional-and-migrate.md`](../../org-os/playbooks/provisional-and-migrate.md).
- Triggering work: the two existing `board/actions/` tickets filed at 2026-06-06 with provisional `type: action` (see Context).
- Sister carryover discipline: [ADR-2026-05-09-005](2026-05-09-005-carryover-load-in-brief.md) — the brief's first read step surfaces `due:`-hit `action` tickets the same way it now surfaces `deferred_to_loop`-hit requests (per [ADR-2026-06-06-002](2026-06-06-002-paused-team-request-acceptance.md)).
{% endraw %}
