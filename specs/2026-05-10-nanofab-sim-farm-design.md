---
layout: default
title: "Spec: 2026-05-10-nanofab-sim-farm-design"
parent: "Design specs"
render_with_liquid: false
---
# Nanofab Sim Farm — Design Spec

**Date:** 2026-05-10
**Status:** Draft, awaiting user review
**Scope:** Sub-project #3 of the resink.ai product family — the verification authority that executes candidate nanofab DAGs against synthetic or shadowed traffic, diffs outputs against a known-good reference, and issues a verdict trusted by both the training validator and the runtime coordinator. Synthetic-data generation lives in training (sub-project #2). Serving/deployment and product UX are separate specs.

## 1. Context & motivation

### 1.1 What this verifies
Three callers depend on Sim Farm verdicts:

| Mode | Caller | When | Inputs | Output |
|---|---|---|---|---|
| **A. Pre-deploy validation** | Training pipeline (gate stage 3) | Before a candidate DAG ships | workspace `sim/gen_data.py` + `coverage_spec.yaml` + DAG manifest | verdict + per-scenario diff report |
| **B. Per-node shadow** | Runtime coordinator (hot swap §6.2) | During shadow phase of per-node swap | live event stream + two write streams (v1, v2) | live diff rate + sample-event diffs |
| **C. DAG blue/green** | Runtime coordinator (hot swap §6.3) | During candidate fleet warmup | candidate fleet's outputs + current fleet's outputs over the same Kafka offset range | equivalence verdict + per-table diff |

Plus a fourth implicit consumer: **drift monitoring** (training spec §7.4) — canary tenants re-trained nightly. That's mode A on a schedule.

### 1.2 What it builds on
The user's prior prototype work established the validation patterns Sim Farm automates:
- DuckDB-based equivalence checking with per-column tolerance, sample-row collection, exclusive-key reporting (`prompts.md` § "Correctness Validation Workflow", `check_consistency.py`).
- ODPS/Spark → DuckDB SQL translation as the reference-query substrate (`prompts.md` cheatsheet).
- Sample-data generation patterns (`gen-data-skill.md`) — consumed by Sim Farm via the workspace's `sim/gen_data.py`, *not* re-implemented here.

### 1.3 Relationship to other sub-projects
- **Sub-project #1 (Nanofab runtime)** — Sim Farm reuses the `nanofab-supervisor` binary in `--mode=sim` (defined by runtime spec §8.4) for every batch sim run; a dedicated sidecar attaches to production supervisors during shadow validation; verdicts are consumed by the runtime coordinator's hot-swap state machine.
- **Sub-project #2 (Training pipeline)** — training writes `sim/gen_data.py` and `coverage_spec.yaml` into the per-tenant workspace; Sim Farm consumes both. Verdict structure is the gate-stage-3 contract.
- **Sub-project #5 (Product UX)** — verdict samples and forensic diffs are surfaced to customers through the product UX, not directly by Sim Farm.

## 2. Decisions summary

| Axis | Choice | Why |
|---|---|---|
| Scope | Execute + diff + verdict only; no data generation | Cleanest boundary; sim-data-gen lives in training where the workspace is owned. |
| Execution engine | Reuse `nanofab-supervisor --mode=sim` on ephemeral k8s Jobs + warm-worker pool for low-latency runs | Bug parity with production; pool absorbs hot-swap latency budget. |
| Diff engine | Hybrid — DuckDB for batch (modes A & C), custom Rust streaming differ for live shadow (mode B) | Operating envelopes differ by 1000×; reuse the prototype's DuckDB patterns where they fit. |
| Live shadow path | Sidecar differ per supervisor pod, Unix-socket write-trace, sampled forensics to Kafka | Lowest-latency correlation; supervisor never blocks on differ. |
| Coverage spec semantics | Three-layer (node coverage + named scenarios + per-table tolerances), per-layer pass/fail | Defense in depth; surfaces *why* a verdict failed, not just *that* it did. |
| Verdict pipeline | Structured per-layer JSON; sync gRPC for batch / streaming heartbeats for live; persisted to Iceberg | Same analytical surface as the runtime; drift monitoring becomes plain SQL. |

