---
layout: default
title: platform-devops OKR — 2026-05-11-2153
date: 2026-05-11
status: active
type: okr
loop: 2026-05-11-2153
owner: teams/platform/devops
grand_parent: Loops
parent: Loop 2026-05-11-2153
nav_order: 14
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/devops
  date: 2026-05-11
  status: active
  loop: 2026-05-11-2153
  links: parent: board/okrs/2026-05-11-2153-ceo-brief.md
-->
{% raw %}

# DevOps OKR — 2026-06-13

## Context

Board-action `2026-06-06-002` (install minikube on dev machine) hit its 2026-06-13 forcing function; the brief's CEO probe records `which minikube` returning "not found", so per the brief's working answer to retro P5 — **toolchain installs are per-team Prerequisite docs, not board-action tickets** — DevOps closes the ticket as `superseded` (not `done`). The chart README at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` already names `brew install minikube` in its Prerequisites section, so the next minikube smoke attempt picks up the toolchain via the team's own surface; no new board-action ticket needed. The minikube smoke itself remains carry (4th loop) until the toolchain lands. This loop also responds to board O1's GitBook publishing scope queries on the chart README + `values.schema.json`.

## Objectives

### O1: DevOps closes board-action 2026-06-06-002 either way and absorbs the GitBook publishing scope query for chart contracts

source: ceo-brief

Why it matters: Board-action `2026-06-06-002` hit its 2026-06-13 forcing function; per the brief's working answer to retro P5, toolchain installs route through team Prerequisite docs (not board-action tickets), and the chart README already names `brew install minikube`. Closing the ticket here as `superseded-by-team-prerequisite-pattern` (probe shows minikube not installed) makes this the LAST toolchain board-action; future toolchain blockers route through `<chart>/README.md § Prerequisites`. The minikube smoke remains carry until the toolchain lands. The companion ask is responding to board O1's GitBook publishing scope queries on chart-side artifacts (chart README + `values.schema.json`) so board can lock the publishing-scope filter for the first publication.

**Key results**

- KR5.1: [x] ✅ done — `board/actions/2026-06-06-002-install-minikube-on-dev-machine.md` flipped `status: open → status: superseded`. Resolution subsection added naming supersession reason verbatim per the brief's standing answer to retro P5, with a forward-link to chart README `§ Prerequisites`. Probe (`which minikube`) confirmed exit 1 in this loop's build phase; default action triggered as predicted.
- KR5.2: [x] ✅ done — Chart README's `Prerequisites` section gained a top-line blockquote policy note pointing at CEO brief 2026-06-13 + ADR-2026-06-06-001 (provisional-and-migrate playbook, pattern reuse). Existing `brew install minikube` invocation preserved verbatim below the policy note.
- KR5.3: [x] ✅ done — Minikube smoke remains carry; toolchain probe negative this loop (exit 1). No re-attempt run. Pickup happens whenever toolchain lands via chart README Prerequisites; no new board-action ticket. Carry documented in this loop's exec summary.
- KR5.4: [x] ✅ done — GitBook publishing scope response authored: chart README IN scope (under `/repos/resink-core/charts/` namespace per board's publishing-tree decision); `values.schema.json` OUT of gitbook scope by default (JSON not markdown; transitively discoverable via in-scope README); `values.yaml` + templates OUT of scope (deployment-time configuration, not user-facing docs). Recommendation: future loop can revisit if external readers ask for chart-config visibility.
- KR5.5: [x] ✅ done — Tenant-isolation invariant holds. Edits landed in `board/actions/2026-06-06-002-install-minikube-on-dev-machine.md` and `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md`; no `org-os/` writes. Final `grep -nrE "(resink|nanofab|acme\.ai)" org-os/` returns only the pre-existing `acme.ai` placeholder at `org-os/conventions.md:107`.

**Tasks**

- [x] ✅ done — Verified `which minikube` exits non-zero (probe returns "minikube not found", exit 1) in this loop's build phase, matching the brief's probe. Updated `board/actions/2026-06-06-002-install-minikube-on-dev-machine.md` with `status: superseded` and Resolution subsection citing brief's standing answer + chart README Prerequisite link.
- [x] ✅ done — Added policy-note one-liner (blockquote) + ADR-2026-06-06-001 forward-link at the top of the chart README's `Prerequisites` section. Existing `brew install minikube` invocation preserved verbatim.
- [x] ✅ done — Minikube-smoke carry documented in this loop's exec summary cross-team coordination + carry-into-next-loop sections; chart README Prerequisites is the toolchain pickup surface; no new board-action ticket.
- [x] ✅ done — One-paragraph GitBook publishing scope response authored in this loop's exec summary's Cross-team coordination section: chart README IN scope (under `/repos/resink-core/charts/` namespace per board's publishing-tree decision); `values.schema.json` OUT of gitbook scope by default; `values.yaml` + templates OUT of gitbook scope.
- [x] ✅ done — Tenant-isolation dry-run: `grep -nrE "(resink|nanofab|acme\.ai)" org-os/` returns only the pre-existing `acme.ai` placeholder (`org-os/conventions.md:107`). Invariant holds.

## Risks

- **Minimal.** Closing the action ticket as `superseded` is mechanical — the supersession reason is named verbatim in the brief's standing answer and the chart README already has the canonical Prerequisite text. The GitBook scope-query response is a one-paragraph note in the exec summary with no implementation cost on DevOps's side (board's publishing script does the actual filtering work in O1).
- **Edge case:** if board's O1 publishing script treats `repos/resink-ai/resink-core/deploy/charts/*/README.md` as out-of-scope by default (only `docs/` trees in-scope), DevOps's KR5.4 recommendation needs board to extend the filter. Mitigation: KR5.4's scope-response note lands early enough in the loop that board can take the recommendation into O1 KR1.6 (publishing-scope documentation) without rework.

## Out of scope this loop

- **Minikube smoke full execution** — depends on toolchain install per the brief's working answer to P5; carry through the chart README Prerequisites; pickup happens whenever the toolchain lands. No new board-action ticket.
- **CI/CD chart-publishing pipeline** (`helm push` to an OCI registry on supervisor-binary release) — sized later.
- **Multi-tenant install automation** (operator or installer fanning `helm install` across tenants) — later sub-slice.
- **Real IRSA** (`irsa.enabled=false` is the only path tested; EKS-shaped IRSA testing remains in a later sub-slice).
- **Frontmatter-lint CI script** — fifth consecutive defer per the 2026-06-13 brief § Out of scope. Manual ADR-2026-05-08-001 enforcement continues.
- **Supervisor-binary resource-budget doc** — still deprioritized; sized once a real (non-sim) workload runs.
- **R2 `/healthz` + `/readyz` flip** — still waits on resink-core's long-running supervisor (`loop+1` per ADR-2026-05-16-001); this loop's resink-core O3 is step 2 (full execution against the dlopen template), not the long-running shape.

## Cross-team asks

- **From `board`:** take this OKR's KR5.1 update to `board/actions/2026-06-06-002` as authoritative — supersession reason and chart README pointer are both verbatim from the brief's standing answer to retro P5. No further board action needed for the close.
- **From `teams/application/resink-core`:** the chart README sees a small refresh this loop (KR5.2 — one-line policy note + ADR forward-link in the Prerequisites section). The README lives in resink-core's tree (`repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md`); the edit is non-load-bearing for chart functionality. One-line ack in resink-core's 2026-06-13 exec summary or counter-propose if resink-core wants the policy note rephrased.
- **From `board` (O1 GitBook publishing pipeline):** confirm whether `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` is in publishing scope (DevOps recommends IN; KR5.4 explains) OR explicitly out. The decision lands in board's O1 KR1.6 (publishing-scope documentation at `scripts/PUBLISHING-SCOPE.md`); DevOps's KR5.4 note is the input.
{% endraw %}
