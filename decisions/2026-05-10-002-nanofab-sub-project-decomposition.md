---
layout: default
title: "ADR 2026-05-10-002: nanofab sub project decomposition"
date: 2026-05-10
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-10
  status: active
  decision: Decompose the resink.ai product into five named nanofab sub-projects (Runtime, Training Pipeline, Sim Farm, Serving & Deployment, Product UX) and assign team ownership; resink-core stretches to own #1 and #2; sim-farm owns #3; DevOps + SRE split #4; #5 ownership is deferred to a future loop.
-->
{% raw %}

# ADR 2026-05-10-002: Nanofab sub-project decomposition and team ownership

## Context

`ORG.md` was updated on 2026-05-10 to introduce **nanofab** — a per-customer distributed Rust application produced by a Training Pipeline and run by a Runtime — as the technical realization of the resink.ai product. Five design specs landed under `docs/superpowers/specs/` on the same day, each defining one named sub-project of the nanofab family:

| # | Sub-project | Spec | What it is |
|---|---|---|---|
| 1 | Nanofab Runtime | [2026-05-10-nanofab-runtime-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) | Distributed Rust runtime: stateless supervisor + cdylib node plugins + external KV + Kafka ingress. Hot-swap (per-node shadow/canary; per-DAG blue/green) gated by Sim Farm verdicts. |
| 2 | Nanofab Training Pipeline | [2026-05-10-nanofab-training-pipeline-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-training-pipeline-design.md) | AI agent system that turns customer parquet samples (and optional SQL ETLs + chat) into a deployable nanofab DAG. Four-stage validation gate. |
| 3 | Nanofab Sim Farm | [2026-05-10-nanofab-sim-farm-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md) | Verification authority. Three execution modes (pre-deploy validation, per-node shadow, blue/green warmup). Reuses the supervisor binary in `--mode=sim`. Generation is explicitly NOT in scope — that lives in training. |
| 4 | Nanofab Serving & Deployment | [2026-05-10-nanofab-serving-deployment-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-serving-deployment-design.md) | v1 multi-tenant SaaS on resink-managed AWS, logical isolation; CI/CD for AI-generated code gated by Sim Farm seal + manifest signature; per-tenant secrets via IRSA; usage-based billing. |
| 5 | Nanofab Product UX | [2026-05-10-nanofab-product-ux-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-product-ux-design.md) | Customer-facing surfaces: web app + power-user CLI + Slack/email notifications. Hybrid chat + sidebar during training; eight-tab operations dashboard; three-role RBAC plus Auditor. |

Loop 2026-05-10-2227-001 had committed plans against a different shape (Spark Structured Streaming via ADR-002, a Spark Job → `dim_user_signup` first-build slice). The pivot to a purpose-built Rust runtime supersedes ADR-002 (recorded separately in `board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md`, owned by DE). This ADR records the *organizational* consequence: the product is no longer one monolithic effort; it is five sub-projects with explicit team owners. Without this recording, every team OKR for loop 2026-05-10-2227-002 onward operates under implicit, undocumented ownership — exactly the gap retros 2026-05-08 and 2026-05-09 said the org-OS should prevent.

## Decision

Recognize five nanofab sub-projects and assign team ownership as follows:

- **#1 Nanofab Runtime → `teams/application/resink-core/`** (stretch).
- **#2 Nanofab Training Pipeline → `teams/application/resink-core/`** (stretch; the existing charter already says "same team owns both training and serving").
- **#3 Nanofab Sim Farm → `teams/application/sim-farm/`** (continuation of existing charter; charter wording may be updated in a near-term team-planning to reflect the new spec, but ownership itself is unchanged).
- **#4 Nanofab Serving & Deployment → `teams/platform/devops/` + `teams/platform/sre/`** (split):
  - DevOps owns: deployment topology, IaC, CI/CD gate (Sim Farm seal + manifest signature), secrets, cost accounting.
  - SRE owns: operational shape, runbooks, on-call rotation, capacity planning, availability response.
- **#5 Nanofab Product UX → ownership deferred** to a future loop. No team is named this loop. Standing up a dedicated Product UX team is itself a decision worth its own ADR (and likely an `onboard-team.md` execution); deferring keeps loop 2026-05-10-2227-002 to a re-baseline scope.

