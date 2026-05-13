---
layout: default
title: CEO Brief — 2026-05-13-1422
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-1422
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1422
  links: parent: ""
-->
{% raw %}

# Resink.ai CEO Brief — 2026-05-13 (loop 2026-05-13-1422)

## Context

Fifth loop on the same calendar date. The prior four closed: org-os ratification bundle (`-0056`); observability skill + hot-swap AE-side (`-0859`); resink-core bundle close-out / ADR-2026-05-16-001 retires (`-1022`); showcase pair + landing reframe (`-1303`). The showcase report's "Honest deferrals" section explicitly named **real Kafka / event-stream ingestion** as the largest unbuilt piece of the documented architecture — the supervisor reads parquet event-source files in `--mode=sim`; the runtime spec's Kafka contract is fully specified but no consumer code ships. This loop opens the architectural arc that closes that deferral.

**The gap.** Distance from vision is now **streaming ingestion**. The runtime spec at `docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md` is detailed about the streaming architecture: Kafka/Redpanda for ingress, consumer-group protocol for shard rebalancing, watermarks on internal Kafka topics, cross-shard re-keying via dedicated topics. None of that is in the supervisor today. The MVP shipped with `read_fact_parquet` returning `Vec<RawEvent>` (eager materialization), which was the right v1 — but it's a deliberate simplification of a documented streaming architecture.

**The shape of this loop.** **Design-first, no code.** Per CEO scope selection, this loop ships two artifacts:

- **O1: New ADR.** `board/decisions/2026-05-13-001-streaming-event-source-rearchitecture.md` — names the current parquet-event-source as a deliberate MVP deviation from the runtime spec's streaming-ingestion design; proposes any-source streaming abstraction with Kafka as the first real implementation; lays out a multi-loop restoration plan with named owners and time-boxes per the established pattern of ADR-2026-05-16-001 (which just closed cleanly across five loops).
- **O2: New spec.** `docs/superpowers/specs/2026-05-13-streaming-event-source-design.md` — the `EventSource` trait shape, lifecycle, watermark + offset semantics, two reference impls (parquet-file replay + Kafka consumer), back-pressure model, error handling, test-harness shape. Sized M; ~600-900 lines. The implementing loops read this as the canonical contract.

This shape **mirrors the ADR-2026-05-16-001 arc start** exactly. That arc filed a named deviation + a multi-loop plan; each step had a named owner and a target loop; the plan closed in 5 loops with one mid-arc planning re-shape. The streaming arc follows the same pattern: file the ADR + spec now; subsequent loops execute against the agreed shape.

**Probe results entering the loop:**

- Existing event-source code at `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/event_source.rs`: `RawEvent` struct, `read_fact_parquet` function, `events_for(specs)` aggregator. Returns `Vec<RawEvent>` — fully eager.
- Runtime spec §3 ("Architecture overview"), §4 ("The Supervisor"), §5.4 ("Event shape"), §6 ("Shard-routing & internal Kafka") all specify the streaming design in detail. The ADR + spec this loop reify the spec's existing design at the code-shape level.
- No Kafka code anywhere in `repos/resink-ai/resink-core/`. Adding Kafka to a future loop introduces a new external dependency: a Rust Kafka client (likely `rdkafka` or `rskafka`).
- Tenant-isolation entering: clean (single canonical `acme.ai` placeholder at `org-os/conventions.md:121`).

**Re recent retro carryovers:**

- **2026-05-13-1303 retro P3 (multi-loop-blocker-arc report-type ADR):** queued for next non-resink-core loop. This loop is non-resink-core (no code touch). Could bundle — but the streaming rearchitecture is itself an ADR + spec, and bundling another ADR doubles the authoring surface. **Defer P3 to the loop after this one** (or to whichever loop has bandwidth + the right composition). Decision recorded explicitly.
- **2026-05-13-1303 retro P4 (link-existence smoke for publish-to-gitbook):** also queued. Same logic — defer.
- **2026-05-13-1303 retro P2 (fresh-clone verification of the walkthrough recipe):** opportunistic; not this loop.
- **2026-05-13-1022 retro P5 (cargo CI cross-repo access):** demand-driven; private-compatible options listed.
- **2026-05-13-1022 retro P6 (branch protection on private repo):** indefinite defer; user-only decision.
- **2026-05-13-1022 retro P2 (supervisor-side NodeCtx bridge):** demand-driven step-3.5 follow-up.

**CEO decisions for this loop:**

- **Activate one team + board.** Board active (ADR + spec are board-class authoring). All other teams paused — the ADR names resink-core, DE, and devops as future-loop owners; this loop none of them need to act.

- **2 objectives, both board-owned authoring.** O1 ADR (M) + O2 spec (M). Total sized M-L; design-first focus discipline means no code branches, no test runs, no compile steps this loop. Reduces the cognitive load to the architectural shape.

