---
layout: default
title: "ADR 2026-05-08-001: frontmatter validation"
date: 2026-05-08
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-08
  status: active
  decision: Org-OS docs require frontmatter validation before merge
-->
# ADR 001: Org-OS docs require frontmatter validation before merge

## Context

Phase 1 produces dozens of artifacts with YAML frontmatter across `board/`, `teams/platform/<team>/`, and `teams/application/<team>/`. The frontmatter is the contract for mechanical rollup — it drives consolidation, retros, onboarding, and cross-team coordination. Without an automated check, there is no enforcement barrier against drift: a file can be committed with missing or incorrect keys, silently breaking the rollup machinery.

No automated frontmatter validation exists in Phase 1, which is markdown-only by spec. This decision establishes the validation requirement and its Phase 1 implementation. Triggered by retro proposal P1 in `board/retros/2026-05-08-ceo-retro.md`.

## Decision

All artifacts under `board/`, `teams/platform/<team>/`, and `teams/application/<team>/` must pass a frontmatter check before commit. The required keys are `type`, `owner`, `date`, and `status`, per `org-os/conventions.md`. In Phase 1 the check is manual: a dedicated "verify frontmatter" step is added to `org-os/playbooks/onboard-team.md` and must be completed before each commit. In Phase 2 the check becomes a CI gate owned by teams/platform/devops, replacing the manual step for new artifacts going forward.

## Alternatives considered

- **A:** No check; rely on review — rejected: drift will accumulate as the artifact count grows and reviewers cannot reliably catch all frontmatter issues across an expanding set of files.
- **B:** CI-only check from day one — rejected: out of Phase 1 scope (Phase 1 is markdown-only by spec; CI infrastructure is a Phase 2 deliverable).

## Consequences

- **Positive:** rollups stay mechanical; retros remain trustworthy; the onboarding playbook now makes the requirement explicit for every new team.
- **Negative / costs:** small overhead per loop until Phase 2 automation lands; each new artifact created during Phase 1 requires a manual check before commit.
- **Follow-ups required:** Phase 2 spec includes the CI gate; teams/platform/devops carries the linter stub in their next OKR.

## Links

- Triggering retro: [`board/retros/2026-05-08-ceo-retro.md`](../retros/2026-05-08-ceo-retro.md)
- Affected playbook: [`org-os/playbooks/onboard-team.md`](../../org-os/playbooks/onboard-team.md)
