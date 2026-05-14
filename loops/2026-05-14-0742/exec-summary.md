---
layout: default
title: Exec Summary — 2026-05-14-0742
date: 2026-05-14
status: active
type: exec-summary
loop: 2026-05-14-0742
owner: board
grand_parent: Loops
parent: Loop 2026-05-14-0742
nav_order: 2
---

<!-- original-frontmatter:
  type: exec-summary
  owner: board
  date: 2026-05-14
  status: active
  loop: 2026-05-14-0742
  links: parent: board/okrs/2026-05-14-0742-ceo-brief.md
-->
{% raw %}

# Resink.ai Loop 2026-05-14-0742 — Company Exec Summary

**Headline.** First loop on a new calendar date (the prior seven all landed 2026-05-13). Design-first architectural loop: opened a **third** architectural arc with [ADR-2026-05-14-001](../decisions/2026-05-14-001-general-tables-rearchitecture.md), naming the supervisor's hardcoded `dim_user`/`dim_account` handling as a deliberate MVP deviation from the runtime spec's general-DAG-executor design. Companion spec at [docs/superpowers/specs/2026-05-14-general-tables-design.md](../../docs/superpowers/specs/2026-05-14-general-tables-design.md) ships the generic schema-driven node runner in concrete Rust (`NodeRunner`, `DimSchema`, `CompositeKey`, the NodeCtx bridge), the `RawFieldValue` type-coverage closure, composite-PK handling, the complex-fact-shape model, the supervisor-integration migration path, the test-harness shape, and a three-phase implementation-loop plan. No code this loop per the design-first scope decision. The streaming arc (ADR-2026-05-13-001 steps 3-4) is deferred behind this theme; the org now carries two open architectural arcs. Tenant-isolation invariant CLEAN. Single team active (board); all other teams paused.

## Per-objective rollup

### O1: ADR-2026-05-14-001 — general-tables rearchitecture — ✅ PASS

- New file at `board/decisions/2026-05-14-001-general-tables-rearchitecture.md`. Frontmatter conforms: `type: adr`, `status: active`, `date: 2026-05-14`, `decision:` field summarizes.
- Canonical ADR body shape preserved: Context (~6 paragraphs framing the deviation against `nodes.rs` / `supervisor.rs` / `RawFieldValue` evidence) → Decision (generic schema-driven node runner consuming `Node` via the dlopen C-ABI; three named phases) → 7 named Alternatives (A-G) with rejection reasons → Consequences (positive + costs + follow-ups) → Status section reserved → Links.
- **Three-phase restoration plan with explicit `loop+N` targets:** Phase 1 (supervisor de-hardcoding; resink-core; M; `loop+1`); Phase 2 (richer schemas + field types; resink-core + DE + AE + sim-farm; M-L; `loop+2`/`+3`); Phase 3 (complex fact-table shapes; resink-core + DE + training pipeline; L; `loop+4`/`+5`). Each phase has an explicit acceptance signal (a third synthetic dim; a float/timestamp/composite-PK dim; a fan-out fact stream).
- **Honest scope:** the ADR names what's NOT in scope (streaming ingestion; ADS / aggregate-table maintainers; cross-DAG / multi-tenant supervisor sharing; schema evolution of a deployed dim). Each gets its own future ADR.
- **Alternative E is load-bearing:** the schema-source-of-truth reconciliation (manifest node-spec vs DE `{key_columns, payload_columns}` vs codegen `schema_json`) is named explicitly as the single most likely cross-team friction site; the ADR decides the codegen `schema_json` is authoritative and the DE convention is a projection.
- Pattern lineage: third clean instance of the named-deviation + time-boxed-multi-loop-restoration shape. ADR-2026-05-16-001 (dlopen, closed) and ADR-2026-05-13-001 (streaming, steps 1-2 shipped) are the worked-example templates; this ADR's structure is observable at the section level as a copy.

### O2: General-tables design spec — ✅ PASS

