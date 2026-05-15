---
layout: default
title: "ADR 2026-05-14-001: general tables rearchitecture"
date: 2026-05-14
status: active
type: adr
owner: board
parent: Decisions (ADRs)
nav_order: 11
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-14
  status: active
  decision: Name the supervisor's hardcoded dim_user/dim_account handling as a deliberate MVP deviation from the runtime spec's general-DAG-executor design; restore generality via a generic schema-driven node runner over a three-phase, multi-loop plan.
  links: parent: board/okrs/2026-05-14-0742-ceo-brief.md
-->
{% raw %}

# ADR-2026-05-14-001 — General-tables rearchitecture

## Context

The runtime spec (`docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md`) describes the supervisor as a **general DAG executor** — it runs "a customer's generated DAG of dim/ADS-table maintainers." A customer arrives with their own schema; the training pipeline turns it into a DAG; the supervisor executes whatever that DAG contains. Nothing in the documented design is specific to any particular table.

The MVP supervisor is not general. It hardcodes exactly two dim tables:

- `crates/nanofab-supervisor/src/nodes.rs` (486 lines) is two hand-written modules — `pub mod user` (lines 33-258) and `pub mod account` (lines 265-486). Each carries its own `raw_to_*_event` adapter, its own hardcoded primary-key column (`user_id` / `account_id`), its own hardcoded output-parquet column shape, its own `ShardKv` struct, and its own `run()` function. The two modules are ~90% structurally identical and ~10% schema-specific — but the schema-specific 10% is woven through hand-written code, not parameterized.
- `crates/nanofab-supervisor/src/supervisor.rs` dispatches with `match node_spec.table.as_str() { "dim_user" | "dim_account" => {} other => Err(...) }`, then `buffers.remove("dim_user")` / `buffers.remove("dim_account")`, then literal `nodes::user::run(...)` / `nodes::account::run(...)` calls. A node-spec whose `table` is anything else is a hard error.

Adding a third dim table today means hand-writing a third `nodes::` module, adding a third `match` arm, and adding a third `run()` call site. The product cannot ingest an arbitrary customer schema without a supervisor source change — which defeats the entire training-pipeline → codegen → deploy story.

The deviation is **localized**: the pieces *around* the supervisor are already schema-driven.

