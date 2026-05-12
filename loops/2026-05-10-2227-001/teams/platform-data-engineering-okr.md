---
layout: default
title: platform-data-engineering OKR — 2026-05-10-2227-001
date: 2026-05-10
status: active
type: okr
loop: 2026-05-10-2227-001
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/data-engineering
  date: 2026-05-10
  status: active
  loop: 2026-05-10-2227-001
  links: parent: board/okrs/2026-05-10-2227-001-ceo-brief.md
-->
# Data Engineering OKR — 2026-05-09

## Context

Carryover from 2026-05-08: the Spark Structured Streaming vs. Flink comparison is drafted; no recommendation is committed. The CEO brief confirms `fact_sign_up.parquet` as the first demo target, which gives the engine choice concrete grounding constraints. This loop, DE commits a recommendation, lands the ADR, and supports resink-core's first product-shaped OKR with whatever streaming primitives the chosen engine implies.

## Objectives

### O1: Commit a streaming engine choice and land the ADR

Why it matters: This decision is on the critical path for resink-core. CEO brief KR1.1 cannot land without it, and resink-core's KR1.1 (Training → Serving plan) cannot name a consumption pattern without it.

**Key results**
- KR1.1: An ADR exists in `board/decisions/` with `status: active`, naming exactly one engine (Spark Structured Streaming or Flink) and giving a one-paragraph recommendation grounded in `fact_sign_up.parquet` constraints.
- KR1.2: The ADR includes a one-page comparison appendix (refined from the 2026-05-08 draft) covering latency, ordering, joins, and operational footprint as they apply to a single fact-table demo.

**Tasks**
- [x] Refine Spark vs. Flink comparison from 2026-05-08 with `fact_sign_up.parquet` constraints — owner: teams/platform/data-engineering
  - Comparison refined inside the ADR; resink-core's surfaced constraints (sub-minute latency acceptable, per-key ordering, stream-table joins only, late-data 1h tolerance, exactly-once for `dim_user_signup`) made the recommendation defensible without needing Flink's differentiated capabilities.
- [x] Write the ADR with a one-paragraph recommendation and the comparison appendix — owner: teams/platform/data-engineering
  - Written at `board/decisions/2026-05-09-002-streaming-engine-choice.md`. Recommendation: Spark Structured Streaming for the first demo.
- [x] Submit ADR to CEO for approval and land with `status: active` — owner: teams/platform/data-engineering
  - Landed with `status: active`.

### O2: Define the first shared streaming primitive for `fact_sign_up.parquet`

Why it matters: Carryover from 2026-05-08 — DE identified a candidate but did not define the contract. Defining it now lets resink-core depend on a stable interface rather than re-inventing it inside their pipeline.

**Key results**
- KR2.1: A documented contract for the first shared primitive (e.g., the fact-stream consumer pattern, ordering guarantees, partitioning convention) lives at a documented path under `teams/platform/data-engineering/`, referenced by the engine ADR.

**Tasks**
- [ ] Promote the 2026-05-08 candidate primitive into a written contract scoped to `fact_sign_up.parquet` — owner: teams/platform/data-engineering
  - > deferred: scoped out of this loop's critical-path build per CEO scope decision. Engine ADR is sufficient for resink-core's plan; the dedicated primitive contract rolls to next loop.
- [ ] Cross-link the primitive contract from the engine ADR — owner: teams/platform/data-engineering
  - > deferred: depends on the contract above.

## Cross-team asks

- **From teams/application/resink-core, by mid-loop:** concrete streaming constraints for the demo (latency budget, join requirements, ordering / late-data needs). Without these, the engine ADR's recommendation is conservative.

## Risks

- If resink-core's constraints are not surfaced by mid-loop, the ADR may need a follow-up revision next loop. Mitigation: write a recommendation grounded in defensible defaults and explicitly note which inputs would trigger a re-review.

## Out of scope this loop

- Engines beyond Spark Structured Streaming and Flink.
- Multi-fact-table generalization of the shared primitive.
- Production-grade operational tooling for the chosen engine.
