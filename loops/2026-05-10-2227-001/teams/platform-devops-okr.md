---
layout: default
title: platform-devops OKR — 2026-05-10-2227-001
date: 2026-05-10
status: active
type: okr
loop: 2026-05-10-2227-001
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/devops
  date: 2026-05-10
  status: active
  loop: 2026-05-10-2227-001
  links: parent: board/okrs/2026-05-10-2227-001-ceo-brief.md
-->
{% raw %}

# DevOps OKR — 2026-05-09

## Context

Two carryovers from 2026-05-08: (a) the deployment-target ADR is partial — options are enumerated but no recommendation is committed; (b) the markdown-frontmatter linter stub was deferred pending ADR-001, which has now landed (`board/decisions/2026-05-08-001-frontmatter-validation.md`). Both ship this loop. The deployment-target ADR is on the critical path for resink-core; the linter is the product portion of P1 from the 2026-05-08 retro.

## Objectives

### O1: Commit a deployment target for the first demo and land the ADR

Why it matters: Resink-core cannot name the deployment pattern in its first OKR until DevOps commits a target. CEO brief KR1.2 depends on this.

**Key results**
- KR1.1: An ADR exists in `board/decisions/` with `status: active`, choosing exactly one of {local Docker, minikube, cloud} for the first demo and stating a migration path to whatever target the eventual production demo will use.

**Tasks**
- [x] Refine the 2026-05-08 option enumeration with first-demo constraints — owner: teams/platform/devops
  - Local-Docker rejected (no orchestration); cloud rejected (premature ops complexity, no team set up); minikube selected for portability.
- [x] Write the ADR with a one-paragraph recommendation and a migration-path subsection — owner: teams/platform/devops
  - Written at `board/decisions/2026-05-09-003-deployment-target.md`. Recommends minikube; manifests portable to cloud k8s.
- [x] Submit ADR to CEO for approval and land with `status: active` — owner: teams/platform/devops
  - Landed with `status: active`. Two follow-ups noted in the ADR: pick a manifest tool (Helm/Kustomize/raw) and document resource budgets per pipeline.

### O2: Land the frontmatter-lint CI script (P1 product portion)

Why it matters: ADR-001 (`board/decisions/2026-05-08-001-frontmatter-validation.md`) requires frontmatter validation before merge. Without a CI script the contract is enforced manually, which guarantees drift.

**Key results**
- KR2.1: A frontmatter-lint script exists, validates YAML frontmatter per `org-os/conventions.md` (required keys per type, status enum), and is wired into CI on push and PR.
- KR2.2: The script's location and invocation are documented under `teams/platform/devops/` so other teams know how to run it locally.

**Tasks**
- [ ] Implement the frontmatter-lint script (Python or shell; choice up to DevOps) that walks the repo and validates every artifact's frontmatter against the per-type required-keys table in `org-os/conventions.md` — owner: teams/platform/devops
  - > deferred: scoped out of this loop's critical-path build per CEO scope decision. Manual ADR-001 enforcement continues until next loop. Risk: drift accumulates over the next loop; mitigation: retro reviews any frontmatter drift introduced in the meantime.
- [ ] Wire the script into the repo's CI for push and PR events — owner: teams/platform/devops
  - > deferred: depends on the script above.
- [ ] Document the script's path, invocation, and expected output under `teams/platform/devops/` — owner: teams/platform/devops
  - > deferred: depends on the script above.

## Cross-team asks

- None this loop. Both objectives are owned end-to-end by DevOps.

## Risks

- The frontmatter-lint script may surface existing violations in the tenant tree on first run. Mitigation: run locally before wiring into CI; if violations exist, fix them as part of KR2.1 rather than letting the first CI run fail.
- The deployment-target ADR's migration path is a future-decision sketch, not a binding commitment. Mitigation: the ADR should make this explicit so reviewers don't over-read it.

## Out of scope this loop

- Building the actual deployment automation; this loop only commits the target.
- Lint coverage beyond YAML frontmatter (markdown-style, link-validity, etc.).
- Migration to the production target — only the path is documented.
{% endraw %}
