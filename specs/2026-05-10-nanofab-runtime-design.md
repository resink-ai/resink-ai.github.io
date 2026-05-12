---
layout: default
title: "Spec: 2026-05-10-nanofab-runtime-design"
---
# Nanofab Runtime — Design Spec

**Date:** 2026-05-10
**Status:** Draft, awaiting user review
**Scope:** Sub-project #1 of the resink.ai product family — the distributed Rust runtime that executes a customer's generated DAG of dim/ADS-table maintainers in real time. Training pipeline, Sim Farm, serving/deployment, and product UX are separate specs.

## 1. Context & motivation

### 1.1 What resink.ai is
resink.ai is "the data engineering team for a tech company." A customer dumps fact streams (events, binlogs, production data) at it; resink.ai returns clean, well-organized dim/ADS tables ready for BI, risk, growth, and feature serving — without the customer maintaining any ETL DAGs.

It runs in two phases (`ORG.md`):
- **Training:** customer provides sample parquet files; an AI agent designs dim tables, decomposes the data flow into a DAG of small Rust processors, generates and validates the code.
- **Serving:** the generated artifact is deployed and consumes live fact streams, producing dim/ADS tables in real time.

### 1.2 Two ideas inherited from the existing material
From `docs/products/overview.md` and `docs/products/sql-to-realtime.md`:
1. **Skip the SQL execution layer.** The processing engine is purpose-built Rust, not a general SQL engine. No JVM, no shuffle framework, no engine-startup tax.
2. **Synthetic-data simulation.** Validate every generated processor against synthetic streams before letting it touch production traffic.

### 1.3 What this design extends
The existing `sql-to-realtime` skill converts **one SQL job** into **one stream-processor architecture**. This design extends it to convert **a customer's whole portfolio** of fact streams and SQL ETLs into **one distributed Rust application** — Nanofab. Every dim/ADS table in the warehouse is maintained by a node in a single shared DAG, sharing event ingress and state infrastructure rather than each running its own pipeline.

### 1.4 Naming
**Nanofab** = the distributed Rust application produced by training and run during serving. One nanofab per customer (or per tenant). Internally it is a versioned DAG of small Rust processors plus a runtime that schedules, hot-swaps, and observes them.

## 2. Decisions summary

| Axis | Choice | Why |
|---|---|---|
| Deployment unit | Single supervisor binary + per-node `cdylib` plugins | Hot swap individual nodes without touching the runtime; AI codegen produces small, fast-to-build artifacts; in-process IPC between nodes on the same host. |
| State storage | External distributed KV (TiKV or FoundationDB), supervisors stateless | Easy horizontal scale, easy backup, decouples compute from data. Mitigates network-tax with aggressive in-process caching. |
| Sharding | Hash by primary key; small reference tables replicated to every shard | Most computations are key-local; cross-shard fan-in is rare and explicit. |
| Event transport | Kafka/Redpanda for ingress and cross-shard; lock-free in-process channels intra-DAG | Durable, replayable ingress; zero-serialization on the hot path. |
| Hot swap | Per-node shadow + canary for compatible changes; whole-DAG blue/green for breaking changes | Cheap, common case is fast and reversible; safe default for risky changes. |
| Correctness | Event-time + at-least-once + idempotent writes | Matches `sql-to-realtime.md` spec; backfill/replay-safe; avoids the throughput tax of true exactly-once. |
| Output surface | gRPC point-lookup over KV + CDC sink to Apache Iceberg | One source of truth, two read shapes — realtime for risk/features, analytical for BI/growth. |

## 3. Architecture overview

### 3.1 Three planes