## 3. Architecture overview

### 3.1 Three planes

```
                   ┌──────────────────────────────────────────────┐
   CONTROL PLANE   │  Sim Farm coordinator                        │
                   │  - accepts jobs from training & runtime      │
                   │  - schedules sim runs & live-shadow sidecars │
                   │  - aggregates verdicts                       │
                   │  - publishes verdicts to Iceberg             │
                   └────────────────┬─────────────────────────────┘
                                    │ dispatches
                   ┌────────────────▼─────────────────────────────┐
   EXECUTION       │  Sim runners                                 │
   PLANE           │  ┌──────────────────────┐ ┌────────────────┐ │
                   │  │ Batch sim worker     │ │ Live-shadow    │ │
                   │  │ (k8s Job, ephemeral) │ │ sidecar        │ │
                   │  │ supervisor --mode=sim│ │ (per supervisor│ │
                   │  │ + DuckDB ref + diff  │ │  pod, Rust)    │ │
                   │  └──────────────────────┘ └────────────────┘ │
                   │  ┌──────────────────────┐                    │
                   │  │ Warm-worker pool     │                    │
                   │  │ (1-2 pre-warmed sup. │                    │
                   │  │  per region, low-lat)│                    │
                   │  └──────────────────────┘                    │
                   └────────────────┬─────────────────────────────┘
                                    │ writes
                   ┌────────────────▼─────────────────────────────┐
   PERSISTENCE     │  Iceberg tables (resink-owned)               │
   PLANE           │   simfarm.verdicts   ← every verdict, 1y     │
                   │   simfarm.diffs      ← sampled divergences,  │
                   │                        30d retention         │
                   │   simfarm.runs       ← run metadata + timing │
                   │  + Kafka topic       ← live shadow forensics │
                   │    nanofab.simfarm.diffs.<tenant>            │
                   └──────────────────────────────────────────────┘
```

### 3.2 Invariants
1. **Sim Farm never executes production fact data.** Modes A and C use synthetic data from the workspace; mode B observes shadow writes the supervisor is *already* duplicating per the runtime spec. There is no path where Sim Farm re-routes production traffic.
2. **Bug parity.** Every batch sim job runs the same `nanofab-supervisor` binary as production, in `--mode=sim`. There is no second engine to drift from the first.
3. **Verdicts are typed contracts.** The verdict schema is owned by Sim Farm and consumed by both training and the runtime coordinator. Adding a layer to the verdict (e.g., a new check) is a versioned API change.
4. **Diff engine choice follows latency, not preference.** Batch (modes A, C) uses DuckDB because the prior prototype playbook is proven there. Streaming (mode B) uses custom Rust because DuckDB cannot meet the per-event latency budget.
5. **Persistence is the audit and the drift-monitoring substrate.** Iceberg tables let drift monitoring (training spec §7.4) and ops dashboards run plain SQL over historical verdicts.

### 3.3 What Sim Farm is NOT
- Not a synthetic-data generator (training owns that — Sim Farm only consumes `gen_data.py`).
- Not a deployment system (it returns verdicts; runtime coordinator owns the cutover decision).
- Not a runtime — every diff engine is either DuckDB or a small purpose-built Rust daemon, never a general stream processor.
- Not a customer-facing service — verdicts are surfaced to customers *through* training (gate stage 3) and the product UX dashboard, never directly.

## 4. Components

### 4.1 Coordinator (`nanofab-simfarm-coordinator`)
Control plane. One logical instance per region, HA via Raft.
- **Owns:** the job queue, sim-runner pool state (warm/ephemeral), live-shadow sidecar registry, verdict aggregation pipeline, Iceberg commit scheduler.
- **APIs:**
  - gRPC `RunBatchJob(JobSpec) → VerdictStream` for modes A and C — synchronous-feeling streaming RPC with a final terminal verdict.
  - gRPC `StartShadowSession(ShadowSpec) → SessionId` and `WatchShadow(SessionId) → VerdictStream` for mode B.
  - gRPC `EndShadowSession(SessionId) → FinalVerdict` for cutover decisions.
  - HTTP for ops console + drift dashboards.
