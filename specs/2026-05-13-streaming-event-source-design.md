---
layout: default
title: "Spec: 2026-05-13-streaming-event-source-design"
date: 2026-05-13
status: active
type: spec
owner: board
---

<!-- original-frontmatter:
  type: spec
  owner: board
  date: 2026-05-13
  status: active
-->
{% raw %}

# Streaming Event-Source Design — `EventSource` trait + lifecycle + reference impls

> Canonical contract for the supervisor's read side. Reified by [ADR-2026-05-13-001](../../../board/decisions/2026-05-13-001-streaming-event-source-rearchitecture.md). Implementing loops cite this spec; their OKR KRs map directly to spec sections.

---

## 1. Goals & non-goals

### Goals

- Decouple the supervisor's **source-of-events** from its **processing logic**. The supervisor today calls `read_fact_parquet()` directly; this spec replaces that with a trait the supervisor polls.
- Make Kafka a **drop-in implementation** of that trait. The runtime spec's streaming architecture (`docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md` §3, §4, §5.4, §6) lands in code without re-architecting the supervisor's per-shard logic, per-key partitioning, trace writer, or output-parquet emitter.
- Preserve the synthetic-tenant verdict pipeline. `make mvp-loop` must continue to exit 0 with `verdict=pass mismatches=0` throughout the migration. The existing parquet-eager flow becomes a `ParquetReplay` impl behind the trait; nothing else changes.
- Surface real Kafka offsets + watermarks to the supervisor's existing watermark tracker without re-shaping it.
- Stay async-runtime-free in the trait surface. Implementations may wrap async clients internally; the trait is synchronous + batch-yielding.

### Non-goals (explicitly out of scope for this spec)

- Persistent KV state backend. SCD2 state remains in-process; checkpointing across supervisor restarts is a separate arc.
- Multi-tenant supervisor sharing. One supervisor per tenant remains the model.
- Sim-farm Mode B (streaming differ). Diff engine remains file-vs-file; this spec ends at the supervisor's output-parquet emission.
- Cross-shard re-keying via internal Kafka topics (runtime spec §6.4). Out of scope; follows after basic ingestion lands.
- Schema-registry-aware codecs. Out of scope; the wire format remains the supervisor's existing `RawEvent` JSON shape.

---

## 2. The `EventSource` trait

The full trait, in Rust:

