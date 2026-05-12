---
layout: default
title: "Team: application-resink-core"
parent: "Teams"
render_with_liquid: false
---
# Team: application-resink-core

## Mission
Deliver the customer-facing realtime data processing product — both the training session (co-design dim tables from sample fact parquets) and the serving session (run validated pipelines on production data).

## Owned products
- Training experience — the agent + UI flow that consumes `fact_*.parquet` and proposes dim-table designs.
- Serving experience — the deployed processing services that run validated pipelines on production data.
- Customer-facing CLI / API for both phases.

## Interfaces
- **Consumes:** DE transformation primitives; DevOps deployment paths; AE agent libraries.
- **Produces:** validation requests for Sim Farm (`teams/application/sim-farm/`); deployable pipelines.

## Success metrics
- End-to-end demo: customer provides 3 fact parquets → system proposes a dim-table design → user approves → serving runs on a test stream.
- Time from "customer provides data" to "first validated pipeline" trends down per loop.

## Decision rights
- Decides unilaterally on: product UX; default tech choices for customer-facing pipelines.
- Must escalate: cross-tenant data handling decisions; commitments to specific customers.

## Out of scope
- Sim Farm internals.
- Agent infra (AE owns).

## Executive summaries

- [2026-06-06](../../loops/2026-06-06/teams/application-resink-core-exec-summary.html)
- [2026-05-30](../../loops/2026-05-30/teams/application-resink-core-exec-summary.html)
- [2026-05-23](../../loops/2026-05-23/teams/application-resink-core-exec-summary.html)
- [2026-05-16](../../loops/2026-05-16/teams/application-resink-core-exec-summary.html)
- [2026-05-10](../../loops/2026-05-10/teams/application-resink-core-exec-summary.html)
- [2026-05-09](../../loops/2026-05-09/teams/application-resink-core-exec-summary.html)

## OKRs

- [2026-06-13](../../loops/2026-06-13/teams/application-resink-core-okr.html)
- [2026-06-06](../../loops/2026-06-06/teams/application-resink-core-okr.html)
- [2026-05-30](../../loops/2026-05-30/teams/application-resink-core-okr.html)
- [2026-05-23](../../loops/2026-05-23/teams/application-resink-core-okr.html)
- [2026-05-16](../../loops/2026-05-16/teams/application-resink-core-okr.html)
- [2026-05-10](../../loops/2026-05-10/teams/application-resink-core-okr.html)
- [2026-05-09](../../loops/2026-05-09/teams/application-resink-core-okr.html)