- **Depends on:** k8s API (to spawn sim-worker Jobs and manage warm-worker deployment), an Iceberg catalog, the Sim Farm coordinator's own metadata store (Postgres or embedded etcd).

### 4.2 Batch sim worker (`nanofab-simfarm-batch-runner`)
Runs in a k8s Job, ephemeral — fresh pod per job. Used for modes A and C.
- **Owns:** orchestration of one sim run from start to verdict.
  1. Pull the workspace artifacts (`manifest.yaml`, `gen_data.py`, `duckdb_reference.sql`, `coverage_spec.yaml`) from the artifact store.
  2. Run `gen_data.py` to materialize parquet.
  3. Boot `nanofab-supervisor --mode=sim --workspace=. --write-trace=trace.jsonl` and feed it the synthetic events.
  4. (Mode A only) Run the DuckDB reference query against the same parquet.
  5. (Mode C only) Pull the comparison fleet's output snapshot from KV → parquet.
  6. Invoke the DuckDB diff engine.
  7. Evaluate against `coverage_spec.yaml`; build the verdict.
  8. Stream verdict back to coordinator.
- **Depends on:** the supervisor binary, DuckDB, the artifact store (workspace), and the coordinator's gRPC for verdict streaming.

### 4.3 Warm-worker pool (`nanofab-simfarm-warm-pool`)
1-2 long-lived pods per region holding pre-booted supervisors. Same image as the batch runner, different lifecycle.
- **Owns:** picking up coordinator-tagged "fast-path" jobs and resetting state between jobs (`supervisor.reset_sim()` clears the in-memory KV and idempotency log).
- **When used:** training pre-deploy validation usually fits in the batch tier (cold start is fine for minutes-scale gates). The warm pool serves the *DAG-level blue/green warmup verification* — when the runtime needs Sim Farm to confirm catchup-equivalence within seconds of a checkpoint replay completing.
- **Concurrency:** one job at a time per warm worker; coordinator dispatches FIFO with priority for hot-swap requests.

### 4.4 Live shadow sidecar (`nanofab-simfarm-shadow-sidecar`)
Always-present sidecar container deployed alongside every supervisor pod, dormant by default. Avoids a rolling deployment update on every shadow session.
- **Owns:** consuming the supervisor's single write-trace stream over a Unix socket, splitting records by `node_version` (each `WriteRecord` carries the version of the plugin that produced it), correlating writes per `event_id` in a bounded LRU (default 100k entries), aggregating diff counters (per-node, per-table, total rate), sampling N divergent events per minute (default 50) to a Kafka topic, pushing 30s-cadence verdict heartbeats to the coordinator.
- **Idle state:** when no shadow session is active, sidecar consumes the trace stream but only updates a lightweight liveness counter; correlation logic is bypassed.
- **Backpressure:** if the LRU evicts an unmatched event before its partner arrives, increment `differ_orphan_count`. Coordinator alerts if `orphan_count / total > 1%`.
- **Lifecycle:** coordinator's `StartShadowSession` activates correlation logic in the addressed pods via a small gRPC call; `EndShadowSession` deactivates it and triggers final verdict computation. No pod lifecycle change.
- **Depends on:** supervisor's `--write-trace` Unix socket (per runtime spec §8.4), Kafka producer, coordinator gRPC.

### 4.5 DuckDB diff engine (`nanofab-simfarm-batch-diff`)
Library used by the batch sim worker. Implements the prototype's `check_consistency.py` patterns in a callable form.
- **Surface:** `diff_parquet(left_path, right_path, schema, tolerance) → BatchDiffReport`.
- **Owns:** alignment by primary key (sort + merge-join), column-by-column compare with per-column tolerance (float epsilon, datetime null-pattern only, string exact), exclusive-key reporting, sample-row collection (top N divergent rows per column).
- **Tolerance config** comes from the workspace's `coverage_spec.yaml` per-table tolerances layer.

### 4.6 Streaming Rust differ (`nanofab-simfarm-stream-diff`)
Library used by the live shadow sidecar. Pure logic, no I/O — sidecar wraps it with socket and Kafka clients.
- **Surface:** `pub fn process_event(node_id, event_id, side: V1|V2, write: WriteRecord) → Option<DiffOutcome>`.
- **Owns:** the per-event_id LRU (DashMap), pair-completion logic, structural diff of `WriteRecord` (key, columns, valid_from/valid_to), per-node and per-table counter aggregation, divergent-sample selection (reservoir sampling).
- **No state beyond the LRU.** All counters and samples flushed to the sidecar layer for delivery.

