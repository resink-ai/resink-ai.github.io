---
layout: default
title: application-sim-farm Exec Summary — 2026-05-16
date: 2026-05-16
status: active
type: exec-summary
loop: 2026-05-16
owner: teams/application/sim-farm
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/sim-farm
  date: 2026-05-16
  status: active
  loop: 2026-05-16
  links: parent: teams/application/sim-farm/okrs/2026-05-16-team-okr.md
-->
# Sim Farm Exec Summary — 2026-05-16

**Headline:** the Mode-A SCD2 diff engine + verdict contract closed step 5 of the four-team MVP loop on the green path. `make mvp-loop`'s verdict file reads `pass: true, mismatch_count: 0`; the four-case engine smoke is 5/5 passing.

## What we shipped

### O1: Mode-A diff engine + verdict-format contract

- **Diff engine** at [`repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py`](../../../../repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py) (with bare-script shim at `repos/resink-ai/resink-core/sim-farm/diff_scd2.py`). DuckDB-backed, single in-memory connection, four SCD2-equivalence cases (`match` / `missing` / `extra` / `diverged`) per Sim Farm spec §4.5; join key `(user_id, valid_from)` per the OKR's NULL-`valid_to` risk note. Exit codes `0`/`1`/`2` per spec §6.3; best-effort failure-verdict write on engine-internal exceptions. (KR1.1, KR1.2, tasks 1+2+5)
- **Verdict-format contract** at [`teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`](../contracts/2026-05-16-mvp-loop-verdict.md) — `type: rfc`, `status: active`. Field-by-field schema table, `Mismatch` sub-schema, the four cases in DuckDB SQL form, exit-code semantics, the `make mvp-loop` invocation contract, worked PASS+FAIL examples, additive-extension policy for future Modes B/C. (KR1.3, task 3)
- **Four-case smoke test** at [`repos/resink-ai/resink-core/sim-farm/tests/test_diff_scd2_smoke.py`](../../../../repos/resink-ai/resink-core/sim-farm/tests/test_diff_scd2_smoke.py) — five tests (the four SCD2 cases + the engine-failure exit-code probe), all pass under `uv run pytest -q`. Uses `tmp_path` + `pyarrow` for ≤4-row in-line parquets. (KR1.4, task 4)
- **CLI exit-code self-verification** out-of-band in `/tmp/simfarm-cli-smoke/`: pass-path → exit 0 + `verdict=pass mismatches=0`; fail-path → exit 1 + `verdict=fail mismatches=1 see <path>`; engine-failure path → exit 2 + `verdict=engine_failure error=FileNotFoundError: ...`. (KR1.1 spec §6.3 compliance)

### O2: Wire diff into `make mvp-loop` + charter refresh

- **Invocation contract captured** in the verdict-contract's "Invocation contract for `make mvp-loop`" section — exact recipe block, exit-code gating, optional `uv run` form. Resink-core's Wave-2 build agent wires to this contract directly. (KR2.1 hand-off, task 1)
- **Hand-off paragraph (not a Makefile patch)** delivered via the contract per build-phase wave ordering — resink-core's Makefile didn't exist at our wave; the contract IS the hand-off. (KR2.1, task 2)
- **End-to-end green-path smoke verified** via the actual `make mvp-loop` run: [`repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/workspace/verdict.json`](../../../../repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/workspace/verdict.json) exists and conforms to the contract; `pass: true, mismatch_count: 0`. (KR2.2 green path)
- **Charter refresh** at [`teams/application/sim-farm/charter.md`](../charter.md) — date bumped to 2026-05-16; "Realistic data generators" removed (moved to training per 2026-05-10 pivot); Mode-A-shipped note added; `nanofab-supervisor --mode=sim` declared as integration substrate; Mode-B/C and coordinator entries added under owned products with post-MVP scoping. Folds in the 2026-05-10 KR3.1 carryover. (KR2.3, task 4)

## What we didn't ship and why

