---
layout: default
title: "platform-data-engineering convention: dim-schema-json"
date: 2026-06-13
status: active
type: convention
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: convention
  owner: teams/platform/data-engineering
  date: 2026-06-13
  status: active
-->
{% raw %}

# Per-Dim Schema-JSON Convention — `{key_columns, payload_columns}`

**Scope:** every per-dim `schema.json` file produced by the resink-core
orchestrator (or any future consumer) and consumed by sim-farm's Mode-A diff
engine (`run_diff_multi`, engine version `0.3.0+`) via the `--schema-ref` CLI
flag (one `--schema-ref` per `--fixture` / `--output` / `--dim-table` triple,
paired by argument order).

## Context

Sim-farm's engine 0.3.0 (loop 2026-05-11-1631) introduced schema-aware join + payload
columns via a per-pair `schema.json` file, inlining the JSON shape into its
[verdict contract](../../../application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md)
under "Schema-aware columns (0.3.0+)" because DE was paused that loop. DE owns
this shape going forward, consistent with DE owning the
[Kafka ingress contract](../contracts/2026-05-10-kafka-ingress.md), the
[in-memory event-source contract](../contracts/2026-05-16-in-memory-event-source.md),
and the [DuckDB-pytz convention](./duckdb.md) — DE owns the data-shape
vocabulary; sim-farm owns the engine that consumes it. The shape became
canonical via sim-farm engine 0.3.0 at 2026-06-06 (GREEN end-to-end against
both `dim_user_schema.json` and `dim_account_schema.json` in resink-core's
`make mvp-loop`); this convention closes the in-loop request lifecycle filed
at [`requests/2026-06-06-001-schema-json-shape-ack.md`](../requests/2026-06-06-001-schema-json-shape-ack.md)
(open at 2026-06-06; accepted-deferred → fulfilled at 2026-06-13).

## Shape

A per-dim `schema.json` file is a single top-level JSON object with exactly
two required keys, each an array of column names (strings):

```json
{
  "key_columns":     ["string", ...],
  "payload_columns": ["string", ...]
}
```

| Field             | Type                | Required | Semantics                                                                                                                                                                                                              |
|-------------------|---------------------|----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `key_columns`     | array&lt;string&gt; | yes      | The SCD2 join-key tuple for the dim. **Length ≥ 1; ordering matters** (it's the join-key tuple — the first element is the natural row identity, subsequent elements compose into the SCD2 version key, e.g., `valid_from`). |
| `payload_columns` | array&lt;string&gt; | yes      | The non-key value columns participating in the row-equality check under `IS NOT DISTINCT FROM` (NULL-safe equality). **May be empty** (key-only diff; divergence is impossible by construction in that case).            |

**Column-name rules.** Each column name MUST be a SQL identifier safe for
DuckDB binding (alphanumeric + underscore; ASCII; not a DuckDB reserved word
when unquoted; case as it appears in the parquet schema). Names that fail
binding at engine query time surface as a DuckDB binding error and exit code 2
with `engine_error` in the verdict.

**Additivity discipline.** Additional top-level keys (e.g., a future
`tolerances`, `column_types`, `is_scd2`, `compaction_strategy`) are **ignored**
by engine 0.3.0+ parsers. A future minor revision of this convention may admit
such fields; until a consumer surfaces specific pressure, the shape stays
minimal (`{key_columns, payload_columns}` only). Field additions go through a
follow-up cross-team request to DE.

**Path placement convention.** Resink-core's orchestrator writes each per-dim
`schema.json` next to the dim's fixture and output parquets:
`<workspace>/<dim_table>_schema.json` (next to the supervisor output) and
`<fixtures>/<dim_table>_schema.json` (next to the fixture). The orchestrator
passes the path to the engine via `--schema-ref`.

## Worked examples

Two canonical examples, both written byte-stably by the resink-core
orchestrator at `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/sim-farm-schemas/`:

### `dim_user`

[`repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/sim-farm-schemas/dim_user.json`](../../../../repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/sim-farm-schemas/dim_user.json):

```json
{
  "key_columns":     ["user_id", "valid_from"],
  "payload_columns": ["email", "country", "valid_to", "is_current"]
}
```

The legacy hard-coded shape from engine `0.1.0` / `0.2.0` expressed
schema-aware. `user_id` is the natural row identity from `fact_user_signup`
(see [Kafka §2 primary-key rules](../contracts/2026-05-10-kafka-ingress.md));
`valid_from` is the SCD2 version key.

### `dim_account`

[`repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/sim-farm-schemas/dim_account.json`](../../../../repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/sim-farm-schemas/dim_account.json):

```json
{
  "key_columns":     ["account_id", "valid_from"],
  "payload_columns": ["account_type", "status", "valid_to", "is_current"]
}
```

The natural-column shape that the engine-0.2.0 isomorphic-column compromise
wrapped (loop 2026-05-11-1113). Engine 0.3.0+ reads this directly via
`--schema-ref`. `account_id` is the natural row identity from
`fact_account_open` (see [Kafka §9.2](../contracts/2026-05-10-kafka-ingress.md));
`valid_from` is the SCD2 version key.

## Consumers

- **sim-farm engine `0.3.0+`** — `repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py`. CLI flag `--schema-ref <path>` repeated per quadruple; library entrypoint `run_diff_multi(pairs, verdict_path)` accepts each `pair` as a 4-tuple `(fixture_path, output_path, dim_table, schema_ref_path)`. 3-tuples are still accepted for backward compatibility (`schema_ref_path` defaults to `None`, falling back to legacy hard-coded columns). See sim-farm verdict contract [§"Schema-aware columns (0.3.0+)"](../../../application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md) for the CLI argument-order pairing rules.
- **resink-core orchestrator** — writes one `<dim_table>_schema.json` next to each fixture and each supervisor-output parquet; passes the paths to the engine via `--schema-ref`. The orchestrator's per-pair schema generation is straightforward (the orchestrator already knows the column lists from the supervisor's DAG manifest), and the GREEN end-to-end run at 2026-06-06 against both `dim_user` and `dim_account` schemas demonstrates the shape suffices for the current MVP.

