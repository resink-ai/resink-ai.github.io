---
layout: default
title: platform-devops Exec Summary — 2026-05-12-1254
date: 2026-05-12
status: active
type: exec-summary
loop: 2026-05-12-1254
owner: teams/platform/devops
grand_parent: Loops
parent: Loop 2026-05-12-1254
nav_order: 19
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/devops
  date: 2026-05-12
  status: active
  loop: 2026-05-12-1254
  links: parent: board/okrs/2026-05-12-1254-ceo-brief.md
  team_okr: teams/platform/devops/okrs/2026-05-12-1254-team-okr.md
-->
{% raw %}

# DevOps Exec Summary — Loop 2026-05-12-1254

**Headline.** Light supporting role. Three chart gates verified post-submodule-add — all exit 0 against `values/home-cluster-mvp.yaml`. No chart changes needed; the working-tree mount point is unchanged so the chart's relative paths continue to resolve. One pre-existing chart behavior surfaced (out of scope this loop): `helm lint --strict` against default `values.yaml` fails because `tenant: ""` violates the schema's `minLength: 1` — that's by-design (tenant is a required field with no sensible default) and was true pre-promotion. Real lint requires values that supply tenant.

## Per-KR rollup (O1)

| KR | Outcome | Notes |
|---|---|---|
| KR1.1 (`helm lint --strict`) | **PASS** | `helm lint --strict deploy/charts/nanofab-supervisor --values deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml` → "1 chart(s) linted, 0 chart(s) failed". |
| KR1.2 (`helm template`) | **PASS** | `helm template release deploy/charts/nanofab-supervisor --values values/home-cluster-mvp.yaml --namespace nanofab` produces 192-line output, exits 0. |
| KR1.3 (`helm install --dry-run`) | **PASS** | `helm install release deploy/charts/nanofab-supervisor --values values/home-cluster-mvp.yaml --namespace nanofab --dry-run --debug` exits 0. |
| KR1.4 (outcomes recorded) | **PASS** | This exec summary. |
| KR1.5 (tenant-isolation) | **PASS** | Zero `org-os/` writes; zero edits beyond this exec summary. |

## Build phase signal-of-interest

- **`helm lint --strict` against default values has always failed.** Pre-existing behavior; chart's `tenant` field has no default (must be supplied), so the schema's minLength fails against the empty string. This was masked in prior loops because the dry-run target invocations always supplied values files. Not a submodule-induced bug; not a regression. Future-loop bookkeeping candidate: either set a placeholder default like `tenant: "REPLACE_ME"` so lint-without-values surfaces clean (+ schema warning), OR document this in chart README's Prerequisites.

## Cross-team coordination

- **Resink-core's submodule add succeeded;** chart paths still resolve. DevOps's role this loop is purely verification — no chart-side changes needed because the mount point is unchanged.

## Open carries (for next loop)

- **Job-kind chart variant** — last loop's retro P2 (clean fix for the `kubectl cp` race + Deployment-cycling-pod issue). Deferred to a future deployment-focused loop.
- **In-cluster image registry** — home-cluster roadmap Phase 1; deferred.
- **pyinfra wrapper at `services/nanofab_supervisor/`** — home-cluster roadmap Phase 1.1; deferred.
- **`Chart.appVersion` ↔ supervisor crate version alignment** — Chart.appVersion is `0.1.0`, crate is `0.3.0`. Bookkeeping; pickup at next chart revision.
- **`/healthz` + `/readyz` flip** — depends on resink-core's long-running supervisor; no carry this loop (forward-compat values pre-wired).

## Tenant-isolation invariant

Held. Zero writes anywhere outside this exec summary.
{% endraw %}
