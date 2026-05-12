---
layout: default
title: "ADR 2026-05-09-002: streaming engine choice"
parent: "Decisions (ADRs)"
render_with_liquid: false
date: 2026-05-09
status: archived
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-09
  status: archived
  superseded_by: 2026-05-10-001-nanofab-runtime-is-rust
  decision: Adopt Spark Structured Streaming as the streaming engine for the first fact_sign_up.parquet demo; revisit if a future demo surfaces a constraint Spark cannot meet.
-->
# ADR 2026-05-09-002: Streaming engine choice for the first demo

## Context

Carryover from loop 2026-05-09-1715: DE drafted a Spark Structured Streaming vs. Flink comparison but did not commit a recommendation. The CEO brief for 2026-05-09 confirms `fact_sign_up.parquet` as the first demo target, which gives the engine choice concrete grounding constraints. Resink-core's surfaced streaming requirements (in `teams/application/resink-core/okrs/2026-05-10-2227-001-team-okr.md` § Plan, KR1.2) are: sub-minute latency acceptable, per-key ordering preferred but not required, stream-table joins only, late-data tolerance up to 1 hour with dead-letter handling, exactly-once for `dim_user_signup`.

## Decision

Adopt **Spark Structured Streaming** for the first `fact_sign_up.parquet` demo pipeline. The choice is for the demo and the next 1–2 loops; a future demo with constraints Spark cannot meet (e.g., sub-second latency on richer windowed joins) re-opens this ADR.

## Alternatives considered

- **A: Apache Flink** — rejected for the first demo because resink-core's surfaced constraints (KR1.2) do not require Flink's differentiated capabilities (sub-second latency, complex windowed stream-stream joins). Flink's higher operational footprint would slow time-to-first-pipeline without delivering value the demo needs. Re-considered when a future demo surfaces a constraint Spark cannot meet.
- **B: Pure Spark batch (no streaming)** — rejected because the vision in `ORG.md` is realtime; batch-first contradicts the purpose of having an engine ADR at all.

## Consequences

- Positive: Lower bootstrap effort (broader team familiarity); ecosystem integration with Delta / Iceberg is well-trodden; deployment to k8s (per `2026-05-09-003-deployment-target.md`) has prior art.
- Negative / costs: Sub-second latency is not available; richer windowed joins require workarounds; if production demos shift toward those constraints, an engine swap is paid for in re-architecture cost.
- Follow-ups required:
  - DE documents the first shared streaming primitive (the Spark fact-stream consumer pattern) per DE OKR KR2.1.
  - Resink-core's pipeline references this ADR in its consumption-pattern section.

## Links

- Triggering retro: [2026-05-08-ceo-retro](../retros/2026-05-09-1715-ceo-retro.md) (DE engine-choice carryover)
- Related ADRs: [2026-05-09-003-deployment-target](2026-05-09-003-deployment-target.md), [2026-05-08-002-tenant-tree-restructure](2026-05-08-002-tenant-tree-restructure.md)
- Surfaced constraints: [resink-core OKR § Plan](../../teams/application/resink-core/okrs/2026-05-10-2227-001-team-okr.md)
