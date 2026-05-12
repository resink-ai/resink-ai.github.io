---
layout: default
title: application-sim-farm Exec Summary — 2026-05-10-2227-001
date: 2026-05-09
status: active
type: exec-summary
loop: 2026-05-10-2227-001
owner: teams/application/sim-farm
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/sim-farm
  date: 2026-05-09
  status: active
  loop: 2026-05-10-2227-001
  links: parent: teams/application/sim-farm/okrs/2026-05-10-2227-001-team-okr.md
-->
# Sim Farm Exec Summary — 2026-05-09

## What we shipped

- Validation contract for the `fact_sign_up.parquet` pipeline written as the § Validation contract section of [our team OKR](../okrs/2026-05-10-2227-001-team-okr.md). Covers: "valid output" criteria (schema conformance, idempotency, three row-count invariants per dim table, referential consistency); seven failure modes to detect (schema drift, duplicate `event_id`, late data, missing `user_id`, out-of-range `timestamp`, null in required fields, unknown `country_code`); test-data generation sketch (baseline + per-mode mutators + per-variant assertion files, paper sketch only).
- Hand-off to resink-core: their plan KR1.1.e references this contract.
- Hand-off to SRE: the failure-mode list grounds SRE's next-loop runbook scope (SRE drafting was deferred this loop per CEO scope decision; the inputs are in place).

## What we didn't ship and why

- Nothing was deferred. All four tasks ticked. The actual generator implementation is correctly scoped out of this loop (the OKR's "Out of scope" was honored).

## Surprises

- The validation contract was sufficient to ground both resink-core's plan and SRE's (deferred) runbook scope at the same time. We had expected to write two adjacent documents (one for resink-core's contract, one for SRE's failure-mode-list); a single contract document with named hand-off sections served both consumers. Worth keeping as a pattern.
- Writing the contract before the pipeline exists clarified two ambiguities in resink-core's dim-table strategies (SCD T1 vs T2 implications for idempotency assertions; row-count bound on `dim_device_user_link` requires the `device_id is not null` qualifier) that would otherwise have surfaced during validation runs.

## Asks

- None this loop. All inputs and consumers are aligned.

## Metrics

- KR1.1 (validation contract document): met — sub-sections (a) "valid output" criteria, (b) failure modes, (c) generation sketch — all written.
- KR1.2 (contract handed to resink-core by mid-loop): met — referenced from resink-core's plan KR1.1.e.
- KR1.3 (failure-mode list handed to SRE by mid-loop): met — handed off in writing; SRE's runbook drafting deferred but inputs are in place.
