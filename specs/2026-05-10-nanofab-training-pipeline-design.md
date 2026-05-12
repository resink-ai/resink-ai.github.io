---
layout: default
title: "Spec: 2026-05-10-nanofab-training-pipeline-design"
parent: "Design specs"
render_with_liquid: false
---
# Nanofab Training Pipeline — Design Spec

**Date:** 2026-05-10
**Status:** Draft, awaiting user review
**Scope:** Sub-project #2 of the resink.ai product family — the AI agent system that turns customer-supplied parquet samples (and optionally SQL ETLs and chat requirements) into a deployable nanofab DAG. Sim Farm, serving/deployment, and product UX are separate specs. The runtime is sub-project #1; this spec produces the artifact the runtime consumes.

## 1. Context & motivation

### 1.1 What this pipeline does
Per `ORG.md`, resink.ai works in two phases. **Training** is the cold-start: a customer dumps fact-data parquet files, the system proposes dim tables, brainstorms tradeoffs with the customer, generates the data-processing pipelines, and validates them against the sample data. The output of training is a *deployable distributed Rust application* — a nanofab DAG version. **Serving** runs that artifact against live data.

This spec covers training only.

### 1.2 What it builds on
The user's prior prototype work on a single SQL job (everything under `docs/products/`) showed that:
- Faker-based parquet generation from SQL parsing works (`gen-data-skill.md`).
- ODPS/Spark SQL → DuckDB translation gives a clean equivalence baseline (`prompts.md`).
- Hand-tuned Rust beats DuckDB by 3.6× warm using specific patterns (sweep-line join, FxHash SoA, mmap IPC, dep_ring incremental, WAD bucket expiry — `prompts.md`).
- The SQL-to-realtime conversion has stable, reusable patterns (SCD2 vs SCD1 vs partitioned, separate window-stats tables, dedicated counter tables to avoid M×N self-joins — `sql-to-realtime.md`).

The training pipeline's job is **to automate and scale that proven workflow** to a portfolio of fact streams and SQL ETLs producing one consolidated nanofab DAG per customer.

### 1.3 Relationship to other sub-projects
- **Sub-project #1 (Nanofab runtime)** — this spec produces the manifest + plugin `.so` set the runtime executes. Per-tenant retrain leverages the runtime's per-node hot-swap mechanism via content-hash dedup.
- **Sub-project #3 (Sim Farm)** — invoked by training as one of four validation gates. Sim Farm is a service this pipeline calls; its internals are out of scope here.
- **Sub-project #5 (Product UX)** — the chat surface and customer sign-off UI are part of the product UX spec. This spec defines only the training-side API contract that UX consumes.

## 2. Decisions summary

| Axis | Choice | Why |
|---|---|---|
| Inputs accepted | Parquet samples (required) + customer SQL ETLs (optional) + chat requirements (optional) | Parquet matches `ORG.md`; SQL gives ground truth for mature orgs; chat fills feature requests. |
| Agent architecture | Conversational orchestrator + specialized sub-agents | Maps onto the existing skill assets; rewindable; testable in isolation. |
| Synthesis strategy | Hybrid — SQL ground truth, parquet fills gaps, chat adds features; multiple options at design forks | Matches `ORG.md`'s "brainstorm different options" promise without forcing customers to greenfield. |
| Codegen | Pattern library + LLM parameterization, escape hatch for novel logic | Bakes in the playbook's hard-won Rust patterns; novel cases clearly flagged. |
| Validation gate | 4 stages — cargo+per-node tests → DuckDB equivalence → Sim Farm synthetic → customer sign-off | Each catches a different failure class; gate is the contract with the runtime coordinator. |
| Output artifact | Per-tenant Rust workspace repo + CI-built `.so` artifacts; incremental retrain via DAG diff | Customer audit; CI-cached builds; plugs into runtime's per-node hot-swap. |

## 3. Architecture overview

### 3.1 Three planes