```
                    ┌─────────────────────────────────────────────┐
   CONTROL PLANE    │ Coordinator (DAG manifest, versions,        │
                    │ shard assignment, hot-swap orchestration,   │
                    │ Sim Farm verdicts, schema registry)         │
                    └────────────────┬────────────────────────────┘
                                     │ control commands
                    ┌────────────────▼────────────────────────────┐
   DATA PLANE       │  Supervisor fleet (stateless Rust procs)    │
                    │  ┌────────────────────────────────────────┐ │
                    │  │ shard 0    shard 1    shard 2  ...     │ │
                    │  │ ┌──────┐  ┌──────┐  ┌──────┐           │ │
                    │  │ │DAG vN│  │DAG vN│  │DAG vN│           │ │
                    │  │ └──┬───┘  └──┬───┘  └──┬───┘           │ │
                    │  └────┼─────────┼─────────┼───────────────┘ │
   STATE PLANE      │       ▼         ▼         ▼                 │
                    │  External KV (TiKV/FoundationDB) — sharded  │
                    │       │                                     │
                    │       └──► CDC sink ──► Iceberg on S3       │
                    └─────────────────────────────────────────────┘

                    Ingress: Kafka/Redpanda topics (one per fact stream)
                    Egress: gRPC point-lookup API + Iceberg analytical tables
```

### 3.2 Invariants
1. Supervisors are stateless and fungible — kill any one, traffic re-shards via the Kafka consumer-group protocol.
2. State lives in external KV, partitioned by the same key as the Kafka ingress, so a supervisor reads its own shard's KV without cross-shard hops in the hot path.
3. The DAG is data — a manifest the coordinator distributes, not code baked into the supervisor binary. Supervisors load node logic as `cdylib` plugins resolved by `(node_id, version)`.
4. Two hot-swap modes (per-node shadow/canary; per-DAG blue/green), both validated by Sim Farm before cutover.
5. SCD2 history materialized two ways from one source: KV for realtime, Iceberg for analytics.

### 3.3 What Nanofab is NOT
- Not a general stream-processing platform — DAGs come only from the resink.ai training pipeline; there is no user-facing DSL.
- Not a SQL engine.
- Not a database — KV is rented from TiKV / FoundationDB.

## 4. Components

Each component is a separate crate / deployable. Boundaries are testable in isolation.

### 4.1 Coordinator (`nanofab-coordinator`)
Control plane. One logical instance per nanofab cluster, HA via Raft.
- **Owns:** DAG manifest registry (versioned), shard assignment, hot-swap state machine, Sim Farm verdicts, dim-table schema registry.
- **APIs:** gRPC for supervisors (heartbeat, fetch-current-DAG, ack-deploy); gRPC for the training pipeline (publish-new-DAG-version); HTTP for ops console.
- **Depends on:** an embedded etcd-style consistent store (or Postgres) for its own metadata. Does not touch fact data or KV.

### 4.2 Supervisor (`nanofab-supervisor`)
The unit of compute. Stateless Rust process, one per pod/host.
- **Owns:** loaded node plugins, intra-DAG channels, KV client + read-through cache, Kafka consumer group membership, per-shard watermark tracker.
- **Lifecycle:** start → fetch DAG manifest from coordinator → resolve & `dlopen` all node `.so`s → join Kafka consumer group → claim shards → run.
- **Depends on:** coordinator (for DAG), KV cluster (for state), Kafka (for ingress and cross-shard).

### 4.3 Node Plugin ABI (`nanofab-node-abi`)
Crate that defines the contract every generated node implements. Stable, versioned C-ABI surface so plugins built against ABI v1 keep loading after a supervisor upgrades to v2 (with shim).

```rust
pub trait Node {
    type Input: Decode;
    type Output: Encode;
    fn name() -> &'static str;
    fn version() -> u32;
    fn process(
        &mut self,
        ev: Self::Input,
        ctx: &mut NodeCtx,    // KV access, watermark, emit, schedule, dedup
    ) -> Result<(), NodeError>;
}
```

- **Owns:** `NodeCtx` API, encode/decode primitives, deterministic time access, idempotency-log helpers.
- **No dependencies on** transport, KV vendor, or coordinator. Plugins compile in seconds, not minutes.

### 4.4 State Layer (`nanofab-state`)
Wraps the external KV. The boundary the rest of the supervisor sees.
- **Surface:** `get(key, valid_at) → Option<Row>`, `put_scd2(key, row, event_ts)`, `range(prefix, ...)`, `tx(closure)`, `patch_scd2(key, event_ts, mutation)` for late-event corrections.
- **Owns:** read-through cache (LRU per dim table), write coalescing (batch SCD2 versions per Kafka poll), retry/backoff against TiKV/FDB, schema-registry-aware codec.
- **Why a layer:** swapping TiKV → FoundationDB → a future custom store is a one-crate change.

