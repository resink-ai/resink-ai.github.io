---
layout: default
title: application-resink-core Exec Summary — 2026-05-13-1844
date: 2026-05-13
status: active
type: exec-summary
loop: 2026-05-13-1844
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/resink-core
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1844
  links: parent: board/exec-summaries/2026-05-13-1844.md
-->
{% raw %}

# Resink Core Exec Summary — 2026-05-13 (loop 2026-05-13-1844)

**Headline.** Step 1 of ADR-2026-05-13-001's four-step streaming restoration plan shipped clean in one resink-core build session. `EventSource` trait + `ParquetReplay` impl + supervisor wired through trait + `make mvp-loop` green (`verdict=pass mismatches=0`) + `cargo test --workspace --release` green. Three structurally-motivated first-contact revisions to the design spec (recorded in the ADR's new step-1 closure status section): manual `Display`/`Error` impls instead of `thiserror` derive; bare `pub enum CommitToken` instead of newtype-wrapped variant; `drain_to_vec` step-1 bridge helper preserves the existing per-node, per-table routing structure. Tenant-isolation invariant CLEAN throughout.

## Per-objective rollup

### O1: Streaming event-source — step 1 — ✅ PASS

- **KR1.1 (module layout): PASS.** New directory `crates/nanofab-supervisor/src/event_source/` with three files: `mod.rs` (trait + types + `drain_to_vec` bridge), `raw.rs` (relocated existing types verbatim), `parquet_replay.rs` (the impl). Single-file `event_source.rs` deleted.
- **KR1.2 (trait): PASS** (with first-contact spec revisions). `EventSource` trait method set per spec §2: `start` / `poll_events` / `commit_offsets` / `shutdown`. `EventBatch { events, commit_token, low_watermark }`. `CommitToken::ParquetRowGroup(usize)` (bare-enum revision, narrated in ADR). `EventSourceError::{EndOfStream, Retryable, Fatal}` with manual `Display` + `Error` impls (revision: avoided `thiserror` dep, narrated in ADR).
- **KR1.3 (ParquetReplay impl): PASS.** Per spec §5.1. `start()` calls existing `events_for(&self.specs)` (preserves the globally-sorted-by-`(event_ts, event_id)` invariant). `poll_events()` yields `DEFAULT_BATCH_SIZE = 1024`-event batches with low-watermark = `batch.events.iter().map(|e| e.event_ts).min()`. `EndOfStream` returned once cursor passes `events.len()`. `commit_offsets` no-op. `shutdown` clears state.
- **KR1.4 (supervisor wire): PASS** (with first-contact spec revision). `main.rs`'s direct `event_source::events_for(&specs)?` call replaced with `ParquetReplay::new(specs.clone())` + `drain_to_vec(&mut source)`. Per-node, per-table routing structure unchanged. `drain_to_vec` is the step-1 bridge helper (not in spec; narrated in ADR) that handles the start → poll-loop → handle EndOfStream → shutdown lifecycle and returns a `Vec<RawEvent>` matching the pre-step-1 signature.
- **KR1.5 (mvp-loop green): PASS.** `verdict=pass mismatches=0` on the canonical two-dim (`dim_user` + `dim_account`) fixture. 21 events loaded from 2 streams (dim_user); 18 events loaded from 1 stream (dim_account). Output byte-identical to pre-step-1 path.
- **KR1.6 (cargo test green): PASS.** Full workspace test run: 9 supervisor unit tests (3 pre-existing partition tests + 4 new `ParquetReplay` tests + 2 pre-existing plugin_loader tests) + 6 nanofab-coordinator tests + 2 each for nanofab-node-abi / nanofab-plugin-dim-user / nanofab-plugin-dim-user-v2 — all passing. `dlopen_integration` + `hot_swap_correctness` test files compile + run (0/0 — no test cases under default feature flag); the dlopen-plugins-feature run is unchanged from prior baselines.
- **KR1.7 (tenant-isolation): PASS.** Zero `org-os/` edits. All code lands under `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/event_source/` + a single one-line change in `main.rs`. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder.

## Build phase notes

**Three first-contact spec revisions, all structurally motivated:**

