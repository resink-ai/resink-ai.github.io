---
layout: default
title: CEO Brief — 2026-05-13-1944
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-1944
owner: board
grand_parent: Loops
parent: Loop 2026-05-13-1944
nav_order: 1
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1944
  links: parent: ""
-->
{% raw %}

# Resink.ai CEO Brief — 2026-05-13 (loop 2026-05-13-1944)

## Context

Seventh loop on the same calendar date. Step 1 of ADR-2026-05-13-001's four-step streaming restoration plan closed cleanly one loop ago (`-1844`): `EventSource` trait + `ParquetReplay` impl + supervisor wired via `drain_to_vec` bridge; `make mvp-loop` `verdict=pass`; three first-contact spec revisions narrated. **This is loop+1 from step 1; the contract is step 2.**

**The shape of this loop.** **Code-first; single team active (resink-core); single objective (multi-KR).** Per CEO option-set, this loop pulls forward the `Supervisor` struct API extraction that the spec assigns to step 4. Rationale: spec §8's integration-test example is `Supervisor::new(...).with_source(Box::new(source)).run()`. Step 2 ships the verbatim spec §8 test, which requires the `Supervisor` API. Pulling the extraction forward gets the right shape into the tree now; step 3 (Kafka) + step 4 (end-to-end Kafka verdict) become smaller because the supervisor API surface is settled.

**Step 2's contract** (per ADR-2026-05-13-001 § Decision + spec §5.2, §7, §8 + prior retro P1):

- **`InMemoryStream` impl** per spec §5.2 — mpsc-based sender/receiver pair. `start()` no-op; `poll_events()` returns the next batch from the receiver, an empty batch on `TryRecvError::Empty`, `EndOfStream` on `TryRecvError::Disconnected`; `commit_offsets` no-op; `shutdown` no-op. Factory pattern: `InMemoryStream::pair() -> (Sender<EventBatch>, InMemoryStream)`.
- **`Supervisor` struct API** extracted from `main.rs`'s inline `run()` function — builder shape: `Supervisor::new(config: SupervisorConfig)` returning a builder; `.with_source(Box<dyn EventSource>)` to override the default `ParquetReplay`; `.run() -> Result<RunResult, String>`. `main.rs` becomes a thin shell: parse `Args`, build config, build supervisor, invoke `.run()`, exit.
- **Unified-source model.** Today's flow constructs a `ParquetReplay` per node-spec (one source per fact-stream-spec); step 2 collapses this to ONE source per supervisor instance that yields events tagged by `RawEvent.table`. The supervisor routes by `event.table` to per-table buffers. `ParquetReplay::from_node_specs(node_specs, fixtures_dir)` is the new constructor that gathers all fact streams from all nodes into a single (globally-sorted) source.
- **Supervisor outer loop per spec §7.** Replace each per-node `drain_to_vec` call with a single inline poll loop: `source.start()? → loop { match source.poll_events() { Ok(batch) → route events by table to per-table buffers + commit; Err(EndOfStream) → break; Err(Retryable) → log+sleep+continue; Err(Fatal) → return err; } } → source.shutdown()? → per-table nodes::run() → write outputs`. The per-table buffering stays inside the supervisor; node `run()` functions stay slice-based per their current API.
- **Integration test** at `crates/nanofab-supervisor/tests/streaming_event_source.rs` — verbatim spec §8 shape. Constructs `InMemoryStream::pair()`, builds a `Supervisor` with the in-memory source, pushes a sequence of synthetic batches (mix of `dim_user` + `dim_account` events), drops the sender to signal `EndOfStream`, invokes `.run()`, asserts on the resulting per-dim output parquets against an inline-defined expected shape. No fixture file dependency.
- **Spec §2 edit** — update the code blocks to reflect step 1's in-tree shape: manual `Display` + `Error` impls (no `thiserror` derive); bare `pub enum CommitToken` (no newtype wrap). Add a brief note about `drain_to_vec` being removed in step 2 (replaced by the inline poll loop).
- **`make mvp-loop` regression-free** — `verdict=pass mismatches=0` on the canonical two-dim fixture with the new `Supervisor` API + `ParquetReplay::from_node_specs` driving it.

**Probe results entering the loop:**

