---
layout: default
title: CEO Brief — 2026-05-14-0742
date: 2026-05-14
status: active
type: okr
loop: 2026-05-14-0742
owner: board
grand_parent: Loops
parent: Loop 2026-05-14-0742
nav_order: 1
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-14
  status: active
  loop: 2026-05-14-0742
  links: parent: ""
-->
{% raw %}

# Resink.ai CEO Brief — 2026-05-14 (loop 2026-05-14-0742)

## Context

First loop on a new calendar date — the prior seven loops (`2026-05-13-{0056,0859,1022,1303,1422,1844,1944}`) all landed on 2026-05-13. Two architectural arcs closed or progressed in that span: ADR-2026-05-16-001 (dlopen restoration) closed cleanly across five loops; ADR-2026-05-13-001 (streaming event-source) opened design-first and shipped steps 1 + 2.

This loop opens a **third** architectural arc, on a different deviation: **the supervisor hardcodes exactly two dim tables.**

**The gap.** The runtime spec describes the supervisor as a *general* DAG executor — it runs "a customer's generated DAG of dim/ADS-table maintainers." The MVP supervisor instead hand-codes `dim_user` and `dim_account` specifically:

- `crates/nanofab-supervisor/src/nodes.rs` (486 lines) is two hand-written modules — `mod user` and `mod account` — each with its own `raw_to_*_event` adapter, hardcoded primary-key column (`user_id` / `account_id`), hardcoded output-parquet column shape, hardcoded `ShardKv`, and its own `run()` function.
- `crates/nanofab-supervisor/src/supervisor.rs` dispatches with `match node_spec.table.as_str() { "dim_user" | "dim_account" => {} other => Err(...) }` and then `buffers.remove("dim_user")` / `nodes::user::run(...)` literally.
- Adding a third dim table today means hand-writing a third `nodes::` module + a third match arm + a third `run()` call. The product cannot ingest an arbitrary customer schema.

Meanwhile, the pieces *around* the supervisor are already schema-driven: the `codegen-scd2-node` skill takes a `(schema_json, pattern_name, output_dir)` triple and slot-fills a per-dim crate; the orchestrator emits a manifest with per-node schemas; sim-farm's engine 0.3.0 is schema-aware via `--schema-ref`; DE owns a `{key_columns, payload_columns}` schema-JSON convention. **The generality bottleneck is the supervisor consuming nodes through hand-written per-table adapters — not the codegen producing them.**

Two adjacent gaps compound it:

- **`RawFieldValue` is missing types.** It carries `String | Int64 | Bool | Null` — but the codegen schema accepts five types: `string, int64, float64, bool, timestamp_ms`. The supervisor cannot even represent `float64` or `timestamp_ms` events its own codegen path accepts. `arrow_value_to_field` downcasts only String/Int64/Bool.
- **Composite primary keys are spec'd but not honored.** The codegen schema's `primary_key` is "a non-empty list of strings", but `nodes::user` / `nodes::account` hardcode single-column PK extraction (`user_id`, `account_id`).

**The shape of this loop.** **Design-first, no code.** Per CEO scope selection, this loop ships two artifacts:

- **O1: New ADR.** `board/decisions/2026-05-14-001-general-tables-rearchitecture.md` — names the current `dim_user`/`dim_account` hardcoding as a deliberate MVP deviation from the runtime spec's general-DAG-executor design; proposes a generic, schema-driven node runner; lays out a **three-phase** restoration plan with named owners and `loop+N` targets, mirroring ADR-2026-05-13-001's pattern.
- **O2: New spec.** `docs/superpowers/specs/2026-05-14-general-tables-design.md` — the generic schema-driven node-runner shape, the `RawFieldValue` type-coverage closure, composite-PK handling, the complex-fact-shape model, the supervisor-integration migration path, the test-harness shape, the implementation-loop plan. Sized M; the implementing loops read this as the canonical contract.

This shape **mirrors the ADR-2026-05-13-001 arc start** exactly — file the named-deviation ADR + design spec now; subsequent loops execute against the agreed shape. Per the memory'd spec-first discipline (`feedback-spec-first-architectural-arc.md`), validated across two prior arcs.

**The three-phase plan (preview; the ADR is canonical):**

