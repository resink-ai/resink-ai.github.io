---
layout: default
title: "application-sim-farm contract: 2026-05-16-mvp-loop-verdict"
date: 2026-05-16
status: active
type: contract
owner: teams/application/sim-farm
grand_parent: Teams
parent: "Team: application-sim-farm"
---

<!-- original-frontmatter:
  type: contract
  owner: teams/application/sim-farm
  date: 2026-05-16
  status: active
  producers: teams/application/sim-farm
  consumers: - teams/application/resink-core
  links: parent: teams/application/sim-farm/okrs/2026-05-11-0958-team-okr.md
-->
{% raw %}

# MVP Loop Verdict Contract — Mode A (Batch SCD2 Diff)

## Version history

| Engine version | Loop      | Change                                                                                                                                                                                                                                                |
|----------------|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `0.1.0`        | 2026-05-16 | Initial Mode-A single-dim verdict. Top-level fields: `verdict_id`, `mode`, `dim_table`, `fixture_path`, `output_path`, `pass`, `mismatch_count`, `mismatches[]`, `ran_at`, `engine_version`. `Mismatch.type` enum: `missing` / `extra` / `diverged`. |
| `0.2.0`        | 2026-05-23 | Additive: multi-dim invocation emits a new top-level shape with `overall_pass` and `verdicts[]` (one entry per `(fixture, output, dim_table)` triple). Single-dim invocation is preserved byte-identically except `engine_version` bumps to `0.2.0`. |
| `0.3.0`        | 2026-06-06 | Additive: schema-aware join + payload columns via `--schema-ref <path>` per quadruple (one `--schema-ref` per `--fixture` / `--output` / `--dim-table` triple, paired by argument order). Engine reads `{key_columns, payload_columns}` from a per-pair schema-JSON file; omitted `--schema-ref` falls back to legacy hard-coded `("user_id", "valid_from")` key + `("email", "country", "valid_to", "is_current")` payload. The on-disk verdict shape is unchanged from 0.2.0 — only the engine invocation signature gained the optional `--schema-ref` flag and the library entrypoint `run_diff_multi` accepts an optional 4-tuple `(fixture, output, dim_table, schema_ref)` (3-tuples still accepted; schema_ref defaults to None). |

Semver discipline: additive top-level fields and additive `Mismatch.type` enum extensions are 0.x-bump-permitted (see "Future modes" below). Renaming, retyping, or removing any existing field requires a major bump (0.x.y → 1.0.0) and a contract revision.

## Purpose

Defines the verdict JSON shape produced by the MVP closed-loop **Mode-A** diff engine (`repos/resink-ai/resink-core/sim-farm/diff_scd2.py`). The engine is the verification authority that closes step 5 of the MVP loop per Sim Farm spec [§1.3](../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md) and the [2026-05-16 CEO brief](../../../board/okrs/2026-05-11-0958-ceo-brief.md) O2.

**Consumers this loop:**

- The `make mvp-loop` driver in `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/Makefile` (resink-core, Wave 2 of this loop).
- Any operator inspecting `workspace/verdict.json` after a failed run.

**Future consumers (additive extension only):**

- Mode B (per-node shadow, post-MVP) and Mode C (DAG blue/green warmup, post-MVP) — see Sim Farm spec [§5.2 / §5.3](../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md).
- Training pipeline gate stage 3 (post-MVP, when training adopts coverage-spec-driven verdicts).

## Schema

The engine emits one of two top-level shapes, determined by the invocation:

- **Single-dim shape** (engine_version `0.1.0` / `0.2.0` / `0.3.0`): emitted when exactly one `--fixture` / `--output` pair is supplied. Identical field set across all three; only the `engine_version` string differs.
- **Multi-dim shape** (engine_version `0.2.0`+): emitted when two or more `--fixture` / `--output` pairs are supplied. Top-level carries `overall_pass` + `verdicts[]`; per-dim entries carry the per-dim equivalence result.

Both shapes share the same `Mismatch` sub-schema (unchanged across `0.1.0` → `0.2.0` → `0.3.0`). Engine 0.3.0's schema-aware join + payload columns affect the *engine invocation* signature (a new optional `--schema-ref` flag) and the *content* of `Mismatch.key` and `Mismatch.fixture_row` / `output_row` (which now carry whatever columns the per-pair schema declares, rather than the hard-coded `user_id` / `(email, country, valid_to, is_current)` set). The on-disk verdict file shape — every field name, every top-level structure — is unchanged.

### Single-dim shape (0.1.0 / 0.2.0 / 0.3.0)

The verdict is a single top-level JSON object. Every field is required unless explicitly nullable. The `mismatches` array is empty iff `pass == true`.

