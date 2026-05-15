---
layout: default
title: "application-resink-core contract: 2026-05-14-nodectx-cabi-callback"
date: 2026-05-14
status: active
type: contract
loop: 2026-05-14-0857
owner: teams/application/resink-core
grand_parent: Teams
parent: "Team: application-resink-core"
---

<!-- original-frontmatter:
  type: contract
  owner: teams/application/resink-core
  date: 2026-05-14
  status: active
  loop: 2026-05-14-0857
  links: parent: board/decisions/2026-05-14-001-general-tables-rearchitecture.md
-->
{% raw %}

# NodeCtx C-ABI callback contract

**Owner:** resink-core (the `nanofab-node-abi` crate is the canonical home of the `NodeCtx` trait).
**Co-author:** agent-engineering (AE owns `lib.rs.tmpl` — the node side of this C-ABI).
**Status:** active. End-to-end verification lands in Phase 1b (see `Verified-against-environment` below).

## Purpose

ADR-2026-05-14-001 commits the general-tables Phase 1 to consuming nodes through the dlopen `Node` C-ABI. The C-ABI export is:

```rust
unsafe extern "C" fn nanofab_node_process(
    node_ptr: *mut <NodeStruct>,
    event_json: *const u8,
    event_json_len: usize,
    ctx_ptr: *mut std::ffi::c_void,
) -> i32   // one of NANOFAB_NODE_{OK,PANIC,BLOCKED,RETRY}
```

`nanofab_node_process` returns **only a status code**. Every SCD2 mutation the supervisor needs — read the current row, close it, append the successor — must flow back through `ctx_ptr`. Today `ctx_ptr` is ignored (`_ctx_ptr`); the template processes against an in-process `DefaultProcessCtx`.

This contract specifies what lives behind `ctx_ptr`: a **type-erased C-ABI projection of the generic `nanofab_node_abi::NodeCtx<K, R>` trait**. It is the artifact both Phase 1b deliverables execute against — AE's `lib.rs.tmpl` change (node side) and resink-core's `NodeCtxBridge` impl (supervisor side).

## The `NodeCtx<K,R>` trait being projected

From `crates/nanofab-node-abi/src/lib.rs`, the generic surface:

```rust
pub trait NodeCtx<K, R> {
    fn get_current(&self, table: &str, key: &K) -> Option<Scd2Row<R>>;
    fn close_current(&mut self, table: &str, key: &K, event_ts: u64) -> Result<(), NodeError>;
    fn append_current(&mut self, table: &str, key: &K, row: R, event_ts: u64) -> Result<(), NodeError>;
    fn already_processed(&self, _event_id: &str) -> bool { false }   // default
    fn mark_processed(&mut self, _event_id: &str) {}                 // default
}
```

