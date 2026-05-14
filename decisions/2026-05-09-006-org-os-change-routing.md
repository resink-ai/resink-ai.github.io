---
layout: default
title: "ADR 2026-05-09-006: org os change routing"
date: 2026-05-09
status: active
type: adr
owner: board
parent: Decisions (ADRs)
nav_order: 18
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-09
  status: active
  decision: Org-os change proposals that surface mid-loop (not at retro time) route through a dedicated team-proposals path and are picked up as candidate evolution proposals in the next CEO brief.
-->
{% raw %}

# ADR 2026-05-09-006: Out-of-retro routing path for org-os change proposals

## Context

P2 from the 2026-05-09 CEO retro. The bottom-up flow design (`docs/superpowers/specs/2026-05-09-org-os-bottom-up-flow-design.md`, ratified loop 2026-05-10-2227-001) originated from a CEO-level review rather than a retro. `org-os/playbooks/merge-evolution-proposal.md` assumes the entry point is a retro-classified proposal, which left no documented routing for the bottom-up flow's draft ADR — it was hand-routed via CEO consolidation as a one-off that loop. Without a documented out-of-retro path, every mid-loop proposal becomes a one-off; the merge-evolution-proposal playbook quietly accumulates exceptions; teams have no canonical place to file an org-os change idea that surfaces during build (e.g., DE's `contract` enum gap, AE's bundle-slip pattern). The `team-proposals` artifact type already exists in `org-os/conventions.md` for per-loop bottom-up intake; it is the natural home for out-of-retro org-os change proposals.

## Decision

Org-os change proposals that surface outside of retro time follow a dedicated routing path:

1. **Any team may file a `team-proposals` document** at `teams/<layer>/<team>/proposals/<date>-team-proposals.md` flagging the proposed org-os change. The document carries `type: team-proposals` with the required `loop: <date>` frontmatter; the change is described in-body with the same shape as a retro-class proposal (one-line title, brief context, suggested class — which for this routing is always `org-os`).
2. **The next CEO brief's authoring step picks the team-proposals document up as a candidate evolution proposal**, alongside any retro-classified proposals. If the CEO accepts it, the brief instantiates a new objective with `source: team-initiated`, and the ADR + CEO-approval gate from `merge-evolution-proposal.md` applies as usual.
3. **A new playbook, `org-os/playbooks/out-of-retro-org-os-change.md`**, documents this path end-to-end so it is discoverable by teams and not buried inside this ADR.
4. **`org-os/rituals/team-planning.md`** notes that team-planning may pick up `source: team-initiated` objectives that originated as out-of-retro proposals (their `links.source` field points into the originating team's `proposals/` directory).

The retro path remains the default; this ADR adds an additional path for proposals that should not wait until end-of-loop.

## Alternatives considered

- **A: Extend `merge-evolution-proposal.md` to accept any review as the entry point.** Rejected because the playbook documents merge-time mechanics; the question of how proposals enter the queue belongs in a separate playbook.
- **B: Wait for retro every time.** Rejected because the bottom-up flow design itself was an existence proof that high-value org-os proposals can surface mid-loop; forcing them to wait until retro loses momentum and often loses the proposer's context.
- **C: Allow ad-hoc CEO-consolidation routing as the canonical path.** Rejected because routing through CEO consolidation has no team-side artifact, leaving teams with no way to discover or audit what proposals are in flight.

## Consequences

- **Positive:** Teams have a documented path to surface org-os concerns mid-loop. The `team-proposals` artifact type (already in conventions) gets a clear use case. The CEO brief's authoring step gains a documented input source beyond retros. Cross-team visibility improves — proposals are filed under each team's directory, not lost in a one-off CEO review thread.
- **Negative / costs:** One more place the CEO brief author must check at brief time (every team's `proposals/` directory). Risk that teams over-use the path for what should be retro material; mitigated by the next-retro check ("is this team filing proposals more often than retro? — look at why").
- **Follow-ups required:** Create `org-os/playbooks/out-of-retro-org-os-change.md`. Update `team-planning.md` to reference the path. Loop 2026-05-11-1113 brief's authoring step picks up this convention — any team that filed a `team-proposals` document during build of 2026-05-16 has its proposal considered.

## Links

- Triggering retro: [board/retros/2026-05-10-2227-001-ceo-retro.md](../retros/2026-05-10-2227-001-ceo-retro.md) (P2)
- Related design: [2026-05-09-org-os-bottom-up-flow-design](../../docs/superpowers/specs/2026-05-09-org-os-bottom-up-flow-design.md)
- New playbook: [org-os/playbooks/out-of-retro-org-os-change.md](../../org-os/playbooks/out-of-retro-org-os-change.md)
- Ritual cross-link: [org-os/rituals/team-planning.md](../../org-os/rituals/team-planning.md)
- Related ADRs: [2026-05-09-001-org-os-bottom-up-flow](2026-05-09-001-org-os-bottom-up-flow.md), [2026-05-09-005-carryover-load-in-brief](2026-05-09-005-carryover-load-in-brief.md)
{% endraw %}
