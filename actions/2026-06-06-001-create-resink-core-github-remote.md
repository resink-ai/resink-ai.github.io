---
layout: default
title: "Action 2026-06-06-001: create resink core github remote"
date: 2026-06-06
status: open
type: action
owner: board
due: 2026-06-13
---

<!-- original-frontmatter:
  type: action
  owner: board
  date: 2026-06-06
  status: open
  due: 2026-06-13
  links: triggering_retro: ../retros/2026-05-30-ceo-retro.md
-->
# Action 2026-06-06-001: Create the `resink-ai/resink-core` GitHub remote

> **Frontmatter `type: action` is provisional.** The `action` type is not yet admitted to the org-os conventions enum; admission is deferred to loop 2026-06-13 alongside ADR-2026-05-30-001 (P2) to batch the enum-extension changes naturally. Same `provisional-and-migrate` pattern AE used at 2026-05-30 for `rfc → role` migration. Until then, this artifact lives at a tenant-local path (`board/actions/`) under tenant convention only.

## Problem

Workspace promotion for resink-core has been deferred for **three consecutive loops** (2026-05-16, 2026-05-23, 2026-05-30). Same blocker every time: `https://github.com/resink-ai/resink-core.git` does not exist on GitHub (`Repository not found` from `git ls-remote`). Same resolution every time: the human founder needs to create the empty repo.

**Root cause:** the brief-channel ask routes through the same human who would create the remote — so the ask is, mechanically, the human asking themselves. There is no escalation step that bumps the urgency. The pattern keeps recurring because the loop machinery has no way to convert a "human external action" into a tracked, dated, forcing-function deliverable.

This artifact is the forcing function.

## Action required

Create an empty GitHub repository at `https://github.com/resink-ai/resink-core` with the following shape:

- **Visibility:** private (resink-ai org's default for product repos).
- **Default branch:** `master` (matches the in-tree branch in `repos/resink-ai/resink-core/`).
- **Initial commit:** empty / not required. The promotion submodule push will populate it.
- **No README, no .gitignore, no LICENSE** at creation time — those are already in the in-tree resink-core working tree and will be pushed at promotion.

Equivalent `gh` CLI command (run as the resink-ai org owner):

```sh
gh repo create resink-ai/resink-core \
    --private \
    --description "Resink.ai nanofab runtime, training pipeline, sim-farm engine, and orchestrator" \
    --confirm
```

After the remote exists, resink-core's team will execute the submodule promotion (the deferred KR2.1 carry); this action's job is only to unblock that step.

## Forcing function

**If this ticket has not flipped to `status: done` by 2026-06-13** (one loop boundary from the filing), the 2026-06-13 brief's "Out of scope this loop" section MUST name the implication explicitly, with the count of downstream deliverables this is blocking. Current downstream blockers known at filing time:

1. **Workspace promotion** (resink-core, third-consecutive-loop carry).
2. **dlopen restoration step 2 (supervisor swap)** scheduled `loop+1` (= 2026-06-13) per ADR-2026-05-16-001 — without the remote, the supervisor swap commit graph is in-tree-only; future bisect / blame across the runtime separation gains friction.
3. **dlopen restoration step 3 (hot-swap correctness test)** scheduled `loop+2` (= 2026-06-20) — same friction multiplied.
4. **The HTML capabilities report** at `board/reports/2026-05-30-resink-core-capabilities.html` cites in-tree paths for evidence; once promoted, paths become submodule-relative and need a one-paragraph update.

The 2026-06-13 brief authors (board) MUST verify this ticket's status as part of step 1 of `org-os/rituals/ceo-brief.md` (re-read state, including any `status: open` `board/actions/`). If `status: open` at brief-authoring time, the brief invokes the forcing function and surfaces the count of blockers.

## Resolution

When the remote exists:

1. Flip frontmatter `status: open → status: done`.
2. Add a "## Resolution" subsection below with date, who created the remote, and the URL.
3. Reference this ticket in the next loop's exec summary under § "Decisions ratified" or § "Asks closed."
4. resink-core picks up the submodule promotion in the next loop's build phase.

## Read order

This file is read at every CEO brief authoring step (per ADR-2026-05-09-005's carryover-load discipline + the brief's "first read-step" extension) until `status: open → status: done`. Once closed, it remains in the directory as a historical record; closed actions are not re-read.

## Links

- Triggering retro: [board/retros/2026-05-30-ceo-retro.md](../retros/2026-05-30-ceo-retro.md) — P1.
- Worked-example blocker: workspace promotion in [teams/application/resink-core/status.md](../../teams/application/resink-core/status.md) — third-consecutive-loop carry.
- Sister provisional artifact: this is the first `board/actions/` ticket. Future tickets share this provisional-and-migrate pattern until the org-os enum extension at 2026-06-13.
- ADR governing the convention's eventual admission: TBD-2026-06-13 (sibling to ADR-2026-05-30-001 P2).
