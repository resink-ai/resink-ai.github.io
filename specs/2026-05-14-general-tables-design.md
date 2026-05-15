---
layout: default
title: "Spec: 2026-05-14-general-tables-design"
date: 2026-05-14
status: active
type: spec
owner: board
parent: Design specs
---

<!-- original-frontmatter:
  type: spec
  owner: board
  date: 2026-05-14
  status: active
  links: parent: board/decisions/2026-05-14-001-general-tables-rearchitecture.md
-->
{% raw %}

# General-tables design — the generic schema-driven node runner

Companion design spec for [ADR-2026-05-14-001](../../../board/decisions/2026-05-14-001-general-tables-rearchitecture.md). The ADR names the deviation (the supervisor hardcodes `dim_user` / `dim_account`) and commits to a three-phase restoration. This spec is the canonical contract each phase executes against.

## 1. Goals & non-goals

### Goals

- Make the supervisor a **general DAG executor**: an arbitrary set of dim tables, declared in the manifest, runs without a supervisor source change.
- Retire `crates/nanofab-supervisor/src/nodes.rs`'s hand-written `mod user` / `mod account` and `supervisor.rs`'s `match node_spec.table` in favor of one generic schema-driven node runner.
- Close `RawFieldValue`'s type gap (`Float64`, `Timestamp`) against the codegen's five accepted types.
- Honor composite primary keys end-to-end.
- Model complex fact-table shapes (multi-stream-per-dim, fan-out, derived facts) so fact routing is as general as dim handling.
- Preserve byte-stable `make mvp-loop` output through every phase.

### Non-goals (explicitly out of scope for this spec)

- **Streaming ingestion / Kafka.** ADR-2026-05-13-001's domain. The generic runner must *compose* with the `EventSource` trait (§7) but Kafka is not in this arc.
- **ADS / aggregate-table maintainers.** The runtime spec mentions "dim/ADS-table maintainers"; ADS is a sibling arc. The `pattern_name` enum in the codegen is the extensibility seam — this spec keeps the runner pattern-agnostic where cheap, but only `scd2_maintainer` is exercised.
- **Cross-DAG / multi-tenant supervisor sharing.** Sibling arc.
- **Schema evolution / migration of an already-deployed dim.** Sibling arc.
- **The fully data-driven interpreter** (ADR Alternative D). A someday-maybe once the generic runner exists; not this arc.

## 2. The generic node runner

### 2.1 Why a per-node adapter exists today (the thing being removed)

`nodes.rs`'s own module comment states the deviation precisely:

> Each codegen crate inlines its own `Event`, `FieldValue`, `NodeError`, `NodeCtx`, and `Node` types — they are distinct Rust types per crate even though they share a structural shape. The supervisor cannot use a single generic abstraction without changing the codegen template; instead we per-node adapter both the inbound event and the outbound parquet rows.

So the per-node adapter exists because `nanofab_node_dim_user_scd2::Event` and `nanofab_node_dim_account_scd2::Event` are *different Rust types*. The supervisor imports each crate (`use nanofab_node_dim_user_scd2 as crate_user;`) and writes a `raw_to_user_event` / `raw_to_account_event` adapter per crate.

### 2.2 The fix: consume nodes through the C-ABI exclusively

The `PluginNode` wrapper (ADR-2026-05-16-001, `crates/nanofab-supervisor/src/plugin_loader.rs`) already loads a node's cdylib and invokes `nanofab_node_new` / `nanofab_node_process` / `nanofab_node_drop`. The C-ABI is **type-erased** — opaque handles + serialized byte buffers. A supervisor that consumes nodes *purely* through `PluginNode` never names a per-crate Rust type. No `use nanofab_node_dim_user_scd2`. No `DimUserKey`. No `raw_to_user_event`.

The generic runner is therefore: **serialize a `RawEvent` to the C-ABI's event buffer format → `PluginNode::process(...)` → deserialize the returned mutation buffer → apply to the per-shard KV → emit trace records → at end-of-stream, shape the output parquet from the schema.** Everything schema-specific is a *runtime parameter* read from the node-spec's schema, not a compile-time type.

### 2.3 Core types

