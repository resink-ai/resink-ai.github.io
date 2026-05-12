---
layout: default
title: application-resink-core Exec Summary — 2026-05-23
nav_exclude: true
render_with_liquid: false
date: 2026-05-23
status: active
type: exec-summary
loop: 2026-05-11-1113
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/resink-core
  date: 2026-05-23
  status: active
  loop: 2026-05-23
  links: parent: teams/application/resink-core/okrs/2026-05-11-1113-team-okr.md
-->
# Resink Core Exec Summary — 2026-05-23

**Headline: the widened MVP closed loop is GREEN.** `make mvp-loop` exits 0 with `verdict=pass mismatches=0` against `synthetic_tenants/closed_loop_v0/` — now widened to **2 dims (`dim_user` 21 rows, `dim_account` 18 rows), 3 facts (`fact_sign_up`, `fact_profile_update`, `fact_account_open`), and 4 logical shards**. `workspace/verdict.json` reports `overall_pass: true` AND both per-dim entries at `pass: true, mismatch_count: 0`. Real LLM-template-driven codegen (deterministic slot-fill path), real multi-node supervisor, real multi-shard xxHash64 partitioning, real multi-dim diff. O1 closed in full (6/6 KRs passed). O2 closed 3/4 KRs; KR2.1 workspace promotion deferred again (single recorded blocker; second consecutive deferral — flagged for retro).

## What we shipped

### O1 — Widen the fixture + orchestrator + supervisor for 2 dims, 3 facts, 4 shards (6/6 KRs)

- **KR1.1** Widened fixture at [`repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/`](../../../../repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/) (extended in place, not forked): `dim_user_fixture.parquet` (21 rows: 10 users × 2 SCD2 versions, one with 3); new `dim_account_fixture.parquet` (18 rows: 15 accounts spanning 13 owners with 3 SCD2 updates); new `dim_account.schema.json` alongside existing `dim_user.schema.json`. Both parquets sha256-stable across two consecutive `make clean && make mvp-loop` runs.
- **KR1.2** Orchestrator at [`repos/resink-ai/resink-core/training/orchestrator/src/orchestrator/__main__.py`](../../../../repos/resink-ai/resink-core/training/orchestrator/src/orchestrator/__main__.py) loops over `--dim-tables` (default `dim_user,dim_account`), dispatches the codegen pattern twice (once per dim), and writes a per-dim row to `training_history.md`. Default `make mvp-loop` runs `--skip-dispatch` against `codegen_slotfill.py` which reads AE templates verbatim — preserves the ADR-002 strict floor on AE (no codegen-pattern work; templates consumed unchanged). LLM dispatch path remains via `CLAUDE_DISPATCH=1`. Two cdylib path-dep crates land at `<workspace>/nodes/dim_user_scd2/` + `<workspace>/nodes/dim_account_scd2/`; both `manifest.yaml` nodes route by `Event.table` discriminator; per-crate stage-1+2 gate.
- **KR1.3** Multi-node supervisor at [`crates/nanofab-supervisor/src/main.rs`](../../../../repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs) runs `DimUserScd2Maintainer` + `DimAccountScd2Maintainer` as two `Node` instances in one process. Per-node `src/nodes.rs` adapters bridge supervisor-local `RawEvent` into each codegen crate's inlined `Event` type. One event source materializes events from all three fact parquets, sorts globally by `(event_ts, event_id)`, routes by `Event.table` discriminator. Two output parquets land at `<prefix>/dim_user_output.parquet` + `<prefix>/dim_account_output.parquet`; one combined `trace.jsonl` carries `node_id` discriminator on every mutation. New flag `--write-output-prefix` (legacy `--write-output` accepted as fallback). `cargo test --release -p nanofab-supervisor` exit 0; `cargo build --release -p nanofab-supervisor` exit 0. **`on-branch: master-2026-05-10-2232`** (workspace promotion deferred — see KR2.1 for the verifiable-location framing).
- **KR1.4** `xxHash64(seed=0)` partition swap at [`crates/nanofab-supervisor/src/event_source.rs`](../../../../repos/resink-ai/resink-core/crates/nanofab-supervisor/src/event_source.rs) using `twox-hash 1.x` `XxHash64::with_seed(0)`. **Byte-stable verified** against DE contract §4 worked example: `xxh64(b"u-001", 0) == 0x571e3e04781b0ff5` and `xxh64(b"42", 0) == 0x6de6f5d076d742b9`; both `mod 4 == 1`. Three unit tests assert these reference values. `shard_count=4` from `manifest.yaml` instantiates 4 logical `ShardKv` per node; combined per-node entries are sorted by `(pk, valid_from)` before parquet emission for byte-stability. Single-process, no IPC. Closes last loop's named placeholder fold-and-mod gap. **`on-branch: master-2026-05-10-2232`**.
- **KR1.5** Determinism preserved across the widening. `crates/nanofab-supervisor/tests/determinism.rs` runs the supervisor twice into separate `--write-output-prefix` dirs, sha256s all three artifacts (`dim_user_output.parquet`, `dim_account_output.parquet`, `trace.jsonl`), asserts all three pairs match. `cargo test --release -p nanofab-supervisor --test determinism -- --include-ignored` exit 0.
- **KR1.6** `make mvp-loop` exit code **0** on the widened fixture; final stdout line `verdict=pass mismatches=0`; `workspace/verdict.json` has `overall_pass: true` and both per-dim `pass: true, mismatch_count: 0`. `make clean && make mvp-loop` also exit 0 (no hidden state leakage from the multi-node widening). Task list updates at [`teams/application/resink-core/okrs/2026-05-11-1113-team-okr.md`](../okrs/2026-05-11-1113-team-okr.md) — all 7 O1 build-phase tasks ticked.

