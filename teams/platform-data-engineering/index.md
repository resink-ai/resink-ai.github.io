---
layout: default
title: "Team: platform-data-engineering"
---
# Team: platform-data-engineering

## Mission

Move and transform data so analytics, ML, and the realtime pipeline product can rely on it.

## Owned products

- Pipeline runtime conventions (Spark / Flink / Kafka / Airflow / dbt).
- Reusable transformation libraries.
- Schema registry conventions for dim tables.

## Interfaces

**Consumes:** IaC from DevOps.
**Produces:** transformation primitives consumed by `teams/application/resink-core/`.

## Success metrics

- The realtime pipeline ships using DE primitives (no forks).
- A new fact-table type can be onboarded without forking DE primitives.

## Decision rights

- Decides unilaterally on: which transformation engines are first-class.
- Must escalate: cross-team schema conventions.

## Out of scope

- Product UI.
- Agent runtime.
- Pipeline application logic itself.

## Executive summaries

- [2026-05-12-1826](../../loops/2026-05-12-1826/teams/platform-data-engineering-exec-summary.html)
- [2026-05-12-1254](../../loops/2026-05-12-1254/teams/platform-data-engineering-exec-summary.html)
- [2026-05-12-0645](../../loops/2026-05-12-0645/teams/platform-data-engineering-exec-summary.html)
- [2026-05-11-2153](../../loops/2026-05-11-2153/teams/platform-data-engineering-exec-summary.html)
- [2026-05-11-1631](../../loops/2026-05-11-1631/teams/platform-data-engineering-exec-summary.html)
- [2026-05-11-1302](../../loops/2026-05-11-1302/teams/platform-data-engineering-exec-summary.html)
- [2026-05-11-1113](../../loops/2026-05-11-1113/teams/platform-data-engineering-exec-summary.html)
- [2026-05-11-0958](../../loops/2026-05-11-0958/teams/platform-data-engineering-exec-summary.html)
- [2026-05-10-2227-002](../../loops/2026-05-10-2227-002/teams/platform-data-engineering-exec-summary.html)
- [2026-05-10-2227-001](../../loops/2026-05-10-2227-001/teams/platform-data-engineering-exec-summary.html)
- [2026-05-09-1715](../../loops/2026-05-09-1715/teams/platform-data-engineering-exec-summary.html)

## OKRs

- [2026-05-11-2153](../../loops/2026-05-11-2153/teams/platform-data-engineering-okr.html)
- [2026-05-11-1113](../../loops/2026-05-11-1113/teams/platform-data-engineering-okr.html)
- [2026-05-11-0958](../../loops/2026-05-11-0958/teams/platform-data-engineering-okr.html)
- [2026-05-10-2227-002](../../loops/2026-05-10-2227-002/teams/platform-data-engineering-okr.html)
- [2026-05-10-2227-001](../../loops/2026-05-10-2227-001/teams/platform-data-engineering-okr.html)
- [2026-05-09-1715](../../loops/2026-05-09-1715/teams/platform-data-engineering-okr.html)

## Contracts

- [2026-05-10-kafka-ingress](contracts/2026-05-10-kafka-ingress.html)
- [2026-05-16-in-memory-event-source](contracts/2026-05-16-in-memory-event-source.html)

## Conventions

- [dim-schema-json](conventions/dim-schema-json.html)
- [duckdb](conventions/duckdb.html)
