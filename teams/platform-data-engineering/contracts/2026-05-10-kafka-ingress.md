---
layout: default
title: "platform-data-engineering contract: 2026-05-10-kafka-ingress"
date: 2026-05-10
status: active
type: contract
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: contract
  owner: teams/platform/data-engineering
  date: 2026-05-10
  status: active
  producers: teams/platform/data-engineering
  consumers: - teams/application/resink-core
  - teams/application/sim-farm
  - teams/platform/devops
  - teams/platform/sre
-->
# Kafka Ingress Contract — Nanofab Runtime

**Format note:** Named hand-off sections per P3 of the [2026-05-09 retro](../../../../board/retros/2026-05-10-2227-001-ceo-retro.md). Every normative section is labeled with the consumer that relies on it. If two consumers diverge on a single section, that is the signal to split it — until then, one document.

**Scope:** the Kafka surface between the customer's fact streams and the nanofab runtime supervisor fleet, plus the runtime-internal cross-shard Kafka surface. Grounded in [runtime spec §3 (architecture overview)](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) and [§7 (failure handling, idempotency, ordering)](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md). Out of scope: KV-layer contract, plugin-ABI contract, coordinator protocol — these are resink-core's surfaces under the runtime spec.

**Naming convention used below:** `<TENANT>` and `<DAG_VERSION>` are placeholders; no specific customer is named anywhere in this document.

---

## 1. Topic naming convention

**Consumed by: resink-core supervisor, sim-farm `--mode=sim`**

| Topic class | Pattern | Example |
|---|---|---|
| Ingress (per fact stream) | `nanofab.ingress.<tenant>.<fact_stream>` | `nanofab.ingress.<TENANT>.fact_post_publish` |
| Cross-shard internal | `nanofab.internal.<tenant>.<dag_version>.<edge>` | `nanofab.internal.<TENANT>.v17.dim_post_writes` |
| Watermark fan-out | `nanofab.internal.<tenant>.<dag_version>.watermarks` | `nanofab.internal.<TENANT>.v17.watermarks` |
| Per-node dead-letter | `nanofab.dlq.<tenant>.<dag_version>.<node_id>` | `nanofab.dlq.<TENANT>.v17.extract_dim_post` |
| Source-schema quarantine | `nanofab.quarantine.<tenant>.<fact_stream>` | `nanofab.quarantine.<TENANT>.fact_post_publish` |

Rules:
- Lowercase, dot-separated, no underscores in the dotted segments other than `dag_version` literal `v<N>` (where `<N>` is the integer DAG version from the coordinator manifest).
- `<tenant>` is the tenant identifier as issued by the coordinator at provisioning time — opaque to the runtime. It must appear in every topic name; no tenant-less topics exist on the cluster.
- `<fact_stream>` matches the schema-registry subject base name 1:1, so `nanofab.ingress.<TENANT>.<fact_stream>` ↔ `<fact_stream>-value` subject.
- The `nanofab.internal.*` namespace is owned by the supervisor fleet; no customer process produces to it.

Grounded in runtime spec §4.5 ("Transport Layer") and §5.3 ("Cross-shard fan-out").

---

## 2. Partitioning

**Consumed by: resink-core supervisor (shard ownership), sim-farm (replay determinism)**