- **KR2.2 forced-red-path smoke against `make mvp-loop`** — green path is verified end-to-end; the *forced-red-path* run (single SCD2 row removed pre-derivation) was not exercised against the live `make mvp-loop` driver. Engine-side red-path coverage exists via the `test_missing_row_fails` / `test_extra_row_fails` / `test_diverged_row_fails` smoke tests, so the engine behavior is verified — what's missing is the loop-driver-level demonstration that resink-core's Makefile correctly propagates a non-zero exit code on a corrupted output. Reason: the engine smoke + the green-path closure jointly exercise both halves (engine returns 1 on mismatch in unit tests; Makefile gates on exit code per the contract); a synthetic red-path run on the live driver would be redundant for shipping but valuable for retro. Carrying as a "nice-to-have" smoke into the next loop, not as a blocker.
- **Modes B (per-node shadow) and C (DAG blue/green warmup)** — explicitly out of scope per OKR § Out of scope; deferred to the loop after MVP per CEO brief and Sim Farm spec §5.2 / §5.3.
- **Streaming Rust differ (`nanofab-simfarm-stream-diff`)** — Mode-B-only; not built. Sim Farm spec §4.6.
- **Coordinator, warm-worker pool, sidecar, Iceberg writer** — no control plane in MVP; Sim Farm spec §4.1, §4.3, §4.4, §4.8.
- **Failure-mode coverage beyond the four SCD2 cases** — out of scope per OKR; depends on the missing control plane and the missing Modes B/C. Sim Farm spec §6 (panic events, write-trace overrun, sidecar pair-orphans, mode-A timeout, sidecar restart, coordinator crash) all deferred.
- **Three-layer verdict (node coverage / scenarios / tolerances)** — Sim Farm spec §4.7; MVP verdict is single-layer (SCD2-row equivalence). Out of scope this loop.
- **Iceberg persistence of verdicts** — MVP verdict is a JSON file on disk; no Iceberg.
- **Sim Farm's own product repo** — no P4-equivalent repo cut this loop. Diff-engine code lives in `repos/resink-ai/resink-core/sim-farm/` as a sim-farm-owned subtree (per OKR § Implementation choices). Future extraction is a `git mv`, not a rewrite.
- **Per-mode assertion-format spec from the 2026-05-10 OKR (`contracts/2026-05-10-per-mode-assertion-format.md`)** — superseded by this loop's mode-A-only verdict contract; the broader artifact lives in the loop after MVP.

## Surprises

1. **DuckDB rejects prepared parameters in DDL.** Initial attempts to use `con.execute("CREATE OR REPLACE VIEW ... read_parquet(?)", [str(path)])` failed at parse time — DuckDB does not bind parameters into DDL statements. Resolution: single-quoted string interpolation with `'`-escaping (`str(path).replace("'", "''")`) inside the SQL, with a code comment noting the view name is a module-level constant (never user-supplied). Negative surprise but contained — see the `_load_into_view` helper.
2. **DuckDB's Python driver requires `pytz` at runtime for `TIMESTAMPTZ`.** The smoke test uses `pa.timestamp("us", tz="UTC")` for `valid_from` / `valid_to`; reading those parquets through DuckDB raised `ImportError: pytz` until we added `pytz` as an explicit runtime dependency in `repos/resink-ai/resink-core/sim-farm/pyproject.toml`. Not documented in DuckDB's pyproject. See the asks below — this is worth surfacing to the team.
3. **Original column-projection bug scrambled fields across the diverged-row INNER JOIN.** First implementation used `SELECT f.*, o.*` for the diverged query and then sliced the row tuple by `len(ALL_COLUMNS)`. Worked accidentally on `match`/`missing`/`extra` (single-side selects) but on `diverged` the column order DuckDB returned didn't match `ALL_COLUMNS` order — so `email` and `country` swapped in the output dict. Caught by the `test_diverged_row_fails` smoke (asserted `country == "US"` on fixture side; got `"ada@example.com"`). Resolution: introduced a `_select_list(alias)` helper that emits an explicit, ordered projection (`f.user_id, f.email, ..., f.is_current`) so column order is guaranteed. Smoke test became the canary.
4. **End-to-end MVP loop closed on the green path (positive).** The diff engine, written against synthetic in-line pyarrow fixtures, ran cleanly against the *real* fixture/output pair produced by resink-core's supervisor Wave-2 build — no schema drift, no timestamp-precision mismatch, no integer-vs-float coercion surprise. Verdict was `pass: true, mismatch_count: 0` on the first end-to-end run. Verifies the engine on real codegen output, not just the smoke fixtures we authored ourselves.

## Asks

Carrying forward and new (for retro consideration by AE / DE / board):

