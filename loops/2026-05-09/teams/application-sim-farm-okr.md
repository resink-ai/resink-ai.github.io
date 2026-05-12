---
layout: default
title: application-sim-farm OKR — 2026-05-09
date: 2026-05-09
status: active
type: okr
loop: 2026-05-09
owner: teams/application/sim-farm
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/sim-farm
  date: 2026-05-09
  status: active
  loop: 2026-05-09
  links: parent: board/okrs/2026-05-09-ceo-brief.md
-->
# Sim Farm OKR — 2026-05-09

## Context

Sim Farm's first product-shaped OKR. The CEO brief explicitly scopes sim-farm to validating the resink-core demo pipeline this loop — generation breadth is out of scope. Without a validation contract for resink-core's `fact_sign_up.parquet` plan, resink-core's KR1.1.e cannot land. This OKR delivers that contract as a written artifact and prepares (without yet implementing) the test-data generation approach.

## Objectives

### O1: Define the validation contract for the resink-core demo pipeline

Why it matters: A pipeline without a validation contract is a pipeline that "seems to work" until it doesn't. Defining what "valid output" means before resink-core builds the pipeline ensures the demo is provably correct, not just run-able.

**Key results**
- KR1.1: A documented validation contract exists under `teams/application/sim-farm/` specifying: (a) what "valid output" means for the dim tables produced from `fact_sign_up.parquet` — invariants, row counts, schema conformance, idempotency, (b) the enumerated failure modes sim-farm intends to detect, (c) a sketch of how sim-farm will generate test data to exercise each failure mode (no implementation this loop).
- KR1.2: The validation contract is handed to resink-core by mid-loop so resink-core's KR1.1.e can finalize.
- KR1.3: The failure-mode list (KR1.1.b) is handed to SRE by mid-loop so SRE's runbook covers the right scenarios.

**Tasks**
- [x] Define "valid output" criteria for `fact_sign_up`-derived dim tables — owner: teams/application/sim-farm
  - Defined in § Validation contract KR1.1.a: schema conformance, idempotency, three row-count invariants, referential consistency.
- [x] Enumerate failure modes to detect — owner: teams/application/sim-farm
  - Enumerated in § Validation contract KR1.1.b: 7 failure modes covering schema drift, duplicate event_ids, late data, missing user_ids, out-of-range timestamps, nulls in required fields, unknown country codes.
- [x] Sketch the test-data generation approach per failure mode (paper sketch only) — owner: teams/application/sim-farm
  - Sketched in § Validation contract KR1.1.c: baseline + per-mode mutators + per-variant assertion files. Implementation deferred to a future loop after resink-core's first build slice.
- [x] Hand the contract to resink-core (KR1.2) and the failure-mode list to SRE (KR1.3) — owner: teams/application/sim-farm
  - Hand-offs documented in § Validation contract "Hand-offs". Resink-core's plan KR1.1.e references this contract; SRE's runbook drafting was deferred this loop but the failure-mode list is in place for next loop.

## Cross-team asks

- **From teams/application/resink-core, by mid-loop:** proposed dim-table shapes (names, columns, SCD/partitioning). Without these, "valid output" criteria are generic rather than table-specific.

## Risks

- If resink-core's dim-table proposal slips past mid-loop, sim-farm's KR1.1.a falls back to schema-conformance-only criteria. Mitigation: write the generic invariants now and add table-specific ones as resink-core's shape arrives.

## Out of scope this loop

- Building any test-data generator code (sketch only).
- Validation for fact-table types other than `fact_sign_up.parquet`.
- Broader sim-farm capability (generation diversity, replay, chaos injection); first commitment is single-pipeline validation.

## Validation contract: fact_sign_up.parquet pipeline (KR1.1)

### "Valid output" criteria (KR1.1.a)

For each of resink-core's three dim tables (per `teams/application/resink-core/okrs/2026-05-09-team-okr.md` § Plan):

- **Schema conformance.** All columns present with correct types per resink-core's plan.
- **Idempotency.** Running the pipeline N times on the same input produces dim-table state identical to a single run (no duplicate rows, no drift in update timestamps where they should be deterministic).
- **Row-count invariants:**
  - `dim_user_signup`: row count ≤ distinct `user_id`s in input.
  - `dim_signup_funnel_daily`: row count ≤ distinct `(dt, country_code, signup_method)` tuples in input.
  - `dim_device_user_link`: row count ≤ distinct `(device_id, user_id)` pairs in input where `device_id is not null`.
- **Referential consistency.** Every `user_id` in `dim_user_signup` appears in at least one row of input.

### Failure modes to detect (KR1.1.b)

Sim-farm will generate test data that exercises each:

- **Schema drift** — extra column, missing column, type change.
- **Duplicate `event_id`** — same UUID appears twice; pipeline should deduplicate.
- **Late data** — `timestamp` older than the current pipeline watermark.
- **Missing `user_id`** — `user_id is null`; should land in dead-letter, not in `dim_user_signup`.
- **Out-of-range `timestamp`** — future-dated or pre-epoch.
- **Null in required field** — e.g. `signup_method is null`.
- **Unknown `country_code`** — value is not a valid ISO-2 code.

### Test-data generation sketch (KR1.1.c, paper only)

The actual generator is not implemented this loop. Sketch:

- A "happy path" baseline parquet file (10 rows, no failures) serves as the control.
- One mutator per failure mode produces a variant of the baseline (e.g. `with_duplicate_event_id`, `with_late_data`, `with_unknown_country`).
- Sim-farm runs the pipeline against each variant; expected outcomes per variant are encoded in a per-variant assertion file.
- All of the above lives in a future sim-farm product repo; this loop documents the shape only.

### Hand-offs

- **To resink-core (KR1.2):** this contract document — referenced from resink-core's OKR § Plan KR1.1.e.
- **To SRE (KR1.3):** the failure-mode list above — drives the scenarios SRE's "demo pipeline failed validation" runbook will cover.