```
                     ┌────────────────────────────────────────────┐
   CONVERSATION      │  Orchestrator agent                        │
   PLANE             │  - owns the customer chat session          │
                     │  - owns the per-tenant workspace state     │
                     │  - dispatches sub-agents, accumulates      │
                     │    artifacts, surfaces design forks        │
                     │  - rewinds and re-runs stages              │
                     └────────────────┬───────────────────────────┘
                                      │ structured jobs
                     ┌────────────────▼───────────────────────────┐
   AGENT             │  Specialized sub-agents (one per stage)    │
   PLANE             │   data-profiler  ─────►                    │
                     │   sql-importer   ─────►   each returns a   │
                     │   dim-table-designer ──►  structured       │
                     │   dag-planner    ─────►   artifact         │
                     │   node-codegen   ─────►   (JSON/YAML/code) │
                     │   sim-data-gen   ─────►                    │
                     │   validator      ─────►                    │
                     └────────────────┬───────────────────────────┘
                                      │ reads / writes
                     ┌────────────────▼───────────────────────────┐
   ARTIFACT          │  Per-tenant Rust workspace repo            │
   PLANE             │  ├ manifest.yaml         (DAG topology)    │
                     │  ├ schemas/              (dim table DDL)   │
                     │  ├ nodes/                (one crate each)  │
                     │  ├ sim/                  (data + tests)    │
                     │  ├ duckdb_reference.sql  (validation gold) │
                     │  └ training_history.md   (decision log)    │
                     │  CI builds .so + manifest → runtime intake │
                     └────────────────────────────────────────────┘
```

### 3.2 Invariants
1. **Workspace is the single source of truth.** Every sub-agent reads from and writes to the workspace repo; no in-memory state lost between dispatches. Orchestrator can crash and resume.
2. **Sub-agents are stateless and replayable.** Each takes `(workspace path, job spec)` and returns `(workspace diff, structured report)`. Re-running with the same inputs produces a semantically equivalent diff.
3. **The customer sees artifacts, not the agents.** Chat surfaces design forks, schema previews, and validation reports — not raw sub-agent outputs.
4. **Re-training is a diff, not a rebuild.** The orchestrator computes which manifest entries changed, dispatches sub-agents only for affected nodes. Unchanged crates are skipped by the CI builder (cached by content hash), and the runtime coordinator sees an identical plugin URI for those nodes — so its per-node hot-swap diff classifier (runtime spec §6.1) treats them as no-ops and only the changed nodes go through the swap mechanism.
5. **The validation gate is the contract with the runtime.** No DAG version is published unless all four stages pass. The runtime coordinator refuses unsigned manifests.

### 3.3 What this pipeline is NOT
- Not the runtime — it never executes events; it produces the artifact the runtime executes.
- Not Sim Farm — it *invokes* Sim Farm as one validation stage.
- Not the product UX — the chat surface is one *consumer* of the orchestrator; the orchestrator also exposes a non-interactive API for headless re-training (e.g., scheduled refresh after a source schema change).

## 4. Components

### 4.1 Orchestrator (`nanofab-trainer-orchestrator`)
The conversation owner. Long-running per-tenant agent process.
- **Owns:** the workspace path, chat session state, the DAG of stages already run, the queue of pending sub-agent jobs.
- **Inputs:** customer chat messages; webhook callbacks from sub-agent jobs and CI; a "publish" command from the customer once they sign off.
- **Outputs:** chat replies (with surfaced design forks and validation reports); structured DAG version pushes to the runtime coordinator.
- **Depends on:** the workspace repo (file ownership), an LLM provider, a sub-agent dispatch layer (background job queue), the runtime coordinator's gRPC.
- **Recovery:** all conversation state and stage history live as files in the workspace; orchestrator can crash and resume from disk.

### 4.2 Per-tenant workspace repo
Plain git repo, one per tenant, hosted on resink-controlled infrastructure (customer-readable for audit; resink-writable). Layout:

```
<tenant>/
├── manifest.yaml              ← canonical DAG topology + node versions
├── sources/                   ← parquet samples (small, sampled), source-schema definitions
├── customer_sql/              ← original SQL ETLs the customer supplied
├── schemas/                   ← dim table DDL (one .yaml per table)
├── nodes/
│   ├── extract_dim_user/      ← one Cargo crate per node
│   │   ├── Cargo.toml
│   │   ├── src/lib.rs
│   │   └── tests/golden.rs
│   ├── compute_user_velocity/
│   └── ...
├── sim/
│   ├── gen_data.py            ← built by sim-data-gen sub-agent
│   ├── duckdb_reference.sql   ← built by sql-importer or dag-planner
│   └── coverage_spec.yaml     ← edge cases Sim Farm must hit
├── training_history.md        ← decision log: every fork, choice, rationale
└── .nanofab/
    ├── stage_state.json       ← which stages have run, with what input hashes
    ├── pending_forks.yaml     ← design choices waiting on customer
    └── validation_reports/    ← latest report from each gate
```