```rust
//! `EventSource` is the supervisor's read-side abstraction. Implementations
//! yield batches of `RawEvent` (already documented at
//! `crates/nanofab-supervisor/src/event_source/raw.rs`); the supervisor polls,
//! routes, and commits.

use crate::event_source::raw::RawEvent;

/// Yielded from `poll_events`. Carries the events plus the per-batch metadata
/// the supervisor needs to advance offsets + watermarks.
pub struct EventBatch {
    /// Events in this batch, in (event_ts, event_id) order within the batch.
    /// May be empty (signals "no events ready right now"; not end-of-stream).
    pub events: Vec<RawEvent>,
    /// Implementation-opaque token. The supervisor passes this back via
    /// `commit_offsets` once the batch's events have been processed by every
    /// affected shard. The token's interpretation is implementation-private:
    /// `Kafka` uses it for consumer-group offset commits; `ParquetReplay` uses
    /// it as a no-op marker; `InMemoryStream` uses it for test introspection.
    pub commit_token: CommitToken,
    /// The low-watermark for this batch. The supervisor publishes this to its
    /// existing watermark tracker. `None` means "the source can't compute one
    /// for this batch" — the supervisor's tracker treats `None` as a hold,
    /// not a regression.
    pub low_watermark: Option<u64>,
}

/// Implementation-opaque commit cursor. Bare `pub enum` (no newtype wrap) per
/// step-1 first-contact revision (loop 2026-05-13-1844): step 2 has no
/// external consumers, so the encapsulation a `pub struct CommitToken(pub(crate) CommitTokenInner)`
/// would provide isn't motivated yet. Future hardening (re-wrapping behind a
/// newtype if external consumers materialize) is non-breaking. The supervisor
/// never inspects the inner value; it only hands the token back to
/// `commit_offsets`.
#[derive(Debug, Clone)]
pub enum CommitToken {
    /// `ParquetReplay` cursor (index past the last event in the batch).
    /// Replay sources can't rewind, so commit is a no-op regardless.
    ParquetRowGroup(usize),
    // Future: `Kafka(...)` for consumer-group offsets; `InMemoryStream` reuses
    // `ParquetRowGroup(0)` as a benign placeholder (tests don't introspect).
}

/// Trait every event-source implements.
pub trait EventSource: Send {
    /// Open the source. Called once at supervisor start. Implementations
    /// resolve initial offsets (Kafka consumer-group join; parquet file open;
    /// in-memory channel creation) here. Returns immediately on success; on
    /// failure returns a hard error that aborts the supervisor.
    fn start(&mut self) -> Result<(), EventSourceError>;

    /// Pull the next batch. Implementations:
    ///   - Never block indefinitely. Return `Ok(EventBatch { events: vec![], ... })`
    ///     to signal "no events right now"; the supervisor decides whether
    ///     to back off, retry, or shut down.
    ///   - Return events in (event_ts, event_id) order *within a batch*.
    ///     Across batches monotonic ordering is best-effort; the supervisor's
    ///     watermark tracker handles out-of-order arrivals.
    ///   - Batch size is implementation-defined (Kafka: one consumer poll;
    ///     parquet: one row-group; in-memory: drain up to `cap` events).
    ///   - Return `Err(EventSourceError::EndOfStream)` to signal terminal
    ///     completion (replay sources only; Kafka never returns this).
    fn poll_events(&mut self) -> Result<EventBatch, EventSourceError>;

    /// Durably advance the consumer position. Called by the supervisor after
    /// every shard has processed every event in the batch. Implementations:
    ///   - Kafka: commit the consumer-group offset.
    ///   - ParquetReplay: no-op (replay sources can't "rewind"; the next
    ///     `poll_events` returns the next batch unconditionally).
    ///   - InMemoryStream: mark the batch as committed for test assertions.
    /// Errors are retryable; the supervisor logs + retries with backoff.
    fn commit_offsets(&mut self, token: CommitToken) -> Result<(), EventSourceError>;

    /// Orderly shutdown. Called once at supervisor stop. Implementations
    /// flush any in-flight commits, close consumers, drop channel senders.
    /// Errors are surfaced but don't block the shutdown sequence.
    fn shutdown(&mut self) -> Result<(), EventSourceError>;
}

/// Error type. Carrier for both retryable (broker disconnected; partition
/// rebalance in progress) + fatal (config invalid; topic doesn't exist) cases.
///
/// Manual `Display` + `std::error::Error` impls per step-1 first-contact
/// revision (loop 2026-05-13-1844): the workspace doesn't have `thiserror` as
/// a dep, and the existing error idiom is `Result<T, String>` everywhere.
/// Adding a workspace dep for one enum's derive macro was the larger blast
/// radius vs ~12 lines of manual impls. Re-evaluate the dep when 3+ types
/// want the macro.
#[derive(Debug, Clone)]
pub enum EventSourceError {
    /// Replay sources only.
    EndOfStream,
    /// Transient; supervisor logs + retries with backoff (step 3+).
    Retryable(String),
    /// Unrecoverable; supervisor logs + aborts with non-zero exit.
    Fatal(String),
}

impl std::fmt::Display for EventSourceError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            EventSourceError::EndOfStream => write!(f, "end of stream (replay sources only)"),
            EventSourceError::Retryable(msg) => write!(f, "retryable: {}", msg),
            EventSourceError::Fatal(msg) => write!(f, "fatal: {}", msg),
        }
    }
}

impl std::error::Error for EventSourceError {}
```

### Why these methods + this shape

- **`start` / `shutdown` are explicit** so initialization + teardown are visible in the supervisor's lifecycle (rather than implicit in `Drop`, which makes error handling awkward).
- **`poll_events` returns batches, not single events.** Kafka's consumer-poll API is naturally batched; parquet row-groups are naturally batched; in-memory implementations can buffer + drain. Batching also gives the supervisor's per-batch trace + per-batch commit a natural fence.
- **`commit_offsets` is decoupled from `poll_events`.** The supervisor only commits after every affected shard has finished the batch; the trait doesn't assume processing happens synchronously on the polling thread.
- **`low_watermark` is optional per batch.** Sources that can't compute a real watermark (small in-memory tests; replay sources that finished) return `None`; sources that can (Kafka with real partition metadata; parquet with sorted event_ts) return the actual minimum.