### 4.5 Transport Layer (`nanofab-transport`)
- **Ingress:** `rdkafka` consumer per source topic. Manual offset commits gated on KV write success (at-least-once contract).
- **Intra-DAG:** typed `tokio::mpsc` channels keyed by `(producer_node, consumer_node)`. No serialization on this path.
- **Cross-shard:** dedicated Kafka topics named `nanofab.internal.<dag_version>.<edge>`. Used only when a node's output must be re-keyed to a different shard.

### 4.6 CDC Sink (`nanofab-iceberg-sink`)
Long-running task inside the supervisor. Tails the SCD2 write stream from the State Layer and materializes Parquet files into customer-owned Iceberg tables.
- **Owns:** Parquet writer pool, Iceberg manifest commits, schema-evolution mapping.
- **Depends on:** S3 (or equivalent), an Iceberg catalog (Glue / REST / Polaris).
- **Note:** the only heavy I/O component besides KV — runs on a separate Tokio runtime to avoid starving hot path.

### 4.7 Query Gateway (`nanofab-query`)
Stateless gRPC service co-located with supervisors (or fronted by a load balancer that hashes by primary key to the right shard). Translates `GetUserStats(user_id, valid_at)` → State Layer reads → response.
- **Owns:** request authn/authz (per-tenant API keys), request shaping (avoid N+1 via batched lookups).
- **Depends on:** State Layer + read cache.

### 4.8 Sim Farm hook (out-of-scope here, referenced)
Coordinator exposes a "deploy candidate" command that asks Sim Farm to verify a DAG version against synthetic + replayed real traffic before flipping the cutover bit. Designed in its own spec.

## 5. Data flow (worked example: blog post publish)

Source: a `fact_post_publish` event arrives.

### 5.1 Ingress → DAG entry
```
{event_id: e_8421, user_id: 42, post_id: 991, device_id: d_77,
 ip: "1.2.3.4", ts: 1715300000000}
   │
   │  fact_post_publish topic, partition = hash(user_id) % N
   ▼
Supervisor[shard_3] (this user lives on shard 3)
   │
   ├─ idempotency check: has e_8421 been processed by this DAG version?
   │   yes → skip; no → continue
   │
   ▼
DAG entry node `ingest_post_publish`
   │ deserializes once, emits typed event to all downstream channels
   ▼
   ├──► extract_dim_user        (channel A)
   ├──► extract_dim_post        (channel B)
   ├──► extract_dim_device      (channel C)
   └──► resolve_ip              (channel D)
```

### 5.2 Per-node execution (intra-shard, in-process)
Each node reads its input channel, does point-lookups via State Layer (cache-first), writes new SCD2 versions, emits derived events. No serialization between nodes — they exchange typed Rust structs over `mpsc`.

```
extract_dim_user (shard-local):
  read  dim_user[42, valid_at=now]  → hits cache
  write dim_user[42] new SCD2 row {post_count++, last_post_ts=ts}
  emit  user_updated{42, ts} to downstream channel

resolve_ip (shard-local but uses replicated dim):
  read  dim_ip_geo["1.2.3.4"]       → reference table, replicated to all shards
  emit  ip_resolved{ip, country, asn}

compute_user_velocity ◄─ subscribes to user_updated:
  read  dim_user_window_stats[42, '7d']
  write incremented row + schedules decrement at ts + 7d
```

### 5.3 Cross-shard fan-out (the only network hop in the DAG)
`extract_dim_post` writes a row keyed by `post_id`, not `user_id`. If `hash(post_id) → shard_7`, the node cannot write directly — its supervisor only owns shard 3's KV partition. So:

```
extract_dim_post (on shard 3):
  emit dim_post_write{post_id=991, ...} to internal Kafka topic
       partitioned by post_id

  ──────► Kafka: nanofab.internal.dag_v17.dim_post_writes ──────►

Supervisor[shard_7] (consumer of that internal topic):
  receives, applies KV write to dim_post[991]
```

Cross-shard hops are explicit and visible in the DAG manifest. The training pipeline tries to minimize them by co-locating computations on the same primary key.

### 5.4 Watermark advancement
Each shard tracks `watermark = min(event_ts) − allowed_lateness` across its consumed Kafka partitions. Watermark is published on an internal Kafka topic (`nanofab.internal.watermarks`); the coordinator subscribes for ops visibility, and window-expiry tasks fire when the local watermark crosses a scheduled deadline. Decrement events are deterministic across replays.