`K` and `R` are per-node Rust types — known to the node, opaque to the supervisor. The C-ABI must therefore carry `K` and `R` **as serialized bytes**. `already_processed` / `mark_processed` keep their trait defaults this MVP — they are **out of scope for the C-ABI table** (the node uses the no-op defaults; the supervisor's per-shard idempotency log is a runtime-spec §7.1 concern for a later arc).

## The C-ABI callback table

Behind `ctx_ptr` is a pointer to a `#[repr(C)]` v-table. Both sides declare it identically (resink-core in `node_runner.rs`, AE inlined in `lib.rs.tmpl` until the `nanofab-node-abi` crate exports it):

```rust
#[repr(C)]
pub struct NanofabNodeCtxVTable {
    /// Opaque supervisor-side handle. Bound to exactly one shard's KV for
    /// the duration of one `nanofab_node_process` call. The node passes
    /// this back, unmodified, as the first argument to every callback.
    pub ctx_handle: *mut std::ffi::c_void,

    /// Read the current SCD2 row for `key`.
    /// Returns: 0 = found (out buffer filled, `*out_len` set);
    ///          1 = no current row for this key;
    ///          2 = out buffer too small (`*out_len` set to required length;
    ///              caller retries with a larger buffer);
    ///         <0 = supervisor-side error.
    pub get_current: unsafe extern "C" fn(
        ctx_handle: *mut std::ffi::c_void,
        table: *const u8, table_len: usize,
        key_json: *const u8, key_json_len: usize,
        out_row_json: *mut u8, out_row_json_cap: usize,
        out_len: *mut usize,
    ) -> i32,

    /// Close the current row for `key`: set `valid_to = event_ts`,
    /// `is_current = false`. No-op if no current row exists.
    /// Returns: 0 = ok; <0 = supervisor-side error.
    pub close_current: unsafe extern "C" fn(
        ctx_handle: *mut std::ffi::c_void,
        table: *const u8, table_len: usize,
        key_json: *const u8, key_json_len: usize,
        event_ts: u64,
    ) -> i32,

    /// Append a new current row for `key` with `valid_from = event_ts`,
    /// `valid_to = None`, `is_current = true`.
    /// Returns: 0 = ok; <0 = supervisor-side error.
    pub append_current: unsafe extern "C" fn(
        ctx_handle: *mut std::ffi::c_void,
        table: *const u8, table_len: usize,
        key_json: *const u8, key_json_len: usize,
        row_json: *const u8, row_json_len: usize,
        event_ts: u64,
    ) -> i32,
}
```

`ctx_ptr` (the `*mut c_void` the supervisor passes to `nanofab_node_process`) is `*mut NanofabNodeCtxVTable`. The node casts it back, reads the three function pointers + `ctx_handle`, and builds a thin shim that `impl NodeCtx<K, R>` by calling them.

## Serialization across the boundary

All structured values cross as **UTF-8 JSON byte buffers**. Rationale: `event_json` already crosses as JSON; the template already hand-rolls a JSON parser for the `Event` shape; keeping one serialization keeps the template dep-free per its `Cargo.toml.tmpl` "zero external deps" invariant.

### Key JSON

One JSON object, one field per primary-key column, in `schema.primary_key` order:

```json
{ "user_id": "u-001" }
```

Composite keys carry every PK column: `{ "account_id": "a-1", "region": "us" }`. This is the same shape as the existing trace `key_json` field — so the supervisor's trace emitter and the C-ABI key encoding share one encoding.

### Row JSON (`Scd2Row<R>`)

```json
{
  "payload": { "email": "a@example.com", "country": "US" },
  "valid_from": 1700000000000,
  "valid_to": null,
  "is_current": true
}
```

`payload` is a JSON object, one field per **non-PK, non-scd2** column in `schema.payload_columns` order. `valid_to` is `null` while current. Field values follow the `nanofab_node_abi::FieldValue` JSON encoding: string → JSON string, int64 → JSON number, float64 → JSON number, bool → JSON bool, timestamp_ms → JSON number (epoch ms), null → JSON null.

### `append_current` row JSON

Same `payload` object shape; `append_current` synthesizes `valid_from = event_ts`, `valid_to = null`, `is_current = true` on the supervisor side, so the `row_json` passed to `append_current` carries **only the `payload` object** (not the full `Scd2Row` envelope).

## Error propagation

- A callback returning `< 0` is a supervisor-side failure. The node-side shim maps it to `NodeError::StateWriteFailed(...)` — the retryable class — which `nanofab_node_process` already maps to `NANOFAB_NODE_RETRY = 3`.
- `get_current` returning `1` (no current row) is **not** an error — the shim returns `None` from `NodeCtx::get_current`.
- `get_current` returning `2` (buffer too small) is handled inside the shim: it reads `*out_len`, reallocates, and re-calls. It is never surfaced as a `NodeError`.
- A panic inside a callback must not unwind across the `extern "C"` boundary. The supervisor side wraps each callback body in `catch_unwind` and returns a negative code on catch. (Symmetric with the node side's existing `catch_unwind` in `nanofab_node_process`.)

## Lifetime + safety contract

- `ctx_ptr` is valid **only for the duration of one `nanofab_node_process` call.** The node must not retain it, the `ctx_handle`, or any of the function pointers past return.
- `ctx_handle` is bound to **exactly one shard's KV.** The supervisor constructs a fresh `NanofabNodeCtxVTable` per `(node, shard, event)` dispatch — or per shard, reused across that shard's events; either is contract-conformant, the node must not assume which.
- All `*const u8` buffers passed *into* a callback are valid for `..._len` bytes for the duration of that callback only.
- `out_row_json` passed to `get_current` is writable for `out_row_json_cap` bytes; the callback writes at most that many and sets `*out_len` to the bytes written (or, on code `2`, to the required length).
- Both sides declare `NanofabNodeCtxVTable` `#[repr(C)]` with identical field order. If resink-core changes it, AE mirrors it in `lib.rs.tmpl` in lock-step (the same discipline the four `NANOFAB_NODE_*` status codes already follow).

## Phase 1b deliverables this contract gates

- **AE** — extend `lib.rs.tmpl`'s `nanofab_node_process`: cast `ctx_ptr` to `*mut NanofabNodeCtxVTable`, build a `CAbiCtxShim` that `impl NodeCtx<K,R>` over the three callbacks, pass it to `node.process(ev, &mut shim)` instead of `DefaultProcessCtx`. Keep `DefaultProcessCtx` behind the existing `#[cfg(test)]` smoke path + tolerate a null `ctx_ptr` (fall back to `DefaultProcessCtx`) so the template's standalone smoke test still runs.
- **resink-core** — implement `NodeCtxBridge` (`node_runner.rs`): construct a `NanofabNodeCtxVTable` whose `ctx_handle` is a `*mut ShardKv`, whose three callbacks deserialize the JSON, mutate the `ShardKv`, and emit `Mutation` trace records. Wire `NodeRunner::process_event` to build the v-table, serialize the event to `event_json`, and call `PluginNode::process(event_json, ctx_ptr)`.

## Verified-against-environment

(Per ADR-2026-05-16-003.)

- **This loop (2026-05-14-0857):** contract authored by resink-core, co-authored by AE. AE confirmed the three callbacks + `NodeError` propagation are implementable against `lib.rs.tmpl`'s `nanofab_node_process` shape (see the AE co-author confirmation below). **No end-to-end execution this loop** — the v-table is declared in `node_runner.rs` but the bridge call site is `unimplemented!("Phase 1b")`.
- **Phase 1b (next loop):** end-to-end verification gate — the `#[ignore]`'d `supervisor_runs_three_dims` test in `crates/nanofab-supervisor/tests/general_tables.rs` is un-ignored and must pass: a synthetic `dim_widget` runs through `Supervisor::run` via the dlopen C-ABI + this v-table, and the existing `dim_user` / `dim_account` output stays byte-stable.
- Toolchain: Rust 2021, MSRV 1.83 (`[workspace.package]`). OS-agnostic (the v-table is `#[repr(C)]`; no platform-specific layout assumptions). No external deps on either side (JSON hand-rolled, consistent with the template invariant).

## AE co-author confirmation

**Confirmed implementable against `lib.rs.tmpl` (loop 2026-05-14-0857).** AE reviewed the three callbacks + error propagation against `nanofab_node_process` (template lines ~516-545) and `DefaultProcessCtx` (lines ~553-600). Findings: (1) **No C-ABI signature change needed** — `nanofab_node_process` already takes `ctx_ptr: *mut std::ffi::c_void`; Phase 1b only reinterprets it as `*mut NanofabNodeCtxVTable` instead of ignoring it. The v-table passes **by pointer**, as specified. (2) **JSON serialization is the correct call** — `Cargo.toml.tmpl` mandates zero external deps; the template already hand-rolls `parse_event_json`, so the `CAbiCtxShim` reuses that JSON machinery for keys/rows rather than pulling in `serde` or defining `repr(C)` row structs (which would couple the template to per-schema layouts). (3) **`NodeError` never crosses `extern "C"`** — it is a pure-Rust type that stays inside the cdylib; the shim maps callback `i32 < 0` → `NodeError::StateWriteFailed` on the node side, and `nanofab_node_process` already maps `NodeError` → the four status codes. (4) **Panic containment is already present** — `nanofab_node_process` wraps the whole call in `catch_unwind`, so a shim-internal panic is contained without extra template work; the supervisor side still wraps its callback bodies in `catch_unwind` per the contract. (5) **Null-`ctx_ptr` fallback** — AE will keep `DefaultProcessCtx` behind the `#[cfg(test)]` smoke path and have `nanofab_node_process` fall back to it when `ctx_ptr.is_null()`, so the template's standalone smoke test (`tests` in `lib.rs.tmpl`) keeps passing with no supervisor present. **One constraint AE imposes:** the v-table field order is frozen by this contract — AE mirrors `NanofabNodeCtxVTable` inline in `lib.rs.tmpl` verbatim, and any future field change is a lock-step edit on both sides (same discipline as the `NANOFAB_NODE_*` status-code constants). **Phase 1b template-change deliverable accepted** — see AE OKR O2 / KR2.3.
{% endraw %}
