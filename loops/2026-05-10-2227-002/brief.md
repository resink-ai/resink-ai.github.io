---
layout: default
title: CEO Brief — 2026-05-10-2227-002
date: 2026-05-10
status: active
type: okr
loop: 2026-05-10-2227-002
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-10
  status: active
  loop: 2026-05-10-2227-002
  links: parent: ""
-->
# Resink.ai CEO Brief — 2026-05-10

## Context

This is an off-cycle, CEO-called loop. The previous loop closed 2026-05-09 with both critical-path platform ADRs landed (ADR-002 Spark, ADR-003 minikube) and resink-core's first product-shaped plan in writing. Since that loop closed, [`ORG.md`](../../ORG.md) has been updated and five design specs have landed under [`docs/superpowers/specs/`](../../docs/superpowers/specs/) that re-baseline what we're building:

- **Sub-project #1** — [Nanofab Runtime](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md): a distributed *Rust* application (supervisor + cdylib node plugins + external KV + Kafka ingress) that executes a customer's generated DAG. Explicitly *not* a general SQL engine.
- **Sub-project #2** — [Nanofab Training Pipeline](../../docs/superpowers/specs/2026-05-10-nanofab-training-pipeline-design.md): an AI agent system that turns customer parquet samples (and optional SQL ETLs and chat) into a deployable nanofab DAG, gated by a four-stage validator.
- **Sub-project #3** — [Nanofab Sim Farm](../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md): the verification authority. Three execution modes (pre-deploy, per-node shadow, blue/green). Reuses the supervisor binary in `--mode=sim`.
- **Sub-project #4** — [Nanofab Serving & Deployment](../../docs/superpowers/specs/2026-05-10-nanofab-serving-deployment-design.md): v1 multi-tenant SaaS on resink-managed AWS; CI/CD gated by Sim Farm seal + manifest signature.
- **Sub-project #5** — [Nanofab Product UX](../../docs/superpowers/specs/2026-05-10-nanofab-product-ux-design.md): web app + power-user CLI + Slack/email notifications.

