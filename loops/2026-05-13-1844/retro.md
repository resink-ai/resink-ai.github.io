---
layout: default
title: Retro — 2026-05-13-1844
date: 2026-05-13
status: active
type: retro
loop: 2026-05-13-1844
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1844
  links: parent: board/exec-summaries/2026-05-13-1844.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-13-1844

## What worked

- **Spec-first-then-execute pattern executes mechanically.** ADR-2026-05-13-001 + the design spec were filed at loop `-1422` (design-first; ~3.5 hours of authoring including three trait revisions). Loop `-1844` (this loop) executed step 1 against them in one resink-core build session: read spec §2 + §5.1 + §7, type the three new files, wire `main.rs`, run tests, narrate the three first-contact revisions in the ADR Status section. **Total step-1 cost: one short build session.** Compared to the alternative (open the arc with code; let each loop re-litigate the trait shape) this is dramatically cheaper. **Reproducibility:** when a multi-loop architectural arc is opened design-first, subsequent steps execute as "read the contract section + implement + test + narrate revisions." The pre-paid planning is the load-bearing investment.

- **First-contact revisions narrated honestly + structurally motivated.** Three spec revisions surfaced during step 1: (1) `EventSourceError` uses manual `Display`/`Error` impls because the workspace doesn't have `thiserror` and adding a dep for one enum was the larger blast radius; (2) `CommitToken` is a bare `pub enum` instead of a newtype-wrapped struct because step 1 has no external consumers; (3) `drain_to_vec` step-1 bridge helper added (not in spec) because the supervisor's per-node, per-table routing isn't ready for spec §7's generic per-batch outer loop. All three are structural, not aesthetic. The ADR's Status section narrates each verbatim. **Reproducibility:** the CEO brief's explicit allowance ("spec drift allowed when structurally motivated; not allowed for 'wouldn't it be nicer if...' reasoning") drew the line correctly. Future arc-step retros should expect 1-3 first-contact revisions per step; honest narration is the discipline, not zero-revisions.

- **`drain_to_vec` bridge keeps step-1's blast radius tight.** Spec §7 designs the supervisor's outer loop as generic per-batch consumption. The actual `main.rs` separates events by table before routing. Restructuring `main.rs` to the spec §7 shape would have been ~50 lines of orchestration logic touching per-table routing, watermark wiring, and per-shard commit barriers — none of which is yet motivated by the single `ParquetReplay` impl. The `drain_to_vec` helper drives the trait lifecycle and returns a `Vec<RawEvent>` matching the pre-step-1 signature, so the per-table separation stays unchanged. **Reproducibility:** when migrating an eager flow to a streaming abstraction in multiple steps, the early steps want bridge helpers that defer the structural rewrite to the step that's actually motivated to do it. The rewrite happens *naturally* at step 2 (when `InMemoryStream`'s tests need per-batch flow); pre-emptive rewriting at step 1 would have been wasted work.

- **The `make mvp-loop` gate caught zero regressions.** `verdict=pass mismatches=0`, output bytes identical. The trait wrap is byte-equivalent to the pre-step-1 flow because `ParquetReplay::start()` calls existing `events_for(&self.specs)` (which preserves the globally-sorted-by-`(event_ts, event_id)` invariant); the rest is just trait-method ceremony. **Reproducibility:** when wrapping an eager function behind a trait whose first impl just calls the eager function, regression risk is near-zero. The trait's *generality* is unproven until a second impl exists (step 2's `InMemoryStream`); step 1's main risk is just wiring correctness.

- **No `thiserror` adoption.** Workspace stays on the existing `Result<T, String>` + manual-Error-impl idiom. Adding `thiserror` for one error type would have been the worst-of-both-worlds: a new dep used by exactly one error type. 12 lines of manual `Display` + `Error` impls is cheaper. **Reproducibility:** don't add a workspace dep for one-off use. Re-evaluate when 3+ types want the macro.

- **Sixth same-day loop, zero process churn.** Loop IDs -0056, -0859, -1022, -1303, -1422, -1844 in 18 hours of wall-clock time. Same observation as prior retros; the convention's "two-week-equivalent" framing has been demonstrably loose enough to absorb six loops/day. **Reproducibility:** the org-os process is in its "use it, don't extend it" phase; cadence elasticity is a property of the process, not a bug.

## What didn't

- **Trait generality unvalidated until step 2.** Step 1 ships one impl (`ParquetReplay`) which is structurally similar to the pre-step-1 eager flow. The trait's generality — does it really accommodate `InMemoryStream` + `Kafka`? — is not yet validated. The three step-1 revisions are the floor, not the ceiling; step 2 may surface more. **Mild:** structural test is what step 2 is for. **Action:** none this loop; expect 1-3 more revisions when `InMemoryStream` lands (and possibly more when `Kafka` lands).

