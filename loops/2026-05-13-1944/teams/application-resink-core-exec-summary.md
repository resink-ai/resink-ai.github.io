---
layout: default
title: application-resink-core Exec Summary — 2026-05-13-1944
date: 2026-05-13
status: active
type: exec-summary
loop: 2026-05-13-1944
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/resink-core
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1944
  links: parent: board/exec-summaries/2026-05-13-1944.md
-->
{% raw %}

# Resink Core Exec Summary — 2026-05-13 (loop 2026-05-13-1944)

**Headline.** Step 2 of ADR-2026-05-13-001's four-step streaming restoration plan shipped in one resink-core build session — sized L when CEO pulled the `Supervisor` API extraction forward from step 4. `InMemoryStream` impl + `Supervisor` struct API + unified-source model + per-batch outer loop + `streaming_event_source` integration test + spec §2 edit all landed. `make mvp-loop`: `verdict=pass mismatches=0`; `cargo test --workspace --release` green (5 new InMemoryStream unit tests + 3 new integration tests + 1 new ParquetReplay test, plus all prior tests). **Zero new first-contact spec revisions** — the trait surface accommodated `InMemoryStream` cleanly. Tenant-isolation invariant CLEAN.

## Per-objective rollup

### O1: Streaming event-source — step 2 — ✅ PASS

All 10 KRs cleared. Highlights:

- **KR1.1 (InMemoryStream): PASS.** mpsc-backed impl per spec §5.2 with `InMemoryStream::pair()` factory; 5 unit tests passing.
- **KR1.2 (Supervisor API): PASS.** `Supervisor::new(config).with_source(Box<dyn EventSource>).run() → Result<RunResult, String>` extracted to `src/supervisor.rs`. Manifest types (`Manifest`, `DagSpec`, `NodeSpec`, `ManifestFactStream`) relocated from `main.rs`. `lib.rs` exposes `supervisor` + `event_source` (nodes + trace stay `pub(crate)`).
- **KR1.3 (unified-source `from_specs`): PASS.** New `ParquetReplay::from_specs(specs)` constructor. The supervisor's default source path gathers all fact-streams from all node-specs into one unified source yielding events tagged by `RawEvent.table`.
- **KR1.4 (outer-loop restructure): PASS.** `Supervisor::run()` body matches spec §7 (start → poll loop with `Empty + EndOfStream + Retryable + Fatal` match arms → per-table buffer routing → shutdown → per-table `nodes::run`). `drain_to_vec` deleted.
- **KR1.5 (integration test): PASS.** `tests/streaming_event_source.rs` with 3 cases passing per spec §8 verbatim. Drives `Supervisor::run()` via `InMemoryStream::pair()` with no fixture parquets.
- **KR1.6 (spec §2 edit): PASS.** Code blocks now match in-tree shape: manual `Display`/`Error` impls; bare-enum `CommitToken`. New bullet on `drain_to_vec` retirement in "Why these methods + this shape."
- **KR1.7 (mvp-loop green): PASS.** `verdict=pass mismatches=0`. Output bytes identical to step-1 baseline.
- **KR1.8 (cargo test green): PASS.** Full workspace test run green; no regressions.
- **KR1.9 (ADR step-2 closure section): PASS.** ADR-2026-05-13-001 gains `## Status (2026-05-13, loop 2026-05-13-1944)` step-2 closure section narrating what shipped + the three architectural decisions taken during the build.
- **KR1.10 (tenant-isolation): PASS.** Zero `org-os/` edits.

## Build phase notes

**Zero new first-contact spec revisions** vs step 1's three. The trait surface accommodated `InMemoryStream` cleanly — mpsc's `TryRecvError::Empty` maps directly to `Ok(empty_batch)`; `TryRecvError::Disconnected` maps directly to `Err(EndOfStream)`. The empty-batch semantics in spec §3 worked verbatim. The `Supervisor::run()` outer loop matched spec §7's shape verbatim.

**Three architectural decisions taken during the build** (narrated but distinct from first-contact revisions — these are deliberate scope-shaping calls, not contract corrections):

1. **Unified-source model.** Step 1 had one `ParquetReplay` per node-spec; step 2 collapses to one source per supervisor instance, yielding events tagged by `RawEvent.table`. The supervisor routes via `HashMap<String, Vec<RawEvent>>` per-table buffers. This shape is what `InMemoryStream` + `Kafka` naturally produce (mixed events from one channel/topic-subscription); the per-node-source model was a `ParquetReplay`-specific quirk.
2. **Per-table buffering kept inside the Supervisor; node APIs unchanged.** Spec §7's pseudocode shows `nodes::route_event(event, shard_id)` — single-event. Actual `nodes::user::run` / `nodes::account::run` take slices. Refactoring nodes to single-event APIs would be substantive (touching `nodes::user`'s ShardKv + panic::catch_unwind wrapping); no forcing function this step. The supervisor buffers + dispatches at shutdown.
3. **`Retryable` errors treated as fatal in step 2.** Step 2's impls don't construct `Retryable`. The supervisor's poll loop returns an error if it sees one. Step 3 (Kafka) adds the real backoff loop.