- New file at `docs/superpowers/specs/2026-05-14-general-tables-design.md`. Frontmatter: `type: spec`, `owner: board`, `date: 2026-05-14`, `status: active`.
- Ten sections per the brief's prescribed shape: (1) goals & non-goals; (2) the generic node runner — concrete Rust (`NodeRunner`, `ShardKv`, `CompositeKey`, `RawFieldValueKey`, `Scd2RowGeneric`, the `NodeCtxBridge` trait); (3) the schema model + the three-shape reconciliation; (4) `RawFieldValue` type-coverage closure (`Float64`, `Timestamp`); (5) composite primary keys; (6) complex fact-table shapes (the model); (7) supervisor integration (what retires, the new `Supervisor::run` shape, composition with the streaming arc, the byte-stability migration contract); (8) test-harness shape (a third synthetic dim is the generality gate); (9) implementation-loop plan (the three-phase cross-walk table); (10) cross-references.
- The generic node-runner surface is concrete Rust, not pseudocode — `NodeRunner` + `DimSchema` + `CompositeKey` + `Scd2RowGeneric` + `NodeCtxBridge` all have signatures, bounds, doc comments. The `unimplemented!("Phase 1 — resink-core")` marker convention (from ADR-2026-05-13-001's `unimplemented!("loop+N")`) marks the bodies the implementing loop owns.
- **The spec names the deviation's root cause precisely:** §2.1 quotes `nodes.rs`'s own module comment — the per-node adapter exists because each codegen crate inlines its own distinct Rust `Event`/`Node` types. §2.2's fix (consume nodes purely through the type-erased C-ABI) is the load-bearing architectural move.
- **§9 flags the three most likely first-contact revision sites in advance:** the NodeCtx bridge wire shape (Phase 1, may pull an AE codegen-template change forward), the DE projection lossiness (Phase 2), derived facts needing the ADS sibling arc (Phase 3).

## Per-team rollup

### board (active, primary)

Two architectural authoring artifacts: 1 ADR (M) + 1 spec (M). Total sized M-L; design-first focus discipline held throughout (no code branches, no test runs, no compile steps). Tenant-isolation invariant trivially CLEAN — both artifacts live under `board/decisions/` and `docs/superpowers/specs/`, not `org-os/`.

### All other teams (paused, silent)

No team's surface was touched. The ADR names resink-core, DE, AE, sim-farm, and training pipeline as future-phase owners (per-phase assignments in the three-phase plan); none act this loop. resink-core owns the generic runner (Phase 1, every phase); DE owns the schema-JSON convention richness (Phase 2) and the fact-stream contract (Phase 3); AE owns the codegen-template type + composite-PK coverage (Phase 2) and possibly the NodeCtx-bridge C-ABI extension (Phase 1 first-contact); sim-farm owns engine composite-key awareness (Phase 2); training pipeline owns richer manifest emission (Phase 3).

## Cross-cutting wins

- **The arc-opener pattern is now mechanically reproducible — third clean instance.** ADR-2026-05-16-001 (closed), ADR-2026-05-13-001 (steps 1-2 shipped), and now ADR-2026-05-14-001 all use the same shape: name the deviation honestly in Context with code-level evidence; lay out a phased plan in Decision with `loop+N` targets per ADR-2026-05-30-002; enumerate alternatives; reserve a Status section. The reproducibility is mechanical — and this loop's authoring was visibly faster than the streaming arc's opener because the template is settled. **This is the third validating data point for `feedback-spec-first-architectural-arc.md` and strengthens the case for codifying the pattern in `org-os/conventions.md` (carryover P4 from the prior retro).**
- **The spec pinned the deviation's *root cause*, not just its symptom.** The symptom is "the supervisor hardcodes two tables." The root cause, surfaced by reading `nodes.rs`'s own module comment, is "each codegen crate inlines its own distinct Rust `Event`/`Node` types, so a single generic abstraction is impossible without consuming through the type-erased C-ABI." Naming the root cause means Phase 1's implementing loop executes against the real fix (pure-C-ABI consumption) rather than a surface patch.
- **Three arcs now compose into the documented runtime.** dlopen (closed) + streaming (in progress) + general-tables (opening) are the three deviations between the MVP supervisor and the runtime spec's "stateless general supervisor loading cdylib node plugins, fed by a streaming event source." Each arc was named, planned, and sequenced independently; the spec's §7.3 explicitly works the composition contract between general-tables and streaming so the two in-flight arcs don't collide.
- **Alternative E (schema-source-of-truth reconciliation) was named as cross-team friction *before* it became a blocker.** Three schema shapes exist (manifest node-spec, codegen `schema_json`, DE `{key_columns, payload_columns}`). Rather than discover the conflict mid-Phase-2, the ADR decides now: codegen `schema_json` is authoritative, DE convention is a projection. The spec's §3.2 flags the projection-lossiness risk as a pre-identified first-contact revision site.
- **Subsuming a deferred carryover into a new arc's Phase 1.** ADR-2026-05-16-001's "step 3.5 — supervisor-side NodeCtx bridge" had been demand-driven carryover with no scheduling pressure. The general-tables arc *needs* that bridge (the generic runner's connection to the node), so Phase 1 absorbs it. A deferred item found its forcing function by becoming load-bearing for a later arc — worth recording as a pattern.
- **Tenant-isolation invariant trivially held.** Eighth consecutive loop with zero `org-os/` edits (-0056, -0859, -1022, -1303, -1422, -1844, -1944, -0742). Product / architectural work runs cleanly under the existing process without new process authoring.

## Cross-cutting blockers

None.

## Asks for the CEO

- **(from board, next loop):** Bake Phase 1 of the ADR's restoration plan into the next CEO brief — generic `NodeRunner` + `DimSchema` + the NodeCtx bridge; `nodes.rs` retires; a third synthetic dim runs through the supervisor; `make mvp-loop` byte-stable. Resink-core owns; sized M; target `loop+1` per the ADR.
- **(from board, sequencing decision needed):** The org now carries **two open architectural arcs** — general-tables (Phases 1-3) and streaming (ADR-2026-05-13-001 steps 3-4). Both name resink-core as the primary code owner. The CEO must decide the interleave: strictly finish general-tables Phases 1-3 then resume streaming steps 3-4, or alternate. Recommendation: general-tables Phase 1 first (it subsumes the NodeCtx bridge and is load-bearing for the runtime's generality story), then re-evaluate. Flagged for the retro.
- **(from board, deferred — third time):** Multi-loop-blocker-arc report-type ADR (prior retro § P2) + link-existence smoke (§ P3) + spec-first codification in `org-os/conventions.md` (§ P4). Now deferred through three consecutive arc-opener loops. Strongly motivated; needs a non-resink-core loop that is *not itself an arc-opener* — the CEO should schedule a dedicated org-os-process loop.
- **(from board, awareness):** Branch protection on resink-core master is still gated on the GitHub Pro upgrade decision — single-contributor reality keeps the helm-CI-only setup acceptable; reevaluate when contributor count > 1.

## Decisions ratified this loop

**One new ADR drafted (ADR-2026-05-14-001) with `status: active`.** Third substantive arc-opener ADR. The trend: the org opens architectural arcs design-first as a settled habit, not an experiment.

## Decisions filed this loop (not new ADRs)

- **Design-first scope discipline — third application.** The architectural rearchitecture loop opens with an ADR + spec, not code. Pattern unchanged from ADR-2026-05-13-001's opener; the reproducibility is the point.
- **Schema source of truth: codegen `schema_json` is authoritative; the DE `{key_columns, payload_columns}` convention is a projection.** Recorded in ADR Alternative E + spec §3. Decided now, before Phase 2, to pre-empt cross-team friction.
- **A deferred carryover can find its forcing function by becoming load-bearing for a later arc.** ADR-2026-05-16-001's step-3.5 NodeCtx bridge — demand-driven carryover — is absorbed into general-tables Phase 1 because the generic runner needs it. Worth a convention note if it recurs.
- **Two-open-arc sequencing is a CEO decision, surfaced explicitly.** The general-tables arc deferred the streaming arc's steps 3-4; the brief recorded the deferral, the exec surfaces the interleave decision, the retro will carry it.

## Multi-loop plan slippage absorbed

The streaming arc (ADR-2026-05-13-001) steps 3-4 are **deferred, not slipped** — a deliberate CEO re-sequencing behind a new theme, not a missed target. ADR-2026-05-14-001's three-phase plan opens here; its first slip-record opportunity is at `loop+1`'s brief authoring.

## Request-flow metrics this loop (per ADR-2026-05-09-006)

| Metric | Count | Note |
|---|---|---|
| `cross-team-request` objectives `fulfilled` | 0 | None this loop. |
| `dropped` | 0 | None. |
| `declined` | 0 | None. |
| `escalated` | 0 | None. |
| In-loop requests filed | 0 | The ADR names future-phase owners but does not file requests (the brief is the request mechanism). |

## Tenant-isolation invariant

Held trivially. Zero `org-os/` writes this loop. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `board/{okrs,decisions,exec-summaries,retros}/` and `docs/superpowers/specs/`. None of these are under `org-os/`.

## Notes for the retro

Four patterns worth recording:

1. **Arc-opener pattern reproducibility at three instances.** Two clean instances was "canonical"; three is "mechanical, and visibly faster each time." This is now the strongest possible motivation for P4 (codify spec-first in `org-os/conventions.md`) — but P4 keeps being deferred because every recent loop has been an arc-opener. The retro should name the chicken-and-egg.
2. **Two open architectural arcs — coordination debt or healthy parallelism?** general-tables + streaming both name resink-core. The exec summary surfaced the interleave as a CEO decision; the retro should record the sequencing call and whether carrying two arcs is sustainable.
3. **Root-cause naming in the spec.** §2.1 quoting `nodes.rs`'s own module comment — the deviation's root cause is the per-crate inlined types, not the surface "two-table hardcoding." Worth recording: arc-opener specs should dig to root cause, because the implementing loop executes against whatever the spec names.
4. **A deferred carryover becoming load-bearing for a later arc.** The NodeCtx bridge. Worth a convention note: demand-driven carryovers don't need a forcing function invented for them — sometimes a later arc supplies one.
{% endraw %}