| Field            | Type                              | Nullable | Semantics                                                                                                                                                  |
|------------------|-----------------------------------|----------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `verdict_id`     | string (UUID v4)                  | no       | Unique per engine invocation. Generated client-side via `uuid.uuid4()`. Used as the join key in any future Iceberg verdict table.                          |
| `mode`           | string (enum)                     | no       | MVP value is the literal `"A"`. Future Modes B/C will add their own values; MVP consumers may switch on this field.                                        |
| `dim_table`      | string                            | no       | Logical table name being diffed. MVP value is `"dim_user"`; the engine accepts other names via `--dim-table` for future tables in the same fixture family. |
| `fixture_path`   | string (absolute resolved path)   | no       | The fixture parquet that defines truth. Always resolved to an absolute path via `pathlib.Path.resolve()` before serialization.                             |
| `output_path`    | string (absolute resolved path)   | no       | The runtime output parquet. Resolved the same way.                                                                                                         |
| `pass`           | bool                              | no       | `true` iff `mismatch_count == 0`. The single load-bearing field that `make mvp-loop` reads via `verdict=pass` / `verdict=fail` stdout/stderr.              |
| `mismatch_count` | int (≥ 0)                         | no       | The length of `mismatches`. Materialized so consumers can branch without parsing the array.                                                                |
| `mismatches`     | array&lt;Mismatch&gt;             | no       | Empty when `pass == true`. Each entry is one of the three non-match cases (see below). `match` rows are not enumerated in the MVP.                         |
| `ran_at`         | int64 (unix ms, UTC)              | no       | Engine invocation time, milliseconds since epoch. Used for ordering, not for correctness.                                                                  |
| `engine_version` | string (semver)                   | no       | The engine's own version. MVP value is `"0.1.0"` (loop 2026-05-11-0958); bumped to `"0.2.0"` (additive multi-dim extension, loop 2026-05-11-1113); bumped to `"0.3.0"` (additive schema-aware columns, loop 2026-05-11-1631). No field changes in this shape across any of the bumps. Future bumps follow the additivity rules in "Future modes" below.                           |
| `engine_error`   | string                            | yes      | Present **only** in the engine-failure best-effort verdict (exit code 2 path). Carries `<ExceptionClass>: <message>`. Absent on every normal run.          |

### `Mismatch` object

| Field         | Type                              | Nullable | Semantics                                                                                                                                                              |
|---------------|-----------------------------------|----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `type`        | string (enum)                     | no       | One of `"missing"` / `"extra"` / `"diverged"`. The fourth case `"match"` is implied (count = total fixture rows − `mismatch_count`); `match` rows never appear here.   |
| `key`         | object                            | no       | The SCD2 join key. Engine `0.1.0` / `0.2.0` hard-coded this to `{user_id, valid_from}` for every dim. Engine `0.3.0`+ reads the key column names from the per-pair schema-ref JSON (see "Schema-aware columns (0.3.0+)" below); for the legacy fallback path (no `--schema-ref`) the key remains `{user_id, valid_from}`. For `dim_account` under engine 0.3.0 with the natural-column schema, the key reads `{account_id, valid_from}`.                 |
| `fixture_row` | object &#124; null                | yes      | The full row from the fixture. Under engines `0.1.0` / `0.2.0` this is always the six legacy SCD2 columns. Under engine `0.3.0`+ this is the union of `key_columns` + `payload_columns` declared by the per-pair schema-ref (or the legacy six when the schema-ref is omitted). `null` for `type == "extra"` (no fixture side).                                                              |
| `output_row`  | object &#124; null                | yes      | The full row from the output, under the same column-set semantics as `fixture_row`. `null` for `type == "missing"` (no output side).                                                                                         |

The four SCD2-equivalence cases per Sim Farm spec [§4.5](../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md), in DuckDB SQL form:

- `match`: `INNER JOIN` on the key columns where every payload column passes `IS NOT DISTINCT FROM` (NULL-safe equality). Under engines `0.1.0` / `0.2.0` (and the engine `0.3.0` legacy-fallback path) the key columns are `(user_id, valid_from)` and the payload columns are `(email, country, valid_to, is_current)`. Under engine `0.3.0`'s schema-aware path the columns are whatever the per-pair schema-ref declares.
- `missing`: `LEFT JOIN ... WHERE output_side.<first_key_col> IS NULL` (anti-join from fixture side).
- `extra`: `LEFT JOIN ... WHERE fixture_side.<first_key_col> IS NULL` (anti-join from output side).
- `diverged`: `INNER JOIN` on the key columns where `NOT (col_eq(...) AND ...)` — at least one payload column differs under `IS NOT DISTINCT FROM`.

