---
layout: default
title: application-sim-farm OKR — 2026-05-11-0958
date: 2026-05-11
status: active
type: okr
loop: 2026-05-11-0958
owner: teams/application/sim-farm
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/sim-farm
  date: 2026-05-11
  status: active
  loop: 2026-05-11-0958
  links: parent: board/okrs/2026-05-11-0958-ceo-brief.md
-->
{% raw %}

# Sim Farm OKR — 2026-05-16

## Context

Last loop (2026-05-10) was plan-only: we re-aimed sim-farm scope onto the supervisor `--mode=sim` interface and walked the failure-mode list, but landed neither the validation contract nor the per-mode assertion-format spec. Both were carryover into this loop. The 2026-05-16 CEO brief re-anchors that carryover: this loop is a **build** loop, the deliverable is the **MVP closed-loop diff/verdict** (Mode-A only) that closes step 5 of the four-team MVP, and Modes B/C plus the full failure-mode coverage are deferred to the loop after MVP. We own [O2](../../../../board/okrs/2026-05-11-0958-ceo-brief.md) and our verdict file is the gate that decides whether `make mvp-loop` exits 0 (per [O1 KR1.5](../../../../board/okrs/2026-05-11-0958-ceo-brief.md)). Spec ground: Sim Farm spec [§1.3](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md), [§4.2](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md), [§4.5](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md), [§6](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md).

## Objectives

### O1: Ship the Mode-A diff engine and the verdict-format contract

- source: ceo-brief

Why it matters: Without a diff engine that takes two parquets and emits a verdict, `make mvp-loop` cannot close. Without a written verdict-format contract, resink-core's driver and any future caller (training stage 3, future Modes B/C) have nothing typed to consume. Both surfaces are sim-farm's authority per Sim Farm spec §1.3 and §4.5. Maps to CEO brief [O2 KR2.1, KR2.2](../../../../board/okrs/2026-05-11-0958-ceo-brief.md) and supplies CEO brief [O1 KR1.5](../../../../board/okrs/2026-05-11-0958-ceo-brief.md).

**Key results**

