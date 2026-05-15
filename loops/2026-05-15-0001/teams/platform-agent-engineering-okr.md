---
layout: default
title: platform-agent-engineering OKR — 2026-05-15-0001
date: 2026-05-15
status: active
type: okr
loop: 2026-05-15-0001
owner: teams/platform/agent-engineering
grand_parent: Loops
parent: Loop 2026-05-15-0001
nav_order: 12
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/agent-engineering
  date: 2026-05-15
  status: active
  loop: 2026-05-15-0001
  links: parent: board/okrs/2026-05-15-0001-ceo-brief.md
-->
{% raw %}

# Agent Engineering OKR — 2026-05-15 (loop 2026-05-15-0001)

## Context

Cross-team loop. General-tables **Phase 1b** — AE ships the `lib.rs.tmpl` template change accepted as a named deliverable in loop 2026-05-14-0857's contract co-authorship. The contract (`teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md`) specifies the `#[repr(C)] NanofabNodeCtxVTable` and JSON serialization; this loop AE wires `nanofab_node_process`'s `ctx_ptr` to consume it. The template's standalone smoke test must keep passing via null-`ctx_ptr` fallback to `DefaultProcessCtx`. Resink-core's O1 implements the supervisor side; neither side's deliverable is complete alone — the `make mvp-loop` byte-stability gate + `supervisor_runs_three_dims` are the cross-team acceptance signals.

## Objectives

### O2: Wire `ctx_ptr` in `lib.rs.tmpl` per the ratified contract

source: ceo-brief

Why it matters: AE owns the node side of the C-ABI. Without this template change, every codegen'd cdylib continues ignoring `ctx_ptr` and using `DefaultProcessCtx` — the v-table the supervisor passes is unconsumed, the bridge is unilateral, and the integration fails. The contract's lock-step discipline (`NanofabNodeCtxVTable` field order frozen, mirrored verbatim in `lib.rs.tmpl`) is what AE protects by landing the inline declaration alongside resink-core's authoritative one.

Maps to brief O2 → KR2.1 (template `nanofab_node_process` change), KR2.2 (null-`ctx_ptr` fallback), KR2.3 (`NanofabNodeCtxVTable` inline + field order verbatim), KR2.4 (JSON serialization reuses template machinery), KR2.5 (error mapping + panic containment), KR2.6 (template compiles + smoke test passes), KR2.7 (record any contract revisions), KR2.8 (tenant-isolation; `nanofab-node-abi` untouched).

**Key results**

- KR2.1: `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/lib.rs.tmpl`'s `nanofab_node_process` extended: when `ctx_ptr` is non-null, cast it to `*mut NanofabNodeCtxVTable`, read the three function pointers + `ctx_handle`, build a `CAbiCtxShim` that `impl NodeCtx<K, R>` over them (where `K` / `R` are the per-tenant key/row types slot-filled by the template — the shim's methods serialize `K` / `R` to JSON per the contract, call the v-table function pointers, deserialize the response into `Option<Scd2Row<R>>` / `Result<(), NodeError>`). Pass `&mut shim` to `node.process(ev, &mut shim)` instead of `DefaultProcessCtx::default()`.

- KR2.2: Null-`ctx_ptr` fallback: when `ctx_ptr.is_null()`, fall back to `DefaultProcessCtx::default()`. `DefaultProcessCtx` stays defined (unconditionally or behind `#[cfg(test)]` — whichever is structurally simpler) so the template's standalone smoke test (the `#[cfg(test)]` block at the bottom of `lib.rs.tmpl`) keeps passing without a supervisor v-table present.

- KR2.3: `NanofabNodeCtxVTable` is declared inline in `lib.rs.tmpl` — `#[repr(C)]`, field order **verbatim identical** to resink-core's authoritative declaration in `crates/nanofab-supervisor/src/node_runner.rs`. (Doc comments may be abbreviated; field names + types + order must match.) This is the lock-step discipline the contract imposed; same discipline as the four `NANOFAB_NODE_*` status-code constants.

- KR2.4: `CAbiCtxShim`'s JSON serialization reuses `lib.rs.tmpl`'s existing hand-rolled JSON machinery (the `parse_event_json` style) per the template's zero-external-deps invariant (`Cargo.toml.tmpl` mandates no `serde`). Key JSON / Row JSON shapes match the contract §"Serialization across the boundary" exactly. `append_current`'s `row_json` carries only the payload object — no `Scd2Row` envelope — per the contract.

