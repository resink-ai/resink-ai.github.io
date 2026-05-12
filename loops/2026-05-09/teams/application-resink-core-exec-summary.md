---
layout: default
title: application-resink-core Exec Summary — 2026-05-09
nav_exclude: true
render_with_liquid: false
date: 2026-05-09
status: active
type: exec-summary
loop: 2026-05-09
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/resink-core
  date: 2026-05-09
  status: active
  loop: 2026-05-09
  links: parent: teams/application/resink-core/okrs/2026-05-09-team-okr.md
-->
# Resink Core Exec Summary — 2026-05-09

## What we shipped

- Full Training → Serving plan for `fact_sign_up.parquet` written as the § Plan section of [our team OKR](../okrs/2026-05-09-team-okr.md), covering: input schema (8 columns, `event_id` PK), three proposed dim tables (`dim_user_signup` SCD T1, `dim_signup_funnel_daily` daily-partition append, `dim_device_user_link` SCD T1), the streaming consumption pattern (Spark Structured Streaming per [ADR-002](../../../board/decisions/2026-05-09-002-streaming-engine-choice.md)), the deployment pattern (minikube per [ADR-003](../../../board/decisions/2026-05-09-003-deployment-target.md)), and the validation contract (per sim-farm's contract).
- Streaming constraints surfaced in writing to DE before the engine ADR landed. DE's ADR explicitly cites this team's surfaced constraints; the recommendation moved from speculative to defensible because of it.
- Validation failure-mode shapes surfaced in writing to sim-farm; sim-farm's contract enumerates all seven of the failure modes this team named.
- Deployment shape (single Spark job in a pod, PVC outputs, liveness signal via Spark driver UI on its k8s service) surfaced for SRE's runbook drafting (deferred this loop, but inputs are captured for next loop).
- "First build slice" subsection in the OKR plan names the smallest committable next-loop build chunk: a Spark job that reads one parquet file and writes only `dim_user_signup` with idempotency, deployed to minikube via a single Job manifest.

## What we didn't ship and why

- Nothing was deferred. All six tasks ticked. The first build slice is the next-loop deliverable.

## Surprises

- Writing constraints down inside our OKR plan addendum *before* DE and DevOps committed their ADRs let both ADRs land cleanly in one loop. This was unanticipated leverage from the new playbook (`org-os/playbooks/onboard-application-team.md`, ADR-004), specifically its "surface cross-team asks explicitly" step.
- The dim-table SCD strategies turned out smaller than feared: SCD Type 1 covers the first cut for two of the three dim tables. We had budgeted more design time for SCD; that budget rolls into the next-loop build slice.

## Asks

- None this loop. All inputs landed; the next-loop build slice is well-scoped.

## Metrics

- KR1.1 (Training → Serving plan): met — sub-sections (a) input schema, (b) dim-table outputs, (c) streaming pattern, (d) deployment pattern, (e) validation contract — all written.
- KR1.2 (streaming constraints surfaced to DE): met — surfaced in writing; DE's ADR cites them.
- KR1.3 (validation failure-mode shapes surfaced to sim-farm): met — sim-farm's contract enumerates all seven shapes.
