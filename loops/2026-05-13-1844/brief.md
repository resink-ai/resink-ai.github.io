---
layout: default
title: CEO Brief — 2026-05-13-1844
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-1844
owner: board
grand_parent: Loops
parent: Loop 2026-05-13-1844
nav_order: 1
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1844
  links: parent: ""
-->
{% raw %}

# Resink.ai CEO Brief — 2026-05-13 (loop 2026-05-13-1844)

## Context

Sixth loop on the same calendar date. The prior loop (`-1422`) opened the streaming-event-source architectural arc design-first: ADR-2026-05-13-001 + a ~700-line design spec at `docs/superpowers/specs/2026-05-13-streaming-event-source-design.md`. The ADR's four-step restoration plan has step 1 targeted at `loop+1`. **This is loop+1.** This loop executes step 1.

**The shape of this loop.** **Code-first; single team active (resink-core); single objective.** No new ADR; no new spec. The contract was filed last loop; this loop's job is to execute against it without re-litigation.

**Step 1's contract**, per ADR-2026-05-13-001 § Decision + spec §9 § "Step 1":

- Introduce the `EventSource` trait in `crates/nanofab-supervisor/src/event_source/mod.rs` per spec §2 (full Rust signatures: `start`, `poll_events`, `commit_offsets`, `shutdown` + `EventBatch` struct + opaque `CommitToken` + `EventSourceError` enum).
- Ship `ParquetReplay` as the first concrete impl per spec §5.1 (wraps existing `read_fact_parquet` + `events_for` logic; surfaces `EndOfStream` once the parquet input is drained).
- Wire the supervisor to consume events through the trait per spec §7 (the `main.rs` event-loop replaces the eager `events_for(specs)` call with a `poll_events()` loop driving a `Box<dyn EventSource>`).
- `make mvp-loop` regression-free: `verdict=pass mismatches=0`. The existing two-dim canonical fixture must still pass under the new trait-mediated event-flow path.

**Probe results entering the loop:**

- Existing event-source code lives at `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/event_source.rs` — 266 lines, single file. Public surface: `RawFieldValue` / `RawOp` / `RawEvent` / `FactStreamSpec` / `read_fact_parquet(spec) -> Result<Vec<RawEvent>>` / `events_for(specs) -> Result<Vec<RawEvent>>` / `partition` / `key_value_for` / `xxh64_hash`. Both `read_fact_parquet` and `events_for` are eager. Five call sites in `main.rs` (lines 24, 112-116, 161, 165-186) reference `event_source::*` types and call `events_for(&specs)`.
- The runtime spec's documented contract sits at `docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md` §3/§4/§5.4/§6; the new design spec at `docs/superpowers/specs/2026-05-13-streaming-event-source-design.md` is what this loop's code executes against — cite by section when uncertain.
- Tenant-isolation entering: clean (single canonical `acme.ai` placeholder at `org-os/conventions.md:121`).

**Re recent retro carryovers:**

- **2026-05-13-1422 retro P2 (arc-report-type ADR) + P3 (link-existence smoke):** explicitly bundled for "next non-resink-core loop." This loop **is** a resink-core loop. **Defer both.** Don't bundle into a step-1 code loop — keeps the focus tight.
- **2026-05-13-1422 retro P4 + P5 (spec-first discipline + `unimplemented!("loop+N")` convention):** deferred per codification-when-motivated; nothing to do this loop.
- **2026-05-13-1303 retro P2 (fresh-clone verification):** opportunistic; not this loop.
- **2026-05-13-1022 retro P5/P6 (cargo CI cross-repo, branch protection):** user-only decisions; not this loop.

**CEO decisions for this loop:**

- **Activate one team: resink-core. Board active for brief + exec + retro.** All other teams paused. DE + devops + sim-farm are downstream of later restoration-plan steps (Kafka broker, end-to-end driven verdict) and don't act this loop.

- **One objective.** O1: ship step 1 of the restoration plan. Sized M. No bundling — single-objective scope is the discipline that lets step 1 surface trait-shape gaps cleanly.

