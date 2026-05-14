---
layout: default
title: Retro — 2026-05-13-1944
date: 2026-05-13
status: active
type: retro
loop: 2026-05-13-1944
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1944
  links: parent: board/exec-summaries/2026-05-13-1944.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-13-1944

## What worked

- **Spec-first pattern compounds returns — third validating data point.** Step 1 (loop -1844) executed against the spec with three first-contact revisions. Step 2 (this loop) executed against the step-1-revised spec with **zero** new revisions. The trait surface is now structurally validated under two structurally-different impls (`ParquetReplay`'s eager-from-disk shape; `InMemoryStream`'s mpsc-receive-with-disconnect shape). Step 3's Kafka revisions, if any, will be Kafka-specific (consumer-group state semantics, partition rebalance error mapping) — different in kind from the trait-shape revisions step 1 surfaced. **Reproducibility:** the spec-first-then-execute pattern's per-step revision count decreases monotonically as the trait stabilizes; revisions in later steps are domain-specific, not contract-shape. Three validating data points now exist; the memory entry from -1844 is justified, not provisional.

- **Pull-forward of structural-API work was the right scope-shaping call.** `Supervisor::new(...).with_source(Box::new(source)).run()` was originally a step-4 concern per spec §9; pulling it forward made step 2 sized L instead of M but gave us the spec §8 integration-test shape verbatim. Step 3 (Kafka) now just plugs into a settled `Supervisor` API surface via `.with_source(Box::new(KafkaSource::new(...)))`; it doesn't need to restructure `main.rs`. **Reproducibility:** when a step's surface already touches a structural change that a later step would need anyway, pulling the work forward is positive-sum. The earlier step grows; the later step shrinks more.

