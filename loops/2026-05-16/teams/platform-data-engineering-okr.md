---
layout: default
title: platform-data-engineering OKR — 2026-05-16
date: 2026-05-16
status: active
type: okr
loop: 2026-05-16
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/data-engineering
  date: 2026-05-16
  status: active
  loop: 2026-05-16
  links: parent: board/okrs/2026-05-16-ceo-brief.md
-->
# Data Engineering OKR — 2026-05-16

## Context

Light loop for DE. Last loop landed the [Kafka ingress contract](../contracts/2026-05-10-kafka-ingress.md) and ratified the runtime-is-Rust ADR; the [CEO brief](../../../../board/okrs/2026-05-16-ceo-brief.md) re-aims the company at a single closed-loop MVP on a developer laptop. That MVP runs **without** Kafka — resink-core's supervisor consumes parquet files via an in-memory event source. Per the brief, "no revision required" on the Kafka contract; instead DE ships **one** small additive contract (the in-memory event-source addendum) that maps field-for-field onto the Kafka contract so the eventual swap is mechanical. This is brief O4 — a contract loop, not a code loop.

## Objectives

### O1: Ship the in-memory event-source contract addendum the MVP supervisor consumes

- source: ceo-brief

Why it matters: O1 KR1.4 of the [CEO brief](../../../../board/okrs/2026-05-16-ceo-brief.md) requires resink-core's supervisor to read the two MVP fact parquets via an in-memory event source. Resink-core needs a single, named `Event` shape that (a) matches [runtime spec §5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) byte-stably, and (b) maps one-to-one onto the Kafka contract's payload fields so that the future swap from in-memory to Kafka is mechanical. DE owns event-source contracts; this is squarely DE's surface. Maps to CEO brief **O4 KR4.1, KR4.2, KR4.3**.

**Key results**
- KR1.1: `teams/platform/data-engineering/contracts/2026-05-16-in-memory-event-source.md` exists with `status: active`, filed as `type: rfc` per the still-open `contract` enum workaround (proposer = DE EM). Defines the `Event` record shape per [runtime spec §5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md): `{table, key: {<pk>: <value>}, op: insert|update|delete, before?: <row>, after?: <row>, event_ts, event_id}`. Defines the in-memory source's contract: events arrive in `event_ts` order, partitioned by `hash(key) % shard_count`, no replay, no DLQ, the file path IS the topic. Maps to CEO brief KR4.1.
- KR1.2: Contract includes a "Production parity" section that maps every in-memory `Event` field one-to-one to the existing [Kafka ingress contract](../contracts/2026-05-10-kafka-ingress.md)'s payload fields. The mapping makes the swap from in-memory to Kafka mechanical — no field is renamed, dropped, or re-typed at the swap. Maps to CEO brief KR4.2.
- KR1.3: Contract restates the per-fact partitioning convention from §2 of the Kafka ingress contract: **xxHash64 (seed=0)** over the canonical UTF-8 byte representation of the primary-key field, modulo shard count. Resink-core's in-memory source uses this same hash so the swap is byte-stable. Default shard count for the MVP is 1 (single-shard fixture), but the hash function and width are pinned now. Maps to CEO brief KR4.3.
- KR1.4: Contract carries a named **"Consumer hand-off: resink-core"** section that surfaces any consumer-side constraints the in-memory source places on the supervisor — e.g., expectation that the supervisor sorts by `event_ts` on load (because the in-memory source is a parquet file, not a stream), expectation that `event_id` is non-empty (matches Kafka contract §7), expectation that `before`/`after` row shapes match the dim's SCD2 column set. Resink-core confirms or pushes back by mid-loop (2026-05-19).
- KR1.5: Contract cross-links the [Kafka ingress contract](../contracts/2026-05-10-kafka-ingress.md), [runtime spec §5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md), and the [CEO brief](../../../../board/okrs/2026-05-16-ceo-brief.md) O4. The Kafka contract gets a one-line link back from its Links section pointing at this addendum (the only edit to the Kafka contract this loop — pure cross-reference, not a content revision).

**Tasks**
- [x] (Optional, this turn) Create the contract file at `teams/platform/data-engineering/contracts/2026-05-16-in-memory-event-source.md` with frontmatter and a placeholder body marked `status: draft` — owner: teams/platform/data-engineering. (Per planning ritual: the contract body itself is a build-phase deliverable; the OKR ships first.)
  - Created `teams/platform/data-engineering/contracts/2026-05-16-in-memory-event-source.md`. Skipped the `status: draft` intermediate — landed straight at `status: active` since the body was authored in the same turn (next task). Frontmatter validates against `org-os/conventions.md` (`type: rfc` per the still-open `contract` workaround; required `proposer` field present; `links.parent` points at this OKR).
