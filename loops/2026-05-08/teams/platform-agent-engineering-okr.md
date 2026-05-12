---
layout: default
title: platform-agent-engineering OKR — 2026-05-08
nav_exclude: true
render_with_liquid: false
date: 2026-05-08
status: active
type: okr
loop: 2026-05-08
owner: teams/platform/agent-engineering
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/agent-engineering
  date: 2026-05-08
  status: active
  loop: 2026-05-08
  links: parent: board/okrs/2026-05-08-ceo-brief.md
-->
# Agent Engineering Team OKR — 2026-05-08

## Context

AE is bootstrapping. The `org-os/` skeleton has landed with charters, rituals, templates, playbooks, and ADR infrastructure in place. AE's job this loop is to make sure that skeleton stays runnable — internal links resolve, and the spawn-agent playbook actually executes — as new agents and rituals land in future loops. The marquee product (plugin marketplace) is not started this loop; instead, AE commits to drafting the RFC so development is unblocked in the next loop.

## Objectives

### O1: Agent runtime contract is honored

Why it matters: The value of `org-os/templates/agent-spec.md` and the ritual library depends entirely on internal links resolving and the spawn-agent playbook actually working end-to-end. If a future team tries to spawn a new agent and hits a broken link or an incomplete step, the org-OS fails at the moment it is most needed. Validating the contract now, while the skeleton is small and fresh, costs almost nothing — validating it after six loops of accumulated artifacts costs much more.

**Key results**
- KR1.1: Every internal link in `org-os/` resolves.
- KR1.2: A dry-run "spawn an agent" walkthrough succeeds against `org-os/playbooks/spawn-agent.md`.

**Tasks**
- [ ] Sanity-check every link in `org-os/` resolves — owner: teams/platform/agent-engineering
- [ ] Walk through `spawn-agent.md` end-to-end with a hypothetical "dbt-author" agent — owner: teams/platform/agent-engineering

### O2: Plan plugin marketplace v0 for next loop

Why it matters: The plugin marketplace is the AE charter's marquee product and the mechanism through which other teams will extend the platform without coupling to AE's release cycle. Before any code lands, the team needs an RFC that pins the marketplace shape — what a plugin is, how it is registered, how it is versioned, and what the security boundary looks like. Without the RFC, the next loop will spend its first week in design churn rather than implementation. Drafting the RFC this loop turns the next loop's kickoff into an execution decision, not a design decision.

**Key results**
- KR2.1: Written RFC sketch of the marketplace shape exists in `teams/platform/agent-engineering/` as a draft RFC.
- KR2.2: At least 2 candidate plugins identified to seed the marketplace v0 scope.

**Tasks**
- [ ] Draft `rfc-plugin-marketplace-v0.md` (separate file, deferred to next loop if blocked by bootstrap overhead) — owner: teams/platform/agent-engineering
- [ ] Identify 2 candidate plugins for the marketplace v0 — owner: teams/platform/agent-engineering

## Risks

- RFC may not land this loop given bootstrap overhead; if so, it carries to next loop as the first concrete deliverable.
- The agent-spec contract may need revision after the first real spawn walkthrough, which would constitute an org-os change and require a retro proposal.

## Out of scope this loop

- Real product code for the marketplace.
- Building or deploying the marketplace itself.
- Spawning real agents (only a hypothetical dry-run is required).
- Any work for application teams (`resink-core`, `sim-farm`), whose OKRs are deferred to next loop.