- **`drain_to_vec` is a step-1-only helper.** It defeats the per-batch supervisor flow that spec §7 intends. Step 2's `InMemoryStream` integration test will *need* the supervisor's outer loop restructured to consume batches directly — that's step 2's load-bearing work, not just "add a second impl." If step 2 forgets this, the bridge gets carried longer than intended; step 3 (`Kafka`) would then need the restructure *plus* the broker work in one loop. **Mild:** the ADR Status section + this retro narrate the bridge's deliberate-step-1-only nature. Step 2's brief should explicitly include the outer-loop restructure as a KR. **Action:** see P1 below.

- **Spec §2 still has the pre-revision shape.** The ADR Status section narrates the revisions, but spec §2's code blocks still show `#[derive(thiserror::Error)]` and `pub struct CommitToken(pub(crate) CommitTokenInner)`. A reader of the spec who hasn't read the ADR's Status section will be misled. **Mild:** spec is documentation; the code is the truth. Step 2's commit will update spec §2 to match the in-tree shape. **Action:** bundle the spec edit with step 2's `InMemoryStream` work; see P1.

- **The `Retryable` variant is never constructed in step 1.** `ParquetReplay` only returns `EndOfStream` + `Fatal`. The `Retryable` variant exists for the future Kafka impl; it's `#[allow(dead_code)]`-silenced. This is the right shape (keep the variant on the trait surface so step 3 doesn't break it), but feels speculative until step 3. **Trivial:** the alternative (add `Retryable` later) would be a breaking change to `EventSourceError`'s shape. Keep it; awareness only.

- **Cross-cutting timing artifact: sixth loop on the same calendar date.** Same observation as the past four retros; the loop-ID convention handles it cleanly via the `HHMM` component. No process surface change motivated. **Trivial:** awareness only.

## Evolution proposals

### P1: Step 2 of the streaming restoration plan — `InMemoryStream` + supervisor outer-loop restructure (class: **tenant**)

- **Problem it solves:** Step 2 of ADR-2026-05-13-001. Spec §5.2 + §8. The supervisor's outer loop restructures from per-table-Vec (the current shape this loop preserved) to per-batch flow per spec §7 (the shape `InMemoryStream`'s integration test will need). Bundle the spec §2 edit (drop `thiserror`; bare-enum `CommitToken`) with this work. The `drain_to_vec` bridge helper either deletes or stays as an explicit pre-step-3 helper depending on how step 2's restructure goes.
- **Proposed change:** Resink-core build loop. Owner: resink-core. Sized M. Target loop+1 from here. KR shape: (a) `InMemoryStream` impl + sender/receiver pair pattern; (b) supervisor `main.rs` restructured to per-batch outer loop with per-partition routing per spec §7; (c) new integration test at `crates/nanofab-supervisor/tests/streaming_event_source.rs` driving the supervisor via a synthetic `InMemoryStream`; (d) spec §2 updated to match step-1's in-tree shape; (e) `make mvp-loop` regression-free.
- **Recommendation:** **Bake into next CEO brief.** Don't skip step 2; the trait generality is unvalidated until `InMemoryStream` lands.
- **Review path:** No new ADR (executes against existing ADR-2026-05-13-001). Step 2's retro records what shipped + any further spec revisions.
- **Owner:** resink-core.
- **Timing:** **Loop+1.**

### P2: Multi-loop-blocker-arc report-type ADR (class: **org-os**, carryover, strongly motivated)

- **Problem it solves:** Prior retro § P2 (2026-05-13-1422); first proposed 2026-05-12-1254 retro § P3. Standardize what a multi-loop ADR's closure narration looks like (timeline, slip record, what-shipped per step, what-deferred + carryover, what-the-arc-validated-or-broke). **Now has three worked examples:** ADR-2026-05-16-001 (closed 5 steps with full closure narrative), ADR-2026-05-13-001 (opened 4 steps), ADR-2026-05-13-001's step-1 closure (the Status section narration this loop authored). Two open + one closed = the pattern is observable at both arc-start and per-step-close granularity.
- **Proposed change:** New ADR drafted as `board/decisions/<date>-NNN-multi-loop-blocker-arc-report-type.md`. Body proposes a new artifact type (`arc-report`) emitted when a multi-loop ADR closes + a `Status` section convention for per-step closures. Sized M.
- **Recommendation:** **Draft + ratify next non-resink-core loop.** Bundle with P3 (link-existence smoke).
- **Review path:** ADR-class.
- **Owner:** board.
- **Timing:** **Next non-resink-core loop.**

### P3: Link-existence smoke for `publish-to-gitbook.py` (class: **tenant**, carryover)

- **Problem it solves:** Prior retro § P3 (2026-05-13-1422); carryover from 2026-05-13-0859 retro § P2. The publish script generates index pages that link to other pages; a link-existence smoke would catch the `.md`-vs-`.html` class of bug at publish time instead of post-merge.
- **Proposed change:** Sized S. Walk the published tree; extract `[text](path)` links from generated index pages; resolve each path; error if any link points at a non-existent destination.
- **Recommendation:** **Bundle with P2 next non-resink-core loop.**
- **Review path:** No ADR (CI-internal).
- **Owner:** board.
- **Timing:** **Next non-resink-core loop.**