- **Step-1's `drain_to_vec` bridge retired in step 2.** Loop 2026-05-13-1844 introduced a `drain_to_vec(&mut dyn EventSource) -> Vec<RawEvent>` helper to keep the supervisor's pre-step-1 per-table-Vec shape working while the trait threaded through. Step 2 (loop 2026-05-13-1944) restructures the supervisor's outer loop to per-batch flow per §7 below; the bridge is deleted. See `crates/nanofab-supervisor/src/supervisor.rs::Supervisor::run`.

---

## 3. Lifecycle

```
┌─────────────────┐
│ supervisor::main│
└────────┬────────┘
         │ build EventSource impl (Kafka | ParquetReplay | InMemoryStream)
         v
   ┌──────────┐
   │  start() │  resolve offsets, join consumer group, open files
   └────┬─────┘
        │
        ├─── loop ──────────────────────────────────────────┐
        v                                                    │
  ┌─────────────────┐                                        │
  │  poll_events()  │ returns EventBatch { events, token, wm }
  └────────┬────────┘                                        │
           │                                                 │
           v                                                 │
  ┌─────────────────┐                                        │
  │  if !empty:     │                                        │
  │    route events │ → existing partition() + shard logic   │
  │  publish wm     │ → existing watermark tracker           │
  └────────┬────────┘                                        │
           │                                                 │
           v                                                 │
  ┌─────────────────┐                                        │
  │ commit_offsets  │ after every shard finishes the batch   │
  └────────┬────────┘                                        │
           │                                                 │
           └─── continue ────────────────────────────────────┘
                                                             │
        ┌──── shutdown signal ────────────────────────────────┘
        v
  ┌────────────┐
  │ shutdown() │ flush, close consumers, drop senders
  └────────────┘
```

### Empty-batch semantics

`poll_events` returning `Ok(EventBatch { events: vec![], commit_token, low_watermark })` means "I have no events ready, but I'm still alive." The supervisor:

1. Does **not** call `commit_offsets` for an empty batch (token has nothing to advance).
2. **Does** publish `low_watermark` if `Some` — empty batches with a watermark are how Kafka publishes watermark advancement when no events flow for a partition.
3. May back off (configurable; default: sleep 100ms before next poll) to avoid hot-spinning.

### End-of-stream semantics

`ParquetReplay` returns `Err(EventSourceError::EndOfStream)` after the last batch. The supervisor:

1. Calls `commit_offsets` on the last successful batch's token if not already.
2. Drains all in-flight per-shard work.
3. Calls `shutdown()`.
4. Emits the final output parquet + verdict.

`Kafka` impl **never** returns `EndOfStream`. The supervisor's only exit path under Kafka is an external shutdown signal (SIGTERM; Helm pod stop; explicit `--max-events` flag for one-shot testing).

---

## 4. Watermark + offset semantics

The runtime spec at §6.2 specifies watermarks as "the minimum of all in-flight partition's max-event-ts, minus an allowed-lateness window." This spec maps that to the trait surface as follows:

- **Kafka impl** computes the low-watermark per batch from the consumer's per-partition position metadata. The watermark for a batch is `min(committed_offset.event_ts) - allowed_lateness` across the consumer's owned partitions.
- **ParquetReplay impl** computes the low-watermark as `batch.events.iter().map(|e| e.event_ts).min()` — pure replay sources have perfect knowledge of their own watermark.
- **InMemoryStream impl** lets the test set the watermark explicitly via a sender method `push_with_watermark(events, wm)`. Tests that don't care about watermarks return `None`.

The supervisor's existing watermark tracker (whose internals are spec'd in the runtime spec §6.2) consumes whatever value the trait surfaces. **The trait does not specify the tracker's internal semantics**; it only specifies the contract by which the source publishes watermark values upstream.

### Offsets

