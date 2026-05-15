---
layout: default
title: application-resink-core OKR — 2026-05-14-0857
date: 2026-05-14
status: active
type: okr
loop: 2026-05-14-0857
owner: teams/application/resink-core
grand_parent: Loops
parent: Loop 2026-05-14-0857
nav_order: 10
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-14
  status: active
  loop: 2026-05-14-0857
  links: parent: board/okrs/2026-05-14-0857-ceo-brief.md
-->
{% raw %}

# Resink Core OKR — 2026-05-14 (loop 2026-05-14-0857)

## Context

General-tables Phase 1, re-shaped to a contract-first split after a first-contact finding: the ADR's dlopen-C-ABI Phase 1 needs an AE codegen-template change to wire `ctx_ptr`, so it cannot land solo in one loop. This loop resink-core leads the **NodeCtx C-ABI callback contract** (AE co-authors) and lands the pure-Rust scaffolding that doesn't depend on the bridge. `nodes.rs` / `supervisor.rs` are untouched; `make mvp-loop` is byte-stable trivially.

## Objectives

### O1: NodeCtx C-ABI callback contract + general-tables Phase 1a scaffolding

source: ceo-brief

Why it matters: The contract de-risks Phase 1b — co-designing an unsafe-FFI callback ABI without a contract first is the classic integration failure. The pure-Rust scaffolding is everything Phase 1 can land without the bridge; landing it now means Phase 1b is purely "wire the bridge + retire `nodes.rs`".

Maps to brief O1 → KR1.1 (contract), KR1.2 (`schema.rs`), KR1.3 (`node_runner.rs` skeleton), KR1.4 (`tests/general_tables.rs`), KR1.5 (compile + mvp-loop byte-stable + cargo test green), KR1.6 (ADR Status + spec update), KR1.7 (tenant-isolation).

**Key results**

- KR1.1: `teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md` — `type: contract`, AE-co-authored. Specifies the `repr(C)` callback table behind `ctx_ptr` (fn pointers for `get_current` / `close_current` / `append_current`; `already_processed` / `mark_processed` MVP-stubbed or in-scope — contract decides); key + row JSON serialization across FFI; `NodeError` → C-ABI error propagation; `ctx_ptr` lifetime (one `process` call, one shard). `Verified-against-environment` section per ADR-2026-05-16-003 (honest: end-to-end verification is Phase 1b).
- KR1.2: `crates/nanofab-supervisor/src/schema.rs` — `DimSchema` + `ColumnDesc` + `ColumnType` (5 codegen types) + `Scd2ColumnTriple` + `DimSchema::from_json` with validation per spec §3.1-3.4. Fully implemented + unit-tested.
- KR1.3: `crates/nanofab-supervisor/src/node_runner.rs` — `NodeRunner`, `ShardKv`, `CompositeKey`, `RawFieldValueKey`, `Scd2RowGeneric`, `NodeCtxBridge` trait per spec §2.3. Struct defs + pure-Rust methods implemented; `process_event` bridge call site `unimplemented!("Phase 1b — NodeCtx C-ABI bridge")`. `CompositeKey` projection + partition-key byte encoding implemented + unit-tested.
- KR1.4: `crates/nanofab-supervisor/tests/general_tables.rs` — tempdir manifest with 3 node-specs (`dim_user`, `dim_account`, synthetic `dim_widget`). Passing this loop: `DimSchema` parse + validate ×3, `CompositeKey` projection, partition-key byte-encoding determinism. End-to-end `Supervisor::run` assertion written + `#[ignore]`'d (`Phase 1b` reason) — the Phase 1b gate.
- KR1.5: `schema.rs` + `node_runner.rs` declared in `lib.rs` (compile) but NOT wired into `Supervisor::run`. `nodes.rs` + `supervisor.rs` untouched. `make mvp-loop` `verdict=pass mismatches=0` byte-stable; `cargo test --workspace --release` green incl. new tests.
- KR1.6: ADR-2026-05-14-001 gains `## Status (2026-05-14, loop 2026-05-14-0857)` narrating the contract-first re-shape. Spec §2.4 + §9 cross-reference the new contract.
- KR1.7: Tenant-isolation invariant holds; zero `org-os/` edits.

**Tasks**

- [ ] Draft `contracts/2026-05-14-nodectx-cabi-callback.md` — the `repr(C)` callback table, FFI serialization, error propagation, lifetime contract; hand to AE for co-author review
- [ ] Author `crates/nanofab-supervisor/src/schema.rs` — `DimSchema` model + `from_json` validation; unit tests
- [ ] Author `crates/nanofab-supervisor/src/node_runner.rs` — skeleton structs + pure-Rust methods (`CompositeKey` projection, partition-key byte encoding, `write_output` schema-driven shaping); `process_event` bridge call `unimplemented!`'d; unit tests for the implemented surface
- [ ] Declare `pub mod schema;` + `pub mod node_runner;` in `crates/nanofab-supervisor/src/lib.rs`
- [ ] Author `crates/nanofab-supervisor/tests/general_tables.rs` — 3-dim tempdir manifest; parse/validate/projection tests pass; end-to-end test `#[ignore]`'d
- [ ] Fold AE's contract co-author feedback into the contract; mark `status: active`
- [ ] `cargo test --workspace --release` green; `make mvp-loop` byte-stable
- [ ] Append `## Status (2026-05-14, loop 2026-05-14-0857)` to ADR-2026-05-14-001; cross-reference the contract in spec §2.4 + §9

## What success looks like

The NodeCtx C-ABI callback contract is ratified + AE-co-authored. `schema.rs` + `node_runner.rs` scaffolding compiles + is unit-tested; the bridge call site is cleanly stubbed for Phase 1b. The 3-dim test harness exists. `make mvp-loop` byte-stable. Phase 1b is now purely "AE wires `ctx_ptr` + resink-core implements `NodeCtxBridge` + retires `nodes.rs`".

## Out of scope

- Touching `nodes.rs` / `supervisor.rs` (Phase 1b).
- The `NodeCtxBridge` implementation (Phase 1b — executes against this loop's contract).
- Wiring `NodeRunner` into `Supervisor::run` (Phase 1b).
- `RawFieldValue` type closure (Phase 2).
{% endraw %}