### 5.5 State write → CDC sink → Iceberg
Every SCD2 write the State Layer commits is also appended to an in-memory change log per supervisor. The Iceberg sink batches every ~30s (or N MB) and commits a Parquet file + Iceberg manifest update. Customer's Trino / DuckDB / Spark sees new partitions within the minute.

### 5.6 Query path
```
gRPC GetPostStats(post_id=991, valid_at=now)
   │
   │  Query Gateway hashes post_id → shard_7 → routes
   ▼
Supervisor[shard_7].state.get("dim_post", 991, valid_at=now)
   │  cache hit → ~50µs   |   miss → ~2ms TiKV read
   ▼
returns SCD2 row
```

### 5.7 Replay (backfill or after blue/green hot-swap)
1. Coordinator declares DAG version V+1, picks a Kafka offset checkpoint.
2. New supervisor fleet boots, joins a *new* consumer group, replays from checkpoint.
3. Idempotency log scoped to `(dag_version, node_id, event_id)` lets V and V+1 coexist without double-counting in shared KV.
4. Old fleet keeps serving reads. When V+1 catches up to live, query gateway flips to V+1, V drains and exits.

## 6. Hot swap mechanism

Two paths, picked by the coordinator based on the diff between manifest V and V+1.

### 6.1 Diff classification
| Change kind | Path |
|---|---|
| Node logic only (same input/output schemas, same KV schema) | Per-node shadow + canary |
| New node added with no consumers yet | Per-node load (no shadow needed; just `dlopen`) |
| Removed node with no producers | Per-node drain & unload |
| Output schema change, KV schema change, edge re-routing, key change | DAG-level blue/green |
| Anything ambiguous | DAG-level blue/green (default to safe) |

### 6.2 Per-node shadow + canary (the common case)

Namespacing note: per-node shadow uses an inline KV suffix (e.g., `dim_user_candidate_<node_version>`) for ephemeral validation writes. Per-DAG blue/green (§6.3) uses a different scheme — a top-level keyspace prefix per DAG version — because the unit of cutover is the whole DAG, not a single node.


```
state: V live      ─── all events ──► node_A v1
                                            │
                                            ▼
                                       writes KV

step 1 — load:
   coordinator pushes A v2 .so to all supervisors
   supervisors dlopen, instantiate, but do NOT subscribe to channel yet

step 2 — shadow:
   supervisors duplicate input channel: events go to BOTH A v1 (writes KV)
   AND A v2 (writes to a shadow KV namespace, suffixed _candidate)
   Sim Farm subscribes to both write streams, diffs them per event_id

step 3 — verdict:
   after N events or T minutes with diff_rate < threshold,
   coordinator marks A v2 as canary-ready.

step 4 — canary:
   shift X% of events from A v1 to A v2 (still writing real KV now,
   v1 stops writing for those event_ids). Idempotency log prevents
   double writes if a routing decision flips mid-flight.

step 5 — cutover:
   100% to A v2, A v1 drained, .so unloaded after grace period.
   shadow KV namespace torn down.
```

If Sim Farm reports diffs, coordinator aborts: A v2 unloaded, V live untouched.

### 6.3 DAG-level blue/green

```
fleet B (live, DAG vN)         fleet G (candidate, DAG vN+1)
   │                              │
   reads/writes KV                replays Kafka from checkpoint
   serves queries                 writes to KV namespace _candidate
                                  Sim Farm validates against B's outputs

   ───────── cutover atomic flag in coordinator ─────────►

fleet B → drain (finish in-flight, stop consuming)
fleet G → flip from _candidate namespace to live namespace
       → swing query gateway routing to G
       → B exits when its consumer-group lag → 0 on legacy topics
```

**KV namespace switch.** During blue/green, fleet G writes to a parallel keyspace prefix (e.g., `t<tenant>/v<N+1>/dim_user/...`). At cutover, the coordinator atomically rewrites a tenant-level pointer (`live_version=N+1`); query gateway dereferences this pointer per request. No data copy needed if the schema is compatible at read time. If schemas diverged, a one-shot backfill copies vN+1 namespace forward from a snapshot, then live traffic continues against vN+1.