### 4.7 Verdict schema (`nanofab-simfarm-verdict`)
Strongly-typed shared crate consumed by training and runtime coordinator.
```rust
pub struct Verdict {
    pub run_id: Uuid,
    pub mode: Mode,                 // A | B | C
    pub overall: PassFail,
    pub layers: Layers {
        pub node_coverage: LayerResult,
        pub scenarios: LayerResult,
        pub tolerances: LayerResult,
    },
    pub sampled_diffs: Vec<DiffSample>,   // top N divergent samples
    pub throughput: ThroughputMetrics,
    pub started_at: i64, pub ended_at: i64,
    pub trace_url: String,                 // pointer to full trace in object store
}
```
- **Owns:** the canonical schema, serde encode/decode, version negotiation, the Iceberg row mapping.
- **Versioning:** schema changes are semver-gated; consumers pin the version they support; coordinator refuses jobs from incompatible client versions.

### 4.8 Iceberg writer (`nanofab-simfarm-iceberg`)
Long-running task inside the coordinator. Tails the verdict stream and the sampled-diff topic, batches into Iceberg tables.
- **Owns:** Parquet writer pool, Iceberg manifest commits, retention policy (verdicts 1y, diffs 30d via Iceberg snapshot expiry).
- **Depends on:** S3 (or equivalent), an Iceberg catalog. Same infrastructure as runtime's CDC sink — possibly the same instance.

### 4.9 Drift monitor hook (out-of-scope here, referenced)
Training spec §7.4 mentions canary tenants re-trained nightly. The "monitor that pages on regression" is implemented as a scheduled SQL query over `simfarm.verdicts` Iceberg table. That query lives with training, not Sim Farm — Sim Farm only provides the substrate.

## 5. Execution flow per mode

### 5.1 Mode A — Pre-deploy validation (training gate stage 3)

```
T=0   Training validator → coordinator.RunBatchJob({
        mode: A,
        workspace_uri: "s3://nanofab-workspaces/<tenant>/<commit_sha>",
        timeout_s: 600,
      })

T+1   Coordinator validates job, allocates a k8s Job from batch tier.
      Coordinator opens VerdictStream RPC; replies "scheduled" heartbeat to caller.

T+5   Pod boots (~30s for cold pull). Batch sim worker:
        - clones workspace from S3 to local disk
        - runs sim/gen_data.py --bizdate=20260510 --rows=10000 → data/
        - boots: nanofab-supervisor --mode=sim --workspace=. \
                 --write-trace=trace.jsonl --metrics-port=9100
        - feeds in-memory event source from data/*.parquet
        - waits for supervisor watermark to reach end-of-input
        - dumps in-memory KV → output/*.parquet (one file per dim table)

T+90  - runs duckdb -c "$(cat sim/duckdb_reference.sql)"  → ref/*.parquet

T+100 - invokes DuckDB diff per dim table with coverage_spec tolerances
      → BatchDiffReport per table

T+105 - evaluates layers:
        Layer 1 (node coverage): every node has ≥ N events of each input type
        Layer 2 (scenarios): every named scenario passes its declared assertions
        Layer 3 (tolerances): batch diff already produced per-table pass/fail

T+106 - constructs Verdict, streams to coordinator.
        Coordinator persists to simfarm.verdicts Iceberg table.
        Coordinator returns final verdict on the open VerdictStream RPC.
        Pod exits.

T+107 Training validator sees verdict.layers.* — if all PASS, advances to stage 4.
```

If timeout fires: coordinator kills the pod, returns `Verdict{overall: TIMEOUT, partial layers}` so the caller sees what completed.

### 5.2 Mode B — Per-node shadow during hot swap