- **The build is allowed to revise the trait shape if first-contact surfaces a gap.** The spec's three authoring revisions caught the obvious gaps; first-impl contact may surface a fourth. If the build phase finds the trait needs a tweak (e.g., a new `EventSourceError` variant motivated by `ParquetReplay`'s actual error surface, or a richer `EventBatch` field needed for the supervisor to advance offsets), update the spec in the same loop. The retro records what changed and why. **Spec drift is allowed when it's structurally motivated by first contact; spec drift is not allowed for "wouldn't it be nicer if…" reasoning.**

- **The legacy event-source.rs becomes a directory.** Existing module becomes `src/event_source/mod.rs` exporting the trait + types; existing `read_fact_parquet` / `events_for` / `RawEvent` / `RawOp` / `RawFieldValue` / `FactStreamSpec` / `partition` / `key_value_for` / `xxh64_hash` move into `src/event_source/raw.rs` (or are re-exported from there); `ParquetReplay` impl in `src/event_source/parquet_replay.rs`. The decision **NOT** to delete the old `events_for` helper this loop: future steps may build `InMemoryStream` (step 2) and `Kafka` (step 3) on the same underlying `RawEvent` + parquet-reading primitives; keep them as private helpers. Removal happens only after step 4 (end-to-end Kafka-driven verdict).

- **Tenant-isolation invariant maintained.** Zero `org-os/` edits expected — all changes under `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/` + the resink-core team's OKR + exec.

**Standing CEO answers:**

- **Don't re-litigate the trait shape unless first contact forces it.** The spec is the contract. If `ParquetReplay` types out cleanly against spec §2 + §5.1, ship it. If a real gap surfaces (not aesthetic), name it in the OKR, update the spec, narrate the revision in the retro.

- **Supervisor integration is the load-bearing piece.** The trait extraction is mechanical; the supervisor `main.rs` change is the place where the new shape meets reality. The `poll_events()` loop must preserve the existing semantics: gather all events → run plugin → emit verdict. Step 1 explicitly does NOT introduce streaming back-pressure or partial-batch handling; `ParquetReplay` returns one batch with all events + then `EndOfStream`.

- **Test surface for step 1.** `make mvp-loop` green is the load-bearing test. Additionally: keep the existing `cargo test --workspace` green. The trait-level unit tests (mocked `EventSource` driving the supervisor) belong to step 2 (`InMemoryStream` + integration test); step 1 doesn't need them.

**Carryover load by team:**

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| resink-core | O1 step-1 build (trait + ParquetReplay + supervisor wire + mvp-loop green) | M | Execute against spec §2 + §5.1 + §7; spec drift allowed if structurally motivated. |
| board | Brief + exec + retro | S | Loop accounting only. |
| All other teams | Paused; no review-ack ask | — | Paused. DE / devops / sim-farm are downstream of later steps. |

## Objectives

### O1: Streaming event-source — step 1 (`EventSource` trait + `ParquetReplay` impl + supervisor wire)

source: ceo-brief
owner: resink-core
sized: M
target loop: 2026-05-13-1844 (this loop)
links.source: board/decisions/2026-05-13-001-streaming-event-source-rearchitecture.md

Why it matters: First step of the four-step streaming restoration plan filed last loop. The trait extraction is the load-bearing handoff — every subsequent step (`InMemoryStream`, `Kafka`, end-to-end verdict) executes against the trait. Getting the trait shape right + the supervisor's consumption shape right is what makes the remaining three steps mechanical execution rather than re-litigation.

KR1.1: New module layout at `crates/nanofab-supervisor/src/event_source/` (was: single-file `event_source.rs`). At minimum: `mod.rs` (trait + `EventBatch` + `CommitToken` + `EventSourceError` per spec §2), `raw.rs` (existing `RawEvent` / `RawOp` / `RawFieldValue` / `FactStreamSpec` types + `xxh64_hash` / `partition` / `key_value_for` helpers — relocated verbatim from existing event_source.rs), `parquet_replay.rs` (the `ParquetReplay` impl per spec §5.1).

