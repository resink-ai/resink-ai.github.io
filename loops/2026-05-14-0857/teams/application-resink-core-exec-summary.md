---
layout: default
title: application-resink-core Exec Summary — 2026-05-14-0857
date: 2026-05-14
status: active
type: exec-summary
loop: 2026-05-14-0857
owner: teams/application/resink-core
grand_parent: Loops
parent: Loop 2026-05-14-0857
nav_order: 11
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/resink-core
  date: 2026-05-14
  status: active
  loop: 2026-05-14-0857
  links: parent: board/exec-summaries/2026-05-14-0857.md
-->
{% raw %}

# Resink Core Exec Summary — 2026-05-14 (loop 2026-05-14-0857)

**Headline.** General-tables Phase 1a shipped — the contract-first split half of Phase 1, after a first-contact probe found Phase 1 needs an AE codegen-template change and so cannot land solo. resink-core authored the **NodeCtx C-ABI callback contract** (AE-co-authored), and landed the pure-Rust scaffolding: `schema.rs` (`DimSchema` + validation), `node_runner.rs` (the generic `NodeRunner` skeleton + `CompositeKey` + the byte-stability-critical partition-key encoding + schema-driven `write_output`), and `tests/general_tables.rs` (a third synthetic dim harness). `nodes.rs` / `supervisor.rs` untouched; `make mvp-loop` byte-stable; `cargo test --workspace --release` green (27 unit + 3 new integration tests). One first-contact spec revision narrated. Tenant-isolation invariant CLEAN.

## Per-objective rollup

### O1: NodeCtx C-ABI callback contract + general-tables Phase 1a scaffolding — ✅ PASS

All 7 KRs cleared.