**State warmup time bound.** Replaying Kafka from checkpoint can be slow on huge state. Bounded by: the coordinator picks the checkpoint as `min(retained_kafka_offset, last_iceberg_snapshot_offset)`. The Iceberg sink doubles as a state snapshot — fleet G first imports the most recent Iceberg snapshot into its KV namespace (parallel parquet reads), then replays only the Kafka tail. Catchup goes from "hours" to "minutes" for billion-row tables.

### 6.4 ABI compatibility
The plugin ABI carries a version number. Supervisor refuses to load a plugin whose ABI is newer than supervisor's. Supervisor upgrades roll out *before* plugin upgrades. ABI changes are infrequent and gated by the `nanofab-node-abi` crate's semver.

### 6.5 Rollback
- **Per-node:** re-load v1 `.so` (kept for grace period), re-route, done. < 5 seconds.
- **DAG-level:** flip `live_version` pointer back to N, drain G, restart B if it has already exited. Bounded by Iceberg snapshot freshness.

## 7. Failure handling, idempotency, ordering

### 7.1 Idempotency
Every node maintains a small dedup table in KV at `idemp/<dag_version>/<node_id>/<event_id>` with a 7-day TTL (configurable). Before `process()` runs, supervisor checks; on hit, skip. After successful state writes, supervisor inserts the dedup record in the same KV transaction as the writes — dedup record + state writes are atomic.

Per-`(dag_version, node_id)` (not per-`event_id` alone): lets DAG vN and DAG vN+1 process the same event during blue/green without one short-circuiting the other. After cutover + grace, old `dag_version` dedup namespaces are garbage-collected.

### 7.2 Supervisor crash
- Tokio panics → process exits → orchestrator (k8s/nomad) restarts it → Kafka consumer-group rebalance reassigns its shards within seconds.
- In-flight events that hadn't reached "offset commit" get redelivered → idempotency layer absorbs duplicates.
- Intra-DAG channels (in-process) are lost on crash; safe because no Kafka offset is committed until the *terminal* node of a DAG branch successfully writes KV. Redelivery covers everything.

### 7.3 KV unavailable
- State Layer retries with exponential backoff up to a budget (default 30s).
- If exhausted, supervisor stops committing Kafka offsets → consumer-group lag grows → coordinator alerts → operator investigates.
- Reads fall back to cache during brief outages; writes do not (correctness > availability for SCD2).

### 7.4 Plugin panics
- Each `process()` call runs inside `std::panic::catch_unwind`.
- A panicked event goes to a per-node dead-letter Kafka topic with full context.
- Repeated panics on the same `event_id` → coordinator marks the node poisoned → triggers per-node hot-swap to last-known-good `.so` automatically.

### 7.5 Late & out-of-order events
- "Late" = `event_ts < watermark`. Allowed within `allowed_lateness` (default 24h).
- A late event triggers a retroactive SCD2 patch via `State Layer.patch_scd2`: insert a new row with `valid_from = event_ts` and adjust the surrounding rows' `valid_to` atomically.
- Beyond `allowed_lateness`: event lands on a `__quarantine` topic per source. Operator (or scheduled batch fixup) decides whether to manually rewind window stats.
- Window decrement events are scheduled by event-time, not wall clock — replays produce identical decrements regardless of when they run.

### 7.6 Schema mismatch at the source
- Schema registry validates incoming Kafka events against the source's registered schema before they enter the DAG.
- New fields → ignored, no break.
- Removed/renamed required fields → events route to quarantine topic, coordinator surfaces an "incompatible source schema" alert.

### 7.7 Cross-shard consistency
- Each shard's writes are independent; the only inter-shard coupling is the internal Kafka topics.
- Reads that need to join across shards are not served by the realtime gRPC path — they go through Iceberg/SQL. We deliberately avoid distributed transactions in the hot path.

## 8. Testing & Sim Farm integration

### 8.1 Per-node unit tests (built by codegen)
Each generated node ships with a synthetic test suite the AI agent writes alongside the implementation:
- Golden input/output pairs derived from the customer's parquet samples.
- Property tests for invariants: monotonic counters never decrease, SCD2 rows are non-overlapping, `valid_to > valid_from`.
- Idempotency check: feed the same event twice, assert single state mutation.

