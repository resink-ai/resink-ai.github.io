---
layout: default
title: "ADR 2026-05-09-004: onboard application team"
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
  decision: Adopt onboard-application-team.md as the canonical playbook for an application team's first product-shaped OKR.
-->
# ADR 2026-05-09-004: Onboarding playbook for application teams' first product OKR

## Context

P2 from the 2026-05-08 CEO retro proposed adding `org-os/playbooks/onboard-application-team.md` to give application teams a worked playbook for their first product-shaped OKR (distinct from `onboard-team.md`, which covers charter-only onboarding). This loop is the first time application teams (resink-core, sim-farm) are producing product-shaped OKRs, so the playbook is needed now both to land the proposal and to retroactively validate against this loop's experience.

The change is scoped to a single new file inside `org-os/playbooks/`. Per `org-os/playbooks/merge-evolution-proposal.md`, an org-os change requires an ADR with `status: active` and CEO approval before the edit is applied.

## Decision

Adopt the playbook as written in `org-os/playbooks/onboard-application-team.md`. The playbook codifies five steps (dependency confirmation, demo-target definition, OKR drafting, explicit cross-team asks, CEO submission via team-planning) and three anti-patterns (deferred demo target, implicit dependencies, product-feature-shaped KRs). It complements rather than replaces `onboard-team.md`.

## Alternatives considered

- **A: Reuse `onboard-team.md` for both charter and first-product-OKR onboarding.** Rejected — charter onboarding is about establishing the team's existence; first-product-OKR onboarding is about scoping a concrete deliverable and surfacing dependencies. Conflating the two produces a long playbook that's wrong for both cases.
- **B: Inline the playbook content into the OKR template.** Rejected — the OKR template is used by every team every loop; bloating it for the application-team-first-OKR case would clutter every other use.

## Consequences

- Positive: Application teams have an explicit step-by-step for their first product OKR; cross-team asks become explicit by playbook contract rather than by EM discipline.
- Negative / costs: One more playbook to maintain; future evolution of how teams onboard requires editing two playbooks.
- Follow-ups required:
  - Validate the playbook retroactively against this loop's resink-core and sim-farm OKRs at retro time (CEO retro 2026-05-09).
  - Update `org-os/README.md` if the playbooks table grows further (no change needed for this single addition since the table is small).

## Links

- Triggering retro: [2026-05-08-ceo-retro](../retros/2026-05-09-1715-ceo-retro.md) (P2)
- Related ADRs: [2026-05-08-001-frontmatter-validation](2026-05-08-001-frontmatter-validation.md), [2026-05-08-002-tenant-tree-restructure](2026-05-08-002-tenant-tree-restructure.md)
- Playbook: [onboard-application-team.md](../../org-os/playbooks/onboard-application-team.md)