- [x] Draft the contract body: Event shape (§5.4-grounded JSON shape with worked example), in-memory source semantics (event_ts ordering, no replay, no DLQ, file-path-is-topic), Production parity table (in-memory ↔ Kafka field-by-field), partitioning restatement (xxHash64 seed=0, MVP shard_count=1) — owner: teams/platform/data-engineering.
  - Body organized as §1 Event record shape (pseudo-Rust + worked JSON example identical to the OKR sketch — `dim_user`, `user_id=42`, US→CA update at `event_ts=1715400000000`), §2 In-memory source semantics (event_ts ascending sort with `event_id` lexicographic tiebreak, no replay, no DLQ, no quarantine, file-path-is-topic, no watermark fan-out), §3 Production parity (full mapping table covering all 7 Event fields), §4 Partitioning convention (xxHash64 seed=0, MVP shard_count=1 with multi-shard activation path).
- [x] Write the "Consumer hand-off: resink-core" section listing the supervisor-side constraints in plain language (sorted-on-load, non-empty event_id, SCD2-shaped before/after) — owner: teams/platform/data-engineering.
  - Landed as §5 of the contract, framed as expectations the in-memory source places on the supervisor (not specs the supervisor must invent). All three constraints named: (a) supervisor sorts by `event_ts` on load, (b) `event_id` non-empty + deterministic with the `<fact_stream>:<pk_col>=<pk_value>:<event_ts>` derivation pinned, (c) `before`/`after` carry the dim's full SCD2 column set including `valid_from`/`valid_to`/`is_current`. Section ends with the explicit "resink-core may push back; amendments land same-day as addendum-to-the-addendum" framing.
- [x] Cross-check field-by-field against `teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md` §7 (offset commit + idempotency) and §2 (partitioning) to confirm the parity claim is real, not aspirational — owner: teams/platform/data-engineering.
  - Re-read Kafka contract §1, §2, §7 while authoring the parity table. 5 of 7 fields map cleanly and identically (`key`, `before`, `after`, `event_ts`, `event_id`). 2 fields needed inline follow-up notes: (1) `table` vs Kafka topic name — the in-memory `table` is the destination dim while Kafka topics are named per fact stream; resolved by routing on `table` after Kafka decoding; (2) `op` field name — Kafka contract defers the value-schema field name to the schema registry rather than pinning `op`. Both follow-ups are schema-registry-resolvable without revising the Kafka contract body, so the "swap is mechanical" claim holds for the MVP's single-fact, single-dim path. Noted both inline rather than fudging.
- [x] Land the contract with `status: active`. Add a one-line Links-section back-reference to it from the Kafka ingress contract (no body edit) — owner: teams/platform/data-engineering.
  - Contract landed at `status: active`. Back-reference added as a single bullet at the end of the Kafka contract's Links section (no body edit, no rewrite of any existing bullet): `- MVP companion (in-memory parquet-replay path; field-for-field parity per its §3): [2026-05-16 in-memory event source](2026-05-16-in-memory-event-source.md)`.
