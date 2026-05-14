---
layout: default
title: Retro — 2026-05-13-1422
date: 2026-05-13
status: active
type: retro
loop: 2026-05-13-1422
owner: board
grand_parent: Loops
parent: Loop 2026-05-13-1422
nav_order: 3
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1422
  links: parent: board/exec-summaries/2026-05-13-1422.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-13-1422

## What worked

- **Pattern reproducibility is mechanical now.** ADR-2026-05-16-001 (dlopen restoration) closed cleanly 1.5 hours ago across five loops; this loop opened a second instance of the same shape — named-deviation + time-boxed multi-loop restoration plan with `loop+N` targets — against a different deviation (parquet-eager event-source). The reproducibility is mechanical at this point: name the deviation honestly in Context; lay out the multi-loop plan in Decision with `loop+N` targets per ADR-2026-05-30-002; enumerate alternatives in Alternatives; reserve a Status section for future closure narration. Two clean instances is enough to call the pattern canonical. **Reproducibility:** when an architectural deviation needs a multi-loop fix, reach for this shape first — don't invent. The Context-Decision-Alternatives-Consequences-Plan-Status-Links section sequence is now the default.

- **Spec-first authoring let the trait shape settle without code pressure.** The `EventSource` trait went through three revisions during authoring (initial: `next_event()` single-event API; second: batched but without `commit_offsets`; final: batched + opaque `CommitToken` + watermark + `EndOfStream` variant). Each revision was a re-read of the runtime spec sections (§4 consumer-group, §6.2 watermark, §6.4 cross-shard) to verify the trait surface accommodates the documented semantics. Doing this in markdown is ~30 minutes per revision; doing it after the trait shipped in code would be a refactor cycle each time. **Reproducibility:** when a multi-loop arc would otherwise risk implementation drift (different loops interpreting the architecture differently), file the spec first so the trait shape + lifecycle settle before code is written. Code-first commits to a shape before the docs are re-read; that's a refactor cycle waiting to happen.

- **The implementation-loop plan table is the load-bearing handoff.** Spec §9 cross-walks spec sections to ADR plan steps; future OKR KRs reference this table. Implementing loops don't re-litigate the trait shape or the plan structure; they execute against named contract sections. ADR-2026-05-16-001's three-step plan had this implicitly; this ADR + spec make it explicit. **Reproducibility:** for any multi-loop ADR that has an attached spec, add a cross-walk table at the end of the spec mapping spec sections to ADR plan steps. The implementing loop's brief cites the table row by row.

- **`unimplemented!("loop+N")` as a deliberate spec marker.** The `Kafka` impl's poll body is marked `unimplemented!` — not because we don't know what it does, but because the rdkafka-vs-rskafka decision needs the implementing loop's runtime-environment evaluation (which crate ships consumer-group rebalancing this month?). Capturing the integration contract while leaving the implementation choice in the right scope is the discipline. **Reproducibility:** when a reference implementation's body depends on an implementing-loop decision, the spec marks the body unimplemented with the target loop. Future authors look for these markers when starting their step. Codify when a second spec authors with the marker (per codification-when-motivated discipline; see P3).

- **Honest deferrals in the ADR's "out of scope" list.** Persistent KV state, multi-tenant supervisor sharing, sim-farm Mode B, cross-shard re-keying, schema-registry codecs — all named, each gets its own future ADR. The streaming-rearchitecture ADR is intentionally narrow: `EventSource` + Kafka ingestion only. Future arcs can compose without conflict. **Reproducibility:** every multi-loop ADR should have an "out of scope" list naming the adjacent deviations that will need their own ADR. Naming them is positioning, not weakness — it signals the scope discipline that keeps the multi-loop plan tractable.

- **Fifth same-day loop, zero process churn.** Loop IDs -0056, -0859, -1022, -1303, -1422 in 14 hours of wall-clock time. The org-os process scales down to this density cleanly. The convention's "two-week-equivalent" framing has been demonstrably loose enough to absorb 5 loops/day; no convention edit needed. **Reproducibility:** the org-os process is in its "use it, don't extend it" phase — extensions need real motivation (a new pattern that won't fit existing shapes), not aspirational codification.

## What didn't

