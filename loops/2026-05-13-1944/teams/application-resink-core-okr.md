---
layout: default
title: application-resink-core OKR — 2026-05-13-1944
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-1944
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1944
  links: parent: board/okrs/2026-05-13-1944-ceo-brief.md
-->
{% raw %}

# Resink Core OKR — 2026-05-13 (loop 2026-05-13-1944)

## Context

Single-objective L build loop. Step 2 of ADR-2026-05-13-001's four-step streaming restoration plan; the CEO brief pulls forward the `Supervisor` API extraction (originally a step-4 concern) to enable spec §8's verbatim integration-test shape. Six interlocked deliverables (10 KRs); the `Supervisor` API extraction is the load-bearing piece.

## Objectives

### O1: Streaming event-source — step 2 (`InMemoryStream` + `Supervisor` API + integration test)

source: ceo-brief

Why it matters: First step that validates the trait's generality beyond `ParquetReplay`'s structural similarity to the pre-step-1 flow. Pulling the `Supervisor` extraction forward gets the spec §8 test verbatim into tree and pre-pays restructure work for step 3 (Kafka) + step 4 (end-to-end verdict).

Maps to brief O1 → KR1.1 (InMemoryStream impl), KR1.2 (Supervisor API), KR1.3 (unified-source `from_node_specs`), KR1.4 (outer-loop restructure), KR1.5 (integration test), KR1.6 (spec §2 edit), KR1.7 (mvp-loop green), KR1.8 (cargo test green), KR1.9 (ADR step-2 closure), KR1.10 (tenant-isolation).

**Key results**

- KR1.1: `InMemoryStream` impl per spec §5.2. mpsc-based sender/receiver pair via `InMemoryStream::pair()`. Empty-batch on `TryRecvError::Empty`; `EndOfStream` on `TryRecvError::Disconnected`. Re-exported from `event_source/mod.rs`.
- KR1.2: `Supervisor` struct API in new `crates/nanofab-supervisor/src/supervisor.rs`. Shape: `Supervisor::new(config) → .with_source(Box<dyn EventSource>) → .run() → Result<RunResult, String>`. `SupervisorConfig` + `RunResult` co-located. `main.rs` reduced to ~20-line shell.
- KR1.3: `ParquetReplay::from_node_specs(node_specs, fixtures_dir)` constructor that gathers all fact streams into a single unified source. Existing `ParquetReplay::new(specs)` preserved for direct callers.
- KR1.4: Outer-loop restructure per spec §7. `Supervisor::run()` body matches the §7 shape (start → poll_loop → handle EndOfStream/Retryable/Fatal → shutdown → per-table nodes::run → outputs). `drain_to_vec` deleted from `event_source/mod.rs`.
- KR1.5: Integration test at `crates/nanofab-supervisor/tests/streaming_event_source.rs`. Builds a tempdir manifest + inline `RawEvent` sequence; pushes batches through `InMemoryStream::pair()`; asserts on output parquet row counts + per-dim PK existence + SCD2 invariants.
- KR1.6: Spec §2 code blocks updated to match step-1's in-tree shape: manual `Display`/`Error` impls; bare-enum `CommitToken`. "Why these methods" note about `drain_to_vec` retirement.
- KR1.7: `make mvp-loop`: `verdict=pass mismatches=0`. Output bytes identical to step-1 baseline.
- KR1.8: `cargo test --workspace --release` green. New tests added; no regressions.
- KR1.9: ADR-2026-05-13-001 gains `## Status (2026-05-13, loop 2026-05-13-1944)` step-2 closure section.
- KR1.10: Tenant-isolation invariant holds; zero `org-os/` edits.

**Tasks**

- [ ] Author `crates/nanofab-supervisor/src/event_source/in_memory_stream.rs` (mpsc pair pattern + impl)
- [ ] Re-export `InMemoryStream` from `event_source/mod.rs`; delete `drain_to_vec`
- [ ] Author `crates/nanofab-supervisor/src/supervisor.rs` (Supervisor + SupervisorConfig + RunResult + run())
- [ ] Refactor `crates/nanofab-supervisor/src/main.rs` to thin shell consuming Supervisor
- [ ] Add `ParquetReplay::from_node_specs(node_specs, fixtures_dir)` constructor
- [ ] Implement Supervisor outer-loop body per spec §7
- [ ] Add unit tests for `InMemoryStream` (pair roundtrip; empty-batch; sender-drop → EndOfStream)
- [ ] Author `crates/nanofab-supervisor/tests/streaming_event_source.rs` integration test
- [ ] Edit `docs/superpowers/specs/2026-05-13-streaming-event-source-design.md` § 2 code blocks
- [ ] `cargo test --workspace --release` green
- [ ] `make mvp-loop` `verdict=pass mismatches=0`
- [ ] Append step-2 Status section to ADR-2026-05-13-001

## What success looks like

Step 2 of the streaming restoration plan landed. The `Supervisor` struct API + unified-source model + `InMemoryStream` impl + integration test are all in tree. The spec body matches the in-tree shape. Steps 3-4 build against the now-settled supervisor API.

## Out of scope

- Step 3 (`Kafka` impl + broker) — multi-loop work involving devops.
- Step 4 (end-to-end Kafka-driven verdict).
- Watermark tracker wiring (step 3-4 concern).
- Back-pressure/retry-with-backoff on Retryable errors.
- Refactoring node `run()` to single-event APIs.
- Bundling P2/P3 carryovers.
{% endraw %}
