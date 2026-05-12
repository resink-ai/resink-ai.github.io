---
layout: default
title: platform-sre OKR — 2026-05-08
date: 2026-05-08
status: active
type: okr
loop: 2026-05-08
owner: teams/platform/sre
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/sre
  date: 2026-05-08
  status: active
  loop: 2026-05-08
  links: parent: board/okrs/2026-05-08-ceo-brief.md
-->
# SRE Team OKR — 2026-05-08

## Context

SRE is bootstrapping. Most observability work depends on a running service to instrument, and no product code ships this loop. However, SRE can pre-commit to the minimal metric list and runbook target that the first pipeline demo will need on day one, so that the first demo is not built in an observability vacuum and does not ship with a promise to "add SLOs later."

## Objectives

### O1: Define what observability the first demo pipeline needs

Why it matters: Production-grade data pipelines need SLOs and a runbook from day one, not as an afterthought. For the first demo the set can be small — three to five metrics and one runbook is a realistic and defensible minimum — but the set cannot be zero. If SRE waits until the pipeline is running to define metrics, the first demo ships without any way to detect that it has silently failed, which makes the demo untrustworthy as an internal validation artifact. Defining the list now also gives the realtime-pipeline team concrete emission targets to build toward.

**Key results**
- KR1.1: A list of 3–5 metrics the first demo pipeline must emit, documented and agreed upon.
- KR1.2: One runbook identified as the first to write; the "demo pipeline failed validation" runbook is the strong candidate.

**Tasks**
- [ ] Draft the metric list for the first demo pipeline — owner: teams/platform/sre
- [ ] Identify and propose the first runbook target — owner: teams/platform/sre

## Risks

- The metric list may need revision once realtime-pipeline picks an engine, since the specific metric names and instrumentation APIs differ between Spark Structured Streaming and Flink.

## Out of scope this loop

- Actual dashboards or alerting configuration.
- Pagerduty or on-call rotation setup.
- Chaos scripts or fault-injection testing.
- The runbook content itself (only the runbook target is identified this loop).
- Any work for application teams (`resink-core`, `sim-farm`) — deferred to next loop.