- **Phase 1 — De-hardcode the supervisor's table dispatch.** Retire `nodes::user` / `nodes::account` and the `match` on table name. Replace with a generic schema-driven node runner: the supervisor reads each node-spec's schema, builds a generic per-dim handler (PK extraction from the schema's `primary_key` list; output-parquet column shape from the schema's columns), routes `RawEvent`s to it. The dlopen `PluginNode` path (ADR-2026-05-16-001) is the natural vehicle — consume nodes through the `Node` C-ABI, not hand-written adapters. This also subsumes ADR-2026-05-16-001's deferred "step 3.5 supervisor-side NodeCtx bridge." Owner: resink-core.
- **Phase 2 — Richer schemas + field types.** `RawFieldValue` gains `Float64` + `Timestamp` (closes the gap vs the codegen's five accepted types); `arrow_value_to_field` handles them; composite primary keys flow through Phase 1's generic runner (the `primary_key` list with len > 1). Owners: resink-core (supervisor + `RawFieldValue`) + DE (schema-JSON convention richer column descriptors) + AE (codegen template type + composite-PK coverage) + sim-farm (engine composite-key awareness).
- **Phase 3 — Complex fact-table shapes.** Multiple fact streams feeding one dim with mixed ops (partially supported today via the per-row `op` override + the `fact_streams` list); fact tables that fan out to multiple dims; derived / joined facts. Owners: resink-core (supervisor fact routing) + DE (fact-stream contract) + training pipeline (orchestrator manifest emission).

**Probe results entering the loop:**

- `crates/nanofab-supervisor/src/nodes.rs` — 486 lines; `pub mod user` (lines 33-258) + `pub mod account` (lines 265-486); each hand-written, hardcoded PK + output shape.
- `crates/nanofab-supervisor/src/supervisor.rs` — `match node_spec.table.as_str()` validation + `buffers.remove("dim_user"|"dim_account")` + literal `nodes::user::run` / `nodes::account::run` calls.
- `crates/nanofab-supervisor/src/event_source/raw.rs` — `RawFieldValue { String, Int64, Bool, Null }`; `arrow_value_to_field` downcasts String/Int64/Bool only.
- `codegen-scd2-node/SKILL.md` — schema accepts five types (`string, int64, float64, bool, timestamp_ms`); `primary_key` is a non-empty list; `scd2_columns` triple.
- `teams/platform/data-engineering/conventions/dim-schema-json.md` — `{key_columns, payload_columns}` convention; DE owns the data-shape vocabulary.
- Tenant-isolation entering: clean (single canonical `acme.ai` placeholder at `org-os/conventions.md:121`).

**Re recent retro carryovers:**

- **2026-05-13-1944 retro P1 (streaming step 3 — Kafka + broker):** The streaming arc (ADR-2026-05-13-001 steps 3-4) is **deferred behind this theme** per CEO direction. The arc is not abandoned — steps 3-4 remain carryover; the CEO will re-sequence them after the general-tables arc opens. Decision recorded explicitly.
- **2026-05-13-1944 retro P2 (multi-loop-blocker-arc report-type ADR) + P3 (link-existence smoke) + P4 (spec-first codification in `org-os/conventions.md`):** all queued for "next non-resink-core loop." This loop IS non-resink-core (board authoring only) — but this loop authors a general-tables ADR + spec; bundling two more org-os ADRs (P2 + P4) would triple the authoring surface, exactly the reasoning that deferred them at loop 2026-05-13-1422. **Defer P2 + P3 + P4 again**, to the next non-resink-core loop that is not itself an arc-opener. Decision recorded explicitly.
- **2026-05-13-1944 retro P5 (`drain_to_vec`-style bridge pattern):** demand-driven; not this loop.

**CEO decisions for this loop:**

- **Activate board only.** Board active (ADR + spec are board-class authoring). All other teams paused — the ADR names resink-core, DE, AE, sim-farm, and training pipeline as future-phase owners; none act this loop.

- **2 objectives, both board-owned authoring.** O1 ADR (M) + O2 spec (M). Total sized M-L; design-first discipline means no code branches, no test runs, no compile steps this loop.

- **Naming the deviation matters as much as the plan.** The ADR's framing — the hardcoded two-table supervisor is a deliberate MVP shortcut against a fully-specified general-DAG-executor design, not a missing feature — sets the right operator mental model. Same body shape as ADR-2026-05-13-001 + ADR-2026-05-16-001: Context → Decision → Alternatives → Consequences → multi-loop plan → Status (reserved) → Links.

- **Phase 1 is the load-bearing phase; the ADR sequences it first.** De-hardcoding the supervisor unblocks everything else — Phase 2 (richer types) and Phase 3 (complex facts) both assume a generic runner exists to extend. The ADR's plan makes Phase 1 `loop+1`, Phase 2 `loop+2/+3`, Phase 3 `loop+4/+5`.

- **The spec sizes the implementing loops, not the implementation.** Spec §"implementation-loop plan" breaks the work into loops with owner + sized estimate + target loop + KR shape — the cross-walk table future briefs cite.

- **No code this loop.** Per CEO decision and the design-first scope. Subsequent loops execute.

- **Tenant-isolation invariant maintained.** New artifacts under `board/decisions/` and `docs/superpowers/specs/`; both tenant-bearing, not under `org-os/`. Zero `org-os/` edits expected.

**Standing CEO answers:**

- **ADR body shape.** Mirror the canonical ADR template (ADR-2026-05-13-001 / ADR-2026-05-16-001): Context → Decision → Alternatives considered → Consequences → three-phase restoration plan (per-phase owner + `loop+N` target per ADR-2026-05-30-002) → Status section reserved → Links. Alternatives to evaluate explicitly: keep hand-written per-table modules forever; a codegen-the-supervisor-too approach; a generic trait-object runner consuming `Node` via the dlopen C-ABI (the recommended path); a fully data-driven interpreter with no per-dim crate at all; whether the schema source of truth is the manifest, the DE schema-JSON convention, or the codegen schema_json — and how the three reconcile.

- **Spec body shape.** Header + frontmatter (`type: spec`, `owner: board`, `date: 2026-05-14`, `status: active`) + sections covering: (1) goals & non-goals; (2) the generic node-runner shape (how the supervisor builds a per-dim handler from a schema — Rust signatures); (3) the schema model (how the manifest node-spec, the DE `{key_columns, payload_columns}` convention, and the codegen `schema_json` reconcile into one supervisor-side schema type); (4) `RawFieldValue` type-coverage closure (`Float64`, `Timestamp`) + `arrow_value_to_field` extension; (5) composite primary-key handling; (6) complex fact-table shapes (multi-stream-per-dim, fan-out, derived facts) — the model, not the full impl; (7) supervisor integration (what retires from `nodes.rs` + `supervisor.rs`, the migration path, what `make mvp-loop` must keep green); (8) test-harness shape (how a third synthetic dim table proves generality); (9) implementation-loop plan (the three-phase cross-walk); (10) cross-references (runtime spec, ADR-2026-05-16-001, ADR-2026-05-13-001, DE schema-JSON convention, codegen SKILL).

- **Honest scope on what this ADR doesn't cover.** Streaming ingestion (ADR-2026-05-13-001's domain — the general-tables runner must compose cleanly with the `EventSource` trait, but Kafka is not in this arc); ADS / aggregate tables beyond plain SCD2 dims (the runtime spec mentions "dim/ADS-table maintainers" — ADS is a sibling arc); cross-DAG / multi-tenant supervisor sharing; schema evolution / migration of an already-deployed dim. Each named as out-of-scope; each its own future ADR when motivated.

