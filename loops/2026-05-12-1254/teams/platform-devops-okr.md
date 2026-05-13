---
layout: default
title: platform-devops OKR — 2026-05-12-1254
date: 2026-05-12
status: active
type: okr
loop: 2026-05-12-1254
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/devops
  date: 2026-05-12
  status: active
  loop: 2026-05-12-1254
  links: parent: board/okrs/2026-05-12-1254-ceo-brief.md
-->
{% raw %}

# DevOps OKR — 2026-05-12 (loop 2026-05-12-1254)

## Context

Light supporting role. The chart at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` lives inside the directory being promoted to a submodule. After promotion, the working-tree mount point is identical — `helm lint --strict` + `helm template` + `helm install --dry-run` should all continue to pass. This OKR's role is verification, not change.

## Objectives

### O1: Verify chart gates pass post-submodule-add

source: ceo-brief

Why it matters: The chart is the most direct live consumer of the resink-core working-tree. If submodule-add breaks the chart's gates (it shouldn't), this loop must surface that before the parent newbase commit lands. One-pass verification.

**Key results**

- KR1.1: Post-O1-completion (resink-core's submodule add), run `helm lint --strict deploy/charts/nanofab-supervisor` (from `repos/resink-ai/resink-core/`). Expected: exit 0.
- KR1.2: Run `helm template release deploy/charts/nanofab-supervisor --values deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml --namespace nanofab > /dev/null`. Expected: exit 0.
- KR1.3: Run `helm install release deploy/charts/nanofab-supervisor --values deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml --namespace nanofab --dry-run --debug > /dev/null`. Expected: exit 0.
- KR1.4: Record outcomes in DevOps exec summary; no chart changes expected.
- KR1.5: Tenant-isolation invariant holds; zero edits beyond the team's exec summary.

**Tasks**

- [ ] Run all three helm gates against the chart post-submodule-add.
- [ ] Record outcomes in exec summary.
- [ ] Tenant-isolation dry-run.

## Risks

- **Chart paths reference parent-relative paths.** Unlikely (chart uses workspace-relative paths), but if any template references `../../../../` to escape the chart dir, those paths might differ under submodule. Mitigation: helm-template is the canonical test.

## Out of scope

- Job-kind chart variant (last loop's P2; pickup in a future deployment-focused loop).
- Image registry (home-cluster roadmap Phase 1; deferred).
- pyinfra wrapper at `services/nanofab_supervisor/` (deferred).
{% endraw %}