- **Unified-source model fell out naturally from the streaming abstraction.** Step 1 carried forward the per-node-spec source structure from the pre-step-1 eager flow (because the trait's first impl was structurally similar). Step 2's `InMemoryStream` doesn't fit that mold — tests push mixed-table events through one channel. The supervisor's natural shape became "one source per supervisor + route by `event.table` to per-table buffers." The per-node-spec structure was a `ParquetReplay` quirk, not a contract. **Reproducibility:** when a streaming abstraction's second impl exposes that the first impl's structure was implementation-specific, refactor the supervisor to the abstraction's natural shape (not the legacy one).

- **`drain_to_vec` retired cleanly.** Step 1 introduced the bridge helper explicitly as a step-1-only concession. Step 2 deleted it — the per-batch inline poll loop in `Supervisor::run` replaced it. The bridge served its purpose: keep step 1's blast radius tight; retire it when the next step's restructure makes it unnecessary. **Reproducibility:** the bridge-helper pattern (defer structural rewrites to the step that motivates them) works as designed when the next step actually deletes the bridge, not when it carries forward. Bridge helpers that survive multiple steps are a smell.

- **Integration test shape is reusable.** `tests/streaming_event_source.rs` drives `Supervisor::run()` via `InMemoryStream::pair()` with inline `RawEvent` construction + no fixture dependency. Step 3 (`Kafka`) can swap in via `.with_source(Box::new(KafkaSource::new(broker_addr)))`; step 4 (end-to-end Kafka verdict) uses the same harness. The test body is parametrized only by the source factory. **Reproducibility:** for trait-abstraction integration tests, parametrize by source factory; assert at the supervisor surface (`RunResult` + output existence + per-table row counts).

- **Lib-tree exposure of `event_source` + `supervisor` was the cleanest split.** The integration test imports from `nanofab_supervisor::supervisor::Supervisor` and `nanofab_supervisor::event_source::{InMemoryStream, RawEvent, ...}`. `nodes` + `trace` stay `pub(crate)` (no external consumers). `main.rs` consumes the lib via `use nanofab_supervisor::supervisor::{Supervisor, SupervisorConfig};`. No double-module-declaration boilerplate; the lib and bin trees both compile clean. **Reproducibility:** when a binary crate needs to be exercised by integration tests, lib-tree exposure of the public types is cheaper than duplicating module declarations across both trees.

- **Seventh same-day loop with zero process churn.** Same observation as the past six retros; loop-ID convention handles it cleanly. **Reproducibility:** the org-os process scales down to ~3-hour-per-loop cadence cleanly; the convention's "two-week-equivalent" framing has been demonstrably loose enough.

## What didn't

- **`Retryable`-as-fatal path is unproven.** Neither `ParquetReplay` nor `InMemoryStream` constructs `Retryable`. The supervisor treats it as fatal in step 2. Step 3 (Kafka) is the first impl that will actually produce transient errors during partition rebalance, broker disconnect, etc. The supervisor's poll loop needs a real backoff-then-retry path with exponential backoff + cap (spec §6 specifies: default 1s, capped at 30s with exp backoff). **Mild:** the variant exists on the trait surface; the failure mode is "step 3 forgets to wire the backoff loop and treats Kafka transient errors as fatal." **Action:** see P1 — explicit KR in step 3's brief.

- **Per-table buffering is a memory bomb for real Kafka streams.** Step 2 buffers all events from a source into `HashMap<String, Vec<RawEvent>>` before dispatching to per-table `nodes::*::run` at `EndOfStream`. For replay (`ParquetReplay` + `InMemoryStream`) this is fine — finite event counts. For continuous Kafka streams, the supervisor needs to dispatch per-batch to single-event `nodes::*::process_event(event)` APIs — and that requires refactoring `nodes::user::run` / `nodes::account::run` to streaming shape. **Mild:** spec §7's aspirational `nodes::route_event(event, shard_id)` shape is what step 4 needs. The debt is correctly scoped to step 4 (end-to-end Kafka-driven verdict). **Action:** step 4's brief should include the node-API refactor as an explicit KR.

- **Single-source model assumes correct `event.table` tagging.** If a future `Kafka` impl's topic-to-table mapping is buggy, the supervisor would silently buffer events into a table that's never dispatched. Step 2's supervisor validates manifest-side tables at startup but doesn't validate inbound `event.table` against that allow-list. **Mild:** trivial fix — validate `event.table` against the manifest's known set; log+skip+count unknowns. **Action:** see P1 sub-ask (b).

- **Test fixtures for streaming integration are inline-constructed, not externally byte-validated.** The integration test asserts on `RunResult.user_rows + RunResult.account_rows + output parquet existence + trace non-emptiness`. It doesn't byte-compare the output parquets against a fixture (because the input is constructed inline; no fixture exists). For the canonical `make mvp-loop` path, output bytes are validated by sim-farm's diff engine — that's the load-bearing gate. The streaming integration test validates structural correctness, not byte-stability. **Trivial:** different validation paths; both load-bearing in their own scope. No action.

- **The `Supervisor::run()` is consuming-self (`mut self`).** The builder pattern returns `Self`, then `.run()` consumes it. Future tests that want to introspect supervisor state after run would need a different shape (e.g., `.run(&mut self) -> Result<RunResult, String>` keeping `self` alive). **Trivial:** no test needs this yet. Awareness only.

## Evolution proposals

### P1: Step 3 of the streaming restoration plan — `Kafka` impl + broker (class: **tenant**, with devops cross-team)

- **Problem it solves:** Step 3 of ADR-2026-05-13-001. The implementing loop's first decision: `rdkafka` vs `rskafka` (per ADR § Alternative G — deferred to implementing loop's owner). Adds the first impl that actually produces `Retryable` errors; the supervisor's poll loop needs the real backoff-then-retry path. Adds broker provisioning (devops). Adds the `kafka-source` Cargo feature flag scaffold.
- **Proposed change:** Multi-team loop. Owner: resink-core (client) + devops (broker). Sized L. Target loop+1 or loop+2 (depending on broker provisioning lead time). KRs include:
  - (a) Cargo feature flag `kafka-source` (default off); `kafka.rs` module behind the flag.
  - (b) `Kafka` impl per spec §5.3 (consumer-group join in `start`; `poll_events` consuming + parsing JSON events + mapping topic→table + `CommitToken::Kafka(...)`; `commit_offsets` calling consumer's commit; `shutdown` synchronous commit + close).
  - (c) Supervisor's poll loop: `Retryable` backoff-then-retry per spec §6 (default 1s, capped at 30s, exp backoff).
  - (d) Manifest-table allow-list validation for inbound `event.table` (log+skip+count unknowns).
  - (e) Broker provisioning: minikube-deployable Redpanda or Kafka chart; devops owns.
  - (f) Integration test reusing the `streaming_event_source.rs` harness shape: spin up an in-process broker (or skip if `kafka-source` feature off); drive `Supervisor::run().with_source(KafkaSource::new(...))`; assert on `RunResult`.
- **Recommendation:** **Bake into next CEO brief.** Don't skip; the Kafka impl is the load-bearing third step.
- **Review path:** No new ADR (executes against existing ADR-2026-05-13-001). The implementing loop's first decision (rdkafka vs rskafka) gets a one-paragraph note in the loop's brief + a Status-section narration when step 3 closes.
- **Owner:** resink-core + devops.
- **Timing:** **Loop+1 or loop+2.**

### P2: Multi-loop-blocker-arc report-type ADR (class: **org-os**, carryover, strongly motivated)

- **Problem it solves:** Prior retro § P2 (2026-05-13-1844; first proposed 2026-05-12-1254). Standardize what a multi-loop ADR's closure narration looks like. Worked examples now: ADR-2026-05-16-001 (closed 5-step dlopen arc), ADR-2026-05-13-001 (4-step streaming arc with steps 1 + 2 closed). Both have substantive Status sections; the pattern is reproducible.
- **Proposed change:** New ADR `board/decisions/<date>-NNN-multi-loop-blocker-arc-report-type.md`. Body proposes `arc-report` artifact type + `Status` section convention for per-step closures. Sized M.
- **Recommendation:** **Draft + ratify next non-resink-core loop.** Bundle with P3 (link-existence smoke).
- **Review path:** ADR-class.
- **Owner:** board.
- **Timing:** **Next non-resink-core loop.**

### P3: Link-existence smoke for `publish-to-gitbook.py` (class: **tenant**, carryover)

- **Problem it solves:** Prior retro § P3 (2026-05-13-1844). Catch the `.md`-vs-`.html` class of bug at publish time.
- **Proposed change:** Sized S. Walk the published tree; extract `[text](path)` links; verify resolutions.
- **Recommendation:** **Bundle with P2 next non-resink-core loop.**
- **Review path:** No ADR (CI-internal).
- **Owner:** board.
- **Timing:** **Next non-resink-core loop.**

### P4: Codify the spec-first pattern in `org-os/conventions.md` (class: **org-os**, motivated)

- **Problem it solves:** Three validating data points for spec-first-then-execute now exist: ADR-2026-05-16-001 (closed 5-step; opened design-first); ADR-2026-05-13-001 step 1 (3 revisions); ADR-2026-05-13-001 step 2 (0 revisions). Pattern is observably reproducible across two arcs + three executing loops. Memory entry `feedback-spec-first-architectural-arc.md` is justified, not provisional. **Codification-when-motivated says three instances IS motivation.**
- **Proposed change:** Add a one-paragraph section to `org-os/conventions.md` § "Body-shape rules": "For multi-loop architectural arcs spanning multiple teams or multiple weeks, the opening loop's body is ADR + design spec authoring (no code). Subsequent loops execute against the spec; first-contact revisions are allowed but must be narrated in the ADR's Status section in the same loop." Cite ADR-2026-05-16-001 + ADR-2026-05-13-001 as worked examples.
- **Recommendation:** **Draft + ratify next non-resink-core loop.** Bundle with P2 + P3.
- **Review path:** ADR-class (per the org-os evolution path: any `org-os/` change requires an ADR in `board/decisions/`).
- **Owner:** board.
- **Timing:** **Next non-resink-core loop.** Bundles with the arc-report-type ADR (P2) since they're both org-os process-evolution items.

### P5: `Supervisor::run()` consuming-self refactor (class: **deferred**)

- **Problem it solves:** "What didn't" point 5. Current shape: `Supervisor::new(config).with_source(...).run() → Result<...>` consumes self. Tests that want to introspect supervisor state across multiple `.run()` invocations (e.g., a hot-restart scenario; partial-failure recovery test) need `&mut self` shape.
- **Proposed change:** None this loop. No test or impl needs the introspection yet.
- **Recommendation:** **Deferred.** Demand-driven.
- **Review path:** Trivial refactor when motivated; no ADR.
- **Owner:** resink-core.
- **Timing:** **Demand-driven.**

## Decisions to record

**New ADR placeholders this loop: 0.** ADR-2026-05-13-001's Status section was updated with the step-2 closure narration; no new ADRs drafted.

**P4 (spec-first codification) is the candidate ADR for next non-resink-core loop**, bundled with P2 (arc-report-type) since they're both org-os process-evolution items.

## Carryover ADRs / open work still on the books

- **ADR-2026-05-13-001 step 3 (P1 this retro):** Loop+1 or loop+2. `Kafka` impl + broker; resink-core + devops; sized L.
- **ADR-2026-05-13-001 step 4:** Loop+3/+4. End-to-end Kafka-driven `verdict=pass`. Includes node-API refactor to single-event shape.
- **Multi-loop-blocker-arc report-type ADR (P2 this retro; carryover):** Strongly motivated. Next non-resink-core loop.
- **Spec-first pattern codification (P4 this retro):** Motivated by three data points. Bundles with P2.
- **Link-existence smoke for `publish-to-gitbook.py` (P3 this retro; carryover):** Bundles with P2 + P4.
- **Showcase refresh cadence (2026-05-13-1303 retro P1):** Demand-driven.
- **Fresh-clone verification of the walkthrough recipe (2026-05-13-1303 retro P2):** Opportunistic.
- **Supervisor-side NodeCtx bridge** (step 3.5 follow-up to ADR-2026-05-16-001; demand-driven).
- **Cargo CI cross-repo access** (private-compatible options documented; user-only decision).
- **Branch protection on resink-core master** (gated on GitHub Pro upgrade decision; user-only).
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing carry from 2026-05-23.
- **`<verified-against-rust-impl: pending>` tag in DE Kafka contract §2.1** — owner = DE.

## Tenant-isolation dry-run

Held throughout. Zero `org-os/` edits. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `board/{okrs,decisions,exec-summaries,retros}/`, `teams/application/resink-core/`, `docs/superpowers/specs/`, and `repos/resink-ai/resink-core/crates/nanofab-supervisor/{src,tests}/`. None of these are under `org-os/`; the invariant has no edits to gate.
{% endraw %}
