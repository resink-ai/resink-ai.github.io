---
layout: default
title: platform-sre OKR — 2026-05-12-1254
date: 2026-05-12
status: active
type: okr
loop: 2026-05-12-1254
owner: teams/platform/sre
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/sre
  date: 2026-05-12
  status: active
  loop: 2026-05-12-1254
  links: parent: board/okrs/2026-05-12-1254-ceo-brief.md
-->
{% raw %}

# SRE OKR — 2026-05-12 (loop 2026-05-12-1254)

## Context

Light supporting role. The deployment SOP at `teams/platform/sre/runbooks/nanofab-supervisor-deployment.md` and the failed-validation runbook at `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` both reference paths inside `repos/resink-ai/resink-core/`. Post-submodule-add, these paths resolve identically. This OKR's role is one-pass verification + a footer note on each runbook acknowledging the submodule status.

## Objectives

### O1: Verify SOP cross-references resolve post-submodule + add runbook footer notes

source: ceo-brief

Why it matters: The SOP is the canonical deploy reference. If submodule-add breaks any of its cited commands, a future operator following the SOP would hit confusion. One-pass spot-check + a small footer note keeps the runbook's surface clean.

**Key results**

- KR1.1: Spot-check `cd repos/resink-ai/resink-core` resolves to the submodule's working tree post-promotion. `docker build -f deploy/charts/nanofab-supervisor/Dockerfile -t nanofab-supervisor:0.3.0 . --no-cache=false` runs (use cache for speed; we're testing path resolution, not full rebuild). Expected: docker build progresses past the first `COPY Cargo.toml ./` step (which confirms the path resolves). No full rebuild needed.
- KR1.2: `kubectl logs` invocation from the SOP still resolves the home-cluster context. (Spot-check, no actual deploy.)
- KR1.3: Add a single-line footer note to `teams/platform/sre/runbooks/nanofab-supervisor-deployment.md` § Cross-references: "**Post-promotion note (2026-05-12-1254):** `repos/resink-ai/resink-core/` is now a git submodule; all cited paths continue to resolve at the same working-tree mount point. Future runbook revisions may shift to submodule-relative paths if external consumers need disambiguation."
- KR1.4: Add the same single-line footer to `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` § Cross-references (or equivalent section).
- KR1.5: Refresh `teams/platform/sre/status.md` "Current focus" with this loop's verification + the workspace-promotion note (1 paragraph).
- KR1.6: Tenant-isolation invariant holds.

**Tasks**

- [ ] Spot-check deploy SOP commands resolve post-submodule.
- [ ] Add footer note to both runbooks (single line each).
- [ ] Refresh status.md.
- [ ] Tenant-isolation dry-run.

## Risks

- **None.** SOPs reference paths that resolve identically pre- and post-submodule. The footer notes are advisory; no path rewrites.

## Out of scope

- Job-kind chart variant SOP update (depends on chart variant; future loop).
- `dlopen-plugins` failure modes in the failed-validation runbook (depends on CI integration; future loop).
- `BLOCKED` observable verification (multi-loop wait on resink-core).
- Observability-stack-aware SOPs (depend on Phase 1.3 home-cluster roadmap).
{% endraw %}
