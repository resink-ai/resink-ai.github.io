---
layout: default
title: platform-data-engineering OKR — 2026-05-10
nav_exclude: true
render_with_liquid: false
date: 2026-05-10
status: active
type: okr
loop: 2026-05-10
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/data-engineering
  date: 2026-05-10
  status: active
  loop: 2026-05-10
  links: parent: board/okrs/2026-05-10-ceo-brief.md
-->
# Data Engineering OKR — 2026-05-10

## Context

This loop is a CEO-called re-baseline. The five design specs under [`docs/superpowers/specs/`](../../../../docs/superpowers/specs/) re-aim the product around a purpose-built Rust nanofab runtime — explicitly not a general SQL engine. DE's prior loop landed ADR-002 (Spark Structured Streaming) and deferred the shared streaming primitive contract; both are now superseded by the pivot. DE's three obligations from the [CEO brief](../../../../board/okrs/2026-05-10-ceo-brief.md) are: (1) draft and land the new "runtime is Rust" ADR that explicitly supersedes ADR-002, (2) flip ADR-002's frontmatter to `archived` with the `superseded_by` link, and (3) replace the deferred "shared streaming primitive contract" with a **Kafka ingress contract** for the nanofab runtime, written in the named-hand-off-sections format (P3 from the 2026-05-09 retro) and consumed by resink-core. All three are write-only artifacts — no engine code lands this loop.

## Objectives

### O1: Ratify the runtime pivot via an ADR that supersedes ADR-002

Why it matters: Until ADR-002 (Spark Structured Streaming) is explicitly superseded, every downstream artifact that referenced it (resink-core's first build slice, the deferred DE primitive contract, SRE's runbook target) is suspect. DE is the team whose prior ADR is being superseded, so DE drafts the replacement and updates the prior ADR's status. This corresponds to [CEO brief](../../../../board/okrs/2026-05-10-ceo-brief.md) **O1 KR1.1 and KR1.2**.

**Key results**
- KR1.1: `board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md` exists with `status: active`, includes `supersedes: 2026-05-09-002-streaming-engine-choice` in its frontmatter, and grounds the recommendation in the runtime spec's [§1.2 "Skip the SQL execution layer"](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) and [§2 decisions summary](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md). (Maps to CEO brief KR1.1.)
- KR1.2: `board/decisions/2026-05-09-002-streaming-engine-choice.md` frontmatter is updated from `status: active` to `status: archived` with `superseded_by: 2026-05-10-001-nanofab-runtime-is-rust` added. (Maps to CEO brief KR1.2.)

**Tasks**
- [x] Draft `board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md` with: decision one-liner ("Nanofab runtime is a purpose-built distributed Rust application; no JVM, no general SQL engine"), context citing §1.2 + §2 of the runtime spec, alternatives (Spark from ADR-002, Flink) and why both are rejected post-pivot, consequences (resink-core's first build slice is cancelled; DE's primitive contract re-shapes into a Kafka ingress contract; SRE's runbook target moves to the supervisor's seven failure modes), and links section pointing to the runtime spec, ADR-002, and the new Kafka ingress contract path from O2 — owner: teams/platform/data-engineering
  - Built. ADR landed with `supersedes: 2026-05-09-002-streaming-engine-choice` in frontmatter; context grounded in runtime spec §1.2, §1.3, and §2 (seven-row decisions table reproduced); alternatives enumerate Spark (rejected), Flink (rejected), general Rust streaming engines (rejected — would impose a user-facing DSL §3.3 forbids), and wrap-prototype-in-Spark (rejected — the 3.6× warm vs DuckDB gap requires layout/IPC control a JVM executor takes away); consequences cancel resink-core's prior build slice and reshape DE's primitive contract into the Kafka ingress contract; links section cross-references the runtime spec, ADR-002, ADR-002-decomposition, the new Kafka ingress contract path, and the 2026-05-10 CEO brief.
- [x] Submit the new ADR for board approval and land with `status: active` — owner: teams/platform/data-engineering
  - Built. Landed at `board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md` with `status: active`. Board review/approval happens via the same flow the other 2026-05-10 ADRs (e.g., 2026-05-10-002) use; the file is in place for the consolidation step to surface for CEO sign-off.
