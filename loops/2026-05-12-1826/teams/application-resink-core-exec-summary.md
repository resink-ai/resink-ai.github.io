---
layout: default
title: application-resink-core Exec Summary — 2026-05-12-1826
date: 2026-05-12
status: active
type: exec-summary
loop: 2026-05-12-1826
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/resink-core
  date: 2026-05-12
  status: active
  loop: 2026-05-12-1826
  links: parent: board/okrs/2026-05-12-1826-ceo-brief.md
  team_okr: teams/application/resink-core/okrs/2026-05-12-1826-team-okr.md
-->
{% raw %}

# Resink Core Exec Summary — Loop 2026-05-12-1826

**Headline.** First user-facing surface shipped: the `resink` CLI v1. New workspace member `crates/resink-cli/` (5 subcommands; 6 unit tests pass; arrow-prettyprint-formatted tabular output for parquet; clap-derive structure with subcommand help). Build green at `cargo build --release -p resink-cli`. All 5 subcommands exercised against the live `make mvp-loop` workspace: `workspace status` shows 5/5 ✓ artifacts; `verdict latest` prints `PASS / engine_version: 0.3.0 / per-dim PASS x2`; `manifest get` pretty-prints the YAML; `table head dim_user --limit 5` shows the 5-row SCD2 table; `trace tail --lines 3` pretty-prints the last 3 events with indented JSON. **First non-promotion content commit pushed to the resink-core remote at `057a18b`.**

## Per-KR rollup (O1)

| KR | Outcome | Notes |
|---|---|---|
| KR1.1 (new crate) | **PASS** | `crates/resink-cli/Cargo.toml` declares `name = "resink-cli"`, `[[bin]] name = "resink"`, `version = "0.1.0"`. Workspace `Cargo.toml` includes it as 5th member. Deps: clap, serde, serde_json, serde_yaml, arrow (with prettyprint feature), parquet, anyhow 1.x. |
| KR1.2 (clap structure) | **PASS** | Top-level `resink [--workspace <path>] <subcommand>`; 5 subcommands via clap derive. |
| KR1.3 (workspace status) | **PASS** | Output: workspace path; git rev (`f8b5f5da994f723d739e080434ebb894bb0ff3f8`); status (dirty during dev; clean after commit); 5/5 artifact ✓. |
| KR1.4 (verdict latest) | **PASS** | Output: `overall: PASS`, `engine_version: 0.3.0`, per-dim PASS for dim_user + dim_account. `--json` flag exposes raw JSON. |
| KR1.5 (manifest get) | **PASS** | YAML output by default; `--json` flag converts to pretty-printed JSON via serde_yaml → serde_json. |
| KR1.6 (table head) | **PASS** | Uses `parquet::arrow::arrow_reader::ParquetRecordBatchReaderBuilder` + `arrow::util::pretty::pretty_format_batches`. Streaming batch read with `batch_size = limit` so the CLI never holds the full table in memory. Supports `dim_user` and `dim_account`; rejects unsupported dims with a useful error. |
| KR1.7 (trace tail) | **PASS** | Reads `trace.jsonl`, takes last N lines (default 20), pretty-prints each as 2-space-indented JSON. `--raw` flag for unmodified output. Handles malformed lines gracefully (prints verbatim). |
| KR1.8 (workspace resolution) | **PASS** | Resolution order: `--workspace`, `$RESINK_WORKSPACE`, walk-up from cwd looking for `synthetic_tenants/*/workspace/`, useful error if all fail. The walk-up heuristic uses the presence of `manifest.yaml` OR `verdict.json` OR `nodes/` as the workspace indicator. |
| KR1.9 (build green) | **PASS** | `cargo build --release -p resink-cli` exits 0 in 28s clean (incremental ~3s). Binary at `target/release/resink`. `--help` produces expected text. |
| KR1.10 (tests) | **PASS** | 6 unit tests: workspace::status_runs_on_a_directory_without_panicking, verdict::deserialize_minimal_verdict, verdict::deserialize_with_missing_optional_fields, manifest::get_reads_and_prints_yaml, table::unsupported_dim_errors, trace::tail_returns_last_n_lines. `cargo test --release -p resink-cli` → 6 passed; 0 failed. |
| KR1.11 (user-guide.md) | **PASS** | New section "## Power-user CLI (`resink`)" inserted before "## Environment variables". ~110 lines: Install (cargo install), workspace resolution, 5 subcommand sections with sample output, "Out of scope for v1" footer. |
| KR1.12 (push to remote) | **PASS** | Commit `feat(cli): v1 of the resink power-user CLI` at `057a18b`; pushed to `git@github.com:resink-ai/resink-core.git` master. First non-promotion content on the new remote. |
| KR1.13 (parent submodule bump) | **PASS (pending parent commit)** | Parent's `repos/resink-ai/resink-core` submodule pointer updates to `057a18b` in this loop's parent commit. |
| KR1.14 (tenant-isolation) | **PASS** | Final dry-run: `grep -nrE "(resink\|nanofab\|home-cluster\|acme\.ai)" org-os/` returns only the canonical `acme.ai` placeholder. No `org-os/` writes. |