- **No code = no compile / no test feedback this loop.** Design-first scope is deliberate, but the trait surface won't be validated until the implementing loop attempts step 1 (`ParquetReplay`). The three revisions during authoring caught the obvious gaps (single-event vs batched, missing commit, missing EndOfStream variant), but a fourth gap may emerge once `ParquetReplay` is being typed out. **Mild:** that's exactly what step 1 is for — surface gaps in the trait while the implementation cost is small (one impl in one file). **Action:** none; flag for step 1's retro to evaluate whether the spec's trait shape survives first-impl contact.

- **The `Kafka` impl section's `unimplemented!("loop+3")` is a tradeoff in expressiveness.** A reader of the spec can't see the consumer-poll body — they have to read the rdkafka or rskafka docs to fill it in mentally. This is intentional (scope discipline), but it means the spec is less self-contained than the trait + `ParquetReplay` + `InMemoryStream` sections. **Trivial:** the integration contract is fully specified; the marker is honest about what's deferred. No action.

- **The implementation-loop plan table couples spec evolution to ADR evolution.** If the spec is revised in a future loop (e.g., a new variant on `EventSourceError` motivated by `ParquetReplay` first contact), the ADR's restoration plan may need to follow. The cross-walk table is good for the implementing loop but adds a maintenance burden on revisions. **Mild:** the maintenance burden is "update one table" when the spec changes; not zero, but cheap. No action.

- **No fresh-clone test of the runtime spec ↔ design spec link path.** The design spec cites runtime spec sections (§3, §4, §5.4, §6, §6.2, §6.4) by number; if the runtime spec is renumbered or restructured, these citations rot silently. **Trivial:** the runtime spec has been section-stable for 8+ loops. No action; awareness.

- **Cross-cutting timing artifact: fifth loop on the same calendar date.** Loop IDs preserve uniqueness via `HHMM`, but five loops in 14 hours is a different cadence than the convention's "two-week-equivalent" framing implies. Same observation as the prior retro (§ "What didn't" #4); the cadence held cleanly here too. **Trivial:** loop-ID convention works; no convention edit needed. Surface as awareness only — possibly worth one explicit sentence in `org-os/conventions.md` § "Cadence" framing when a future loop touches that surface, but not motivated enough to author this loop.

## Evolution proposals

### P1: Step 1 of the streaming restoration plan — `EventSource` trait + `ParquetReplay` (class: **tenant**)

- **Problem it solves:** Opens the multi-loop arc filed in ADR-2026-05-13-001. Step 1 extracts the existing parquet-eager logic into the new `EventSource` trait's first concrete impl; the supervisor consumes through the trait; `make mvp-loop` regression-free.
- **Proposed change:** Resink-core loop. Owner: resink-core. Sized M. Target loop+1. Cite spec §2 (trait), §5.1 (`ParquetReplay` impl), §7 (supervisor integration). The spec's implementation-loop plan table § "Step 1" is the contract.
- **Recommendation:** **Bake into next CEO brief.** First-impl contact will validate (or invalidate) the trait shape; either way, that feedback informs steps 2-4. Don't skip.
- **Review path:** No new ADR (executes against existing ADR-2026-05-13-001). Step 1's retro records what shipped + any trait revisions needed.
- **Owner:** resink-core.
- **Timing:** **Loop+1.**

### P2: Multi-loop-blocker-arc report-type ADR (class: **org-os**, carryover, strongly motivated)

- **Problem it solves:** Prior retro § P3 (2026-05-13-1303); first proposed 2026-05-12-1254 retro § P3. The arc-report-type artifact would standardize what a multi-loop ADR's closure looks like (timeline, slip record, what-shipped per step, what-deferred + carryover, what-the-arc-validated-or-broke). **Now strongly motivated:** worked examples exist in two ADRs (ADR-2026-05-16-001's closed-section + ADR-2026-05-13-001's plan-section, which is the open form of the same shape). Two instances = canonical pattern.
- **Proposed change:** New ADR drafted as `board/decisions/<date>-NNN-multi-loop-blocker-arc-report-type.md`. Body proposes a new artifact type (`arc-report`) emitted when a multi-loop ADR closes. Sized M.
- **Recommendation:** **Draft + ratify next non-resink-core loop.** Bundle with P3 (link-existence smoke).
- **Review path:** ADR-class.
- **Owner:** board.
- **Timing:** **Next non-resink-core loop.**

### P3: Link-existence smoke for `publish-to-gitbook.py` (class: **tenant**, carryover)