The two existing platform teams that are *not* assigned a sub-project — Data Engineering and Agent Engineering — retain their existing charters. **DE** continues to own streaming-data platform concerns; under the nanofab vision its most immediate work is the Kafka ingress contract for the runtime (see DE's 2026-05-10 OKR § O2). **AE** continues to own org-OS evolution and agent infrastructure (see AE's 2026-05-10 OKR — bottom-up flow implementation).

## Alternatives considered

- **A: Stand up dedicated teams for every sub-project this loop (5 new teams).** Rejected: spawning three new charter-only teams (Runtime, Training Pipeline, Product UX) inside a re-baseline loop multiplies coordination cost without delivering more product per dollar of attention. Charter-only onboarding consumes brief-time and shifts the loop's center of gravity from "absorb the pivot cleanly" to "stand up the org-chart." Resink-core stretching to own #1 + #2 is consistent with its existing charter and lets the team set its own internal seams when it splits.
- **B: Fold all five sub-projects under resink-core.** Rejected: the operational surface (#4) and the customer-facing surface (#5) are not "build the engine" work; their natural homes are the platform teams (DevOps + SRE) and a future product-UX team, respectively. Folding everything under resink-core would either re-create platform/application separation inside resink-core or starve those concerns of attention.
- **C: Defer the decomposition decision entirely.** Rejected: the specs are already written and committed; without an ADR-recorded ownership decision, every team would author their 2026-05-10 OKR against implicit assumptions about what they own. The cost of *not* deciding is exactly the kind of "every loop carries forward unresolved structure" pattern the 2026-05-09 retro flagged.

## Consequences

- **Positive:**
  - Every spec has a named owner (with the deliberate exception of #5).
  - Resink-core's charter ("same team owns Training and Serving") aligns naturally with owning #1 + #2.
  - The platform/application split is preserved: platform teams own infrastructure (#4); application teams own the data product (#1, #2, #3).
  - Future Product UX team standup is queued without forcing a decision under the time pressure of this loop.
- **Negative / costs:**
  - Resink-core's stretch covers two large sub-projects. The plan-only framing of loop 2026-05-10-2227-002 mitigates near-term risk, but resink-core will need to scope an internal seam (likely two crates under `repos/resink-ai/resink-core/`) and may eventually split into two teams.
  - DevOps + SRE share #4. The team-planning artifacts for both teams must record the seam explicitly (DevOps: deployable artifact + IaC + gate; SRE: runbooks + on-call + capacity); both teams' charters are updated this loop to do so.
  - #5 (Product UX) work is paused — the spec exists but no team owns it. Customer-facing surface work cannot begin until ownership lands.
- **Follow-ups required:**
  - Resink-core scopes its internal Runtime ↔ Training seam in a near-future loop. The repo skeleton at `repos/resink-ai/resink-core/` (created this loop) provides the workspace for this.
  - A future loop's brief proposes a Product UX team standup or assigns #5 to an existing team (most likely a new application team using `org-os/playbooks/onboard-application-team.md`).
  - DevOps and SRE charters are updated this loop (per DevOps and SRE OKRs O3 KR3.1 / KR3.2) to claim their respective halves of #4.

## Links

- **Triggering vision update:** [ORG.md](../../ORG.md) (commit on master 2026-05-10) and the five design specs in `docs/superpowers/specs/`.
- **Engine-choice pivot ADR (same loop):** [2026-05-10-001-nanofab-runtime-is-rust](2026-05-10-001-nanofab-runtime-is-rust.md) (owned by DE).
- **Superseded ADR:** [2026-05-09-002-streaming-engine-choice](2026-05-09-002-streaming-engine-choice.md) (now `archived`).
- **Related ADRs:** [2026-05-09-003-deployment-target](2026-05-09-003-deployment-target.md) (minikube — survives the pivot for local dev under #4 work), [2026-05-09-004-onboard-application-team](2026-05-09-004-onboard-application-team.md) (the playbook that will be used when #5's team is stood up).
- **Loop CEO brief:** [board/okrs/2026-05-10-2227-002-ceo-brief.md](../okrs/2026-05-10-ceo-brief.md).
{% endraw %}