### O2 — Workspace promotion + forced-red-path smoke + status-claim hygiene (3/4 KRs; KR2.1 deferred)

- **KR2.2** Forced-red-path smoke captured at [`synthetic_tenants/closed_loop_v0/red-path-smoke.txt`](../../../../repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/red-path-smoke.txt). Procedure: after green run, drop one SCD2 row from `dim_user_output.parquet`, re-invoke the Makefile's diff step. Outcome: `diff_scd2.py` exits 1; stderr `verdict=fail mismatches=1 see <verdict-path>`; verdict.json `overall_pass: false`; per-dim attribution correct (`dim_user` missing row at `(user_id=u-010, valid_from=1716100000000)`, `dim_account` passes cleanly). Makefile propagates the non-zero exit. Green path restored by re-running `make mvp-loop`. Joint verification with sim-farm complete.
- **KR2.3** Status-claim hygiene per ADR-2026-05-10-003 (ratified this loop). Every shipped item above carries a verifiable git location (`on-branch: master-2026-05-10-2232` since the repo is still in-tree pending KR2.1; the branch is the canonical integration branch in this loop). The KR2.1 promotion claim is truthfully framed as "deferred" rather than "shipped" — see below.
- **KR2.4** `status.md` rewritten end-of-loop with the three-item carryover deviation list explicit. See [`teams/application/resink-core/status.md`](../status.md): (a) ABI Option A static linking remains (ADR-001 multi-loop plan owns the swap; no code lands this loop); (b) `claude --bare` dropped from dispatch — **known-not-fixed**, deferred to 2026-05-30 alongside AE Bundle C per ADR-002 strict floor; (c) placeholder `partition()` fold-and-mod **CLOSED this loop** via the KR1.4 xxHash64 swap.

## What we didn't ship and why