- [ ] Mid-loop ping (by 2026-05-19): post the contract path to resink-core and ask explicit yes/no on KR1.4's three named constraints — owner: teams/platform/data-engineering.
  - Deferred to mid-loop (target 2026-05-19, per the OKR's stated date). The contract is landed and ready to ping; the explicit ask is documented in the contract's §5 framing. This task is a calendar-driven follow-up, not a build-turn deliverable.

## Cross-team asks

- **From `teams/application/resink-core`, by mid-loop (2026-05-19):** confirm the `Event` shape per [runtime spec §5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) works for the in-memory source impl powering [CEO brief](../../../../board/okrs/2026-05-16-ceo-brief.md) O1 KR1.4 — specifically: (a) is `event_ts`-sorted-on-load acceptable (vs supervisor sorting), (b) is `event_id` derived deterministically from `(table, key, valid_from)` in the fixture's `derive_facts.py` acceptable, (c) does the SCD2-shaped `before`/`after` row schema match what the LLM-generated `dim_user_scd2` node expects to receive. If any answer is no, DE files an addendum-to-the-addendum the same day; the contract stays `status: active` either way. Without this confirmation by 2026-05-19, the contract may need a same-loop revision.
- **From the board (already on the 2026-05-23 batch slot per the brief):** ADR-004 (or equivalent evolution) to add a `contract` type to `org-os/conventions.md` — reaffirmed from the [2026-05-10 exec summary](../exec-summaries/2026-05-10.md). Not blocking this loop; the new contract is filed as `type: rfc` per the existing workaround.

## Risks

- **Field-shape divergence with the Kafka contract surfaces during build.** Mitigation: the Production parity table is the explicit gate — if a one-to-one mapping cannot be written, that's the signal to either revise the in-memory shape (preferred) or file a Kafka-contract addendum (last resort, against the brief's "no revision required" guidance).
- **Resink-core needs a different Event shape than §5.4 verbatim.** Mitigation: KR1.4 surfaces the constraints explicitly by mid-loop. If resink-core pushes back, DE issues a same-day addendum rather than holding the contract draft.
- **Hash-function pinning surprises a future consumer.** xxHash64 seed=0 was already locked in the Kafka contract §2; the in-memory restatement only mirrors it. If a future stream library defaults to Murmur2 or xxHash3, the addendum needs an explicit "this is xxHash64, not the others" callout. Mitigation: the contract names the function and seed in one normative line.
- **`type: rfc` filing for a contract is still defensible but not principled.** Mitigation: ADR-004 on the 2026-05-23 batch slot fixes this institutionally; out-of-scope this loop, restated below.

## Out of scope this loop

- Revising the [Kafka ingress contract](../contracts/2026-05-10-kafka-ingress.md) body. The brief explicitly says no revision required; the only Kafka-contract edit this loop is a one-line back-reference in its Links section.
- Broader event-source patterns (webhook source, S3-poll source, CDC-from-snapshot source). The MVP only needs the in-memory parquet-replay shape; the contract names that and only that.
- DLQ / quarantine / late-event handling in the in-memory source. The MVP fixture is deterministic; no DLQ surface exists. Production DLQ surfaces remain owned by the Kafka contract §4.
- Throughput planning beyond the MVP fixture's three-events-or-so scale. Production-shape throughput is a sim-farm Mode-B/C concern in the loop after MVP.
- Multi-shard semantics in the in-memory source. MVP shard count is 1; multi-shard is a swap-time concern (Kafka contract §2 already covers it).
- ADR-004 (the `contract` type evolution proposal). Sits on the 2026-05-23 batch slot per the [CEO brief](../../../../board/okrs/2026-05-16-ceo-brief.md); reaffirmed from [last loop's exec summary](../exec-summaries/2026-05-10.md). DE does not author it this loop.
- Any work supporting paused teams (DevOps Helm work, SRE runbooks). Both teams resume the loop after MVP.

## Contract shape

Sketch of the contract this OKR commits to landing (the cross-team artifact downstream teams read first; full body lands in the build phase):

The `Event` record per [runtime spec §5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md):

```json
{
  "table": "dim_user",
  "key": {"user_id": 42},
  "op": "update",
  "before": {
    "user_id": 42,
    "email": "alice@example.com",
    "country": "US",
    "valid_from": 1715300000000,
    "valid_to": 1715400000000,
    "is_current": false
  },
  "after": {
    "user_id": 42,
    "email": "alice@example.com",
    "country": "CA",
    "valid_from": 1715400000000,
    "valid_to": null,
    "is_current": true
  },
  "event_ts": 1715400000000,
  "event_id": "fact_profile_update:user_id=42:1715400000000"
}
```

In-memory source semantics:
- **Order:** events sorted by `event_ts` ascending at load time. Ties broken by `event_id` lexicographic.
- **Partitioning:** `xxHash64(seed=0)` over the canonical UTF-8 bytes of the single primary-key value, modulo `shard_count`. MVP `shard_count = 1`.
- **Topic:** the parquet file path. One file per fact stream. No internal cross-shard topics in the in-memory source (single-shard MVP makes them unnecessary).
- **No replay, no DLQ, no quarantine.** The in-memory source is deterministic and finite; failure surfaces are sim-farm Mode-B/C concerns.

Production parity (preview — full table in the contract body): every field above maps 1:1 to the [Kafka ingress contract](../contracts/2026-05-10-kafka-ingress.md) §2 (partitioning), §7 (idempotency key includes `event_id`), and the schema-registry-defined message shape. The swap from in-memory to Kafka is "change the source plugin, keep every field."