### Schema-aware columns (0.3.0+)

Engine `0.3.0` adds an optional `--schema-ref <path>` CLI flag, repeated per `--fixture` / `--output` / `--dim-table` triple in argument-order pairing — so the engine now accepts **quadruples** rather than triples. The library entrypoint `run_diff_multi(pairs, verdict_path)` accepts each `pair` as a 4-tuple `(fixture_path, output_path, dim_table, schema_ref_path)`; 3-tuples are still accepted for backward compatibility (`schema_ref_path` defaults to `None`).

The schema-JSON shape is a flat object with two required keys:

| Field             | Type                | Required | Semantics                                                                                                                                                            |
|-------------------|---------------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `key_columns`     | array&lt;string&gt; | yes      | The SCD2 join key for this dim. Length ≥ 1. Used for `INNER JOIN` (match / diverged) and `LEFT JOIN` (missing / extra) predicates.                                  |
| `payload_columns` | array&lt;string&gt; | yes      | The non-key columns to compare under `IS NOT DISTINCT FROM`. May be empty (key-only diff; divergence is impossible by construction in that case).                    |

Additional keys are **ignored** by engine `0.3.0` (additivity discipline — future minor bumps may add e.g. `tolerances`, `column_types` without breaking parsers). Required minimum is `{key_columns, payload_columns}`.

Worked examples:

```json
// dim_user_schema.json — the legacy hard-coded shape expressed schema-aware.
{
    "key_columns":     ["user_id", "valid_from"],
    "payload_columns": ["email", "country", "valid_to", "is_current"]
}
```

```json
// dim_account_schema.json — the natural-column shape that the engine-0.2.0
// isomorphic-column compromise wrapped. Engine 0.3.0 reads this directly.
{
    "key_columns":     ["account_id", "valid_from"],
    "payload_columns": ["account_type", "status", "valid_to", "is_current"]
}
```

Path placement convention (proposed; resink-core orchestrator opts in or files an addendum): `<workspace>/<dim_table>_schema.json` and `<fixtures>/<dim_table>_schema.json`, matching the existing `<dim_table>_output.parquet` / `<dim_table>_fixture.parquet` convention from loop 2026-05-11-1113.

Backward compatibility:

- Omitting `--schema-ref` for a given pair-position falls back to legacy hard-coded columns for that pair (`KEY_COLUMNS = ("user_id", "valid_from")`, `PAYLOAD_COLUMNS = ("email", "country", "valid_to", "is_current")`).
- Mixed-mode invocation is supported: some pairs may supply `--schema-ref`, others omit it. The engine pairs `--schema-ref` to `--fixture` / `--output` / `--dim-table` by argument order; missing tail-positions get `None` (legacy fallback). An empty-string `--schema-ref ""` is treated as the explicit "use legacy for this pair" sentinel for cases where mid-quadruple omission is needed.
- The verdict file shape is unchanged. Only `Mismatch.key` (and the columns present in `fixture_row` / `output_row`) reflect the schema-aware columns — and `Mismatch.key` always carried whatever columns were named the join key, so this is a content change, not a shape change.

Schema-JSON shape canonical-ownership note: the shape is documented here, embedded in the sim-farm verdict contract. If `teams/platform/data-engineering` prefers to canonicalize the shape under DE's `conventions/` tree in a future loop, the move is a frontmatter + path change with no engine-side impact (the `--schema-ref` arg and in-memory JSON format stay the same). Coordination request filed at `teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md` with a mid-loop 2026-06-09 deadline for one-paragraph ack.

### Multi-dim shape (0.2.0+)

Emitted when the engine is invoked with two-or-more `--fixture` / `--output` pairs. The top-level structure is:

| Field            | Type                          | Nullable | Semantics                                                                                                                                                                  |
|------------------|-------------------------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `verdict_id`     | string (UUID v4)              | no       | Unique per engine invocation. One UUID per combined verdict file, not per per-dim entry.                                                                                   |
| `mode`           | string (enum)                 | no       | MVP value is `"A"`. Same enum as the single-dim shape.                                                                                                                     |
| `engine_version` | string (semver)               | no       | The engine's own version. MVP+1 value is `"0.2.0"`; MVP+2 (loop 2026-05-11-1631) is `"0.3.0"` for the schema-aware-columns additive bump. The multi-dim shape itself is unchanged across the bump.                                                                                                                       |
| `overall_pass`   | bool                          | no       | `true` iff every `verdicts[i].pass` is `true` (equivalently, iff every `verdicts[i].mismatch_count == 0`). The single load-bearing field consumers gate on.                |
| `verdicts`       | array&lt;PerDimVerdict&gt;    | no       | One entry per `(fixture, output, dim_table)` triple, in invocation order. Length ≥ 1 — a single-pair invocation prefers the single-dim shape (see "Invocation contract"). |
| `ran_at`         | int64 (unix ms, UTC)          | no       | Engine invocation time, milliseconds since epoch. One timestamp per combined verdict.                                                                                       |
| `engine_error`   | string                        | yes      | Present **only** in the engine-failure best-effort verdict (exit code 2 path). `verdicts` is `[]` and `overall_pass` is `false` in that case.                              |