- The `codegen-scd2-node` skill takes a `(schema_json, pattern_name, output_dir)` triple and slot-fills a per-dim Cargo crate. Its schema accepts five column types (`string, int64, float64, bool, timestamp_ms`), a `primary_key` that is "a non-empty list of strings" (composite PKs are spec'd), and an `scd2_columns` triple. The codegen is general.
- The orchestrator emits a `manifest.yaml` with a per-node schema and `fact_streams` list.
- Sim-farm's diff engine 0.3.0 is schema-aware via `--schema-ref` and a per-dim `schema.json`.
- DE owns a `{key_columns, payload_columns}` schema-JSON convention (`teams/platform/data-engineering/conventions/dim-schema-json.md`) — "DE owns the data-shape vocabulary."

So the generality bottleneck is precisely **the supervisor consuming nodes through hand-written per-table adapters** — `nodes.rs` + the `match` in `supervisor.rs`. The codegen produces general per-dim crates; the supervisor refuses to run anything but two named ones.

Two adjacent gaps compound the deviation and belong in the same arc:

- **`RawFieldValue` cannot represent two of the five codegen types.** `crates/nanofab-supervisor/src/event_source/raw.rs` defines `RawFieldValue { String, Int64, Bool, Null }`. The codegen schema accepts `float64` and `timestamp_ms` — the supervisor has no variant for either, and `arrow_value_to_field` downcasts only `StringArray` / `Int64Array` / `BooleanArray`. A customer schema with a price column or an event-time column cannot round-trip through the supervisor today.
- **Composite primary keys are spec'd but not honored.** The codegen schema's `primary_key` is a list; `nodes::user` / `nodes::account` hardcode single-column extraction (`ev.after.get("user_id")`, `ev.after.get("account_id")`). A dim with a two-column natural key has no path through the supervisor.

This ADR names the two-table hardcoding (and the two adjacent gaps) as a **deliberate, time-boxed MVP deviation** and commits to a three-phase restoration plan. It follows the named-deviation + multi-loop-restoration pattern proven by ADR-2026-05-16-001 (dlopen, closed across five loops) and ADR-2026-05-13-001 (streaming event-source, steps 1-2 shipped).

## Decision

**Restore generality via a generic, schema-driven node runner that consumes nodes through the `Node` C-ABI rather than through hand-written per-table adapters.**

The end state:

1. **`nodes.rs`'s `mod user` / `mod account` retire.** In their place: one generic node runner that, given a node-spec's schema, builds a per-dim handler — primary-key extraction driven by the schema's `primary_key` list, output-parquet column shape driven by the schema's columns, SCD2 column triple driven by the schema's `scd2_columns`. The ~90% structurally-identical code becomes the runner; the ~10% schema-specific code becomes runtime parameters.
2. **`supervisor.rs`'s `match node_spec.table` retires.** The supervisor iterates the manifest's node-specs and dispatches each through the generic runner. An arbitrary set of dim tables — three, five, fifty — runs without a supervisor source change.
3. **The dlopen `PluginNode` path (ADR-2026-05-16-001) is the vehicle.** The supervisor already has a safe `PluginNode` wrapper that loads a node's cdylib and invokes it through the C-ABI. The generic runner consumes nodes through that surface — the `Node` trait, not a hand-written `raw_to_*_event` adapter. This subsumes ADR-2026-05-16-001's deferred "step 3.5 — supervisor-side NodeCtx bridge": the bridge *is* the generic runner's connection to the node.
4. **`RawFieldValue` gains `Float64` + `Timestamp`**, closing the gap against the codegen's five accepted types; `arrow_value_to_field` is extended to match.
5. **Composite primary keys flow through the generic runner** — the `primary_key` list, used as a list, with len > 1 handled natively.
6. **Complex fact-table shapes** — multiple fact streams feeding one dim with mixed ops (partially supported today), fact tables that fan out to multiple dims, derived / joined facts — are modeled so the supervisor's fact routing is as general as its dim handling.

The restoration is sequenced as **three phases**, each its own executing loop (or loops), each closing named spec sections by code per the ADR-2026-05-13-001 convention:

### Phase 1 — De-hardcode the supervisor's table dispatch — resink-core; sized M; target `loop+1`

Retire `nodes::user` / `nodes::account` and the `match` on table name. Ship the generic schema-driven node runner: the supervisor reads each node-spec's schema, builds a generic per-dim handler, routes `RawEvent`s to it through the `Node` C-ABI. **Acceptance signal: a *third* synthetic dim table runs through `make mvp-loop` with no supervisor source change**, and the existing two-dim fixture stays byte-stable. Phase 1 is load-bearing — Phases 2 and 3 both assume a generic runner exists to extend.

### Phase 2 — Richer schemas + field types — resink-core + DE + AE + sim-farm; sized M-L; target `loop+2` / `loop+3`

`RawFieldValue` gains `Float64` + `Timestamp`; `arrow_value_to_field` handles them. Composite primary keys flow through Phase 1's runner (the `primary_key` list with len > 1). DE extends the schema-JSON convention with richer column descriptors. AE extends the codegen template to cover all five types + composite PKs. Sim-farm's engine gains composite-key awareness. **Acceptance signal: a synthetic dim with a `float64` column, a `timestamp_ms` column, and a two-column composite PK runs end-to-end with `verdict=pass`.**

### Phase 3 — Complex fact-table shapes — resink-core + DE + training pipeline; sized L; target `loop+4` / `loop+5`

Multiple fact streams feeding one dim with mixed ops (extend the partial support that exists today); fact tables that fan out to multiple dims; derived / joined facts. DE owns the fact-stream contract extension. The training pipeline's orchestrator owns the manifest emission for the richer fact topology. **Acceptance signal: a fact stream that fans out to two dims runs end-to-end with `verdict=pass`.**

The companion spec — `docs/superpowers/specs/2026-05-14-general-tables-design.md` — is the canonical contract each phase executes against. Its §9 implementation-loop plan is the cross-walk between spec sections and these three phases.

## Alternatives considered

**A. Keep the hand-written per-table modules forever.** Rejected. It makes the product fundamentally not a product — every customer onboarding requires a supervisor source change + a release. The training-pipeline → codegen → deploy story is incoherent without a general supervisor.

**B. Codegen the supervisor too — generate `nodes.rs` per tenant alongside the node crates.** Rejected. It trades one hardcoding for a generated hardcoding: the supervisor binary would still be per-tenant, per-DAG-version (the exact property ADR-2026-05-16-001's dlopen work was undoing). The supervisor should be *one* general binary that loads *many* node plugins, not a generated artifact.

**C. Generic trait-object runner consuming `Node` via the dlopen C-ABI.** **Selected.** The supervisor already has the `PluginNode` wrapper (ADR-2026-05-16-001). A generic runner that consumes nodes through the `Node` trait — extracting PK by schema, shaping output by schema — keeps the supervisor a single general binary and reuses the dlopen surface already in tree. It also subsumes the deferred "step 3.5 NodeCtx bridge." The ~90% structural duplication between `mod user` and `mod account` is direct evidence the runner is extractable.

**D. A fully data-driven interpreter — no per-dim crate at all; the supervisor interprets the schema directly.** Rejected for this arc (but noted as a possible future simplification). It would eliminate codegen entirely — the supervisor would run SCD2 maintenance as a built-in interpreted operation parameterized by schema. Attractive, but: (i) it discards the codegen + pattern-library investment; (ii) it cannot express non-SCD2 patterns (the codegen's `pattern_name` enum is the extensibility seam for ADS tables and beyond); (iii) it is a far larger blast radius than this arc should carry. The generic-runner-over-codegen'd-nodes path (C) is the incremental move; the interpreter is a someday-maybe.

**E. Schema source of truth — manifest node-spec vs DE `{key_columns, payload_columns}` convention vs codegen `schema_json`.** There are three schema shapes in the system today. Decision: **the manifest node-spec is the supervisor's runtime source of truth; it carries (or references) a schema in the codegen `schema_json` shape** (the richest of the three — it has column types, nullability, the `scd2_columns` triple). The DE `{key_columns, payload_columns}` convention is a *projection* of that richer schema, consumed by sim-farm; it is not lost, but it is derived, not authoritative. The spec's §3 works the reconciliation precisely; if it surfaces a contradiction requiring a DE contract change, that is a first-contact revision narrated in this ADR's Status section. This alternative is recorded because the reconciliation is the single most likely place this arc surfaces cross-team friction.

**F. Do Phase 2 (richer types) before Phase 1 (de-hardcoding).** Rejected. Adding `Float64` / `Timestamp` to `RawFieldValue` while `nodes::user` / `nodes::account` are still hand-written means extending two hand-written modules by hand — the duplication tax, paid again. Phase 1 first means Phase 2 extends *one* generic runner. Sequencing is load-bearing.

**G. Fold streaming (ADR-2026-05-13-001) and general-tables into one arc.** Rejected. They are orthogonal deviations — streaming is about *where events come from*; general-tables is about *what tables the supervisor can run*. The generic runner must **compose** cleanly with the `EventSource` trait (a `RawEvent` from `InMemoryStream` or `Kafka` flows into the generic runner identically to one from `ParquetReplay`), but the two arcs are planned, sequenced, and closed independently. The spec's §7 names the composition contract.

## Consequences

- **Positive.** The supervisor becomes a genuine general DAG executor — the runtime spec's documented design, reified in code. A customer's arbitrary schema runs without a supervisor source change. `nodes.rs` shrinks from 486 lines of hand-written duplication to a single parameterized runner. The dlopen surface (ADR-2026-05-16-001) gets its load-bearing consumer. `RawFieldValue` stops being a silent ceiling on what schemas the product accepts. The three arcs (dlopen — closed; streaming — in progress; general-tables — opening here) compose into the documented runtime: a stateless general supervisor loading cdylib node plugins, fed by a streaming event source.

- **Negative / costs.** Three more phases of architectural work across up to five teams (resink-core for the runner; DE for schema + fact contracts; AE for codegen template coverage; sim-farm for engine schema-awareness; training pipeline for richer manifest emission). The schema-model reconciliation (Alternative E) is real cross-team surface and may force a DE contract revision. Phase 1's generic runner must preserve the byte-stable `make mvp-loop` output the project's whole regression story depends on — a generic runner that produces *almost* the same bytes is a failure. The streaming arc (ADR-2026-05-13-001 steps 3-4) is deferred behind this arc; the org now carries two open architectural arcs and the CEO must sequence them.

- **Follow-ups required.** Implementing loops execute Phases 1-3 per the plan. The Status section is reserved here; it gets a closure narration when each phase lands and again when Phase 3 closes the arc. Sibling-arc ADRs may be filed as motivation surfaces: ADS / aggregate-table maintainers; cross-DAG / multi-tenant supervisor sharing; schema evolution of a deployed dim; possibly the data-driven-interpreter simplification (Alternative D) once the generic runner exists and the codegen pattern-library's shape is clearer.

## Status

### 2026-05-14, loop 2026-05-14-0857 — Phase 1 re-shaped + Phase 1a shipped

**Phase 1 was re-shaped on a first-contact finding, then split.** A probe before authoring the loop+1 brief found that Phase 1, as this ADR sized it ("resink-core; M"), was mis-scoped:

- This ADR's Decision §3 commits Phase 1 to consuming nodes through the dlopen `Node` C-ABI. But `nanofab_node_process(node_ptr, event_json, event_json_len, ctx_ptr) -> i32` returns **only a status code** — every SCD2 mutation must flow back through `ctx_ptr`.
- `ctx_ptr` is unwired today: the codegen template `lib.rs.tmpl` (AE-owned) ignores it and uses an in-process `DefaultProcessCtx`. So Phase 1 needs an **AE codegen-template change** — it is a cross-team loop, not a solo resink-core M. (The companion spec's §9 pre-flagged this exact site.)

**Resolution: a contract-first split.** Phase 1a (loop 2026-05-14-0857) and Phase 1b (next loop):

**Phase 1a — shipped this loop:**

- **`teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md`** — the NodeCtx C-ABI callback contract. Specifies the `#[repr(C)] NanofabNodeCtxVTable` (function pointers for `get_current` / `close_current` / `append_current` + an opaque `ctx_handle`), JSON serialization of keys + rows across the FFI boundary, `NodeError` propagation, and the `ctx_ptr` lifetime contract (valid for one `process` call, bound to one shard). resink-core-authored, **AE-co-authored** — AE confirmed it is implementable against `lib.rs.tmpl`'s `nanofab_node_process` shape with no C-ABI signature change (only a reinterpretation of the already-present `ctx_ptr`).
- **`crates/nanofab-supervisor/src/schema.rs`** — `DimSchema` + `ColumnDesc` + `ColumnType` + `Scd2ColumnTriple` + `DimSchema::from_json` with validation per spec §3. Fully implemented; 7 unit tests.
- **`crates/nanofab-supervisor/src/node_runner.rs`** — `NodeRunner`, `ShardKv`, `CompositeKey`, `RawFieldValueKey`, `Scd2RowGeneric`, the `NodeCtxBridge` trait, and the `NanofabNodeCtxVTable` declaration. Pure-Rust surface implemented: `CompositeKey` projection, the partition-key byte encoding (byte-stability-critical — a single-column string key encodes to exactly its bare UTF-8 bytes, reproducing the current per-table partitioner's input), `write_output`'s schema-driven parquet shaping. `NodeRunner::process_event`'s bridge call site is `unimplemented!("Phase 1b")`. 6 unit tests.
- **`crates/nanofab-supervisor/tests/general_tables.rs`** — the harness with a third synthetic dim (`dim_widget`). 3 tests pass this loop (three-dim parse/validate; composite-key projection; cross-dim encoding determinism); `supervisor_runs_three_dims` is `#[ignore]`'d as the Phase 1b acceptance gate.
- `nodes.rs` + `supervisor.rs` **untouched**; `make mvp-loop` byte-stable trivially (`verdict=pass mismatches=0`); `cargo test --workspace --release` green.

**First-contact spec revision narrated:** `RawFieldValueKey` ships as `String | Int64 | Bool` only — spec §2.3 sketched a `Timestamp` variant, but `RawFieldValue` carries no timestamp until Phase 2, so the variant would be unconstructible dead code. It joins `RawFieldValueKey` in Phase 2 alongside `RawFieldValue`'s widening.

**Phase 1b — next loop:**

- **AE** extends `lib.rs.tmpl`'s `nanofab_node_process` to interpret `ctx_ptr` as `*mut NanofabNodeCtxVTable` and process against a real `CAbiCtxShim` instead of `DefaultProcessCtx` (with null-`ctx_ptr` fallback for the template's standalone smoke test).
- **resink-core** implements `NodeCtxBridge` for `&mut ShardKv`, wires `NodeRunner::process_event`, wires the generic runner into `Supervisor::run`, retires `nodes.rs`'s `mod user`/`mod account` + `supervisor.rs`'s `match node_spec.table`, and un-ignores `supervisor_runs_three_dims`.

**Sized:** Phase 1a — M (one resink-core build session + a light AE co-author). Phase 1b — M (cross-team, AE + resink-core).

### 2026-05-15, loop 2026-05-15-0001 — Phase 1b shipped + Phase 1c carved out

**Phase 1b shipped the contract-implementation work.** The NodeCtx C-ABI callback contract is wired across both sides; the v-table bridge is proven end-to-end on real FFI.

- **AE** — `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/lib.rs.tmpl` extended: `nanofab_node_process` accepts `ctx_ptr` (no longer ignored); `NanofabNodeCtxVTable` declared inline, field order verbatim from the supervisor's authoritative declaration per the contract's lock-step discipline. The per-tenant `CAbiCtxShim` slot-fill is deferred to Phase 1c (the template's `_ = ctx_ptr` line is a marker for that follow-on); regenerated tenant crates continue using `DefaultProcessCtx` until the orchestrator's slot-fill is extended.
- **resink-core (cdylib side)** — `crates/nanofab-plugin-dim-user/src/lib.rs` + `crates/nanofab-plugin-dim-user-v2/src/lib.rs` hand-updated with concrete `CAbiCtxShim` implementations for dim_user (the in-tree cdylibs are hand-slot-fills, not regenerated; they carry the reference shape for Phase 1c's slot-fill).
- **resink-core (supervisor side)** — `crates/nanofab-supervisor/src/node_runner.rs` `impl NodeCtxBridge for ShardKv` + `NodeRunner::process_event` wired end-to-end: builds a v-table over the routed shard's `ShardKv`, serializes the event to event_json (codegen's tagged form), calls `PluginNode::process(event_json, ctx_ptr)`, drains accumulated mutations + writes trace records via `TraceWriter`, maps status codes per `NANOFAB_NODE_*`. `PluginNode::process` signature extended (`plugin_loader.rs`) to accept `ctx_ptr` (call sites updated in `tests/dlopen_integration.rs` + `tests/hot_swap_correctness.rs`).
- **Bridge proven end-to-end** — `tests/dlopen_integration.rs::bridge_v_table_round_trips_dim_user_insert` + `bridge_v_table_close_and_append_on_update` drive `dim_user` events through `NodeRunner` against the real `nanofab-plugin-dim-user` cdylib; the supervisor's `ShardKv` is mutated through the v-table callbacks; output parquets + trace records match expectations. 5/5 dlopen integration tests green; all 3 hot-swap correctness tests green (null-`ctx_ptr` fallback path preserved); 27/27 supervisor unit tests green.
- **`make mvp-loop` byte-stable** — `verdict=pass mismatches=0` on the widened 2-dim / 3-fact / 4-shard fixture. Trivially satisfied — the live `make mvp-loop` path goes through `nodes.rs` (static-plugins, untouched). The genuine "output via C-ABI matches the golden" gate is Phase 1c.

**In-loop re-shape: Phase 1b → 1b + 1c.** A build-time probe at brief-authoring found the brief was mis-sized at M+M:

- `supervisor.rs:189-211` unconditionally calls `nodes::user::run` + `nodes::account::run` — not gated by the dlopen-plugins feature.
- `Cargo.toml` carries `nanofab_node_dim_user_scd2` + `nanofab_node_dim_account_scd2` as path-deps by name; truly retiring static-plugins requires removing them.
- Only `nanofab-plugin-dim-user` (+ `-v2`) exists as a cdylib in-tree; dim_account + dim_widget have no cdylib equivalent.

The brief's KR1.4 (nodes.rs deleted), KR1.5 (supervisor_runs_three_dims un-ignored), and KR1.6 (output via C-ABI byte-stable) together require Makefile + Cargo.toml + orchestrator changes that the M+M sizing missed. **Re-shape (recorded in the loop's brief in-place, before code landed):**

- **Phase 1b (this loop) shipped** the contract-implementation work — the load-bearing v-table bridge proven on real FFI. `nodes.rs` stays alive; `supervisor_runs_three_dims` stays `#[ignore]`'d with its reason updated to name Phase 1c as the gate.
- **Phase 1c (`loop+1`)** carries the static-plugins retirement: build cdylibs for dim_account + dim_widget; extend the orchestrator to emit cdylib paths into the manifest; flip `make mvp-loop` default to dlopen-plugins; remove `Cargo.toml` path-deps; delete `nodes.rs` + `supervisor.rs`'s `match node_spec.table`; un-ignore + author `supervisor_runs_three_dims`. Acceptance signal: `make mvp-loop` byte-stable through the C-ABI.

**Contract revisions discovered during implementation: 0.** The contract held cleanly. One implementation-level finding worth recording: the codegen template's `parse_event_json` accepts the *tagged* FieldValue form (`{"string": "..."}`), while the contract specifies the *plain* form for the Row JSON (key/payload). Events flow INTO the node in tagged form; rows flow OUT in plain form. The contract was internally consistent on this; the AE template's CAbiCtxShim handles the asymmetry (the in-tree cdylib's `parse_scd2_row_json_dim_user` uses the plain form). Worth a clarifying note in the contract's serialization § for Phase 2's wider type coverage, but not a revision.

**Sized:** Phase 1a — M shipped. Phase 1b — M+M as sized for the contract-implementation work (shipped this loop). Phase 1c — M+M now-explicit (cdylibs + orchestrator + Makefile + Cargo.toml + nodes.rs retirement); previously folded into a notional M+M Phase 1b that the brief-time-probe missed.

**Sequencing impact:** Phase 2 moves to `loop+2` (was `loop+1` before the split); Phase 3 to `loop+4` (was `loop+3`); the org-os-process loop (CEO decision (b) of this loop's brief) moves to `loop+3` (was `loop+2`).

### _(Reserved — Phase 1c closure as it lands; Phase 2 + Phase 3 closure as each lands; Phase 3 closes the arc.)_

## Links

- Companion design spec: [docs/superpowers/specs/2026-05-14-general-tables-design.md](../../docs/superpowers/specs/2026-05-14-general-tables-design.md) — the canonical generic-node-runner contract + schema model + three-phase implementation-loop plan.
- Runtime spec: [docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) — describes the supervisor as a general DAG executor; this ADR closes the gap to that design.
- Sister ADR (dlopen — closed): [ADR-2026-05-16-001](2026-05-16-001-abi-option-a-mvp-deviation.md) — the `PluginNode` C-ABI surface this arc's generic runner consumes; its deferred "step 3.5 NodeCtx bridge" is subsumed by Phase 1.
- Sister ADR (streaming — in progress): [ADR-2026-05-13-001](2026-05-13-001-streaming-event-source-rearchitecture.md) — the `EventSource` trait the general-tables runner must compose with; orthogonal arc, planned independently.
- DE schema-JSON convention: [teams/platform/data-engineering/conventions/dim-schema-json.md](../../teams/platform/data-engineering/conventions/dim-schema-json.md) — the `{key_columns, payload_columns}` shape; reconciled in spec §3 as a projection of the richer codegen schema.
- Codegen skill: `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/SKILL.md` — the `schema_json` shape (five types, list `primary_key`, `scd2_columns` triple) that becomes the supervisor's runtime schema source of truth.
- Triggering brief: [board/okrs/2026-05-14-0742-ceo-brief.md](../okrs/2026-05-14-0742-ceo-brief.md).
- Relative-dating convention for the `loop+N` targets: [ADR-2026-05-30-002](2026-05-30-002-multi-loop-plan-relative-dating.md).
{% endraw %}
