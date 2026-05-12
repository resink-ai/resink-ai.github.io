---
layout: default
title: "resink-core: architecture"
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
# Architecture — resink-core

This document describes what the repo runs **today** (loop 2026-05-23 closure state, as carried into loop 2026-05-30's documentation loop). Forward-pointers to specs and ADRs are explicitly marked; anything not yet implemented is named as a deviation in the final section.

<!-- rit-docs-init:begin auto-generated -->

## System shape

resink-core is a polyglot product surface: three Rust crates under `crates/` (managed as one Cargo workspace at the repo root) plus a Python orchestrator at `training/orchestrator/` plus a Python diff engine at `sim-farm/` plus a Helm chart skeleton at `deploy/charts/nanofab-supervisor/`. The Cargo workspace declares `members = ["crates/nanofab-node-abi", "crates/nanofab-supervisor", "crates/nanofab-coordinator"]` and `excludes = ["synthetic_tenants", "training", "sim-farm"]`; the excluded directories carry their own build tooling (Python `uv` projects with their own `pyproject.toml`).

The three Rust crates have one-line responsibilities:

- **`crates/nanofab-node-abi/`** — published `Node` / `NodeCtx<K,R>` / `Event` / `Scd2Row<R>` / `Op` / `FieldValue` / `NodeError` trait surface (see `src/lib.rs`). Field-for-field equivalent with the codegen template at `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/lib.rs.tmpl` — the template currently inlines these types verbatim rather than `use`-ing the upstream crate (this is the preparatory-publish state; the next-loop template revision will switch from inline to upstream).
- **`crates/nanofab-supervisor/`** — `--mode=sim` driver binary (`src/main.rs` + `event_source.rs` + `nodes.rs` + `trace.rs`). Takes a tenant workspace, reads `manifest.yaml`, loads fact parquets, routes events by `Event.table` to the two statically-linked codegen-output node crates, runs each through 4 logical shard handlers, writes per-dim output parquets and a combined `trace.jsonl`.
- **`crates/nanofab-coordinator/`** — `publish-dag --workspace=<path>` validates that `manifest.yaml` and `.nanofab/release_seal.json` are well-shaped and that stages 1 (compile) + 2 (smoke) are `passed` per node; stages 3 (sim_farm) + 4 (deploy) are MVP-skipped. Prints `accepted, version=<seal.version>` on green.

The Python orchestrator at `training/orchestrator/src/orchestrator/` is the **codegen driver**: `__main__.py` is the CLI entry, `dispatch.py` invokes AE's `nanofab:codegen-scd2-node` skill via the `claude` CLI, `codegen_slotfill.py` is the deterministic in-process fallback used when `--skip-dispatch` is passed (the Makefile's default; flipped off when `CLAUDE_DISPATCH=1`), and `manifest.py` emits `manifest.yaml` + `.nanofab/release_seal.json`. The orchestrator runs `cargo build --release` + `cargo test --release` per codegen-output crate and flips the per-node stages in the seal accordingly.

The Python sim-farm at `sim-farm/sim_farm/diff_scd2.py` is the **verdict layer**: it takes one-or-more `(fixture.parquet, output.parquet, dim_table)` triples, computes a DuckDB-backed SCD2 row diff, and emits a verdict JSON conforming to `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`. Engine version is 0.2.0; mode is A (batch). The pytest suite is 9 tests, all passing as of loop 2026-05-23.

The Helm chart skeleton at `deploy/charts/nanofab-supervisor/` is the **deployment surface skeleton**: 15 files total (8 templates under `templates/`, 2 minikube values files under `values/`, plus `Chart.yaml`, `values.yaml`, `Dockerfile`, `Makefile`, `README.md`). `helm lint --strict`, `helm template`, and `helm install --dry-run` all exit 0; minikube smoke is a 2026-06-06 deferred follow-up owned by DevOps.

## The MVP closed-loop seven-step flow