- KR1.1: A diff engine entrypoint exists at `repos/resink-ai/resink-core/sim-farm/diff_scd2.py` (Python, single file, invokable as `python -m sim_farm.diff_scd2 --fixture <path> --output <path> --verdict <path>`). It exits 0 iff the two parquets are SCD2-equivalent and writes a verdict JSON to `--verdict` regardless of pass/fail. Implementation language: **Python with the `duckdb` package** — see § Implementation choices below for justification.
- KR1.2: The diff engine handles the four SCD2-equivalence cases per Sim Farm spec §4.5 (alignment by primary key + SCD2 version key): `match` (row exists in both with identical payload), `missing` (row in fixture but not in output), `extra` (row in output but not in fixture), `diverged` (key matches but payload differs). Each non-match case is recorded in the verdict's `mismatches` array with the offending key and the two rows.
- KR1.3: `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` exists with `type: rfc`, `status: active`, defining the verdict JSON schema named in CEO brief [O2 KR2.2](../../../../board/okrs/2026-05-11-0958-ceo-brief.md): `{verdict_id, mode: "A", dim_table, fixture_path, output_path, pass: bool, mismatch_count: int, mismatches: [...], ran_at, engine_version}`. The contract names the four mismatch types as the closed enum for this loop and notes that future modes (B/C) will extend, not redefine, the schema. Filed as `type: rfc` per the still-open `contract` enum ask (DE's standing workaround, not unblocking this loop).
- KR1.4: Engine self-test: a CI-runnable smoke at `repos/resink-ai/resink-core/sim-farm/tests/test_diff_scd2_smoke.py` covers the four mismatch cases via tiny in-line parquet fixtures (≤10 rows each). `pytest sim-farm/tests/test_diff_scd2_smoke.py -q` exits 0.

**Tasks**

- [x] Decide diff-engine implementation language and location; record decision inline (§ Implementation choices) — owner: teams/application/sim-farm
  - Decision: Python with the `duckdb` package, located at `repos/resink-ai/resink-core/sim-farm/`. Already named in § Implementation choices below; no change required.
- [x] Author `diff_scd2.py` against the fixture/output pair using DuckDB `SELECT ... ANTI JOIN`/`EXCEPT` patterns per Sim Farm spec §4.5 — owner: teams/application/sim-farm
  - Shipped at `repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py` (package module) plus a top-level shim at `repos/resink-ai/resink-core/sim-farm/diff_scd2.py` so both `python -m sim_farm.diff_scd2` and bare-script invocations work without PYTHONPATH gymnastics. Uses `LEFT JOIN ... WHERE ... IS NULL` for missing/extra and `INNER JOIN ... WHERE NOT (col IS NOT DISTINCT FROM ...)` for diverged. Join key is `(user_id, valid_from)` per the OKR's NULL-`valid_to` risk note.
- [x] Author the verdict-format contract at `contracts/2026-05-16-mvp-loop-verdict.md` with the schema, an example PASS, and an example FAIL — owner: teams/application/sim-farm
  - Contract body filled in. Frontmatter `status` flipped from `draft` to `active`. Includes field-by-field schema table, `Mismatch` sub-schema, the four SCD2-equivalence cases in DuckDB SQL form, exit-code semantics (0 / 1 / 2), invocation contract for `make mvp-loop`, worked PASS + FAIL examples, future-modes additive-extension policy.
- [x] Author the four-case smoke test (`test_diff_scd2_smoke.py`) using inline `pyarrow`/`duckdb` parquet writes — owner: teams/application/sim-farm
  - Shipped at `repos/resink-ai/resink-core/sim-farm/tests/test_diff_scd2_smoke.py`. Five test cases (the four SCD2 cases + an engine-failure exit-code probe). All five pass under `uv run pytest -q` (exit code 0). Uses `tmp_path` + `pyarrow` to construct ≤4-row parquets per case.
- [x] Self-verify the engine's exit-code semantics (0 on `pass=true`, non-zero on `pass=false`, distinct non-zero on engine failure per Sim Farm spec §6.3) — owner: teams/application/sim-farm
  - Verified out-of-band via direct CLI runs in `/tmp/simfarm-cli-smoke/`: pass-path → exit 0 + stdout `verdict=pass mismatches=0`; fail-path (one missing row) → exit 1 + stderr `verdict=fail mismatches=1 see <abs-path>`; engine-failure path (bogus parquet path) → exit 2 + stderr `verdict=engine_failure error=FileNotFoundError: ...`. The smoke test `test_engine_failure_exit_code` asserts the code-2 path programmatically.

### O2: Wire the diff into `make mvp-loop` and refresh the charter

- source: ceo-brief

Why it matters: A diff engine that nobody calls does not close the loop. Resink-core owns the `make mvp-loop` driver per CEO brief [O1 KR1.1](../../../../board/okrs/2026-05-11-0958-ceo-brief.md) and we own its final step per [O2 KR2.3](../../../../board/okrs/2026-05-11-0958-ceo-brief.md); the invocation shape is a coordination point, not a sim-farm-only decision. The charter touch-up is required by [O2 KR2.4](../../../../board/okrs/2026-05-11-0958-ceo-brief.md) and also clears a carryover from the 2026-05-10 OKR (KR3.1 charter update never landed).

**Key results**

- KR2.1: `make mvp-loop`'s final step invokes `python -m sim_farm.diff_scd2 --fixture synthetic_tenants/closed_loop_v0/fixtures/dim_user_fixture.parquet --output synthetic_tenants/closed_loop_v0/workspace/dim_user_output.parquet --verdict synthetic_tenants/closed_loop_v0/workspace/verdict.json` (or whatever path-shape resink-core finalizes). The Makefile recipe checks the engine's exit code: 0 → print `verdict=pass`; non-zero → print `verdict=fail` followed by the verdict file path and exit non-zero. Cross-team ask to resink-core (mid-loop 2026-05-19) confirms the invocation shape; if shape changes, sim-farm files an addendum to the contract — not a rewrite.
- KR2.2: After a successful `make mvp-loop` run on the green path, `synthetic_tenants/closed_loop_v0/workspace/verdict.json` exists and conforms to the schema in `contracts/2026-05-16-mvp-loop-verdict.md` (KR1.3); `pass == true` and `mismatch_count == 0`. On a forced red-path run (e.g., a single SCD2 row removed from the input fixture before derivation), the verdict file lists the missing row with its primary key and SCD2 version, readable by an operator without grepping `trace.jsonl`.
- KR2.3: `teams/application/sim-farm/charter.md` is updated per CEO brief O2 KR2.4 to add a one-line note: "Mode-A batch diff shipped 2026-05-16 against the MVP loop; Modes B/C remain scoped per Sim Farm spec §1.3." The 2026-05-10 carryover scope edits (remove "Realistic data generators" from owned products; declare `nanofab-supervisor --mode=sim` as integration substrate) are folded in at the same time so the charter is a single coherent document. Charter date bumped to 2026-05-16; status remains `active`.

**Tasks**

- [x] Coordinate with resink-core on the exact CLI invocation shape, exit-code semantics, and verdict file path; capture the agreed shape in the contract — owner: teams/application/sim-farm (asks resink-core)
  - Captured unilaterally in the contract's "Invocation contract for `make mvp-loop`" section per the OKR fallback: sim-farm proposes the shape by 2026-05-19 and resink-core opts in or files an addendum. Resink-core's Wave-2 Makefile build will see the contract and wire to it; if resink-core wants a different invocation, they file a contract addendum (the contract explicitly invites this).
- [x] Submit a PR-shaped patch (or named hand-off) to resink-core's Makefile recipe with the diff invocation; verify locally that `make mvp-loop` exercises the diff step — owner: teams/application/sim-farm
  - Draft contract; resink-core wires in Wave 2 of this loop. The "Invocation contract for `make mvp-loop`" section of `contracts/2026-05-16-mvp-loop-verdict.md` carries the exact recipe block resink-core's build agent should add. Per the build-phase wave ordering, sim-farm did NOT modify resink-core's Makefile (which doesn't exist yet); the hand-off is the contract paragraph.
  - > deferred to Wave 2 of this loop: actual Makefile edit is owned by resink-core's build agent.
- [ ] Run the green-path and forced-red-path smoke once `make mvp-loop` exists end-to-end; confirm verdict.json shape against the contract — owner: teams/application/sim-farm
  - > blocked: resink-core's `make mvp-loop` Makefile + the supervisor that produces `dim_user_output.parquet` ship in Wave 2 of this loop; this end-to-end smoke depends on both. Sim-farm runs this smoke after Wave 2 lands. The four-case engine smoke (KR1.4) plus the manual CLI verification (pass / fail / engine-failure) covers the engine in isolation.
- [x] Update `charter.md` per KR2.3 (Mode-A note + folded 2026-05-10 carryover edits) — owner: teams/application/sim-farm
  - Charter rewritten. Date bumped to 2026-05-16. "Realistic data generators" removed from owned products (moved to training per the 2026-05-10 pivot). Added Mode-A shipped-2026-05-16 note. Declared `nanofab-supervisor --mode=sim` as the integration substrate under Interfaces. Added Mode-B/C and coordinator entries under owned products with their post-MVP scoping.

## Cross-team asks

- **From `teams/application/resink-core`, by mid-loop 2026-05-19:** confirmation of the diff-engine invocation shape — exact CLI args (fixture path, output path, verdict path), exit-code semantics (0 = pass, non-zero = fail; distinct code for engine failure ≥ 2), and the workspace-relative path where `dim_user_output.parquet` will land. Reason: KR2.1 wires into `make mvp-loop`; we need to commit the contract before encoding it. Fallback: sim-farm proposes the shape encoded in the contract (KR1.3) by 2026-05-19; resink-core opts in or files an addendum.
- **From `teams/application/resink-core`, by mid-loop 2026-05-19:** confirmation that the diff engine source can live under `repos/resink-ai/resink-core/sim-farm/` (single-repo simplicity for the MVP; sim-farm has no product repo yet). Reason: location decision is named in § Implementation choices; if resink-core prefers a sibling tree, sim-farm relocates before the wire-up. Fallback: sim-farm assumes yes; resink-core may move it later.
- **Surface to `teams/platform/data-engineering` (informational, no response required):** the verdict contract is filed as `type: rfc` again per the still-open `contract` enum ask (originally surfaced by DE in their 2026-05-10 OKR). No new ask; just continuing the workaround. Reason: the contract is part of a sequence DE will eventually unblock with the enum addition.
- **Surface to `teams/platform/agent-engineering` (informational, no response required):** the diff is purely structural (DuckDB SQL); no LLM-generated path. AE's codegen output is exercised only via the supervisor's `dim_user_output.parquet`; if AE's codegen produces non-SCD2-shaped output, the diff will surface it as `extra`/`missing`/`diverged` rather than crashing.

## Risks

- **Invocation-shape drift with resink-core.** If resink-core lands `make mvp-loop` before our cross-team ask is confirmed, we may need to amend the contract to match the shipped shape. Mitigation: the contract is `type: rfc`; an addendum is the documented path. We commit our preferred shape by 2026-05-19 so resink-core can wire to it.
- **DuckDB SCD2 join semantics may surprise on `valid_from`/`valid_to` boundaries.** `valid_to` is typically NULL for the current row; equality joins on NULL behave differently across DuckDB versions. Mitigation: join key is `(user_id, valid_from)` (always non-null), not `(user_id, valid_to)`; the smoke test (KR1.4) covers a current-row case explicitly.
- **Parquet non-determinism.** Per CEO brief Risks, parquet may not be byte-stable. Mitigation: the diff is value-based (DuckDB reads parquets into typed rows), not byte-based; parquet binary differences do not cause false `diverged` verdicts. Resink-core owns the byte-stable-fixture risk; we are insulated.
- **`type: rfc` workaround creates a contract typed differently from what consumers expect.** Mitigation: the contract body explicitly says "this contract documents an active interface; the `rfc` type reflects the open `contract` enum ask, not contract maturity." Same convention used by DE and previously by sim-farm.
- **Mode-A scope may surface a Mode-B/C constraint we cannot defer.** Unlikely (the closed loop has no shadow path), but if MVP build reveals the runtime emits a multi-version state we cannot collapse, sim-farm files an addendum naming the gap. Mitigation: keep the verdict schema explicitly mode-tagged (`mode: "A"`) so future modes can extend without breaking the MVP shape.

## Out of scope this loop

- **Modes B (per-node shadow) and C (DAG blue/green warmup)** — Sim Farm spec §5.2, §5.3. Defer to the loop after MVP per CEO brief.
- **The streaming Rust differ (`nanofab-simfarm-stream-diff`)** — Sim Farm spec §4.6. Mode-B-only; not needed.
- **Coordinator (`nanofab-simfarm-coordinator`), warm-worker pool, sidecar, Iceberg writer** — Sim Farm spec §4.1, §4.3, §4.4, §4.8. The MVP runs on a developer laptop; no control plane.
- **Failure-mode coverage beyond the four SCD2-equivalence cases** — Sim Farm spec §6 (panic events, write-trace overrun, sidecar pair-orphans, mode-A timeout, sidecar restart, DuckDB engine failure beyond exit-code, coordinator crash). These are properties of the missing control plane and the missing Modes B/C; defer.
- **Synthetic-data generation** — moved to training under the 2026-05-10 pivot; the fixture (`dim_user_fixture.parquet` + derivation) is now resink-core's per CEO brief O1 KR1.2.
- **The full per-mode assertion-format spec from the 2026-05-10 OKR (`contracts/2026-05-10-per-mode-assertion-format.md`)** — superseded by this loop's mode-A-only contract; the broader artifact lives in the loop after MVP.
- **Three-layer verdict (node coverage / scenarios / tolerances) from Sim Farm spec §4.7** — the MVP verdict is single-layer (SCD2-row equivalence); the three-layer structure is a Mode-A-with-coverage-spec concern, deferred.
- **Iceberg persistence of verdicts** — MVP verdict is a JSON file on disk.
- **Sim Farm's own product repo** — still no P4-equivalent for sim-farm; diff-engine code lives in resink-core's tree this loop (see § Implementation choices).

## Implementation choices

- **Language: Python with the `duckdb` package.** Justification: (a) the smoke test, fixture authoring, and resink-core's training-pipeline glue are all Python — keeping the diff in Python avoids a second runtime on the MVP critical path; (b) the DuckDB CLI requires shipping a separate binary in the dev environment, while `pip install duckdb` is one line in resink-core's existing `uv` setup; (c) Python lets us compose `pyarrow` reads with DuckDB SQL where it's cleanest; (d) the future Mode-A-with-coverage-spec work will be Python anyway because tolerances per-column are per-tenant configurable. Cost: slower than CLI for huge parquets — irrelevant for the MVP fixture (≥3 users with ≥2 SCD2 updates each → ≤20 rows).
- **Location: `repos/resink-ai/resink-core/sim-farm/` (in the resink-core product repo, sim-farm-owned subtree).** Justification: (a) MVP simplicity — single repo, single `make mvp-loop` driver; (b) sim-farm has no product repo yet (deferred per the 2026-05-10 OKR § Out of scope); (c) the subtree is sim-farm-owned by `CODEOWNERS` convention (we name it in our charter touch-up KR2.3); (d) future extraction to a sim-farm-owned repo is a `git mv`, not a rewrite — the contract path stays in `teams/application/sim-farm/contracts/`.
- **Verdict file location: `<workspace>/verdict.json` in the closed-loop fixture's workspace.** Justification: matches the pattern of `trace.jsonl` and `dim_user_output.parquet` named in CEO brief O1 KR1.4 — every per-run artifact lives under `workspace/`. The verdict is a per-run artifact, so it lives there too. The contract names this convention; resink-core may finalize the parent path in their OKR.

## Verdict format spec

The authoritative spec lives at `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` (KR1.3). The shape, restated here for teams who read OKRs first:

```json
{
  "verdict_id": "uuid-v4",
  "mode": "A",
  "dim_table": "dim_user",
  "fixture_path": "synthetic_tenants/closed_loop_v0/fixtures/dim_user_fixture.parquet",
  "output_path": "synthetic_tenants/closed_loop_v0/workspace/dim_user_output.parquet",
  "pass": true,
  "mismatch_count": 0,
  "mismatches": [],
  "ran_at": 1747353600000,
  "engine_version": "0.1.0"
}
```

A failing example with one missing and one diverged row:

```json
{
  "verdict_id": "9f1c...-...",
  "mode": "A",
  "dim_table": "dim_user",
  "fixture_path": "...",
  "output_path": "...",
  "pass": false,
  "mismatch_count": 2,
  "mismatches": [
    {
      "type": "missing",
      "key": {"user_id": "u-007", "valid_from": "2026-04-01T00:00:00Z"},
      "fixture_row": {"user_id": "u-007", "email": "ada@example.com", "country": "GB", "valid_from": "2026-04-01T00:00:00Z", "valid_to": null, "is_current": true},
      "output_row": null
    },
    {
      "type": "diverged",
      "key": {"user_id": "u-003", "valid_from": "2026-03-15T00:00:00Z"},
      "fixture_row": {"user_id": "u-003", "email": "bo@example.com", "country": "DE", "valid_from": "2026-03-15T00:00:00Z", "valid_to": "2026-04-20T00:00:00Z", "is_current": false},
      "output_row": {"user_id": "u-003", "email": "bo@example.com", "country": "FR", "valid_from": "2026-03-15T00:00:00Z", "valid_to": "2026-04-20T00:00:00Z", "is_current": false}
    }
  ],
  "ran_at": 1747353601234,
  "engine_version": "0.1.0"
}
```

The four `type` values (`match`, `missing`, `extra`, `diverged`) are a closed enum for this loop's verdict. `match` rows are not enumerated in `mismatches` (count = total fixture rows − `mismatch_count`); they appear in the array only if a future caller opts in to a `verbose` mode (post-MVP). Future modes (B/C) extend the schema with additional fields (e.g., `divergence_rate` for streaming) but never remove or rename the MVP fields. The `engine_version` field is bumped on any backwards-incompatible change.
{% endraw %}