- **Problem it solves:** Prior retro § P4 (2026-05-13-1303); carryover from 2026-05-13-0859 retro § P2. The publish script generates index pages that link to other pages; a link-existence smoke would catch the `.md`-vs-`.html` class of bug at publish time instead of post-merge.
- **Proposed change:** Sized S. Walk the published tree; extract `[text](path)` links from generated index pages; resolve each path; error if any link points at a non-existent destination.
- **Recommendation:** **Bundle with P2 next non-resink-core loop.**
- **Review path:** No ADR (CI-internal).
- **Owner:** board.
- **Timing:** **Next non-resink-core loop.**

### P4: Spec-first authoring as a memory'd discipline (class: **deferred**)

- **Problem it solves:** "What worked" point 2 — the three trait revisions during spec authoring would have been refactor cycles in code. The discipline is currently a one-instance observation; codification-when-motivated says wait for a second instance.
- **Proposed change:** Save as a memory entry ("for multi-loop architectural arcs, file the spec first so the contract settles before code") OR codify in `org-os/conventions.md` § "Body-shape rules" when a second arc validates the pattern. Not both.
- **Recommendation:** **Deferred.** One worked instance is not yet enough. Revisit if step 1 of the streaming arc surfaces a refactor that the spec saved us from (positive evidence) or doesn't (neutral). Codify only on the second arc that benefits.
- **Review path:** Memory entry or convention edit; no ADR.
- **Owner:** board.
- **Timing:** **Demand-driven.**

### P5: `unimplemented!("loop+N")` as a spec convention (class: **deferred**)

- **Problem it solves:** Decisions-filed (this loop) and "What worked" point 4. One instance of the marker exists (Kafka poll body in the design spec). Codification-when-motivated says wait for a second.
- **Proposed change:** When a second spec uses the same marker for a deferred-to-implementing-loop body, codify in `org-os/conventions.md` § "Body-shape rules" as a one-sentence convention.
- **Recommendation:** **Deferred.** Same as P4.
- **Review path:** Convention edit; no ADR.
- **Owner:** board.
- **Timing:** **Demand-driven.**

## Decisions to record

**New ADR placeholders this loop: 1.** ADR-2026-05-13-001 drafted this loop with `status: active`. The four-step restoration plan is the body; closure narration deferred to the implementing loops (step 1: loop+1; step 4: loop+5/+6).

**Net new ADR drafts this retro: 0.** Codification proposals P4 + P5 are deferred per codification-when-motivated; P2 is the carryover candidate for next loop. The zero-ADR-retro streak now sits at 6 consecutive (counting the 2026-05-13-0056 ratification bundle as the last), but the streak is not load-bearing — this loop *did* draft a new ADR (ADR-2026-05-13-001); it just wasn't an org-os ADR.

## Carryover ADRs / open work still on the books

- **ADR-2026-05-13-001's four-step restoration plan:** Active. Step 1 (P1 this retro) = loop+1; step 4 closes the arc at loop+5/+6. Slip-record opportunity at each step's exec summary.
- **Multi-loop-blocker-arc report-type ADR (P2 this retro; carryover from 2026-05-12-1254 retro P3 + 2026-05-13-1303 retro P3):** **Strongly motivated** — two worked examples (ADR-2026-05-16-001 closed + ADR-2026-05-13-001 just-opened). Next non-resink-core loop.
- **Link-existence smoke for `publish-to-gitbook.py` (P3 this retro; carryover from 2026-05-13-0859 retro P2 + 2026-05-13-1303 retro P4):** Next non-resink-core loop. Bundles with P2.
- **Showcase refresh cadence (2026-05-13-1303 retro P1):** Demand-driven; revisit when the showcase pair feels stale.
- **Fresh-clone verification of the walkthrough recipe (2026-05-13-1303 retro P2):** Opportunistic; ideally before first external share of the showcase pair.
- **Supervisor-side NodeCtx bridge** (step 3.5 follow-up to ADR-2026-05-16-001; demand-driven).
- **Cargo CI cross-repo access** (private-compatible options documented; user-only decision when to act — deploy key / scoped PAT / in-repo fixtures; no public-repo option).
- **Branch protection on resink-core master** (gated on GitHub Pro upgrade decision; user-only).
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing carry from 2026-05-23.
- **`<verified-against-rust-impl: pending>` tag in DE Kafka contract §2.1** — owner = DE; spec §6 of this loop assumes the contract holds.

## Tenant-isolation dry-run

Held trivially this loop. Zero `org-os/` edits. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `board/{okrs,decisions,exec-summaries,retros}/` and `docs/superpowers/specs/`. None of these are under `org-os/`; the invariant has no edits to gate.
{% endraw %}
