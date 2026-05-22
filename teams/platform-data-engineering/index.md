---
layout: default
title: "Team: platform-data-engineering"
parent: Teams
has_children: true
nav_order: 4
---
{% raw %}
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

## Contracts

- [2026-05-10-kafka-ingress](contracts/2026-05-10-kafka-ingress.html)
- [2026-05-16-in-memory-event-source](contracts/2026-05-16-in-memory-event-source.html)

## Conventions

- [dim-schema-json](conventions/dim-schema-json.html)
- [duckdb](conventions/duckdb.html)
{% endraw %}
