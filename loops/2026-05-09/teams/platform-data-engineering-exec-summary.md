---
layout: default
title: platform-data-engineering Exec Summary — 2026-05-09
nav_exclude: true
render_with_liquid: false
date: 2026-05-09
status: active
type: exec-summary
loop: 2026-05-09
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/data-engineering
  date: 2026-05-09
  status: active
  loop: 2026-05-09
  links: parent: teams/platform/data-engineering/okrs/2026-05-09-team-okr.md
-->
# Data Engineering Exec Summary — 2026-05-09

## What we shipped

- Streaming engine ADR landed: [2026-05-09-002-streaming-engine-choice.md](../../../board/decisions/2026-05-09-002-streaming-engine-choice.md). Recommends Spark Structured Streaming for the first `fact_sign_up.parquet` demo, with explicit re-open conditions if a future demo surfaces a constraint Spark cannot meet.
- Spark vs. Flink comparison from 2026-05-08 refined inside the ADR using resink-core's surfaced constraints (sub-minute latency acceptable, per-key ordering, stream-table joins only, late-data 1h tolerance, exactly-once for `dim_user_signup`).

## What we didn't ship and why

- KR2.1 (shared primitive contract) deferred to next loop — scoped out of this loop's critical-path build per CEO scope decision. The engine ADR is sufficient for resink-core's plan to integrate; the dedicated primitive contract becomes the first item in next loop's DE OKR.

## Surprises

- Resink-core surfaced their streaming constraints inside their OKR plan addendum *before* this team finalized the engine ADR. That changed the recommendation from speculative ("the demo probably doesn't need Flink") to defensible ("here are the explicit constraints from the consumer; Spark satisfies all of them"). Worth replicating: have application teams write down their constraints before platform teams commit ADRs.

## Asks

- None this loop. Engine ADR landed cleanly; the deferred primitive-contract work has no CEO blockers and rolls into next-loop planning.

## Metrics

- KR1.1 (engine ADR with recommendation): met — `status: active`.
- KR1.2 (one-page comparison appendix in the ADR): met — comparison covers latency, ordering, joins, operational footprint as scoped to the demo.
- KR2.1 (shared primitive contract): deferred to next loop.