Runs in CI on every plugin rebuild.

### 8.2 Whole-DAG simulation in the Sim Farm
The supervisor runs in a deterministic mode: replace Kafka with an in-memory event source, replace the KV with an embedded ACID store (e.g., `redb`), replace the wall clock with a fake clock advanced by event_ts.

What Sim Farm verifies:
- **Equivalence:** if vN is live and vN+1 is candidate, run both against an identical event stream, diff the output dim tables row-for-row. A clean run is the green light for canary/cutover.
- **Coverage:** the synthetic event generator (also AI-built during training) is required to exercise every node and every documented edge case (late event, replay, window expiry, schema variant).
- **Throughput:** at customer-stated event rates, measure end-to-end latency p50/p95/p99 and KV ops/sec. Reject DAG versions that regress > X% without operator override.

### 8.3 Live shadow validation
Per §6.2 — Sim Farm subscribes to both v1 and v2 write streams during shadow phase and diffs them on real traffic. Catches issues the synthetic stream missed.

### 8.4 Supervisor hooks for Sim Farm
- `--mode=sim` flag swaps Kafka/KV/clock for in-memory implementations.
- `--write-trace=path` flag emits a structured log of every state mutation with `(node_id, event_id, key, before, after, event_ts)` — the diff substrate.
- Control gRPC `EnableShadow(node_id, candidate_version)` lets Sim Farm toggle shadow mode at runtime.

Beyond that, no special test harness — Sim Farm runs the same supervisor binary in sim mode.

### 8.5 Observability
- **Metrics:** per-node throughput, lag, dedup hit rate, KV op latency, plugin panics, watermark gap, shard skew. Prometheus-format on a sidecar port.
- **Tracing:** OpenTelemetry spans per `event_id`, propagated through the DAG. One trace shows the full path of an event from Kafka offset to KV write to Iceberg commit.
- **Audit:** every state mutation has the trace_id stamped on it for forensics.

## 9. Open questions deferred to other specs

- **Training pipeline (#2):** how the AI agent decomposes a portfolio of fact streams + sample SQL into a DAG manifest; how it generates per-node Rust code and synthetic test fixtures; how it surfaces design tradeoffs to the customer ("SCD2 vs daily partition," "build device-linkage features?").
- **Sim Farm (#3):** the synthetic data generator; the equivalence-diff engine; the orchestration of shadow runs; the verdict pipeline that feeds the coordinator.
- **Serving & deployment (#4):** packaging the supervisor + plugins for k8s/nomad; multi-tenancy isolation (per-tenant KV prefix, per-tenant Kafka ACLs, per-tenant query API keys); billing hooks; CI/CD for AI-generated plugin rebuilds.
- **Product UX (#5):** customer-facing chat, design dialogue, deployment console, ADS query/export interface.

## 10. Out of scope for Nanofab runtime

- User-facing DSL or query language (DAGs come only from training).
- An embedded SQL engine (we delegate analytical SQL to Trino/DuckDB/Spark over Iceberg).
- A bespoke distributed KV (we rent TiKV/FDB).
- A bespoke message broker (we use Kafka/Redpanda).

## 11. Glossary

- **DAG:** directed acyclic graph of nodes that, together, maintain all of a customer's dim/ADS tables.
- **Node:** a Rust struct implementing the `Node` trait, compiled to a `cdylib` plugin loaded by the supervisor.
- **Supervisor:** the stateless Rust process that hosts plugins, pulls from Kafka, and writes to KV.
- **Coordinator:** the control-plane service that owns the DAG manifest and orchestrates hot swaps.
- **Shard:** a partition of state and a partition of the input Kafka stream, both keyed identically.
- **SCD2:** Slowly Changing Dimension Type 2 — versioned rows with `valid_from` / `valid_to` epoch-millisecond columns; same convention as `sql-to-realtime.md`.
- **ADS:** Application Data Store — high-quality dim/feature tables ready for downstream use.
- **Sim Farm:** the synthetic-data + diff harness that validates DAG versions before they go live.
- **`live_version`:** per-tenant pointer in the coordinator that tells the query gateway which KV namespace to read from. Atomic flip = atomic blue/green cutover.