- **Naming the deviation matters as much as the restoration plan.** The ADR's framing — current parquet-event-source is a deliberate MVP deviation from a fully-specified streaming architecture, not a missing feature — sets the right operator mental model. ADR-2026-05-16-001 worked partly because the deviation was named honestly + the plan time-boxed. This ADR follows the same body shape (Context, Decision, Alternatives, Consequences, multi-loop plan, Status section reserved for future closure).

- **Spec sizes the implementation loops, not the implementation itself.** The spec includes a "Implementation-loop plan" section breaking the work into loops, each with: owner, sized estimate, target loop, KR shape. Future briefs cite the spec; future OKRs map their KRs to spec sections. This is the same shape the runtime spec used for its own multi-loop sub-projects.

- **No code this loop.** Per CEO decision and per the design-first scope. Subsequent loops execute.

- **Tenant-isolation invariant maintained.** New artifacts under `board/decisions/` and `docs/superpowers/specs/`; both are tenant-bearing, not under `org-os/`. Zero `org-os/` edits expected.

**Standing CEO answers:**

- **ADR body shape.** Mirror the canonical ADR template (used by ADR-2026-05-16-001 et al.): Context → Decision → Alternatives considered → Consequences → multi-loop restoration plan (with per-step owner + target loop) → Status section reserved → Links. The streaming ADR has more substance in Alternatives (parquet streaming vs Kafka vs Kinesis vs in-memory queue first; pull vs push semantics; sync vs async; rdkafka vs rskafka) than ADR-2026-05-16-001 did. Be explicit about each.

- **Spec body shape.** Header + frontmatter (`type: spec`, `status: active`, `date: 2026-05-13`) + sections covering: (1) goals & non-goals; (2) the `EventSource` trait shape (Rust signatures); (3) lifecycle (start, poll, commit, shutdown); (4) watermark + offset semantics (per the runtime spec §6.2); (5) two reference implementations (`ParquetReplay`, `Kafka`); (6) error handling + retry; (7) integration with the existing supervisor (where the trait sits, what changes in main.rs); (8) test-harness shape (how integration tests drive the supervisor from a synthetic EventSource); (9) implementation-loop plan; (10) cross-references to the runtime spec.

- **No premature naming of subsystems we haven't designed.** Watermarks, cross-shard topics, KV state — all are documented in the runtime spec but the ADR + spec this loop focus on the **EventSource trait + Kafka ingestion only**. Cross-shard re-keying and persistent state are separate arcs; the ADR can name them as out-of-scope.

- **Honest scope on what this ADR doesn't cover.** Multi-tenant supervisor sharing, persistent state backend, streaming diff (sim-farm Mode B), schema-registry-aware codecs — all stay deferred. Each is a sibling arc that the streaming rearchitecture enables but doesn't include.

**Carryover load by team:**

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| board | O1 ADR (M) + O2 spec (M) + brief + exec + retro | M-L | Design-first focus loop; mirrors the ADR-2026-05-16-001 arc-start shape. Reproducible: 5 loops from now an arc-close retro can narrate this loop's framing. |
| All other teams | Paused; no review-ack ask | — | Paused. Resink-core, DE, devops will be named as future-loop owners in the ADR's restoration plan; they don't act this loop. |

## Objectives

### O1: New ADR — streaming-event-source rearchitecture

source: ceo-brief

Why it matters: First architectural artifact that names the gap between the runtime spec's documented streaming architecture and the MVP's parquet-eager implementation. Establishes the multi-loop plan that will close the gap. Mirrors the ADR-2026-05-16-001 pattern exactly — name the deviation honestly, time-box the restoration, name owners per step. The ADR is also the canonical entry point for any future contributor evaluating "why doesn't this read from Kafka?"

KR1.1: New file at `board/decisions/2026-05-13-001-streaming-event-source-rearchitecture.md`. Frontmatter: `type: contract` (or `type: rfc` — pick what conventions.md requires for this class), `status: active`, `date: 2026-05-13`, `decision:` field naming the rearchitecture summary.

