---
layout: default
title: application-sim-farm Exec Summary — 2026-05-11-1631
date: 2026-06-06
status: active
type: exec-summary
loop: 2026-05-11-1631
owner: teams/application/sim-farm
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/sim-farm
  date: 2026-06-06
  status: active
  loop: 2026-05-11-1631
  links: parent: board/okrs/2026-05-11-1631-ceo-brief.md
-->
# Sim Farm Exec Summary — 2026-06-06

**Headline:** Engine 0.3 GREEN with schema-aware join + payload columns; retro P1 from 2026-05-23 closed; 13/13 pytest green; verdict contract gains its first `Verified-against-environment` subsection per ADR-2026-05-16-003.

## KR outcomes (O1 — Ship engine 0.3 with schema-aware join + payload columns)

- **KR1.1 — engine bump 0.2.0 → 0.3.0: PASS.** `repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py` `ENGINE_VERSION = "0.3.0"`; `pyproject.toml` `version = "0.3.0"`. `run_diff_multi(pairs, verdict_path)` accepts 4-tuples `(fixture, output, dim_table, schema_ref)` with 3-tuple backward-compat (`schema_ref` defaults to `None` → legacy hard-coded fallback). Verified by `uv run python -c "from sim_farm.diff_scd2 import ENGINE_VERSION; print(ENGINE_VERSION)"` → `0.3.0`.
- **KR1.2 — `--schema-ref` CLI arg + helper refactor: PASS.** `argparse action="append"` repeated per `--fixture` / `--output` / `--dim-table` quadruple, paired by argument order. Empty-string `--schema-ref ""` is the explicit-sentinel "use legacy for this pair"; missing tail positions pad with `None`. Helpers `_load_into_view` / `_find_missing` / `_find_extra` / `_find_diverged` refactored to take `key_columns` + `payload_columns` parameters with the module-level `KEY_COLUMNS` / `PAYLOAD_COLUMNS` constants preserved as defaults. Schema-JSON shape documented inline in the engine module docstring with `dim_user` + `dim_account` worked examples.
- **KR1.3 — 13/13 pytest green: PASS.** `tests/test_diff_scd2_smoke.py` carries the 9 preserved from loop 2026-05-11-1113 plus 4 new: `test_multi_dim_schema_aware_both_pass` (distinct PKs `user_id` + `account_id` both clean → `overall_pass=true`), `test_multi_dim_schema_aware_mismatched_pk` (corrupted `dim_account` → `overall_pass=false`, per-dim `pass=false`, `mismatch_count=2`, mismatches name natural `account_id` / `valid_from`), `test_schema_ref_omitted_falls_back_to_legacy_hardcoded`, `test_engine_version_constant_is_0_3_0`. A 13th regression-prevention test `test_run_diff_multi_accepts_legacy_3_tuple_pairs` covers the library-entrypoint backward-compat for mixed 3-tuple / 4-tuple input lists. `cd repos/resink-ai/resink-core/sim-farm && uv run --extra dev pytest -v` → `13 passed in 0.23s` on Python 3.12 / DuckDB 1.x / pyarrow 14+.
- **KR1.4 — verdict contract update + first Verified-against-environment subsection: PASS.** `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` gains: version-history row for `0.3.0`; "Schema-aware columns (0.3.0+)" subsection inlining the flat `{key_columns, payload_columns}` shape with worked `dim_user` + `dim_account` schemas; "Schema-aware invocation (new in 0.3.0)" subsection under Invocation contract; backward-compat additions covering the 3-tuple-still-accepted clause and the `Mismatch.key` content note. **First cross-team contract to receive the `Verified-against-environment` subsection** per ADR-2026-05-16-003 — names verification date, engine path + version, toolchain (Python 3.12 / DuckDB 1.x / pyarrow 14+ / pytest 7.4+ / uv 0.6+), `pytz >= 2024.1` hard runtime dep (DuckDB TIMESTAMPTZ materialization, carried from ADR-2026-05-16-003 Context #2), OS-agnostic surface (macOS arm64 + Linux x86_64), no-auth-mode prerequisites, verification commands (`uv run pytest -q` + `make mvp-loop`), and failure modes.
- **KR1.5 — resink-core orchestrator integration: PASS WITH ADDENDUM.** Engine 0.3.0 fully consumes per-dim `schema.json` files via `--schema-ref` and is validated against this consumption pattern by `test_multi_dim_schema_aware_both_pass` (library + CLI round-trip with both quadruples schema-ref'd; natural `account_id` / `account_type` / `status` columns in `dim_account`). Resink-core's `make mvp-loop` end-to-end exercise against the widened-fixture parquet files with natural `dim_account` columns is **resink-core's deliverable** this loop (board O2 KR2.5 / orchestrator + per-dim `schema.json` write). The standard producer/consumer split holds: sim-farm defines the SHAPE in its contract and consumes what resink-core writes; the engine-side proof is in place via the schema-aware smoke tests regardless of resink-core's `make mvp-loop` integration timing.
- **KR1.6 — forced-red-path smoke under schema-aware engine: PASS WITH ADDENDUM.** Two-loop carryover (2026-05-16 PARTIAL → 2026-05-23 NOT MET → 2026-06-06 PASS at engine layer) is absorbed via `test_multi_dim_schema_aware_mismatched_pk`: corrupts `dim_account` output (one `account_id` renamed), invokes via CLI `main()`, asserts CLI exit code 1, asserts `overall_pass: false`, asserts the `dim_account` per-dim entry has `pass: false` with `mismatch_count: 2` and each `mismatches[].key` carries the natural `account_id` + `valid_from` (no synthetic `user_id`). The Makefile-driver-level demonstration depends on resink-core's `make mvp-loop` orchestrator landing this loop (KR1.5 addendum); the engine's exit-code propagation is the same code path the Makefile gates on.

## Shipped this loop

- **Engine 0.3.0 schema-aware columns** at `repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py`: `ENGINE_VERSION` bumped 0.2.0 → 0.3.0; new `--schema-ref <path>` CLI arg paired per quadruple in argument-order pairing; `run_diff_multi(pairs, verdict_path)` accepts 4-tuples with 3-tuple backward-compat; helpers refactored to take `key_columns` + `payload_columns` as parameters with legacy module-level constants preserved as defaults; SQL identifiers quoted defensively against reserved-word column names.
- **13/13 pytest smoke green** at `tests/test_diff_scd2_smoke.py` — 9 preserved + 4 new schema-aware tests + a 13th regression-prevention test for mixed 3/4-tuple input lists.
- **Verdict contract 0.3.0 update** at `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` — version-history row, "Schema-aware columns (0.3.0+)" subsection, "Schema-aware invocation (new in 0.3.0)" subsection, backward-compat additions, and the first `Verified-against-environment` subsection per ADR-2026-05-16-003.
- **DE coordination request** filed at `teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md` with `status: open`, mid-loop deadline 2026-06-09, fallback "sim-farm-embedded spec stands" on silence.

## Cross-team coordination

- **resink-core (board O2 KR2.5):** engine 0.3.0 consumes per-dim `schema.json` files via `--schema-ref`; sim-farm proposed the path convention (`<workspace>/<dim_table>_schema.json` and `<fixtures>/<dim_table>_schema.json`, matching the existing parquet path convention from 2026-05-23) and the flat `{key_columns, payload_columns}` shape inline in the contract. Orchestrator dispatch path lands in resink-core's wave; sim-farm's engine-side readiness is CLI-validated end-to-end via the schema-aware smoke tests.
- **DE (paused this loop):** one-paragraph schema-JSON-shape-ack requested via the request mechanism rather than blocking on a synchronous answer. Deadline 2026-06-09; silence is taken as "spec stands embedded in sim-farm contract; revisit canonical ownership next loop without engine-side change."
- **board (informational):** the verdict contract update is the worked-example application of ADR-2026-05-16-003's `Verified-against-environment` subsection. Future cross-team contracts include the subsection per the ADR; existing contracts grandfather per the ADR's transition clause.
- **SRE / agent-engineering (informational, no response):** no `Mismatch.type` enum changes (still the closed four-value set `match` / `missing` / `extra` / `diverged`); per-dim `mismatches[].key` for `dim_account` now carries natural `account_id` + `valid_from` rather than the synthetic `user_id` from engine 0.2.0 — diagnostic improvement, not a breaking change.

## Architectural-debt closure

- **`dim_account` isomorphic-shape compromise LIFTED.** Resink-core's `dim_account` fixture/output parquets can now carry natural `account_id` + `account_type` + `status` columns; engine 0.3.0 reads the natural-column schema via the per-pair `--schema-ref` JSON. The compromise NAMED in resink-core's `docs/architecture.md § Named deviations` (engine-0.2.0-era synthetic `user_id`/`email`/`country` wrapping) retires when resink-core's orchestrator wave lands this loop — sim-farm's engine-side lift is the enabling artifact.
- **Forced-red-path smoke CLOSED at the engine level.** The two-loop carryover (2026-05-16 PARTIAL → 2026-05-23 NOT MET) is absorbed by `test_multi_dim_schema_aware_mismatched_pk`: CLI exit 1, `overall_pass: false`, per-dim `pass: false`, natural-column mismatch keys. The library + CLI round-trip exercises the exact code path the Makefile gates on; the Makefile-driver demonstration completes when resink-core's `make mvp-loop` integration lands.
- **Retro P1 from 2026-05-23 CLOSED.** The engine 0.2.0 hard-coded `KEY_COLUMNS = ("user_id", "valid_from")` + payload `(email, country, valid_to, is_current)` shape is lifted; the additivity discipline carried from 0.2.0 holds (omitted `--schema-ref` → legacy fallback; 3-tuple `run_diff_multi` pairs still accepted).

## Asks for CEO consolidation

- **DE response on schema-JSON shape:** one-paragraph ack by 2026-06-09 (filed at `teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md`). Silence past 2026-06-09 means the sim-farm-embedded spec stands; if DE prefers canonical ownership next loop, migration to `conventions/dim-schema-json.md` is a frontmatter + path move with no engine-side change.

## Tenant-isolation invariant

- **Held.** Dry-run after the engine + contract edits: the engine module, the new test additions, the verdict contract body, and the DE request are tenant-only artifacts (sim-farm-owned subtree under `repos/resink-ai/resink-core/sim-farm/` + `teams/application/sim-farm/` + `teams/platform/data-engineering/requests/`); no `org-os/` strings introduced, no cross-tenant identifiers leaked, no `resink.ai` literals embedded in engine code or test fixtures.
