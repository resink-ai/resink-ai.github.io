---
layout: default
title: "platform-data-engineering contract: 2026-05-16-in-memory-event-source"
date: 2026-05-16
status: active
type: contract
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: contract
  owner: teams/platform/data-engineering
  date: 2026-05-16
  status: active
  producers: teams/platform/data-engineering
  consumers: - teams/application/resink-core
  - teams/application/sim-farm
  links: parent: teams/platform/data-engineering/okrs/2026-05-16-team-okr.md
-->
# In-Memory Event Source — Nanofab Runtime (MVP Addendum)

This document defines the in-memory event-source contract that the nanofab
supervisor consumes during the 2026-05-16 MVP closed-loop test. Its single
purpose is to give resink-core's supervisor a named, byte-stable `Event` shape
that maps **field-for-field** onto the
[2026-05-10 Kafka ingress contract](2026-05-10-kafka-ingress.md) so that the
production swap from in-memory parquet replay to Kafka ingress is mechanical:
change the source plugin, keep every field name, type, and semantic.

This is a sibling contract to the Kafka ingress contract, not a replacement.
The Kafka contract remains the production surface; this addendum scopes the
laptop-only MVP fixture path described in
[CEO brief O1 KR1.4](../../../../board/okrs/2026-05-16-ceo-brief.md) and
[CEO brief O4](../../../../board/okrs/2026-05-16-ceo-brief.md). The
[runtime spec §5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md)
is the source of truth for the per-event shape; the Kafka ingress contract is
the source of truth for partitioning, idempotency, and producer-side framing.

**Format note:** Named hand-off sections per P3 of the
[2026-05-09 retro](../../../../board/retros/2026-05-09-ceo-retro.md). Every
normative section names the consumer that relies on it.

---

## 1. Event record shape

**Consumed by: resink-core supervisor (in-memory event source impl), sim-farm (closed-loop diff)**

The `Event` record shape is defined verbatim against
[runtime spec §5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md)
and the Kafka ingress contract §7 (`event_id` requirement). Pseudo-Rust /
TypeScript-flavored:

```rust
type Op = "insert" | "update" | "delete";

struct Event {
    table:    String,                       // dim-table name the event targets, e.g. "dim_user"
    key:      Map<String, Value>,           // {<pk_col>: <value>}, single entry for MVP
    op:       Op,                           // insert | update | delete
    before:   Option<Map<String, Value>>,   // pre-image row; None for op=insert
    after:    Option<Map<String, Value>>,   // post-image row; None for op=delete
    event_ts: i64,                          // unix epoch milliseconds (UTC), 13-digit
    event_id: String,                       // non-empty, deterministic; matches Kafka contract §7
}
```

