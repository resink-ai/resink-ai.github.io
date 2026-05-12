---
layout: default
title: 2026-05-30-resink-core-capabilities
date: 2026-05-30
status: active
type: report
owner: board
---

<!-- original-frontmatter:
  type: report
  owner: board
  date: 2026-05-30
  status: active
  audience: technical reviewer (engineer, investor, prospective hire)
  links: parent: board/okrs/2026-05-11-1302-ceo-brief.md
-->
# Resink.ai · resink-core capabilities — 2026-05-30

This report describes what `repos/resink-ai/resink-core/` can do **today**, with verdicts and test counts cited as evidence. It does not describe aspirations or roadmap.

## 1. Product summary

`resink-core` is an AI-native dim-table maintainer. Given a customer's fact-parquet streams and a synthetic-tenant fixture describing the target dim-table shape, the system uses an LLM-driven orchestrator (AE's `nanofab:codegen-scd2-node` skill) to generate per-dim Rust SCD2-maintainer "node" crates, compiles them as path-deps of a host supervisor, and runs the supervisor against the fact streams to produce per-dim SCD2-shaped parquets. A sim-farm diff engine then byte-compares the supervisor's output against the fixture and emits a single verdict file. The result is a deployable per-tenant Rust processing pipeline whose output is reproducible against its specification, with a Helm chart skeleton ready for cluster delivery.

## 2. What works today (with evidence)

Each bullet is a present-tense capability backed by an on-disk verdict, a test exit code, or a checked-in artifact.

- **End-to-end MVP closed loop is GREEN on a multi-dim, multi-shard fixture.** `make mvp-loop` exits 0 with `overall_pass: true` on the `closed_loop_v0` synthetic tenant. The verdict file at [`repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/workspace/verdict.json`](../../repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/workspace/verdict.json) contains, verbatim:

  ```json
  {
    "verdict_id": "e2569351-7e5f-461d-b6c3-2080ef0b851e",
    "mode": "A",
    "engine_version": "0.2.0",
    "overall_pass": true,
    "verdicts": [
      { "dim_table": "dim_user",    "pass": true, "mismatch_count": 0 },
      { "dim_table": "dim_account", "pass": true, "mismatch_count": 0 }
    ],
    "ran_at": 1778521885912
  }
  ```

- **Real LLM-driven code generation produces compilable, deterministic Rust.** The orchestrator dispatches AE's `nanofab:codegen-scd2-node` skill via the local `claude` CLI; the generated Rust crate compiles with `cargo build --release` exit code 0 and passes its own per-node `cargo test` exit code 0. Loop 2026-05-11-0958's first end-to-end attempt produced compilable Rust with zero template iterations (~$0.77 in API cost, ~106s wall on a developer laptop) — see [`board/exec-summaries/2026-05-11-0958.md`](../exec-summaries/2026-05-11-0958.md). When `CLAUDE_DISPATCH=1` is not set, the orchestrator falls back to in-process deterministic slot-fill so the loop runs with no API key.

- **Multi-dim parallel maintenance.** The supervisor processes two dim-tables (`dim_user` 21 SCD2 rows; `dim_account` 18 SCD2 rows) in one process driven by three fact streams (`fact_sign_up`, `fact_profile_update`, `fact_account_open`). Per loop 2026-05-11-1113, verdict shows both `dim_user` and `dim_account` with `pass: true` and `mismatch_count: 0`.

- **Multi-shard partitioning with byte-stable hash.** Per-key routing uses `twox-hash::xxh64(key_bytes, seed=0)` against `shard_count=4`. The hash is byte-stable against DE's reference worked example: `xxh64(b"u-001", 0) == 0x571e3e04781b0ff5`.

- **Observable supervisor execution trace.** Each run writes a structured JSONL trace at `workspace/trace.jsonl`. The current verdict's trace has **53 records** of supervisor lifecycle events (node init, partition routing, SCD2 mutations, finalize).

- **Sim-farm Mode-A diff engine is at version 0.2.0 with multi-dim support.** The engine is in `repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py` (467 lines). Its test suite at `repos/resink-ai/resink-core/sim-farm/tests/test_diff_scd2_smoke.py` (419 lines) reports **9/9 passing** under `uv run pytest -q`.