- **KR2.1 workspace promotion to a real git submodule — DEFERRED for the second consecutive loop.** Recorded blocker: the proposed remote `https://github.com/resink-ai/resink-core.git` does not yet exist on GitHub. `git ls-remote` returns `Repository not found`. Per the work-spec's hard rule ("if the remote doesn't exist, STOP and defer"), we invoked the OKR's permitted pivot. `repos/resink-ai/resink-core/` remains an in-tree directory; `.gitmodules` unchanged; README continues to carry the deferral note. **This is the second consecutive deferral** and the OKR explicitly named "we explicitly do not pivot a second time without flagging the cumulative cost to the board at retro" — surfaced under Asks and Surprises.
- **AE DISPATCH.md `--bare` addendum** — not authored by us. Strict ADR-002 floor: AE owns DISPATCH.md. We name the deviation in `status.md` as known-not-fixed; AE's addendum stays scheduled for 2026-05-30 alongside Bundle C per the CEO brief's Standing Answers.
- **`dlopen` restoration** — not pre-staged this loop. ADR-001's multi-loop plan owns the schedule (AE template extension at 2026-05-30; resink-core supervisor swap at 2026-06-06). ABI Option A (Cargo path-dep, static link) carries forward; both codegen-output crates remain path-deps of `nanofab-supervisor`.
- **`teams/application/resink-core/resink-core/` symlink cleanup** — not addressed this loop; flagged for retro consideration. Not load-bearing.

## Surprises

- **(NEGATIVE — architectural-debt flag for retro) `dim_account` parquet column-shape compromise.** Sim-farm's `diff_scd2` engine 0.2.0 hard-codes `user_id` as the join column and `(email, country, valid_to, is_current)` as the payload (per the engine's smoke-test `_account_rows()` caveat). To close the widened loop without forcing a sim-farm contract extension same-loop, resink-core gave `dim_account` an **isomorphic column shape**: `account_id` literals carried in the `user_id` column, an `account_type|status` compound carried in the `country` column, `email` synthetic. SCD2 semantics + 4-shard partitioning + multi-node routing all exercised correctly — but only **column-name semantics were renamed**, not generalized. **This is real architectural debt: the sim-farm engine needs schema-aware join columns (lift the `user_id` hard-coding) before any third dim lands.** Flagged for retro and surfaced as a cross-team ask below.
- **(POSITIVE) xxHash64 swap landed first-attempt with byte-stable verification.** `twox-hash::xxh64::xxh64(key_bytes, 0)` in `crates/nanofab-supervisor/src/event_source.rs::partition()` matches DE's worked example bit-for-bit: `0x571e3e04781b0ff5` for `u-001`; `0x6de6f5d076d742b9` for `42`. The `twox-hash 1.x` crate `XxHash64::with_seed(0)` API was the right pick; no compat regression with `arrow`/`parquet` 53.x. The risk named in the OKR ("`xxHash64` Rust crate selection") never fired.
- **(POSITIVE) Multi-dim supervisor with two `Node` instances + 4-shard logical partitioning worked first-end-to-end.** The CEO brief's Risk #2 ("supervisor single-dim assumptions surface during multi-node wiring") never fired — no shared-mutable-state surface, no concurrency primitive promotion needed beyond the per-shard `HashMap`. Two consecutive `make clean && make mvp-loop` runs produced byte-identical outputs across both dims and the combined trace.
- **(NEGATIVE — KR2.1 carryover) Workspace promotion blocker is now a known-not-fixed pattern.** The promotion was deferred from 2026-05-16 because MVP closure consumed bandwidth; it is deferred again from 2026-05-23 because the GitHub remote doesn't exist. **Single-blocker shape; second-consecutive deferral.** The OKR named this case explicitly and asks for board flagging at retro. Cumulative cost: two loops of in-tree drift over three crates × two more being extended.
- **(POSITIVE) Determinism held all the way through the widening.** Parquet was byte-stable on `pyarrow 24.0.0`, the supervisor's combined `trace.jsonl` and both output parquets were byte-identical across two `--write-output-prefix` runs, and the slot-fill codegen produced byte-stable crates under the deterministic path. The CEO brief's Risk #3 (parquet determinism) again never fired.

## Asks

