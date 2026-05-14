---
layout: default
title: Retro — 2026-05-10-2227-001
date: 2026-05-10
status: active
type: retro
loop: 2026-05-10-2227-001
owner: board
grand_parent: Loops
parent: Loop 2026-05-10-2227-001
nav_order: 3
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-10
  status: active
  loop: 2026-05-10-2227-001
  links: parent: board/exec-summaries/2026-05-10-2227-001.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-09

## What worked

- **The "application teams surface constraints before platform teams write ADRs" pattern.** Codified by ADR-004's new playbook (`org-os/playbooks/onboard-application-team.md`). Resink-core wrote its streaming constraints into the OKR plan addendum *before* DE wrote ADR-002; DE's ADR explicitly cites those constraints. Both critical-path ADRs (engine + deployment) landed in one loop instead of taking another iteration each. Reproducibility: when an application team's first OKR fires the playbook, its surfaced cross-team asks must be inputs to the platform teams' ADRs in the same loop.
- **Single-document multi-consumer hand-off.** Sim-farm wrote one validation contract that served both resink-core's plan KR1.1.e and SRE's runbook scope (deferred but well-grounded) from a single document with named hand-off sections. Reproducibility: when one team's artifact is needed by two other teams, prefer one document with named hand-off sections over two adjacent per-consumer documents.
- **Critical-path-only build scope decided at brief time.** With 6 teams in the loop and ~15 substantial build artifacts on the table, the explicit "critical-path only" decision made by the CEO at brief time let the loop land in a single session without dropping the headline goal (unblock resink-core). Reproducibility: when artifact volume at brief time exceeds the loop's capacity, name the critical-path subset explicitly and defer the rest with named owners.

## What didn't

- **Four teams enter next loop with explicit carryovers** (DE: shared primitive contract; DevOps: frontmatter-lint script; SRE: runbook draft; AE: bottom-up flow sizing). The deferrals were deliberate (CEO scope decision), not blocked. **Root cause:** the brief committed to all four objectives at full scope without visibility into the cumulative deferred load entering the loop. The "deferred this loop = additional load on next loop" relationship was not surfaced at brief time. Pattern risk: every loop carrying deferred work forward.
- **Manual ADR-001 enforcement carries another loop of drift risk.** The frontmatter-lint script defer is the highest-risk carryover because it's the only one with an explicit drift-accumulation cost. **Root cause:** the brief did not weight defer-cost against new-work-cost; both DevOps objectives (deployment ADR, lint script) were treated as equal-priority within the team rather than recognizing that one had ongoing drift cost while the other was a one-shot decision.
- **Out-of-band org-os change had no natural routing.** The bottom-up flow design originated from a CEO-level review (not a retro), which means it bypassed the existing `merge-evolution-proposal.md` flow whose assumed entry point is a retro. We routed it via CEO consolidation as a special case this loop. **Root cause:** the playbook's classification flow assumes a retro-shaped origin and doesn't say what to do with org-os changes proposed outside that origin.

## Evolution proposals

### P1: Add a "Carryover load by team" section to the CEO brief template (class: **org-os**)

- **Problem it solves:** "What didn't" #1 — the brief commits to new work without visibility into deferred-load entering the loop.
- **Proposed change:** Update the CEO brief authoring step (and `org-os/templates/okr.md` brief notes, or a dedicated section in `org-os/rituals/ceo-brief.md` step 2) to require a "Carryover load by team" tally. The CEO reads every team's most recent exec summary and writes one line per team listing what's carrying over. Brief commitments must explicitly account for it.
- **Review path:** ADR mandatory for the org-os portion. Draft placeholder created at `board/decisions/2026-05-09-005-carryover-load-in-brief.md`.
- **Owner:** board.
- **Timing:** **deferred to loop 2026-05-11-1113** (after AE's bottom-up flow implementation completes in loop 2026-05-11-0958; this is a small ritual change but stacking it on top of bottom-up flow execution would overload AE).

### P2: Add an out-of-retro routing path for org-os changes (class: **org-os**)

- **Problem it solves:** "What didn't" #3 — `merge-evolution-proposal.md` assumes a retro-shaped origin; out-of-band proposals (like the bottom-up flow design that came from a CEO review) have no documented entry path.
- **Proposed change:** Update `org-os/playbooks/merge-evolution-proposal.md` to explicitly handle origins beyond retro: any review (CEO-level, decision-review ritual, or external) can produce an org-os-class proposal; the proposal still goes through the same ADR + CEO-approval gate. The `decision-review` ritual is the natural triage for out-of-band proposals; cross-link both directions.
- **Review path:** ADR mandatory for the org-os portion. Draft placeholder created at `board/decisions/2026-05-09-006-org-os-change-routing.md`.
- **Owner:** board.
- **Timing:** **deferred to loop 2026-05-11-1113** (paired with P1, both small playbook/template edits).

### P3: Codify the "named hand-off sections in one document" pattern as a team-practice convention (class: **tenant**)

- **Problem it solves:** "What worked" #2 — the single-document multi-consumer hand-off pattern emerged organically. Naming it makes it reproducible and prevents teams from defaulting to adjacent per-consumer documents.
- **Proposed change:** When a team produces a hand-off document needed by two or more consumers (other teams), the document carries named hand-off sections in one document rather than being split into per-consumer documents. Each team's charter (or `teams/<layer>/<team>/conventions.md` if introduced) notes this preference.
- **Review path:** Tenant change; no ADR needed (charter-shaped wording within team boundaries). Each team's EM approves locally.
- **Owner:** each platform and application team's EM (decentralized adoption).
- **Timing:** **picked up loop 2026-05-11-0958** (lightweight; can be added to each team's charter alongside other next-loop work).

### P4: Resink-core product repo creation (class: **product**)

- **Problem it solves:** Resink-core's first build slice for next loop has nowhere to land yet. The plan references "resink-core's product repo (path TBD)"; without the repo, the build slice cannot ship.
- **Proposed change:** Resink-core picks a location under `repos/` (most likely `repos/resink-ai/resink-core/`) and creates the repo before the build phase of loop 2026-05-11-0958. Mounted as a submodule into the parent repo, mirroring the pattern set by `resink-marketplace` and `home-cluster`.
- **Review path:** Product change; no ADR needed; rolled into resink-core's next-loop OKR.
- **Owner:** teams/application/resink-core.
- **Timing:** **picked up loop 2026-05-11-0958** (must land in the planning step so the build phase has somewhere to ship).

## Decisions to record

- ADR-005 (P1): "Carryover load tally added to CEO brief authoring step." Draft placeholder at `board/decisions/2026-05-09-005-carryover-load-in-brief.md`. Status: `draft`. Owner: board. Picked up loop 2026-05-11-1113.
- ADR-006 (P2): "Out-of-retro routing path for org-os change proposals." Draft placeholder at `board/decisions/2026-05-09-006-org-os-change-routing.md`. Status: `draft`. Owner: board. Picked up loop 2026-05-11-1113.
- (P3 needs no ADR — tenant-class within-team charter wording.)
- (P4 needs no ADR — product-class.)

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: passed. The new `org-os/playbooks/onboard-application-team.md` (added this loop via ADR-004) uses only generic placeholders (`<team>`, `<layer>`, `<receiver>`). No tenant or product names appear inside `org-os/`.
{% endraw %}
