---
layout: default
title: CEO Brief — 2026-05-14-0857
date: 2026-05-14
status: active
type: okr
loop: 2026-05-14-0857
owner: board
grand_parent: Loops
parent: Loop 2026-05-14-0857
nav_order: 1
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-14
  status: active
  loop: 2026-05-14-0857
  links: parent: ""
-->
{% raw %}

# Resink.ai CEO Brief — 2026-05-14 (loop 2026-05-14-0857)

## Context

Second loop on 2026-05-14. The prior loop (`-0742`) opened the general-tables architectural arc design-first: ADR-2026-05-14-001 + a design spec, naming the supervisor's hardcoded `dim_user`/`dim_account` handling as a deliberate MVP deviation, with a three-phase restoration plan. **This is loop+1; the theme is Phase 1.**

**A first-contact finding reshaped Phase 1.** The ADR sized Phase 1 as "resink-core; M". Probing the C-ABI before authoring this brief found that wasn't right:

- ADR-2026-05-14-001 Decision §3 commits Phase 1 to consuming nodes through the **dlopen `Node` C-ABI** — that is the vehicle for a *single general supervisor binary*.
- But the C-ABI export is `nanofab_node_process(node_ptr, event_json, event_json_len, ctx_ptr) -> i32` — it returns **only a status code**. Every SCD2 mutation the supervisor needs (close the current row, append the successor, emit the trace record) must flow back through `ctx_ptr`.
- And `ctx_ptr` is **unwired today.** The codegen template `lib.rs.tmpl` (AE-owned) ignores it (`_ctx_ptr`) and processes against an in-process `DefaultProcessCtx`. The dlopen `PluginNode::process` even passes `ctx_ptr` as `null`.
- So Phase 1, done faithfully to the ADR, requires an **AE codegen-template change** to wire `ctx_ptr` → a real callback bridge. It is a cross-team loop, not a solo resink-core "M". The companion spec's §9 pre-flagged this exact site as the likeliest first-contact revision.