Worked JSON example (mirrors the OKR's "Contract shape" sketch and the
fixture's `dim_user` schema):

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

Field rules:
- `table` is the destination dim-table name. The supervisor uses `table` to
  route the event to the appropriate node loaded from the manifest.
- `key` is a map with exactly **one** entry for the MVP (single-PK fixture).
  Composite keys are out of scope for this addendum; production composite-key
  handling is owned by the Kafka contract §2.
- `op` is one of `insert`, `update`, `delete`. The MVP fixture's
  `derive_facts.py` emits only `insert` (for `fact_sign_up`) and `update`
  (for `fact_profile_update`); `delete` is reserved for future fact streams.
- `before` is omitted (or `null`) for `op=insert`; `after` is omitted (or
  `null`) for `op=delete`; both are present for `op=update`.
- `event_ts` is unix milliseconds UTC, 13-digit. Aligns with the SCD2
  `valid_from` / `valid_to` columns in the same epoch-ms convention used
  throughout the runtime spec.
- `event_id` MUST be non-empty and deterministic. The fixture's
  `derive_facts.py` derives it as `<fact_stream>:<pk_col>=<pk_value>:<event_ts>`
  (worked example above). This satisfies the Kafka contract §7 requirement
  that "every ingress event MUST carry a non-empty `event_id` field" and is
  stable across re-runs of the seeded generator.

---

## 2. In-memory source semantics

**Consumed by: resink-core supervisor (event source impl)**

The in-memory event source is a deterministic, finite parquet replayer. It has
none of the operational surfaces a real Kafka cluster has — no consumer group,
no offset commit, no retention, no DLQ.

- **Order.** Events are sorted by `event_ts` ascending at load time. Ties
  (same `event_ts`) are broken by `event_id` lexicographic ascending. The
  supervisor sees a single linearized stream per `table`.
- **Partitioning.** `xxHash64(seed=0)` over the canonical UTF-8 byte
  representation of the **single primary-key value** in `key`, modulo
  `shard_count`. For the MVP `shard_count = 1` (single-shard fixture; see
  §4 for why and §5 for the multi-shard activation path). The hash function
  and seed are pinned to match Kafka contract §2 byte-for-byte so the swap is
  partition-stable.
- **Topic convention.** The parquet file path **is** the topic. One file per
  fact stream. For the MVP fixture this is exactly two files:
  `fixtures/fact_sign_up.parquet` and `fixtures/fact_profile_update.parquet`.
  No name-mangling, no broker indirection — the file path is the durable
  identifier.
- **No replay.** The in-memory source reads each parquet exactly once per
  supervisor invocation. There is no checkpoint, no offset, no resume. A
  re-run of `make mvp-loop` re-reads from the start.
- **No DLQ.** The in-memory source has no dead-letter surface. The MVP
  fixture is deterministic and finite; the supervisor either processes every
  event successfully or the supervisor exits non-zero. Production DLQ
  surfaces remain owned by the Kafka contract §4 and are not mirrored here.
- **No quarantine.** Same reasoning. The MVP fixture is generated by
  `derive_facts.py` from a known-good `dim_user_fixture.parquet`; schema
  mismatches at ingress are impossible by construction.
- **No watermark fan-out.** Watermarks are tracked locally by the supervisor
  per [runtime spec §5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md);
  there is no `nanofab.internal.watermarks` topic equivalent in the
  in-memory source because there is no second supervisor to fan out to.

---

## 3. Production parity

**Consumed by: resink-core supervisor (swap-time refactor), sim-farm (sim-vs-live equivalence)**

This is the gate that makes the in-memory → Kafka swap mechanical. Every
field of the in-memory `Event` record maps one-to-one to a location in the
[2026-05-10 Kafka ingress contract](2026-05-10-kafka-ingress.md). No field is
renamed, dropped, or re-typed at the swap.

| In-memory `Event` field | Kafka ingress contract location | Notes |
|---|---|---|
| `table`                 | §1 Topic naming convention — `nanofab.ingress.<tenant>.<fact_stream>` | The in-memory `table` corresponds to the destination **dim** the event maintains; the Kafka topic name encodes the **fact stream** that produces it. The mapping is `table` → the dim that the `<fact_stream>`-suffixed topic feeds. The training-pipeline manifest names this binding explicitly. (Follow-up: if a single fact stream feeds multiple dims, the swap-time code must route by `table` after Kafka decoding, not by topic name alone.) |
| `key`                   | §2 Partitioning — "Primary-key field declared per `<fact_stream>` in the schema-registry subject's `key` schema" | The single-entry `Map<pk_col, value>` in the in-memory `key` corresponds 1:1 to the Kafka message **key** (the schema-registry `<fact_stream>-key` subject's payload). MVP single-PK matches Kafka contract §2's "every ingress topic has exactly one primary-key field." |
| `op`                    | §1 + §7 (implicit, via the `<fact_stream>-value` schema-registry subject) | The Kafka message **value** schema includes the operation kind for CDC-shaped fact streams; the in-memory `op` field maps to that schema field. (Follow-up: the Kafka contract does not explicitly normalize this name to `op`; the schema-registry subject defines the field. The swap-time mapping pins `op` for cross-source consistency.) |
| `before`                | §1 + §7 (Kafka value schema) | Maps to the Kafka value schema's pre-image row. Same SCD2-shaped row schema as the destination dim. |
| `after`                 | §1 + §7 (Kafka value schema) | Maps to the Kafka value schema's post-image row. Same SCD2-shaped row schema as the destination dim. |
| `event_ts`              | §7 Offset commit + idempotency expectations — "Window decrement events are scheduled by event-time, not wall clock" + §2 watermark grounding | Maps to the Kafka message's event-timestamp field (the Kafka **header** or value-schema field carrying the producer-stamped event time, not the broker-assigned timestamp). Unix epoch milliseconds UTC in both. |
| `event_id`              | §7 — "Every ingress event MUST carry a non-empty `event_id` field" | Direct one-to-one mapping. Same field name, same non-empty constraint, same role as the idempotency-key tail in `(dag_version, node_id, event_id)`. The in-memory source pins the same determinism property the Kafka contract requires of producers. |

**Mapping verification.** Each row above was checked against the actual text
of the Kafka contract sections cited (re-read while writing this addendum).
Two follow-ups noted inline (the `table`-vs-topic name and the `op` field
name) are the only places where the parity is "consistent" rather than
"identical at the byte level"; both can be resolved by the Kafka schema
subject definition without revising the Kafka contract body. Neither
prevents the swap from being mechanical for the MVP's single-fact,
single-dim path.

---

## 4. Partitioning convention

**Consumed by: resink-core supervisor (in-memory hash impl), sim-farm (replay determinism)**

Restated from the Kafka ingress contract §2 verbatim, so the in-memory source
and the eventual Kafka source are byte-stable on the same key:

- **Hash function:** `xxHash64` with `seed=0`.
- **Hash input:** the canonical UTF-8 byte representation of the single
  primary-key field's value (the value side of the `Map<pk_col, value>` in
  `Event.key`).
- **Modulo:** `shard_count`.
- **MVP `shard_count`:** **1**.

Why `shard_count=1` for the MVP. The closed-loop test runs on a single
developer laptop with a single supervisor instance; there is no second
supervisor to fan to and no internal cross-shard topic to plumb (per §2).
Pinning `shard_count=1` makes the partition number deterministic (always 0)
without affecting hash byte-stability — the same `xxHash64(seed=0)` computed
on the same key value gives the same hash; only the modulo step differs.

How multi-shard activates downstream. When the production swap to Kafka
happens (or when an in-memory variant adds a second shard for sim-farm
Mode-B/C work), the consumer flips `shard_count` to the Kafka contract §2
value (multiple of 12, default 36). Because the hash function and seed are
identical, the partition assignment for any given key is preserved across
the swap up to the modulo. No re-keying of the in-flight events is required.

**Back-reference to Kafka contract §2.1 (added 2026-05-23):** the worked
examples and the UTF-8 decimal-string key-encoding rule are pinned in the
[Kafka ingress contract §2.1 (xxHash64 ratification)](2026-05-10-kafka-ingress.md).
The in-memory event source MUST encode the PK value identically (decimal-string
UTF-8 bytes) before hashing. This back-reference mirrors the 2026-05-16
pattern of restating Kafka contract semantics in §4 without forking them.

---

## 5. Consumer hand-off: resink-core

**Consumed by: resink-core supervisor (implementation expectations)**

The in-memory event source places the following constraints on the consuming
supervisor. These are expectations, not specs the supervisor must invent;
each is grounded in the Kafka contract or the runtime spec.

- **The supervisor sorts events by `event_ts` on load.** The in-memory
  source's stream-flavored input is a parquet file, not a partitioned Kafka
  topic that delivers in offset order. The supervisor MUST sort the rows by
  `event_ts` ascending (with `event_id` lexicographic as the tie-breaker)
  before feeding them through the DAG. This expectation is consistent with
  the OKR text: "expectation that the supervisor sorts by `event_ts` on load
  (because the in-memory source is a parquet file, not a stream)."
- **`event_id` is non-empty and deterministic.** The supervisor MUST treat
  `event_id` as required input and MAY assert non-emptiness defensively. The
  fixture's `derive_facts.py` derives `event_id` as
  `<fact_stream>:<pk_col>=<pk_value>:<event_ts>`, which is deterministic
  across reruns of the seeded generator and matches the Kafka contract §7
  idempotency requirement (same `event_id` → same idempotency key tail).
- **`before` / `after` row shapes match the dim's SCD2 column set.** Both
  rows MUST carry the full SCD2 column set for the `table` they target,
  including `valid_from` (epoch ms, non-null), `valid_to` (epoch ms or null),
  and `is_current` (boolean). The MVP fixture's `dim_user` columns are
  `{user_id, email, country, valid_from, valid_to, is_current}`; the
  generated `dim_user_scd2` node consumes events whose `before`/`after` rows
  match this shape exactly.

These three constraints are surfaced explicitly so that resink-core can
push back during the build phase. A mid-loop confirmation from
`teams/application/resink-core` (target: 2026-05-19) may amend any of them;
the contract stays `status: active` either way and any amendment lands as
an addendum-to-the-addendum the same day.

---

## 5.1 fact_account_open schema (additive 2026-05-23, cross-link)

**Consumed by: resink-core supervisor (in-memory event source impl for the
widened MVP fixture), sim-farm (closed-loop diff for `dim_account`)**

The `fact_account_open` fact stream is the second fact registered on the
widened MVP fixture introduced by
[CEO brief 2026-05-23 O1](../../../../board/okrs/2026-05-23-ceo-brief.md).
Its full schema (topic name, primary key, partitioning, event shape,
cross-dim referential note, production parity restatement) is pinned in
**[Kafka ingress contract §9 (fact_account_open schema)](2026-05-10-kafka-ingress.md)**.
This addendum does **not** re-state the schema — single source of truth, no
divergence. The pointer here exists so a reader who lands first in the
in-memory contract finds the new fact stream named alongside the existing
`fact_sign_up` and `fact_profile_update` paths.

**In-memory parquet-replay path for `fact_account_open`:**
- **Parquet file path:** `fixtures/fact_account_open.parquet` (per §2
  "topic convention — the parquet file path is the topic"; one file per
  fact stream).
- **Event shape:** matches the `Event` record shape pinned in §1 of this
  addendum; the `after` row schema is the SCD2 column set
  `(account_id, user_id, account_type, status, valid_from, valid_to,
  is_current)` per [Kafka contract §9.3](2026-05-10-kafka-ingress.md).
- **`op` distribution:** MVP fixture emits **`insert` only** (accounts open,
  v1 has no update/delete path); §1's `before`/`after` rule applies —
  `before` is `null`, `after` is the full SCD2 row.
- **`table` field value:** the string literal `"dim_account"` (routes to
  the `dim_account_scd2` node in resink-core's manifest).
- **`event_id` derivation:** `fact_account_open:account_id=<pk>:<event_ts>`,
  per §1's deterministic-and-non-empty rule and Kafka contract §7's
  idempotency-key requirement.
- **Partitioning:** `xxHash64(seed=0)` over the UTF-8 decimal-string bytes
  of the `account_id` value, modulo `shard_count`. For the MVP
  `shard_count=1`; the widened multi-shard fixture (CEO brief 2026-05-23
  O1 KR1.4) uses `shard_count=4`. The worked example in
  [Kafka contract §2.1 example B](2026-05-10-kafka-ingress.md) applies
  verbatim (`account_id=42` → `xxh64=0x6de6f5d076d742b9`).

**Cross-dim referential note:** the `after.user_id` field references
`dim_user.user_id`. DE does not own the join in this contract; the
reference is named so resink-core's `dim_account_scd2` node knows the FK
column without inferring from the parquet. Same wording as Kafka contract
§9.4; consistency, not duplication.

The mapping table in §3 of this addendum (in-memory `Event` ↔ Kafka
contract location) applies to `fact_account_open` unchanged — every field
maps to the same Kafka contract location for `fact_account_open` as it
does for the existing fact streams. No new rows are needed.

---

## 6. Out of scope for this addendum

- **Real Kafka.** The Kafka ingress contract is the production source of
  truth; this addendum is the laptop-only MVP companion.
- **Multi-shard semantics.** MVP is single-shard; multi-shard is a
  swap-time concern already covered by Kafka contract §2.
- **DLQ / quarantine / late-event handling.** The in-memory source is
  deterministic and finite; failure surfaces remain owned by the Kafka
  contract §4 and the runtime spec §7.4–§7.6.
- **Webhook source, S3-poll source, CDC-from-snapshot source.** Other
  event-source patterns are out of scope; the MVP only needs in-memory
  parquet replay.
- **Throughput planning.** The MVP fixture is three-events-or-so scale;
  production-shape throughput is a sim-farm Mode-B/C concern.

---

## Links

- Production source-of-truth contract (parity gate): [Kafka ingress contract](2026-05-10-kafka-ingress.md)
- Event-shape source of truth: [runtime spec §5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md)
- Driving CEO brief (O4 — this addendum; O1 KR1.4 — the consumer): [2026-05-16 CEO brief](../../../../board/okrs/2026-05-16-ceo-brief.md)
- DE team OKR for this loop: [2026-05-16 team OKR](../okrs/2026-05-16-team-okr.md)
- Runtime adoption ADR: [2026-05-10-001 nanofab runtime is Rust](../../../../board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md)
- Format-pattern source: P3 in [2026-05-09 CEO retro](../../../../board/retros/2026-05-09-ceo-retro.md)
- Consumers this loop:
  - `teams/application/resink-core/` — supervisor + in-memory source impl
  - `teams/application/sim-farm/` — closed-loop diff (consumes the supervisor output, not the events directly, but relies on event-shape determinism)