- **Determinism guard end-to-end.** The supervisor's `trace.jsonl` and output parquet are byte-identical across two consecutive runs of the widened fixture; verified via `cargo test --release -p nanofab-supervisor --test determinism -- --include-ignored` (exit code 0).

- **Helm chart skeleton lints, templates, and dry-runs clean.** `deploy/charts/nanofab-supervisor/` (`Chart.yaml` v0.1.0, appVersion `0.1.0`, 15 files) passes `helm lint --strict`, `helm template`, and `helm install --dry-run` — all three exit code 0 on the `values/minikube-mvp.yaml` profile.

## 3. Architecture diagram

The closed loop is a seven-step pipeline. Each arrow is labeled with the data artifact crossing the boundary.

```
+-----------------------+
|     fixture           |
|  (synthetic tenant)   |
+-----------------------+
           |
           |  schema.json + dim_*_fixture.parquet + fact_*.parquet
           v
+-----------------------+
|    orchestrator       |     (Python, training/orchestrator/)
|  AE codegen skill x N |
+-----------------------+
           |
           |  Cargo.toml + src/lib.rs (per-dim node crate)
           v
+-----------------------+
|     cargo build       |     (per-node crate, then supervisor host)
|  (per-dim node x N)   |
+-----------------------+
           |
           |  manifest.yaml + .nanofab/release_seal.json
           v
+-----------------------+
|     coordinator       |     (nanofab-coordinator binary)
|    publish-dag        |
+-----------------------+
           |
           |  validated release_seal + DAG ready signal
           v
+-----------------------+
|     supervisor        |     (nanofab-supervisor binary, --mode=sim)
|  multi-node, sharded  |
+-----------------------+
           |
           |  Event records -> Scd2Row mutations -> trace.jsonl
           v
+-----------------------+
|    diff engine        |     (sim-farm, Mode-A, engine_version 0.2.0)
|   (sim-farm 0.2.0)    |
+-----------------------+
           |
           |  dim_*_output.parquet vs dim_*_fixture.parquet (byte-diff)
           v
+-----------------------+
|       verdict         |
|     verdict.json      |
+-----------------------+
```

## 4. Component catalog

Every shipped surface, with a one-line purpose and a quantitative anchor where available.

| Component | Path | Purpose |
|-----------|------|---------|
| `nanofab-node-abi` crate | `repos/resink-ai/resink-core/crates/nanofab-node-abi/` | Rust trait surface every codegen-output node implements; the contract between supervisor and node. |
| `nanofab-supervisor` crate | `repos/resink-ai/resink-core/crates/nanofab-supervisor/` | Sim-mode driver binary; routes events to per-dim Node instances across logical shards; writes `trace.jsonl` + `dim_*_output.parquet`. |
| `nanofab-coordinator` crate | `repos/resink-ai/resink-core/crates/nanofab-coordinator/` | Validates `release_seal.json`; publishes DAG ready signal so the supervisor knows the per-node crates have completed `cargo build` + `cargo test` successfully. |
| `training/orchestrator/` | `repos/resink-ai/resink-core/training/orchestrator/` | Python module that dispatches AE's codegen skill once per dim, writes `manifest.yaml` + `.nanofab/release_seal.json`, runs `cargo build`/`cargo test` on each codegen-output crate. Entrypoint: `src/orchestrator/__main__.py` (283 lines). |
| `synthetic_tenants/closed_loop_v0/` | `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/` | The end-to-end driver: deterministic fixture generators (`fixtures/generate.py`, `fixtures/derive_facts.py`), a 7-step `Makefile`, the `workspace/` directory where the supervisor + diff outputs land. |
| `sim-farm/` (Mode-A diff engine) | `repos/resink-ai/resink-core/sim-farm/` | Diff engine + verdict-format owner; reads fixture + supervisor-output parquets into DuckDB, emits per-dim PASS/FAIL with one of four mismatch types. `sim_farm/diff_scd2.py` 467 lines; **9/9** tests passing. |
| `deploy/charts/nanofab-supervisor/` | `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` | Helm chart skeleton (v0.1.0, appVersion 0.1.0; 15 files: 8 templates + 2 values files + chart manifests + `Dockerfile` + chart `Makefile` + chart `README.md`). |

## 5. Named deviations

The system runs with four named, documented deviations from the long-term target architecture. Each has an explicit multi-loop restoration plan.