## Build phase signal-of-interest

- **Workspace had to be regenerated.** During the recovery from the prior loop's local-state cleanup (the `git submodule deinit -f` after PR #13 merge), the gitignored workspace artifacts at `synthetic_tenants/closed_loop_v0/workspace/` were wiped. The supervisor's path-deps on `workspace/nodes/dim_*_scd2/` couldn't resolve. **Fix:** ran `uv sync --extra dev` in both `training/orchestrator/` and `sim-farm/` to recreate venvs, then `make mvp-loop` to regenerate the workspace + nodes. `verdict=pass mismatches=0`. **Pattern observation:** after a submodule deinit + re-init, gitignored runtime artifacts need explicit regeneration. Future-loop retro candidate: document this in `docs/user-guide.md § Troubleshooting` or add a `make bootstrap` target that regenerates venvs + workspace in one shot.
- **arrow's `prettyprint` feature wasn't on by default.** The supervisor crate doesn't pretty-print so the feature wasn't needed; resink-cli does. Added `features = ["prettyprint"]` to the resink-cli crate's `arrow` dependency without affecting other workspace members. Clean isolated feature opt-in.
- **anyhow as the first new top-level dep.** The 5 existing crates don't use anyhow (they roll their own error types). Pinned `anyhow = "1"` directly in resink-cli's `Cargo.toml`. Future-loop bookkeeping: if anyhow gets adopted by other crates, lift to workspace deps.
- **Streaming parquet read.** The `table head` impl uses `batch_size = limit` so the reader emits one batch and stops. CLI memory footprint stays small even for large parquets.

## Cross-team coordination

- **SRE** issued a single-paragraph review-ack confirming the CLI's invocation pattern doesn't conflict with the supervisor SOP; recommended a next-loop SOP update to optionally suggest `resink verdict latest` as a friendlier alternative to `cat workspace/verdict.json | jq`. (See SRE exec summary.)
- **Other teams paused** (devops, AE, sim-farm, DE). CLI is read-only on artifacts they don't author or operate.

## Open carries (for next loop)

- **Write commands** (`resink retrain ...`, `resink hot-swap rollback`) — require backend API. Future loop.
- **OAuth + remote API integration** — requires a backend service. Future loop.
- **Homebrew tap** — pre-built binaries for macOS arm64 + amd64 + linux amd64. Future loop after command surface stabilizes.
- **Additional dim_* support in `table head`** — only `dim_user` + `dim_account` in v1; future dims as they land.
- **CI/CD on the new remote** (last-loop retro P2) — still deferred.
- **`Chart.appVersion` ↔ supervisor crate version alignment** — bookkeeping.
- **Cargo MSRV declaration sync** — bookkeeping.
- **Hot-swap correctness test (ADR-2026-05-16-001 step 3)** — slipped twice now; new target loop+4.
- **Document submodule-deinit-then-regenerate pattern** — surfaced this loop; retro candidate.

## Tenant-isolation invariant

Held. Zero `org-os/` writes. All edits in resink-core submodule (`crates/resink-cli/`, `Cargo.toml`, `Cargo.lock`, `docs/user-guide.md`) + parent's submodule pointer bump (next commit) + this OKR + exec summary. Final tenant-isolation grep clean.
{% endraw %}
