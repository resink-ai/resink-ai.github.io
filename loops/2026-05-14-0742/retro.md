---
layout: default
title: Retro — 2026-05-14-0742
date: 2026-05-14
status: active
type: retro
loop: 2026-05-14-0742
owner: board
grand_parent: Loops
parent: Loop 2026-05-14-0742
nav_order: 3
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-14
  status: active
  loop: 2026-05-14-0742
  links: parent: board/exec-summaries/2026-05-14-0742.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-14-0742

## What worked

- **The arc-opener pattern is mechanical at three instances — and visibly faster each time.** ADR-2026-05-16-001 (dlopen, closed), ADR-2026-05-13-001 (streaming, steps 1-2 shipped), ADR-2026-05-14-001 (general-tables, opening here) all use the identical shape: Context with code-level evidence → Decision with a phased `loop+N` plan → enumerated Alternatives → Consequences → reserved Status → Links. This loop's authoring was the fastest of the three because nothing about the *structure* needed thought — only the content. **Reproducibility:** the named-deviation + time-boxed-multi-loop-restoration pattern is no longer a practice to validate; it is the org's default reflex for architectural deviations. Three instances is past "canonical."

- **The spec dug to root cause, not symptom.** The symptom: "the supervisor hardcodes two dim tables." The root cause, found by reading `nodes.rs`'s own module comment: each codegen crate inlines its own *distinct* Rust `Event`/`Node` types, so a single generic abstraction is structurally impossible without consuming nodes through the type-erased C-ABI. The spec's §2.2 makes pure-C-ABI consumption the load-bearing move. **Reproducibility:** an arc-opener spec that names only the symptom hands the implementing loop a surface patch; one that names the root cause hands it the real fix. Read the code's own comments — the deviation is often already documented in-place.

- **Cross-team friction was named before it became a blocker.** Three schema shapes exist (manifest node-spec, codegen `schema_json`, DE `{key_columns, payload_columns}`). Rather than let Phase 2 discover the conflict mid-loop, ADR Alternative E + spec §3 decide now: codegen `schema_json` is authoritative, the DE convention is a derived projection, and §3.2 pre-flags the projection-lossiness risk as a known first-contact revision site. **Reproducibility:** when an arc touches a surface multiple teams own, the opener ADR should *make the ownership/authority call* — not defer it to the implementing loop, where it becomes a mid-flight cross-team negotiation.

- **A deferred carryover found its forcing function by becoming load-bearing for a new arc.** ADR-2026-05-16-001's "step 3.5 — supervisor-side NodeCtx bridge" had been demand-driven carryover with no scheduling pressure since the dlopen arc closed. The general-tables generic runner *needs* that bridge — so Phase 1 absorbs it. **Reproducibility:** demand-driven carryovers don't always need a forcing function invented for them. Sometimes a later arc supplies one. Don't aggressively schedule demand-driven items; let the next arc that needs them pull them in.

