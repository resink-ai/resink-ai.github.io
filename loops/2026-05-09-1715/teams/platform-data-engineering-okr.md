---
layout: default
title: platform-data-engineering OKR — 2026-05-09-1715
date: 2026-05-08
status: active
type: okr
loop: 2026-05-09-1715
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/data-engineering
  date: 2026-05-08
  status: active
  loop: 2026-05-09-1715
  links: parent: board/okrs/2026-05-09-1715-ceo-brief.md
-->
# Data Engineering Team OKR — 2026-05-08

## Context

DE is bootstrapping. No product code ships this loop, so the most valuable work DE can do is make the decisions that unblock the realtime-pipeline team in the next loop. The company has already committed to `fact_sign_up.parquet` as the first fact-table type; DE's job is to pick the engine that will process it and identify the first shared transformation primitive that realtime-pipeline will need on day one.

## Objectives

### O1: Pick the realtime engine for the first demo

Why it matters: The realtime-pipeline team's first OKR (next loop) cannot be written without an engine pinned. Every tool choice — connectors, serialization format, deployment target — flows from whether the engine is Spark Structured Streaming or Flink. Without a documented decision with rationale, the next loop begins in design churn rather than implementation, and DE's first real deliverable slips by a full loop. A lightweight ADR this loop eliminates that entire class of delay.

**Key results**
- KR1.1: ADR drafted comparing Spark Structured Streaming vs. Flink, with a clear recommendation and rationale documented.
- KR1.2: One shared transformation primitive identified that the realtime-pipeline team will need first when working on `fact_sign_up.parquet`.

**Tasks**
- [ ] Draft engine-choice ADR (Spark Structured Streaming vs. Flink) — owner: teams/platform/data-engineering
- [ ] Identify the first shared transformation primitive for `fact_sign_up.parquet` — owner: teams/platform/data-engineering

## Risks

- The ADR may need a follow-up revision once `realtime-pipeline` joins the loop and surfaces constraints (throughput targets, connector availability, operator familiarity) that DE does not have visibility into today.

## Out of scope this loop

- Implementing the engine (no product code this loop).
- Building or publishing transformation primitives.
- Any work on fact types beyond `fact_sign_up.parquet`.
- Application teams' OKRs (`resink-core`, `sim-farm`) — deferred to next loop.