`synthetic_tenants/closed_loop_v0/Makefile`'s `mvp-loop` target chains seven sequential steps. Each step reads inputs only from earlier steps' outputs; `make clean` drops everything written from step 1 onward and the next run reproduces byte-identical artifacts (this is the determinism property the supervisor's `tests/determinism.rs` exercises).

1. **Fixture generation** (`make fixture`) — `fixtures/generate.py` writes `dim_user_fixture.parquet` and `dim_account_fixture.parquet` plus matching `dim_user.schema.json` and `dim_account.schema.json`. Seeded deterministic.
2. **Fact derivation** (`make facts`) — `fixtures/derive_facts.py` derives three fact parquets from the dim fixtures: `fact_sign_up.parquet` (insert ops feeding dim_user), `fact_profile_update.parquet` (update ops feeding dim_user), `fact_account_open.parquet` (per-row insert/update mixed feeding dim_account).
3. **Orchestrate** (`make orchestrate`) — runs `uv run python -m orchestrator` with `--fixture-dir` / `--workspace` / `--plugin-dir` and `--skip-dispatch` (or no flag when `CLAUDE_DISPATCH=1`). The orchestrator iterates over `dim_user` and `dim_account`, slot-fills (or LLM-dispatches) one codegen-output crate per dim under `workspace/nodes/<dim>_scd2/`, writes `workspace/manifest.yaml` (one DAG, two nodes, `shard_count=4`) plus `workspace/.nanofab/release_seal.json` (per-node stages, all initially `pending`), runs `cargo build --release` + `cargo test --release` in each codegen-output crate, flips each per-node stage to `passed`/`failed` in the seal, and appends one history row per node to `workspace/training_history.md`.
4. **Cargo build** (`make build-rust`) — back at the workspace root, `cargo build --release -p nanofab-supervisor` followed by `cargo build --release -p nanofab-coordinator`. The supervisor has Cargo `path = "..."` dependencies on both codegen-output crates, so this step **statically links** the two codegen-output node implementations into the supervisor binary. There is no `dlopen` and no `cdylib` runtime load this loop (see § ABI Option A below).
5. **publish-dag** (`make publish-dag`) — `target/release/nanofab-coordinator publish-dag --workspace=...` reads the manifest and seal, validates per-node stage 1+2 are `passed`, prints `accepted, version=<seal.version>` and exits 0.
6. **Run supervisor** (`make run-supervisor`) — `target/release/nanofab-supervisor --mode=sim --workspace=... --fixtures-dir=... --write-trace=... --write-output-prefix=...`. Reads manifest, loads each node's fact streams into `RawEvent`s, globally sorts each per-node stream by `(event_ts, event_id)`, routes events to the two node instances by `Event.table`, runs each through 4 logical shard handlers (shard = `xxHash64(seed=0, partition_key_bytes) % shard_count`), emits per-event JSONL trace records to `workspace/trace.jsonl`, writes `workspace/dim_user_output.parquet` and `workspace/dim_account_output.parquet`.
7. **Verdict** (`make verdict`) — `uv run python -m sim_farm.diff_scd2` with two `--fixture`/`--output`/`--dim-table` triples and `--verdict=workspace/verdict.json`. Engine 0.2.0 emits a multi-dim verdict JSON with top-level `overall_pass` plus `verdicts: [...]` per-dim. The Makefile prints `verdict=pass mismatches=0` to stdout on green; the recipe exits 0 iff `overall_pass = true`.

## ABI Option A — static linking deviation

The runtime spec (parent newbase: `docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md` §4.3) describes the supervisor as `dlopen`-ing per-tenant cdylib node plugins so that node code is hot-swappable without a supervisor restart. The MVP supervisor instead consumes codegen-output crates as Cargo **path-dependencies** declared statically in `crates/nanofab-supervisor/Cargo.toml` (the two lines beginning `nanofab_node_dim_user_scd2 = { path = "..." }` and `nanofab_node_dim_account_scd2 = { path = "..." }`). Concretely, this means the supervisor binary is **rebuilt per tenant per DAG-version** rather than a single supervisor binary loading swappable plugins. Hot-swap is sacrificed knowingly; the closing seam for MVP is byte-stable static linking.

This is a **named, time-boxed deviation** — the multi-loop restoration plan is recorded in **ADR-2026-05-16-001** (parent newbase: `board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md`). The three steps in that ADR:

1. **AE (loop 2026-05-30):** extend `templates/scd2_maintainer/lib.rs.tmpl` to export `nanofab_node_process` as a C-ABI symbol; the `nanofab-node-abi` crate gains the matching declaration.
2. **Resink-core (loop 2026-06-06):** swap the supervisor from Cargo path-dep to `libloading::Library::open(...)` against the codegen output's `.so`/`.dylib`. The static-linking path remains as a `--features static-plugins` rollback fallback.
3. **Joint, loop after:** hot-swap correctness test against a candidate plugin version.

Until step 2 lands, every new codegen pattern (post-`scd2_maintainer`) MUST be authored against the static-linking path. This is the explicit cost AE re-discovers per pattern.

## Multi-shard partitioning

The supervisor partitions each per-node event stream across 4 logical shard handlers. Sharding is computed as `shard = xxHash64(seed=0, partition_key_bytes) % shard_count` per **DE contract §2 (Partitioning) and §2.1 (xxHash64 ratification — worked examples)** (parent newbase: `teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md`). `shard_count` is read from `manifest.yaml` (default 1; the closed-loop manifest sets it to 4). The implementation uses `twox-hash` 1.x; byte-stability against DE's §2.1 reference value was verified at build time (`u-001` UTF-8 → `0x571e3e04781b0ff5` → `mod 4 = 1`).

The four shard handlers run **single-process, no IPC** — they are logical partitions inside one supervisor process, not separate workers. Multi-process / cross-shard transport is a future-Kafka concern (runtime spec §4.5). The byte-stability of the partition key is preserved so that, when Kafka ingress lands, a key routed to shard `k` in the MVP will route to the same `k` after Kafka ingestion (the partitioning is the contract; the transport is swappable). Per-shard handlers maintain independent in-memory state; the SCD2 close-out semantics (close existing current row, append new current row) execute inside the shard owning the row's partition key.

The supervisor sorts each per-node event stream **globally** by `(event_ts, event_id)` **before** partitioning — i.e., ordering is established across all events for a node, then each event is dispatched to its shard. This preserves SCD2 close-out correctness regardless of shard-handler scheduling order (DE contract §1's deterministic ordering invariant).

## Named deviations

This section records every place the live repo diverges from a canonical spec or a future-loop target. Each item names what is true today, what the target is, and which loop/owner is on the hook for the lift.

- **ABI Option A (static linking, not dlopen).** Carried since loop 2026-05-16. The supervisor's `Cargo.toml` declares the two codegen-output crates as path-deps; there is no `libloading` import and no `cdylib` runtime load. The runtime-spec §4.3 hot-swap property is forgone for MVP. **Multi-loop plan:** ADR-2026-05-16-001 — AE template extension this loop (2026-05-30); resink-core supervisor swap next loop (2026-06-06); joint hot-swap correctness test the loop after. **Where in code:** `crates/nanofab-supervisor/Cargo.toml` lines 34–35.
- **`--bare` dropped from `claude` dispatch.** Carried since loop 2026-05-16. AE's DISPATCH.md recommends `claude --bare ...` for determinism; the orchestrator's `dispatch.py` omits `--bare` because `claude` 2.1.138's `--bare` flag enforces strict env-only auth (`ANTHROPIC_API_KEY` / `apiKeyHelper`) and refuses to read the user's OAuth keychain. **Owner:** AE — DISPATCH.md addendum in Bundle C (loop 2026-05-30). **Where in code:** `training/orchestrator/src/orchestrator/dispatch.py` lines 88–100.
- **`dim_account` isomorphic column shape.** Carried since loop 2026-05-23. Sim-farm's `diff_scd2` engine 0.2.0 hard-codes `(user_id, valid_from)` as join key and `(email, country, valid_to, is_current)` as payload columns (`sim_farm/diff_scd2.py` lines 56–59). To close the widened multi-dim loop without a sim-farm contract extension, `dim_account` was shaped **isomorphically** to `dim_user`: `account_id` literals are written into the `user_id` column, the compound `account_type|status` is encoded in the `country` column, and `email` is synthesized. **Target:** sim-farm engine 0.3 with schema-aware join columns. **Owner:** sim-farm (paused this loop). **Lift target loop:** 2026-06-06. **Flagged for retro this loop.**
- **Workspace promotion to a real git submodule.** Third consecutive defer (carried since loop 2026-05-16). resink-core still lives in-tree under `newbase/repos/resink-ai/resink-core/`. **Blocker:** non-tenant action — the `resink-ai/resink-core` GitHub remote does not yet exist and creating it is a board action, not a team action. The 2026-05-30 documentation loop **partially substitutes** for outside-the-tree observability (resink-core is now navigable via `CLAUDE.md` + `docs/`), but does **not** substitute for structural separation. **Flagged for retro this loop with a fourth-loop deferral note carrying into loop 2026-06-06.**
- **DevOps minikube smoke.** Carried since loop 2026-05-23. Helm chart skeleton at `deploy/charts/nanofab-supervisor/` passes `helm lint --strict`, `helm template`, and `helm install --dry-run`; minikube `helm install` against a real cluster is deferred. **Owner:** DevOps. **Lift target loop:** 2026-06-06.
- **`<verified-against-rust-impl: pending>` tag in DE Kafka contract §2.1.** Pure housekeeping; the supervisor's `xxHash64(seed=0)` byte-stability is verified against the DE reference value in code comments (see `crates/nanofab-supervisor/Cargo.toml` lines 22–24), but the tag in the upstream contract has not been flipped. **Owner:** DE. **Lift target loop:** TBD (deferred this loop).
- **Inlined `nanofab-node-abi` types in codegen template.** AE's `templates/scd2_maintainer/lib.rs.tmpl` inlines `Node` / `NodeCtx` / `Event` / `Scd2Row` / `Op` / `FieldValue` / `NodeError` verbatim rather than `use nanofab_node_abi::*`. Publishing the upstream crate (this loop's pre-condition) is preparatory; the template switch to `use`-ing it is part of AE's Bundle C this loop or a follow-up. **Owner:** AE.

<!-- rit-docs-init:end -->
