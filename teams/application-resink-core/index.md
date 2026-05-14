---
layout: default
title: "Team: application-resink-core"
---
{% raw %}
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

- [2026-05-13-1844](../../loops/2026-05-13-1844/teams/application-resink-core-exec-summary.html)
- [2026-05-13-1022](../../loops/2026-05-13-1022/teams/application-resink-core-exec-summary.html)
- [2026-05-12-1826](../../loops/2026-05-12-1826/teams/application-resink-core-exec-summary.html)
- [2026-05-12-1254](../../loops/2026-05-12-1254/teams/application-resink-core-exec-summary.html)
- [2026-05-12-0645](../../loops/2026-05-12-0645/teams/application-resink-core-exec-summary.html)
- [2026-05-11-2153](../../loops/2026-05-11-2153/teams/application-resink-core-exec-summary.html)
- [2026-05-11-1631](../../loops/2026-05-11-1631/teams/application-resink-core-exec-summary.html)
- [2026-05-11-1302](../../loops/2026-05-11-1302/teams/application-resink-core-exec-summary.html)
- [2026-05-11-1113](../../loops/2026-05-11-1113/teams/application-resink-core-exec-summary.html)
- [2026-05-11-0958](../../loops/2026-05-11-0958/teams/application-resink-core-exec-summary.html)
- [2026-05-10-2227-002](../../loops/2026-05-10-2227-002/teams/application-resink-core-exec-summary.html)
- [2026-05-10-2227-001](../../loops/2026-05-10-2227-001/teams/application-resink-core-exec-summary.html)

## OKRs

- [2026-05-13-1844](../../loops/2026-05-13-1844/teams/application-resink-core-okr.html)
- [2026-05-13-1022](../../loops/2026-05-13-1022/teams/application-resink-core-okr.html)
- [2026-05-12-1826](../../loops/2026-05-12-1826/teams/application-resink-core-okr.html)
- [2026-05-12-1254](../../loops/2026-05-12-1254/teams/application-resink-core-okr.html)
- [2026-05-12-0645](../../loops/2026-05-12-0645/teams/application-resink-core-okr.html)
- [2026-05-11-2153](../../loops/2026-05-11-2153/teams/application-resink-core-okr.html)
- [2026-05-11-1631](../../loops/2026-05-11-1631/teams/application-resink-core-okr.html)
- [2026-05-11-1302](../../loops/2026-05-11-1302/teams/application-resink-core-okr.html)
- [2026-05-11-1113](../../loops/2026-05-11-1113/teams/application-resink-core-okr.html)
- [2026-05-11-0958](../../loops/2026-05-11-0958/teams/application-resink-core-okr.html)
- [2026-05-10-2227-002](../../loops/2026-05-10-2227-002/teams/application-resink-core-okr.html)
- [2026-05-10-2227-001](../../loops/2026-05-10-2227-001/teams/application-resink-core-okr.html)
{% endraw %}