```
T=0   Runtime coordinator → simfarm coordinator.StartShadowSession({
        tenant: "acme",
        node_id: "extract_dim_user",
        v1_version: 17, v2_version: 18,
        target_duration_s: 600,
        critical_diff_rate: 0.001,
      })

T+1   Simfarm coordinator:
       - replies SessionId to runtime coordinator
       - issues activate-correlation gRPC to the sidecar in every supervisor
         pod serving this tenant (sidecars are always present, dormant)
       - opens an internal Kafka topic nanofab.simfarm.diffs.acme for forensics
      Runtime coordinator opens WatchShadow(SessionId) → VerdictStream.

T+5   Sidecars active in every supervisor pod. Each:
       - reads /var/run/nanofab/trace.sock (single supervisor trace stream;
         records carry node_version so the sidecar splits v1 vs v2)
       - per WriteRecord: process_event(node_id, event_id, side, write)
       - completed pairs: matches++ or divergences++ + sample to Kafka
       - every 30s: ship aggregated counters to simfarm coordinator

T+60  Simfarm coordinator aggregates per-pod counters across the fleet:
       - if global diff rate > critical_diff_rate:
           emit Verdict{layer: tolerances=FAIL, overall: ABORT}
           runtime coordinator receives via VerdictStream → triggers rollback
       - else: emit Verdict{overall: IN_PROGRESS, partial counters}

...   continues for target_duration_s, with verdict heartbeats every 30s

T+600 Runtime coordinator → simfarm coordinator.EndShadowSession(SessionId)
       Simfarm:
        - flushes final counters
        - reads sampled forensic events from Kafka topic
        - constructs final Verdict, persists to Iceberg
        - issues deactivate-correlation gRPC to the sidecars (pods unchanged)
       Returns FinalVerdict to runtime coordinator on its VerdictStream.

T+605 Runtime coordinator advances to step 3 (canary) or aborts (per its own §6.2 logic).
```

Key property: simfarm coordinator does *not* itself decide cutover. It reports; the runtime coordinator decides.

### 5.3 Mode C — DAG blue/green warmup verification

```
T=0   Runtime coordinator → simfarm coordinator.RunBatchJob({
        mode: C,
        tenant: "acme",
        live_fleet_kv_namespace: "t/acme/v17",
        candidate_fleet_kv_namespace: "t/acme/v18_candidate",
        sample_window_kafka: { topic: ..., offset_lo: ..., offset_hi: ... },
        timeout_s: 1800,
      })

T+5   Coordinator allocates a warm-pool worker (low-latency path).
      Worker:
       - opens read sessions to both KV namespaces
       - exports both → parquet snapshots scoped to keys touched by the offset window
       - invokes DuckDB diff per dim table with the workspace's tolerances
       - evaluates coverage_spec scenarios that apply to "blue/green warmup"
       - constructs Verdict, ships to coordinator

T+120 Coordinator persists Verdict, returns to runtime coordinator.
      Runtime coordinator either cuts over (PASS) or aborts (FAIL).
```

Mode C reuses the batch sim worker code path *but with two real KV snapshots as inputs* instead of supervisor sim outputs vs DuckDB ref.

### 5.4 Drift monitoring (mode A on a schedule)

Training's nightly canary-tenant retrain (training spec §7.4) calls `RunBatchJob(mode=A, ...)` against synthetic fixtures. Verdicts land in `simfarm.verdicts`. A scheduled SQL query (lives in training, not Sim Farm) checks whether any canary's verdict regressed vs the previous night's verdict for the same tenant — that query is what pages on drift.

Drift detection is just a SQL pattern over Sim Farm's persistence — no special API.

## 6. Failure handling

### 6.1 Sim worker pod crash (modes A & C)
- k8s Job retry policy: 0 (Sim Farm controls retry, not k8s).
- Coordinator detects via gRPC stream EOF + pod status. Marks the job FAILED, surfaces to caller as `Verdict{overall: INFRA_FAILURE, layers: empty}`.
- Caller (training validator or runtime coordinator) decides whether to retry. Training retries up to 2× before escalating. Runtime treats `INFRA_FAILURE` on mode C as `PROCEED_WITH_CAUTION` only if a resink operator has set `allow_simfarm_unavailable_for_blue_green` — otherwise blocks the swap.
- Pod crashes never silently produce a PASS verdict.

