---
layout: default
title: "ADR 2026-05-13-001: streaming event source rearchitecture"
date: 2026-05-13
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-13
  status: active
  decision: "Name the current parquet-eager event-source as a deliberate MVP deviation from the runtime spec's streaming architecture; introduce an `EventSource` trait abstraction with Kafka as the first real implementation; commit to a four-step multi-loop restoration plan owned across resink-core + devops + sim-farm"
-->
{% raw %}

# ADR 2026-05-13-001: Streaming-Event-Source Rearchitecture (Trait + Multi-Loop Restoration Plan)

## Context

The nanofab runtime spec ([docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md)) is detailed about the supervisor's streaming architecture: §3 ("Architecture overview") names Kafka/Redpanda as the canonical event transport; §4 ("The Supervisor") names the consumer-group protocol as the shard-rebalancing mechanism; §5.4 ("Event shape") defines the wire format the supervisor consumes; §6 ("Shard-routing & internal Kafka") names internal Kafka topics for cross-shard re-keying + watermark publication.

None of that exists in code today. The supervisor at `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/event_source.rs` reads parquet files eagerly via `read_fact_parquet()`, returning a fully-materialised `Vec<RawEvent>` consumed by the main loop. No Kafka client is in `Cargo.toml`; no consumer-group protocol; no watermark advancement on a real timer; no incremental offset commit. The MVP closed loop (`make mvp-loop` → `verdict=pass mismatches=0`) is green and load-bearing for the synthetic-tenant verification flow — but it is a deliberate v1 simplification of the documented streaming design, not a missing feature.

This deviation is now the **largest unbuilt piece of the documented architecture**. The 2026-05-13-1303 showcase capabilities report explicitly named "real Kafka / event-stream ingestion" as the first of eight honest deferrals; the gap between the runtime spec and the implementation is the single highest-leverage architectural arc remaining. This ADR is the entry point for closing it.

The pattern this ADR follows is the named-deviation + time-boxed-multi-loop-restoration shape established by [ADR-2026-05-16-001](2026-05-16-001-abi-option-a-mvp-deviation.md), which closed cleanly across five loops on 2026-05-13-1022. The same shape applies here: name the deviation honestly; commit to a multi-loop plan with per-step owners and target loops; reserve a Status section for future closure narration.

## Decision

The board ratifies the current parquet-eager event-source as a named MVP deviation from the runtime spec's streaming-ingestion design. For the next-N-loops a **streaming-capable `EventSource` trait abstraction** lands in the supervisor; **Kafka becomes the first real streaming implementation**; the existing parquet-eager path continues to live as a feature-gated `ParquetReplay` impl so the synthetic-tenant verdict pipeline + integration tests remain green throughout the migration.

The architectural shape:

- An `EventSource` trait (defined in `crates/nanofab-supervisor/src/event_source/mod.rs`) abstracts the read side. The trait's responsibilities: `poll_events` (yield one batch of events or none, never block indefinitely; back-pressure handled by the caller); `commit_offsets` (durably advance consumer position after the supervisor processes a batch); `watermark` (publish the current low-watermark for shard-routing's watermark tracker); `shutdown` (orderly close).
- Two reference implementations:
  - **`ParquetReplay`** — preserves the current `--mode=sim` flow with synthetic-tenant fixtures. Yields events in event-ts order; commits a no-op; advances a synthetic watermark by reading the last batch's max-ts. Used by `make mvp-loop` + every integration test.
  - **`Kafka`** — production-mode. Wraps a Kafka consumer (likely `rdkafka` per the canonical Rust ecosystem choice; alternatives evaluated below). Joins a consumer group per the runtime spec §4 contract; commits offsets via the consumer-group protocol; surfaces real Kafka offsets to the watermark tracker.
- The supervisor's `main.rs` `run_supervisor()` function changes from "eagerly read all events; then process" to "loop: poll a batch from the EventSource; route events to shards; commit offsets when shards finish that batch." The watermark tracker, the trace writer, and the per-shard mutation flow all remain unchanged at the per-event level; only the source-of-events pivots.
- The full design specification — trait signatures, lifecycle, watermark + offset semantics, error handling, integration shape — is at [docs/superpowers/specs/2026-05-13-streaming-event-source-design.md](../../docs/superpowers/specs/2026-05-13-streaming-event-source-design.md), shipped alongside this ADR in loop 2026-05-13-1422.

