---
layout: default
title: "User guide"
nav_order: 5
parent: "resink-core"
render_with_liquid: false
date: 2026-05-30
status: active
type: doc
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: doc
  owner: teams/application/resink-core
  date: 2026-05-30
  status: active
-->
# User Guide — resink-core

How to get the MVP closed loop running locally, what artifacts it produces, how to reset, and how to triage the common failure modes. Every command in this document is read directly off the `synthetic_tenants/closed_loop_v0/Makefile` and the orchestrator's `__main__.py`; if the Makefile or orchestrator changes, this doc changes with it.

<!-- rit-docs-init:begin auto-generated -->

## Prerequisites

- **Rust toolchain ≥ 1.83** via `rustup`. The Cargo workspace's `[workspace.package]` declares `rust-version = "1.83"`; older toolchains will fail at workspace resolve time. `cargo --version` should print 1.83 or newer. The release profile compiles with `lto = "thin"` and `codegen-units = 1` — first-run builds are slow (~30s on a recent laptop); subsequent incremental builds are seconds.
- **Python 3.12** (≥3.11 is the declared floor for both `training/orchestrator/pyproject.toml` and `sim-farm/pyproject.toml`).
- **`uv`** — the Python package manager used by both `training/orchestrator/` and `sim-farm/`. Both projects ship a `.venv` populated via `uv sync` on first use; the Makefile shells out to each `.venv/bin/python` directly so a one-time `uv sync` per subproject is enough.
- **`make`** — standard GNU make. Confirmed against macOS bsdmake and Linux gnumake; the Makefile uses `$(abspath ...)`, `ifeq`, and `.PHONY`, all portable.
- **Optional: `claude` CLI on PATH.** The orchestrator's default codegen path is in-process slot-fill (`--skip-dispatch`), so the `claude` CLI is not required for `make mvp-loop` to pass. The CLI is only consulted when `CLAUDE_DISPATCH=1` is set; in that case, the CLI must be authenticated either via `ANTHROPIC_API_KEY` env var or the user's OAuth keychain (per `dispatch.py`'s notes, `claude --bare` is **not** used because its strict env-only auth breaks the laptop critical path).
- **Optional: Helm 3** — only needed if you want to exercise the chart gates under `deploy/charts/nanofab-supervisor/`. `helm version` must print v3.

## First run — `make mvp-loop`

```bash
cd repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0
make mvp-loop
```

The recipe runs steps 1 through 7 sequentially (each step's recipe is its own `.PHONY` target; `make mvp-loop` is an alias for `verdict`, which transitively depends on all earlier steps):

1. `make fixture` — `[mvp-loop] step 1: generate dim_user + dim_account fixtures`. Invokes `orchestrator/.venv/bin/python fixtures/generate.py --out fixtures/dim_user_fixture.parquet`. Writes both dim parquets + their schema JSONs into `fixtures/`.
2. `make facts` — `[mvp-loop] step 2: derive fact parquets (sign_up, profile_update, account_open)`. Invokes `orchestrator/.venv/bin/python fixtures/derive_facts.py --in fixtures/dim_user_fixture.parquet --out-dir fixtures`. Writes the three fact-stream parquets.
3. `make orchestrate` — `[mvp-loop] step 3: orchestrator (codegen x2 + manifest + seal + cargo build/test x2)`. Cd's into `training/orchestrator/` and runs `uv run python -m orchestrator --fixture-dir <fixtures> --workspace <workspace> --plugin-dir <nanofab-plugin> --skip-dispatch` (the `--skip-dispatch` is omitted when `CLAUDE_DISPATCH=1`). The orchestrator slot-fills two codegen-output crates under `workspace/nodes/`, writes `workspace/manifest.yaml` + `workspace/.nanofab/release_seal.json`, runs per-crate `cargo build --release` + `cargo test --release`, flips seal stages.
4. `make build-rust` — `[mvp-loop] step 4: cargo build --release -p nanofab-supervisor + coordinator`. Cd's back to the repo root and builds the supervisor (which path-deps both codegen-output crates from step 3, so this is where static linking happens), then the coordinator.
5. `make publish-dag` — `[mvp-loop] step 5: coordinator publish-dag`. Runs `target/release/nanofab-coordinator publish-dag --workspace <workspace>`. Stdout: `accepted, version=0.2.0` on green.
6. `make run-supervisor` — `[mvp-loop] step 6: supervisor --mode=sim (multi-node)`. Runs `target/release/nanofab-supervisor --mode=sim --workspace <workspace> --fixtures-dir <fixtures> --write-trace <workspace>/trace.jsonl --write-output-prefix <workspace>`. Stdout: `supervisor: ok`; stderr: per-node row count summary.
7. `make verdict` — `[mvp-loop] step 7: sim-farm diff_scd2 (multi-dim)`. Cd's into `sim-farm/` and runs `uv run python -m sim_farm.diff_scd2` with two `--fixture` / `--output` / `--dim-table` triples and `--verdict <workspace>/verdict.json`. Stdout: `verdict=pass mismatches=0` on green; exit 0.

## What you should see on green

After `make mvp-loop` returns, the tenant workspace contains:

- **`workspace/verdict.json`** — JSON conforming to the sim-farm verdict contract; multi-dim shape with `overall_pass: true` and two per-dim verdicts each carrying `pass: true` and `mismatches: []`.
- **`workspace/trace.jsonl`** — one JSONL record per supervised event across both nodes (the `node_id` field is the discriminator).
- **`workspace/dim_user_output.parquet`** — the supervisor's `dim_user` SCD2 output.
- **`workspace/dim_account_output.parquet`** — the supervisor's `dim_account` SCD2 output.
- **`workspace/manifest.yaml`** + **`workspace/.nanofab/release_seal.json`** — the orchestrator's manifest and the per-node seal (both nodes' compile + smoke stages `passed`).
- **`workspace/training_history.md`** — one row per node per run, append-only.
- **`workspace/nodes/dim_user_scd2/`** and **`workspace/nodes/dim_account_scd2/`** — full Cargo crates with `Cargo.toml`, `Cargo.lock`, `src/lib.rs`, `target/release/lib*.dylib` (or `.so` on Linux), and `STATUS.json`.

Stdout's final line contains `verdict=pass`; the recipe exits 0. On failure, the final stderr line contains `verdict=fail mismatches=N see <verdict-path>` and the recipe exits non-zero.

## Resetting — `make clean`

```bash
cd repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0
make clean
```

`make clean` is the canonical reset. The exact artifacts removed (from the Makefile's `clean` recipe):

- `fixtures/*.parquet`, `fixtures/dim_user.schema.json`, `fixtures/dim_account.schema.json` — the generated dim and fact parquets plus the two schema JSONs.
- `workspace/trace.jsonl`, `workspace/dim_user_output.parquet`, `workspace/dim_account_output.parquet`, `workspace/verdict.json` — the supervisor and sim-farm outputs.
- `workspace/.nanofab/` (recursive), `workspace/manifest.yaml`, `workspace/training_history.md` — the orchestrator's manifest, seal, and history.
- `workspace/nodes/dim_user_scd2/target/`, `workspace/nodes/dim_account_scd2/target/` — per-node cargo build artifacts.
- `workspace/nodes/dim_user_scd2/STATUS.json`, `workspace/nodes/dim_account_scd2/STATUS.json` — per-node skill status.
- `workspace/nodes/dim_user_scd2/Cargo.lock`, `workspace/nodes/dim_account_scd2/Cargo.lock` — per-node lockfiles.
- `workspace/__det_a`, `workspace/__det_b`, `workspace/__det_trace_a.jsonl`, `workspace/__det_trace_b.jsonl` — determinism-test scratch dirs/files (written by `cargo test --release -p nanofab-supervisor --test determinism -- --include-ignored`).

`make clean` does **not** drop the workspace-root `target/` (the supervisor + coordinator binaries); a `cargo clean` from the repo root handles that.

## Environment variables

- **`CLAUDE_DISPATCH`** — defaults to unset. When `CLAUDE_DISPATCH=1`, the Makefile's `ORCH_FLAGS` toggles from `--skip-dispatch` to empty, which makes the orchestrator route codegen through `claude` CLI (LLM-driven dispatch via AE's `nanofab:codegen-scd2-node` skill) instead of the deterministic in-process slot-fill. Note: the live env var is `CLAUDE_DISPATCH=1` to **enable** dispatch; there is no `CLAUDE_SKIP_DISPATCH` env var (the negation lives on the orchestrator CLI as `--skip-dispatch`, which the Makefile passes by default).
- **`ANTHROPIC_API_KEY`** — required when `CLAUDE_DISPATCH=1` *and* the user's `claude` CLI is not authenticated via OAuth keychain. The orchestrator's `dispatch.py` deliberately drops the `--bare` flag so the CLI can pick up OAuth credentials when the env var is absent.

## Troubleshooting

- **`cargo build` fails on a codegen-output crate (step 3).** The orchestrator prints `[orchestrator] [<dim>] stage compile: FAILED (exit=N)` to stderr and dumps the cargo stderr. Inspect `workspace/nodes/<dim>_scd2/src/lib.rs` (the slot-filled source). The seal at `workspace/.nanofab/release_seal.json` will carry `status: failed` plus `reason: cargo build exit=N` for the offending node. Per-node failures are independent; one node's compile failure does not silently mark the other passed.
- **`verdict=fail mismatches=N`.** Open `workspace/verdict.json`. The `verdicts[].mismatches[]` list carries per-dim diff records with `type` (`missing` | `extra` | `diverged`), `key`, `fixture_row`, and `output_row`. Compare to the corresponding `workspace/<dim>_output.parquet` and `fixtures/<dim>_fixture.parquet`.
- **`verdict.json` shape mismatch.** Check the `engine_version` field. As of loop 2026-05-23 the sim-farm engine is at `0.2.0` (multi-dim); a `0.1.0` legacy single-dim verdict would surface here only if `sim-farm/sim_farm/diff_scd2.py` is older than the Makefile expects.
- **Parquet non-determinism (repeat runs produce different bytes).** Reset with `make clean` and re-run. If non-determinism persists, run the determinism test directly: `cargo test --release -p nanofab-supervisor --test determinism -- --include-ignored` from the repo root; it produces `workspace/__det_a` and `workspace/__det_b` and compares them byte-for-byte.
- **`claude` CLI authentication errors (only with `CLAUDE_DISPATCH=1`).** If you see `Not logged in` or auth-related stderr from `claude`, the dispatch path can't reach Anthropic's API. Two fixes: (a) set `ANTHROPIC_API_KEY` in your shell; or (b) run `claude` interactively once to populate the OAuth keychain. The default `--skip-dispatch` path bypasses this entirely.
- **Supervisor terminal failure / panic.** See the SRE runbook at `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` (in parent newbase). The supervisor uses `panic::catch_unwind` and emits the panic message into `trace.jsonl`; the runbook is the triage entry-point for any non-success terminal state.

<!-- rit-docs-init:end -->