### 6.2 Supervisor panic inside sim mode
- Supervisor in `--mode=sim` enables the same `panic::catch_unwind` as production (per runtime spec §7.4). A panicked event is recorded in `trace.jsonl` as a `panic_event`.
- Sim worker treats any `panic_event` as a stage-1 failure. Verdict layers gets `node_coverage=FAIL` with the panic node + event surfaced.

### 6.3 DuckDB diff failure
- DuckDB query error (syntax, OOM): worker logs the error, retries once with a smaller row sample, then fails the job with `Verdict{overall: DIFF_ENGINE_FAILURE}`.
- Schema mismatch between candidate and reference (column added/removed): treated as a structural diff, not an engine failure — produces a normal verdict with the schema-diff layer flagged FAIL. Mode C blue/green often expects schema changes; the caller decides.

### 6.4 Sidecar failure mid-shadow (mode B)
- Sidecar OOM or crash: k8s restarts the sidecar container (supervisor keeps running unaffected). On restart, sidecar reconnects to the trace sockets. Loses any in-flight LRU pairs; counter `sidecar_restart_lost_pairs` increments.
- Coordinator monitors sidecar uptime per session. If a sidecar restarts > 3× in 5 min → coordinator marks the session UNRELIABLE; runtime coordinator receives this as a verdict update.
- Persistent sidecar-down (> 1 min): coordinator emits `Verdict{overall: SHADOW_DEGRADED}` and asks runtime coordinator for guidance — never silently keeps reporting partial data as if it were complete.

### 6.5 Supervisor write-trace socket overrun
- Supervisor's `--write-trace` socket is non-blocking with a bounded SO_SNDBUF (default 8 MiB). If sidecar can't drain fast enough, supervisor drops trace events and increments `trace_dropped`.
- Sidecar reads `trace_dropped` from supervisor's metrics endpoint each heartbeat. If `trace_dropped > 0`, sidecar's verdict heartbeat carries `dropped_events_observed: true`, coordinator marks the session DEGRADED.
- A slow differ never produces a falsely-clean verdict — it produces an honestly-degraded one.

### 6.6 Coordinator crash
- Coordinator state lives in metadata store (Postgres/etcd) — job queue, in-flight sessions, sidecar registry. Restart reads from disk and resumes.
- **Mode A/C** (pod-based): pods continue running; coordinator reattaches via job-id lookup on restart.
- **Mode B** (sidecar-based): sidecars continue diffing locally and buffering counters. On coordinator reconnect, sidecars flush buffered counters.
- If both coordinator AND sidecars fail (regional outage): runtime coordinator sees session timeout → aborts hot swap → reverts to live fleet. Conservative default.

### 6.7 What we deliberately do not do
- **Auto-retry verdicts.** Sim Farm never retries an internal job and reports the second result as if the first never happened. Every job attempt creates a verdict row in Iceberg with the attempt number; a retry produces a new verdict referencing its predecessor. Audit trail is immutable.
- **Self-tune tolerances.** Tolerances come from `coverage_spec.yaml` (training-owned). Sim Farm refuses jobs with empty tolerances rather than picking defaults — defaults would make verdicts trustable when they shouldn't be.
- **Cross-tenant verdict aggregation in the hot path.** Drift monitoring queries are best-effort, scheduled offline. The hot path only ever considers one tenant's verdict at a time.

## 7. Testing Sim Farm itself

### 7.1 Per-component unit tests
- **`nanofab-simfarm-batch-diff` (DuckDB engine):** golden parquet pairs (identical, single-column-divergence, row-count-mismatch, schema-drift, datetime-only-null-mismatch) → expected `BatchDiffReport`. Tolerance edge cases. Reuses fixtures derived from the user's prior `check_consistency.py` work.
- **`nanofab-simfarm-stream-diff` (Rust streaming differ):** synthetic event-pair sequences fed through `process_event` → expected counters and divergent samples. Property tests: pair-completion is associative regardless of arrival order; LRU eviction never produces false matches; reservoir sampling is uniformly distributed.
- **`nanofab-simfarm-verdict` (schema crate):** serde round-trip on every variant; semver-bump-aware decode (old client decoding new verdict, new client decoding old verdict).
- **Coordinator:** job-state transitions, session lifecycle, Iceberg writer batching.