The restoration plan is a **four-step, time-boxed sequence**:

1. **`EventSource` trait + `ParquetReplay` impl** (resink-core; sized M; target `loop+1`). Extract the `EventSource` trait; refactor `read_fact_parquet` into a `ParquetReplay` impl behind the trait; refactor `run_supervisor()` to poll the trait. `make mvp-loop` → `verdict=pass mismatches=0` continues to hold. No new external dependencies. The supervisor's behavior is byte-identical under `--mode=sim`; the restructure is purely internal.

2. **In-memory streaming impl + integration test** (resink-core; sized M; target `loop+2`). Add `InMemoryStream` impl (channel-backed; ingests events from a Rust-test-driven producer). New integration test at `crates/nanofab-supervisor/tests/streaming_event_source.rs` drives the supervisor from `InMemoryStream` through 3 small batches; asserts the supervisor's output parquet matches a known fixture. This step **proves the abstraction works** without requiring Kafka infrastructure.

3. **`Kafka` impl + broker integration** (resink-core for the client; devops for the broker provisioning; sized L; target `loop+3` OR `loop+4` depending on how the broker provisioning goes). Add `rdkafka` (or `rskafka` — final-decision-deferred-to-implementing-loop based on ergonomics + build complexity) as a workspace dep. Implement `Kafka` impl per the spec. Devops provisions a local Kafka broker (Strimzi-on-kind or docker-compose; choice is devops-owned per their charter). Integration test drives the supervisor against the local broker.

4. **End-to-end closed-loop verdict with Kafka ingestion** (resink-core + sim-farm; sized M; target `loop+5` OR `loop+6`). Add a `make mvp-loop-kafka` Makefile target that publishes the synthetic-tenant fact streams to the local Kafka broker, runs the supervisor in `--mode=streaming`, runs the sim-farm diff, asserts `verdict=pass mismatches=0`. This is the **closing milestone** — same shape as ADR-2026-05-16-001 step 3's hot-swap correctness test. The Kafka-driven pipeline produces the same verdict as the parquet-driven one; the rearchitecture is verified at the end-to-end level.

Sister arcs explicitly **out of scope** for this ADR (each gets its own future ADR when motivated):

- **Persistent KV state backend** (Redis / DynamoDB / TiKV per runtime spec). The supervisor's in-process SCD2 state remains a separate v1 simplification; a streaming-supervisor-without-checkpointing is still useful, and the work to add persistence is orthogonal to the EventSource trait.
- **Multi-tenant supervisor sharing.** One supervisor per tenant remains the model; cross-tenant supervisor sharing is a sibling arc.
- **Sim-farm Mode B (streaming differ).** Diff engine remains file-vs-file (Mode A); a streaming differ is its own arc, separately scoped.
- **Cross-shard re-keying via internal Kafka topics** (runtime spec §6.4). Out of scope for the basic ingestion ADR; lands as a follow-up after step 4 closes.
- **Schema-registry-aware codecs.** Spec'd; not in this ADR.

## Alternatives considered

- **A: Keep parquet-eager indefinitely.** Rejected. The runtime spec's whole premise — supervisors are stateless and fungible, Kafka offsets are the truth of where each shard is — assumes a streaming ingress. Without it, the multi-tenant fleet story doesn't ship.

- **B: Skip the trait; just hardcode a Kafka consumer where `read_fact_parquet` lives today.** Rejected. The trait abstraction lets the synthetic-tenant verdict pipeline keep working without a Kafka broker (which would be needed in every developer + CI environment). Decoupling the source-of-events from the supervisor's processing logic is the load-bearing architectural property; treating Kafka as one impl rather than the impl is what makes the migration safe.

- **C: Skip Kafka; use an in-memory queue as the production stream transport.** Rejected. The runtime spec specifies Kafka for durability + replayability + cross-supervisor shard rebalancing via the consumer-group protocol. An in-memory queue trades all three away for simpler v1 — but the v1 simpler thing is already shipped (parquet-eager), and the migration is precisely to add the durable streaming property. In-memory is fine as a **test impl** (step 2 above), not as the production path.