KR1.2: Body sections per the canonical ADR template: **Context** (the deviation: parquet-eager vs streaming-spec); **Decision** (any-source EventSource trait with Kafka as first real impl); **Alternatives considered** (≥4 named alternatives with rejection reasons: keep parquet-eager forever; in-memory queue first; Kafka directly with no abstraction; Kinesis instead of Kafka; pull vs push semantics; async vs sync trait); **Consequences** (positive + negative + follow-ups required); **Multi-loop restoration plan** (per-step owner + target loop, mirror the ADR-2026-05-16-001 §"Decision" three-step shape); **Status** (left blank; future loops update); **Links** (runtime spec, related ADRs, this loop's spec doc).

KR1.3: Restoration plan names at least 3 loops with explicit ownership: (a) supervisor extracts `EventSource` trait + parquet impl + in-memory impl (resink-core; M); (b) supervisor gains Kafka impl + integration test against a local broker (resink-core + devops for the broker; L); (c) the closed-loop verdict pipeline runs against the Kafka-driven supervisor with a synthetic-tenant fixture (resink-core + sim-farm; M). Each step has a target loop expressed as `loop+N` per ADR-2026-05-30-002 (relative dating).

KR1.4: Honest scope. The ADR names what it does **not** cover: persistent KV state backend (separate arc); multi-tenant supervisor sharing (separate arc); sim-farm Mode B streaming differ (separate arc); cross-shard re-keying via internal Kafka topics (deferred to a follow-up after the basic Kafka ingestion lands).

KR1.5: Tenant-isolation invariant holds. The ADR lives under `board/decisions/`, not `org-os/`. Zero `org-os/` edits this loop.

### O2: New design spec — streaming-event-source contract

source: ceo-brief

Why it matters: The ADR names the decision; the spec is the contract the implementing loops execute against. Without the spec, each implementing loop re-litigates the trait shape, the lifecycle semantics, the error model. With it, each loop's OKR maps directly to spec sections.

KR2.1: New file at `docs/superpowers/specs/2026-05-13-streaming-event-source-design.md`. Frontmatter `type: spec`, `owner: board`, `date: 2026-05-13`, `status: active`.

KR2.2: Body sections: **Goals & non-goals** (what this spec is for + what it explicitly defers); **The `EventSource` trait** (Rust signatures with doc comments; the trait method set); **Lifecycle** (start, poll, commit, shutdown — when each fires, what guarantees each provides); **Watermark + offset semantics** (per runtime spec §6.2; how the trait surfaces watermarks; how the supervisor advances offsets); **Reference implementations** (`ParquetReplay` for back-compat + tests; `Kafka` for production); **Error handling** (retryable vs fatal; how the trait surfaces each; supervisor response); **Integration with the supervisor** (where the trait sits in `crates/nanofab-supervisor/src/`; what changes in `main.rs`'s `run_supervisor()`; the migration path from the eager `read_fact_parquet` shape); **Test-harness shape** (how integration tests drive the supervisor from a synthetic in-memory EventSource without needing a broker); **Implementation-loop plan** (cross-link to the ADR's plan; per-loop KR shape); **Cross-references** (runtime spec sections, related ADRs).

KR2.3: The `EventSource` trait is **fully written in Rust** in the spec — function signatures, trait bounds, associated types. Not pseudocode. Future contributors copy it into the supervisor with minimal interpretation friction.

KR2.4: The spec is **size-conservative**: ~600-900 lines. Detailed enough to not need re-litigation; not so detailed that it locks in implementation choices the implementing loops should make.

KR2.5: Tenant-isolation invariant holds.

## What success looks like

- `board/decisions/2026-05-13-001-streaming-event-source-rearchitecture.md` exists; frontmatter conforms to conventions.md; body has all six canonical ADR sections + the multi-loop plan + the Status reservation.
- `docs/superpowers/specs/2026-05-13-streaming-event-source-design.md` exists; ~600-900 lines; the `EventSource` trait is fully written in Rust; implementation-loop plan maps to the ADR's restoration plan.
- Zero `org-os/` edits this loop; tenant-isolation grep returns single canonical placeholder.
- Both artifacts publish through `scripts/publish-to-gitbook.py` to `blog.resink.ai`; post-publish CI green per standing practice.
- Retro surfaces: (i) the design-first focus pattern — when does ADR-first work vs code-first? (ii) the ADR-arc-start template — what's reusable from the ADR-2026-05-16-001 shape vs what's bespoke per arc? (iii) the spec's "implementation-loop plan" section — does this shape future OKRs cleanly or does it duplicate effort?

## Out of scope

- Any code change in `crates/nanofab-supervisor/` or anywhere else. Design-first; code lands in subsequent loops per the multi-loop plan.
- Persistent KV state backend. Sibling arc; future ADR.
- Multi-tenant supervisor sharing. Sibling arc; future ADR.
- Sim-farm Mode B streaming differ. Sibling arc; future ADR.
- Cross-shard re-keying via internal Kafka topics. Deferred to a follow-up after basic Kafka ingestion.
- Schema-registry-aware event codecs. Documented in the runtime spec; out of scope for the ingestion ADR.
- Bundling carryover ADRs (P3 multi-loop-blocker-arc report-type; P4 link-existence smoke). Both deferred to a future non-resink-core loop with bandwidth.
- Any `org-os/` process work. Org-os surface is stable; this loop is product-architecture, not process.
- Bundling broker setup or any other deployment work. The ADR names devops as a future-loop owner for broker provisioning.
{% endraw %}
