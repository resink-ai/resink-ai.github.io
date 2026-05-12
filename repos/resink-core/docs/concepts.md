---
layout: default
title: "resink-core: concepts"
date: 2026-05-30
status: active
type: doc
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: doc
  owner: teams/application/resink-core
  date: 2026-05-30
  status: active
-->
# Concepts — resink-core

The vocabulary used across resink-core's code, manifests, traces, and verdicts. Each term is grounded in the file path that defines its shape so a future agent can chase a definition without grepping.

<!-- rit-docs-init:begin auto-generated -->

## Node

A **Node** is the unit of streaming compute in the nanofab runtime — a Rust type that implements the `Node` trait (defined in `crates/nanofab-node-abi/src/lib.rs`). Each Node declares an associated `Key` type and `Row` type, a static `name()` and `version()`, and a `process(event, ctx)` hot path. The MVP supervisor instantiates one Node per dim table (`dim_user_scd2_maintainer` and `dim_account_scd2_maintainer`); the codegen output crates under `synthetic_tenants/closed_loop_v0/workspace/nodes/<dim>_scd2/` carry the concrete implementations. Generated Nodes are slot-filled from AE's `scd2_maintainer` pattern.

## NodeCtx

**NodeCtx** is the mutation surface a Node calls into during `process()` — defined as `trait NodeCtx<K, R>` in `crates/nanofab-node-abi/src/lib.rs`. Its methods are `get_current(table, key)`, `close_current(table, key, event_ts)`, `append_current(table, key, row, event_ts)`, `already_processed(event_id)`, and `mark_processed(event_id)`. The MVP supervisor implements `NodeCtx` against an in-memory `HashMap` (one per shard handler); production targets the State Layer wrapper around TiKV / FoundationDB per runtime spec §4.4. `already_processed` returns false in the MVP — the real per-shard idempotency log is a future-loop deliverable.

## Event

An **Event** is one CDC fact delivered into a Node's `process()` — defined as `struct Event` in `crates/nanofab-node-abi/src/lib.rs` (mirrored from DE's in-memory event-source contract). Fields: `table` (the destination dim discriminator the supervisor routes on), `op` (`Insert` | `Update` | `Delete`), `event_ts` (epoch ms UTC), `event_id` (the deterministic idempotency tail), `before` (pre-image `HashMap<String, FieldValue>`, `None` for insert), `after` (post-image, `None` for delete). The supervisor populates `before`/`after` from upstream fact-stream parquets; the codegen-emitted `build_key`/`build_row` helpers read these maps.

## Scd2Row

A **Scd2Row** is the materialized SCD2 row payload written to the State Layer — defined as `struct Scd2Row<R>` in `crates/nanofab-node-abi/src/lib.rs`. Fields: `payload: R` (the non-PK columns of the slot-filled row type), `valid_from: u64` (set to `event_ts`), `valid_to: Option<u64>` (None while the row is current; a later event for the same key flips it to `Some(closing_ts)`), `is_current: bool` (redundant materialization of `valid_to.is_none()` at write time). The redundant boolean is what query-path readers filter on without scanning every historical version.

## manifest

A **manifest** is the per-tenant declarative spec the supervisor reads on startup — a `manifest.yaml` written by the orchestrator at `<workspace>/manifest.yaml`. Shape is defined by `training/orchestrator/src/orchestrator/manifest.py::write_manifest`: top-level `version`, `shard_count` (4 in the closed loop), and `dags: [...]`. Each DAG has a `name` and a `nodes` list; each node entry carries `id`, `table`, `pk` (the PK column list), `node_version`, `cdylib_path` (the per-tenant `lib<crate>.dylib` path; not yet load-bearing per ABI Option A), and `fact_streams`. The supervisor parses this in `crates/nanofab-supervisor/src/main.rs::load_manifest`.

## release_seal

The **release_seal** is the per-tenant gate-status document — a `.nanofab/release_seal.json` file the orchestrator writes and the coordinator validates. Shape defined by `training/orchestrator/src/orchestrator/manifest.py::initial_seal`: top-level `version`, `nodes` list, `stages` (legacy single-node mirror of the first node's stages for the coordinator's flat reader), `overall_pass` boolean. Each node entry has `node_id` and a `stages` list of `{name, status, reason?}` records with the four stage names `compile`, `smoke`, `sim_farm`, `deploy`. Stage statuses are `pending` | `passed` | `failed` | `skipped`. Stages 1+2 are exercised; 3+4 are MVP-skipped with reason `out-of-scope-for-MVP`.

## verdict

The **verdict** is sim-farm's combined output document at `<workspace>/verdict.json` — the JSON the Makefile's step 7 emits and the final `verdict=pass mismatches=N` line summarizes. Shape conforms to `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`. Multi-dim (engine 0.2.0) shape: top-level `overall_pass` boolean, `verdicts: [...]` list of per-dim verdicts, plus `engine_version` (`0.2.0`), `mode` (`A`), and a stable `run_id` UUID. Each per-dim verdict carries `dim_table`, `pass`, `mismatches[]`, and counts. Exit code is 0 iff `overall_pass = true`; non-zero exit codes surface in `make mvp-loop`'s failure stderr.

