---
layout: default
title: platform-data-engineering Exec Summary — 2026-05-11-0958
date: 2026-05-11
status: active
type: exec-summary
loop: 2026-05-11-0958
owner: teams/platform/data-engineering
grand_parent: Loops
parent: Loop 2026-05-11-0958
nav_order: 17
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/data-engineering
  date: 2026-05-11
  status: active
  loop: 2026-05-11-0958
  links: parent: teams/platform/data-engineering/okrs/2026-05-11-0958-team-okr.md
-->
{% raw %}

# Data Engineering Exec Summary — 2026-05-16

## What we shipped

### O1: In-memory event-source contract addendum the MVP supervisor consumes
- New addendum contract landed at [`../contracts/2026-05-16-in-memory-event-source.md`](../contracts/2026-05-16-in-memory-event-source.md) with `status: active`, filed as `type: rfc` per the still-open `contract` enum workaround (proposer = DE). Six sections: §1 `Event` record shape (pseudo-Rust + worked `dim_user` US→CA JSON example), §2 in-memory source semantics (event_ts ascending sort with `event_id` lexicographic tiebreak; no replay, no DLQ, no quarantine, file-path-is-topic, no watermark fan-out), §3 production parity (full 7-row mapping table to the Kafka ingress contract), §4 partitioning convention (xxHash64 seed=0, MVP `shard_count=1`), §5 consumer hand-off to resink-core (three named constraints), §6 out of scope. Grounded in [runtime spec §5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) and the Kafka ingress contract §2 + §7. (KR1.1, KR1.2, KR1.3, KR1.4)
- One-line back-reference added to the Kafka contract's Links section pointing at the addendum — pure cross-reference, no body edit, no rewrite of any existing bullet ([`../contracts/2026-05-10-kafka-ingress.md`](../contracts/2026-05-10-kafka-ingress.md), final Links bullet). (KR1.5)

## What we didn't ship and why

- **Mid-loop ping to `teams/application/resink-core` (target 2026-05-19) on the three §5 consumer-hand-off constraints — deferred as a calendar follow-up, not a build-turn deliverable.** The task is calendar-driven (mid-loop date sits inside the build window), so it was always going to outlast the build turn. In substance, resink-core's MVP build implicitly confirmed the field shape works: their supervisor consumed the contract's `Event` shape directly without surfacing a constraint mismatch and the closed-loop MVP closed green. A formal one-liner closure is still good housekeeping (see Asks).

## Surprises

- **Production parity table came in mostly clean — 5 of 7 fields (`key`, `before`, `after`, `event_ts`, `event_id`) map identically to the Kafka contract.** The mapping was tighter than expected; the field names landed equivalent on first pass without renaming.
- **Two fields needed honest follow-up notes rather than fudged claims.** `table` (in-memory targets the destination dim; Kafka topic name encodes the fact stream — resolved by routing on `table` after Kafka decoding) and `op` (Kafka contract defers the value-schema field name to the schema registry rather than pinning `op`). Both are schema-registry-resolvable without revising the Kafka contract body, so the "swap is mechanical" claim holds for the MVP's single-fact, single-dim path. We named them inline rather than pretending the parity was byte-identical everywhere.
- **Resink-core consumed the field shape directly without a single addendum needed — the contract held.** No addendum-to-the-addendum was filed; the supervisor's MVP build did not surface a constraint mismatch. A pleasant surprise relative to the "may need a same-loop revision" risk flagged in the OKR.
- **Light loop scope held end-to-end.** ADR-004 (`contract` type enum) remained out of scope as planned and stays on its existing 2026-05-23 batch slot. No scope drift mid-loop, no unplanned authorship pulled in.

## Asks

- **From the board:** reaffirm the ADR-004 batch slot at 2026-05-23 to add a `contract` type to `org-os/conventions.md`. Now triple-confirmed by sim-farm, DE, and AE all filing this loop's hand-off documents as `type: rfc` under the same workaround. No action needed before 2026-05-23; just keeping the slot warm.
- **From `teams/application/resink-core` (optional, housekeeping):** a one-line written closure on the three §5 named constraints (sorted-on-load by `event_ts`; deterministic non-empty `event_id`; SCD2-shaped `before`/`after` row schema). The green MVP build implicitly confirms all three, but a written one-liner makes the contract's `status: active` defensible without leaning on inference.

## Metrics

- **KR1.1** (in-memory event-source contract exists with `status: active`, filed as `type: rfc`, defines `Event` shape per runtime spec §5.4, defines in-memory source semantics — event_ts order, no replay, no DLQ, file-path-is-topic): **met** — [`../contracts/2026-05-16-in-memory-event-source.md`](../contracts/2026-05-16-in-memory-event-source.md) §1 (Event shape) + §2 (in-memory source semantics).
- **KR1.2** (Production parity section maps every in-memory `Event` field 1:1 to the Kafka ingress contract; swap is mechanical — no field renamed, dropped, or re-typed): **met** — §3 carries the full 7-row mapping table; 5 fields map identically, 2 (`table`, `op`) carry honest inline follow-up notes that resolve at the schema-registry layer without revising the Kafka contract body.
- **KR1.3** (per-fact partitioning restated: xxHash64 seed=0 over canonical UTF-8 PK bytes, modulo shard count; MVP `shard_count=1`): **met** — §4 restates the convention verbatim against Kafka contract §2 and pins the MVP shard count.
- **KR1.4** (named "Consumer hand-off: resink-core" section surfaces supervisor-side constraints; resink-core confirms or pushes back by mid-loop 2026-05-19): **partially met** — §5 names all three constraints (sorted-on-load by `event_ts`; deterministic non-empty `event_id` derived as `<fact_stream>:<pk_col>=<pk_value>:<event_ts>`; SCD2-shaped `before`/`after` row schema). Mid-loop formal confirmation deferred (see Asks); MVP build implicitly confirmed.
- **KR1.5** (contract cross-links Kafka contract, runtime spec §5.4, CEO brief O4; Kafka contract gets a one-line back-link with no body edit): **met** — addendum's Links section cites all three; Kafka contract's Links section gained one back-reference bullet, no body edit.
{% endraw %}