Offsets are **opaque to the supervisor**. The `CommitToken` carries whatever the impl needs (Kafka: a `TopicPartition → offset` map; ParquetReplay: a row-group index it doesn't care about; InMemoryStream: a batch counter for test assertions). The supervisor calls `commit_offsets(token)` once per batch after the batch is fully processed; the impl decides what "commit" means.

This decoupling is the key property: a supervisor-side bug in offset tracking is impossible, because the supervisor never tracks offsets — it only hands the opaque token back.

---

## 5. Reference implementations

### 5.1 `ParquetReplay`

```rust
pub struct ParquetReplay {
    specs: Vec<FactStreamSpec>,
    events: Vec<RawEvent>,        // materialised at start()
    cursor: usize,                // next batch starts here
    batch_size: usize,            // default 1024
    end_of_stream: bool,
}

impl EventSource for ParquetReplay {
    fn start(&mut self) -> Result<(), EventSourceError> {
        let all_events = read_fact_parquets(&self.specs)?;
        // Existing globally-sorted-by-(event_ts, event_id) invariant.
        self.events = all_events;
        Ok(())
    }
    fn poll_events(&mut self) -> Result<EventBatch, EventSourceError> {
        if self.cursor >= self.events.len() {
            return Err(EventSourceError::EndOfStream);
        }
        let end = (self.cursor + self.batch_size).min(self.events.len());
        let batch_events: Vec<RawEvent> = self.events[self.cursor..end].to_vec();
        let low_wm = batch_events.iter().map(|e| e.event_ts).min();
        let token = CommitToken(CommitTokenInner::ParquetRowGroup(self.cursor));
        self.cursor = end;
        Ok(EventBatch { events: batch_events, commit_token: token, low_watermark: low_wm })
    }
    fn commit_offsets(&mut self, _token: CommitToken) -> Result<(), EventSourceError> {
        Ok(())  // replay sources can't rewind; no-op.
    }
    fn shutdown(&mut self) -> Result<(), EventSourceError> {
        Ok(())
    }
}
```

**Behavior:** Byte-equivalent to today's `read_fact_parquet` → eager-process flow for `make mvp-loop`. The synthetic-tenant verdict pipeline produces identical `verdict.json` output. This is the load-bearing back-compat property.

### 5.2 `InMemoryStream`

```rust
pub struct InMemoryStream {
    receiver: std::sync::mpsc::Receiver<EventBatch>,
    /// Producer side — tests retain the sender to push batches.
    pub sender: std::sync::mpsc::Sender<EventBatch>,
}

impl EventSource for InMemoryStream {
    fn start(&mut self) -> Result<(), EventSourceError> { Ok(()) }
    fn poll_events(&mut self) -> Result<EventBatch, EventSourceError> {
        use std::sync::mpsc::TryRecvError;
        match self.receiver.try_recv() {
            Ok(batch) => Ok(batch),
            Err(TryRecvError::Empty) => Ok(EventBatch {
                events: vec![], commit_token: CommitToken::empty(), low_watermark: None,
            }),
            Err(TryRecvError::Disconnected) => Err(EventSourceError::EndOfStream),
        }
    }
    fn commit_offsets(&mut self, _token: CommitToken) -> Result<(), EventSourceError> {
        Ok(())  // test impl; tests assert commit invariants via the sender.
    }
    fn shutdown(&mut self) -> Result<(), EventSourceError> { Ok(()) }
}
```

**Use case:** Integration tests drive the supervisor through synthetic batches without needing a Kafka broker. The test sender pushes 3 small batches; the supervisor processes them; the test asserts on the output parquet. This impl proves the trait abstraction works at step 2 of the restoration plan.

### 5.3 `Kafka`

```rust
pub struct Kafka {
    consumer: rdkafka::consumer::StreamConsumer,  // or rskafka equivalent
    topics: Vec<String>,                          // topics-to-tables mapping
    table_for_topic: HashMap<String, String>,
    poll_timeout: std::time::Duration,
}

impl EventSource for Kafka {
    fn start(&mut self) -> Result<(), EventSourceError> {
        self.consumer.subscribe(&self.topics.iter().map(String::as_str).collect::<Vec<_>>())
            .map_err(|e| EventSourceError::Fatal(format!("subscribe: {}", e)))?;
        Ok(())
    }
    fn poll_events(&mut self) -> Result<EventBatch, EventSourceError> {
        // Pseudocode — full impl owned by the implementing loop.
        // (a) poll the consumer with self.poll_timeout
        // (b) for each Message, parse the value as RawEvent JSON,
        //     map topic → table, push to batch
        // (c) record the per-partition max offset into a CommitToken
        // (d) compute low-watermark from consumer.position() per-partition
        unimplemented!("loop+3")
    }
    fn commit_offsets(&mut self, token: CommitToken) -> Result<(), EventSourceError> {
        match token.0 {
            CommitTokenInner::Kafka(map) => {
                self.consumer.commit(&map.into_topic_partition_list(), CommitMode::Async)
                    .map_err(|e| EventSourceError::Retryable(format!("commit: {}", e)))
            }
            _ => Err(EventSourceError::Fatal("token type mismatch".into())),
        }
    }
    fn shutdown(&mut self) -> Result<(), EventSourceError> {
        // Final synchronous commit + consumer close.
        unimplemented!("loop+3")
    }
}
```

**Use case:** Production. Joins a consumer group per the runtime spec §4 contract; tracks offsets via the consumer-group protocol; surfaces real Kafka offsets to the supervisor's watermark tracker. The implementing loop (step 3) decides between `rdkafka` and `rskafka` based on consumer-group feature completeness at decision time.

---

## 6. Error handling

The `EventSourceError` enum has three variants:

- **`EndOfStream`** — only `ParquetReplay` returns this. The supervisor reacts by draining + shutting down.
- **`Retryable(String)`** — broker disconnected; partition rebalance in progress; transient commit failure. The supervisor logs + sleeps (default 1s, capped at 30s with exponential backoff) + retries the operation.
- **`Fatal(String)`** — topic doesn't exist; config invalid; codec mismatch. The supervisor logs + aborts with a non-zero exit code. No retry.

The supervisor's main loop has explicit match arms for each variant; the trait impl chooses correctly per-error.

---

## 7. Integration with the supervisor

### File layout

The existing `crates/nanofab-supervisor/src/event_source.rs` becomes `crates/nanofab-supervisor/src/event_source/` (a module directory):

```
crates/nanofab-supervisor/src/event_source/
├── mod.rs                  # `EventSource` trait + `EventBatch` + `CommitToken` + `EventSourceError`
├── raw.rs                  # The existing `RawEvent`, `RawFieldValue`, `RawOp`, `FactStreamSpec` types
├── parquet_replay.rs       # `ParquetReplay` impl (step 1 of the restoration plan)
├── in_memory_stream.rs     # `InMemoryStream` impl (step 2)
└── kafka.rs                # `Kafka` impl (step 3); cfg-gated behind a `kafka-source` Cargo feature
```

### `main.rs` changes

The supervisor's `run_supervisor()` function changes from:

```rust
// Current (parquet-eager):
let events = event_source::events_for(&dag.fact_streams)?;
for event in events {
    let shard_id = partition(&event, dag.shard_count);
    nodes::route_event(event, shard_id, &mut trace_writer)?;
}
```

To:

```rust
// New (trait-driven; supports any EventSource impl):
let mut source: Box<dyn EventSource> = build_source_from_config(&dag, &cli_args)?;
source.start()?;
loop {
    match source.poll_events() {
        Ok(batch) if batch.events.is_empty() && batch.low_watermark.is_none() => {
            // Empty poll, no watermark: back off briefly.
            std::thread::sleep(BACKOFF);
            continue;
        }
        Ok(batch) => {
            if let Some(wm) = batch.low_watermark {
                watermark_tracker.publish(wm);
            }
            for event in &batch.events {
                let shard_id = partition(event, dag.shard_count);
                nodes::route_event(event.clone(), shard_id, &mut trace_writer)?;
            }
            // After every shard finishes the batch — see step 2's
            // explicit barrier; under static-plugins this is synchronous.
            source.commit_offsets(batch.commit_token)?;
        }
        Err(EventSourceError::EndOfStream) => break,
        Err(EventSourceError::Retryable(msg)) => {
            log::warn!("event_source retryable: {}; retrying", msg);
            std::thread::sleep(retry_backoff());
        }
        Err(EventSourceError::Fatal(msg)) => {
            return Err(format!("event_source fatal: {}", msg).into());
        }
    }
}
source.shutdown()?;
```

### `build_source_from_config`

A small dispatcher selects the impl based on CLI args + the DAG manifest:

- `--mode=sim` → `ParquetReplay` (from `dag.fact_streams`).
- `--mode=streaming --source=kafka` → `Kafka` (from `dag.kafka_config`).
- Test-only constructor (`InMemoryStream::new()`) is used directly by integration tests.

The CLI arg `--mode` gains a `streaming` variant (alongside existing `sim`); the DAG manifest gains an optional `kafka_config` block.

### Migration path

Step 1 of the restoration plan refactors `read_fact_parquet` into `ParquetReplay` without touching `main.rs`'s outer interface — `make mvp-loop` keeps using `--mode=sim`, and `ParquetReplay` is the only impl wired in step 1. Steps 2-4 expand the wiring incrementally.

---

## 8. Test-harness shape

Integration tests at `crates/nanofab-supervisor/tests/streaming_event_source.rs`:

```rust
#[test]
fn supervisor_processes_in_memory_stream_to_expected_output() {
    let (sender, source) = InMemoryStream::pair();
    let supervisor = Supervisor::new(/* DAG, output paths, ... */)
        .with_source(Box::new(source));

    // Push 3 batches.
    sender.send(synthetic_batch(events_e1_e2, watermark=199)).unwrap();
    sender.send(synthetic_batch(events_e3, watermark=299)).unwrap();
    drop(sender);  // signals EndOfStream when receiver drains.

    let result = supervisor.run().unwrap();
    assert_eq!(result.verdict.overall_pass, true);
    // Existing parquet-output assertions against a known fixture.
}
```

This shape proves at step 2 of the restoration plan that:

- The trait abstraction works.
- The supervisor's per-shard logic, trace writer, and output emitter are source-agnostic.
- The `EndOfStream` shutdown sequence works.

The same test harness is reusable at step 4 — drive against a real Kafka broker by swapping the source factory; the test body is identical.

---

## 9. Implementation-loop plan

Each row maps to the ADR's restoration plan. Future briefs cite this table; future OKRs' KR1.x map to the named spec sections.

| Step | Owner | Sized | Target | Closes which spec sections |
|---|---|---|---|---|
| 1. `EventSource` trait + `ParquetReplay` | resink-core | M | `loop+1` | §2, §5.1, §7 |
| 2. `InMemoryStream` + integration test | resink-core | M | `loop+2` | §5.2, §8 |
| 3. `Kafka` impl + broker integration | resink-core (client) + devops (broker) | L | `loop+3` / `loop+4` | §4 (Kafka mapping), §5.3, §6 (retryable surface) |
| 4. End-to-end Kafka-driven `verdict=pass` | resink-core + sim-farm | M | `loop+5` / `loop+6` | full §7 production wiring + Makefile target |

Each step:

- Closes its spec sections by code (not by re-spec). The spec is the canonical contract; implementations execute against it.
- Surfaces drift to the spec if discovered (e.g., if a step finds that the trait surface needs a method we didn't anticipate). Drift is captured as an in-loop spec edit + a one-line `Status` note here.
- Reports back to the ADR's Status section at close (per the ADR-2026-05-16-001 closure convention).

---

## 10. Cross-references

- **Ratifying ADR:** [board/decisions/2026-05-13-001-streaming-event-source-rearchitecture.md](../../../board/decisions/2026-05-13-001-streaming-event-source-rearchitecture.md) — names the deviation + commits to this plan.
- **Runtime spec:** [docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md](2026-05-10-nanofab-runtime-design.md) — the canonical streaming-ingestion design this spec reifies at the code-shape level. Especially §3 ("Architecture overview"), §4 ("The Supervisor"), §5.4 ("Event shape"), §6 ("Shard-routing & internal Kafka").
- **Sister ADR (worked example of the pattern):** [board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md](../../../board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md) — the named-deviation + multi-loop-restoration template. Closed cleanly across 5 loops.
- **Existing code this rearchitecture refactors:** `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/event_source.rs` (the `RawEvent` + `read_fact_parquet` surface that step 1 wraps).
- **Triggering brief:** [board/okrs/2026-05-13-1422-ceo-brief.md](../../../board/okrs/2026-05-13-1422-ceo-brief.md).
- **Showcase deferral surface that motivates this work:** [board/reports/2026-05-13-resink-ai-capabilities.html](../../../board/reports/2026-05-13-resink-ai-capabilities.html) § "Honest deferrals."
- **Relative-dating convention** for the `loop+N` targets: [board/decisions/2026-05-30-002-multi-loop-plan-relative-dating.md](../../../board/decisions/2026-05-30-002-multi-loop-plan-relative-dating.md).
{% endraw %}