- **KR1.1 (contract): PASS.** `teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md`, `type: contract`, `status: active`. Specifies the `#[repr(C)] NanofabNodeCtxVTable` (function pointers for `get_current` / `close_current` / `append_current` + opaque `ctx_handle`), JSON key + row serialization across FFI, `NodeError` propagation (callback `i32 < 0` → `NodeError::StateWriteFailed` → `NANOFAB_NODE_RETRY`), and the `ctx_ptr` lifetime contract (one `process` call, one shard). AE co-authored — confirmation paragraph appended, with the `Verified-against-environment` section honest that end-to-end verification is Phase 1b.
- **KR1.2 (`schema.rs`): PASS.** `DimSchema` + `ColumnDesc` + `ColumnType` (the five codegen types) + `Scd2ColumnTriple` + `DimSchema::from_json` with validation per spec §3.1-3.4. 7 unit tests: valid single + composite-PK schemas parse; empty `primary_key`, missing PK column, nullable PK column, `Float64` PK column, non-distinct SCD2 columns all rejected.
- **KR1.3 (`node_runner.rs`): PASS.** `NodeRunner`, `ShardKv`, `CompositeKey`, `RawFieldValueKey`, `Scd2RowGeneric`, the `NodeCtxBridge` trait, and the `NanofabNodeCtxVTable` declaration. Pure-Rust implemented: `CompositeKey::project`, `encode_bytes` (the partition-key byte encoding), `to_key_json`, `NodeRunner::shard_for`, `write_output` (schema-driven parquet shaping). `process_event`'s bridge call site is `unimplemented!("Phase 1b — NodeCtx C-ABI bridge")` *after* the real, testable routing decision (key projection + shard computation). 6 unit tests including the byte-stability assertion: a single-column string key encodes to exactly its bare UTF-8 bytes.
- **KR1.4 (`tests/general_tables.rs`): PASS.** Three synthetic dims (`dim_user`, `dim_account`, `dim_widget`). 3 tests pass this loop: three-dim parse/validate; composite-key projection for the third dim; cross-dim encoding determinism. `supervisor_runs_three_dims` is `#[ignore]`'d with a `Phase 1b` reason — the Phase 1b acceptance gate.
- **KR1.5 (compile + mvp-loop + cargo test): PASS.** `schema.rs` + `node_runner.rs` declared in `lib.rs` (`pub mod`), compile clean, NOT wired into `Supervisor::run`. `nodes.rs` + `supervisor.rs` untouched. `make mvp-loop` `verdict=pass mismatches=0` — byte-stable trivially (the live `static-plugins` path is unchanged). `cargo test --workspace --release` green.
- **KR1.6 (ADR Status + spec update): PASS.** ADR-2026-05-14-001 gained the `## Status (2026-05-14, loop 2026-05-14-0857)` section narrating the contract-first re-shape, Phase 1a deliverables, the first-contact spec revision, and the Phase 1b carryover. Spec §2.4 + §9 updated: §2.4 gets a "wire shape ratified" note pointing at the contract; §9's implementation-loop plan splits Phase 1 into 1a (shipped) + 1b.
- **KR1.7 (tenant-isolation): PASS.** Zero `org-os/` edits. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121`.

## Build phase notes

**One first-contact spec revision, narrated:** `RawFieldValueKey` ships as `String | Int64 | Bool` only. Spec §2.3 sketched a `Timestamp` variant, but the supervisor's `RawFieldValue` carries no timestamp until Phase 2 — a `Timestamp` variant would be unconstructible dead code today. It joins `RawFieldValueKey` in Phase 2 alongside `RawFieldValue`'s widening. Narrated in the ADR Status section.

**Byte-stability design call (highest-risk surface, landed early on purpose):** the partition-key byte encoding. The current per-table partitioner does `partition(key_val.as_bytes(), shard_count)` where `key_val` is the single `String` PK value. The generic `CompositeKey::encode_bytes` therefore special-cases the len-1 path to return *exactly* that value's bare UTF-8 bytes — no separator, no length prefix — so a single-column key partitions byte-identically to the current code. Composite keys (len > 1) use length-prefixed concatenation (`u64`-LE length + bytes per part). The byte-stability assertion is a unit test this loop; the end-to-end byte-stability proof against the golden output is Phase 1b (when the runner is actually wired).

**`write_output` implemented but unexercised.** It is pure Rust (no FFI), so it landed this loop per the brief — schema-driven column layout (PK columns in `primary_key` order → payload columns in `payload_columns()` order → the SCD2 triple), rows sorted by `(key bytes, valid_from)`. It cannot be exercised end-to-end until `process_event` populates the shards (Phase 1b). It is the most likely Phase 1b first-contact revision site on the resink-core side.

**Build artifacts:**

- New: `crates/nanofab-supervisor/src/schema.rs` (~210 lines incl. 7 tests).
- New: `crates/nanofab-supervisor/src/node_runner.rs` (~400 lines incl. 6 tests).
- New: `crates/nanofab-supervisor/tests/general_tables.rs` (~150 lines, 4 tests).
- Modified: `crates/nanofab-supervisor/src/lib.rs` (added `pub mod schema;` + `pub mod node_runner;`).
- New: `teams/application/resink-core/contracts/` (first contract for the team) + `2026-05-14-nodectx-cabi-callback.md`.

## What didn't / risks for Phase 1b

- **The contract is unverified end-to-end.** AE confirmed implementability against `lib.rs.tmpl`'s shape, but no code crosses the C-ABI this loop. The first real test of the v-table is Phase 1b — the contract may need a revision once both sides implement against it. The contract's `Verified-against-environment` section is honest about this.
- **`write_output` is pure-Rust-but-unexercised.** It will first run in Phase 1b. If the schema-driven column layout doesn't reproduce the current per-table writers' byte layout exactly, Phase 1b's byte-stability gate catches it — but that is a Phase 1b debugging surface, not a Phase 1a one.
- **The composite-key byte encoding's stability is asserted at the unit level, not end-to-end.** The unit test proves a single-column string key → bare bytes. The proof that this produces byte-identical `make mvp-loop` output only lands when `process_event` is wired (Phase 1b).

## Asks for the CEO / next loop's brief

- **(from resink-core, next loop — Phase 1b):** Cross-team with AE. AE wires `lib.rs.tmpl`'s `ctx_ptr` per the contract; resink-core implements `NodeCtxBridge for &mut ShardKv`, wires `NodeRunner::process_event` + the generic runner into `Supervisor::run`, retires `nodes.rs`'s `mod user`/`mod account` + `supervisor.rs`'s `match node_spec.table`, and un-ignores `supervisor_runs_three_dims`. Sized M. The byte-stability gate (generic runner output == frozen golden of the current hand-written output) is the load-bearing acceptance signal.

## Tenant-isolation invariant

Held throughout. Zero `org-os/` writes. Single canonical `acme.ai` placeholder at `org-os/conventions.md:121`.

All code under `repos/resink-ai/resink-core/crates/nanofab-supervisor/`; the contract + team artifacts under `teams/application/resink-core/`. None under `org-os/`.
{% endraw %}