#### `PerDimVerdict` object

| Field            | Type                          | Nullable | Semantics                                                                                                                            |
|------------------|-------------------------------|----------|--------------------------------------------------------------------------------------------------------------------------------------|
| `dim_table`      | string                        | no       | Logical dim-table name. MVP+1 values: `"dim_user"`, `"dim_account"`. Engine accepts arbitrary strings via `--dim-table`.            |
| `fixture_path`   | string (absolute resolved)    | no       | The fixture parquet for this dim. Absolute via `pathlib.Path.resolve()`.                                                            |
| `output_path`    | string (absolute resolved)    | no       | The runtime output parquet for this dim. Absolute the same way.                                                                      |
| `pass`           | bool                          | no       | `true` iff `mismatch_count == 0` for this dim.                                                                                       |
| `mismatch_count` | int (≥ 0)                     | no       | Length of `mismatches` for this dim.                                                                                                 |
| `mismatches`     | array&lt;Mismatch&gt;         | no       | Empty when `pass == true`. Same `Mismatch` sub-schema as the single-dim shape — no changes to `type` / `key` / `fixture_row` / `output_row`. |

The `Mismatch` sub-schema (defined above for the single-dim shape) is unchanged in the multi-dim shape; consumers can share parsing code between the two.

### Pass criterion (multi-dim)

`make mvp-loop` reads `overall_pass`. The PASS criterion is the conjunction:

```
overall_pass == true
  AND every verdicts[i].pass == true
  AND every verdicts[i].mismatch_count == 0
```

By construction, all three clauses are equivalent — `overall_pass` is `true` iff every per-dim `pass` is `true` iff every per-dim `mismatch_count == 0`. Consumers who only need a single boolean can read `overall_pass` alone; consumers who want per-dim breakdowns iterate `verdicts[]`.

## Exit-code semantics

Per Sim Farm spec [§6.3](../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md) (engine-failure case) and the 2026-05-16 sim-farm OKR KR1.1 + KR2.1:

| Exit code | Meaning                                                                  | stdout/stderr                                                                                          | `workspace/verdict.json`                                                                                                                       |
|-----------|--------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| `0`       | overall pass (no mismatches in any dim)                                  | stdout final line: `verdict=pass mismatches=0`                                                         | written. Single-dim shape: `pass: true`, `mismatches: []`. Multi-dim shape: `overall_pass: true`, every `verdicts[i].pass: true`.              |
| `1`       | one or more mismatches in at least one dim                               | stderr final line: `verdict=fail mismatches=<sum> see <verdict-path>`                                  | written. Single-dim shape: `pass: false`, `mismatches: [...]`. Multi-dim shape: `overall_pass: false`, one or more `verdicts[i].pass: false`.  |
| `2`       | engine-internal failure (per spec §6.3)                                  | stderr: `verdict=engine_failure error=<ExceptionClass>: <message>` followed by a Python traceback      | best-effort write of a minimal verdict including an `engine_error` field; in multi-dim mode `verdicts: []` and `overall_pass: false`.          |

`<sum>` on exit code `1` is the **sum of `mismatch_count` across every dim** in multi-dim mode (and is just the single-dim `mismatch_count` in single-dim mode).

Exit code `2` is reserved for engine-internal faults — bad parquet path, DuckDB query exception, schema mismatch that the engine cannot resolve. A schema mismatch where the engine **can** classify rows (e.g., column added on output side that is unknown to the fixture) is treated as a structural diff and surfaces as `extra`/`diverged` rows in the relevant per-dim entry, not as exit code 2 — matching the spec §6.3 distinction.

## Invocation contract

The engine accepts both module and bare-script forms; pick whichever is least painful in the calling Makefile. The shape choice (single-dim vs multi-dim) is determined by **the number of `--fixture` / `--output` pairs** supplied, not by a mode flag.

### Single-dim invocation (preserved across 0.1.0 → 0.2.0)

```
# Module form:
python -m sim_farm.diff_scd2 \
  --fixture <path> --output <path> --verdict <path>

# Bare-script form (no PYTHONPATH plumbing required):
python repos/resink-ai/resink-core/sim-farm/diff_scd2.py \
  --fixture <path> --output <path> --verdict <path>
```

