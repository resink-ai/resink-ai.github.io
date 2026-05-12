---
layout: default
title: application-resink-core OKR — 2026-05-10-2227-001
date: 2026-05-09
status: active
type: okr
loop: 2026-05-10-2227-001
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-09
  status: active
  loop: 2026-05-10-2227-001
  links: parent: board/okrs/2026-05-10-2227-001-ceo-brief.md
-->
# Resink Core OKR — 2026-05-09

## Context

Resink Core's first product-shaped OKR. The CEO brief confirms `fact_sign_up.parquet` as the demo target, names DE's engine-choice ADR and DevOps's deployment-target ADR as critical-path inputs, and explicitly scopes this loop to a *plan*, not yet a running pipeline. The job here is to produce a reproducible Training → Serving plan that can absorb the two ADRs as they land and that gives sim-farm a concrete validation target.

## Objectives

### O1: Produce a reproducible Training → Serving plan for `fact_sign_up.parquet`

Why it matters: This is the company's first product-shaped commitment. Without a written plan that names the input schema, dim-table outputs, engine consumption pattern, deployment pattern, and validation contract, every downstream team is guessing about what to build against. The plan is the alignment artifact.

**Key results**
- KR1.1: A plan document exists under `teams/application/resink-core/` covering: (a) the `fact_sign_up.parquet` input schema, (b) the proposed dim-table outputs (table names, columns, SCD strategy or partitioning choice), (c) the streaming engine consumption pattern (referencing DE's engine ADR landing this loop), (d) the deployment pattern (referencing DevOps's deployment-target ADR landing this loop), (e) the validation contract negotiated with sim-farm.
- KR1.2: Concrete streaming constraints are surfaced to DE in writing before DE's ADR lands: latency budget, ordering needs, join shape, late-data tolerance.
- KR1.3: Validation failure-mode shapes are surfaced to sim-farm in writing: schema drift, late data, type errors, duplicate keys, etc., so sim-farm's failure-mode list is grounded in real expectations.

**Tasks**
- [x] Define the `fact_sign_up.parquet` input schema (columns, types, primary key, expected cardinality) — owner: teams/application/resink-core
  - Schema written in § Plan KR1.1.a: 8 columns, `event_id` PK, ~10K rows/day expected.
- [x] Propose dim-table outputs with names, columns, SCD strategy or partitioning approach — owner: teams/application/resink-core
  - Three dim tables proposed in § Plan KR1.1.b: `dim_user_signup` (SCD T1), `dim_signup_funnel_daily` (daily-partition append), `dim_device_user_link` (SCD T1).
- [x] Surface streaming constraints to DE (latency, ordering, joins, late-data) by mid-loop — owner: teams/application/resink-core
  - Surfaced in § Plan "Streaming constraints surfaced to DE"; consumed by `board/decisions/2026-05-09-002-streaming-engine-choice.md` (the ADR's Context section explicitly cites this team's KR1.2 surfaced constraints).
- [x] Surface validation failure-mode shapes to sim-farm by mid-loop — owner: teams/application/resink-core
  - Surfaced in § Plan "Validation failure-mode shapes surfaced to sim-farm"; sim-farm's contract reflects them.
- [x] Surface deployment shape (where pipeline runs, liveness signals) to SRE by mid-loop — owner: teams/application/resink-core
  - Surfaced in § Plan "Deployment shape surfaced to SRE". SRE's runbook drafting was deferred this loop; the inputs are captured for next loop.
- [x] Write the Training → Serving plan document, integrating the engine ADR (KR1.1.c) and deployment ADR (KR1.1.d) once they land — owner: teams/application/resink-core
  - Plan written as a § Plan section in this OKR; ADR-002 (engine) and ADR-003 (deployment) referenced by path.

## Cross-team asks

- **From teams/platform/data-engineering, by mid-loop:** engine-choice ADR landed with `status: active`. Without it, KR1.1.c is a stub.
- **From teams/platform/devops, by mid-loop:** deployment-target ADR landed with `status: active`. Without it, KR1.1.d is a stub.
- **From teams/application/sim-farm, by mid-loop:** validation contract proposal. Without it, KR1.1.e cannot be finalized.

## Risks

- Any of the three cross-team asks slipping past mid-loop pushes KR1.1's affected sub-section to a stub. Mitigation: write the plan section-by-section as inputs arrive; do not block the unaffected sections on the missing inputs.
- The plan is paper-only this loop, so estimation of feasibility is limited. Mitigation: include a "first build slice" subsection naming what the next loop's smallest committable build chunk would look like, so next-loop planning has a concrete starting point.