## Verified-against-environment

Per [ADR-2026-05-16-003](../../../../board/decisions/2026-05-16-003-contract-environment-verification.md),
this convention carries a Verified-against-environment subsection naming the
toolchain, runtime deps, OS surface, and failure modes. This is the **second
DE-owned contract/convention** to receive the discipline (after sim-farm's
2026-06-06 verdict-contract update — the first cross-team contract).

- **Verification date:** 2026-06-13 (loop 2026-05-11-2153).
- **Consumer under test:** sim-farm engine `0.3.0` (`repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py`) via `run_diff_multi` with per-pair `schema_ref_path`. Resink-core orchestrator confirmed GREEN end-to-end at 2026-06-06 against `closed_loop_v0/sim-farm-schemas/{dim_user,dim_account}.json` (verdict.json: `overall_pass: true`, `engine_version: "0.3.0"`).
- **Toolchain versions exercised:**
  - Python `>=3.11` (developer-laptop exercised against Python 3.12).
  - Schema-JSON parsing: Python `json` stdlib (no third-party dep) — `json.load(open(path))` returns the two-key dict; engine 0.3.0's loader extracts `key_columns` and `payload_columns` and raises `ValueError` on missing/empty `key_columns` or non-list values.
  - Rust-side parsing (future, if surfaced): `serde_json` (no schema file on the Rust side today; the orchestrator writes the file, and the engine reads it from Python).
- **Runtime dependencies (hard, not optional) for the consuming engine:**
  - `duckdb >= 1.0` — binds `key_columns` and `payload_columns` into the diff query.
  - `pyarrow >= 14.0.0` — fixture and output parquet loading.
  - `pytz >= 2024.1` — `TIMESTAMPTZ` materialization for tz-aware `valid_from` / `valid_to` columns (per [DE's duckdb convention](./duckdb.md); transitive lazy-import requirement of the DuckDB Python driver).
- **OS surface:** OS-agnostic. Schema-JSON is plain text; no OS-specific paths or syscalls in parsing. Exercised against macOS 24.6.0 arm64 (developer-laptop) and Linux x86_64 (CI baseline).
- **Auth-mode prerequisites:** none. Schema-JSON files are local artifacts; no network calls, no credentials. Operators need filesystem read access at the path passed via `--schema-ref` and filesystem write access (orchestrator side only).
- **Verification commands:**
  - **Parse-only sanity (no engine required):** `python3 -c "import json; d = json.load(open('<path>')); assert set(d) >= {'key_columns', 'payload_columns'}; assert isinstance(d['key_columns'], list) and len(d['key_columns']) >= 1; assert isinstance(d['payload_columns'], list)"`. Runs in milliseconds; depends only on the Python stdlib.
  - **End-to-end (consuming engine):** `cd repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0 && make mvp-loop` → produces `workspace/verdict.json` with `overall_pass: true`, `engine_version: "0.3.0"`, and per-dim entries for both `dim_user` and `dim_account` resolved via the per-pair `--schema-ref` paths.
- **Failure modes if prerequisites are not met:**
  - Missing schema-JSON file: `FileNotFoundError` at engine schema-ref-load — surfaces as exit code 2 with `engine_error` in the verdict.
  - Malformed schema-JSON (missing `key_columns` or `payload_columns`, non-list values, empty `key_columns`): `ValueError` at engine schema-ref-load — surfaces as exit code 2 with `engine_error`.
  - Column name in `key_columns` / `payload_columns` not present in the parquet: DuckDB raises a binding error at engine query execution — surfaces as exit code 2 with `engine_error`.
  - Column name not SQL-identifier-safe (special characters; reserved word unquoted): DuckDB binding error at engine query execution — surfaces as exit code 2 with `engine_error`. Fix: quote-safe the column name at fixture-generation time (the orchestrator and supervisor already do this for the MVP fixture).

## Links

- **Sim-farm verdict contract** (current canonical inline-spec source; gains a back-reference next loop replacing the inline shape with a link here): [`teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`](../../../application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md) — see "Schema-aware columns (0.3.0+)".
- **Originating cross-team request** (this loop's lifecycle close): [`teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md`](../requests/2026-06-06-001-schema-json-shape-ack.md).
- **ADR-2026-05-16-003** (Verified-against-environment discipline; this convention is the second adopter): [`board/decisions/2026-05-16-003-contract-environment-verification.md`](../../../../board/decisions/2026-05-16-003-contract-environment-verification.md).
- **Sister DE convention** (shape and depth reference): [`teams/platform/data-engineering/conventions/duckdb.md`](./duckdb.md).
- **Worked-example files (canonical on-disk artifacts):**
  - [`repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/sim-farm-schemas/dim_user.json`](../../../../repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/sim-farm-schemas/dim_user.json)
  - [`repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/sim-farm-schemas/dim_account.json`](../../../../repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/sim-farm-schemas/dim_account.json)
{% endraw %}