Optional flag: `--dim-table <name>` (default `dim_user`). Output is the single-dim shape.

### Multi-dim invocation (new in 0.2.0)

Repeat the `--fixture` / `--output` / `--dim-table` flags in matching counts. Triples are **paired by argument order** (the first `--fixture` pairs with the first `--output` and the first `--dim-table`, and so on).

```
python -m sim_farm.diff_scd2 \
  --fixture <f1> --output <o1> --dim-table dim_user    \
  --fixture <f2> --output <o2> --dim-table dim_account \
  --verdict <combined-verdict-path>
```

Counting rules:

- The number of `--fixture` flags MUST equal the number of `--output` flags.
- If `--dim-table` is supplied, its count MUST equal the `--fixture` / `--output` count.
- If `--dim-table` is omitted entirely, every triple defaults to `dim_user` (legacy single-dim default carried over).
- A count mismatch is an engine-internal failure (exit code 2) — the engine writes a best-effort failure verdict.

Output is the multi-dim shape when ≥ 2 pairs are supplied, the single-dim shape when exactly 1 pair is supplied (regardless of whether `--dim-table` is present).

### Schema-aware invocation (new in 0.3.0)

Add an optional `--schema-ref <path>` per quadruple. Paired by argument order against `--fixture` / `--output` / `--dim-table`. Omitted positions fall back to legacy hard-coded columns for that pair (see "Schema-aware columns (0.3.0+)" above).

```
python -m sim_farm.diff_scd2 \
  --fixture <f_user> --output <o_user> --dim-table dim_user    --schema-ref <s_user> \
  --fixture <f_acct> --output <o_acct> --dim-table dim_account --schema-ref <s_acct> \
  --verdict <combined-verdict-path>
```

Additional counting rule (engine 0.3.0+):

- If `--schema-ref` is supplied, its count MUST be `≤` the `--fixture` count. Missing tail-positions get `None` (legacy fallback for that pair). A count `>` `--fixture` is an engine-internal failure (exit code 2).
- An empty-string `--schema-ref ""` is treated as "use legacy for this pair" — the explicit-sentinel pattern for mixed-mode invocations.

### Deferred alternatives

A `--pairs <json-file>` form pointing to a JSON array `[{"fixture": ..., "output": ..., "dim_table": ...}, ...]` was considered and **deferred** — the repeated-flag form is simpler for Makefile recipes and avoids a side-channel input file. The deferred form may reappear in a future loop if a Makefile finds 4+ repeated flag groups painful; the on-disk verdict shape would be unchanged.

### Invocation contract for `make mvp-loop`

Resink-core's `synthetic_tenants/closed_loop_v0/Makefile` should add the following recipe step after the supervisor writes the dim output parquet(s). The bare-script form is used so no `cd` is required and the recipe is one line per shell command.