- Step-1 in-tree: `crates/nanofab-supervisor/src/event_source/` with `mod.rs` (trait + types + `drain_to_vec`), `raw.rs` (existing types), `parquet_replay.rs` (impl + 4 unit tests). `main.rs` calls `ParquetReplay::new(specs.clone()) + drain_to_vec(&mut source)` per-node-spec (5 call sites under the `for node_spec in &dag.nodes` loop).
- `nodes.rs`: `nodes::user::run(events: &[RawEvent], shard_count, &mut trace_writer, &user_out_path)` and `nodes::account::run(...)` — both slice-based; structurally unchanged.
- ADR-2026-05-13-001 Status section narrates step-1 closure + three first-contact revisions. Spec §2 still shows the pre-revision code (`#[derive(thiserror::Error)]` + newtype `CommitToken`).
- Tenant-isolation entering: clean (single canonical `acme.ai` placeholder at `org-os/conventions.md:121`).

**Re recent retro carryovers:**

- **2026-05-13-1844 retro P1:** **This loop**. Step 2 of streaming restoration.
- **2026-05-13-1844 retro P2 (multi-loop-blocker-arc report-type ADR):** Strongly motivated (three worked instances now). Carryover. **Defer to next non-resink-core loop.**
- **2026-05-13-1844 retro P3 (link-existence smoke):** Carryover. Bundle with P2.
- **2026-05-13-1844 retro P4 (spec-first pattern):** Memory entry saved this past loop. Done.
- **2026-05-13-1844 retro P5 (`drain_to_vec`-style bridge as recognized pattern):** Demand-driven; one instance only. Not this loop.
- **2026-05-13-1303 retro P2 (fresh-clone verification):** Opportunistic; not this loop.

**CEO decisions for this loop:**

- **Activate one team: resink-core. Board active for brief + exec + retro.** All other teams paused. DE + devops + sim-farm are downstream of later steps; none act this loop.