### 7.2 Component integration tests
- **Batch sim worker against fixture workspace:** a hand-built tiny workspace (3 dim tables, 5 nodes, 1k synthetic events) that exercises every batch-worker step end-to-end. Assert produced verdict matches a golden. Runs in CI in under 60s.
- **Sidecar against a mock supervisor:** a stub binary that emits a scripted write-trace stream over a Unix socket; sidecar reads and produces counters; assertions on per-node and per-table breakdowns. Includes adversarial scripts (one side stops, both sides duplicate, event ordering swapped).
- **Coordinator + warm-pool integration:** spin up a kind cluster in CI, deploy coordinator + 1 warm worker, fire a `RunBatchJob` against a fixture workspace, assert the warm worker handled it without cold-start.

### 7.3 End-to-end "Sim Farm verifies a fixture DAG"
A `fixture_dags/` directory ships nanofab DAG manifests with known-good and known-bad variants. CI runs the full Sim Farm pipeline against each on every coordinator/sidecar/diff-engine change:
- Known-good DAG → expect verdict `overall: PASS`, all layers green.
- Known-bad-by-tolerance DAG → expect `tolerances: FAIL`.
- Known-bad-by-coverage DAG → expect `node_coverage: FAIL`.
- Known-bad-by-scenario DAG → expect specific scenario FAIL with the correct scenario name surfaced.

Catches regressions where a refactor changes diff-engine semantics — the verdict should change in the expected direction.

### 7.4 Drift monitoring of Sim Farm itself
A meta-canary: a fixture workspace re-run nightly through the entire Sim Farm pipeline. Verdict written to a dedicated `simfarm.self_canary_verdicts` Iceberg table. A scheduled SQL query alerts if the verdict changes from baseline — catches: silent DuckDB version bumps, supervisor binary changes affecting `--mode=sim`, coordinator state-management bugs.

This is "Sim Farm watching Sim Farm" — necessary because everything downstream trusts its output.

### 7.5 Observability
- **Per-job metrics:** queue time, supervisor sim duration, diff duration, verdict size, failure rate by category. Prometheus, dashboards by mode (A/B/C separately).
- **Per-session metrics (mode B):** pair-completion rate, orphan rate, divergence rate, sidecar restart count, write-trace drop rate.
- **Iceberg health:** commit lag, manifest growth, retention pruning success.
- **Verdict consumer reliability:** how often training/runtime callers reconnect mid-stream (signal of network instability between Sim Farm and consumers).

## 8. Open questions deferred to other specs

- **Serving & deployment (#4):** k8s deployment topology for coordinator + warm pool + sidecar lifecycle; multi-region failover; per-tenant resource quotas; cost accounting per mode.
- **Product UX (#5):** how verdict samples and forensic diffs are surfaced to customers in the training stage-3 review and the post-deploy hot-swap dashboards.

## 9. Out of scope for Sim Farm

- Synthetic data generation (training pipeline #2 owns this).
- Cutover decisions during hot swap (runtime coordinator owns this — Sim Farm only reports).
- Customer-facing UI components (product UX #5).
- Re-implementing DuckDB or rolling a custom batch query engine.

## 10. Glossary

- **Mode A / B / C:** the three caller-facing execution modes (pre-deploy / per-node shadow / DAG blue/green).
- **Verdict:** the typed JSON object Sim Farm returns; consumed by training and runtime coordinator.
- **Coverage spec:** the per-tenant `coverage_spec.yaml` declared by training, defining the three layers of validation Sim Farm enforces.
- **Layer:** one of {node coverage, scenarios, tolerances}; verdict is per-layer pass/fail.
- **Sidecar differ:** per-supervisor-pod container that consumes both v1/v2 write-trace streams during shadow mode.
- **Warm-worker pool:** small set of pre-booted supervisor processes ready to accept a low-latency sim job.
- **Write-trace:** the structured per-state-mutation log emitted by `nanofab-supervisor --write-trace` (defined in runtime spec §8.4).
- **Self canary:** the fixture workspace re-run nightly through Sim Farm to detect drift in Sim Farm itself.