- **D: Use Kinesis instead of Kafka.** Rejected for v1. Kafka is the named choice in the runtime spec; the consumer-group semantics + topic-partitioning model + Strimzi-on-k8s deployment path are all specified against Kafka. Kinesis is a viable alternative for AWS-only deployments — but the home-cluster-first deployment story (per ADR-2026-05-12-0645 first-real-deploy) doesn't ship with AWS access, and migrating to Kinesis later is a smaller change than migrating to a different abstraction now.

- **E: Pull semantics (the supervisor polls the source) vs Push semantics (the source pushes batches into a channel the supervisor consumes).** Decision: **pull**. Pull keeps the supervisor in control of when events flow — important for back-pressure (the supervisor naturally back-pressures by polling slower when its shards are saturated) + for graceful shutdown (the supervisor stops polling rather than receiving in-flight events it can't drain). Push has lower per-event overhead but harder back-pressure and shutdown semantics; not worth the trade for our scale.

- **F: Synchronous trait vs async trait (`async fn poll_events`).** Decision: **synchronous, with explicit batch-yielding.** The Rust async ecosystem is solid but adds runtime complexity (tokio version pinning across the workspace; cancellation semantics; the async-trait macro vs the 1.75+ native async-fn-in-trait). Synchronous trait + the supervisor manages its own thread pool keeps the trait surface small + portable across Kafka clients that ship both sync and async APIs. The Kafka impl can wrap an async consumer behind a synchronous-blocking interface internally if needed; the trait stays sync.

- **G: `rdkafka` vs `rskafka` vs hand-rolled.** Decision deferred to implementing loop (step 3). `rdkafka` is the canonical Rust binding to librdkafka — battle-tested but requires the C library to be available at build + run time (manageable). `rskafka` is pure-Rust — simpler build surface but less feature-complete (e.g., consumer-group rebalancing not fully implemented as of the last evaluation). Hand-rolled is excessive scope. The implementing loop's owner picks based on the consumer-group feature set at decision time.

## Consequences

- **Positive:** The runtime spec's streaming architecture starts shipping in code. The supervisor's shape becomes consistent with the documented multi-tenant fleet story; per-shard offset commits become the truth of where each supervisor is. The `EventSource` trait is also a future-extensibility lever: a websocket source, a Kinesis source, a CDC source — each becomes a new impl without changing the supervisor. The MVP closed loop stays green throughout the migration because `ParquetReplay` is the first impl.

- **Negative / costs:** Five additional loops of architectural work owned across three teams (resink-core for code; devops for broker provisioning; sim-farm for the closing verdict integration). One new external runtime dependency (Kafka broker; modest deployment + ops surface). One new Rust workspace dependency (`rdkafka` or `rskafka`; modest build complexity). The supervisor's behavior under load becomes subject to Kafka broker availability; the `--mode=sim` fallback remains for offline testing.

- **Follow-ups required:** Implementing loops execute steps 1-4 per the plan. Status section is reserved here; it gets filled when each step closes and again when step 4 closes the arc. Sibling-arc ADRs may be filed as motivation surfaces: persistent KV state; multi-tenant supervisor sharing; sim-farm Mode B; cross-shard re-keying.

## Status

### 2026-05-13, loop 2026-05-13-1844 — step 1 closed

**Step 1 of the four-step restoration plan shipped.** The `EventSource` trait + `ParquetReplay` impl land in tree at `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/event_source/`. The supervisor consumes events through the trait via a step-1 bridge (`drain_to_vec`) that preserves the per-node, per-table routing structure unchanged.

**What shipped:**

- New module directory `crates/nanofab-supervisor/src/event_source/` (replaces the single-file `event_source.rs`):
  - `mod.rs` — `EventSource` trait + `EventBatch` + `CommitToken` + `EventSourceError` per spec §2; plus a `drain_to_vec` step-1 bridge helper.
  - `raw.rs` — existing `RawEvent` / `RawOp` / `RawFieldValue` / `FactStreamSpec` + `xxh64_hash` / `partition` / `key_value_for` / `read_fact_parquet` / `events_for` relocated verbatim.
  - `parquet_replay.rs` — `ParquetReplay` impl per spec §5.1.
- `main.rs` wired through the trait: `ParquetReplay::new(specs) → drain_to_vec(&mut source)` replaces the direct `event_source::events_for(&specs)` call. Per-node, per-table routing unchanged.
- `make mvp-loop` green: `verdict=pass mismatches=0` on the canonical two-dim fixture. Supervisor output byte-identical to the pre-step-1 path.
- `cargo test --workspace --release` green; 4 new `ParquetReplay` unit tests passing.

**First-contact spec revisions** (narrated per CEO brief allowance "spec drift allowed when structurally motivated"):

1. **`EventSourceError` uses manual `Display` + `std::error::Error` impls, not `thiserror::Error` derive.** The workspace doesn't have `thiserror` as a dep; spec §2 assumed it. Manual impl is ~12 lines; adding a new workspace dep for one enum variant set was the larger blast radius. The trait shape is unchanged; only the derive macro differs.
2. **`CommitToken` is a bare `pub enum`, not a `pub(crate)`-wrapped newtype.** Spec §2 has `pub struct CommitToken(pub(crate) CommitTokenInner)`. The newtype was motivated by external-consumer encapsulation; in step 1 there are no external consumers (the supervisor binary is the only caller). The bare-enum shape is simpler + future hardening (re-wrapping behind a newtype) is non-breaking if + when an external consumer materializes.
3. **`drain_to_vec` helper added** (not in spec). The supervisor's per-node-spec event loading is per-table-separated (user_events / account_events Vecs are constructed before the per-table `nodes::user::run` / `nodes::account::run` invocations); migrating that structure to a per-batch outer loop is a step 2-3 concern. `drain_to_vec` is the step-1 bridge that keeps the trait wired without restructuring `main.rs`'s per-table separation. Spec §7's "main.rs changes" example shows the post-step-3 shape; step 1 doesn't get there yet.

These revisions are narrated in detail in `board/retros/2026-05-13-1844-ceo-retro.md`. Spec §2 will be updated in step 2 alongside the `InMemoryStream` impl that also needs these types.

**Sized:** M; landed in one resink-core build session.

**Carryover to step 2:** `InMemoryStream` impl + integration test against a synthetic event sender. Target loop+1 from here. Resink-core owns; sized M.

### 2026-05-13, loop 2026-05-13-1944 — step 2 closed

**Step 2 of the four-step restoration plan shipped — sized up to L when CEO pulled the `Supervisor` API extraction forward from step 4.** The `InMemoryStream` impl + `Supervisor` struct API + unified-source model + per-batch outer loop + integration test + spec §2 edit all landed in one resink-core build session.

**What shipped:**

- `crates/nanofab-supervisor/src/event_source/in_memory_stream.rs` — `InMemoryStream` impl per spec §5.2. mpsc-backed; `InMemoryStream::pair()` factory; 5 unit tests (pair roundtrip; empty-batch on no-events; sender-drop → EndOfStream; batch ordering preserved; commit/shutdown no-ops).
- `crates/nanofab-supervisor/src/supervisor.rs` — `Supervisor` struct extracted from `main.rs`'s inline `run()`. Builder shape: `Supervisor::new(SupervisorConfig).with_source(Box<dyn EventSource>).run() → Result<RunResult, String>`. Hosts the manifest types (`Manifest`, `DagSpec`, `NodeSpec`, `ManifestFactStream`) + outer poll loop per spec §7.
- `crates/nanofab-supervisor/src/event_source/parquet_replay.rs` — new `ParquetReplay::from_specs(specs)` constructor; the supervisor builds a single unified source gathering all fact streams from all node-specs in the DAG (rather than one source per node-spec as in step 1).
- `crates/nanofab-supervisor/src/main.rs` — slimmed to a ~80-line shell: parse `Args` → `SupervisorConfig` → `Supervisor::new(config).run()` → emit ok/error.
- `crates/nanofab-supervisor/src/lib.rs` — now exposes `event_source` + `supervisor` modules (in addition to the prior `plugin_loader`); `nodes` + `trace` stay `pub(crate)`. The `streaming_event_source` integration test imports from this library surface.
- `crates/nanofab-supervisor/tests/streaming_event_source.rs` — new integration test (3 cases) drives `Supervisor::run()` via `InMemoryStream::pair()` per spec §8 verbatim. Tests: mixed dim_user + dim_account batch routing → 2/2 rows per table + outputs + trace exist; empty stream → 0/0 rows; empty-batches-then-real-events → 1/0 rows.
- `crates/nanofab-supervisor/src/event_source/mod.rs` — `drain_to_vec` step-1 bridge **deleted** (replaced by the inline poll loop in `Supervisor::run`).
- `docs/superpowers/specs/2026-05-13-streaming-event-source-design.md` § 2 — code blocks updated: `EventSourceError` shows manual `Display`/`Error` impls; `CommitToken` shows bare-enum shape; "Why these methods" gained a `drain_to_vec`-retirement bullet.

**Zero further first-contact spec revisions.** Step 1 surfaced three; step 2 surfaced zero. The trait surface accommodates `InMemoryStream` cleanly (mpsc receive → `Ok(empty_batch)` on `TryRecvError::Empty`; `Err(EndOfStream)` on `TryRecvError::Disconnected`). The empty-batch semantics in spec §3 worked verbatim under the supervisor's new inline poll loop.

**Architectural decisions taken during the build (narrated but not first-contact-revisions):**

1. **Unified-source model.** Step 1 had one `ParquetReplay` per node-spec; step 2 collapses to one source per supervisor instance, yielding events tagged by `RawEvent.table`. The supervisor routes by table to a `HashMap<String, Vec<RawEvent>>` of per-table buffers, then dispatches to `nodes::user::run` / `nodes::account::run` at `EndOfStream`. This shape is naturally what `InMemoryStream` + `Kafka` need (both yield mixed-table events from a single channel/topic-subscription); the per-node-source model was a `ParquetReplay`-specific quirk.
2. **Per-table buffering kept inside the Supervisor.** Spec §7's pseudocode shows `nodes::route_event(event, shard_id)` — a single-event API. The actual `nodes::user::run` / `nodes::account::run` take slices. Refactoring `nodes::*` to single-event APIs is deferred (no forcing function this step); the supervisor's outer loop accumulates into per-table buffers + dispatches at shutdown. This is a deliberate per-table-buffer-before-run pattern that step 3 / step 4 may revisit if Kafka's continuous-flow semantics motivate single-event node APIs.
3. **`Retryable` errors treated as fatal in step 2.** Neither `ParquetReplay` nor `InMemoryStream` constructs `Retryable`. The supervisor's poll loop treats an unexpected `Retryable` as fatal (returns an error). Step 3 (Kafka) adds the real backoff loop.

**Test surface:**

- `cargo test --workspace --release`: green. New: 5 `InMemoryStream` unit tests + 3 `streaming_event_source` integration tests + 1 `parquet_replay` test (`fact_stream_spec_construction_is_infallible`).
- `make mvp-loop`: `verdict=pass mismatches=0`. Output bytes identical to step-1 baseline.

**Sized:** L (revised up from spec §9's M sizing when CEO pulled `Supervisor` extraction forward from step 4); landed in one resink-core build session.

**Carryover to step 3:** `Kafka` impl + broker integration. Resink-core (client) + devops (broker). Sized L. Target `loop+1` / `loop+2` from here. Steps 3 + 4 are now smaller than originally sized because the `Supervisor` API + unified-source model are settled — step 3 adds a `Kafka` impl behind the `kafka-source` Cargo feature; step 4 wires end-to-end Kafka-driven `verdict=pass`.

## Links

- Runtime spec: [docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) — §3, §4, §5.4, §6 specify the streaming architecture this ADR closes the gap to.
- Design spec for this ADR's contract: [docs/superpowers/specs/2026-05-13-streaming-event-source-design.md](../../docs/superpowers/specs/2026-05-13-streaming-event-source-design.md) — the canonical `EventSource` trait + lifecycle + reference impls.
- Sister ADR (just closed): [ADR-2026-05-16-001](2026-05-16-001-abi-option-a-mvp-deviation.md) — the named-deviation + multi-loop-restoration pattern this ADR mirrors. Closed cleanly across 5 loops; worked example of the pattern at full closure.
- Triggering brief: [board/okrs/2026-05-13-1422-ceo-brief.md](../okrs/2026-05-13-1422-ceo-brief.md).
- Showcase deferral surface that motivates this ADR: [board/reports/2026-05-13-resink-ai-capabilities.html](../reports/2026-05-13-resink-ai-capabilities.html) § "Honest deferrals" — names "real Kafka / event-stream ingestion" as the first deferral.
- Relative-dating convention for the multi-loop plan's `loop+N` targets: [ADR-2026-05-30-002](2026-05-30-002-multi-loop-plan-relative-dating.md).
{% endraw %}
