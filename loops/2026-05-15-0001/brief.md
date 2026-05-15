---
layout: default
title: CEO Brief — 2026-05-15-0001
date: 2026-05-15
status: active
type: okr
loop: 2026-05-15-0001
owner: board
grand_parent: Loops
parent: Loop 2026-05-15-0001
nav_order: 1
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-15
  status: active
  loop: 2026-05-15-0001
  links: parent: ""
-->
{% raw %}

# Resink.ai CEO Brief — 2026-05-15 (loop 2026-05-15-0001)

## Context

First loop on 2026-05-15. **Theme: general-tables Phase 1b — wire the NodeCtx C-ABI bridge across the just-ratified contract.** Phase 1a (loop 2026-05-14-0857) split Phase 1 on a first-contact finding, ratified the cross-team contract, and landed the pure-Rust scaffolding (`schema.rs`, `node_runner.rs` skeleton, the byte-stability-critical partition-key encoding, the 3-dim test harness with `supervisor_runs_three_dims` `#[ignore]`'d as the Phase 1b acceptance gate). **This loop closes Phase 1 of ADR-2026-05-14-001.**

**The contract executing across:** `teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md` (ratified loop 2026-05-14-0857). It specifies a `#[repr(C)] NanofabNodeCtxVTable` behind `ctx_ptr` (function pointers for `get_current` / `close_current` / `append_current` + an opaque `ctx_handle`), JSON serialization of keys + rows, `NodeError` propagation (callback `i32 < 0` → `NodeError::StateWriteFailed` → `NANOFAB_NODE_RETRY = 3`), and the one-`process`-call / one-shard lifetime. AE confirmed implementability against `lib.rs.tmpl`'s shape last loop with **no C-ABI signature change** required (`ctx_ptr` has been a parameter of `nanofab_node_process` since the dlopen scaffold; Phase 1b stops *ignoring* it). The contract is the artifact both Phase 1b deliverables execute against.

**The shape of this loop.** Cross-team, two teams active:

- **AE** ships the `lib.rs.tmpl` change: cast `ctx_ptr` to `*mut NanofabNodeCtxVTable`, build a `CAbiCtxShim` that `impl NodeCtx<K, R>` over the three callbacks, pass it to `node.process(ev, &mut shim)` instead of `DefaultProcessCtx`. `DefaultProcessCtx` stays behind `#[cfg(test)]` + null-`ctx_ptr` fallback so the template's standalone smoke test keeps passing.
- **resink-core** ships the supervisor side: `impl NodeCtxBridge for &mut ShardKv`, wire `NodeRunner::process_event` (build the v-table over a `ShardKv`, serialize the event to `event_json`, call `PluginNode::process(event_json, ctx_ptr)`, emit trace records), extend `PluginNode::process` to accept `ctx_ptr`, hand-update the in-tree `nanofab-plugin-dim-user` cdylib (+ `-v2` hot-swap variant) to mirror the new template, and prove the bridge end-to-end with a new dlopen integration test that exercises the v-table against the dim_user cdylib (asserting the `ShardKv` is mutated through the callbacks).

## In-loop re-shape — Phase 1b → 1b + 1c

**A build-time probe found the brief mis-sized.** Authoring the brief I sized Phase 1b at M+M (contract-implementation across two teams). The probe of `crates/nanofab-supervisor/Cargo.toml` + `synthetic_tenants/closed_loop_v0/Makefile` reveals deeper coupling:

- `supervisor.rs:189-211` unconditionally calls `nodes::user::run` + `nodes::account::run` — not gated by the `dlopen-plugins` feature. Today `make mvp-loop` (and `make mvp-loop-dlopen`!) both go through `nodes.rs`'s typed path-dep crates; `PluginNode::process` is never called in the live path.
- `Cargo.toml`'s `[dependencies]` carries `nanofab_node_dim_user_scd2 = { path = ... }` + `nanofab_node_dim_account_scd2 = { path = ... }` by name. Retiring static-plugins requires removing those (and changing the `default = ["static-plugins"]` feature).
- Only `nanofab-plugin-dim-user` (+ its `-v2` hot-swap variant) exists as a cdylib in-tree. Dim_account has no cdylib equivalent; dim_widget (the third synthetic dim) has none either. Truly retiring `nodes.rs` + having `supervisor_runs_three_dims` execute via real dlopen requires building cdylibs for at least three dims, plus orchestrator changes so the closed_loop_v0 pipeline emits cdylibs into the manifest.

The brief's KR1.4 ("nodes.rs deleted") + KR1.5 ("supervisor_runs_three_dims un-ignored") + KR1.6 ("make mvp-loop byte-stable") together require the Makefile + Cargo.toml + orchestrator changes — genuinely L+L+L, not M+M.

**Re-shape: Phase 1b stays this loop; Phase 1c becomes loop+1.**

- **Phase 1b (this loop) — the contract-implementation work.** AE wires `lib.rs.tmpl` (template change); resink-core implements `NodeCtxBridge for &mut ShardKv`, extends `PluginNode::process` to accept `ctx_ptr`, wires `NodeRunner::process_event` to build the v-table + call the plugin, hand-updates `nanofab-plugin-dim-user` (+ `-v2`) to mirror the new template, and adds a new dlopen integration test that exercises the v-table end-to-end against the dim_user cdylib (the bridge is proven on real FFI, not mocked). `nodes.rs` stays alive; `make mvp-loop` byte-stable **trivially** (the static path is untouched).
- **Phase 1c (loop+1) — the static-plugins retirement.** Build cdylibs for dim_account + dim_widget; update the orchestrator to emit cdylib paths into manifest; switch `make mvp-loop` default to dlopen-plugins; remove `Cargo.toml` path-deps; delete `nodes.rs`; retire `supervisor.rs`'s `match`; un-ignore `supervisor_runs_three_dims` and author its body. The byte-stability gate then proves the generic runner reproduces today's output through the C-ABI.

**Sequencing shift (Phase 2 still `loop+2`; Phase 3 still `loop+3`; the org-os-process loop bumps to `loop+3`).** The sequencing table in the next section is updated below to reflect the shift. Two open arcs + the org-os-process loop are still active under WIP-1; only the calendar moves by one.

**Why narrate this rather than silently re-scope.** The brief-time-probe discipline (memory: `feedback-spec-first-architectural-arc.md`) says: when a probe finds the sizing wrong, the loop re-shapes at the earliest possible point and narrates the finding. The earliest point was at brief authoring (I missed the Cargo.toml + Makefile angle); the next-earliest is here, in-loop, before code lands. The retro at loop close will record the brief-time-probe checklist gap (the brief probed the C-ABI surface but did not probe the build/Cargo/Makefile surface).

**Acceptance signals.** Two:

1. **Byte-stability gate.** `make mvp-loop` is byte-stable — `verdict=pass mismatches=0` on the widened 2-dim / 3-fact / 4-shard closed_loop_v0 fixture. The generic `NodeRunner`'s output parquet bytes match the frozen golden of today's hand-written `nodes::user` / `nodes::account` output. This is the regression-critical surface the partition-key byte encoding was landed early last loop to protect; Phase 1b proves that protection end-to-end.
2. **Generality gate.** `supervisor_runs_three_dims` (un-ignored) passes — the synthetic `dim_widget` runs end-to-end through `Supervisor::run` alongside `dim_user` + `dim_account`, with no supervisor source change to admit it. The third dim is what the hardcoded `match` path could not fake.

Both gates must clear. The byte-stability gate proves nothing existing broke; the generality gate proves the de-hardcoding actually de-hardcoded.

## CEO decisions for this loop

Two standing decisions ratified per retro 2026-05-14-0857 P2 (escalated explicitly to this brief):

- **(a) WIP limit on architectural arcs = 1.** Standing sequencing rule: **at most one architectural arc in active execution at a time; the second waits.** The org currently carries two open arcs (general-tables Phase 1-3; streaming Phases 3-4) plus a thrice-deferred org-os-process backlog plus a parked visual-docs brainstorm. The retro made the call explicit: a soft "recommended slot" is what produced the four+ consecutive deferrals; a hard rule is what the situation now needs. General-tables holds the active slot through Phase 3 closure. Streaming (ADR-2026-05-13-001 steps 3-4) remains **deferred carryover**, reactivated only when general-tables closes or when the CEO explicitly swaps slots in a future brief.

- **(b) The org-os-process loop is hard-scheduled at `loop+2`.** Not "after the next phase" (the formulation that produced four+ deferrals); a specific loop. After Phase 1b closes this loop, `loop+1` is general-tables Phase 2 (richer field types / composite-PK end-to-end / cross-team coordination with DE + AE + sim-farm) per the ADR-2026-05-14-001 plan; **`loop+2` is the org-os-process loop** — a dedicated, no-code, no-arc-work loop whose theme is the org-os-process backlog: (i) draft + ratify the arc-report-type ADR (carried since 2026-05-12-1254); (ii) ship the link-existence smoke for publish-to-gitbook (carried since 2026-05-13-0859); (iii) add spec-first codification to `org-os/conventions.md` per ADR (carried since 2026-05-13-1944). The WIP-1 rule in (a) is what makes `loop+2` reachable instead of perpetually slipped.

Both decisions are **brief-level standing rules**, not new ADRs. (a) edits no `org-os/` doc this loop (per the no-bundling discipline — this is a code loop); the org-os-process loop at `loop+2` is where the WIP-1 rule, if it earns a permanent home, gets codified into `org-os/conventions.md` alongside the spec-first codification. This loop only *records* both decisions in the brief.

**Concrete sequencing implied (post-reshape):**

| `loop+N` | Theme | Class |
|---|---|---|
| `loop+0` (this) | **General-tables Phase 1b** — contract-implementation (cross-team AE + resink-core; the v-table bridge wired end-to-end, proven via dim_user cdylib; `nodes.rs` + static-plugins stay alive for Phase 1c) | code |
| `loop+1` | **General-tables Phase 1c** — static-plugins retirement (cdylibs for dim_account + dim_widget; orchestrator cdylib emission; Makefile flips `make mvp-loop` default to dlopen; `Cargo.toml` path-deps removed; `nodes.rs` deleted; `supervisor.rs` match retired; `supervisor_runs_three_dims` un-ignored. Acceptance signal: `make mvp-loop` byte-stable through the C-ABI) | code |
| `loop+2` | General-tables Phase 2 (richer types + composite PK end-to-end; cross-team DE + AE + sim-farm + resink-core) | code |
| `loop+3` | **Org-os-process loop** (arc-report-type ADR + link-existence smoke + spec-first codification + WIP-rule codification candidate) | org-os |
| `loop+4` | General-tables Phase 3 (complex fact-table shapes; cross-team DE + training pipeline + resink-core) | code |
| `loop+5`+ | General-tables arc closes; streaming arc (ADR-2026-05-13-001 steps 3-4) re-activates under WIP-1 | code |

The sequencing is **a CEO commitment, not a contract** — first-contact findings in any future loop's brief-time probe can reshape it (per the brief-time-probe discipline that worked twice now — and is being applied a third time, in-loop, by the re-shape above).

## Re recent retro carryovers

- **2026-05-14-0857 retro P1 (Phase 1b cross-team):** this loop, executed.
- **2026-05-14-0857 retro P2 (WIP limit + hard-schedule org-os-process):** ratified above as standing CEO decisions (a) + (b). The org-os-process loop is committed to `loop+2`.
- **2026-05-14-0857 retro P3 (reconcile local `master` divergence — the unpushed `035b33e`):** **user owns this; user has elected "leave 035b33e alone" both loops it surfaced.** The workaround is now established: branch every loop's work from `origin/master`, not local `master`. This loop continues that pattern (branch `loop-2026-05-15-0001-general-tables-phase-1b` is off freshly-fetched `origin/master = f07d27c`). The papercut is real but the user's call stands.
- **2026-05-14-0742 retro P2 (sequence two open arcs):** subsumed by CEO decision (a) above; streaming stays deferred under WIP-1.
- **2026-05-14-0742 retro P1 (org-os-process backlog) + P4 (spec-first codification):** subsumed by CEO decision (b) above; both land in `loop+2`'s org-os-process loop.

## Probe results entering the loop

- `git status` clean against `origin/master = f07d27c`; PR #28 (loop-0857 submodule bump) merged loop-close. Working branch `loop-2026-05-15-0001-general-tables-phase-1b` carries the loop-0742 ADR + loop-0857 Phase 1a artifacts in-tree (verified by reading `repos/resink-ai/resink-core/crates/nanofab-supervisor/tests/general_tables.rs`, `src/schema.rs`, `src/lib.rs`, `board/decisions/2026-05-14-001-general-tables-rearchitecture.md`, `teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md`).
- The contract is precise on what each side ships (§"Phase 1b deliverables this contract gates"). No ambiguity entering the build.
- `node_runner.rs` carries the `NodeCtxBridge` trait declaration, the `NanofabNodeCtxVTable` `#[repr(C)]` declaration, `CompositeKey` + projection + byte encoding, `Scd2RowGeneric`, `write_output` — all the pure-Rust scaffolding. Only `process_event`'s bridge call site is `unimplemented!("Phase 1b — NodeCtx C-ABI bridge")`.
- `tests/general_tables.rs` carries `supervisor_runs_three_dims` as the `#[ignore]`'d acceptance gate; un-ignoring is one-line, but the test body is `unimplemented!()` and must be authored.
- Tenant-isolation entering: clean (single canonical `acme.ai` placeholder at `org-os/conventions.md:121`).

## Standing CEO answers

- **The byte-stability gate is non-negotiable.** A generic runner that produces *almost* the same bytes is a Phase 1b failure, not a Phase 1b success. The frozen golden is what `make mvp-loop`'s `verdict=pass mismatches=0` already verifies against; the new path must reproduce it exactly. The single-column string-PK byte encoding landed in Phase 1a (`CompositeKey::encode_bytes` returns bare UTF-8 bytes) is the load-bearing piece; Phase 1b proves it.

- **`nodes.rs` is deleted, not soft-retired.** No `#[allow(dead_code)]` parking, no `pub(crate)` reduction with the module still in tree. The file is removed; `lib.rs` drops `pub(crate) mod nodes;`. The retirement is its own scoped step inside this loop, applied **after** the generic runner is wired in and `make mvp-loop` is green — but before commit, not deferred.

- **`supervisor.rs`'s `match node_spec.table` is deleted, not generalized to "match anything in [user, account, widget]".** The new dispatch iterates `manifest.node_specs`, constructs a `NodeRunner` per node-spec (passing the spec's `DimSchema` + the spec's `PluginNode`), routes events to `runners[event.table].process_event(event, trace_writer)`, and calls `runner.write_output(out_path)` at EndOfStream. The table-name match becomes a hash-map lookup. **No special case for `dim_user` / `dim_account` survives Phase 1b.**

- **`supervisor_runs_three_dims` test body — what it must do.** Build a tempdir manifest with three node-specs (`dim_user`, `dim_account`, `dim_widget`), each with its own `schema.json` (the three schemas already declared in `tests/general_tables.rs` as `DIM_USER` / `DIM_ACCOUNT` / `DIM_WIDGET`). Drive a minimal fact-event stream that touches all three dims. Call `Supervisor::run(config)`. Assert: (i) the run returns Ok; (ii) three output parquets exist (`dim_user.parquet`, `dim_account.parquet`, `dim_widget.parquet`); (iii) each parquet's row count + column shape matches the schema's expected projection; (iv) for `dim_user` + `dim_account`, the output bytes match the frozen golden (or, equivalently, equal what `nodes::user::run` / `nodes::account::run` would have produced for the same input — but `nodes.rs` is deleted, so the golden has to be captured before deletion or recomputed via the generic runner from a known-good input). The test is the loop's *integration* signal beyond `make mvp-loop`'s end-to-end signal.

- **AE's `lib.rs.tmpl` change is template-only.** AE does NOT generate or build per-tenant crates this loop. The change is to the slot-fill template's `nanofab_node_process` body; the template's standalone smoke test (the `#[cfg(test)]` block) must still pass with null `ctx_ptr` (fallback path). The `nanofab-node-abi` crate is NOT changed this loop (per the contract: the v-table is mirrored inline in `lib.rs.tmpl`; exporting it from `nanofab-node-abi` is a later cleanup). The constraint AE imposed on the contract — `NanofabNodeCtxVTable` field order is frozen, mirrored verbatim — must be honored: if resink-core's authoritative `node_runner.rs` declaration drifts from AE's `lib.rs.tmpl` inline copy, the dlopen ABI is broken.

- **The bridge implementation lives in `node_runner.rs`.** `impl NodeCtxBridge for &mut ShardKv` provides the methods the v-table function pointers call. The v-table itself is constructed in `NodeRunner::process_event` (one v-table per `(node, shard, event)` dispatch, per the contract's lifetime spec — or per shard if reusing across events; either is contract-conformant, the implementation picks the simpler one). The callbacks must `catch_unwind` per the contract; panics are caught and returned as negative codes.

- **Trace emission stays correct.** Today `nodes::user::run` / `nodes::account::run` emit Mutation trace records (`kind="close"` / `kind="append"`) keyed by `key_json`. The generic `NodeRunner` must emit identical records — same `key_json` shape (the contract's "Key JSON" format already matches today's trace `key_json` field, per the contract §"Serialization across the boundary"), same `kind` values, same ordering. The trace is consumed by the determinism test (`crates/nanofab-supervisor/tests/determinism.rs`); a trace-shape drift breaks that test silently.

- **Composite primary keys are NOT activated end-to-end this loop.** The runner *supports* them (`CompositeKey` is `Vec<RawFieldValueKey>`); the test harness exercises single-column keys (the `dim_user` / `dim_account` / `dim_widget` schemas all have single-column PKs). Composite-PK end-to-end is **Phase 2's deliverable** — alongside `RawFieldValue` gaining `Float64` + `Timestamp`. Phase 1b proves the architecture admits composite PKs; Phase 2 proves they work end-to-end.

- **ADR-2026-05-14-001 gains a Phase 1b closure Status section.** Narrates what shipped (the v-table wiring + `nodes.rs` retirement + the three-dim end-to-end), any contract revisions discovered during implementation (none expected; flagged if any surface), the byte-stability gate result. Phase 1 closes; the Decision §1-§3 commitments are reified in code. Phase 2 and Phase 3's `loop+N` targets shift by one (already reflected in the sequencing table above).

- **Tenant-isolation invariant.** All code under `repos/resink-ai/resink-core/` + `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/lib.rs.tmpl`. Team artifacts under `teams/application/resink-core/` + `teams/platform/agent-engineering/`. Board artifacts under `board/{okrs,decisions,exec-summaries,retros}/`. Spec updates under `docs/superpowers/specs/`. **Zero `org-os/` edits.** Final sweep returns the single canonical `acme.ai` placeholder.

## Carryover load by team

| Team | Carried items | Sized | Note |
|---|---|---|---|
| resink-core | O1 — impl `NodeCtxBridge for &mut ShardKv`; wire `NodeRunner::process_event`; wire generic runner into `Supervisor::run`; delete `nodes.rs` + `supervisor.rs`'s `match`; un-ignore + author `supervisor_runs_three_dims` body; byte-stability gate green | M | Cross-team primary. Phase 1b closure. |
| agent-engineering (AE) | O2 — extend `lib.rs.tmpl`'s `nanofab_node_process` to wire `ctx_ptr` → `*mut NanofabNodeCtxVTable` + `CAbiCtxShim`; null-`ctx_ptr` fallback to `DefaultProcessCtx` for smoke test | S-M | Cross-team. AE's accepted Phase 1b deliverable from loop 2026-05-14-0857. |
| board | Brief + exec + retro + ADR-2026-05-14-001 Phase 1b closure Status section + spec §9 update | S | Loop accounting + arc-narration. |
| data-engineering (DE) | Paused; no review-ack ask this loop. (DE re-activates in Phase 2 for the schema-JSON convention extension.) | — | Paused. |
| sim-farm | Paused; no ask. (Re-activates in Phase 2 for composite-key + richer-type awareness.) | — | Paused. |
| training pipeline | Paused; no ask. (Re-activates in Phase 3 for richer-fact manifest emission.) | — | Paused. |
| devops + sre | Paused; no ask. | — | Paused. |

## Objectives

### O1: Wire the NodeCtx C-ABI bridge + retire the hardcoded supervisor path

source: ceo-brief
owner: teams/application/resink-core
sized: M
target loop: 2026-05-15-0001 (this loop)
links.source: board/decisions/2026-05-14-001-general-tables-rearchitecture.md
links.contract: teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md

Why it matters: This is Phase 1 closure. Phase 1a ratified the contract and landed the pure-Rust scaffolding; Phase 1b implements across the contract. After this loop the supervisor consumes nodes through one general schema-driven runner — `nodes::user` / `nodes::account` and `match node_spec.table` are gone. The product becomes a genuine general DAG executor (the runtime spec's documented design, reified in code) one phase earlier than the ADR sized for, recovered from the Phase 1 re-shape.

KR1.1: `impl NodeCtxBridge for &mut ShardKv` in `crates/nanofab-supervisor/src/node_runner.rs`. Implements `get_current` / `close_current` / `append_current` per the contract — each callback deserializes the JSON key (and row, for `append_current`), mutates the `ShardKv`, and (for `close_current` / `append_current`) emits a Mutation trace record with `kind="close"` / `kind="append"` and the contract's Key JSON as `key_json`. Each callback body is wrapped in `catch_unwind` per the contract; a caught panic returns a negative code.

KR1.2: `NodeRunner::process_event` (currently `unimplemented!`'d) wired end-to-end. Builds a `NanofabNodeCtxVTable` whose `ctx_handle` points to the routed shard's `ShardKv`, serializes the `RawEvent` to `event_json` per the same shape today's `PluginNode::process` accepts, and calls `PluginNode::process(event_json, ctx_ptr)` where `ctx_ptr` is the `*mut c_void` cast of the v-table pointer. The plugin's returned status code is mapped to `Ok` (NANOFAB_NODE_OK) / a retry-or-fail decision per existing supervisor conventions (NANOFAB_NODE_RETRY → retry; PANIC / other → error). `PluginNode::process`'s signature is extended (or a new method added) to accept `ctx_ptr` — today it passes null.

KR1.3 (re-shaped): **`PluginNode::process` is extended** to accept `ctx_ptr: *mut std::ffi::c_void` (today it unconditionally passes `null_mut()` to `nanofab_node_process`). The supervisor side now has a way to pass the v-table pointer through the safe wrapper. `Supervisor::run`'s rewire to iterate `manifest.node_specs` through `NodeRunner` is **Phase 1c** — this loop the static-plugins path stays as the live `make mvp-loop` driver.

KR1.4 (re-shaped): `crates/nanofab-supervisor/src/nodes.rs` **stays alive this loop**. Re-shape: `nodes.rs` retires in Phase 1c when `make mvp-loop` switches to dlopen-plugins. `nodes.rs` gains an updated module-comment cross-referencing the contract + naming Phase 1c as its retirement loop. **The end-state is unchanged** — `nodes.rs` is genuinely retired, one loop later than the brief originally sized.

KR1.5 (re-shaped): `crates/nanofab-supervisor/tests/general_tables.rs`'s `supervisor_runs_three_dims` **stays `#[ignore]`'d this loop with an updated reason naming Phase 1c as its gate** (the body cannot author cleanly without cdylibs for dim_account + dim_widget, which Phase 1c builds). **In its place this loop, a new test in `crates/nanofab-supervisor/tests/dlopen_integration.rs`** (`bridge_v_table_round_trips_dim_user_insert` or a similarly named test) proves the v-table bridge works end-to-end via real FFI against the existing `nanofab-plugin-dim-user` cdylib: drive a `dim_user` insert event through `NodeRunner::process_event`, assert the supervisor-side `ShardKv` is mutated (one current row; the correct payload; `valid_from = event_ts`); for the multi-event case, an update closes the current row + appends the new one. This is the load-bearing Phase 1b acceptance signal — the bridge is proven on real FFI, not mocked.

KR1.6 (re-shaped, trivially satisfied): **`make mvp-loop` byte-stable gate.** `cd repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0 && make mvp-loop` exits 0 with `verdict=pass mismatches=0`. Trivially satisfied this loop because `nodes.rs` + `supervisor.rs`'s live path are untouched — the static-plugins path doesn't go near the v-table. The **genuine byte-stability gate** (output via the C-ABI matches today's golden) is Phase 1c's gate; this loop the gate is "we didn't break what we didn't touch."

KR1.6.5 (new this re-shape): **`nanofab-plugin-dim-user` (+ `-v2` hot-swap variant) hand-updated to mirror the new `lib.rs.tmpl`.** Both in-tree cdylibs are hand-slot-fills of the template; AE's template change must propagate by hand to both (the orchestrator regenerates `workspace/nodes/dim_user_scd2/src/lib.rs` automatically on every `make mvp-loop`, but the two in-tree cdylibs are checked-in artifacts). The existing `dlopen_integration` + `hot_swap_correctness` tests stay green under the new template (null-`ctx_ptr` fallback covers them); the new bridge-roundtrip test exercises the v-table.

KR1.7: `cargo test --workspace --release` is green. The new bridge-roundtrip test passes under `--features dlopen-plugins`. The existing `dlopen_integration` + `hot_swap_correctness` tests stay green (null-`ctx_ptr` fallback). `general_tables::supervisor_runs_three_dims` stays `#[ignore]`'d (Phase 1c gate).

KR1.8: ADR-2026-05-14-001 gains a `## Status — 2026-05-15, loop 2026-05-15-0001 — Phase 1b shipped + 1c carved out` section. Narrates: the v-table bridge wired both sides + proven on real FFI; the brief-time-probe-miss finding that surfaced Phase 1c; the static-plugins retirement now owns Phase 1c (`loop+1`). Spec §9 implementation-loop plan is updated to add Phase 1c and bump Phase 2 + Phase 3 + the org-os-process loop by one.

KR1.9: Tenant-isolation invariant holds. Final sweep `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns the single canonical line at `org-os/conventions.md:121`. Zero `org-os/` edits this loop.

### O2: Wire `ctx_ptr` in `lib.rs.tmpl` per the ratified contract

source: ceo-brief
owner: teams/platform/agent-engineering
sized: S-M
target loop: 2026-05-15-0001 (this loop)
links.source: board/decisions/2026-05-14-001-general-tables-rearchitecture.md
links.contract: teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md

Why it matters: AE owns the node side of the C-ABI. The contract ratified loop 2026-05-14-0857 specifies the v-table; AE accepted this template change as the Phase 1b deliverable. Without it, `PluginNode::process(event_json, ctx_ptr)` from O1 has a v-table pointer that the loaded cdylib still ignores — the bridge is unilateral, the integration fails.

KR2.1: `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/lib.rs.tmpl`'s `nanofab_node_process` is extended: when `ctx_ptr` is non-null, cast it to `*mut NanofabNodeCtxVTable`, read the three function pointers + `ctx_handle`, build a `CAbiCtxShim` that `impl NodeCtx<K, R>` over them (where `K` / `R` are the per-tenant key/row types the template's slot-fill substitutes — the shim's methods serialize `K` / `R` to JSON per the contract's Key JSON / Row JSON shapes, call the v-table function pointers, and deserialize the response). Pass `&mut shim` to `node.process(ev, &mut shim)` instead of `DefaultProcessCtx::default()`.

KR2.2: Null-`ctx_ptr` fallback: when `ctx_ptr.is_null()`, fall back to `DefaultProcessCtx::default()` (today's path). `DefaultProcessCtx` stays defined behind `#[cfg(test)]` or unconditionally — whichever is structurally simpler — so the template's standalone smoke test (the `#[cfg(test)]` block at the bottom of `lib.rs.tmpl`) keeps passing without a supervisor v-table present.

KR2.3: `NanofabNodeCtxVTable` is declared inline in `lib.rs.tmpl` — `#[repr(C)]`, field order **verbatim identical** to resink-core's authoritative declaration in `node_runner.rs` (per the constraint AE imposed on the contract). If resink-core's declaration carries doc-comments, AE may abbreviate them but must not reorder fields. This is the lock-step discipline that protects the dlopen ABI (same discipline as the four `NANOFAB_NODE_*` status-code constants).

KR2.4: The `CAbiCtxShim`'s JSON serialization reuses `lib.rs.tmpl`'s existing hand-rolled JSON machinery (the `parse_event_json` style) per the template's zero-external-deps invariant (`Cargo.toml.tmpl` mandates no `serde`). Key JSON / Row JSON shapes match the contract §"Serialization across the boundary" exactly. `append_current`'s `row_json` carries only the payload object (no Scd2Row envelope), per the contract.

KR2.5: Error mapping: a callback `i32 < 0` is mapped to `NodeError::StateWriteFailed(...)` (the retryable class). The shim wraps each callback in `catch_unwind` (or relies on `nanofab_node_process`'s outer `catch_unwind` — whichever is structurally cleaner) so a callback-side panic is contained.

KR2.6: A regenerated tenant crate (via `nanofab:codegen-scd2-node` slot-fill or a `--dry-run` template render — whichever AE's tooling supports without requiring a full closed-loop run) compiles. The template's own smoke test (`cargo test` inside a slot-filled crate) passes via the null-`ctx_ptr` fallback path. End-to-end execution via the supervisor v-table is verified by resink-core's O1.KR1.5 + O1.KR1.6 (cross-team acceptance — neither team's deliverable is complete alone).

KR2.7: AE's exec summary records any constraint discovered during implementation that the contract under-specified or got wrong. (Zero expected — the contract was AE-co-authored and AE confirmed implementability — but the discipline is to flag honestly.)

KR2.8: Tenant-isolation invariant holds. `nanofab-node-abi` crate is NOT changed this loop. Only `lib.rs.tmpl` is edited under AE ownership.

## What success looks like (post-reshape)

- AE's `lib.rs.tmpl` wires `ctx_ptr` → `*mut NanofabNodeCtxVTable` + `CAbiCtxShim`; null-`ctx_ptr` fallback preserves the template smoke test.
- resink-core's `NodeCtxBridge for &mut ShardKv` is implemented; `NodeRunner::process_event` is wired end-to-end (build v-table, serialize event JSON, call `PluginNode::process(event_json, ctx_ptr)`, map status, emit trace).
- `PluginNode::process` extended to accept `ctx_ptr`.
- `nanofab-plugin-dim-user` + `-v2` cdylibs hand-updated to mirror the new template; existing dlopen + hot-swap tests stay green under null-`ctx_ptr` fallback.
- A new dlopen integration test proves the v-table bridge round-trips against the dim_user cdylib (the `ShardKv` is mutated by the node's process via the v-table callbacks).
- `make mvp-loop` exits 0 with `verdict=pass mismatches=0` — trivially (static path untouched).
- `cargo test --workspace --release` green; `cargo test --workspace --release --features dlopen-plugins` green incl. the new bridge-roundtrip test.
- ADR-2026-05-14-001 Phase 1b closure Status section + Phase 1c carve-out narrated; spec §9 updated to add Phase 1c and bump Phase 2/3/org-os-process by one loop.
- Two CEO decisions (WIP=1 + org-os-process at `loop+3` post-reshape) recorded in this brief and carried into the retro + exec summary.
- Zero `org-os/` edits; tenant-isolation grep returns the single canonical placeholder.
- Retro surfaces: (i) the brief-time-probe miss (Cargo.toml + Makefile not probed) — what checklist closes that gap? (ii) the in-loop re-shape — did surfacing it mid-build vs at brief-authoring cost the loop anything material? (iii) the bridge proved on real FFI — did the contract need any first-contact revisions when implemented across, or did the contract-first split fully de-risk it? (iv) is the Phase 1c sequencing right, or should the static-plugins retirement bundle into Phase 2's cross-team coordination loop instead?

## Out of scope

- **Phase 2 work.** `RawFieldValue`'s widening to `Float64` + `Timestamp`. Composite-PK end-to-end exercise. DE's schema-JSON convention extension. AE's codegen-template type-coverage. Sim-farm's composite-key + richer-type engine work. All Phase 2 — `loop+1`.
- **Phase 3 work.** Complex fact-table shapes (multi-fact-stream-to-one-dim, fact-table fan-out, derived/joined facts). DE's fact-stream contract extension. Training pipeline's richer-fact manifest. All Phase 3 — `loop+3`.
- **Streaming arc (ADR-2026-05-13-001 steps 3-4).** Deferred carryover under WIP-1. Re-activates after general-tables closes (or when the CEO explicitly swaps slots in a future brief).
- **Org-os-process backlog.** Hard-scheduled at `loop+2`. Not this loop.
- **Visual-docs brainstorm.** Parked at design Section 1. Subject to WIP-1; reactivates when no architectural arc is active or when the CEO explicitly slots it.
- **`nanofab-node-abi` crate changes.** The contract specifies the v-table is mirrored inline in `lib.rs.tmpl` for this loop; exporting `NanofabNodeCtxVTable` from `nanofab-node-abi` is a later cleanup.
- **`PluginNode::process` signature stabilization.** Whatever shape O1.KR1.2 lands (extended signature vs new method) is acceptable; a future arc may revisit it.
- **Reconciling local `master` divergence (`035b33e`).** User-owned; user has chosen "leave it alone" twice. The branch-from-origin/master workaround stands.
- **resink-marketplace SKILL.md edit (the loop-close-conventions addition from a prior session, still uncommitted).** Pending user review; not gated on this loop. If user reviews + approves during this loop, it ships alongside; if not, it carries.
- **Any `org-os/` process work.** Reserved for `loop+2`.
{% endraw %}
