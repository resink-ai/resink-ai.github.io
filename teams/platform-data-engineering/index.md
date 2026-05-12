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

- [2026-06-06](../../loops/2026-06-06/teams/platform-data-engineering-exec-summary.html)
- [2026-05-30](../../loops/2026-05-30/teams/platform-data-engineering-exec-summary.html)
- [2026-05-23](../../loops/2026-05-23/teams/platform-data-engineering-exec-summary.html)
- [2026-05-16](../../loops/2026-05-16/teams/platform-data-engineering-exec-summary.html)
- [2026-05-10](../../loops/2026-05-10/teams/platform-data-engineering-exec-summary.html)
- [2026-05-09](../../loops/2026-05-09/teams/platform-data-engineering-exec-summary.html)
- [2026-05-08](../../loops/2026-05-08/teams/platform-data-engineering-exec-summary.html)

## OKRs

- [2026-06-13](../../loops/2026-06-13/teams/platform-data-engineering-okr.html)
- [2026-05-23](../../loops/2026-05-23/teams/platform-data-engineering-okr.html)
- [2026-05-16](../../loops/2026-05-16/teams/platform-data-engineering-okr.html)
- [2026-05-10](../../loops/2026-05-10/teams/platform-data-engineering-okr.html)
- [2026-05-09](../../loops/2026-05-09/teams/platform-data-engineering-okr.html)
- [2026-05-08](../../loops/2026-05-08/teams/platform-data-engineering-okr.html)

## Contracts

- [2026-05-10-kafka-ingress](contracts/2026-05-10-kafka-ingress.html)
- [2026-05-16-in-memory-event-source](contracts/2026-05-16-in-memory-event-source.html)

## Conventions

- [duckdb](conventions/duckdb.html)