- **Hash function:** xxHash64 (seed = 0) over the canonical UTF-8 byte representation of the primary-key field, modulo partition count. xxHash64 is chosen over the default Kafka Murmur2 because xxHash64 is the same hash used by the nanofab state layer for KV sharding (see runtime spec §3.1 "data plane") — using one hash everywhere preserves the §3.2 invariant that "state lives in external KV, partitioned by the same key as the Kafka ingress, so a supervisor reads its own shard's KV without cross-shard hops in the hot path."
- **Primary-key field:** declared per `<fact_stream>` in the schema-registry subject's `key` schema. Examples: `fact_post_publish` → `user_id`; `fact_user_signup` → `user_id`; `fact_payment_event` → `account_id`. Every ingress topic has exactly one primary-key field; composite keys are concatenated by the producer before hashing.
- **Partition count:** chosen at provisioning time per tenant; **multiple of 12** so that re-sharding to 12/24/36/48 supervisor pods on the standard pod sizes does not produce skew. Default = 36.
- **Cross-shard internal topics:** partitioned by the *re-key* field (the destination shard's primary key), not the upstream key. Same hash, same partition count as the destination dim's owning ingress topic.

Grounded in runtime spec §2 ("Sharding: Hash by primary key") and §5.3.

### 2.1 xxHash64 ratification — worked examples (Ratification 2026-05-23)

**Consumed by: resink-core supervisor `partition()` impl (byte-stable citation), sim-farm replay**

This subsection ratifies the hash semantics already pinned in §2 by adding two
concrete, byte-stable worked examples. Resink-core's `partition()` implementation
in the supervisor cites this subsection directly; if the `twox-hash::xxh64::xxh64`
Rust crate output diverges from the values below, this subsection is the
authoritative entry point for a one-day correction (per the 2026-05-23 team OKR
risk row). The hash function, seed, and key-encoding rule are not negotiable —
only the example hex values are eligible for correction.

**Pinned semantics (restated from §2):**
- **Hash function:** `xxHash64`.
- **Seed:** `0`.
- **Key encoding:** the canonical UTF-8 byte representation of the **decimal-string
  form** of the primary-key value (e.g., integer `42` encodes as the two-byte
  ASCII string `"42"` → `0x34 0x32`, **not** the eight-byte little-endian `i64`
  binary form). This matches `derive_facts.py`'s key derivation for the MVP
  fixture and the in-memory event-source contract §4. Producer and consumer
  agree byte-for-byte by stringifying the PK before hashing.
- **Reference Rust impl:** `twox-hash::xxh64::xxh64(bytes, 0)`.
- **Shard count:** `4` for the multi-shard widened fixture (board CEO brief
  2026-05-23 O1 KR1.4); `1` for the single-shard MVP path (in-memory contract
  §4); production default remains `36` per §2.

**Worked example A — `user_id="u-001"` (string PK, fact_user_signup-flavored):**

```
key value (UTF-8 string)  : "u-001"
key_bytes (hex)           : 0x75 0x2d 0x30 0x30 0x31
key_bytes (length)        : 5
xxh64(key_bytes, seed=0)  : 0x571e3e04781b0ff5   (decimal 6277523119516880885)
partition (shard_count=4) : 6277523119516880885 % 4 = 1
partition (shard_count=1) : 6277523119516880885 % 1 = 0
```

**Worked example B — `account_id=42` (integer PK stringified, fact_account_open):**

```
key value (Python repr)   : 42  (integer)
canonical encoding        : str(42) → "42"
key_bytes (hex)            : 0x34 0x32
key_bytes (length)         : 2
xxh64(key_bytes, seed=0)  : 0x6de6f5d076d742b9   (decimal 7919287270473417401)
partition (shard_count=4) : 7919287270473417401 % 4 = 1
partition (shard_count=1) : 7919287270473417401 % 1 = 0
```

Both values above were computed against the Python `xxhash` package
(`xxhash.xxh64(key_bytes, seed=0).intdigest()`) at ratification time
(2026-05-23). The expected equivalence is byte-for-byte with
`twox-hash::xxh64::xxh64`; tagged as
`<verified-against-rust-impl: verified-2026-06-13-by-resink-core — Example A
(`user_id="u-001"` → `0x571e3e04781b0ff5`) confirmed byte-stable by
resink-core's supervisor `partition()` impl against `twox-hash::xxh64::xxh64`
at 2026-05-23; tag flipped 2026-06-13 as part of DE's schema-JSON
canonicalization loop>`. If a future Rust observation diverges, the hex value
gets a one-line correction here (and the team OKR's risk row triggers).

**Why the decimal-string encoding rule is pinned, not the i64 binary form.**
The MVP fixture path uses Arrow-typed integer PKs in parquet but stringifies
them for the topic message **key** so that the same hash semantics work for
mixed PK types across fact streams (`fact_user_signup` uses string `user_id`;
`fact_account_open` uses integer `account_id`; both go through the same
producer-side hasher). Pinning the rule avoids ambiguity between Python's
`int.to_bytes(8, "little")` and `str(int).encode("utf-8")` — the contract picks
the latter as the only one that is the same across producer SDKs.

Grounded in: §2 (hash function + seed + modulo), in-memory contract §4
(byte-stable swap requirement), team OKR 2026-05-23 KR1.2 (the ratification
requirement), CEO brief 2026-05-23 O1 KR1.4 (the consumer of this ratification).

---

## 3. Consumer-group naming

**Consumed by: resink-core supervisor (group membership), sim-farm (parallel sim runs without group collisions)**

| Group class | Pattern | Lifetime |
|---|---|---|
| Live supervisor fleet | `nanofab.sup.<tenant>.<dag_version>` | DAG-version-lifetime |
| Blue/green candidate | `nanofab.sup.<tenant>.<dag_version>.cand` | candidate-warmup → cutover |
| Internal cross-shard consumer | `nanofab.internal.<tenant>.<dag_version>.<edge>.consumer` | DAG-version-lifetime |
| Sim Farm replay | `nanofab.sim.<tenant>.<sim_run_id>` | sim-run-lifetime |
| DLQ inspector (ops) | `nanofab.dlq.<tenant>.<dag_version>.<node_id>.inspector` | manual |

Rules:
- The supervisor fleet for DAG `vN` is exactly one group; a new DAG version means a new group name (per runtime spec §5.7 "Replay" — blue/green requires a fresh group). The group is **never reused** across DAG versions.
- Sim Farm groups are scoped per `sim_run_id` so that parallel sim runs against the same ingress topic do not steal each other's offsets. Sim-mode supervisors (`--mode=sim`) typically replace Kafka with an in-memory event source per spec §8.4, but for "live shadow with real ingress" they use this group pattern.
- DLQ inspector groups exist for human ops; supervisors do not consume DLQ topics.

Grounded in runtime spec §5.7 ("Replay") and §6.3 ("DAG-level blue/green").

---

## 4. Dead-letter / quarantine topics

**Consumed by: resink-core supervisor (DLQ producer), sim-farm (DLQ verdicts), SRE (alerting)**

Three distinct failure surfaces, three distinct topic families:

- **`nanofab.dlq.<tenant>.<dag_version>.<node_id>`** — per-node plugin panic surface (§7.4). Producer: the supervisor that caught the panic via `std::panic::catch_unwind`. Payload: the original event + panic context (node version, panic message, stack pointer, trace_id). Trigger: any `process()` panic. Repeated panics on the same `event_id` trigger an automatic per-node hot-swap to last-known-good (per §7.4).
- **`nanofab.quarantine.<tenant>.<fact_stream>`** — source-schema-mismatch surface (§7.6) and beyond-allowed-lateness surface (§7.5). Producer: the supervisor's ingress decoder, before the event enters the DAG. Payload: raw bytes + decode error or `late_by_ms`. Triggers: schema-registry validation fails on a required field; or `event_ts < watermark - allowed_lateness`.
- **No global DLQ.** The runtime intentionally has no single firehose DLQ — diagnosis depends on the failure class and per-node retry semantics differ.

Retention: DLQ topics default to 14 days; quarantine topics default to 30 days. Retention is a per-tenant config; the contract enforces only minimums.

Grounded in runtime spec §7.4, §7.5, §7.6.

---

## 5. Per-tenant SASL format

**Consumed by: resink-core supervisor (broker auth), sim-farm (sim-mode skips SASL), SRE (credential rotation)**

- **Mechanism:** SASL/SCRAM-SHA-512 over TLS. Plaintext SASL/PLAIN is disallowed on the cluster.
- **Username format:** `<role>.<tenant>` where `<role>` is one of:
  - `sup` — supervisor fleet (read ingress, read/write internal, write DLQ).
  - `prod` — customer-side producer for ingress topics (write ingress only).
  - `sim` — sim-farm runner (read ingress with group prefix `nanofab.sim.*`, write nothing on the live cluster).
  - `ops` — DLQ inspector (read DLQ + quarantine, write nothing).
- **ACLs (per role):**
  - `sup.<tenant>`: READ on `nanofab.ingress.<tenant>.*`; READ/WRITE on `nanofab.internal.<tenant>.*`; WRITE on `nanofab.dlq.<tenant>.*` and `nanofab.quarantine.<tenant>.*`. CREATE on `nanofab.internal.<tenant>.*` (cross-shard topics are auto-created at DAG-version load).
  - `prod.<tenant>`: WRITE on `nanofab.ingress.<tenant>.*` only.
  - `sim.<tenant>`: READ on `nanofab.ingress.<tenant>.*` only; group-prefix restricted to `nanofab.sim.<tenant>.*`.
  - `ops.<tenant>`: READ on `nanofab.dlq.<tenant>.*` and `nanofab.quarantine.<tenant>.*`.
- **No cross-tenant ACLs.** A credential is scoped to exactly one tenant. The cluster enforces this via the principal prefix — there is no superuser principal that can read more than one tenant's ingress.
- **Rotation:** SCRAM credentials rotate quarterly. The supervisor reloads credentials on SIGHUP; no restart is required for rotation.

This section is the per-tenant isolation hinge — it is the only place where tenant separation is enforced at the transport layer. Subsequent contracts (KV-layer, plugin-ABI) may rely on this guarantee without restating it.

---

## 6. Broker bootstrap shape

**Consumed by: resink-core supervisor (startup), sim-farm (sim-mode bypass), serving/deployment (k8s config)**

- **Bootstrap servers:** supplied via the supervisor's coordinator-fetched config under `transport.kafka.bootstrap`. The supervisor does not parse a `bootstrap.servers` env var directly; the coordinator is the source of truth so that broker fleet changes propagate without rolling supervisors.
- **Shape:** a list of `host:port` entries, minimum length 3, all reachable from the supervisor pod's network. The supervisor refuses to start with fewer than 3 entries (forces operator to think about HA at config time).
- **TLS:** required. Server-cert verification on; client-cert optional (SASL is the primary auth, mTLS is defense-in-depth for the same-VPC case).
- **Client ID:** `<role>.<tenant>.<supervisor_pod_id>` — used for broker-side observability, not for auth. The `<supervisor_pod_id>` is the k8s pod name or equivalent.
- **Compression:** ingress producers (customer side) use `zstd` (level 3); the supervisor accepts any compression and emits `zstd` on internal and DLQ topics.
- **Idempotent producer:** the supervisor's internal-topic producer runs with `enable.idempotence=true`, `acks=all`, `max.in.flight.requests.per.connection=5`. Customer-side ingress producers MUST also set `acks=all`; idempotence is recommended but not required (the supervisor's per-event idempotency layer in §7 backstops customer-side retries).

Grounded in runtime spec §4.5 ("Transport Layer") and §8.4 ("Supervisor hooks for Sim Farm").

---

## 7. Offset commit + idempotency expectations

**Consumed by: sim-farm `--mode=sim` (equivalence to live behavior), resink-core supervisor (the actual implementation)**

This is the section sim-farm relies on most heavily for sim-vs-live equivalence (per runtime spec §8.2 "Whole-DAG simulation in the Sim Farm").

- **At-least-once delivery, not exactly-once.** The supervisor's contract with downstream is at-least-once + idempotent writes (runtime spec §7.1). True exactly-once is rejected by §2 ("avoids the throughput tax of true exactly-once") — sim-farm MUST model at-least-once to match production.
- **Offset commits are manual.** Auto-commit is disabled. The supervisor commits an offset only after the **terminal node** of every DAG branch reached by that event has successfully written to KV (and to any cross-shard internal topic with `acks=all`). Per spec §7.2: "no Kafka offset is committed until the terminal node of a DAG branch successfully writes KV."
- **Commit batching.** Offsets are committed in batches per Kafka poll cycle (default ~100ms or every 1,000 events, whichever first). The batched commit is atomic at the consumer-group level — partial commits do not occur.
- **Idempotency key.** `(dag_version, node_id, event_id)` — the same key used by the per-node dedup table in KV (§7.1). This composite key is what makes blue/green safe under §6.3: vN and vN+1 can each process the same `event_id` because they hold distinct `dag_version` prefixes. Sim-mode MUST stamp these three fields on every event for diff equivalence.
- **Dedup TTL.** 7 days, configurable. Sim-farm runs that span more than 7 sim-days MUST extend the TTL or risk replaying a duplicate as fresh.
- **Cross-shard internal topics carry the same idempotency key.** The destination supervisor (on the post-rekey shard) MUST dedup using the same `(dag_version, node_id, event_id)` it would have used for the original ingress event — `node_id` here is the **producing** node, not the consuming one.
- **`event_id` requirement.** Every ingress event MUST carry a non-empty `event_id` field. Producers without a natural event ID MUST synthesize one (e.g., hash of source-system primary key + source-system commit timestamp). Events missing `event_id` route to `nanofab.quarantine.<tenant>.<fact_stream>` and never enter the DAG.

The single normative restatement for sim-farm: **a sim-mode supervisor that replays the same event stream MUST produce byte-identical state mutations to a live supervisor**, because the idempotency surface is fully deterministic on `(dag_version, node_id, event_id)`. This is the property that makes §8.2's row-for-row equivalence diff well-defined.

Grounded in runtime spec §7.1, §7.2, §8.2, §8.4.

---

## 8. Cross-cutting: tenant-isolation invariant

**Consumed by: every consumer of this contract**

Every topic name, every consumer-group name, every SASL principal in this contract contains `<tenant>`. The cluster MUST refuse a topic create, a group join, or a producer connect that violates this — there is no path by which a supervisor with `sup.<TENANT_A>` credentials reads `nanofab.ingress.<TENANT_B>.*`. This is the §5 ACL guarantee restated as an invariant for the rest of the contract; later sections of any downstream contract may rely on it without re-deriving.

This is the only operational mechanism that prevents cross-tenant data leakage at the ingress; it is non-negotiable.

---

## 9. fact_account_open schema (additive 2026-05-23)

**Consumed by: resink-core supervisor (DAG node `dim_account_scd2`), sim-farm
(closed-loop diff for `dim_account`), orchestrator `generate.py` (downstream
schema_json payload), in-memory event-source contract §1 (sibling restatement)**

The `fact_account_open` event stream is the second fact stream registered on the
nanofab runtime alongside the existing `fact_post_publish` / `fact_user_signup`
streams named in §2. It feeds the new `dim_account_scd2` node introduced by
[CEO brief 2026-05-23 O1 KR1.1](../../../../board/okrs/2026-05-11-1113-ceo-brief.md).
This section is **additive** to §1–§8: every normative rule there (topic naming,
partitioning hash, consumer-group naming, DLQ shape, SASL, broker bootstrap,
offset/idempotency, tenant-isolation invariant) applies to `fact_account_open`
without restatement. Only the fact-stream-specific fields below are new.

### 9.1 Topic name

Per §1 topic-naming convention:

| Topic class | Concrete name |
|---|---|
| Ingress | `nanofab.ingress.<tenant>.fact_account_open` |
| Quarantine | `nanofab.quarantine.<tenant>.fact_account_open` |

The schema-registry subject base name is `fact_account_open`; the value subject
is `fact_account_open-value`, the key subject is `fact_account_open-key`.

### 9.2 Primary key and partitioning

- **Primary-key field:** `account_id` (single PK, per §2 "every ingress topic
  has exactly one primary-key field").
- **Partitioning:** `xxHash64(account_id_str_bytes, seed=0) mod partition_count`,
  using the decimal-string encoding rule pinned in §2.1. With
  `account_id=42`, the worked example in §2.1 example B applies verbatim.
- This restates §2 production parity: the fact_account_open stream uses the
  same hash function, the same seed, and the same key-encoding rule as every
  other ingress topic on the cluster.

### 9.3 Event shape

The `fact_account_open` value schema is CDC-shaped, matching the in-memory
event-source contract's `Event` record so the in-memory → Kafka swap is
mechanical (per [in-memory contract §3](2026-05-16-in-memory-event-source.md)).
The fixture emits **`op="insert"` only** for the MVP (accounts open; they do
not update or delete in the v1 widened fixture). `op="update"` and
`op="delete"` are reserved for future fact streams against `dim_account`.

```rust
struct FactAccountOpenEvent {
    table:    "dim_account",                 // always; routes to dim_account_scd2 node
    op:       "insert",                      // MVP fixture emits insert only
    key:      { account_id: i64 },           // single-PK map
    before:   None,                          // always None for op=insert
    after:    {                              // SCD2-shaped row matching dim_account_fixture
        account_id:   i64,
        user_id:      String,                // FK reference to dim_user.user_id (see §9.4)
        account_type: String,                // e.g., "checking" | "savings" | "credit"
        status:       String,                // e.g., "active" | "closed"
        valid_from:   i64,                   // unix epoch ms UTC, 13-digit
        valid_to:     Option<i64>,           // null for is_current=true rows
        is_current:   bool,
    },
    event_ts: i64,                           // unix epoch ms UTC, 13-digit
    event_id: String,                        // non-empty deterministic, per §7
}
```

Worked JSON example (matches the in-memory contract §1 worked-example shape):

```json
{
  "table": "dim_account",
  "key": {"account_id": 42},
  "op": "insert",
  "before": null,
  "after": {
    "account_id": 42,
    "user_id": "u-001",
    "account_type": "checking",
    "status": "active",
    "valid_from": 1716422400000,
    "valid_to": null,
    "is_current": true
  },
  "event_ts": 1716422400000,
  "event_id": "fact_account_open:account_id=42:1716422400000"
}
```

`event_id` derivation follows the in-memory contract §1 rule:
`<fact_stream>:<pk_col>=<pk_value>:<event_ts>`. Determinism + non-emptiness +
§7 idempotency-key tail apply unchanged.

### 9.4 Cross-dim referential note (`account.user_id → dim_user.user_id`)

The `after.user_id` field is a foreign-key reference into `dim_user.user_id`.
This contract does **not** own the join — DE owns the event-source surface
only; resink-core's `dim_account_scd2` node and sim-farm's diff engine
materialize the relationship. The reference is named here so the contract is
self-documenting: any consumer reading this section knows that
`fact_account_open.after.user_id` is the join column without inferring from
the parquet or schema-registry payload. Referential integrity is the
fixture-generator's responsibility (`derive_facts.py` emits only
`account.user_id` values that also exist in `dim_user_fixture`).

### 9.5 Production parity (restated against §2 and §4)

This subsection restates the §2 partitioning rule and the §4 DLQ/quarantine
rule against `fact_account_open` so a consumer reading §9 in isolation has
the full normative surface.

- **Partitioning (§2 + §2.1):** `xxHash64(seed=0)` over the UTF-8 decimal-string
  bytes of `account_id`, modulo `partition_count`. `partition_count` follows
  §2's "multiple of 12, default 36" rule; the multi-shard widened fixture
  (CEO brief 2026-05-23 O1 KR1.4) uses `shard_count=4`, and the MVP
  single-shard fixture uses `shard_count=1` (in-memory contract §4). All
  three values produce the same hash; only the modulo differs.
- **DLQ (§4):** `nanofab.dlq.<tenant>.<dag_version>.dim_account_scd2`. Producer:
  the supervisor that catches a `dim_account_scd2` node panic. Trigger:
  `process()` panic, per §4 bullet 1. No fact-stream-specific DLQ semantics.
- **Quarantine (§4):** `nanofab.quarantine.<tenant>.fact_account_open`.
  Producer: the supervisor's ingress decoder. Triggers: schema-registry
  validation fails on a required `after.*` field; or `event_ts < watermark -
  allowed_lateness`. No fact-stream-specific quarantine semantics.
- **At-least-once + idempotency (§7):** `event_id` is the idempotency-key
  tail; offset commits are manual, batched, post-terminal-write. No
  fact-stream-specific override.

### 9.6 Cross-link to in-memory event-source contract

The [in-memory event-source contract](2026-05-16-in-memory-event-source.md)
carries a sibling section that cross-links here for the in-memory parquet-replay
path. The two sections together pin the `fact_account_open` schema across the
in-memory → Kafka swap surface.

Grounded in: [CEO brief 2026-05-23 O1 KR1.1 + KR1.6](../../../../board/okrs/2026-05-11-1113-ceo-brief.md),
[DE team OKR 2026-05-23 KR1.1](../okrs/2026-05-11-1113-team-okr.md), runtime spec §5.4.

---

## Links

- Runtime spec (primary ground): [docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) (§3, §4.5, §5.3, §5.7, §6.3, §7, §8)
- Adopting ADR: [board/decisions/2026-05-10-001-nanofab-runtime-is-rust](../../../../board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md)
- Superseded contract precedent (Spark-flavored, never landed): the deferred KR2.1 from [DE 2026-05-09 team OKR](../okrs/2026-05-10-2227-001-team-okr.md)
- Format-pattern source: P3 in [2026-05-09 CEO retro](../../../../board/retros/2026-05-10-2227-001-ceo-retro.md)
- Consumers this loop:
  - `teams/application/resink-core/` — supervisor fleet implementation
  - `teams/application/sim-farm/` — `--mode=sim` equivalence harness
- MVP companion (in-memory parquet-replay path; field-for-field parity per its §3): [2026-05-16 in-memory event source](2026-05-16-in-memory-event-source.md)