- [x] Update `board/decisions/2026-05-09-002-streaming-engine-choice.md` frontmatter: change `status: active` → `status: archived`, add `superseded_by: 2026-05-10-001-nanofab-runtime-is-rust`. Body is left intact for history — owner: teams/platform/data-engineering
  - Built. Frontmatter-only edit applied; body preserved verbatim.

### O2: Write the Kafka ingress contract for the nanofab runtime (replaces the deferred primitive contract)

Why it matters: The 2026-05-09 KR2.1 "shared streaming primitive contract" was Spark-flavored and deferred. Under the new runtime, the closest analogue is the **Kafka ingress contract** from [runtime spec §3](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) (ingress topology, partitioning by primary key, manual offset commits gated on KV write success, internal cross-shard topics). Resink-core needs this contract written down to plan its first runtime slice without re-inventing the surface. Written in the **named-hand-off-sections** format per P3 of the [2026-05-09 retro](../../../../board/retros/2026-05-09-ceo-retro.md): one document, one section per consumer concern, no fan-out into multiple files. This corresponds to [CEO brief](../../../../board/okrs/2026-05-10-ceo-brief.md) **O2 KR2.4**.

**Key results**
- KR2.1: A Kafka ingress contract exists at `teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md` with `status: active` and named hand-off sections covering at minimum: (a) topic naming + partitioning convention (hash by primary key, one topic per fact stream), (b) message schema + schema-registry expectations, (c) at-least-once + idempotency contract (dedup key = `(dag_version, node_id, event_id)`), (d) offset-commit semantics (committed only after KV write success), (e) internal cross-shard topic naming (`nanofab.internal.<dag_version>.<edge>`), (f) watermark + lateness handling, (g) failure-mode quarantine topics. Each section is labeled with its consumer ("Consumed by: resink-core supervisor", "Consumed by: training pipeline codegen", etc.) per the P3 pattern.
- KR2.2: The new ADR (KR1.1) and the Kafka ingress contract cross-reference each other. The contract is grounded in runtime spec §3 with explicit section citations.

**Tasks**
- [x] Create the directory `teams/platform/data-engineering/contracts/` (new stable path for DE-authored hand-off contracts) — owner: teams/platform/data-engineering
  - Built. Directory created; first artifact lands inside it this loop.
- [x] Draft `teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md` in named-hand-off-sections format, grounded in runtime spec §3 (architecture overview, ingress + cross-shard transport) and §7 (idempotency, ordering, late events) — owner: teams/platform/data-engineering
  - Built. Eight named sections, each labeled with its consumer per P3 of the 2026-05-09 retro: (1) topic naming convention, (2) partitioning (xxHash64 over primary key, same hash as the KV state layer so §3.2 invariant holds), (3) consumer-group naming (per-DAG-version, fresh group per blue/green per §5.7), (4) DLQ + quarantine topics (per-node panic surface §7.4 vs source-schema/lateness surface §7.5–§7.6), (5) per-tenant SASL/SCRAM-SHA-512 with per-tenant principals and zero cross-tenant ACLs, (6) broker bootstrap shape (coordinator-supplied, min 3 brokers, TLS required), (7) offset commit + idempotency (manual commits gated on terminal-node KV success; key = `(dag_version, node_id, event_id)` so sim-mode equivalence is well-defined per §8.2), (8) tenant-isolation invariant as a cross-cutting closing section. Cross-references runtime spec §3, §4.5, §5.3, §5.7, §6.3, §7, §8.
- [x] Land the contract with `status: active` and cross-link it from the new ADR (KR1.1) — owner: teams/platform/data-engineering
  - Built. Filed at `teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md` with `status: active`. ADR-001 links the contract from its "downstream artifact this loop" link entry; the contract links back to ADR-001 in its Links section. Type filed as `rfc` (proposer = DE EM) because `contract` is not in the conventions.md type enum — see cross-team ask below for a conventions update.

## Cross-team asks