**Build artifacts:**

- New: `crates/nanofab-supervisor/src/event_source/in_memory_stream.rs` (~125 lines).
- New: `crates/nanofab-supervisor/src/supervisor.rs` (~280 lines).
- New: `crates/nanofab-supervisor/tests/streaming_event_source.rs` (~200 lines).
- Modified: `crates/nanofab-supervisor/src/event_source/mod.rs` (re-export `InMemoryStream`; delete `drain_to_vec`).
- Modified: `crates/nanofab-supervisor/src/event_source/parquet_replay.rs` (add `from_specs` constructor).
- Modified: `crates/nanofab-supervisor/src/lib.rs` (expose `event_source`, `supervisor`, `nodes(pub(crate))`, `trace(pub(crate))`).
- Modified: `crates/nanofab-supervisor/src/main.rs` (slimmed from ~240 lines to ~85 lines; thin shell).
- Modified: `docs/superpowers/specs/2026-05-13-streaming-event-source-design.md` § 2 + "Why these methods" bullets.

**Test surface:**

- Unit tests new: 5 (`InMemoryStream`) + 1 (`ParquetReplay::from_specs` infallibility). 16 total in `nanofab-supervisor` lib tests (up from 9 at step 1's close).
- Integration tests new: 3 (`streaming_event_source.rs`). 3 total streaming integration tests; prior `dlopen_integration` + `hot_swap_correctness` + `determinism` test files unchanged in shape.
- `make mvp-loop`: `verdict=pass mismatches=0`.

## What didn't / risks for step 3

- **The `Retryable`-as-fatal path is unproven.** Step 2 has no impl that constructs `Retryable`; step 3 (Kafka) is the first that will. The supervisor's poll loop currently treats `Retryable` as fatal — when Kafka starts emitting transient errors during partition rebalance, the supervisor needs a real backoff-then-retry path. **Action:** step 3's brief should include the `Retryable` backoff loop as an explicit KR.

- **Single-source model assumes per-batch `RawEvent.table` tagging is correct.** The supervisor routes by `event.table`; if a Kafka impl somehow yielded events with an unknown table (topic-to-table mapping bug), the supervisor currently has no graceful failure path — events would be dropped into a buffer that's never dispatched. **Trivial:** the manifest's node-spec list is the authoritative table set; the supervisor validates at startup that every node-spec's table is known. Step 3 should add: validate inbound `event.table` against the manifest's allow-list, log + skip + count unknowns instead of silently dropping.

- **Per-table buffering doesn't yet exercise streaming back-pressure.** Step 2 buffers all events before dispatch; if a real Kafka stream produces millions of events, this is a memory bomb. Step 4 (end-to-end Kafka-driven verdict) is where the supervisor's outer loop will need to dispatch per-batch to `nodes::*::process_event(event)` style APIs (single-event) — and that requires refactoring `nodes::user::run` / `nodes::account::run`. **Mild:** spec §7's aspirational shape is single-event-routing; this debt is properly scoped to step 4.

## Asks for the CEO / next loop's brief

- **(from resink-core, next loop):** Step 3 of the restoration plan — `Kafka` impl behind a `kafka-source` Cargo feature flag + broker provisioning. Resink-core (client) + devops (broker). Sized L (originally; should be smaller now that the `Supervisor` API + unified-source model are settled). The implementing loop's first decision: `rdkafka` vs `rskafka` based on consumer-group feature completeness. Target loop+1 or loop+2 depending on broker provisioning lead time.
- **(from resink-core, step-3 sub-asks):** (a) include `Retryable` backoff loop in `Supervisor::run` as an explicit KR; (b) include manifest-table allow-list validation for inbound events; (c) Cargo feature flag scaffold for `kafka-source`.

## Tenant-isolation invariant

Held throughout. Zero `org-os/` writes. Single canonical `acme.ai` placeholder at `org-os/conventions.md:121`.

All code under `repos/resink-ai/resink-core/crates/nanofab-supervisor/{src,tests}/`; spec edit under `docs/superpowers/specs/`. Team artifacts under `teams/application/resink-core/`. None of these are under `org-os/`.
{% endraw %}