- **Option A: Cargo path-dep static linking, no `dlopen`.** The codegen template exports `nanofab_node_new` / `nanofab_node_drop` C-ABI symbols but no `nanofab_node_process` symbol, so pure `dlopen` is not yet possible. The supervisor links per-tenant node crates as Cargo path-deps, which means a per-tenant supervisor rebuild rather than hot-swap. Documented in [ADR-2026-05-16-001](../decisions/2026-05-16-001-abi-option-a-mvp-deviation.md). Restoration plan: AE's codegen template extension to add `nanofab_node_process` → resink-core supervisor swaps to `libloading::Library` (multi-loop; AE template extension was targeted 2026-05-30 but rescheduled to 2026-06-06 as AE's bandwidth shifted to Bundle C).

- **Sim-farm diff engine hard-codes join columns.** The Mode-A diff engine hard-codes `user_id` as the join column and `(email, country, valid_to, is_current)` as payload columns, so the `dim_account` SCD2 dim is currently filed with an **isomorphic-shape compromise** (same schema as `dim_user` so the diff engine works). Flagged in loop 2026-05-11-1113's retro as P1. Restoration plan: schema-aware join column resolution — deferred from this loop, carries to 2026-06-06.

- **Workspace promotion deferred (3rd consecutive loop).** `resink-core` lives in-tree at `repos/resink-ai/resink-core/` rather than as a real git submodule. Blocker: the `resink-ai/resink-core` GitHub remote does not yet exist. The promotion is gated on a non-tenant action (board creates the remote). Carries to loop 2026-05-11-1631 with a 4th-loop deferral flag for retro.

- **Minikube smoke deferred.** The Helm chart skeleton passes `helm lint --strict`, `helm template`, and `helm install --dry-run`, but no real `helm install` has run against a minikube cluster. Blocker: developer-laptop toolchain (minikube + docker). DevOps owns; carries to 2026-06-06.

## 6. What is NOT in scope yet

The system **today** does not include the following — they are roadmap items not present in the current `repos/resink-ai/resink-core/` tree:

- **Real Kafka ingestion.** The supervisor reads parquet event-source files in `--mode=sim`; the Kafka ingress contract is specified but no Kafka consumer code ships.
- **Real KV state backend.** SCD2 state is held in-process per supervisor instance; no Redis/DynamoDB/persistent KV is wired.
- **Multi-tenant isolation.** One supervisor process per tenant is the current single-tenant assumption; cross-tenant supervisor sharing is not implemented.
- **Hot-swap node code.** Replaced by the Option A static-linking trade-off above; restoration depends on the multi-loop ABI plan.
- **Sim Farm Modes B/C.** Only Mode-A (deterministic file-vs-file diff) is shipped. Mode-B (streaming Rust differ) and Mode-C (sidecar comparison) are spec'd but not implemented.
- **Three-layer verdict.** Today the verdict has one layer (per-dim pass/fail with mismatch counts). The fuller three-layer verdict (engine layer, contract layer, data layer) named in the sim-farm spec is not implemented.
- **Product UX surface (sub-project #5).** No customer-facing UI; the system is exercised via the CLI/Makefile.

## 7. How to read further

- **Resink-core's CLAUDE.md** — `repos/resink-ai/resink-core/CLAUDE.md` (shipped this loop): one-paragraph product summary, tech stack, common commands, repo layout.
- **Resink-core's docs/** — `repos/resink-ai/resink-core/docs/{architecture,concepts,user-guide,module-catalog}.md` (shipped this loop): deeper internal documentation, target audience is an IC opening the repo for the first time.
- **Canonical specs (in parent newbase repo)** — `docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md`, `docs/superpowers/specs/2026-05-10-nanofab-training-pipeline-design.md`.
- **ADRs** — `board/decisions/` — every architectural deviation has an ADR; start with [ADR-2026-05-16-001 (Option A)](../decisions/2026-05-16-001-abi-option-a-mvp-deviation.md) and [ADR-2026-05-10-001 (Rust runtime)](../decisions/2026-05-10-001-nanofab-runtime-is-rust.md).
- **Loop-by-loop history** — `board/exec-summaries/2026-05-11-0958.md` (first GREEN end-to-end), `board/exec-summaries/2026-05-11-1113.md` (widened MVP + 6-ADR batch).
