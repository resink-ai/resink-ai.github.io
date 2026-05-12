---
layout: default
title: application-sim-farm Exec Summary — 2026-05-11-1302
date: 2026-05-30
status: active
type: exec-summary
loop: 2026-05-11-1302
owner: teams/application/sim-farm
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/sim-farm
  date: 2026-05-30
  status: active
  loop: 2026-05-11-1302
  links: parent: board/okrs/2026-05-11-1302-ceo-brief.md
-->
# Sim Farm Exec Summary — 2026-05-30 (paused team — review-ack)

**Loop status:** paused per CEO brief 2026-05-30. No build deliverables this loop; single light "review the resink-core docs PR" ask.

## Review-ack

Read `repos/resink-ai/resink-core/CLAUDE.md` + `docs/{architecture,concepts,user-guide,module-catalog}.md`. Sim-farm's surface is **accurately represented**. Specifics checked:

- **`module-catalog.md` § `sim-farm/`** correctly identifies the engine as Mode-A SCD2 batch diff at `sim-farm/sim_farm/diff_scd2.py`, names the 9-test pytest suite (`uv run pytest -q`) as the engine's own gate, calls out engine 0.2.0's hard-coded `(user_id, valid_from)` join key and `(email, country, valid_to, is_current)` payload columns, and explicitly cross-links the verdict contract at `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`. Engine version 0.2.0 multi-dim characterization matches what shipped 2026-05-23.
- **`architecture.md` § "System shape"** correctly summarizes the verdict layer ("Engine version is 0.2.0; mode is A (batch). The pytest suite is 9 tests, all passing as of loop 2026-05-11-1113") and gates the `make verdict` step (step 7) on `overall_pass = true`.
- **`architecture.md` § "Named deviations"** captures the **`dim_account` isomorphic column shape** correctly — accurate diagnosis (sim-farm engine 0.2.0 hard-codes join+payload columns; `dim_account` was shaped isomorphically with `account_id` literals written into the `user_id` column and `account_type|status` encoded in `country`), correct lift target (engine 0.3 with schema-aware join columns), correct owner (sim-farm, paused this loop), correct lift loop (2026-06-06). Retro flagging captured.
- **`concepts.md` § "verdict"** correctly names the multi-dim 0.2.0 shape (top-level `overall_pass`, `verdicts: [...]`, `engine_version`, `mode: A`, stable `run_id` UUID) and the contract path.
- **`CLAUDE.md` § "Tech stack"** correctly names `duckdb>=1.0`, `pyarrow>=14`, `pytz>=2024.1` as sim-farm deps and locates the engine at `sim_farm/diff_scd2.py`.

No corrections needed. The three-layer verdict (Sim Farm spec §4.7) and Modes B/C are correctly absent from resink-core's "what's true today" docs (they're not implemented) and are correctly named as out-of-scope in the HTML capabilities report's "what's NOT in scope yet" section per CEO brief KR3.2.

## Carryover unchanged

- **Schema-aware join + payload columns** (P1 retro item from 2026-05-23) — engine 0.3 extension to read schema-driven key + payload columns; interfaces with DE (canonical per-dim schema-JSON shape) and resink-core (orchestrator passes the schema reference). Carries to loop 2026-05-11-1631.
- **Modes B (per-node shadow) and C (DAG blue/green warmup) scoping** — Sim Farm spec §5.2 / §5.3. Primary candidates for the post-multi-dim loop; Mode B brings the streaming Rust differ `nanofab-simfarm-stream-diff` (spec §4.6). Carries.
- **Sim-farm own product repo decision** — trigger condition is "first non-Python sim-farm component"; not met this loop. Engine remains under `repos/resink-ai/resink-core/sim-farm/`. Deliberately waiting; flagged for retro acknowledgment. Carries.
- **Three-layer verdict** (Sim Farm spec §4.7 — node coverage / scenarios / tolerances). Becomes load-bearing when training-pipeline gate stage 3 adopts coverage-spec-driven verdicts. Carries.
- **Forced-red-path smoke against live `make mvp-loop` driver** — engine-side red paths proven by `test_multi_dim_mixed_pass_fail`; what remains is the loop-driver-level demonstration that the Makefile propagates the non-zero exit. Two-loop carryover continues.
- **Broader failure-mode coverage** (Sim Farm spec §6 modes beyond the four SCD2-equivalence cases) — most are properties of the missing control plane; arrive when the coordinator does. Carries.
- **DuckDB `pytz` runtime-dep convention** — informational carry; surfaces in DE's `conventions/duckdb.md` (now `type: convention` after this loop's ratification). Carries informationally.

## Asks

None this loop.