```rust
//! The generic node runner. One instance per node-spec in the manifest.
//! Replaces `nodes::user` / `nodes::account`. Lives at
//! `crates/nanofab-supervisor/src/node_runner.rs`.

use crate::event_source::RawEvent;
use crate::plugin_loader::PluginNode;
use crate::schema::DimSchema;          // §3
use crate::trace::TraceWriter;
use std::path::Path;

/// A generic per-dim handler, built from a node-spec's schema. Holds the
/// loaded plugin, the per-shard KV partitions, and the schema that drives
/// PK extraction + output shaping. There is exactly one `NodeRunner` per
/// node-spec; the supervisor builds N of them for an N-node DAG.
pub struct NodeRunner {
    /// The dim table this runner serves (e.g. "dim_user"). Used only for
    /// routing + trace labels — never matched against a hardcoded set.
    table: String,
    /// The schema: PK columns, payload columns, column types, scd2 triple.
    /// Drives every schema-specific decision at runtime. See §3.
    schema: DimSchema,
    /// The loaded node plugin, consumed purely through the C-ABI. The
    /// supervisor never names a per-crate Rust type.
    plugin: PluginNode,
    /// One KV partition per logical shard. Generic over schema: the key is
    /// the schema-projected composite PK (§5), not a per-crate `DimUserKey`.
    shards: Vec<ShardKv>,
}

/// Per-shard KV. Generic — keyed by the schema-projected composite primary
/// key (a `Vec<RawFieldValue>` in `schema.primary_key` order) plus
/// `valid_from`. Replaces the per-crate `ShardKv { HashMap<(DimUserKey,u64),
/// Scd2Row<DimUserRow>> }`.
#[derive(Default)]
pub struct ShardKv {
    /// All persisted SCD2 versions, keyed by (composite-pk, valid_from).
    rows: std::collections::HashMap<(CompositeKey, u64), Scd2RowGeneric>,
    /// The current open version per composite-pk.
    current: std::collections::HashMap<CompositeKey, u64>,
}

/// The schema-projected composite primary key: the `RawFieldValue`s of the
/// event's `after` map at the schema's `primary_key` column names, in order.
/// Single-column PKs are the len-1 case — no special-casing.
#[derive(Clone, PartialEq, Eq, Hash)]
pub struct CompositeKey(pub Vec<RawFieldValueKey>);

/// `RawFieldValue` is not `Hash`/`Eq` (it carries `f64`). `RawFieldValueKey`
/// is the hashable projection used only for KV keying — strings/ints/bools/
/// timestamps hash directly; floats are rejected as PK columns at schema-
/// validation time (§3.4), so the float case is unreachable here.
#[derive(Clone, PartialEq, Eq, Hash)]
pub enum RawFieldValueKey {
    String(String),
    Int64(i64),
    Bool(bool),
    Timestamp(i64),
}

/// A generic persisted SCD2 row: the payload columns as a `Vec<RawFieldValue>`
/// in `schema.payload_columns` order, plus the scd2 triple. Replaces the
/// per-crate `Scd2Row<DimUserRow>`.
pub struct Scd2RowGeneric {
    pub key: CompositeKey,
    pub payload: Vec<crate::event_source::RawFieldValue>,
    pub valid_from: u64,
    pub valid_to: Option<u64>,
    pub is_current: bool,
}

impl NodeRunner {
    /// Build a runner from a node-spec's schema + the node's cdylib path.
    /// Infallible construction; `start` does the fallible work.
    pub fn new(table: String, schema: DimSchema, plugin: PluginNode, shard_count: u32) -> Self {
        let shards = (0..shard_count.max(1)).map(|_| ShardKv::default()).collect();
        Self { table, schema, plugin, shards }
    }

    /// Route one `RawEvent` to its shard and process it through the plugin.
    /// Shard = `partition(composite_key_bytes, shard_count)` — the existing
    /// xxHash64(seed=0) partitioner, fed the schema-projected key bytes.
    pub fn process_event(
        &mut self,
        event: &RawEvent,
        trace_writer: &mut TraceWriter,
    ) -> Result<(), String> {
        // (a) project the composite key from `event.after` per `schema.primary_key`
        // (b) shard = partition(key_bytes, self.shards.len())
        // (c) serialize event -> C-ABI event buffer
        // (d) PluginNode::process via the NodeCtx bridge (§2.4) bound to
        //     `self.shards[shard]`
        // (e) deserialize returned mutations -> apply to the shard KV
        // (f) emit one trace record per mutation
        unimplemented!("Phase 1 — resink-core")
    }

    /// At end-of-stream: shape this dim's output parquet from `self.schema`.
    /// Column order, types, and the scd2 triple all come from the schema —
    /// no hardcoded `Field::new("user_id", ...)`. Returns the row count.
    pub fn write_output(&self, out_path: &Path) -> Result<usize, String> {
        unimplemented!("Phase 1 — resink-core")
    }
}
```

### 2.4 The NodeCtx bridge

ADR-2026-05-16-001 deferred "step 3.5 — supervisor-side NodeCtx bridge." It is **not deferred any longer — it is the heart of Phase 1.** The C-ABI node, mid-`process`, must call back into the supervisor's per-shard KV for `get_current` / `close_current` / `append`. The bridge is the set of C-ABI callbacks the supervisor hands the node at `process` time, bound to `self.shards[shard]`.

The exact callback ABI (function-pointer table vs. a serialized request/response loop) is the **first decision Phase 1's implementing loop owns** — it depends on the `nanofab_node_process` signature already shipped in the codegen template. The spec fixes the *contract* (the node can read the current version, close it, and append a new one, scoped to one shard, within one `process` call); the implementing loop fixes the *wire shape*.

> **Wire shape ratified (loop 2026-05-14-0857 — Phase 1a).** The probe found `nanofab_node_process` returns *only* a status code and `ctx_ptr` is unwired in the codegen template — so Phase 1 was re-shaped into a contract-first split (see ADR-2026-05-14-001 § Status). The wire shape is now fixed by **[`teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md`](../../../teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md)**: a `#[repr(C)] NanofabNodeCtxVTable` of function pointers behind `ctx_ptr`, JSON key/row serialization across the FFI boundary. The `NodeCtxBridge` trait below is unchanged; its `impl` for `&mut ShardKv` and the v-table that drives it land in Phase 1b.

```rust
/// The supervisor-side NodeCtx bridge. Bound to one shard's KV for the
/// duration of one `process_event` call. The node invokes these through the
/// C-ABI callback table (wire shape: Phase 1 implementing loop's call).
trait NodeCtxBridge {
    fn get_current(&self, key: &CompositeKey) -> Option<Scd2RowGeneric>;
    fn close_current(&mut self, key: &CompositeKey, event_ts: u64) -> Result<(), String>;
    fn append(&mut self, row: Scd2RowGeneric) -> Result<(), String>;
}
// `impl NodeCtxBridge for &mut ShardKv` — Phase 1.
```

### Why this shape

- **Type-erased C-ABI consumption is what makes the runner general.** The moment the supervisor stops importing `nanofab_node_dim_user_scd2 as crate_user`, the per-table modules have nothing left to do.
- **`CompositeKey` as `Vec<RawFieldValueKey>`** makes single-column PKs the len-1 case — no `if pk.len() == 1` branch anywhere.
- **`Scd2RowGeneric` carries payload as `Vec<RawFieldValue>` in schema order** — output-parquet shaping reads the schema, not a hardcoded `Field::new`.
- **The runner is one struct, N instances.** The supervisor builds one `NodeRunner` per manifest node-spec. Three dims = three runners = zero new code.

## 3. The schema model

There are three schema shapes in the system today (ADR Alternative E). This section reconciles them.

| Shape | Where | Keys |
|---|---|---|
| Manifest node-spec | `manifest.yaml` `dags[].nodes[]` | `id`, `table`, `pk`, `fact_streams`, `node_version` |
| Codegen `schema_json` | `codegen-scd2-node` input | `table`, `columns[]{name,type,nullable}`, `primary_key[]`, `scd2_columns{}` |
| DE `{key_columns, payload_columns}` | `dim-schema-json.md` convention, consumed by sim-farm | `key_columns[]`, `payload_columns[]` |

### 3.1 Decision: the codegen `schema_json` shape is the supervisor's runtime source of truth

It is the richest of the three — it has column types, nullability, and the `scd2_columns` triple. The supervisor's runtime `DimSchema` type is built from it:

```rust
//! `crates/nanofab-supervisor/src/schema.rs`

pub struct DimSchema {
    pub table: String,
    pub columns: Vec<ColumnDesc>,
    /// Column names forming the primary key, in order. Len >= 1.
    pub primary_key: Vec<String>,
    /// The SCD2 bookkeeping triple: (valid_from, valid_to, is_current) column names.
    pub scd2_columns: Scd2ColumnTriple,
}

pub struct ColumnDesc {
    pub name: String,
    pub ty: ColumnType,
    pub nullable: bool,
}

/// The five accepted column types — exactly the codegen's accepted set.
pub enum ColumnType { String, Int64, Float64, Bool, TimestampMs }

pub struct Scd2ColumnTriple {
    pub valid_from: String,
    pub valid_to: String,
    pub is_current: String,
}
```

### 3.2 The DE convention is a *projection*, not a competitor

DE's `{key_columns, payload_columns}` is **derivable** from `DimSchema`:
- `key_columns` == `DimSchema.primary_key`
- `payload_columns` == `DimSchema.columns` minus the PK columns minus the scd2 triple

The supervisor does not consume the DE convention directly — it emits it (or the orchestrator does) as a projection for sim-farm. The DE convention is not lost; it is downstream. **If Phase 2's implementing work finds the projection is lossy** (e.g. sim-farm needs a type hint the `{key_columns, payload_columns}` shape can't carry), that is a first-contact revision: the DE convention gets a contract change, narrated in the ADR's Status section.