**The gap.** Two loops ago, the company was bootstrapping its org-OS. Last loop, it landed two Spark/k8s ADRs and a Spark-shaped first-build-slice plan for resink-core. The new vision says the runtime is purpose-built Rust — no JVM, no general SQL engine. The largest gap from `ORG.md` is now organizational, not technical: every artifact downstream of the engine-choice ADR (resink-core's plan, sim-farm's contract scope, SRE's runbook target) was shaped against Spark. Until we explicitly absorb the pivot and re-baseline ownership, every team's next-loop work will be aimed at the wrong target. This loop's job is to absorb the pivot — cleanly and ADR-gated — and re-aim every in-flight workstream. It is **not** to start building the new runtime; that's multi-loop work.

**CEO decisions for this loop:**

- **ADR-002 (Spark Structured Streaming) is superseded by a new ADR this loop.** The new ADR records that the nanofab runtime is purpose-built Rust per sub-project #1's spec. ADR-002's status moves to `archived` with a `superseded_by:` link.
- **ADR-003 (minikube) stays `active`.** The k8s-shaped deployment surface survives the pivot (the serving spec is k8s-based; minikube is still the right local-dev target). Cloud target lands in sub-project #4 work, not this loop.
- **Team ownership of the five sub-projects:**
  - #1 Nanofab Runtime → `teams/application/resink-core/` (stretch).
  - #2 Nanofab Training Pipeline → `teams/application/resink-core/` (stretch; same team owns both ends of the customer journey per the existing charter).
  - #3 Nanofab Sim Farm → `teams/application/sim-farm/` (already aligned; charter update only).
  - #4 Nanofab Serving & Deployment → split: `teams/platform/devops/` (deployment topology, IaC, CI/CD gate, secrets) + `teams/platform/sre/` (operational shape, runbooks, on-call, capacity).
  - #5 Nanofab Product UX → ownership decision **deferred** to a future loop. No team work this loop.
- **Resink-core's pre-pivot "first build slice"** (Spark Job → `dim_user_signup` on minikube) **does not survive the pivot.** Resink-core's next-loop product-shaped OKR targets the nanofab runtime and training pipeline sub-projects, not a Spark pipeline.

**Answer to the open DevOps ask (manifest tool, from 2026-05-09 exec summary):** **Helm** is the preferred manifest tool. Rationale: sub-project #4's multi-tenant logical-isolation pattern (per-tenant IAM, KV prefix, Kafka ACLs in shared infrastructure) is exactly the per-tenant-parameterization shape Helm charts handle well. Kustomize overlays scale poorly when every tenant needs distinct values; raw YAML is non-starter for multi-tenant. DevOps may push back in their team OKR if the serving spec's CI/CD details surface a constraint Helm cannot meet.

**Carryover load entering this loop** (informal; the formal "Carryover load by team" section lands when ADR-005 is picked up loop 2026-05-11-1113):

- **DE** — shared streaming primitive contract (deferred from 2026-05-09 KR2.1). **Re-shaped by pivot:** the "streaming primitive" was Spark-flavored; under the new runtime, the closest analogue is the Kafka ingress contract from runtime spec §3. DE absorbs this re-aim as part of O2.
- **DevOps** — frontmatter-lint script (deferred again this loop; manual ADR-001 enforcement continues); manifest-tool decision (answered above).
- **SRE** — runbook draft "demo pipeline failed validation" (deferred from 2026-05-09 KR4.2). **Re-shaped by pivot:** SRE's runbook target is now the nanofab supervisor's seven Sim-Farm-enumerated failure modes against the new runtime, not a Spark Job. SRE absorbs this re-aim as part of O3.
- **AE** — 17-task bottom-up flow implementation (scheduled for this loop). **Unchanged by pivot** — this is org-os work, orthogonal to the product re-baseline.
- **Resink-core** — first build slice (cancelled by pivot; replaced by O2 work).
- **Sim-farm** — minimal test-data generator + first assertions (scheduled for this loop). **Re-shaped by pivot:** target is nanofab supervisor in `--mode=sim`, not a Spark pipeline. Sim-farm absorbs the re-aim as part of O2.

**P3 (named hand-off sections convention) and P4 (resink-core product repo creation)** from the 2026-05-09 retro both land in their respective team OKRs this loop. P4's repo purpose changes — it now hosts the Rust nanofab workspace, not a Spark pipeline.

## Objectives

### O1: Ratify the vision pivot via ADRs

Why it matters: The runtime spec (`#1`) is incompatible with ADR-002 (Spark). Until ADR-002 is superseded explicitly, every downstream artifact is suspect. The five-sub-project decomposition is the largest structural product decision the company has made; per `org-os/playbooks/merge-evolution-proposal.md` and ADR-001's frontmatter contract, it needs an ADR. Closing both this loop drains every "is this still the plan?" question from team planning.

**Key results**
- KR1.1: ADR `2026-05-10-001-nanofab-runtime-is-rust.md` exists with `status: active`, supersedes ADR-002 explicitly (`supersedes: 2026-05-09-002-streaming-engine-choice`), and grounds the recommendation in the runtime spec's §1.2 ("Skip the SQL execution layer") and §2 decisions summary.
- KR1.2: `board/decisions/2026-05-09-002-streaming-engine-choice.md` status updated from `active` to `archived` with `superseded_by: 2026-05-10-001-nanofab-runtime-is-rust` in frontmatter.
- KR1.3: ADR `2026-05-10-002-nanofab-sub-project-decomposition.md` exists with `status: active`, names the five sub-projects, and records the team-ownership mapping above (including the deferred Product UX team).

**Tasks**
- [ ] Draft + land ADR `2026-05-10-001-nanofab-runtime-is-rust.md` — owner: teams/platform/data-engineering (DE drafts as the team whose prior ADR is being superseded; board approves)
- [ ] Update ADR-002's frontmatter to `archived` with `superseded_by` link — owner: teams/platform/data-engineering
- [ ] Draft + land ADR `2026-05-10-002-nanofab-sub-project-decomposition.md` — owner: board

### O2: Re-aim resink-core and sim-farm against the new specs (no code yet)

Why it matters: Resink-core and sim-farm both shipped product-shaped OKRs last loop pointed at Spark. The pivot means both teams need new product-shaped OKRs this loop that target the new specs. This is plan-only this loop — the goal is alignment, not implementation. Code starts next loop or the one after. The "named hand-off sections" pattern from the 2026-05-09 retro applies: any hand-off doc consumed by ≥2 teams uses one document with named sections, not multiple.

**Key results**
- KR2.1: `teams/application/resink-core/okrs/2026-05-10-2227-002-team-okr.md` exists with `status: active`, declares ownership of sub-projects #1 (Runtime) and #2 (Training Pipeline), and contains a re-aimed plan that picks one narrow first slice grounded in the new runtime + training-pipeline specs (no Spark). The plan names the cross-team hand-offs (DE for Kafka/KV contract surfaces, sim-farm for the supervisor `--mode=sim` interface, DevOps for the Helm deployment shape).
- KR2.2: `teams/application/sim-farm/okrs/2026-05-10-2227-002-team-okr.md` exists with `status: active`, re-scopes the validation contract from "Spark pipeline outputs" to "nanofab supervisor outputs", and names the supervisor `--mode=sim` interface (per Sim Farm spec §1.3) as the integration target. Generation breadth remains out of scope (per the spec, generation lives in training, not Sim Farm).
- KR2.3: Resink-core creates the product repo per P4 (path under `repos/`, most likely `repos/resink-ai/resink-core/`), mounted as a submodule. Repo is a Rust Cargo workspace skeleton, not a Spark project. README links back to the runtime and training-pipeline specs.
- KR2.4: DE writes the Kafka ingress contract for the nanofab runtime (`teams/platform/data-engineering/`-located, named-hand-off-sections format, consumed by resink-core), replacing the deferred "shared streaming primitive contract" from 2026-05-09 KR2.1.

**Tasks**
- [ ] Draft resink-core's re-aimed product-shaped OKR — owner: teams/application/resink-core
- [ ] Draft sim-farm's re-aimed product-shaped OKR — owner: teams/application/sim-farm
- [ ] Create the resink-core product repo + Cargo workspace skeleton — owner: teams/application/resink-core
- [ ] Write the Kafka ingress contract — owner: teams/platform/data-engineering

### O3: Seat DevOps + SRE on Serving & Deployment (#4); first OKR shape against the spec

Why it matters: Sub-project #4 has two natural homes that map to existing platform charters. Naming this explicitly this loop lets both teams start producing artifacts against the spec next loop (Terraform module sketches, CI/CD gate design, runbook drafts) rather than waiting for an org-design step. Both teams' charters need a charter-touch update to reflect the new ownership; their team OKRs this loop seed the first chunk of #4 work without committing to building it.

**Key results**
- KR3.1: DevOps charter (`teams/platform/devops/charter.md`) updated to declare ownership of sub-project #4's deployment topology, IaC, CI/CD gate, and secrets surfaces. Helm preference (from this brief's context) recorded as a charter-level convention.
- KR3.2: SRE charter (`teams/platform/sre/charter.md`) updated to declare ownership of sub-project #4's operational shape, runbooks, on-call, and capacity surfaces.
- KR3.3: DevOps and SRE each produce a team OKR this loop that names their first product-shaped slice under sub-project #4, grounded in the serving spec. DevOps's first slice may be: "Helm chart skeleton for the supervisor binary deployment, parameterized for tenant + version, against minikube." SRE's first slice may be: re-aim the deferred 2026-05-09 KR4.2 runbook ("demo pipeline failed validation") to the nanofab supervisor's seven Sim-Farm-enumerated failure modes.
- KR3.4: Both team OKRs ship as **plan-only this loop** (no code/runbooks yet) — implementation starts next loop. The plan-only framing matches O2 and is a conscious capacity choice.

**Tasks**
- [ ] Update DevOps charter to claim sub-project #4 deployment/CI-CD surfaces — owner: teams/platform/devops
- [ ] Update SRE charter to claim sub-project #4 operational surfaces — owner: teams/platform/sre
- [ ] Draft DevOps's first sub-project-#4 team OKR (plan-only) — owner: teams/platform/devops
- [ ] Draft SRE's first sub-project-#4 team OKR (plan-only) — owner: teams/platform/sre

### O4: AE executes the bottom-up flow implementation

Why it matters: AE was scheduled to pick up the 17-task bottom-up flow implementation this loop per the 2026-05-09 retro (P-classified org-os change, ADR-001 of 2026-05-09 ratified). The product pivot does not touch this work — it's org-os infrastructure, not nanofab. Executing it on schedule preserves the org-os evolution cadence and demonstrates that the org-os layer and the tenant product evolve independently.

**Key results**
- KR4.1: All 17 tasks in `docs/superpowers/plans/2026-05-09-org-os-bottom-up-flow.md` are sized (S/M/L), grouped into 2–3 milestone bundles with explicit definition-of-done per bundle, and the first bundle is **executed** (not just planned) this loop. (Sizing + grouping was AE's deferred "next loop carryover" per status.md.)
- KR4.2: AE's team OKR records which tasks shipped vs. carried over, by bundle, with reasons for any defer.
- KR4.3: Tenant-isolation dry-run (`org-os/playbooks/extract-org-os.md`) passes after the first bundle ships — no tenant or product names appear inside `org-os/` from the implementation.

**Tasks**
- [ ] Size + group the 17 tasks into 2–3 milestone bundles — owner: teams/platform/agent-engineering
- [ ] Execute the first milestone bundle — owner: teams/platform/agent-engineering
- [ ] Run the tenant-isolation dry-run after the first bundle — owner: teams/platform/agent-engineering

## Risks

- **Pivot under-absorbed in team plans.** Teams may write next-loop OKRs that still carry Spark-shaped thinking under new labels. Mitigation: O1's ADRs land before team planning is reviewed; O2/O3 require explicit citations of the new spec paths.
- **Resink-core stretch capacity.** Owning sub-projects #1 and #2 is a meaningful stretch for a team that just shipped its first product-shaped OKR. Mitigation: plan-only framing for both O2 and O3 — no code this loop — keeps the load to writing rather than building, and lets resink-core size implementation in the *next* loop with full information.
- **AE bottom-up flow execution capacity.** The 17 tasks plus sizing-and-grouping is the largest single-team commitment this loop. Mitigation: O4 KR4.1 only requires the *first* bundle to execute, not all three. Sizing-and-grouping is the gate; first bundle is the floor; remaining bundles are stretch.
- **Helm answer may be wrong.** The CEO answer to DevOps's manifest-tool ask was made before reading the serving spec's CI/CD section in detail. Mitigation: DevOps's O3 team OKR may push back with a concrete blocker; the answer is a default, not a lock.
- **No carryover-load tally template yet.** Per the 2026-05-09 retro, the formal "Carryover load by team" section is gated on ADR-005 (deferred to 2026-05-23). This brief uses an informal carryover-load paragraph in Context. Mitigation: when ADR-005 lands, the formal template subsumes the paragraph; nothing in this loop's brief blocks that.

## Out of scope this loop

- Starting actual Rust nanofab runtime build (multi-loop work; this loop is plan-only for sub-projects #1 and #2).
- Creating teams for sub-projects (specifically: no Product UX team charter this loop).
- Sub-project #5 (Product UX) work of any kind — ownership and first OKR are deferred to a future loop.
- P1 / ADR-005 (carryover-load section in brief) — deferred to loop 2026-05-11-1113 per the 2026-05-09 retro.
- P2 / ADR-006 (out-of-retro routing path for org-os changes) — deferred to loop 2026-05-11-1113 per the 2026-05-09 retro.
- DevOps frontmatter-lint script — deferred again; manual ADR-001 enforcement continues.
- Cloud k8s deployment target (lives inside sub-project #4 work, not this loop's plan-only scope).
- Broader fact-table coverage beyond what resink-core scopes in O2 KR2.1.