- **`make mvp-loop` is the generality gate's anchor.** The spec's test-harness section must specify: the existing two-dim fixture stays byte-stable through every phase, AND a *third* synthetic dim table (proving the runner is general) is the acceptance signal for Phase 1. The third dim is the thing that cannot be faked by the hardcoded path.

**Carryover load by team:**

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| board | O1 ADR (M) + O2 spec (M) + brief + exec + retro | M-L | Design-first focus loop; third arc-opener using the proven ADR-2026-05-13-001 template. |
| All other teams | Paused; no review-ack ask | — | Paused. resink-core / DE / AE / sim-farm / training pipeline are named as future-phase owners in the ADR's plan; none act this loop. |

## Objectives

### O1: New ADR — general-tables rearchitecture

source: ceo-brief

Why it matters: First architectural artifact that names the gap between the runtime spec's general-DAG-executor design and the MVP's two-table-hardcoded supervisor. Establishes the three-phase plan that will close the gap. Mirrors the ADR-2026-05-13-001 + ADR-2026-05-16-001 pattern — name the deviation honestly, phase the restoration, name owners per phase. The ADR is also the canonical entry point for any future contributor asking "why can the supervisor only run dim_user and dim_account?"

KR1.1: New file at `board/decisions/2026-05-14-001-general-tables-rearchitecture.md`. Frontmatter conforms (`type: adr`, `status: active`, `date: 2026-05-14`, `decision:` field summarizing the rearchitecture).

KR1.2: Body sections per the canonical ADR template: **Context** (the deviation: two-table-hardcoded supervisor vs general-DAG-executor spec, with the `nodes.rs` / `supervisor.rs` / `RawFieldValue` evidence); **Decision** (generic schema-driven node runner consuming `Node` via the dlopen C-ABI); **Alternatives considered** (≥4 named with rejection reasons — see the standing CEO answer); **Consequences** (positive + costs + follow-ups); **three-phase restoration plan** (per-phase owner + `loop+N` target); **Status** (reserved for future-loop closure narration); **Links**.