### 3.3 Where the supervisor reads the schema

The manifest node-spec gains a `schema_ref` field — a path to the node's `schema.json` (the codegen `schema_json` shape), already produced by the codegen path under `workspace/nodes/<dim>_scd2/`. The supervisor reads + parses it into `DimSchema` at `NodeRunner::new` time. The manifest's existing `pk` field becomes redundant with `schema.primary_key` — Phase 1 keeps both (manifest `pk` as a cross-check assertion against `schema.primary_key`); a later cleanup loop may drop the manifest `pk`.

### 3.4 Schema validation

`DimSchema::from_json` validates at parse time: `primary_key` non-empty; every PK column present in `columns` with `nullable: false`; **no PK column of type `Float64`** (floats are not safe hash/eq keys — §2.3's `RawFieldValueKey` has no float variant); `scd2_columns` all three present in `columns`. A schema that fails validation is a `Fatal` error — the supervisor refuses to build a `NodeRunner` for it.

## 4. `RawFieldValue` type-coverage closure

Today: `RawFieldValue { String, Int64, Bool, Null }`. The codegen accepts five types. Phase 2 closes the gap:

```rust
pub enum RawFieldValue {
    String(String),
    Int64(i64),
    Float64(f64),       // NEW — Phase 2
    Bool(bool),
    Timestamp(i64),     // NEW — Phase 2; epoch-millis, matches codegen `timestamp_ms`
    Null,
}
```

`arrow_value_to_field` (in `event_source/raw.rs`) gains downcasts for `Float64Array` and `TimestampMillisecondArray`. The output-parquet writer in `NodeRunner::write_output` gains the matching arrow `DataType` arms (`Float64`, `Timestamp(Millisecond, None)`).

**Determinism note.** `f64` has no total order and no `Eq`. `RawFieldValue` therefore cannot derive `Hash`/`Eq` once `Float64` lands. This is why §2.3 splits out `RawFieldValueKey` (the hashable subset, used only for KV keys) and why §3.4 forbids `Float64` PK columns. Float *payload* columns are fine — they are written to the output parquet, never hashed.

## 5. Composite primary keys

Single-column PKs are the len-1 case of the composite path; there is no separate single-column code path.

- **Key projection.** `CompositeKey` = the `RawFieldValueKey`s of `event.after` at each name in `schema.primary_key`, in order. A missing PK column in an event's `after` map is a `Fatal` error for that event.
- **Partitioning.** The shard is `partition(key_bytes, shard_count)` where `key_bytes` is the canonical byte-encoding of the `CompositeKey` — the existing xxHash64(seed=0) partitioner, fed a deterministic concatenation of the key columns' byte representations (the encoding is fixed in Phase 1 and must be byte-stable; it is part of the regression contract).
- **Trace records.** `key_json` becomes an object with one field per PK column (today's `{"user_id": ...}` is the len-1 instance) — built from `schema.primary_key` + the projected values.
- **Output parquet.** The PK columns appear in the output in `schema.primary_key` order, then payload columns in `schema.payload_columns` order, then the scd2 triple — a fully schema-driven column layout.

## 6. Complex fact-table shapes — the model

Phase 3. This section fixes the *model*; the implementing loop owns the wiring.

- **Multiple fact streams per dim (mixed ops).** Already partially supported — the manifest's `fact_streams` is a list, and the per-row `op` column override exists. Phase 3 makes the supervisor's fact-stream merge fully order-correct across N streams with mixed ops (the existing global sort by `(event_ts, event_id)` generalizes; the work is validating it under > 2 streams).
- **Fact fan-out — one fact stream feeding multiple dims.** New. A fact stream's events are routed to *every* dim whose node-spec subscribes to it. The manifest node-spec's `fact_streams` already names streams per node; fan-out is the case where two node-specs name the same stream. The supervisor reads a fact stream once and dispatches each event to every subscribing `NodeRunner`. **Acceptance: a fact stream feeding two dims runs end-to-end with `verdict=pass`.**
- **Derived / joined facts.** New, and the largest piece. A derived fact is computed from other facts (or from a dim's current state) before it reaches a maintainer. The model: a derived-fact node is itself a DAG node with a `pattern_name` other than `scd2_maintainer` — which means **derived facts are the first real consumer of the `pattern_name` extensibility seam**, and may motivate pulling the ADS-table sibling arc forward. Phase 3 specs the manifest shape for a derived-fact node + the supervisor's two-pass execution (derive, then maintain); the maintainer-pattern codegen for derived facts is explicitly a Phase-3-implementing-loop scope question, possibly its own sub-arc.

## 7. Supervisor integration

### 7.1 What retires

- `crates/nanofab-supervisor/src/nodes.rs` — **deleted.** Its 486 lines of `mod user` + `mod account` become `node_runner.rs` (the generic `NodeRunner`) + `schema.rs` (the `DimSchema` model).
- `supervisor.rs`'s `match node_spec.table.as_str() { "dim_user" | "dim_account" => ... }` — **deleted.** Replaced by: for each node-spec, build a `NodeRunner` from its schema.
- `supervisor.rs`'s `buffers.remove("dim_user")` / `buffers.remove("dim_account")` + literal `nodes::user::run` / `nodes::account::run` — **deleted.** Replaced by: route each `RawEvent` to `runners[event.table].process_event(...)`.
- The `Cargo.toml` path-deps `nanofab_node_dim_user_scd2` / `nanofab_node_dim_account_scd2` — **deleted** once consumption is pure-C-ABI (the supervisor no longer compiles the node crates in-tree; it `dlopen`s their cdylibs). This also retires the `static-plugins` feature's reason to exist for these two crates.

### 7.2 The new `Supervisor::run` shape

```rust
// Pseudocode — Phase 1 implementing loop owns the exact body.
let manifest = load_manifest(&config.workspace)?;
let mut runners: HashMap<String, NodeRunner> = HashMap::new();
for node_spec in &manifest.dag.nodes {
    let schema = DimSchema::from_json_file(&node_spec.schema_ref)?;   // §3
    let plugin = PluginNode::load(&node_spec.cdylib_path)?;           // ADR-2026-05-16-001
    runners.insert(
        node_spec.table.clone(),
        NodeRunner::new(node_spec.table.clone(), schema, plugin, manifest.shard_count),
    );
}
// event loop — composes with the EventSource trait (ADR-2026-05-13-001):
let mut source = build_source(...);          // ParquetReplay | InMemoryStream | Kafka
source.start()?;
loop {
    match source.poll_events() {
        Ok(batch) => for event in batch.events {
            // fan-out (§6): dispatch to every subscribing runner. For plain
            // dims this is exactly one runner — `runners[&event.table]`.
            for runner in subscribers(&runners, &event) {
                runner.process_event(&event, &mut trace_writer)?;
            }
            source.commit_offsets(batch.commit_token)?;
        },
        Err(EndOfStream) => break,
        Err(Retryable(_)) => { /* backoff — streaming arc */ }
        Err(Fatal(m)) => return Err(m),
    }
}
source.shutdown()?;
for (table, runner) in &runners {
    let rows = runner.write_output(&output_path_for(table))?;
}
```

### 7.3 Composition with the streaming arc

The general-tables runner and the `EventSource` trait (ADR-2026-05-13-001) are orthogonal and compose by construction: a `RawEvent` is a `RawEvent` regardless of whether it came from `ParquetReplay`, `InMemoryStream`, or `Kafka`. The general-tables arc touches *what happens to an event after `poll_events` returns it*; the streaming arc touches *where the event came from*. Neither blocks the other. The only shared surface is the `RawEvent` type itself — and Phase 2's `RawFieldValue` extension (`Float64`, `Timestamp`) is the one change both arcs see; it is additive and non-breaking.

### 7.4 The migration path — byte-stability is the contract

`make mvp-loop` produces a byte-stable verdict; the project's entire regression story depends on it. Phase 1's generic `NodeRunner`, fed the existing `dim_user` / `dim_account` schemas, **must produce byte-identical output parquets to the hand-written `nodes::user` / `nodes::account`.** A generic runner that produces *almost* the same bytes is a Phase 1 failure. The migration is: build the generic runner alongside the hand-written modules; assert byte-identical output on the canonical fixture; *then* delete the hand-written modules. The composite-key byte-encoding (§5) is the highest-risk byte-stability surface — it must reproduce the single-column key bytes the current code produces.

## 8. Test-harness shape

The generality gate is **a third synthetic dim table** — the thing the hardcoded path cannot fake.

- **Phase 1 acceptance.** A `tests/general_tables.rs` integration test (reusing the `streaming_event_source.rs` harness shape from ADR-2026-05-13-001 step 2): a tempdir manifest with *three* node-specs — `dim_user`, `dim_account`, and a synthetic `dim_widget` with its own schema (single-column PK, the existing field types). Drive it through `Supervisor::run` via `InMemoryStream`. Assert all three output parquets are produced + row counts are correct. The existing two-dim `make mvp-loop` stays byte-stable in parallel.
- **Phase 2 acceptance.** Extend the synthetic dim: `dim_widget` gains a `float64` price column, a `timestamp_ms` column, and a two-column composite PK. Assert end-to-end `verdict=pass`.
- **Phase 3 acceptance.** A fact stream feeding two dims (fan-out). Assert end-to-end `verdict=pass`.
- **The byte-stability assertion** (§7.4) is a distinct test: generic `NodeRunner` output vs. a frozen golden copy of the current hand-written output, byte-compared.

## 9. Implementation-loop plan

Each row maps to the ADR's three phases. Future briefs cite this table; future OKR KRs map to the named spec sections.

| Phase | Owner(s) | Sized | Target | Closes which spec sections |
|---|---|---|---|---|
| 1a. NodeCtx C-ABI callback contract + pure-Rust scaffolding (`DimSchema`, `NodeRunner` skeleton, composite-key encoding, test harness) — **shipped loop 2026-05-14-0857** | resink-core (lead) + AE (co-author) | M | `loop+1` | §2.3, §3.1-3.4, §5 (composite-key projection + encoding), §8 (Phase 1a tests) |
| 1b. Wire the NodeCtx bridge — AE `lib.rs.tmpl` `ctx_ptr` consumption + v-table declaration; resink-core `NodeCtxBridge for ShardKv` + `NodeRunner::process_event` + `PluginNode::process` extended for `ctx_ptr`; in-tree `nanofab-plugin-dim-user` (+ `-v2`) hand-updated; bridge proven end-to-end via real FFI — **shipped loop 2026-05-15-0001** | AE + resink-core | M+M | `loop+2` | §2.4 (bridge wire-shape resolved + reified), §8 (`bridge_v_table_round_trips_dim_user_insert` + `bridge_v_table_close_and_append_on_update` new tests) |
| 1c. Static-plugins retirement — cdylibs for dim_account + dim_widget; orchestrator emits cdylib paths into manifest; `make mvp-loop` default flipped to dlopen-plugins; `Cargo.toml` path-deps removed; `nodes.rs` deleted; `supervisor.rs::match` retired; `supervisor_runs_three_dims` un-ignored + authored | resink-core + AE (orchestrator slot-fill extension) | M+M | `loop+3` | §7.1 (nodes.rs retirement), §7.2 (Supervisor::run rewire), §7.4 (byte-stability via C-ABI), §8 (`supervisor_runs_three_dims` un-ignored) |
| 2. Richer schemas + field types | resink-core + DE + AE + sim-farm | M-L | `loop+4` / `loop+5` | §4, §3.2 (DE projection), §5 (composite-PK acceptance), §8 (Phase 2 test) |
| 3. Complex fact-table shapes | resink-core + DE + training pipeline | L | `loop+6` / `loop+7` | §6, §7.2 (fan-out dispatch), §8 (Phase 3 test) |

> **Phase 1 split history.** Phase 1 originally sized as resink-core; M (the ADR's sizing from design). Loop 2026-05-14-0857's brief-time probe found cross-team need (`ctx_ptr` unwired in the codegen template) and split into 1a + 1b. Loop 2026-05-15-0001's brief-time probe missed a second coupling — `supervisor.rs:189-211` unconditionally calls `nodes::user::run` + path-deps in `Cargo.toml` — and the in-loop re-shape further split into 1b + 1c. Phase 1b (contract implementation, bridge proven on real FFI) shipped loop 2026-05-15-0001; Phase 1c (static-plugins retirement, byte-stability via C-ABI) shipped at `loop+1`. Phases 2 + 3 `loop+N` targets shift by one each time. The build-time-probe checklist gap is captured in the loop 2026-05-15-0001 retro.

Each phase:

- Closes its spec sections **by code, not by re-spec.** The spec is the canonical contract.
- Surfaces drift to the spec if discovered (a phase finds the contract needs a change) — captured as an in-loop spec edit + a one-line note in the ADR's Status section, per the ADR-2026-05-13-001 first-contact-revision convention.
- Reports back to the ADR's Status section at close.

The most likely first-contact revision sites, flagged in advance:

- **§2.4 NodeCtx bridge wire shape** — ✅ **resolved (loop 2026-05-14-0857).** The probe confirmed the C-ABI needs an AE codegen-template change; rather than improvise it cross-team in one loop, Phase 1 split into 1a (contract + scaffolding) and 1b (bridge wiring). Wire shape ratified in `contracts/2026-05-14-nodectx-cabi-callback.md`.
- **§3.2 DE projection** — Phase 2 may find `{key_columns, payload_columns}` is lossy for sim-farm; that is a DE contract revision.
- **§6 derived facts** — Phase 3 may find derived facts need the ADS sibling arc pulled forward.

## 10. Cross-references

- **Ratifying ADR:** [board/decisions/2026-05-14-001-general-tables-rearchitecture.md](../../../board/decisions/2026-05-14-001-general-tables-rearchitecture.md).
- **Runtime spec:** [docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md](2026-05-10-nanofab-runtime-design.md) — the general-DAG-executor design this spec reifies.
- **Sister ADR (dlopen — closed):** [board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md](../../../board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md) — the `PluginNode` C-ABI surface §2.2 consumes; its deferred step-3.5 NodeCtx bridge is §2.4.
- **Sister ADR (streaming — in progress):** [board/decisions/2026-05-13-001-streaming-event-source-rearchitecture.md](../../../board/decisions/2026-05-13-001-streaming-event-source-rearchitecture.md) — the `EventSource` trait §7.3 composes with.
- **DE schema-JSON convention:** [teams/platform/data-engineering/conventions/dim-schema-json.md](../../../teams/platform/data-engineering/conventions/dim-schema-json.md) — reconciled in §3.2 as a projection of the richer codegen schema.
- **Codegen skill:** `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/SKILL.md` — the `schema_json` shape §3.1 adopts as the runtime source of truth.
- **Relative-dating convention** for the `loop+N` targets: [board/decisions/2026-05-30-002-multi-loop-plan-relative-dating.md](../../../board/decisions/2026-05-30-002-multi-loop-plan-relative-dating.md).
{% endraw %}