KR1.2: `EventSource` trait fully implemented per spec §2. Method set: `start(&mut self) -> Result<(), EventSourceError>`, `poll_events(&mut self) -> Result<EventBatch, EventSourceError>`, `commit_offsets(&mut self, token: CommitToken) -> Result<(), EventSourceError>`, `shutdown(&mut self) -> Result<(), EventSourceError>`. `EventBatch { events, commit_token, low_watermark }`. `EventSourceError::{EndOfStream, Retryable, Fatal}`. If first contact surfaces a structurally-motivated revision, update spec §2 in the same loop and narrate in the retro.

KR1.3: `ParquetReplay` impl per spec §5.1 — wraps the existing parquet-eager logic. `start()` validates input paths; `poll_events()` returns one `EventBatch` with all events on first call + `EventSourceError::EndOfStream` on subsequent calls; `commit_offsets()` is a no-op (offsets aren't meaningful for replay); `shutdown()` clears internal state.

KR1.4: Supervisor wired through the trait per spec §7. The `main.rs` event-loop replaces direct calls to `event_source::events_for(&specs)` with a `Box<dyn EventSource>` constructed from a `ParquetReplay::new(specs)` factory + a `start() → poll_events() loop → handle EndOfStream → shutdown()` lifecycle. Existing `op_from_str` + `FactStreamSpec` parsing logic preserved.

KR1.5: `make mvp-loop` green: `verdict=pass mismatches=0`. The canonical two-dim (`dim_user` + `dim_account`) fixture passes under the new trait-mediated path with byte-identical output to the prior eager path.

KR1.6: `cargo test --workspace --release` passes. No regressions in `dlopen_integration` or `hot_swap_correctness` test suites.

KR1.7: Tenant-isolation invariant holds. Zero `org-os/` edits this loop. All changes under `repos/resink-ai/resink-core/` (the resink-core remote) and `teams/application/resink-core/`. Final sweep returns the single canonical `acme.ai` placeholder.

## What success looks like

- Step 1 of ADR-2026-05-13-001's four-step restoration plan shipped. ADR body gains a `## Status (2026-05-13, loop 2026-05-13-1844)` section narrating step-1 closure + carryover to step 2.
- `crates/nanofab-supervisor/src/event_source/` exists as a directory with at minimum `mod.rs`, `raw.rs`, `parquet_replay.rs`; the trait + the impl + the supervisor wire are all in tree.
- `make mvp-loop` green; `cargo test --workspace --release` green.
- If the spec needed a structurally-motivated revision: spec updated in the same commit; retro narrates the revision.
- Tenant-isolation grep returns single canonical placeholder.
- Retro surfaces: (i) did the spec-first authoring pay off — was the trait shape right on first contact, or did first contact force a revision? This is the key signal for retro P4 from last loop (spec-first discipline as memory'd practice); (ii) did the supervisor-integration shape match spec §7, or did `main.rs`'s actual structure force a layout change?

## Out of scope

- Step 2 (`InMemoryStream` impl + integration test) — next loop or whenever resink-core has bandwidth.
- Step 3 (`Kafka` impl + broker) — multi-loop work involving devops; not this loop.
- Step 4 (end-to-end Kafka-driven verdict) — final step of the arc; not this loop.
- Deletion of legacy `events_for` / `read_fact_parquet` helpers — keep as private helpers; subsequent steps build on them. Removal post-step-4.
- Any test surface beyond `make mvp-loop` green + `cargo test --workspace` green. Trait-level mock-driven unit tests belong to step 2 (`InMemoryStream`).
- Bundling carryover ADRs (2026-05-13-1422 retro P2 arc-report-type; P3 link-existence smoke). Both stay queued for next non-resink-core loop.
- Any `org-os/` process work. Org-os surface stable; this loop is product code, not process.
- Any review-ack from sim-farm / DE / devops / AE. Step 1 is internal to resink-core.
{% endraw %}