Single-dim (last loop's invocation — still valid against the legacy fixture):

```
@python repos/resink-ai/resink-core/sim-farm/diff_scd2.py \
  --fixture synthetic_tenants/closed_loop_v0/fixtures/dim_user_fixture.parquet \
  --output  synthetic_tenants/closed_loop_v0/workspace/dim_user_output.parquet \
  --verdict synthetic_tenants/closed_loop_v0/workspace/verdict.json
```

Multi-dim (this loop's widened fixture under `closed_loop_v0_two_dim/`):

```
@python repos/resink-ai/resink-core/sim-farm/diff_scd2.py \
  --fixture synthetic_tenants/closed_loop_v0_two_dim/fixtures/dim_user_fixture.parquet    \
  --output  synthetic_tenants/closed_loop_v0_two_dim/workspace/dim_user_output.parquet     \
  --dim-table dim_user                                                                     \
  --fixture synthetic_tenants/closed_loop_v0_two_dim/fixtures/dim_account_fixture.parquet \
  --output  synthetic_tenants/closed_loop_v0_two_dim/workspace/dim_account_output.parquet  \
  --dim-table dim_account                                                                  \
  --verdict synthetic_tenants/closed_loop_v0_two_dim/workspace/verdict.json
```

The recipe **must** check the engine's exit code:

- `0` → print `verdict=pass`; the Makefile target succeeds.
- `1` → print `verdict=fail` and the verdict path; exit non-zero.
- `2` → print `verdict=engine_failure` and the verdict path; exit non-zero.

The engine itself prints the relevant final stdout/stderr line; the Makefile only needs to gate on the exit code.

If `make mvp-loop` is invoked from a directory other than `repos/resink-ai/resink-core/`, adjust the relative paths accordingly. The engine resolves all paths to absolute paths internally and records them in the per-dim entries of the verdict.

### Dependency

The engine ships its own `pyproject.toml` at `repos/resink-ai/resink-core/sim-farm/pyproject.toml`. The host environment needs `python>=3.11` and the engine's runtime deps (`duckdb`, `pyarrow`, `pytz`). The recommended setup, matching resink-core's `uv` convention:

```
cd repos/resink-ai/resink-core/sim-farm && uv sync
```

The Makefile may either (a) require the operator to have run `uv sync` once, or (b) wrap the invocation in `uv run --directory repos/resink-ai/resink-core/sim-farm python -m sim_farm.diff_scd2 ...`. The contract is agnostic; resink-core picks.

## Worked examples

### Single-dim PASS (0.2.0; legacy `make mvp-loop` green path)

`make mvp-loop` green path against last loop's single-dim fixture. The supervisor's output dim is byte-for-byte SCD2-equivalent to the fixture. The single-dim invocation shape is preserved across `0.1.0 → 0.2.0`; only `engine_version` changes.

```json
{
  "verdict_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "mode": "A",
  "dim_table": "dim_user",
  "fixture_path": "/abs/path/to/synthetic_tenants/closed_loop_v0/fixtures/dim_user_fixture.parquet",
  "output_path":  "/abs/path/to/synthetic_tenants/closed_loop_v0/workspace/dim_user_output.parquet",
  "pass": true,
  "mismatch_count": 0,
  "mismatches": [],
  "ran_at": 1747353600000,
  "engine_version": "0.2.0"
}
```

Engine prints to stdout: `verdict=pass mismatches=0`. Exit code `0`.

### Single-dim FAIL (0.2.0; one missing row, one diverged row)

Forced red-path run (e.g., a single SCD2 row removed from the input fixture before derivation, plus a country-column corruption in the supervisor's output).

```json
{
  "verdict_id": "9f1c4e0a-1d3b-4a8e-9f5b-7c2e1d4f6a8c",
  "mode": "A",
  "dim_table": "dim_user",
  "fixture_path": "/abs/path/to/synthetic_tenants/closed_loop_v0/fixtures/dim_user_fixture.parquet",
  "output_path":  "/abs/path/to/synthetic_tenants/closed_loop_v0/workspace/dim_user_output.parquet",
  "pass": false,
  "mismatch_count": 2,
  "mismatches": [
    {
      "type": "missing",
      "key": {"user_id": "u-007", "valid_from": "2026-04-01T00:00:00+00:00"},
      "fixture_row": {
        "user_id": "u-007",
        "valid_from": "2026-04-01T00:00:00+00:00",
        "email": "ada@example.com",
        "country": "GB",
        "valid_to": null,
        "is_current": true
      },
      "output_row": null
    },
    {
      "type": "diverged",
      "key": {"user_id": "u-003", "valid_from": "2026-03-15T00:00:00+00:00"},
      "fixture_row": {
        "user_id": "u-003",
        "valid_from": "2026-03-15T00:00:00+00:00",
        "email": "bo@example.com",
        "country": "DE",
        "valid_to": "2026-04-20T00:00:00+00:00",
        "is_current": false
      },
      "output_row": {
        "user_id": "u-003",
        "valid_from": "2026-03-15T00:00:00+00:00",
        "email": "bo@example.com",
        "country": "FR",
        "valid_to": "2026-04-20T00:00:00+00:00",
        "is_current": false
      }
    }
  ],
  "ran_at": 1747353601234,
  "engine_version": "0.2.0"
}
```

Engine prints to stderr: `verdict=fail mismatches=2 see /abs/path/to/.../verdict.json`. Exit code `1`.

### Multi-dim PASS (0.2.0; two dims, both clean)

`make mvp-loop` green path against this loop's widened two-dim fixture. Both `dim_user` and `dim_account` outputs are SCD2-equivalent to their fixtures.

```json
{
  "verdict_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "mode": "A",
  "engine_version": "0.2.0",
  "overall_pass": true,
  "verdicts": [
    {
      "dim_table": "dim_user",
      "fixture_path": "/abs/path/to/closed_loop_v0_two_dim/fixtures/dim_user_fixture.parquet",
      "output_path":  "/abs/path/to/closed_loop_v0_two_dim/workspace/dim_user_output.parquet",
      "pass": true,
      "mismatch_count": 0,
      "mismatches": []
    },
    {
      "dim_table": "dim_account",
      "fixture_path": "/abs/path/to/closed_loop_v0_two_dim/fixtures/dim_account_fixture.parquet",
      "output_path":  "/abs/path/to/closed_loop_v0_two_dim/workspace/dim_account_output.parquet",
      "pass": true,
      "mismatch_count": 0,
      "mismatches": []
    }
  ],
  "ran_at": 1747958400000
}
```

Engine prints to stdout: `verdict=pass mismatches=0`. Exit code `0`.

### Multi-dim FAIL (0.2.0; one dim missing a row, the other clean)

Forced red-path run where `dim_user` has a deleted row and `dim_account` is clean. `overall_pass` is `false`; per-dim entries carry the individual results so an operator can diagnose which dim failed.

```json
{
  "verdict_id": "9f1c4e0a-1d3b-4a8e-9f5b-7c2e1d4f6a8c",
  "mode": "A",
  "engine_version": "0.2.0",
  "overall_pass": false,
  "verdicts": [
    {
      "dim_table": "dim_user",
      "fixture_path": "/abs/path/to/closed_loop_v0_two_dim/fixtures/dim_user_fixture.parquet",
      "output_path":  "/abs/path/to/closed_loop_v0_two_dim/workspace/dim_user_output.parquet",
      "pass": false,
      "mismatch_count": 1,
      "mismatches": [
        {
          "type": "missing",
          "key": {"user_id": "u-007", "valid_from": "2026-04-01T00:00:00+00:00"},
          "fixture_row": {
            "user_id": "u-007",
            "valid_from": "2026-04-01T00:00:00+00:00",
            "email": "ada@example.com",
            "country": "GB",
            "valid_to": null,
            "is_current": true
          },
          "output_row": null
        }
      ]
    },
    {
      "dim_table": "dim_account",
      "fixture_path": "/abs/path/to/closed_loop_v0_two_dim/fixtures/dim_account_fixture.parquet",
      "output_path":  "/abs/path/to/closed_loop_v0_two_dim/workspace/dim_account_output.parquet",
      "pass": true,
      "mismatch_count": 0,
      "mismatches": []
    }
  ],
  "ran_at": 1747958401234
}
```

Engine prints to stderr: `verdict=fail mismatches=1 see /abs/path/to/.../verdict.json`. Exit code `1`.

## Backward compatibility

The `0.1.0 → 0.2.0 → 0.3.0` chain is **strictly additive** at the schema layer:

- Existing single-dim invocations (`--fixture <p> --output <p> --verdict <p>` with at most one `--dim-table`, no `--schema-ref`) continue to emit the single-dim verdict shape. Every `0.1.0` field is present with unchanged semantics; only `engine_version` changes value (`0.1.0` → `0.2.0` → `0.3.0`).
- Existing single-dim consumers (last loop's `make mvp-loop`, any operator parsing `verdict.json`) see no structural change. Code that switches on `engine_version` should accept the `0.x` minor band.
- The `Mismatch` sub-schema is unchanged. The four-value `Mismatch.type` enum (`match` implicit / `missing` / `extra` / `diverged`) is unchanged. Per-dim entries in the multi-dim shape reuse the same `Mismatch` objects so consumers can share parsing code.
- Exit-code semantics are unchanged at the boundary: `0` = overall pass, `1` = at least one mismatch, `2` = engine-internal failure. The stdout/stderr final-line format is unchanged.
- Engine 0.3.0's `--schema-ref` flag is **optional**; the `run_diff_multi` library entrypoint accepts both 3-tuples (legacy, `schema_ref` defaults to `None`) and 4-tuples (new, explicit `schema_ref`). Callers that haven't upgraded to the 4-tuple form keep working through the 0.3.0 bump.
- `Mismatch.key` content under engine 0.3.0's schema-aware path carries the natural key column names declared by the per-pair schema-ref (e.g. `{account_id, valid_from}` for `dim_account` under its natural schema), rather than the hard-coded `{user_id, valid_from}` from 0.2.0's isomorphic-column compromise. This is a content improvement, not a shape change — consumers that read `key` as an opaque object continue to work; consumers that hard-code lookups like `key["user_id"]` MUST migrate to read by the declared key column names.

A consumer that wants to handle both shapes uniformly can read:

```python
overall_pass = verdict.get("overall_pass", verdict.get("pass"))
```

— the multi-dim shape carries `overall_pass`; the single-dim shape carries `pass`. They are equivalent semantics at the top-level.

## Future modes (additive extension only)

The schema is **stable for the lifetime of `engine_version: 0.x.y`**: existing fields will not be renamed, retyped, or removed. Future modes (B/C) and post-MVP layers (training-pipeline gate stage 3, three-layer verdict per Sim Farm spec §4.7) extend the schema **additively**:

- Adding a new top-level field (e.g., `divergence_rate` for streaming Mode B): permitted in any 0.x bump.
- Adding a new `Mismatch.type` enum value: permitted; consumers must treat unknown `type` values as unprocessable rather than crashing.
- Adding a new value to the `mode` enum (e.g., `"B"`, `"C"`): permitted; consumers branch on `mode` to know which mode-specific extensions to expect.
- Renaming, retyping, or removing **any** existing MVP field requires an `engine_version` major bump (0.x.y → 1.0.0) and a contract revision.

Modes B/C will likely add: `coverage_layer` (node coverage / scenarios / tolerances per Sim Farm spec §4.7), `divergence_rate`, `sampled_diffs`, `sidecar_session_id`. None of these are present in the MVP.

## Out of scope for this contract

- Modes B (per-node shadow) and C (DAG blue/green warmup) verdict shapes — Sim Farm spec §5.2 / §5.3, deferred to the loop after MVP.
- The three-layer verdict structure (node coverage / scenarios / tolerances) from Sim Farm spec §4.7.
- Iceberg row mapping (no Iceberg in MVP; the verdict is a JSON file on disk).
- `coverage_spec.yaml` per-column tolerance — not consumed by the MVP engine; payload comparison is exact equality under `IS NOT DISTINCT FROM`.
- Streaming heartbeat verdicts (Mode B) — out of scope; the streaming Rust differ (`nanofab-simfarm-stream-diff`, Sim Farm spec §4.6) is not built this loop.

## Verified-against-environment

Per [ADR-2026-05-16-003](../../../board/decisions/2026-05-16-003-contract-environment-verification.md), this contract carries a Verified-against-environment subsection naming the toolchain, runtime deps, OS surface, and auth-mode prerequisites it was exercised against. The 2026-06-06 update is the **first cross-team contract** to receive this subsection going forward; the contract's prior body (engine `0.1.0` / `0.2.0` worked examples) grandfathers under the ADR's transition clause and is not retroactively required.

- **Verification date:** 2026-06-06 (loop 2026-05-11-1631).
- **Engine under test:** `repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py` at `ENGINE_VERSION = "0.3.0"`.
- **Toolchain versions exercised:**
  - Python `>=3.11` (CI / developer-laptop exercised against Python 3.12).
  - `uv` 0.6+ for environment management (`uv sync --extra dev`, `uv run pytest`).
  - DuckDB Python driver `>=1.0.0` (pinned in sim-farm `pyproject.toml`).
  - `pyarrow >= 14.0.0` (pinned in sim-farm `pyproject.toml`).
  - `pytest >= 7.4.0` (dev extra in sim-farm `pyproject.toml`).
- **Runtime dependencies (hard, not optional):**
  - `pytz >= 2024.1` — DuckDB's Python-driver materialization of `TIMESTAMPTZ` columns requires `pytz` at fetch time. SCD2 `valid_from` / `valid_to` are tz-aware UTC, so this is load-bearing. Pinned in sim-farm `pyproject.toml`. (Carried from the engine-0.1.0 mid-build discovery noted in ADR-2026-05-16-003 Context item 2.)
- **OS surface:** OS-agnostic. Exercised against macOS 24.6.0 arm64 (developer-laptop) and Linux x86_64 (CI baseline). No OS-specific paths or syscalls in the engine.
- **Auth-mode prerequisites:** none. The engine is a pure-Python diff over local parquet files; no network calls, no credentials, no service accounts. Operators only need filesystem read access to the fixture + output parquet files and the per-pair `schema.json` files, plus filesystem write access to the verdict-output directory.
- **Verification commands:**
  - `cd repos/resink-ai/resink-core/sim-farm && uv sync --extra dev` (install deps).
  - `cd repos/resink-ai/resink-core/sim-farm && uv run pytest -q` (sim-farm pytest suite — 13/13 green for engine 0.3.0).
  - `cd repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0_two_dim && make mvp-loop` (resink-core end-to-end driver; produces `workspace/verdict.json` with `overall_pass: true`, `engine_version: 0.3.0`).
- **Failure modes if prerequisites are not met:**
  - Missing `duckdb` / `pyarrow` / `pytz`: `ImportError` at engine import time — surfaces as exit code 2 with `engine_error` in the verdict.
  - Missing parquet file: `FileNotFoundError` at view-load time — surfaces as exit code 2 with `engine_error`.
  - Malformed `schema.json` (missing `key_columns` or `payload_columns`, non-list values, empty `key_columns`): `ValueError` at schema-ref-load time — surfaces as exit code 2 with `engine_error`.
  - Column named in `key_columns` / `payload_columns` not present in the parquet: DuckDB raises a binding error at query execution — surfaces as exit code 2 with `engine_error`.
{% endraw %}
