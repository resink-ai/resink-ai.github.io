---
layout: default
title: "Team: application-sim-farm"
---
# Team: application-sim-farm

## Mission
Internal product — execute candidate nanofab DAGs against synthetic or shadowed traffic, diff outputs against a known-good reference, and issue a verdict trusted by both the training validator and the runtime coordinator. Synthetic-data generation is **not** owned here; it lives in training (sub-project #2) per the 2026-05-10 nanofab pivot.

## Owned products
- Mode-A batch diff engine — DuckDB-based equivalence check between a runtime output parquet and a known-good fixture parquet. Mode-A batch diff shipped 2026-05-16 against the MVP loop; Modes B/C remain scoped per Sim Farm spec §1.3.
- Mode-B per-node shadow differ — streaming Rust differ + per-supervisor sidecar (post-MVP, Sim Farm spec §5.2 / §4.6).
- Mode-C DAG blue/green warmup differ — DuckDB-based equivalence check over Kafka offset ranges (post-MVP, Sim Farm spec §5.3).
- Verdict-format contract — the typed schema both training (gate stage 3) and the runtime coordinator (hot-swap) consume.
- Coordinator (`nanofab-simfarm-coordinator`) — control-plane scheduler + verdict aggregator (post-MVP, Sim Farm spec §4.1).

## Interfaces
- **Consumes:** runtime output parquets and fixture parquets from `teams/application/resink-core/`; the supervisor binary in `--mode=sim` is the integration substrate (Sim Farm spec §1.3, runtime spec §8.4) — every batch sim run reuses `nanofab-supervisor --mode=sim` rather than a parallel engine.
- **Produces:** verdict JSON files (and, post-MVP, Iceberg verdict tables) consumed by the `make mvp-loop` driver (this loop), and later by the training pipeline gate stage 3 and the runtime coordinator's hot-swap state machine.

## Success metrics
- The MVP closed loop closes: `make mvp-loop` exits 0 on the green path with `verdict=pass`. (2026-05-16 loop.)
- A forced red-path run produces a verdict file an operator can read without grepping `trace.jsonl`.
- Post-MVP: Modes B and C wire into the runtime coordinator's hot-swap path without verdict-schema rewrites (additive extension only).

## Decision rights
- Decides unilaterally on: what counts as a "valid" pipeline output (the verdict-format contract is sim-farm-owned per Sim Farm spec §1.3); diff-engine algorithmic choices; Mode-A vs B vs C scope sequencing.
- Must escalate: production deployment of Sim Farm itself (DevOps); cross-team verdict-schema breaking changes (CEO brief / ADR).

## Out of scope
- Synthetic-data generation — moved to training under the 2026-05-10 pivot. Sim Farm consumes `gen_data.py` from the workspace; it does not author it.
- Customer-facing surfaces — verdicts are surfaced through the product UX, not by Sim Farm directly.
- Production deployment of Sim Farm (DevOps).
- General SQL execution — DuckDB is the diff substrate, not a query engine for callers.

## Executive summaries

- [2026-05-12-1826](../../loops/2026-05-12-1826/teams/application-sim-farm-exec-summary.html)
- [2026-05-12-1254](../../loops/2026-05-12-1254/teams/application-sim-farm-exec-summary.html)
- [2026-05-12-0645](../../loops/2026-05-12-0645/teams/application-sim-farm-exec-summary.html)
- [2026-05-11-2153](../../loops/2026-05-11-2153/teams/application-sim-farm-exec-summary.html)
- [2026-05-11-1631](../../loops/2026-05-11-1631/teams/application-sim-farm-exec-summary.html)
- [2026-05-11-1302](../../loops/2026-05-11-1302/teams/application-sim-farm-exec-summary.html)
- [2026-05-11-1113](../../loops/2026-05-11-1113/teams/application-sim-farm-exec-summary.html)
- [2026-05-11-0958](../../loops/2026-05-11-0958/teams/application-sim-farm-exec-summary.html)
- [2026-05-10-2227-002](../../loops/2026-05-10-2227-002/teams/application-sim-farm-exec-summary.html)
- [2026-05-10-2227-001](../../loops/2026-05-10-2227-001/teams/application-sim-farm-exec-summary.html)

## OKRs

- [2026-05-11-1631](../../loops/2026-05-11-1631/teams/application-sim-farm-okr.html)
- [2026-05-11-1113](../../loops/2026-05-11-1113/teams/application-sim-farm-okr.html)
- [2026-05-11-0958](../../loops/2026-05-11-0958/teams/application-sim-farm-okr.html)
- [2026-05-10-2227-002](../../loops/2026-05-10-2227-002/teams/application-sim-farm-okr.html)
- [2026-05-10-2227-001](../../loops/2026-05-10-2227-001/teams/application-sim-farm-okr.html)

## Contracts

- [2026-05-16-mvp-loop-verdict](contracts/2026-05-16-mvp-loop-verdict.html)