1. **`thiserror` dep avoided.** Spec §2 used `#[derive(thiserror::Error)]` on `EventSourceError`. The workspace doesn't have `thiserror`; the existing error idiom is `Result<T, String>` everywhere. Adding a new workspace dep for one enum's Display impl was the larger blast radius vs ~12 lines of manual `Display` + `Error` impls. Trait shape unchanged.
2. **`CommitToken` bare enum.** Spec §2 wrapped variants behind `pub struct CommitToken(pub(crate) CommitTokenInner)`. The newtype was motivated by external-consumer encapsulation; step 1 has no external consumers. Bare `pub enum CommitToken { ParquetRowGroup(usize) }` is simpler; re-wrapping behind a newtype later is non-breaking.
3. **`drain_to_vec` step-1 bridge.** Spec §7 shows the supervisor running a generic per-batch outer loop with `for event in &batch.events { route_event(event) }`. The actual `main.rs` has per-node, per-table event-Vec construction (user_events vs account_events) followed by per-table `nodes::user::run` / `nodes::account::run` invocations. Restructuring to the spec §7 shape is a step 2-3 concern (only motivated when `InMemoryStream` + `Kafka` need the per-batch + per-partition flow). `drain_to_vec` is the step-1 helper: drives the trait lifecycle, collects events into a `Vec<RawEvent>`, returns the result matching the pre-step-1 `events_for` signature. Spec drift narrated; ADR's Status section records the revisions verbatim.

**Build artifacts:**

- New: `crates/nanofab-supervisor/src/event_source/mod.rs` (~140 lines).
- New: `crates/nanofab-supervisor/src/event_source/raw.rs` (~270 lines; verbatim relocate of old `event_source.rs` with a docstring header noting the relocation).
- New: `crates/nanofab-supervisor/src/event_source/parquet_replay.rs` (~140 lines).
- Modified: `crates/nanofab-supervisor/src/main.rs` (one-block replacement of `events_for(&specs)` → `ParquetReplay::new(specs.clone())` + `drain_to_vec`).
- Deleted: `crates/nanofab-supervisor/src/event_source.rs` (single-file; superseded by the directory).

**Test surface:**

- 4 new `ParquetReplay` unit tests: `start_with_no_specs_yields_end_of_stream`, `shutdown_clears_state`, `commit_token_is_a_noop_for_replay`, `fact_stream_spec_construction_is_infallible`.
- 3 pre-existing `partition_*` tests still passing.
- `make mvp-loop` (the canonical green-bar gate) passes: `verdict=pass mismatches=0` on the widened 2-dim / 3-fact / 4-shard fixture.

## What didn't / risks for step 2

- **The trait surface hasn't been validated under `InMemoryStream` yet.** Step 1's only impl is `ParquetReplay`, which is structurally similar to the pre-step-1 eager flow. The first real test of the trait's generality is `InMemoryStream` (step 2) — until then, the trait may still need revisions. The three step-1 revisions are the floor, not the ceiling.
- **The `drain_to_vec` bridge is a step-1-only helper.** It defeats the per-batch supervisor flow that the spec §7 design intends. Step 2's `InMemoryStream` integration test will need the supervisor's outer loop restructured to consume batches directly — that's step 2's load-bearing work, not just adding a second impl.
- **`thiserror` deferral may need revisiting.** If `EventSourceError` grows more variants in step 2-3 (`Retryable` will actually be constructed by Kafka; possibly more), the manual `Display` impl gets noisy. Re-evaluate when adding the third variant body.

## Asks for the CEO / next loop's brief

- **(from resink-core, next loop):** Step 2 of the restoration plan — `InMemoryStream` impl + supervisor integration test that drives the trait through a synthetic event sender. Spec §5.2 + §8. Sized M. The supervisor's outer loop (`main.rs`) restructures from per-table-Vec to per-batch flow as part of this step. Target loop+1; resink-core owns.
- **(from resink-core, awareness):** The step-1 first-contact revisions imply spec §2 needs a small edit in step 2's loop: drop `thiserror` from the error type; bare-enum the `CommitToken`. Bundling the spec edit with step 2's `InMemoryStream` work keeps the spec drift narrated in the same commit.

## Tenant-isolation invariant

Held throughout. Zero `org-os/` writes. Single canonical `acme.ai` placeholder at `org-os/conventions.md:121`.

All code lives under `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/event_source/` and a one-block update in `main.rs`. Team artifacts live under `teams/application/resink-core/`. None of these are under `org-os/`.
{% endraw %}
