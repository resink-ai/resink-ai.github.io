---
layout: default
title: platform-devops OKR — 2026-06-13
date: 2026-06-13
status: active
type: okr
loop: 2026-06-13
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/devops
  date: 2026-06-13
  status: active
  loop: 2026-06-13
  links: parent: board/okrs/2026-06-13-ceo-brief.md
-->
# DevOps OKR — 2026-06-13

## Context

Board-action `2026-06-06-002` (install minikube on dev machine) hit its 2026-06-13 forcing function; the brief's CEO probe records `which minikube` returning "not found", so per the brief's working answer to retro P5 — **toolchain installs are per-team Prerequisite docs, not board-action tickets** — DevOps closes the ticket as `superseded` (not `done`). The chart README at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` already names `brew install minikube` in its Prerequisites section, so the next minikube smoke attempt picks up the toolchain via the team's own surface; no new board-action ticket needed. The minikube smoke itself remains carry (4th loop) until the toolchain lands. This loop also responds to board O1's GitBook publishing scope queries on the chart README + `values.schema.json`.

## Objectives

### O1: DevOps closes board-action 2026-06-06-002 either way and absorbs the GitBook publishing scope query for chart contracts

source: ceo-brief

Why it matters: Board-action `2026-06-06-002` hit its 2026-06-13 forcing function; per the brief's working answer to retro P5, toolchain installs route through team Prerequisite docs (not board-action tickets), and the chart README already names `brew install minikube`. Closing the ticket here as `superseded-by-team-prerequisite-pattern` (probe shows minikube not installed) makes this the LAST toolchain board-action; future toolchain blockers route through `<chart>/README.md § Prerequisites`. The minikube smoke remains carry until the toolchain lands. The companion ask is responding to board O1's GitBook publishing scope queries on chart-side artifacts (chart README + `values.schema.json`) so board can lock the publishing-scope filter for the first publication.

**Key results**

- KR5.1: `board/actions/2026-06-06-002-install-minikube-on-dev-machine.md` flips `status: open → status: superseded` this loop. Resolution subsection added naming the supersession reason verbatim ("per CEO brief 2026-06-13's working answer to retro P5: toolchain installs route through team Prerequisite docs, not board-action tickets") with a forward-link to the chart README's existing `brew install minikube` Prerequisite. Per the brief's standing answer, this is the default action since the brief's `which minikube` probe returned "not found" at brief-authoring time; DevOps confirms the same probe state in this loop's build phase before flipping. (If `which minikube` exits 0 at flip time — unlikely given the probe state — the close-reason becomes `done` with a one-line install-date + version capture instead.)
- KR5.2: Chart README (`repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md`) `Prerequisites` section gets a one-line policy note at the top: "Toolchain installs are documented here, not in board-action tickets, per CEO brief 2026-06-13" with a forward-link to ADR-2026-06-06-001 (provisional-and-migrate playbook — pattern reuse). The existing `brew install minikube` invocation is preserved verbatim.
- KR5.3: Minikube smoke (KR carry from 2026-06-06) remains carry. Re-attempt happens only if `which minikube` exits 0 in DevOps's build phase this loop (probe says no). If not, the smoke carries via the chart README's Prerequisites; no new board-action ticket. The carryover hand-off is documented in this loop's exec summary.
- KR5.4: Board's GitBook publishing scope query response (one-paragraph note in this loop's exec summary): chart README at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` is **IN scope** (it's chart-side documentation that documents the public chart interface — Prerequisites, Parameters, Cross-team contracts — and is part of the resink-core docs surface that board O1 KR1.3 names as in-scope under the `repos/resink-core/` namespace). `values.schema.json` is JSON not markdown; **out of gitbook scope by default** (it's referenced from the chart README which IS in scope, so it's discoverable transitively without standalone publishing). DevOps recommends the publishing script either skips JSON files or renders them as fenced code blocks under the chart README's "Schema" anchor (script author's choice).
- KR5.5: Tenant-isolation invariant holds. No `org-os/` edits this objective. Edits land in `board/actions/` (board-namespace artifact) and `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` (product-repo docs). Dry-run captured in exec summary.

**Tasks**

- [ ] Verify `which minikube` exits non-zero in this loop's build phase (matches the brief's probe); update `board/actions/2026-06-06-002-install-minikube-on-dev-machine.md` with `status: superseded` and a Resolution subsection citing the brief's standing answer + chart README Prerequisite link — owner: teams/platform/devops.
- [ ] Add the policy-note one-liner + ADR-2026-06-06-001 forward-link to the chart README's `Prerequisites` section — owner: teams/platform/devops.
- [ ] Document the minikube-smoke carry in this loop's exec summary (carry note + chart README Prerequisites pointer; no new board-action ticket) — owner: teams/platform/devops.
- [ ] Author the one-paragraph GitBook publishing scope response (chart README in / `values.schema.json` out) for this loop's exec summary; surface the recommendation to board's O1 planning so the publishing-script filter can lock chart-side scope — owner: teams/platform/devops.
- [ ] Tenant-isolation dry-run after edits (`grep -nrE "(resink|nanofab|acme\.ai)" org-os/` returns only the pre-existing `acme.ai` placeholder); capture in exec summary — owner: teams/platform/devops.

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