### 4.3 `data-profiler` sub-agent
Reads `sources/*.parquet`, infers per-column types and value distributions (cardinality, null %, value ranges, top-K), detects entity columns by naming convention (`*_id`, `user_id`, `device_id`, `ip`), suggests candidate dim-table grains.
- **Output:** `sources/<name>.profile.json` per source.
- **Reuses:** the heuristic table in `gen-data-skill.md`.
- **Pure inference; no LLM required for the bulk of the work** (LLM only summarizes findings for the chat surface).

### 4.4 `sql-importer` sub-agent
Active only if customer supplied SQL. Parses each SQL file (re-uses `gen-data-skill.md`'s parser logic — base tables, columns, joins, status-value extraction, partition column), and produces:
- A normalized intermediate representation (IR) per SQL job.
- A **DuckDB-equivalent query** per SQL job, using the ODPS→DuckDB cheatsheet from `prompts.md`. This is the validation gold for stage 2 of the gate.
- A first-pass mapping `customer_sql_table → proposed_nanofab_dim_table`.

### 4.5 `dim-table-designer` sub-agent
Takes profiler output + (optional) sql-importer IR + customer chat history. Proposes the dim/ADS table set. Uses the patterns from `sql-to-realtime.md` (SCD2 vs SCD1 vs partitioned, grain selection, separate window-stats table, dedicated counter tables for cross-joins).
- **Output:** `schemas/*.yaml` (full schema per table, including SCD type, primary key, columns, types, update triggers, change-trigger conditions).
- **Surfaces forks to orchestrator:** "Should `dim_user_signal_stats` be SCD2 or daily-partitioned?" with both options' tradeoffs prefilled.
- **LLM-driven, but constrained by a fixed table-design schema** (no free-form output).

### 4.6 `dag-planner` sub-agent
Takes the dim-table set and decomposes the maintenance work into a DAG of nodes. Each node maps a (source event, dim table) pair to a maintenance operation. Plans node dependencies, identifies cross-shard edges (different keys = explicit Kafka hop), groups nodes into shards.
- **Output:** `manifest.yaml` (node id, type, input streams, output keys, depends-on, cross-shard flag).
- **Pattern selection happens here:** for each node, the planner records which pattern from the pattern library should generate it.
- **Constraint solver, not free-form LLM:** uses heuristics (co-locate by primary key, minimize cross-shard hops, push enrichment as close to source as possible) with LLM only for tie-breaking and human-readable summaries.

### 4.7 `node-codegen` sub-agent
Per node. Takes the node's manifest entry + the schemas it touches + the chosen pattern.
- **If pattern ∈ library:** instantiates the pattern's Rust template with the node's parameters. No LLM in the body; LLM only writes the per-node golden test fixtures based on profiler stats. Output: a complete Cargo crate under `nodes/<node_id>/` with a passing `cargo check`.
- **If pattern = `novel`:** LLM-from-scratch with a strict system prompt that includes the relevant `prompts.md` strategies (FxHash, SoA, mmap IPC for batch warmup, sweep-line) and the `nanofab-node-abi` trait surface. Novel-pattern nodes are flagged in `training_history.md` and tagged in the manifest so a resink engineer reviews them before customer sign-off; they also get an extra Sim Farm coverage pass (the validator runs additional generated edge-case fixtures targeted at the new node, beyond what the standard `coverage_spec.yaml` requires).
- **Always emits:** the crate, the golden test, an idempotency property test, and a one-line entry in `training_history.md`.

### 4.8 `sim-data-gen` sub-agent
Reuses `gen-data-skill.md` directly. Produces `sim/gen_data.py` covering all source streams the DAG consumes, with proper foreign-key consistency (shared user pool), realistic time distributions, and the edge cases listed in `sim/coverage_spec.yaml` (late events, replay, window expiry, out-of-order, schema variants).

### 4.9 `validator` sub-agent
Runs the four-stage gate:
1. **Cargo + per-node tests** — `cargo test --workspace` over the whole repo. Blocking.
2. **DuckDB equivalence** — execute `sim/gen_data.py` to produce parquet, run `duckdb_reference.sql` over it, run the whole DAG in supervisor `--mode=sim` (per runtime spec §8.4) over the same data, diff outputs column-by-column with the tolerance pattern from `prompts.md`. Blocking.
3. **Sim Farm synthetic coverage** — dispatch to Sim Farm with `coverage_spec.yaml`; receive verdict. Blocking.
4. **Customer sign-off** — orchestrator surfaces a sample of generated dim-table outputs (paginated, exportable) to the chat; customer clicks "approve" via the product UX. Blocking.
- **Output:** `.nanofab/validation_reports/<stage>.json` per stage; a single `.nanofab/release_seal.json` once all four pass — this is the artifact the orchestrator presents to the runtime coordinator.

### 4.10 Pattern library (`nanofab-node-patterns`)
Shared resource, version-controlled separately from per-tenant workspaces.
- **Owns:** Rust template files with named slots, plus a `pattern.yaml` per pattern declaring its parameters, applicability conditions, and which `prompts.md` strategies it implements.
- **Initial set** (from `prompts.md` + `sql-to-realtime.md`):
  - `scd2_counter_maintainer` — running counts/sums per key with SCD2 versioning
  - `scd1_first_event` — immutable first-event-time per key
  - `window_stats_with_decrement` — rolling window with scheduled decrement task
  - `sweep_line_pair_count` — replaces M×N self-joins (the dep_ring + binary search pattern)
  - `enrichment_lookup` — KV lookup against a reference dim (rate, geo, master mapping)
  - `cross_shard_re_key` — emits to internal Kafka topic when output key differs from input
  - `incremental_v6` — per-user state with delta processing (the v6/v7 pattern from `prompts.md`, for nodes with high clean-user fraction)
- **Patterns are themselves tested** (see §6.2) so a pattern regression is caught before any tenant pulls it.

### 4.11 CI builder
Per-tenant CI workflow. Trigger: orchestrator pushes a commit to the workspace repo with a release tag.
- Steps: `cargo build --release` per node crate → upload `.so` artifacts to a content-addressed object store keyed by `(node_id, content_hash)` → emit a manifest pointing to those URIs → notify orchestrator.
- Cached: unchanged crates (same content hash) skip the build.

### 4.12 Runtime handoff
Orchestrator calls `coordinator.PublishDagVersion(manifest, release_seal)` (defined in the runtime spec §4.1). Coordinator validates the seal, registers the version, and triggers the runtime's blue/green or per-node hot-swap machinery (per runtime spec §6.1, based on diff classification).

## 5. Stage flow

### 5.1 First-time training (cold start)

```
T=0   Customer creates tenant, uploads:
       fact_user_signup.parquet
       fact_user_signin.parquet
       fact_post_publish.parquet
       fact_comment.parquet
      (Optional) Uploads existing SQL: dws_user_engagement.sql, dws_post_metrics.sql
      Sends first chat: "Build a data warehouse for our blog. We need
                         growth metrics and risk signals — device linkage matters."

T+1   Orchestrator initializes workspace, dispatches:
        ┌─ data-profiler (parallel, one per parquet file)
        └─ sql-importer  (parallel, one per SQL file, only if supplied)

T+2   Orchestrator collects results; dispatches dim-table-designer with full context.

T+3   dim-table-designer proposes:
        dim_user, dim_post, dim_comment, dim_device, dim_ip            (entity dims, SCD2)
        dim_user_engagement_stats, dim_post_engagement_stats           (counters, SCD2)
        dim_user_engagement_window_stats (1d/7d/30d)                   (window, SCD2)
        dim_device_user_link, dim_ip_user_link                         (cross-entity counters)
      Surfaces 3 forks:
        Fork A: SCD2 vs daily-partition for dim_user_engagement_window_stats
        Fork B: separate dim_device_user_link table vs columns on dim_user
        Fork C: include dim_ip_geo (vendor lookup) now or defer to phase 2

T+4   Orchestrator surfaces forks to customer, one at a time. Customer answers.

T+5   dim-table-designer re-runs with customer choices baked in.

T+6   dag-planner runs:
        - Maps each fact stream → which dim tables it must update
        - Identifies cross-shard edges (dim_post is keyed by post_id, not user_id)
        - Picks pattern per node (extract_dim_user → scd2_counter_maintainer, etc.)
      Emits manifest.yaml. Surfaces 1 fork on cross-shard tradeoff.

T+7   Customer chooses; manifest finalized.
      Orchestrator dispatches node-codegen in parallel, one job per node.

T+8   Orchestrator dispatches sim-data-gen.

T+9   Orchestrator dispatches validator. All four stages.

T+10  Customer signs off. release_seal.json minted. Workspace tagged release/v1.

T+11  CI builds per-node .so files, content-addressed, uploaded.

T+12  Orchestrator calls coordinator.PublishDagVersion(manifest, release_seal).
      Runtime accepts; this is DAG v1. Customer enters serving phase.
```

### 5.2 Re-training (incremental)
Customer comes back: "Add fraud-detection signals — flag accounts that publish more than 5 posts in their first hour."

```
T=0   Customer chat → orchestrator.

T+1   Orchestrator infers incremental change. Dispatches dim-table-designer.

T+2   Designer proposes one new table: dim_user_first_hour_post_velocity (SCD1).

T+3   dag-planner computes diff:
        + new node: compute_first_hour_velocity
      Pattern: novel. Surfaces fork: LLM-from-scratch + human review, or extract pattern first?

T+4   Customer chooses LLM. Orchestrator dispatches node-codegen for new node only.
      All other nodes' content hashes unchanged → CI skips their builds.

T+5   sim-data-gen --diff: extends fixtures for the new node.

T+6   Validator runs in scoped mode (changed nodes + downstream impact).

T+7   Manifest v1 → v2. Runtime coordinator triggers per-node load (cheap path).
```

### 5.3 Schema-change retrain
Customer adds `is_draft: bool` to `fact_post_publish`. Orchestrator detects new field on next parquet sample upload, dispatches `data-profiler`, surfaces a fork: *"New field `is_draft`. Should it (a) cause `dim_post` to gain an `is_draft` column, (b) feed a new `dim_user_draft_stats` table, or (c) be ignored?"* Customer picks (a). Same incremental flow as 5.2, but with a schema migration entry for `dim_post` — orchestrator decides this is a breaking schema change and signals the runtime to use blue/green (per runtime spec §6.1) at deploy time.

## 6. Failure handling, retries, customer iteration

### 6.1 Sub-agent failure (crash, timeout, LLM provider error)
- Each sub-agent dispatch is a job in a durable queue with `(workspace_path, stage, input_hash)` as the dedup key.
- Crash or timeout → automatic retry with exponential backoff (3 retries default).
- After retries exhausted: orchestrator surfaces "agent failed" to chat with the sub-agent's last log; customer can `retry`, `skip` (only for non-blocking stages — never for validator), or `abort`.
- LLM provider outage → orchestrator parks the job, surfaces "AI service degraded — paused" to chat, resumes when health recovers.

### 6.2 LLM produces invalid output
Every sub-agent that uses LLM output validates the response against a strict schema (Pydantic for Python sub-agents, serde for Rust ones) *before* writing to the workspace.
- Schema-invalid → automatic retry with the validation error appended to the system prompt. Up to 3 self-correction attempts.
- After 3 failures: surface to customer with the offending output in a collapsible block; customer can rephrase, edit YAML directly, or escalate.

### 6.3 `cargo` failure on a generated node
- Pattern instantiation that fails `cargo check` is treated as a pattern-library bug, not a tenant issue. Job records the failure, orchestrator escalates internally to resink engineers, customer-visible chat says "patching a generation bug, retrying shortly."
- LLM-generated novel-pattern code that fails `cargo check`: same auto-correction loop as 6.2 with the compiler error fed back. After 3 failures: customer is asked whether to (a) accept a stub implementation that passes shape checks but logs `TODO` at runtime — useful only for non-critical features, with a loud banner — or (b) defer the feature and ship the rest of the DAG.

### 6.4 Validation gate failure
- **Stage 1 (cargo + per-node tests):** same as 6.3.
- **Stage 2 (DuckDB equivalence diff):** orchestrator gets a structured diff report (column, expected, actual, sample rows). Dispatches a `diff-triage` LLM job that classifies the diff as (a) bug in our codegen — re-run codegen with diff in context, (b) bug in customer's SQL we're mirroring — surface for confirmation, (c) acceptable rounding/ordering — auto-add to allowed-tolerance list with a note in `training_history.md`, (d) genuine semantic ambiguity — surface as a fork.
- **Stage 3 (Sim Farm coverage):** Sim Farm verdict comes back with which edge cases failed. Orchestrator dispatches codegen scoped to the affected nodes with the failing case as a new test fixture. Loops until pass or budget exhausted.
- **Stage 4 (customer sign-off):** treated as feedback, not failure — comments on individual rows route back to the appropriate sub-agent as a refinement request.

### 6.5 Customer interrupts / changes their mind mid-flight
- Orchestrator triages incoming messages: (a) clarification of an open fork → bake into the relevant sub-agent's input and let it complete; (b) new requirement → queue as follow-on, current jobs continue; (c) "stop / undo" → cancel in-flight jobs, rewind workspace to last released-and-signed state (or to a customer-named checkpoint), surface diff for confirmation.
- Workspace is git — every sub-agent commit is on a branch named `stage/<run_id>/<sub_agent>`. Rewind is a `git reset` to the last `release/v<n>` tag plus a fresh branch.

### 6.6 Customer abandonment
- No chat activity and no pending validations for N days (default 14): orchestrator marks workspace `cold`, releases compute, persists state to the workspace repo. Re-engagement reloads from disk in seconds.
- `cold` for M months (default 6): workspace moved to cheap object storage; re-activating asks the customer to confirm before reconstituting.

### 6.7 Orchestrator crash
Same property as the runtime supervisor. All relevant state lives in the workspace (`.nanofab/stage_state.json`, `.nanofab/pending_forks.yaml`, periodic chat-log commits). Restart reads these and resumes — including resuming any in-flight sub-agent jobs (job queue is durable; jobs key on input hash for dedup).

### 6.8 What we deliberately do not retry
- **Customer sign-off** (stage 4 of validator). Rejection is feedback, not failure. No "auto-approve" fallback exists.
- **Cross-tenant escalations.** If a sub-agent fails in a way suggesting a pattern-library bug or runtime-spec violation, orchestrator pauses *all* tenants on that pattern and notifies resink engineers. Prevents a bad pattern from quietly producing wrong data warehouses across the customer base.

## 7. Testing the training pipeline itself

### 7.1 Per-sub-agent unit tests
Each sub-agent has `tests/` with golden `(input → output)` fixtures.
- `data-profiler`: fixture parquet → expected `profile.json` (no LLM in the assertion path).
- `sql-importer`: 20+ ODPS SQL samples → expected normalized IR + DuckDB query.
- `dim-table-designer` and `dag-planner`: golden customer-context fixture → expected `schemas/` + `manifest.yaml`. LLM nondeterminism handled by checking *invariants* (counts, presence/absence of specific tables, edge sets) rather than exact output. Re-runs are seeded for reproducibility where possible.
- `node-codegen`: per-pattern fixture → expected Cargo crate; assert `cargo check && cargo test` passes.

CI runs all of these on every change to a sub-agent or pattern.

### 7.2 Pattern library tests (`nanofab-node-patterns/tests/`)
Each pattern is its own unit of test:
- **Compile test:** instantiate with representative parameters, assert `cargo build`.
- **Behavioral test:** run the instantiated node against golden input events using a stubbed `NodeCtx`, assert state mutations and emitted events match expected.
- **Performance test:** benchmark vs DuckDB equivalent on the canonical workload from `prompts.md` (300K rows, 5 runs, warm avg). Patterns that regress > 10% vs their last-known-good fail CI.
- **Property test:** idempotency, monotonic counters, SCD2 non-overlap.

Patterns are versioned. A pattern bump rebuilds every tenant's affected nodes on next training run; orchestrator surfaces it as routine maintenance retrain.

### 7.3 End-to-end synthetic-tenant tests
A `synthetic_tenants/` directory ships fixture customers (parquet + optional SQL + scripted chat transcript). CI runs the full training pipeline against each on every change to the orchestrator or sub-agent set:
- Asserts the resulting `manifest.yaml` matches a golden (or that diffs are within a stated, maintained allowlist).
- Asserts the four-stage validation gate completes successfully.
- Loads the resulting workspace into a runtime supervisor in `--mode=sim`, runs a canonical event stream, asserts dim-table outputs match a golden.

Initial fixture set: a pure-greenfield startup (parquet only), a SQL-rich enterprise (parquet + 20+ SQL ETLs), the blog example from §5.1, and a port of the existing `prompts.md` prototype.

### 7.4 Production drift monitoring
LLM providers update models; pattern libraries evolve; cargo crates get yanked. Drift makes silent failures more likely than for the runtime.
- **Canary tenants:** 2-3 designated synthetic tenants are re-trained from scratch nightly. Any unexpected manifest change, validation regression, or codegen diff is paged.
- **Per-tenant invariants:** every workspace's `manifest.yaml` carries a `schema_invariants` block. Re-training that violates an invariant blocks at the validator stage and surfaces a fork to the customer rather than silently changing schema.
- **Coverage drift:** Sim Farm's coverage report includes a "scenarios that no longer exercise any of the changed nodes" warning so a shrinking validation surface is visible.

### 7.5 Observability (training pipeline metrics)
- Time-to-first-DAG per cohort (cold-start customer median; SQL-rich vs greenfield).
- Stage-pass rates: % of training runs that pass each gate without retry.
- Pattern usage distribution: which patterns are used most; which novel patterns recur (candidate for extraction into the library).
- LLM cost per training run, by sub-agent.
- Customer fork count per cold-start training (proxy for how opinionated the agent is).

Per-sub-agent metrics on a Prometheus endpoint; cohort dashboards in the resink ops console.

## 8. Open questions deferred to other specs

- **Sim Farm internals (#3):** the synthetic data generator engine, the equivalence-diff implementation, the orchestration of shadow runs, the verdict pipeline. This spec only defines the API training calls.
- **Serving & deployment (#4):** how per-tenant `.so` artifacts are distributed to runtime supervisors; per-tenant isolation; CI infrastructure; billing.
- **Product UX (#5):** the chat surface itself; the customer sign-off UI; the design-fork rendering; the audit-view of the workspace repo.

## 9. Out of scope for the training pipeline

- Any execution of customer events at scale — that's the runtime (#1).
- The synthetic-data engine itself — that's Sim Farm (#3).
- Customer-facing UI components — that's the product UX (#5).
- A bespoke LLM — we use a vendor (Anthropic / OpenAI / open-weights) accessed through a thin abstraction.

## 10. Glossary

- **Workspace:** the per-tenant git repo holding all training-state files for one customer.
- **Sub-agent:** a single-purpose LLM-or-deterministic worker dispatched by the orchestrator; takes a job spec, returns a workspace diff plus a structured report.
- **Design fork:** a non-trivial design choice surfaced by a sub-agent (e.g., SCD2 vs daily partition); orchestrator pauses and asks the customer.
- **Pattern:** a parameterized Rust template in the pattern library implementing one of the playbook's known shapes (SCD2 maintainer, sweep-line join, etc.).
- **Novel pattern:** a node whose logic doesn't fit any library pattern; generated by LLM-from-scratch and flagged for human review.
- **Validation gate:** the four-stage check (cargo+tests → DuckDB equivalence → Sim Farm → customer sign-off) every DAG version must pass before the runtime accepts it.
- **`release_seal.json`:** the artifact emitted when all four gate stages pass; presented to the runtime coordinator's `PublishDagVersion` API.
- **DAG diff:** the set of manifest entries that changed between two DAG versions; used to scope codegen, CI builds, and validation in retrains.
- **Diff-triage:** a specialized LLM job that classifies a stage-2 equivalence-diff into (codegen bug | customer-SQL bug | acceptable tolerance | semantic ambiguity).