## Out of scope this loop

- Building any pipeline code; this loop is plan-only per the CEO brief.
- Fact-table types other than `fact_sign_up.parquet`.
- Customer-facing Training experience UX; this loop scopes the engine + data plan only.

## Plan: fact_sign_up.parquet Training → Serving (KR1.1)

### Input schema (KR1.1.a)

`fact_sign_up.parquet` columns:
- `event_id` (UUID, primary key)
- `user_id` (string)
- `device_id` (string, nullable)
- `signup_method` (enum: `email | google | apple | wallet`)
- `country_code` (ISO-2)
- `timestamp` (UTC, ms-precision)
- `client_ip` (string, nullable)
- `referrer` (string, nullable)

Expected cardinality: ~10K rows/day in test data. Primary key uniqueness on `event_id` is invariant.

### Dim-table outputs (KR1.1.b)

Three dim tables proposed:

- **`dim_user_signup`** — one row per user. Columns: `user_id`, `first_signup_at`, `last_signup_at`, `signup_method_first`, `country_code_current`. Strategy: SCD Type 1 (overwrite) for the first cut; revisit Type 2 if customer requirements emerge.
- **`dim_signup_funnel_daily`** — daily aggregate. Columns: `dt`, `country_code`, `signup_method`, `count`. Strategy: daily-partition append.
- **`dim_device_user_link`** — many-to-many bridge. Columns: `device_id`, `user_id`, `first_seen_at`, `last_seen_at`. Strategy: SCD Type 1.

### Streaming consumption pattern (KR1.1.c)

Per `board/decisions/2026-05-09-002-streaming-engine-choice.md`: Spark Structured Streaming. Consumer reads `fact_sign_up.parquet` from a watched directory, processes per-microbatch, and writes dim-table updates as idempotent merges to Delta-style storage.

### Deployment pattern (KR1.1.d)

Per `board/decisions/2026-05-09-003-deployment-target.md`: minikube. The pipeline runs as a single Spark job in a k8s pod; outputs land in a persistent volume claim. Manifests live alongside the pipeline code in resink-core's product repo (path TBD when first build slice lands next loop).

### Validation contract (KR1.1.e)

Per sim-farm's contract (in `teams/application/sim-farm/okrs/2026-05-10-2227-001-team-okr.md` § Validation contract):
- Schema conformance for all three dim tables.
- Idempotency: re-running on the same input produces identical dim-table state.
- Row-count invariants per dim table (see sim-farm's contract for the exact bounds).
- Failure modes covered: schema drift, late data, duplicate `event_id`, missing `user_id`, out-of-range `timestamp`, null in required fields, unknown `country_code`.

### First build slice (next loop)

The smallest committable build chunk for next loop:
- Spark job that reads one parquet file (no file-watcher yet); writes only `dim_user_signup` with idempotency.
- Deployed to minikube via a single Job manifest.
- Sim-farm validates schema conformance + idempotency only.

This excludes streaming (file-watcher comes after); the other two dim tables; cross-loop SCD strategy; CI wiring.

### Streaming constraints surfaced to DE (KR1.2)

For DE's engine ADR:
- **Latency:** sub-minute is acceptable; sub-second is not needed.
- **Ordering:** per-key (`user_id`) ordering preferred but not required for first cut.
- **Joins:** stream-table only (lookup against existing dim tables); no stream-stream joins this loop.
- **Late data:** handle out-of-order events up to 1 hour; later events go to a dead-letter table.
- **Exactly-once:** required for `dim_user_signup`; at-least-once acceptable for `dim_signup_funnel_daily`.

### Validation failure-mode shapes surfaced to sim-farm (KR1.3)

For sim-farm's failure-mode list:
- Schema drift (extra column, missing column, type change).
- Duplicate `event_id` (should be deduplicated, not double-counted).
- Late data (`timestamp` older than current watermark).
- Missing `user_id` (lands in dead-letter, not in `dim_user_signup`).
- Out-of-range `timestamp` (future-dated, pre-epoch).
- Null in required field (e.g., `signup_method is null`).
- Unknown `country_code` (not ISO-2).

### Deployment shape surfaced to SRE

For SRE's runbook draft:
- Pipeline runs as a single Spark job in a minikube pod.
- Liveness signal: pod ready + Spark driver UI reachable on its k8s service.
- Validation-failure detection: sim-farm's assertions are run as a separate post-job step; failures surface as a Job exit code + a structured log line.
