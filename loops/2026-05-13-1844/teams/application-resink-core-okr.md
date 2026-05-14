---
layout: default
title: application-resink-core OKR — 2026-05-13-1844
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-1844
owner: teams/application/resink-core
grand_parent: Loops
parent: Loop 2026-05-13-1844
nav_order: 10
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1844
  links: parent: board/okrs/2026-05-13-1844-ceo-brief.md
-->
{% raw %}

# Resink Core OKR — 2026-05-13 (loop 2026-05-13-1844)

## Context

Single-objective build loop. Step 1 of ADR-2026-05-13-001's four-step streaming restoration plan. Per the brief, this loop executes against the spec at `docs/superpowers/specs/2026-05-13-streaming-event-source-design.md` (sections §2 trait, §5.1 ParquetReplay, §7 supervisor integration) without re-litigating the trait shape unless first contact forces a structurally-motivated revision.

Probe entering: existing event_source.rs is 266 lines, single file, eager-only (`Vec<RawEvent>` returns). Five call sites in `main.rs`. The trait extraction converts this to a directory module with three files: `mod.rs` (trait + types), `raw.rs` (existing helpers verbatim), `parquet_replay.rs` (the impl wrapping the existing eager logic).

## Objectives

### O1: Streaming event-source — step 1 (`EventSource` trait + `ParquetReplay` impl + supervisor wire)

source: ceo-brief

Why it matters: The trait extraction is the load-bearing handoff for the entire four-step arc. Steps 2 (`InMemoryStream`), 3 (`Kafka` + broker), and 4 (end-to-end Kafka-driven verdict) all execute against the trait shipped this loop. Getting the trait shape + supervisor consumption shape right makes the remaining three steps mechanical execution rather than re-litigation.

Maps to brief O1 → KR1.1 (module layout), KR1.2 (trait), KR1.3 (ParquetReplay impl), KR1.4 (supervisor wire), KR1.5 (mvp-loop green), KR1.6 (cargo test green), KR1.7 (tenant-isolation).

**Key results** (KR numbering preserves traceability to brief O1)

- KR1.1: New module layout at `crates/nanofab-supervisor/src/event_source/`. Files: `mod.rs` (trait + `EventBatch` + `CommitToken` + `EventSourceError` per spec §2; re-export `raw::*`); `raw.rs` (existing `RawFieldValue` / `RawOp` / `RawEvent` / `FactStreamSpec` types + `xxh64_hash` / `partition` / `key_value_for` / `read_fact_parquet` / `events_for` helpers relocated verbatim from the existing `event_source.rs`); `parquet_replay.rs` (the `ParquetReplay` impl per spec §5.1). The single-file `event_source.rs` deleted.
- KR1.2: `EventSource` trait fully implemented per spec §2. Method set as documented (`start`, `poll_events`, `commit_offsets`, `shutdown`). Types: `EventBatch { events, commit_token, low_watermark }`, opaque `CommitToken`, `EventSourceError::{EndOfStream, Retryable, Fatal}`. If first contact surfaces a structurally-motivated revision, the spec §2 is updated in the same loop + the retro narrates the revision.
- KR1.3: `ParquetReplay` impl per spec §5.1. `start()` validates input paths; `poll_events()` returns one `EventBatch` with all events on first call + `EventSourceError::EndOfStream` on subsequent calls; `commit_offsets()` is a no-op; `shutdown()` clears internal state.
- KR1.4: Supervisor wired through the trait per spec §7. `main.rs`'s direct `event_source::events_for(&specs)` call replaced with a `Box<dyn EventSource>` constructed from `ParquetReplay::new(specs)` + a `start() → poll_events() loop → handle EndOfStream → shutdown()` lifecycle. Existing parsing (`op_from_str`, `FactStreamSpec` construction) preserved.
- KR1.5: `make mvp-loop` green: `verdict=pass mismatches=0`. The canonical two-dim (`dim_user` + `dim_account`) fixture passes under the new trait-mediated path; output byte-identical to prior eager path.
- KR1.6: `cargo test --workspace --release` passes. No regressions in `dlopen_integration` or `hot_swap_correctness` test suites.
- KR1.7: Tenant-isolation invariant holds. Zero `org-os/` edits. All code changes under `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/`. Final sweep returns the single canonical `acme.ai` placeholder at `org-os/conventions.md:121`.

**Tasks**

- [ ] Create `crates/nanofab-supervisor/src/event_source/` directory; move existing event_source.rs body into `raw.rs` (verbatim relocate, no API changes); delete the old single-file event_source.rs
- [ ] Author `mod.rs`: `EventSource` trait + `EventBatch` / `CommitToken` / `EventSourceError` types per spec §2; re-export raw types via `pub use raw::*` so existing call sites in main.rs keep their `event_source::RawEvent` paths working
- [ ] Author `parquet_replay.rs`: `ParquetReplay` struct holding the FactStreamSpec vec + a "drained" flag; `impl EventSource for ParquetReplay` per spec §5.1
- [ ] Wire `main.rs`: replace `let events = event_source::events_for(&specs)?` with `Box<dyn EventSource>` lifecycle (construct → start → poll until EndOfStream → shutdown)
- [ ] `cargo build --release` clean; `cargo test --workspace --release` green
- [ ] `make mvp-loop`; verify `verdict=pass mismatches=0`
- [ ] If first-contact revisions to the trait shape: update spec §2 in the same commit; note revisions for the retro
- [ ] Append `## Status (2026-05-13, loop 2026-05-13-1844)` to ADR-2026-05-13-001 narrating step-1 closure + carryover to step 2

## What success looks like

Step 1 of the streaming restoration plan landed in tree. The trait + `ParquetReplay` + supervisor consumption shape are all observable in `crates/nanofab-supervisor/src/event_source/`. `make mvp-loop` green; cargo workspace tests green. ADR-2026-05-13-001 gains its step-1-closure status section. Spec drift (if any) is narrated honestly in the retro.

## Out of scope

- Steps 2-4 of the restoration plan (`InMemoryStream`, `Kafka`, end-to-end Kafka-driven verdict).
- Deletion of `events_for` / `read_fact_parquet` helpers — keep as private helpers; subsequent steps build on them.
- Trait-level mock-driven unit tests — belong to step 2 (`InMemoryStream` + integration test).
- Any test surface beyond `make mvp-loop` + `cargo test --workspace`.
- Cross-team coordination — step 1 is internal to resink-core.
{% endraw %}