- **One objective; six KRs.** O1 ships InMemoryStream + Supervisor API extraction + unified-source model + outer-loop restructure + integration test + spec §2 edit + mvp-loop green. **Sized L** (revised up from spec §9's M sizing because Supervisor extraction was originally a step-4 concern; bringing it forward earns step 3 + step 4 a smaller surface).

- **Pull `Supervisor` API extraction forward.** Per CEO option selection. Spec §8's integration-test example is verbatim `Supervisor::new(...).with_source(Box::new(source)).run()`; pulling extraction forward this loop gets the right API into the tree before Kafka work. The trade-off: step 2 is L not M; steps 3 + 4 shrink accordingly.

- **Unified-source model is the right shape for the streaming abstraction.** Today's per-node-spec-source model is a `ParquetReplay` quirk (one impl, one source per fact-stream); `InMemoryStream` and `Kafka` are naturally single-source (test sender pushes mixed events; Kafka consumer-group yields mixed topics). Collapsing to one source per supervisor + routing by `event.table` is what the spec §7 outer-loop shape requires.

- **Spec §2 edit lands in this loop's commit.** Step 1 narrated the revisions in the ADR Status section but didn't touch the spec body; this loop closes the gap. Future readers of the spec see the in-tree shape.

- **Tenant-isolation invariant maintained.** All changes under `repos/resink-ai/resink-core/` + spec edit at `docs/superpowers/specs/`. Zero `org-os/` edits expected.

**Standing CEO answers:**

- **`drain_to_vec` deletes.** The step-1 bridge helper is no longer needed once the supervisor's outer loop is restructured to inline per-batch flow. Mention in the ADR Status section that the bridge has retired.

- **Per-table buffering stays inside the supervisor.** Don't refactor `nodes::user::run` / `nodes::account::run` to single-event APIs this loop. Spec §7's pseudocode shows `nodes::route_event(event, shard_id)` — that's an aspirational shape; preserving the existing per-table-slice node API keeps the blast radius tight while still satisfying the per-batch outer loop at the EventSource trait surface.

- **Integration test asserts at the parquet level.** The test pushes events with known PKs + timestamps; the supervisor processes them through per-table buffers + `nodes::run()`; outputs go to test-scoped tempdirs. Test asserts: per-table row counts; per-dim PK existence; basic SCD2 invariants (one open row per PK; `valid_from < valid_to` for closed rows). Don't reuse the canonical fixture; build a small inline test sequence (e.g., 4 events: 2 dim_user inserts + 2 dim_account inserts).

- **`Supervisor::new` config shape.** A `SupervisorConfig` struct that mirrors the existing `Args` field set but without the CLI-parsing concerns. Fields: `workspace`, `write_trace`, `write_output_prefix`, `fixtures_dir`. The supervisor reads its manifest + builds its DAG internally. Tests construct `SupervisorConfig` directly without going through `clap`.

- **First-contact revisions allowed.** Same discipline as step 1: spec drift allowed when structurally motivated; narrated in the ADR Status section. The current spec §2 + §5.2 + §7 + §8 shapes are the contract; revisions narrate in `## Status` (step 2's closure section).

**Carryover load by team:**

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| resink-core | O1 step-2 build (InMemoryStream + Supervisor API + unified source + outer-loop restructure + integration test + spec §2 edit + mvp-loop green) | L | Six interlocked KRs; the API extraction is the load-bearing piece. |
| board | Brief + exec + retro + spec §2 edit | S | Loop accounting + spec maintenance. |
| All other teams | Paused; no review-ack ask | — | Paused. DE / devops / sim-farm are downstream of step 3+. |

## Objectives

### O1: Streaming event-source — step 2 (`InMemoryStream` + `Supervisor` API + integration test)

source: ceo-brief
owner: resink-core
sized: L
target loop: 2026-05-13-1944 (this loop)
links.source: board/decisions/2026-05-13-001-streaming-event-source-rearchitecture.md

Why it matters: Step 2 validates the trait's generality — `ParquetReplay` is structurally similar to the pre-step-1 eager flow; `InMemoryStream` is the first impl that exercises the trait's mpsc-driven, partial-batch, sender-disconnects-as-EndOfStream semantics. Pulling the `Supervisor::run()` API extraction forward (originally a step-4 concern) gets the spec §8 integration-test shape verbatim and pre-pays the supervisor restructure that step 3 (Kafka) + step 4 (end-to-end Kafka verdict) would have otherwise needed.

KR1.1: `InMemoryStream` impl at `crates/nanofab-supervisor/src/event_source/in_memory_stream.rs` per spec §5.2. Uses `std::sync::mpsc::{channel, Sender, Receiver}`. Factory: `pub fn pair() -> (Sender<EventBatch>, InMemoryStream)`. `start()` no-op; `poll_events()` matches `receiver.try_recv()` per spec §5.2; `commit_offsets` no-op; `shutdown` no-op. Re-exported from `event_source/mod.rs`.

KR1.2: `Supervisor` struct API extracted from `main.rs`. New module `crates/nanofab-supervisor/src/supervisor.rs` (or extension of `lib.rs`) exposing `Supervisor` + `SupervisorConfig` + `RunResult`. Builder shape: `Supervisor::new(config) → .with_source(Box<dyn EventSource>) → .run() → Result<RunResult, String>`. `main.rs` becomes ~20-line shell: parse `Args`, build `SupervisorConfig`, build `Supervisor`, invoke `.run()`, emit ok/error.

KR1.3: Unified-source model. `ParquetReplay::from_node_specs(node_specs: &[NodeSpec], fixtures_dir: &Path) -> Result<Self, String>` constructor gathers all fact streams from all nodes into a single source. Existing `ParquetReplay::new(specs)` constructor preserved for direct callers (event-source unit tests). The Supervisor's `run()` builds a `ParquetReplay::from_node_specs` by default unless `.with_source(...)` overrides.

KR1.4: Supervisor outer loop per spec §7. The `Supervisor::run()` body: `source.start()? → loop { match source.poll_events() { Ok(batch) if empty + no_wm → continue (no sleep needed in step-2; back-off concerns are step-3 Kafka territory); Ok(batch) → for event in batch.events: dispatch to per-table buffer; source.commit_offsets(batch.commit_token)?; Err(EndOfStream) → break; Err(Retryable(_)) → return err (step-2 only impls don't construct Retryable; treat as fatal for now per step-1 precedent); Err(Fatal(msg)) → return err; } } → source.shutdown()? → per-table nodes::user::run + nodes::account::run → write outputs → return RunResult { user_rows, account_rows }`. `drain_to_vec` removed from `event_source/mod.rs`.

KR1.5: Integration test at `crates/nanofab-supervisor/tests/streaming_event_source.rs` per spec §8. Test body: `(sender, source) = InMemoryStream::pair(); supervisor = Supervisor::new(test_config).with_source(Box::new(source)); thread::spawn(|| { sender.send(batch1); sender.send(batch2); drop(sender); }); let result = supervisor.run().unwrap(); assert!(result.user_rows > 0); assert!(result.account_rows > 0);`. Test fixture is inline-constructed `RawEvent`s + a minimal manifest at a `tempdir`. Verifies: trait surface works end-to-end through the Supervisor; `EndOfStream` shutdown sequence works; per-table routing works.

KR1.6: Spec §2 edit at `docs/superpowers/specs/2026-05-13-streaming-event-source-design.md`. Code blocks in §2 updated: `EventSourceError` shows manual `Display` + `std::error::Error` impls (not `thiserror::Error` derive); `CommitToken` shows bare `pub enum` variants (not newtype-wrapped). A new bullet in "Why these methods + this shape" notes that step 2 retired the `drain_to_vec` step-1 bridge in favor of the inline poll loop per §7. Step 2's `## Status` section in the ADR notes the spec edit.

KR1.7: `make mvp-loop` green: `verdict=pass mismatches=0`. The canonical two-dim fixture passes under the new `Supervisor` API + unified-source `ParquetReplay::from_node_specs` driver. Output byte-identical to the step-1 path.

KR1.8: `cargo test --workspace --release` passes. No regressions in `dlopen_integration`, `hot_swap_correctness`, or `determinism` test suites. New `InMemoryStream` unit tests + new `streaming_event_source` integration test added; both green.

KR1.9: ADR-2026-05-13-001 body gains `## Status (2026-05-13, loop 2026-05-13-1944)` step-2 closure section. Narrates: what shipped per spec section; any first-contact revisions; the spec §2 edit; carryover to step 3 (Kafka).

KR1.10: Tenant-isolation invariant holds. Zero `org-os/` edits this loop. All code under `repos/resink-ai/resink-core/`; spec edit under `docs/superpowers/specs/`. Final sweep returns single canonical `acme.ai` placeholder.

## What success looks like

- `crates/nanofab-supervisor/src/event_source/in_memory_stream.rs` exists with the impl + at least 3 unit tests (pair roundtrip; empty-batch on no-events; EndOfStream on sender-drop).
- `crates/nanofab-supervisor/src/supervisor.rs` exists with `Supervisor` + `SupervisorConfig` + `RunResult`; `main.rs` reduced to a thin shell.
- `crates/nanofab-supervisor/src/event_source/mod.rs` no longer exports `drain_to_vec` (or the helper is deleted entirely).
- `crates/nanofab-supervisor/tests/streaming_event_source.rs` exists and passes; exercises `InMemoryStream::pair()` + `Supervisor::run()` end-to-end.
- `make mvp-loop`: `verdict=pass mismatches=0`. Output bytes unchanged from step-1 baseline.
- `cargo test --workspace --release` green.
- Spec §2 code blocks reflect step-1's in-tree shape; ADR Status section narrates step-2 closure + any further first-contact revisions.
- Retro surfaces: (i) did the `Supervisor` API extraction stay within sized-L scope, or did it spiral? (ii) did `InMemoryStream`'s test surface validate the trait's generality, or did it surface a fourth revision? (iii) was pulling the supervisor extraction forward worth the step-2 cost — does step 3 (Kafka) materially shrink because the API is settled?

## Out of scope

- Step 3 (`Kafka` impl + broker provisioning). Multi-loop work involving devops; not this loop.
- Step 4 (end-to-end Kafka-driven verdict). Final restoration step.
- Refactoring `nodes::user::run` / `nodes::account::run` to single-event APIs. Spec §7's `nodes::route_event(event, shard_id)` shape is aspirational; per-table-slice API is preserved this loop.
- Watermark tracker wiring. The trait surfaces `low_watermark` but the supervisor still ignores it this loop. Watermark consumption is a step 3-4 concern (motivated by Kafka's real per-partition watermarks).
- Back-pressure / retry-with-backoff on `Retryable` errors. Step 2's impls (`InMemoryStream`, `ParquetReplay`) don't construct `Retryable`; step 3 (Kafka) is where backoff matters.
- Watermark publishing from `ParquetReplay::from_node_specs`. The impl currently sets `low_watermark = batch.events.iter().map(|e| e.event_ts).min()`; the new constructor preserves this. The supervisor doesn't act on it yet.
- Bundling P2 (arc-report-type ADR) + P3 (link-existence smoke). Both stay queued for next non-resink-core loop.
- Any `org-os/` process work.
- Cross-team review-ack. Step 2 is internal to resink-core.
- The `Kafka` impl skeleton or the `kafka-source` Cargo feature flag scaffolding. Both arrive in step 3.
{% endraw %}