### P4: Spec-first-then-execute pattern — second worked instance, codification candidate (class: **deferred → motivated**)

- **Problem it solves:** Prior retro § P4 (2026-05-13-1422) deferred this pending a second worked instance. **The second instance just landed:** ADR-2026-05-13-001 was opened design-first at `-1422` and step 1 closed mechanically at `-1844`. The pattern is observably reproducible: spec-first authoring ~3.5 hours (with three trait revisions caught in markdown); step-1 execution one resink-core session (with three first-contact revisions narrated in the ADR). Without the spec-first phase, both rounds of revisions would have been refactor cycles in code.
- **Proposed change:** Two paths, pick one:
  - **Memory-entry path:** Save as `feedback-spec-first-architectural-arc.md` — "for multi-loop architectural arcs that span multiple teams or weeks, file an ADR + design spec design-first (no code) in the opening loop; subsequent loops execute against the spec with retro-narrated revisions allowed."
  - **Convention-entry path:** Add a one-paragraph section to `org-os/conventions.md` § "Body-shape rules" — "for multi-loop architectural arcs, the opening loop's body is ADR + spec authoring; subsequent steps execute against the spec." Requires an ADR per the org-os evolution path.
- **Recommendation:** **Memory-entry first; convention-entry deferred.** The convention-entry requires an org-os ADR (per evolution path); that's a separate authoring step that doesn't yet have a forcing function. The memory entry captures the pattern for future loops without process-surface change. Re-evaluate the convention entry if a third arc surfaces (which would make codification mandatory rather than discretionary).
- **Review path:** Memory entry; no ADR. Convention entry deferred.
- **Owner:** board.
- **Timing:** **This loop's commit.** Memory file `feedback-spec-first-architectural-arc.md` saved as part of the loop-close artifacts.

### P5: `drain_to_vec`-style bridge helpers as a recognized step-N concession (class: **deferred**)

- **Problem it solves:** "What worked" point 3. `drain_to_vec` is a step-1 bridge that keeps the trait wired without forcing the supervisor's outer-loop restructure. The pattern (bridge helpers that defer structural rewrites to the step that's motivated to do them) is observable but only one instance.
- **Proposed change:** None this loop. Wait for a second instance.
- **Recommendation:** **Deferred.** Codification-when-motivated; one instance is not enough.
- **Review path:** None unless a second instance surfaces.
- **Owner:** board.
- **Timing:** **Demand-driven.**

## Decisions to record

**New ADR placeholders this loop: 0.** ADR-2026-05-13-001's Status section was updated with the step-1 closure narration; no new ADRs drafted.

**Net new ADR drafts this retro: 0.** P2 (arc-report-type) is the candidate for next non-resink-core loop; P3 (link-existence smoke) is CI-internal; P4 (spec-first memory entry) is a memory entry, not an ADR; P5 is deferred.

## Carryover ADRs / open work still on the books

- **ADR-2026-05-13-001 step 2 (P1 this retro):** Loop+1. `InMemoryStream` + supervisor outer-loop restructure + spec §2 edit.
- **ADR-2026-05-13-001 step 3:** Loop+2/+3. `Kafka` impl + broker; resink-core + devops.
- **ADR-2026-05-13-001 step 4:** Loop+4/+5. End-to-end Kafka-driven `verdict=pass`; resink-core + sim-farm.
- **Multi-loop-blocker-arc report-type ADR (P2 this retro; carryover from 2026-05-12-1254 + 2026-05-13-1422):** Strongly motivated — three worked examples. Next non-resink-core loop.
- **Link-existence smoke for `publish-to-gitbook.py` (P3 this retro; carryover from 2026-05-13-0859 + 2026-05-13-1422):** Bundles with P2.
- **Showcase refresh cadence (2026-05-13-1303 retro P1):** Demand-driven.
- **Fresh-clone verification of the walkthrough recipe (2026-05-13-1303 retro P2):** Opportunistic.
- **Supervisor-side NodeCtx bridge** (step 3.5 follow-up to ADR-2026-05-16-001; demand-driven).
- **Cargo CI cross-repo access** (private-compatible options documented; user-only decision).
- **Branch protection on resink-core master** (gated on GitHub Pro upgrade decision; user-only).
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing carry from 2026-05-23.
- **`<verified-against-rust-impl: pending>` tag in DE Kafka contract §2.1** — owner = DE.

## Tenant-isolation dry-run

Held throughout. Zero `org-os/` edits. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `board/{okrs,decisions,exec-summaries,retros}/`, `teams/application/resink-core/`, and `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/event_source/`. None of these are under `org-os/`; the invariant has no edits to gate.
{% endraw %}