- KR2.5: Error mapping: a callback `i32 < 0` → `NodeError::StateWriteFailed(...)` (the retryable class). `nanofab_node_process` already maps `NodeError` → the four `NANOFAB_NODE_*` status codes; `StateWriteFailed` → `NANOFAB_NODE_RETRY = 3` per the contract. The shim relies on `nanofab_node_process`'s existing outer `catch_unwind` for panic containment, plus per-callback `catch_unwind` if any callback body does non-trivial work that could panic.

- KR2.6: A slot-filled tenant crate (rendered via `nanofab:codegen-scd2-node` or a template `--dry-run` — whichever AE's tooling supports without a full closed-loop run) compiles. The template's own smoke test (`cargo test` inside a slot-filled crate, or the in-template `#[cfg(test)]` block) passes via the null-`ctx_ptr` fallback path. End-to-end execution via the supervisor v-table is verified by resink-core's O1.KR1.5 + O1.KR1.6 — cross-team acceptance.

- KR2.7: AE's exec summary records any constraint discovered during implementation that the contract under-specified or got wrong. Zero expected (the contract was AE-co-authored; AE confirmed implementability) but the discipline is to flag honestly.

- KR2.8: Tenant-isolation invariant holds. The `nanofab-node-abi` crate is NOT changed this loop (exporting `NanofabNodeCtxVTable` from there is a later cleanup). Only `lib.rs.tmpl` is edited under AE ownership.

**Tasks**

- [ ] Declare `NanofabNodeCtxVTable` `#[repr(C)]` inline in `lib.rs.tmpl`, field order verbatim from resink-core's `node_runner.rs`
- [ ] Implement `CAbiCtxShim<K, R>` (or appropriate slot-filled types) impl `NodeCtx<K, R>` — three method bodies that serialize `K` / `R` via the template's hand-rolled JSON, call the v-table function pointers, deserialize the response
- [ ] Extend `nanofab_node_process`: if `ctx_ptr.is_null()` → `DefaultProcessCtx`; else cast `ctx_ptr` to `*mut NanofabNodeCtxVTable`, build the shim, pass to `node.process(ev, &mut shim)`
- [ ] Error mapping: callback `i32 < 0` → `NodeError::StateWriteFailed` (relies on existing `NodeError` → status-code mapping)
- [ ] Verify a slot-filled tenant crate compiles + the template smoke test passes via null-`ctx_ptr` fallback
- [ ] Cross-check inline `NanofabNodeCtxVTable` field-by-field against resink-core's authoritative declaration before commit
- [ ] Note any contract revision (zero expected) in AE's exec summary

## What success looks like

`lib.rs.tmpl`'s `nanofab_node_process` consumes the supervisor's v-table when `ctx_ptr` is non-null; falls back to `DefaultProcessCtx` when null. `NanofabNodeCtxVTable` is declared inline, field order verbatim from resink-core. A slot-filled tenant crate compiles + smoke-tests pass. End-to-end the supervisor's `make mvp-loop` is byte-stable and `supervisor_runs_three_dims` passes — both proven by resink-core's O1 verification across the cross-team interface AE landed this loop.

## Cross-team dependency on resink-core (O1)

The supervisor side constructs the v-table and passes its pointer through `PluginNode::process(event_json, ctx_ptr)`. If resink-core's `NodeCtxBridge` doesn't land (or its callbacks have a bug), AE's template change still compiles + the smoke test still passes via the fallback path — but end-to-end the bridge does nothing. The cross-team acceptance signals (`make mvp-loop` byte-stable + `supervisor_runs_three_dims`) require both deliverables. The contract's lock-step `NanofabNodeCtxVTable` discipline is the protocol that protects against silent divergence — if resink-core changes the v-table field order, AE mirrors in lock-step.

## Out of scope

- Changing the `nanofab-node-abi` crate. The contract specifies the v-table is mirrored inline in `lib.rs.tmpl` for this loop; exporting it from `nanofab-node-abi` is a later cleanup.
- Phase 2 type-coverage extensions to the codegen template (`Float64` / `Timestamp` / composite-PK template paths). Phase 2 — `loop+1`.
- Any `org-os/` process work (hard-scheduled at `loop+2`).
{% endraw %}