KR1.3: The three-phase plan names per-phase owners + `loop+N` targets: Phase 1 (supervisor de-hardcoding; resink-core; M; `loop+1`); Phase 2 (richer schemas + field types; resink-core + DE + AE + sim-farm; M-L; `loop+2`/`loop+3`); Phase 3 (complex fact-table shapes; resink-core + DE + training pipeline; L; `loop+4`/`loop+5`). Each phase closes named spec sections by code, per the ADR-2026-05-13-001 convention.

KR1.4: Honest scope. The ADR names what it does **not** cover: streaming ingestion (ADR-2026-05-13-001); ADS / aggregate-table maintainers; cross-DAG / multi-tenant supervisor sharing; schema evolution of a deployed dim. Each a sibling arc.

KR1.5: Tenant-isolation invariant holds. The ADR lives under `board/decisions/`, not `org-os/`. Zero `org-os/` edits this loop.

### O2: New design spec — general-tables contract

source: ceo-brief

Why it matters: The ADR names the decision; the spec is the contract the implementing loops execute against. Without it, each phase re-litigates the node-runner shape, the schema model, the type set. With it, each phase's OKR maps directly to spec sections.

KR2.1: New file at `docs/superpowers/specs/2026-05-14-general-tables-design.md`. Frontmatter `type: spec`, `owner: board`, `date: 2026-05-14`, `status: active`.

KR2.2: Body has the ten sections named in the standing CEO answer (goals & non-goals; generic node-runner shape; schema model reconciliation; `RawFieldValue` type closure; composite-PK handling; complex fact shapes; supervisor integration + migration; test-harness shape; implementation-loop plan; cross-references).

KR2.3: The generic node-runner shape is concrete Rust — signatures, trait bounds, the per-dim-handler struct built from a schema. Not pseudocode. Future contributors copy it into the supervisor with minimal interpretation friction. The `Kafka`-style `unimplemented!("loop+N")` marker convention may be used where a section's body genuinely belongs to the implementing loop.

KR2.4: The spec is size-conservative (~600-900 lines). Detailed enough not to need re-litigation; not so detailed it locks in implementation choices the implementing loops should own.

KR2.5: The implementation-loop plan section (§9) is the cross-walk between spec sections and the ADR's three phases — the table future briefs cite and future OKR KRs map to by row.

KR2.6: Tenant-isolation invariant holds.

## What success looks like

- `board/decisions/2026-05-14-001-general-tables-rearchitecture.md` exists; frontmatter conforms; body has all six canonical ADR sections + the three-phase plan + the reserved Status section.
- `docs/superpowers/specs/2026-05-14-general-tables-design.md` exists; ~600-900 lines; the generic node-runner shape is concrete Rust; the implementation-loop plan maps to the ADR's three phases.
- Zero `org-os/` edits this loop; tenant-isolation grep returns the single canonical placeholder.
- Both artifacts publish through `scripts/publish-to-gitbook.py`; post-publish Jekyll CI green per standing practice.
- Retro surfaces: (i) third clean instance of the arc-opener pattern — is it now mechanical enough to template? (ii) the schema-model reconciliation (manifest node-spec vs DE `{key_columns, payload_columns}` vs codegen `schema_json`) — did the spec settle it cleanly or surface a contradiction needing a DE contract change? (iii) the streaming-arc deferral — is re-sequencing two open arcs (general-tables + streaming) creating coordination debt?

## Out of scope

- Any code change in `crates/nanofab-supervisor/` or anywhere else. Design-first; code lands in subsequent loops per the three-phase plan.
- Streaming ingestion / Kafka. ADR-2026-05-13-001's domain; the general-tables runner must compose with the `EventSource` trait but Kafka is not in this arc.
- ADS / aggregate-table maintainers. Sibling arc; future ADR.
- Cross-DAG / multi-tenant supervisor sharing. Sibling arc; future ADR.
- Schema evolution / migration of an already-deployed dim. Sibling arc; future ADR.
- Bundling carryover ADRs (2026-05-13-1944 retro P2 arc-report-type; P3 link-existence smoke; P4 spec-first codification). Deferred again — this loop is itself an arc-opener; tripling the authoring surface is the failure mode.
- Advancing the streaming arc (ADR-2026-05-13-001 steps 3-4). Deferred behind this theme; remains carryover.
- Any `org-os/` process work. Org-os surface is stable; this loop is product-architecture, not process.
{% endraw %}