- **The composition contract between two in-flight arcs was written down.** general-tables and streaming (ADR-2026-05-13-001) are orthogonal but both touch the supervisor and both name resink-core. Spec §7.3 explicitly states the composition: a `RawEvent` is a `RawEvent` regardless of source; general-tables touches *what happens after `poll_events`*, streaming touches *where the event came from*; the one shared change (Phase 2's `RawFieldValue` extension) is additive. **Reproducibility:** when two arcs are in flight against the same component, the *second* arc's spec should write the composition contract — it's cheap insurance against a mid-flight collision.

- **Eighth consecutive loop, zero process churn.** Loop IDs span 2026-05-13-0056 through 2026-05-14-0742. The org-os process continues to scale cleanly. No convention edit motivated by loop mechanics.

## What didn't

- **P4 (codify spec-first in `org-os/conventions.md`) keeps being deferred — and the reason is now circular.** P4 has been queued "for the next non-resink-core loop" since the 2026-05-13-1944 retro. But every loop since has been an arc-opener (a board-authoring loop that is itself a non-resink-core loop) — and the standing rule is "don't bundle another org-os ADR into an arc-opener; it triples the authoring surface." So P4 is deferred *because* the loops that could carry it are exactly the loops the rule says can't. **Chicken-and-egg.** The motivation for P4 is now overwhelming (three clean arc-opener instances), and the deferral reason is procedural, not substantive. **Action:** see P1 — the CEO should schedule a dedicated org-os-process loop that is explicitly *not* an arc-opener.

- **The org now carries two open architectural arcs and the interleave is undecided.** general-tables (Phases 1-3) and streaming (ADR-2026-05-13-001 steps 3-4) both name resink-core as primary code owner. This loop deferred streaming behind general-tables, but "deferred" is not "sequenced" — there is no decision on whether to finish general-tables Phases 1-3 *then* resume streaming, or alternate. **Mild:** both arcs are well-specified, so neither blocks on the other's design; the cost is only scheduling ambiguity. **Action:** see P2.

- **No code feedback again — third consecutive design-first loop for board.** The general-tables runner's trait shape, the `CompositeKey` byte-encoding, the NodeCtx bridge wire shape — none are validated until Phase 1's implementing loop attempts them. The streaming arc's step 1 surfaced three first-contact revisions; general-tables Phase 1 will likely surface its own. **Mild:** that is exactly what Phase 1 is for, and §9 pre-flags the three likeliest revision sites. **Action:** none; flagged for Phase 1's retro.

- **The streaming arc's momentum was interrupted mid-arc.** ADR-2026-05-13-001 shipped steps 1-2 in consecutive loops with clean momentum; this loop's theme switch parks it at step 2. Restarting a paused arc has a re-warm cost (re-reading the spec, the prior Status sections). **Trivial:** the spec + ADR Status sections are exactly the artifacts that make re-warm cheap — this is the spec-first pattern's payoff. No action; awareness.

- **Cross-cutting timing artifact: the first loop of a new calendar date, after seven on the prior date.** No process consequence — the loop-ID convention handles it. Awareness only.

## Evolution proposals

### P1: Dedicated org-os-process loop — clear the P2/P3/P4 backlog (class: **org-os**)

- **Problem it solves:** "What didn't" #1. Three org-os-process items — multi-loop-blocker-arc report-type ADR (P2, carried since 2026-05-12-1254), link-existence smoke for `publish-to-gitbook.py` (P3, carried since 2026-05-13-0859), spec-first codification in `org-os/conventions.md` (P4, carried since 2026-05-13-1944) — have been deferred through every recent loop because every recent loop was an arc-opener and the no-bundling rule applies. The backlog is now strongly motivated and procedurally stuck.
- **Proposed change:** The CEO schedules one loop whose theme *is* the org-os-process backlog — not an arc-opener, not a code loop. It drafts + ratifies the arc-report-type ADR (P2), ships the link-existence smoke (P3), and adds the spec-first section to `org-os/conventions.md` via an ADR (P4). Board owns; sized M; all three are well-specified from their prior retro proposals.
- **Recommendation:** **Bake into the loop after Phase 1.** Phase 1 (general-tables) is `loop+1` and is load-bearing; the org-os-process loop is the natural `loop+2` slot — a deliberate breather between code phases.
- **Review path:** ADR-class for P2 + P4; P3 is CI-internal.
- **Owner:** board.
- **Timing:** **`loop+2` (the loop after general-tables Phase 1).**

### P2: Sequence the two open architectural arcs (class: **tenant**, CEO-decision)

- **Problem it solves:** "What didn't" #2. general-tables (Phases 1-3) and streaming (ADR-2026-05-13-001 steps 3-4) are both open, both resink-core-owned, and the interleave is undecided.
- **Proposed change:** The CEO records an explicit sequencing decision in the next brief. Recommended sequence: general-tables Phase 1 (`loop+1`) → org-os-process loop (`loop+2`, P1) → re-evaluate. Rationale: general-tables Phase 1 subsumes the NodeCtx bridge and is load-bearing for the runtime's generality story; streaming steps 3-4 (Kafka) have a broker-provisioning dependency on devops that benefits from lead time anyway.
- **Recommendation:** **Decide in the next brief.** Not an ADR — a sequencing call recorded in the brief's "CEO decisions" section. Revisit after Phase 1 closes.
- **Review path:** Brief-level decision; no ADR.
- **Owner:** board (CEO).
- **Timing:** **Next brief.**

### P3: General-tables Phase 1 — generic `NodeRunner` + `DimSchema` + NodeCtx bridge (class: **tenant**)

- **Problem it solves:** Phase 1 of ADR-2026-05-14-001. Retire `nodes.rs`'s hand-written `mod user` / `mod account` and `supervisor.rs`'s `match node_spec.table`; ship the generic schema-driven node runner consuming nodes through the C-ABI; a third synthetic dim runs through `make mvp-loop`; the existing two-dim fixture stays byte-stable.
- **Proposed change:** Resink-core loop. Owner: resink-core. Sized M. Target `loop+1`. Cite spec §2, §3.1-3.4, §5, §7.1, §7.2, §7.4, §8 (Phase 1 test). §9 pre-flags the NodeCtx-bridge wire shape as the likeliest first-contact revision (may pull an AE codegen-template change into Phase 1).
- **Recommendation:** **Bake into the next CEO brief.** First and load-bearing phase; everything else in the arc assumes the generic runner exists.
- **Review path:** No new ADR (executes against ADR-2026-05-14-001). Phase 1's retro records what shipped + any first-contact revisions.
- **Owner:** resink-core.
- **Timing:** **`loop+1`.**

### P4: Spec-first pattern — third validating instance recorded (class: **deferred → folded into P1**)

- **Problem it solves:** The 2026-05-13-1944 retro's P4 proposed codifying the spec-first-for-architectural-arcs pattern in `org-os/conventions.md`. This loop is the third clean instance. The proposal is unchanged and now maximally motivated.
- **Proposed change:** Folded into P1's org-os-process loop — P1 explicitly includes the spec-first codification ADR. This entry exists only to record that the third data point landed and to cross-link [[feedback-spec-first-architectural-arc]].
- **Recommendation:** **Folded into P1.** No separate scheduling.
- **Review path:** ADR-class (handled within P1).
- **Owner:** board.
- **Timing:** **`loop+2` via P1.**

## Decisions to record

**New ADR placeholders this loop: 1.** ADR-2026-05-14-001 drafted with `status: active` — the general-tables arc opener.

**Net new ADR drafts this retro: 0.** P1's org-os-process loop will draft the arc-report-type ADR (long-carried P2) + the spec-first codification ADR (P4); both are queued, not drafted here. Drafting them in a retro whose loop is itself an arc-opener would be the exact authoring-surface-tripling the no-bundling rule forbids.

## Carryover ADRs / open work still on the books

- **ADR-2026-05-14-001 Phase 1 (P3 this retro):** `loop+1`. Generic `NodeRunner` + `DimSchema` + NodeCtx bridge; resink-core; sized M.
- **ADR-2026-05-14-001 Phases 2-3:** `loop+2/+3` and `loop+4/+5`. Richer schemas + field types; complex fact shapes.
- **ADR-2026-05-13-001 streaming steps 3-4:** deferred behind general-tables; sequencing decision is P2 this retro. Not abandoned — carryover.
- **Org-os-process backlog (P1 this retro):** multi-loop-blocker-arc report-type ADR (carried since 2026-05-12-1254) + link-existence smoke (carried since 2026-05-13-0859) + spec-first codification (carried since 2026-05-13-1944). All three folded into P1's dedicated loop at `loop+2`.
- **Visual-docs brainstorm (parked):** a design-stage brainstorm for regenerating resink-core docs as standalone visual HTML + a `gen-resink-core-docs.py` generator + a `publish-to-gitbook.py` rewire (Approach 1 selected). Parked mid-design when this loop's theme was set; nothing written to disk. Resumable.
- **Showcase refresh cadence (2026-05-13-1303 retro P1):** demand-driven.
- **Fresh-clone verification of the walkthrough recipe (2026-05-13-1303 retro P2):** opportunistic.
- **Cargo CI cross-repo access** (private-compatible options documented; user-only decision).
- **Branch protection on resink-core master** (gated on GitHub Pro upgrade; user-only).
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing carry from 2026-05-23.
- **`<verified-against-rust-impl: pending>` tag in DE Kafka contract §2.1** — owner = DE.

## Tenant-isolation dry-run

Held trivially this loop. Zero `org-os/` edits. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `board/{okrs,decisions,exec-summaries,retros}/` and `docs/superpowers/specs/`. None of these are under `org-os/`; the invariant has no edits to gate.
{% endraw %}