- **Board — approve the `resink-ai/resink-core` GitHub remote creation.** Workspace promotion (KR2.1) is now blocked exclusively on this: the proposed `https://github.com/resink-ai/resink-core.git` returns `Repository not found`. Either ratify creation of the remote, or name an alternative URL. Second consecutive deferral of KR2.1; cumulative in-tree drift now spans three landed crates plus two more being extended this loop. **Owner: board / CEO. By: 2026-05-30 OKR ratification.**
- **Sim-farm — lift the `user_id`-hard-coded join column in `diff_scd2.py` (schema-aware join columns).** The `dim_account` isomorphic shape this loop is a working compromise, not a generalization; a third dim with a different PK name + payload shape (or any future tenant fixture) cannot close without this. **Surface for retro as architectural debt.** Owner: sim-farm. By: 2026-05-30 OKR.
- **AE — informational only.** The `claude --bare` deviation in `training/orchestrator/src/orchestrator/dispatch.py` continues to defer per ADR-002 strict floor on AE. AE's DISPATCH.md addendum remains scheduled for 2026-05-30 alongside Bundle C, per the CEO brief's Standing Answers. No floor pull; no new ask.
- **DE — informational only.** The `xxHash64(seed=0)` swap is shipped on branch `master-2026-05-10-2232` and is byte-stable against DE contract §4's worked example. The `partition()` commit can be cited against the byte-stable surface; no further DE ratification needed this loop.

## Metrics

**Widened MVP closure (the loop's exit criterion):**
- `make mvp-loop` exit code: **0**
- `make clean && make mvp-loop` exit code: **0**
- `cargo build --release -p nanofab-supervisor` exit code: **0**
- `cargo test --release -p nanofab-supervisor` exit code: **0**
- `cargo test --release -p nanofab-supervisor --test determinism -- --include-ignored` exit code: **0**
- stdout final line of step 7: `verdict=pass mismatches=0`

**`workspace/verdict.json` (full contents):**
```json
{
  "verdict_id": "e2569351-7e5f-461d-b6c3-2080ef0b851e",
  "mode": "A",
  "engine_version": "0.2.0",
  "overall_pass": true,
  "verdicts": [
    {
      "dim_table": "dim_user",
      "fixture_path": ".../fixtures/dim_user_fixture.parquet",
      "output_path": ".../workspace/dim_user_output.parquet",
      "pass": true,
      "mismatch_count": 0,
      "mismatches": []
    },
    {
      "dim_table": "dim_account",
      "fixture_path": ".../fixtures/dim_account_fixture.parquet",
      "output_path": ".../workspace/dim_account_output.parquet",
      "pass": true,
      "mismatch_count": 0,
      "mismatches": []
    }
  ],
  "ran_at": 1778521885912
}
```

**Per-KR status (10 KRs total: 6 in O1, 4 in O2):**

| KR | Title | Status |
|---|---|---|
| KR1.1 | Widened fixture: ≥20 dim_user rows + new dim_account + new schemas | **passed** (dim_user 21 rows; dim_account 18 rows; both sha256-stable; isomorphic column compromise named) |
| KR1.2 | Orchestrator dispatches AE skill twice; two-node manifest; per-crate stages 1+2 | **passed** (slot-fill default; LLM dispatch via `CLAUDE_DISPATCH=1`; strict ADR-002 floor preserved) |
| KR1.3 | Multi-node supervisor: 2 `Node` instances in one process, `Event.table` routing | **passed** (`on-branch: master-2026-05-10-2232`) |
| KR1.4 | xxHash64(seed=0) partition swap; 4 logical shard handlers per node | **passed** (byte-stable vs DE contract §4 worked example; `on-branch: master-2026-05-10-2232`) |
| KR1.5 | Determinism preserved: three-artifact byte-equality across two runs | **passed** |
| KR1.6 | `make mvp-loop` exit 0 + `overall_pass=true` + 0 mismatches across both dims | **passed** |
| KR2.1 | Workspace promotion to real git submodule | **deferred** (permitted pivot; **2nd consecutive deferral**; blocker: GitHub remote `resink-ai/resink-core.git` doesn't exist; flagged for retro) |
| KR2.2 | Forced-red-path smoke through `make mvp-loop` | **passed** (jointly verified with sim-farm; artifact captured) |
| KR2.3 | Status-claim hygiene per ADR-2026-05-10-003 | **passed** (this exec summary names `on-branch:` for every shipped item; KR2.1 framed as "deferred" not "shipped") |
| KR2.4 | `status.md` rewritten end-of-loop with three-item carryover deviation list | **passed** |

**Tally: 9/10 passed, 1/10 deferred (permitted pivot, second consecutive), 0/10 blocked, 0/10 dropped.**