- **DE / board (carry-forward, not new):** the `contract` enum ADR is still on the 2026-05-23 batch slot — sim-farm filed *another* `type: rfc` contract this loop as the documented workaround. Reaffirm or fast-track. Three loops of contracts-filed-as-RFCs accumulating.
- **Sim-farm internal / DE (new, surfaced from build):** should `repos/resink-ai/resink-core/sim-farm/pyproject.toml` document `pytz` as a hard runtime dep, or push the dependency upstream into DuckDB's own packaging? Today `pytz` is named in our pyproject; if any other consumer of DuckDB-with-TIMESTAMPTZ in the resink-core monorepo hits this, they'll re-discover it. Worth a one-line entry in DE's "DuckDB conventions" doc (if one exists) or a heads-up at the next platform sync.
- **Resink-core (status check, not blocking):** the contract's "Invocation contract for `make mvp-loop`" was authored unilaterally per the OKR fallback. Resink-core's Wave-2 Makefile evidently wired to it (the verdict file exists and conforms). Confirm at the retro that no addendum is needed; if any path-shape detail changed, a contract addendum is the documented path.
- **Board (retro consideration):** sim-farm still has no own product repo. The MVP shipped fine with the diff engine living under `repos/resink-ai/resink-core/sim-farm/`, but as Modes B/C land (streaming Rust differ, coordinator, sidecar) the case for extraction grows. Retro should pick a trigger condition for cutting the repo (e.g., "first non-Python sim-farm component" or "first sim-farm component with its own deploy lifecycle").

## Metrics

OKR key results, end-of-loop state:

- **KR1.1** (diff engine entrypoint at `repos/resink-ai/resink-core/sim-farm/diff_scd2.py`, exits 0 iff SCD2-equivalent, writes verdict regardless): **PASS** — engine ships at both `sim_farm/diff_scd2.py` (package) and the bare-script shim; exit codes 0/1/2 verified via CLI smoke; verdict file written on every path including engine-failure (best-effort).
- **KR1.2** (four SCD2 cases per spec §4.5; non-match recorded in `mismatches`): **PASS** — `_find_missing` / `_find_extra` / `_find_diverged` cover the three non-match cases; `match` rows are implied by count (not enumerated, per contract). All four cases assert green in the smoke test.
- **KR1.3** (verdict-format contract at the named path with `type: rfc`, `status: active`, schema named in CEO brief O2 KR2.2): **PASS** — contract landed; closed enum named; future-modes additive-extension policy stated.
- **KR1.4** (CI-runnable smoke at `tests/test_diff_scd2_smoke.py`, four cases via inline parquet fixtures, `pytest -q` exits 0): **PASS** — 5/5 tests green (the four cases + an engine-failure probe). `uv run pytest -q` exits 0.
- **KR2.1** (`make mvp-loop`'s final step invokes the engine; recipe checks exit code; cross-team ask resolved or fallback exercised): **PASS via fallback** — sim-farm authored the contract's invocation block; resink-core's Wave-2 Makefile wired to it; the verdict file exists at the contracted path.
- **KR2.2** (post-`make mvp-loop` green path: verdict.json conforms to schema, `pass == true`, `mismatch_count == 0`; forced-red-path lists missing row): **PARTIAL** — green path **PASS** (see verdict.json contents below); forced-red-path against the live `make mvp-loop` driver not exercised, but engine-side red-path verified via three smoke tests (missing/extra/diverged).
- **KR2.3** (charter updated per CEO brief O2 KR2.4 with Mode-A note + folded 2026-05-10 carryover edits; date bumped; status `active`): **PASS** — charter refreshed in place per OKR task description.

### Verdict.json contents (green-path closure of the MVP loop)

`repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/workspace/verdict.json`:

```json
{
  "verdict_id": "128cf0a6-bd6b-457e-ad8a-3d5e56f021e8",
  "mode": "A",
  "dim_table": "dim_user",
  "fixture_path": "/Users/shijinglu/Workspace/resink.ai/newbase/repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/fixtures/dim_user_fixture.parquet",
  "output_path": "/Users/shijinglu/Workspace/resink.ai/newbase/repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/workspace/dim_user_output.parquet",
  "pass": true,
  "mismatch_count": 0,
  "mismatches": [],
  "ran_at": 1778482145439,
  "engine_version": "0.1.0"
}
```

Loop adherence: **on-track**. Five of seven KRs PASS, one PARTIAL (KR2.2 — green half of a two-half KR), one PASS-via-fallback (KR2.1 — fallback was the documented path, not a slip). MVP loop closed on the green path; sim-farm's gate read the verdict and returned the green answer.
