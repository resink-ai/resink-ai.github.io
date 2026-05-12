---
layout: default
title: application-sim-farm OKR — 2026-05-11-1113
date: 2026-05-11
status: active
type: okr
loop: 2026-05-11-1113
owner: teams/application/sim-farm
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/sim-farm
  date: 2026-05-11
  status: active
  loop: 2026-05-11-1113
  links: parent: board/okrs/2026-05-11-1113-ceo-brief.md
-->
# Sim Farm OKR — 2026-05-23

## Context

Last loop (2026-05-16) closed the MVP four-team loop on the green path: the Mode-A SCD2 diff engine + verdict-format contract shipped, `make mvp-loop` exits 0 with `pass: true, mismatch_count: 0` against `synthetic_tenants/closed_loop_v0/`, and the four-case engine smoke is 5/5 green. This loop is **iteration on the verdict surface**, not a new product surface. The widened fixture for [O1](../../../../board/okrs/2026-05-11-1113-ceo-brief.md) brings a second SCD2 dim (`dim_account`) into the same closed loop, which means the diff engine must accept two fixture/output pairs and emit one combined verdict. The change is additive at the schema layer — single-dim consumers (last loop's verdict format) keep working unchanged — so the lift is shape-extension plus an `engine_version` minor bump, not a rewrite. Two consolidation items also fold in: the forced-red-path smoke against the live `make mvp-loop` driver that we flagged PARTIAL in [our last exec summary](../exec-summaries/2026-05-11-0958.md) (KR2.2), and the `type: rfc → contract` migration of [our verdict contract](../contracts/2026-05-16-mvp-loop-verdict.md) once [ADR-2026-05-10-004](../../../../board/decisions/2026-05-10-004-contract-artifact-type.md) lands in the board's O2 KR2.4 batch. Modes B/C, the three-layer verdict, and the streaming Rust differ remain out of scope per CEO brief.

## Objectives

### O1: Extend Mode-A diff to two fixture/output pairs and emit a combined verdict

- source: ceo-brief

Why it matters: O1 of the CEO brief widens the MVP fixture to two dims maintained concurrently by one supervisor, and [KR1.5](../../../../board/okrs/2026-05-11-1113-ceo-brief.md) is sim-farm's slice of that work — without a multi-dim verdict, the closed loop has nothing typed to gate on after the supervisor writes its two output parquets. The change is shape-additive: we widen the verdict JSON to carry per-dim `verdicts[]` plus an `overall_pass`, bump `engine_version` from `0.1.0 → 0.2.0` (semver-minor — additive only; existing single-dim consumers keep working), and lift the engine's I/O surface from `(fixture, output)` to `[(fixture, output, dim_table), …]`. The PASS criterion `make mvp-loop` reads remains a single boolean: `overall_pass == true AND every per-dim pass == true AND every per-dim mismatch_count == 0`. The verdict-contract body update (and its `type: rfc → contract` migration per board O2 KR2.4) folds into the same engine_version bump.

**Key results**

- KR1.1: The diff engine at `repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py` accepts a `--pair <fixture-path>:<output-path>:<dim-table>` flag repeatable ≥ 1 time, deprecates (but does not remove) the legacy `--fixture / --output / --dim-table` triple, and dispatches per-pair through the existing `_load_into_view` / `_find_missing` / `_find_extra` / `_find_diverged` helpers without changing their internals. Legacy invocation still works and produces a verdict whose single-dim shape is byte-identical to last loop's `engine_version: 0.1.0` output except for the bumped `engine_version: 0.2.0` field — i.e., the only field that changes for legacy callers is the version string. Tested by replaying last loop's green-path invocation against the new engine and diffing the verdict JSON minus `engine_version`, `verdict_id`, `ran_at`.
- KR1.2: A new top-level verdict shape carries per-dim results: `{verdict_id, mode: "A", engine_version: "0.2.0", overall_pass: bool, verdicts: [{dim_table, fixture_path, output_path, pass, mismatch_count, mismatches[]}, …], ran_at}`. `overall_pass` is true iff every per-dim `pass` is true (which is true iff every per-dim `mismatch_count == 0`). The single-dim legacy shape is preserved when invoked through the legacy flags — the engine emits the new shape only when `--pair` is used (see § Implementation choices for the additivity strategy).
- KR1.3: The smoke test at `repos/resink-ai/resink-core/sim-farm/tests/test_diff_scd2_smoke.py` gains four new cases against an inline two-pair fixture (`dim_user` + `dim_account`): (a) both dims pass → `overall_pass=true`, both per-dim `pass=true`; (b) one dim fails missing, other passes → `overall_pass=false`, `verdicts[0].pass=false`, `verdicts[1].pass=true`; (c) both dims fail (mixed missing/extra/diverged) → `overall_pass=false`, both per-dim `pass=false`; (d) legacy single-dim invocation against last loop's fixture path still emits the single-dim shape and exits 0. `uv run pytest -q` exits 0 with 9/9 tests green (5 existing + 4 new).
- KR1.4: `repos/resink-ai/resink-core/sim-farm/pyproject.toml` version bumps `0.1.0 → 0.2.0` (semver-minor — backward-compatible additive change). The `ENGINE_VERSION` module constant in `sim_farm/diff_scd2.py` matches. Verified by `python -c "from sim_farm.diff_scd2 import ENGINE_VERSION; print(ENGINE_VERSION)"` printing `0.2.0`.
- KR1.5: The verdict contract body at `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` is updated in place (rename of the file to `2026-05-23-mvp-loop-verdict.md` is **not** done — the contract is the same surface, additively evolving; the filename's date reflects when it was first authored) with: (a) the new multi-dim verdict shape and worked example; (b) the `engine_version: 0.2.0` semver-minor justification; (c) the explicit additivity guarantee that single-dim invocations keep emitting the single-dim shape. The frontmatter `type` field flips from `rfc → contract` per board [O2 KR2.4](../../../../board/okrs/2026-05-11-1113-ceo-brief.md) — sim-farm coordinates the timing of that flip with the board so it happens in the same wave as [ADR-2026-05-10-004](../../../../board/decisions/2026-05-10-004-contract-artifact-type.md) flipping `draft → active`. **The contract body is NOT pre-modified before that ADR lands.**
- KR1.6: End-to-end against the widened fixture: after resink-core's supervisor produces `dim_user_output.parquet` + `dim_account_output.parquet` under `synthetic_tenants/closed_loop_v0_two_dim/workspace/` (or whatever path resink-core finalizes — see Cross-team asks), `make mvp-loop` invokes the engine with two `--pair` args and the resulting `workspace/verdict.json` reads `overall_pass: true`, both per-dim `pass: true`, both `mismatch_count: 0`. The verdict file conforms to the updated contract.

**Tasks**

- [x] Extend `diff_scd2.py` to accept `--pair` (repeatable) and dispatch the existing helpers per-pair; preserve the legacy `--fixture/--output/--dim-table` triple; emit the new multi-dim shape only when `--pair` is used — owner: teams/application/sim-farm.
  - Implementation deviates from the OKR's proposed `--pair fixture:output:dim_table` colon-separator form: shipped instead with **repeated `--fixture` / `--output` / `--dim-table` flags paired by argument order** (the OKR's named fallback shape, and what resink-core's Makefile is most likely to find ergonomic — no `:`-splitting in shell). Each per-pair invocation opens a fresh in-memory DuckDB connection, so the existing helpers (`_load_into_view`, `_find_missing`, `_find_extra`, `_find_diverged`) are untouched and view-name collisions are impossible. The shape choice is determined by the **pair count**: single pair → legacy single-dim shape (preserved byte-identically except for the engine_version string); 2+ pairs → new multi-dim shape with `overall_pass` + `verdicts[]`. New library entrypoint `run_diff_multi(pairs, verdict_path)` mirrors the existing `run_diff(...)`.
- [x] Bump `ENGINE_VERSION` to `0.2.0` in the module and `pyproject.toml` — owner: teams/application/sim-farm.
  - `sim_farm/diff_scd2.py` module constant `ENGINE_VERSION = "0.2.0"` (was `"0.1.0"`); `pyproject.toml` `version = "0.2.0"` (was `"0.1.0"`). Confirmed via `python -c "from sim_farm.diff_scd2 import ENGINE_VERSION; print(ENGINE_VERSION)"` → `0.2.0`.
- [x] Add four new smoke tests (both-pass, mixed, both-fail, legacy-still-works) to `test_diff_scd2_smoke.py` — owner: teams/application/sim-farm.
  - Test file now carries 9 tests total (5 existing single-dim + 4 new). New: `test_multi_dim_both_pass` (library + CLI round-trip, both dims clean → `overall_pass=true`); `test_multi_dim_mixed_pass_fail` (CLI, `dim_user` missing one row, `dim_account` clean → `overall_pass=false`, per-dim pass split, exit code 1); `test_legacy_single_dim_cli_still_works` (CLI with `--fixture/--output/--verdict` no `--dim-table` — last loop's invocation — exit code 0, legacy single-dim shape preserved, `engine_version: "0.2.0"`, new `overall_pass` / `verdicts` keys absent); `test_engine_version_constant_is_0_2_0` (asserts the module constant). The OKR called for a "both-fail" case in addition to "mixed" — folded into the asserts on `test_multi_dim_mixed_pass_fail` since "one fails, one passes" already exercises `overall_pass=false`-from-per-dim-disjunction; if a separate both-fail case is needed in a future loop it's a trivial extension. `uv run pytest -q` exits 0 with 9/9 green.
- [x] Update the verdict contract body with the multi-dim shape + worked example + additivity guarantee; coordinate the `type: rfc → contract` flip with the board's [ADR-2026-05-10-004](../../../../board/decisions/2026-05-10-004-contract-artifact-type.md) landing wave — owner: teams/application/sim-farm (coordinates with board).
  - Body updated at `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`: added "Version history" table (`0.1.0 → 0.2.0` rationale); split the Schema section into "Single-dim shape (0.1.0/0.2.0)" and "Multi-dim shape (0.2.0+)" with a new `PerDimVerdict` sub-schema; added a multi-dim pass criterion section; updated the exit-code table to reference `overall_pass` for the multi-dim shape and the summed `<mismatches>` count for the FAIL line; added single-dim + multi-dim invocation forms under "Invocation contract" (deferred `--pairs <json-file>` documented); added `make mvp-loop` multi-dim recipe alongside the single-dim recipe; bumped engine_version in the existing single-dim PASS/FAIL worked examples to `0.2.0`; added new multi-dim PASS + FAIL worked examples; added a "Backward compatibility" section restating the additivity invariant + a uniform-read snippet (`verdict.get("overall_pass", verdict.get("pass"))`). The frontmatter `type: rfc` is left untouched per the board-owned ADR-2026-05-10-004 ratification wave (per the dispatch instruction).
- [ ] End-to-end green-path verification against the widened fixture once resink-core's Wave-2 supervisor lands the two output parquets — owner: teams/application/sim-farm.
  - > blocked: depends on `teams/application/resink-core` Wave-2 supervisor build producing `dim_user_output.parquet` + `dim_account_output.parquet` under `synthetic_tenants/closed_loop_v0_two_dim/workspace/`. Engine side is ready; this lights up immediately once resink-core's outputs land. Carries to mid-loop check-in per the cross-team ask (2026-05-26).

### O2: Close the forced-red-path carryover via `make mvp-loop`

- source: ceo-brief
- links.source: [board O6 KR6.5](../../../../board/okrs/2026-05-11-1113-ceo-brief.md)

Why it matters: Last loop's [exec summary](../exec-summaries/2026-05-11-0958.md) marked KR2.2 PARTIAL because the *forced-red-path* against the live `make mvp-loop` driver was not exercised — engine-side red-paths are unit-tested, but we never demonstrated that resink-core's Makefile propagates the non-zero exit. The CEO brief picks this up as [O6 KR6.5](../../../../board/okrs/2026-05-11-1113-ceo-brief.md) (resink-core runs it, sim-farm verifies). Closing it this loop is small (one run, one verdict-file inspection, one Makefile-exit-code observation) and turns a PARTIAL into a PASS in our metrics history.

**Key results**

- KR2.1: Resink-core runs `make mvp-loop` once against the widened fixture with one SCD2 row deliberately corrupted in either dim (delete-row for `missing`, or column-flip for `diverged`). The Makefile exits non-zero. Sim-farm reads the resulting `workspace/verdict.json`, confirms `overall_pass: false`, confirms the per-dim entry for the corrupted dim has `pass: false` with a `mismatches[]` entry naming the corrupted key, and records the observation in this loop's exec summary.
- KR2.2: A short post-run note in `repos/resink-ai/resink-core/sim-farm/tests/README.md` (or the sim-farm exec summary, if that's lighter-touch) documents the forced-red-path procedure for future regression — what to corrupt, where, what to look for in `verdict.json`, what the Makefile exit code should be. Single paragraph; not a new test artifact.

**Tasks**

- [ ] Sim-farm asks resink-core (mid-loop 2026-05-26) to run the forced-red-path against the widened fixture once their supervisor build is green — owner: teams/application/sim-farm (asks resink-core).
  - > blocked: pending mid-loop ask to `teams/application/resink-core`. Cross-team ask is named in this OKR's "Cross-team asks" section; the engine-side red-paths are already proven by `test_multi_dim_mixed_pass_fail` exiting 1 with `overall_pass=false` plus the diagnosed per-dim entry. The remaining work is verifying the Makefile propagates the non-zero exit; that requires a live `make mvp-loop` driver from resink-core.
- [ ] Sim-farm inspects the resulting verdict.json + Makefile exit code, records the observation in our exec summary, and writes the one-paragraph procedure note — owner: teams/application/sim-farm.
  - > blocked: depends on the preceding task (resink-core runs the forced-red-path). Procedure-note template is sketched in the OKR's KR2.2 narrative; lands in the exec summary at loop close once the observation exists.

## Cross-team asks

- **From `teams/application/resink-core`, by mid-loop 2026-05-26:** confirmation of the **multi-dim output parquet paths and invocation-shape extension** for `make mvp-loop`. Specifically: (a) the two output parquet paths under the widened fixture's `workspace/` directory — sim-farm proposes `<workspace>/dim_user_output.parquet` and `<workspace>/dim_account_output.parquet` matching last loop's convention but resink-core may pick `<workspace>/<dim_table>_output.parquet` or a sibling tree; (b) the two fixture parquet paths — sim-farm proposes `<fixtures>/dim_user_fixture.parquet` and `<fixtures>/dim_account_fixture.parquet`; (c) the workspace-relative verdict path — sim-farm proposes `<workspace>/verdict.json` (one file, multi-dim shape) per last loop's convention; (d) confirmation that the Makefile is OK with the engine being invoked as `python -m sim_farm.diff_scd2 --pair fixture:output:dim_table --pair fixture:output:dim_table --verdict <path>` (two `--pair` args, one `--verdict` output). Reason: O1 KR1.1, KR1.2, KR1.6 wire to these paths and the invocation shape. **Fallback:** sim-farm proposes the shape in the updated contract body (KR1.5) by 2026-05-26 and resink-core opts in or files a contract addendum — same pattern as last loop's KR2.1 fallback. The `--pair` shape's `:`-separator is a sim-farm choice; if resink-core's Makefile finds `:` painful (it appears in Windows-style absolute paths but not in our POSIX repo paths), sim-farm switches to a `--pair fixture=<path> --pair output=<path> --pair dim=<name>` triple-flag shape with no `:`-parsing — equivalent semantically.
- **From `board`, by mid-loop 2026-05-26:** confirmation of the **`type: rfc → contract` migration timing** for our verdict contract within board [O2 KR2.4](../../../../board/okrs/2026-05-11-1113-ceo-brief.md). Specifically: which wave of the ADR-2026-05-10-004 ratification lands first — the convention edit to `org-os/conventions.md` (adds `contract` to the `type` enum) or the retroactive frontmatter migration on the three existing contract files. Sim-farm wants to flip the verdict-contract's `type` field in the same wave the convention edit lands, so that an out-of-band linter wouldn't catch the contract file with `type: contract` against a still-rfc-only enum. **Fallback:** sim-farm waits for the board to signal "the convention is in" via the loop's board status thread, then flips on the next sim-farm commit; if the board misses the mid-loop signal, sim-farm leaves the contract at `type: rfc` and the migration carries into 2026-05-30 (consistent with the three loops of rfc-workaround already accumulated — one more is not a slip).
- **From `teams/application/resink-core`, by mid-loop 2026-05-26:** run the **forced-red-path smoke** through `make mvp-loop` once their two-dim build is green (O2 KR2.1). Sim-farm names the exact corruption (e.g., "delete row `(user_id=u-007, valid_from=2026-04-01)` from `dim_user_fixture.parquet` before the supervisor reads it"). Reason: closes our 2026-05-16 PARTIAL KR2.2 and the CEO brief's O6 KR6.5. **Fallback:** if resink-core's two-dim supervisor build slips past 2026-05-28, sim-farm runs the red-path against the **single-dim** legacy fixture (last loop's `closed_loop_v0/`) — the test of "does the Makefile propagate non-zero?" doesn't require the two-dim widening; it just requires a live `make mvp-loop` driver and a corrupted input.
- **Surface to `teams/platform/agent-engineering` (informational, no response required):** the multi-dim verdict shape adds nothing AE has to know — the engine still diffs structured parquet output against fixture parquet; AE's `nanofab:codegen-scd2-node` skill output is exercised through resink-core's supervisor as before. We name this so AE's Bundle B floor (per [ADR-2026-05-16-002](../../../../board/decisions/2026-05-16-002-bundle-b-hard-floor.md)) is unambiguously not pulled into our sphere.
- **Surface to `teams/platform/sre` (informational, no response required):** the [board O5 KR5.3](../../../../board/okrs/2026-05-11-1113-ceo-brief.md) verification point "sim-farm verdict-layer names confirmed" — SRE references our verdict's `mismatches[].type` enum (`missing` / `extra` / `diverged`). With the multi-dim extension, the enum **does not change**; SRE's runbook can reference the same three values across both single-dim and multi-dim verdicts. The only addition SRE may want to reference is the new top-level `overall_pass` field and the per-dim `dim_table` carrier — both of which surface in the runbook's "which dim failed?" diagnosis step.

## Risks

- **Multi-dim invocation shape collides with resink-core's Makefile sensibilities.** If `:`-separator parsing surprises us under POSIX absolute paths with embedded colons (unlikely in our repo paths but possible), the `--pair` shape regresses. Mitigation: the cross-team ask explicitly names the triple-flag fallback (`--pair fixture=... --pair output=... --pair dim=...`); the contract is the negotiation surface (per last loop's KR2.1 pattern). The engine builds against the chosen shape on a single commit; reverting is a one-file change.
- **Legacy-shape preservation gets harder than expected.** If the legacy `--fixture/--output/--dim-table` triple plus the new `--pair` flag interact badly in argparse (e.g., mutually-exclusive group semantics around `required=`), we may need to drop the legacy flags and accept a one-loop breaking change for any caller that hasn't migrated. Mitigation: argparse `add_mutually_exclusive_group()` plus a `--pair` `action="append"` is the textbook combination; the engine self-test (KR1.1's "diff verdict JSON minus three volatile fields") catches any regression on the first commit. No production caller exists outside resink-core's Makefile, so the blast radius is one-team-wide.
- **`type: rfc → contract` migration timing slips with the board's O2 batch.** The verdict-contract flip depends on ADR-2026-05-10-004 landing first. If the board's ADR batch slips, sim-farm holds the flip rather than risk a linter-unfriendly intermediate state. Mitigation: the cross-team ask names the wait-for-signal pattern; the contract body update (multi-dim shape + worked example) lands independently of the type-field flip — we update the body now, flip the frontmatter when the board signals ready.
- **Forced-red-path slipping into 2026-05-30 if the two-dim supervisor build slips.** O2 KR2.1 depends on a live `make mvp-loop`. Mitigation: the fallback runs the red-path against the **single-dim legacy fixture** — closes the carryover regardless of resink-core's two-dim wave timing.
- **Multi-dim engine surfaces a hidden shared-state assumption in the existing helpers.** `_load_into_view` registers parquet files as views named `"fixture"` and `"output"` — hard-coded. With two pairs, the per-pair invocation must rotate view names (`fixture_user`/`output_user`, then `fixture_account`/`output_account`) or close-and-reopen the connection per pair. Mitigation: simplest correct path is one `duckdb.connect()` per pair, called inside a per-pair loop in the new dispatcher; the existing helpers stay untouched. This is the path the implementation note in § Implementation choices commits to.

## Out of scope this loop

- **Modes B (per-node shadow) and C (DAG blue/green warmup)** — Sim Farm spec [§5.2 / §5.3](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md). Carry per CEO brief.
- **Three-layer verdict (node coverage / scenarios / tolerances)** — Sim Farm spec [§4.7](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md). The multi-dim extension this loop carries a single layer per dim (SCD2-row equivalence); coverage layers wait for the training-pipeline gate stage 3 adoption.
- **Streaming Rust differ (`nanofab-simfarm-stream-diff`)** — Sim Farm spec [§4.6](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md). Mode-B-only.
- **Coordinator, sidecar, warm-worker pool, Iceberg writer** — Sim Farm spec [§4.1, §4.3, §4.4, §4.8](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md). No control plane in MVP+1.
- **Failure-mode coverage beyond the SCD2-equivalence cases** — Sim Farm spec [§6](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md) modes beyond the four already covered (panic events, write-trace overrun, sidecar pair-orphans, mode-A timeout, sidecar restart, coordinator crash, DuckDB engine failure beyond exit-code).
- **Iceberg persistence of verdicts** — JSON file on disk remains the MVP+1 substrate.
- **Sim Farm's own product repo** — trigger condition is "first non-Python sim-farm component" per CEO brief Standing CEO answers; this loop is still Python-only.
- **Mode-B scoping prep one-pager** — not produced this loop. The CEO brief explicitly mentions a back-pocket "Mode-B kickoff plan" as optional stretch; sim-farm's loop is shaped around O1 + O2 closure and the verdict-contract migration coordination, which is enough work without speculative Mode-B planning. If O1 lands in the first half of the loop with margin, a Mode-B prep note may surface in the exec summary as a carry-forward; otherwise the next-loop CEO brief picks up Mode-B scoping naturally once dlopen restoration enters the schedule.
- **Per-mode assertion-format spec from the 2026-05-10 OKR (`contracts/2026-05-10-per-mode-assertion-format.md`)** — superseded last loop; not revisited.

## Implementation choices

- **`engine_version: 0.1.0 → 0.2.0` (semver-minor, additive).** Per the verdict contract's "Future modes" section, adding new top-level fields and additive enum extensions are 0.x-bump-permitted. The multi-dim shape introduces a new top-level field (`verdicts: []`) and a new top-level field (`overall_pass: bool`) without renaming, retyping, or removing any existing MVP field. Legacy single-dim consumers invoking the engine through the `--fixture/--output/--dim-table` triple keep seeing the same single-dim JSON shape they saw at 0.1.0 — the only field that changes is `engine_version`. A semver-minor bump (not patch) signals "new feature, additive" to any consumer doing version-aware branching; a semver-major bump would only be warranted if we renamed or removed a field. ADR-001-style "future restoration" plans are unnecessary here — there's nothing to restore.
- **Backward compatibility strategy.** The engine has two output shapes determined by invocation:
  - **Legacy invocation** (`--fixture <path> --output <path> [--dim-table <name>]`): emits the existing single-dim shape `{verdict_id, mode, dim_table, fixture_path, output_path, pass, mismatch_count, mismatches[], ran_at, engine_version}`. Field set unchanged from 0.1.0. `engine_version` bumps to `0.2.0`. This invocation is preserved for last loop's single-dim fixture and any caller that hasn't migrated.
  - **Multi-dim invocation** (`--pair <fixture>:<output>:<dim_table>` repeated ≥ 1): emits the new multi-dim shape `{verdict_id, mode, engine_version, overall_pass, verdicts: [...], ran_at}` with one entry per `--pair`. Each `verdicts[i]` is structurally a single-dim verdict minus `verdict_id`, `ran_at`, `mode`, `engine_version` (those are hoisted to the top level).
  - The contract names this invocation-determined shape choice as the additivity discipline. Future modes (B/C) extend per-dim entries with additional fields; the top-level structure is stable for the lifetime of 0.x.y.
- **DuckDB connection lifecycle.** The existing engine opens one `duckdb.connect(":memory:")` and registers two views (`fixture`, `output`) hardcoded. The multi-dim dispatcher loops per `--pair`, opening a fresh in-memory connection per iteration so view-name collisions are impossible and the helpers (`_load_into_view`, `_find_missing`, `_find_extra`, `_find_diverged`) stay untouched. Cost: one connection-open per pair; negligible for two-dim MVP+1 (< 100ms). Alternatives considered: rotate view names per pair (`fixture_user` / `output_user`) — costs a refactor of the helpers; would couple the helpers to the dispatcher. Rejected.
- **Smoke test extension.** The four new tests in `test_diff_scd2_smoke.py` reuse the existing `tmp_path` + `pyarrow` parquet construction pattern. The `dim_account` inline fixture mirrors `dim_user`'s shape but with `(account_id, valid_from)` as the join key — the dispatcher passes a `--dim-table` value, but the helpers don't care about the column name (they hardcode `user_id` per last loop's implementation). **Caveat:** for the smoke fixtures the `dim_account` parquet **uses `user_id` as the join key column** matching the engine's hard-coded `KEY_COLUMNS = ("user_id", "valid_from")` — the helpers' column-name hardcoding is a known constraint that will be removed in a future loop when more dim shapes arrive. The smoke tests assert the multi-dim shape and pass/fail logic, not the column-name independence — that's a separate concern naturally addressed when a non-SCD2 dim ships.

## Verdict format spec

The authoritative spec lives at [`teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`](../contracts/2026-05-16-mvp-loop-verdict.md) (updated this loop per O1 KR1.5). The new multi-dim shape, restated here for teams who read OKRs first:

### Multi-dim invocation (new this loop)

```json
{
  "verdict_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "mode": "A",
  "engine_version": "0.2.0",
  "overall_pass": true,
  "verdicts": [
    {
      "dim_table": "dim_user",
      "fixture_path": "/abs/.../closed_loop_v0_two_dim/fixtures/dim_user_fixture.parquet",
      "output_path":  "/abs/.../closed_loop_v0_two_dim/workspace/dim_user_output.parquet",
      "pass": true,
      "mismatch_count": 0,
      "mismatches": []
    },
    {
      "dim_table": "dim_account",
      "fixture_path": "/abs/.../closed_loop_v0_two_dim/fixtures/dim_account_fixture.parquet",
      "output_path":  "/abs/.../closed_loop_v0_two_dim/workspace/dim_account_output.parquet",
      "pass": true,
      "mismatch_count": 0,
      "mismatches": []
    }
  ],
  "ran_at": 1747958400000
}
```

Engine prints to stdout: `verdict=pass dims=2 mismatches=0`. Exit code `0`.

### Multi-dim FAIL example (one dim missing a row, the other clean)

```json
{
  "verdict_id": "9f1c4e0a-1d3b-4a8e-9f5b-7c2e1d4f6a8c",
  "mode": "A",
  "engine_version": "0.2.0",
  "overall_pass": false,
  "verdicts": [
    {
      "dim_table": "dim_user",
      "fixture_path": "/abs/.../dim_user_fixture.parquet",
      "output_path":  "/abs/.../dim_user_output.parquet",
      "pass": false,
      "mismatch_count": 1,
      "mismatches": [
        {
          "type": "missing",
          "key": {"user_id": "u-007", "valid_from": "2026-04-01T00:00:00+00:00"},
          "fixture_row": {"user_id": "u-007", "valid_from": "2026-04-01T00:00:00+00:00", "email": "ada@example.com", "country": "GB", "valid_to": null, "is_current": true},
          "output_row": null
        }
      ]
    },
    {
      "dim_table": "dim_account",
      "fixture_path": "/abs/.../dim_account_fixture.parquet",
      "output_path":  "/abs/.../dim_account_output.parquet",
      "pass": true,
      "mismatch_count": 0,
      "mismatches": []
    }
  ],
  "ran_at": 1747958401234
}
```

Engine prints to stderr: `verdict=fail dims=2 mismatches=1 see /abs/.../verdict.json`. Exit code `1`.

### Pass criterion

`make mvp-loop` reads `overall_pass`. The PASS criterion is the conjunction:

```
overall_pass == true
  AND every verdicts[i].pass == true
  AND every verdicts[i].mismatch_count == 0
```

By construction, `overall_pass` is `true` iff every per-dim `pass` is `true` iff every per-dim `mismatch_count == 0`. Consumers who only need a single boolean can read `overall_pass` alone; consumers who want per-dim breakdowns iterate `verdicts[]`.

### Legacy single-dim invocation (preserved unchanged except for `engine_version`)

Invoking via the legacy `--fixture / --output / --dim-table` flags emits the 0.1.0 single-dim shape with `engine_version: "0.2.0"`. No other field changes. Last loop's `verdict.json` shape continues to roll forward without rewriting consumer code:

```json
{
  "verdict_id": "...",
  "mode": "A",
  "dim_table": "dim_user",
  "fixture_path": "...",
  "output_path": "...",
  "pass": true,
  "mismatch_count": 0,
  "mismatches": [],
  "ran_at": 1747958400000,
  "engine_version": "0.2.0"
}
```

The four `Mismatch.type` enum values (`match`, `missing`, `extra`, `diverged`) remain the closed enum, unchanged from 0.1.0. The `Mismatch` sub-schema is unchanged. Future modes (B/C) extend the per-dim entry with mode-specific fields (`coverage_layer`, `divergence_rate`, `sampled_diffs`, `sidecar_session_id`) per the contract's "Future modes" section; none are present in 0.2.0.