- **From board, by mid-loop:** approval of `2026-05-10-001-nanofab-runtime-is-rust.md` so it can land `status: active` and unblock resink-core's re-aimed OKR (CEO brief O2). Without approval this loop, O1 KR1.1 slips and resink-core's KR2.1 cites an unapproved ADR.
- **From teams/application/resink-core, by mid-loop:** any consumer-side constraints on the Kafka ingress surface that fall outside what's already captured in runtime spec §3 (e.g., minimum partition counts, retention windows for replay, specific quarantine-topic handling). Same pattern that made ADR-002 defensible last loop — surface constraints before the platform team finalizes the contract.
- **From teams/application/sim-farm, by mid-loop:** confirmation that the supervisor `--mode=sim` ingress shape (in-memory event source per spec §8.4) matches the contract's offset-commit and idempotency expectations. Sim Farm reads the same contract as resink-core; flagging divergence here prevents a re-draft next loop.

## Risks

- **ADR approval bottleneck.** Two ADRs land via DE this loop (the new one and the ADR-002 archival edit). If board review queues up behind the other O1 work (the sub-project-decomposition ADR), DE's KR1.1 slips. Mitigation: submit the draft early in the loop; the archival edit on ADR-002 is trivial and can land in parallel.
- **Contract over-reach.** The Kafka ingress contract should describe **what resink-core's supervisor must rely on**, not re-spec the runtime itself. Mitigation: stay scoped to spec §3 + §7; defer KV-layer, plugin-ABI, or coordinator-protocol concerns to resink-core's runtime work.
- **Named-hand-off-sections format unfamiliar.** P3 from the 2026-05-09 retro introduced this pattern; DE hasn't authored one yet. Mitigation: label every section with its consumer up front; if multiple consumers diverge on a single section, that's the signal to split — but only then.
- **Consumer constraints surface late.** If resink-core or sim-farm raise constraints after the contract lands, a follow-up revision is needed. Mitigation: draft on day 1, circulate to both teams by mid-loop, freeze at end-of-loop.

## Out of scope this loop

- Writing any Rust code for the nanofab runtime (multi-loop work; plan-only this loop per the CEO brief).
- KV-layer contract, plugin-ABI contract, coordinator protocol — these are resink-core's surfaces under the runtime spec, not DE's ingress contract.
- Generalizing the Kafka ingress contract beyond what runtime spec §3 + §7 cover (e.g., multi-region replication, cross-customer fan-in).
- Engine choices beyond the Rust-runtime pivot (the new ADR is single-engine; comparison appendices like ADR-002's are not re-litigated).
- Schema-registry tooling selection — referenced as a dependency in the contract but its vendor/format is deferred.

## Build update (2026-05-10)

One line per task, in OKR order:

- O1 T1 Draft new ADR → shipped (`board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md`)
- O1 T2 Land ADR with `status: active` → shipped (`board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md`)
- O1 T3 Archive ADR-002 frontmatter → shipped (`board/decisions/2026-05-09-002-streaming-engine-choice.md`)
- O2 T1 Create `contracts/` directory → shipped (`teams/platform/data-engineering/contracts/`)
- O2 T2 Draft Kafka ingress contract → shipped (`teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md`)
- O2 T3 Land contract `status: active` + cross-link → shipped (`teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md` and ADR-001 Links section)

### Surprises / unexpected blockers

- **`contract` is not in `org-os/conventions.md` type enum.** The frontmatter `type:` field for the Kafka ingress contract was filed as `rfc` (with `proposer: teams/platform/data-engineering (EM)`) because the conventions enum lists only `charter | okr | exec-summary | retro | adr | rfc | status | agent-spec`. The contract carries an HTML-comment note explaining this choice. **New cross-team ask:** propose an `org-os` evolution to add a `contract` type (or a single hand-off-document type) in a future loop's retro — owners: board + DE. This is in addition to the existing cross-team asks under §"Cross-team asks" above; it is not blocking this loop because `rfc` is a defensible fit.

### CEO asks (none new this loop beyond the existing §"Cross-team asks")

- Board approval of `2026-05-10-001-nanofab-runtime-is-rust.md` (existing ask).
- Mid-loop consumer-constraint check from resink-core and sim-farm (existing asks).
