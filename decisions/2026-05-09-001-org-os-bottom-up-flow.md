---
layout: default
title: "ADR 2026-05-09-001: org os bottom up flow"
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
  decision: Adopt the org-os bottom-up flow design (team-initiated work + cross-team requests) as specified in docs/superpowers/specs/2026-05-09-org-os-bottom-up-flow-design.md; AE owns implementation via the 17-task plan in docs/superpowers/plans/2026-05-09-org-os-bottom-up-flow.md.
-->
{% raw %}

# ADR 2026-05-09-001: Org-OS bottom-up flow

## Context

The org-OS as initially designed in `docs/superpowers/specs/2026-05-08-ai-native-org-os-design.md` is strictly top-down: CEO brief → team OKR decomposition → build → exec summaries → CEO consolidation → CEO retro. Team OKRs must trace back to the brief.

Two classes of real work have no home in this model:

1. **Team-initiated stewardship** — service stability, observability, developer ergonomics, on-call ergonomics, runbook hygiene. These don't directly close gaps to the company vision but neglecting them grows operational risk and slows every other team.
2. **Cross-team feedback and requests** — one team needs something from another. Today's only surface is a one-line "ask" in the requesting team's OKR; there is no canonical artifact, no triage protocol, no closure, no roll-up. Asks die in chat.

A design spec landed on the `org-os-bottom-up-flow-design` branch in loop 2026-05-10-2227-001 (`docs/superpowers/specs/2026-05-09-org-os-bottom-up-flow-design.md`, cherry-picked onto `loop-2026-05-09` in loop 2026-05-10-2227-002 so the implementation tree could see it) along with a 17-task implementation plan (`docs/superpowers/plans/2026-05-09-org-os-bottom-up-flow.md`). The 2026-05-09 retro and exec summary record the board's review of the design as "approved as written, no material changes requested." This ADR formalizes that ratification so that AE can execute against `org-os/` per the ADR-001 frontmatter contract and the `merge-evolution-proposal.md` flow.

## Decision

Adopt the bottom-up flow design as specified. Specifically:

- Teams may originate two new artifact types: **team proposals** (team-initiated stewardship work) and **cross-team requests** (a structured ask filed against a receiver team).
- Both artifacts have explicit lifecycles (`open → triaged → accepted | declined | escalated → fulfilled | dropped`).
- Cross-team requests escalate to CEO ratification when any of four triggers fire: effort > ~1 week / `size: L`; receiver wants to decline and requester pushes back; interface/charter change implied; fan-out > 1 receiver.
- The CEO brief ritual gains a section where the CEO ratifies the slate of bottom-up work for the loop; team OKRs no longer must trace every objective to a CEO brief objective if they carry a ratified team-proposal source.

Authority model: **teams propose; CEO ratifies in the brief**. Small inter-team asks stay between two EMs.

The 17-task implementation plan is the execution contract. AE owns it. Bundle A (foundation: this ADR + conventions + templates) is the gate for Bundle B (rituals) and Bundle C (roles + docs + verification), per AE's loop-2026-05-10 OKR.

## Alternatives considered

- **A: Status quo (top-down only).** Rejected because real stewardship and request flow run under the floorboards and surface only as friction at retro time. The exposure compounds as more teams join.
- **B: Two parallel governance tracks (top-down + bottom-up).** Rejected because it splits CEO attention and produces two competing slates per loop. The chosen design keeps one slate, ratified at brief time, with two intake paths.
- **C: Cross-team requests as a free-form note.** Rejected because asks need a lifecycle (acceptance, escalation, closure); free-form notes have neither owner nor close-out semantics.

## Consequences

- **Positive:**
  - SRE-style stewardship work has a first-class home.
  - Cross-team requests get an owner, a lifecycle, and a roll-up.
  - The CEO gains visibility into capacity *balance* (top-down vs. bottom-up vs. request fulfillment) without having to approve every small ask.
  - The org-OS becomes extractable as a tenant-agnostic skill that handles real organizational reality, not just hierarchical decomposition.
- **Negative / costs:**
  - Adds two artifact types and modifies six rituals. Implementation is substantial — 17 tasks, three bundles.
  - Until Bundle B (rituals updated) lands, teams cannot actually use the new flow even though Bundle A defines its vocabulary.
  - Risk of `org-os/` churn during implementation — each bundle introduces conventions changes that ripple to templates and rituals.
- **Follow-ups required:**
  - AE executes Bundle A in loop 2026-05-10-2227-002 (this loop) — see `teams/platform/agent-engineering/okrs/2026-05-10-2227-002-team-okr.md`.
  - Tenant-isolation dry-run (`org-os/playbooks/extract-org-os.md`) after each bundle lands.
  - Bundle B + C land in subsequent loops; no commitment to a specific cadence here.

## Links

- **Triggering retro:** the design originated from a CEO-level review, not a retro; routed to ADR via `org-os/playbooks/merge-evolution-proposal.md`. This is precisely the "out-of-retro origin" gap that the deferred ADR-006 (loop 2026-05-11-1113) is being introduced to formalize. Until ADR-006 lands, the routing for this ADR was ad-hoc; that is acceptable given the design was reviewed by the board and approved as written.
- **Design spec:** [docs/superpowers/specs/2026-05-09-org-os-bottom-up-flow-design.md](../../docs/superpowers/specs/2026-05-09-org-os-bottom-up-flow-design.md)
- **Implementation plan:** [docs/superpowers/plans/2026-05-09-org-os-bottom-up-flow.md](../../docs/superpowers/plans/2026-05-09-org-os-bottom-up-flow.md)
- **Implementation OKR (loop 2026-05-10-2227-002):** [teams/platform/agent-engineering/okrs/2026-05-10-2227-002-team-okr.md](../../teams/platform/agent-engineering/okrs/2026-05-10-2227-002-team-okr.md)
- **Related ADRs:** [2026-05-08-001-frontmatter-validation](2026-05-08-001-frontmatter-validation.md), [2026-05-08-002-tenant-tree-restructure](2026-05-08-002-tenant-tree-restructure.md), [2026-05-09-006-org-os-change-routing](2026-05-09-006-org-os-change-routing.md) (deferred to 2026-05-23 — will retroactively formalize the routing used here).
- **Foundational org-OS spec:** [docs/superpowers/specs/2026-05-08-ai-native-org-os-design.md](../../docs/superpowers/specs/2026-05-08-ai-native-org-os-design.md)
{% endraw %}
