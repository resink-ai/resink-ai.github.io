---
layout: default
title: application-resink-core OKR — 2026-05-15-0001
date: 2026-05-15
status: active
type: okr
loop: 2026-05-15-0001
owner: teams/application/resink-core
grand_parent: Loops
parent: Loop 2026-05-15-0001
nav_order: 10
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-15
  status: active
  loop: 2026-05-15-0001
  links: parent: board/okrs/2026-05-15-0001-ceo-brief.md
-->
{% raw %}

# Resink Core OKR — 2026-05-15 (loop 2026-05-15-0001)

## Context

General-tables **Phase 1b** — closure of Phase 1 of ADR-2026-05-14-001. Phase 1a ratified the cross-team NodeCtx C-ABI callback contract (`teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md`) and landed the pure-Rust scaffolding (`schema.rs`, `node_runner.rs` skeleton with `process_event`'s bridge call `unimplemented!`'d, the 3-dim test harness with `supervisor_runs_three_dims` `#[ignore]`'d). This loop wires the bridge across the contract — `impl NodeCtxBridge for &mut ShardKv`, wire `NodeRunner::process_event`, iterate manifest node-specs through `NodeRunner` in `Supervisor::run`, **delete** `nodes.rs` + `supervisor.rs`'s `match node_spec.table`, **un-ignore + author** `supervisor_runs_three_dims`. AE co-owns the loop via O2 (the `lib.rs.tmpl` template change); neither side's deliverable is complete alone — the byte-stability gate + the three-dim test are the cross-team acceptance signals.

## Objectives

### O1: Wire the NodeCtx C-ABI bridge + retire the hardcoded supervisor path

source: ceo-brief

Why it matters: Phase 1 closure. After this loop the supervisor consumes nodes through one general schema-driven runner — `nodes::user` / `nodes::account` and `match node_spec.table` are gone. The product becomes a genuine general DAG executor (the runtime spec's documented design, reified in code).

Maps to brief O1 → KR1.1 (NodeCtxBridge impl), KR1.2 (NodeRunner::process_event wired), KR1.3 (Supervisor::run iteration), KR1.4 (nodes.rs deleted), KR1.5 (supervisor_runs_three_dims un-ignored + authored), KR1.6 (byte-stability gate), KR1.7 (cargo test green), KR1.8 (ADR + spec status), KR1.9 (tenant-isolation).

**Key results**

- KR1.1: `impl NodeCtxBridge for &mut ShardKv` in `crates/nanofab-supervisor/src/node_runner.rs`. The three callbacks (`get_current` / `close_current` / `append_current`) deserialize JSON key (and row, for `append_current`) per the contract, mutate the `ShardKv`, and emit Mutation trace records (`kind="close"` / `kind="append"`; `key_json` matches the contract's Key JSON shape, which is byte-identical to today's trace `key_json` field). Each callback body is wrapped in `catch_unwind`; a caught panic returns a negative code.

- KR1.2: `NodeRunner::process_event` is wired end-to-end (the `unimplemented!("Phase 1b")` is replaced). Routes the event to the correct shard, builds a `NanofabNodeCtxVTable` whose `ctx_handle` points to that shard's `ShardKv`, serializes the `RawEvent` to `event_json` (same shape `PluginNode::process` already accepts), and calls `PluginNode::process(event_json, ctx_ptr)`. The plugin's returned status code maps to Ok / retry / fail per existing supervisor conventions. `PluginNode::process`'s signature is extended (or a new method added — pick the cleaner) to accept `ctx_ptr` instead of passing null.

- KR1.3: `crates/nanofab-supervisor/src/supervisor.rs`'s `Supervisor::run` iterates `manifest.node_specs`, builds one `NodeRunner` per node-spec (load the spec's `schema.json` into a `DimSchema`; load the spec's cdylib into a `PluginNode`; pass the shard count), routes each `RawEvent` to `runners[event.table].process_event(event, trace_writer)`. At EndOfStream, each runner's `write_output(out_path)` produces the output parquet. The `match node_spec.table.as_str() { "dim_user" | "dim_account" => ... }`, the per-table `buffers.remove("dim_user")` / `buffers.remove("dim_account")`, and the `nodes::user::run` / `nodes::account::run` call sites are deleted.

- KR1.4: `crates/nanofab-supervisor/src/nodes.rs` is **deleted** (the whole 486-line file). `crates/nanofab-supervisor/src/lib.rs`'s `pub(crate) mod nodes;` is removed; `main.rs`'s mirror declaration (if any) likewise. No `#[allow(dead_code)]` parking; no `#[deprecated]` shim. Gone.

- KR1.5: `crates/nanofab-supervisor/tests/general_tables.rs`'s `supervisor_runs_three_dims` is **un-ignored** and its body authored. Build a tempdir manifest with three node-specs (`dim_user`, `dim_account`, `dim_widget`) — schemas already declared as `DIM_USER` / `DIM_ACCOUNT` / `DIM_WIDGET` consts. Construct a minimal fact-event input touching all three dims. Call `Supervisor::run`. Assert: Ok return; three output parquets exist (`dim_user.parquet` / `dim_account.parquet` / `dim_widget.parquet`); each parquet's row count + column shape matches the schema's expected projection. The test passes under `cargo test --workspace --release`.

- KR1.6: **Byte-stability gate.** `cd repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0 && make mvp-loop` exits 0 with `verdict=pass mismatches=0` on the widened 2-dim / 3-fact / 4-shard fixture. The generic runner's output for `dim_user` + `dim_account` is byte-identical to the frozen golden. **If this fails, Phase 1b does not close.** Root-cause and fix — do not paper over with a golden refresh.

- KR1.7: `cargo test --workspace --release` is green — all unit + integration tests including `general_tables::supervisor_runs_three_dims`. The determinism test (slow, `--include-ignored`) is verified green at least once this loop.

- KR1.8: ADR-2026-05-14-001 gains a `## Status — 2026-05-15, loop 2026-05-15-0001 — Phase 1b shipped + Phase 1 closed` section. Narrates what shipped (the v-table wiring landed both sides; `nodes.rs` retired; `match node_spec.table` retired; `supervisor_runs_three_dims` un-ignored + passing; byte-stability gate green); any first-contact contract revisions (zero expected — flag honestly if any). Spec §9 marks Phase 1 closed and re-anchors Phase 2's `loop+N` target.

- KR1.9: Tenant-isolation invariant holds. Final sweep `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns the single canonical line at `org-os/conventions.md:121`. Zero `org-os/` edits.

**Tasks**

- [ ] Implement `impl NodeCtxBridge for &mut ShardKv` in `node_runner.rs`; all three callbacks deserialize JSON, mutate the shard, emit trace records, wrap in `catch_unwind`
- [ ] Extend `PluginNode::process` (or add a method) to pass `ctx_ptr` through to `nanofab_node_process`
- [ ] Wire `NodeRunner::process_event` end-to-end — replace `unimplemented!`, build v-table, serialize event JSON, call `PluginNode::process`, map status
- [ ] Rewrite `Supervisor::run` to iterate `manifest.node_specs` building `NodeRunner` per spec; route events through `runners[event.table].process_event`; `runner.write_output` at EndOfStream
- [ ] **Delete** `crates/nanofab-supervisor/src/nodes.rs`; remove `pub(crate) mod nodes;` from `lib.rs` (and `main.rs` if mirrored)
- [ ] **Delete** the `match node_spec.table` block, per-table buffer dispatch, and `nodes::user::run` / `nodes::account::run` call sites from `supervisor.rs`
- [ ] **Un-ignore** `supervisor_runs_three_dims`; author its body — tempdir manifest with 3 node-specs, minimal fact-event input, assert 3 output parquets + row counts + column shapes
- [ ] **Byte-stability gate**: `cd synthetic_tenants/closed_loop_v0 && make mvp-loop` → `verdict=pass mismatches=0`
- [ ] `cargo test --workspace --release` green; determinism test verified green at least once
- [ ] ADR-2026-05-14-001 Phase 1b closure Status section; spec §9 marks Phase 1 closed
- [ ] Tenant-isolation sweep; final grep returns the single canonical placeholder

## What success looks like

The generic schema-driven `NodeRunner` is the sole supervisor path. `nodes.rs` is deleted; `supervisor.rs`'s `match node_spec.table` is deleted; `supervisor_runs_three_dims` is un-ignored and passes (the synthetic `dim_widget` runs end-to-end alongside `dim_user` + `dim_account` with no supervisor source change). `make mvp-loop` is byte-stable. ADR-2026-05-14-001 records Phase 1 closed. The product is a general DAG executor.

## Cross-team dependency on AE (O2)

AE's `lib.rs.tmpl` change is what makes the v-table the supervisor passes through `ctx_ptr` actually consumed on the node side. If AE's template still uses `DefaultProcessCtx` unconditionally, resink-core's `NodeCtxBridge` is unilateral and the bridge does nothing end-to-end. The `make mvp-loop` byte-stability gate (KR1.6) and `supervisor_runs_three_dims` (KR1.5) are the cross-team acceptance signals — neither passes without both deliverables landing. The contract's lock-step discipline (`NanofabNodeCtxVTable` field order frozen, mirrored verbatim between `node_runner.rs` and `lib.rs.tmpl`) is the protocol that protects against silent divergence.

## Out of scope

- `RawFieldValue`'s widening to `Float64` + `Timestamp` (Phase 2).
- Composite-PK end-to-end exercise (Phase 2 — Phase 1b proves the architecture admits composite PKs via `CompositeKey: Vec<RawFieldValueKey>`; the test harness exercises single-column keys).
- Exporting `NanofabNodeCtxVTable` from `nanofab-node-abi` (later cleanup; this loop the v-table is mirrored inline in `lib.rs.tmpl` per the contract).
- Any `org-os/` process work (hard-scheduled at `loop+2`).
{% endraw %}