The supporting types are in good shape and confirm the path is sound: the `nanofab-node-abi` crate already publishes the generic `NodeCtx<K, R>` trait (`get_current` / `close_current` / `append_current` / `already_processed` / `mark_processed`) and the generic `Event` / `FieldValue` / `Node` surface. `FieldValue` already has `Float64` + `TimestampMs` (the supervisor's `RawFieldValue` is the laggard — that gap is Phase 2's, not this loop's). The C-ABI callback contract's job is precise: **project the generic `NodeCtx<K, R>` trait across a type-erased FFI boundary** — define how keys and rows serialize, how the callback table is shaped behind `ctx_ptr`, how errors propagate.

**The shape of this loop.** **Contract-first split, per CEO scope selection.** Two teams active:

- **resink-core (primary)** leads the cross-team **NodeCtx C-ABI callback contract** and lands the pure-Rust scaffolding that does not depend on the bridge: the `DimSchema` model + validation + parsing (`schema.rs`), the generic `NodeRunner` skeleton with the bridge call site stubbed (`node_runner.rs`), and the `tests/general_tables.rs` harness with a third synthetic dim.
- **AE** co-authors + acks the C-ABI callback contract. AE's *template-change* deliverable (wiring `ctx_ptr` in `lib.rs.tmpl`) is **next loop** — this loop AE only needs to agree the contract is implementable against the template's `nanofab_node_process` shape.

**Next loop (Phase 1b):** AE ships the `lib.rs.tmpl` `ctx_ptr` wiring; resink-core implements the `NodeCtxBridge`, wires the generic `NodeRunner` into `Supervisor::run`, retires `nodes.rs`'s `mod user` / `mod account` and `supervisor.rs`'s `match node_spec.table`, and the third synthetic dim runs end-to-end through `make mvp-loop`.

**Why contract-first.** Co-designing an unsafe-FFI callback ABI across two teams *within a single loop, without a contract first* is the classic integration-failure setup. The org's contract/spec-first discipline (memory: `feedback-spec-first-architectural-arc.md`; the DE contract corpus) exists for exactly this. The contract becomes the artifact both teams' Phase 1b work executes against.

**Probe results entering the loop:**

- `crates/nanofab-supervisor/src/plugin_loader.rs` — the `PluginNode` C-ABI loader. `nanofab_node_process` signature confirmed: `(node_ptr, event_json, event_json_len, ctx_ptr) -> i32`. Four status codes (`OK`/`PANIC`/`BLOCKED`/`RETRY`). `process()` currently passes `ctx_ptr = null`.
- `crates/nanofab-node-abi/src/lib.rs` — the shared trait crate. `NodeCtx<K, R>` trait (5 methods), `Node` trait (`type Key`, `type Row`, `process`), `Event`, `FieldValue` (6 variants incl. `Float64` + `TimestampMs`), `Scd2Row<R>`, `Op`, `NodeError` (3 variants). Documents that the codegen template currently *inlines* this surface and will switch to `use nanofab_node_abi::*` in a future template revision.
- `lib.rs.tmpl` (AE codegen template) — `nanofab_node_process` ignores `_ctx_ptr`, uses an in-process `DefaultProcessCtx`. The template's own comment names "the supervisor's real `NodeCtx` ... that work is owned by resink-core" — i.e. AE expects this contract.
- `crates/nanofab-supervisor/src/nodes.rs` — 486 lines, `mod user` + `mod account`, hand-written per-table adapters. **Not touched this loop** — it retires in Phase 1b.
- `crates/nanofab-supervisor/src/supervisor.rs` — `match node_spec.table` + per-table buffers + `nodes::user::run` / `nodes::account::run`. **Not touched this loop** — retires in Phase 1b.
- Tenant-isolation entering: clean (single canonical `acme.ai` placeholder at `org-os/conventions.md:121`).

**Re recent retro carryovers:**

- **2026-05-14-0742 retro P3 (general-tables Phase 1):** this loop, re-shaped to the contract-first split per the C-ABI finding above.
- **2026-05-14-0742 retro P2 (sequence the two open arcs):** CEO decision recorded — general-tables proceeds; streaming (ADR-2026-05-13-001 steps 3-4) stays deferred carryover. Re-evaluate after Phase 1b.
- **2026-05-14-0742 retro P1 (dedicated org-os-process loop — clear the P2/P3/P4 backlog of arc-report-type ADR + link-existence smoke + spec-first codification):** still queued. This loop is a code loop, not the org-os-process loop. Recommended slot unchanged: a dedicated org-os-process loop after Phase 1b. Defer.
- **2026-05-14-0742 retro P4 (spec-first codification):** folded into the P1 org-os-process loop; deferred with it.

**CEO decisions for this loop:**

- **Activate two teams: resink-core (primary) + AE (light). Board for brief + exec + retro.** All other teams paused. resink-core leads the contract + lands the pure-Rust scaffolding; AE co-authors + acks the contract.

- **One objective per active team.** resink-core O1 (the contract + `schema.rs` + `node_runner.rs` skeleton + the test harness) sized M-L; AE O2 (co-author + ack the contract) sized S.

- **`nodes.rs` and `supervisor.rs` are NOT touched this loop.** The live `static-plugins` path stays exactly as-is; `make mvp-loop` is byte-stable *trivially* (the new `schema.rs` / `node_runner.rs` are additive scaffolding, not wired into `Supervisor::run`). Retirement is Phase 1b.

- **The contract is a resink-core-owned, AE-co-authored artifact.** It lives at `teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md`. resink-core owns it because the `nanofab-node-abi` crate (the canonical `NodeCtx` trait) is resink-core's; AE co-authors because AE implements the node side in `lib.rs.tmpl`. It carries the `Verified-against-environment` discipline per ADR-2026-05-16-003 — with the honest note that end-to-end verification lands in Phase 1b.

- **The generic-runner skeleton compiles but is not wired in.** `node_runner.rs`'s `NodeRunner::process_event` bridge call site is `unimplemented!("Phase 1b — NodeCtx C-ABI bridge")`. The skeleton's *pure-Rust* parts ARE implemented: the `DimSchema`-driven struct definitions, `CompositeKey` projection, the partition-key byte encoding, and `write_output`'s schema-driven parquet shaping. The test harness exercises the parts that work.

- **Tenant-isolation invariant maintained.** All changes under `repos/resink-ai/resink-core/` + `repos/resink-ai/resink-marketplace/` (AE's template is probed but not changed this loop) + team OKR/exec files. Zero `org-os/` edits expected.

**Standing CEO answers:**

- **The C-ABI callback contract must specify, concretely:** (1) the `repr(C)` callback-table struct shape behind `ctx_ptr` — function pointers for `get_current` / `close_current` / `append_current` (and whether `already_processed` / `mark_processed` are in-scope for the MVP or stubbed); (2) the key serialization across FFI — how a composite primary key (a `Vec<FieldValue>`) crosses the boundary (recommended: JSON, consistent with the event already crossing as `event_json`); (3) the row serialization — how an `Scd2Row<R>`'s payload crosses; (4) error propagation — how a `NodeError` from a callback maps back through the C-ABI; (5) lifetime + safety contract — `ctx_ptr` is valid only for the duration of one `nanofab_node_process` call, bound to one shard's KV. Concrete enough that AE can implement the node side and resink-core the supervisor side without a second negotiation.

- **`schema.rs` is the spec §3 model verbatim.** `DimSchema { table, columns: Vec<ColumnDesc>, primary_key: Vec<String>, scd2_columns: Scd2ColumnTriple }`, `ColumnDesc { name, ty: ColumnType, nullable }`, `ColumnType` (the five codegen types), `DimSchema::from_json` with validation (non-empty `primary_key`; PK columns present + `nullable: false`; no `Float64` PK column; `scd2_columns` triple present). This is pure Rust, fully implemented + unit-tested this loop.

- **`node_runner.rs` is the spec §2.3 model.** `NodeRunner`, `ShardKv`, `CompositeKey`, `RawFieldValueKey`, `Scd2RowGeneric`, the `NodeCtxBridge` trait. Struct definitions + the pure-Rust methods implemented; `process_event`'s bridge call `unimplemented!`'d. The composite-key byte encoding (§5) IS implemented this loop — it is the highest-risk byte-stability surface and benefits from landing early with unit tests.

- **`tests/general_tables.rs` harness.** A tempdir manifest with three node-specs — `dim_user`, `dim_account`, and a synthetic `dim_widget` (single-column PK, existing field types). The test exercises what works this loop: `DimSchema` parsing + validation for all three, `CompositeKey` projection, the partition-key byte encoding. The end-to-end `Supervisor::run` assertion is written but `#[ignore]`'d with a `Phase 1b` reason string — it becomes the Phase 1b acceptance gate.

- **AE's contract co-authorship is real but light.** AE reads the draft contract, verifies it is implementable against `lib.rs.tmpl`'s `nanofab_node_process` shape (the node must be able to: receive the callback table, call the three callbacks during `process`, propagate `NodeError`), and acks — appending a one-paragraph confirmation to the contract or filing an ack note. AE does NOT change `lib.rs.tmpl` this loop.

**Carryover load by team:**

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| resink-core | O1 — NodeCtx C-ABI callback contract (lead) + `schema.rs` + `node_runner.rs` skeleton + `tests/general_tables.rs` harness | M-L | Contract-first split of Phase 1; the pure-Rust scaffolding lands real code, the bridge waits on Phase 1b. |
| agent-engineering (AE) | O2 — co-author + ack the NodeCtx C-ABI callback contract | S | Light. AE's template change is Phase 1b, not this loop. |
| board | Brief + exec + retro | S | Loop accounting. |
| All other teams | Paused; no review-ack ask | — | Paused. DE / sim-farm / devops / training pipeline are downstream of later phases. |

## Objectives

### O1: NodeCtx C-ABI callback contract + general-tables Phase 1a scaffolding

source: ceo-brief
owner: teams/application/resink-core
sized: M-L
target loop: 2026-05-14-0857 (this loop)
links.source: board/decisions/2026-05-14-001-general-tables-rearchitecture.md

Why it matters: Phase 1 of ADR-2026-05-14-001 cannot land in one loop without an unsafe-FFI callback ABI co-designed across resink-core and AE. The contract is the artifact that de-risks Phase 1b. The pure-Rust scaffolding (`DimSchema`, the `NodeRunner` skeleton, the composite-key byte encoding, the test harness) is everything Phase 1 can land *without* the bridge — landing it now means Phase 1b is purely "wire the bridge + retire `nodes.rs`".

KR1.1: New contract at `teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md`. Frontmatter `type: contract`, `owner: teams/application/resink-core`, `date: 2026-05-14`, `status: active`. Body specifies: the `repr(C)` callback-table struct behind `ctx_ptr`; key + row serialization across FFI; `NodeError` propagation; the `ctx_ptr` lifetime + safety contract (valid for one `process` call, bound to one shard). Carries a `Verified-against-environment` section per ADR-2026-05-16-003 (honest: end-to-end verification is Phase 1b).

KR1.2: New module `crates/nanofab-supervisor/src/schema.rs` — `DimSchema` + `ColumnDesc` + `ColumnType` + `Scd2ColumnTriple` + `DimSchema::from_json` with validation per spec §3.1-3.4. Fully implemented + unit-tested (valid schema parses; non-empty `primary_key` enforced; `Float64` PK rejected; missing `scd2_columns` rejected).

KR1.3: New module `crates/nanofab-supervisor/src/node_runner.rs` — `NodeRunner`, `ShardKv`, `CompositeKey`, `RawFieldValueKey`, `Scd2RowGeneric`, `NodeCtxBridge` trait per spec §2.3. Struct definitions + pure-Rust methods implemented. `NodeRunner::process_event`'s bridge call site is `unimplemented!("Phase 1b — NodeCtx C-ABI bridge")`. The `CompositeKey` projection + the partition-key byte encoding ARE implemented + unit-tested (the byte encoding is the byte-stability-critical surface).

KR1.4: New `crates/nanofab-supervisor/tests/general_tables.rs` — a tempdir manifest with three node-specs (`dim_user`, `dim_account`, synthetic `dim_widget`). Tests that pass this loop: `DimSchema` parse + validate for all three; `CompositeKey` projection; partition-key byte encoding determinism. The end-to-end `Supervisor::run` assertion is written + `#[ignore]`'d with a `Phase 1b` reason — it is the Phase 1b acceptance gate.

KR1.5: `schema.rs` + `node_runner.rs` are declared (in `lib.rs` and/or `main.rs`) so they compile, but are NOT wired into `Supervisor::run`. `nodes.rs` + `supervisor.rs` are untouched. `make mvp-loop` is byte-stable trivially (the live path is unchanged); `cargo test --workspace --release` green including the new tests.

KR1.6: ADR-2026-05-14-001 gains a `## Status (2026-05-14, loop 2026-05-14-0857)` section narrating the Phase 1 re-shape (contract-first split), what the contract settled, what scaffolding landed, and the Phase 1b carryover. The companion spec's §2.4 + §9 are updated to cross-reference the new contract and record the re-shape as a first-contact revision.

KR1.7: Tenant-isolation invariant holds. Zero `org-os/` edits. All code under `repos/resink-ai/resink-core/`; contract + team artifacts under `teams/application/resink-core/`. Final sweep returns the single canonical `acme.ai` placeholder.

### O2: Co-author + ack the NodeCtx C-ABI callback contract

source: ceo-brief
owner: teams/platform/agent-engineering
sized: S
target loop: 2026-05-14-0857 (this loop)
links.source: board/decisions/2026-05-14-001-general-tables-rearchitecture.md

Why it matters: AE owns `lib.rs.tmpl` — the node side of the C-ABI. A contract resink-core writes alone, that AE then finds un-implementable against the template's `nanofab_node_process` shape, is a contract that fails at Phase 1b. AE's co-authorship this loop is the cheap insurance.

KR2.1: AE reviews the draft `2026-05-14-nodectx-cabi-callback.md` and verifies it is implementable against `lib.rs.tmpl`'s `nanofab_node_process(node_ptr, event_json, event_json_len, ctx_ptr) -> i32` shape — specifically that the node can receive the callback table via `ctx_ptr`, invoke `get_current` / `close_current` / `append_current` during `process`, and propagate a `NodeError`.

KR2.2: AE appends a one-paragraph co-author confirmation to the contract (or files an ack note in AE's exec summary cross-team section) naming any shape constraint the template imposes on the contract — e.g. whether the callback table must be passed by pointer vs. value, whether key/row JSON is acceptable vs. a `repr(C)` buffer.

KR2.3: AE's exec summary records the Phase 1b template-change as a named, accepted next-loop deliverable: extend `lib.rs.tmpl`'s `nanofab_node_process` to wire `ctx_ptr` per the ratified contract. Sized estimate included.

KR2.4: Tenant-isolation invariant holds. `lib.rs.tmpl` is NOT changed this loop. AE's edits are limited to the contract file + AE's team OKR/exec.

## What success looks like

- `teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md` exists, `status: active`, AE-co-authored, specifies the `repr(C)` callback table + serialization + error propagation + lifetime contract.
- `crates/nanofab-supervisor/src/schema.rs` + `node_runner.rs` exist, compile, are unit-tested for the pure-Rust surface; the bridge call site is cleanly `unimplemented!`'d for Phase 1b.
- `crates/nanofab-supervisor/tests/general_tables.rs` exists; the three-dim parse/validate/projection tests pass; the end-to-end test is `#[ignore]`'d as the Phase 1b gate.
- `make mvp-loop` byte-stable; `cargo test --workspace --release` green.
- ADR-2026-05-14-001 Status section + spec §2.4/§9 updated; the Phase 1 re-shape is narrated honestly.
- Zero `org-os/` edits; tenant-isolation grep returns the single canonical placeholder.
- Retro surfaces: (i) was the contract-first split the right re-shape, or did it just defer the integration risk by one loop? (ii) did the C-ABI contract settle cleanly, or did AE's co-authorship surface a template constraint that forces a contract revision? (iii) the composite-key byte encoding landed early — is it actually byte-stable against the current single-column key bytes, or is that unverifiable until Phase 1b wires the runner in?

## Out of scope

- Touching `nodes.rs` or `supervisor.rs`. Retirement is Phase 1b.
- Changing `lib.rs.tmpl` (the AE codegen template). AE's template change is Phase 1b.
- Wiring `NodeRunner` into `Supervisor::run`. Phase 1b.
- The `NodeCtxBridge` implementation (the actual FFI callback bridge). Phase 1b — it executes against the contract this loop ratifies.
- `RawFieldValue` type closure (`Float64`, `Timestamp`). Phase 2.
- Streaming arc (ADR-2026-05-13-001 steps 3-4). Deferred carryover.
- The org-os-process backlog (arc-report-type ADR; link-existence smoke; spec-first codification). Deferred to a dedicated org-os-process loop after Phase 1b.
- Any `org-os/` process work.
{% endraw %}