## fixture

A **fixture** is a deterministic seeded parquet file under `synthetic_tenants/<tenant>/fixtures/` representing either a dim ground-truth or a fact-stream input. The closed-loop tenant ships two dim fixtures (`dim_user_fixture.parquet`, `dim_account_fixture.parquet`) and three fact-stream parquets (`fact_sign_up.parquet`, `fact_profile_update.parquet`, `fact_account_open.parquet`). All are generated by Python scripts (`generate.py`, `derive_facts.py`) and are bit-identical across `make clean` / `make fixture` cycles. Fixtures are the byte-for-byte ground truth the sim-farm diff compares the supervisor's per-dim output parquets against.

## synthetic_tenant

A **synthetic_tenant** is a self-contained test rig living under `synthetic_tenants/<name>/` and named for the closed-loop pattern it exercises. The only one shipped today is `closed_loop_v0/`. Each tenant carries its own `Makefile` (the end-to-end driver), `fixtures/` (the parquet sources), and runtime `workspace/` (gitignored; populated by `make mvp-loop`, dropped by `make clean`). The pattern is: one tenant = one closed-loop demo. Future loops may add additional tenants for new codegen patterns or new fact-stream shapes; the orchestrator + supervisor + sim-farm composition is the same.

## workspace

A **workspace** is the per-tenant runtime scratch directory the orchestrator and supervisor write into — `synthetic_tenants/<tenant>/workspace/`. Contents at end-of-`make mvp-loop`: `manifest.yaml`, `.nanofab/release_seal.json`, `nodes/<dim>_scd2/` (each is a full Cargo crate the orchestrator wrote), `trace.jsonl`, `dim_user_output.parquet`, `dim_account_output.parquet`, `verdict.json`, `training_history.md`, plus per-node `Cargo.lock` + `target/` + `STATUS.json`. The workspace is gitignored (per the repo's `.gitignore`); `make clean` is the canonical way to reset it.

## shard

A **shard** is a logical partition handler inside the supervisor — one of `shard_count` (4 in the closed loop) per-Node in-process worker contexts. Shards are not OS threads, not processes, not network-addressable workers: they are logical splits of the per-Node event stream computed at routing time. Each shard owns the in-memory state for keys whose `xxHash64(seed=0, partition_key_bytes) % shard_count` equals its index. Multi-process / cross-shard transport is a future-Kafka concern (runtime spec §4.5); the MVP runs all shards in one supervisor process with no IPC.

## partition_key

The **partition_key** is the byte sequence fed into `xxHash64(seed=0)` to compute a shard index for a given event. For the MVP, the partition key is the canonical UTF-8 encoding of the dim's PK value (e.g., `u-001` for a `dim_user` row keyed by `user_id="u-001"`). The byte-stability of the partition key is the contract DE owns and the supervisor consumes — when Kafka ingress eventually lands, the same key bytes must route to the same shard, so the supervisor's `twox-hash` 1.x implementation must remain byte-stable against DE's reference values. See `crates/nanofab-supervisor/Cargo.toml` lines 22–24 for the cited reference value (`u-001` → `0x571e3e04781b0ff5`).

## dag

A **dag** is a manifest-declared node-set the supervisor instantiates as one unit. In the MVP, the manifest declares exactly one DAG (`name: closed_loop_dag`) with two nodes (`dim_user_scd2`, `dim_account_scd2`); the supervisor's `load_manifest` enforces `manifest.dags.len() == 1` for the MVP shape. Future loops may declare multi-DAG manifests; for now, one tenant = one DAG. The DAG is the boundary the coordinator's `publish-dag` command validates (the seal's stages must pass per-node across every node in the DAG).

## fact_stream

A **fact_stream** is one CDC source (a parquet file in the MVP) feeding a Node — declared per-node in `manifest.yaml`'s `fact_streams` list. Each entry has `name`, `op` (the stream-level default operation; the parquet itself may carry per-row ops), and `path` (filename-only, resolved against `--fixtures-dir`). dim_user has two fact_streams (`fact_sign_up` insert + `fact_profile_update` update); dim_account has one (`fact_account_open` with per-row mixed inserts/updates). The supervisor's `event_source.rs` reads each stream's parquet into `RawEvent` records and merges per-Node before sorting + sharding.

## trace

A **trace** is the supervisor's JSONL observability log at `<workspace>/trace.jsonl` — one record per event the supervisor processes. Defined by `crates/nanofab-supervisor/src/trace.rs`. Each record carries the `node_id` (the discriminator across nodes; both nodes share one trace file), `shard`, `event_id`, `event_ts`, `op`, and the outcome (state mutation summary, or error). Per runtime spec §8.4, the trace is the supervisor's contract with the SRE observability surface; when a `panic::catch_unwind` fires, the trace carries the panic message and the SRE runbook at `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` is the triage entry-point.

<!-- rit-docs-init:end -->
