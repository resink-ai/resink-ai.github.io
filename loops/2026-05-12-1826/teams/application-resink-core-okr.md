---
layout: default
title: application-resink-core OKR — 2026-05-12-1826
date: 2026-05-12
status: active
type: okr
loop: 2026-05-12-1826
owner: teams/application/resink-core
grand_parent: Loops
parent: Loop 2026-05-12-1826
nav_order: 10
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-12
  status: active
  loop: 2026-05-12-1826
  links: parent: board/okrs/2026-05-12-1826-ceo-brief.md
-->
{% raw %}

# Resink Core OKR — 2026-05-12 (loop 2026-05-12-1826)

## Context

The first user-facing surface in the company. Per CEO brief 2026-05-12-1826 § O1: ship v1 of the `resink` CLI as a new Rust crate `crates/resink-cli/` in the resink-core submodule. 5 read-only subcommands; reads filesystem workspace artifacts; distributed via `cargo install` for v1. First non-promotion content to land on the new resink-core remote.

The CLI is sub-project #5 (Product UX) per ORG.md — the spec at `docs/superpowers/specs/2026-05-10-nanofab-product-ux-design.md` § 3.2 names it as the power-user CLI surface. This loop ships the read-only slice; future loops add write commands + OAuth + remote API.

## Objectives

### O1: Ship `resink-cli` v1 + first content push to the new submodule remote

source: ceo-brief

Why it matters: First user-facing surface in the company. Validates the multi-crate Rust workspace can host both the runtime (supervisor + plugins) AND user-facing binaries cleanly. First development commit on the freshly-bootstrapped resink-core remote — exercises the post-promotion workflow end-to-end.

**Key results**

- KR1.1: New crate at `crates/resink-cli/` with `Cargo.toml` declaring `name = "resink-cli"`, `[[bin]] name = "resink"`, `version = "0.1.0"`. Workspace `Cargo.toml` updated to include `crates/resink-cli` as a 5th workspace member. Dependencies: `clap` 4 (workspace), `serde_json` (workspace), `serde_yaml` (workspace), `arrow` 53 (workspace), `parquet` 53 (workspace), `anyhow` 1.x (new pin).
- KR1.2: CLI structure via clap derive macros. Top-level: `resink [--workspace <path>] <subcommand>`. Subcommands: `workspace status`, `verdict latest`, `manifest get`, `table head <dim> [--limit N]`, `trace tail [--lines N]`.
- KR1.3: `workspace status` subcommand — prints workspace path + git rev (best-effort, via `git rev-parse` if cwd is inside a repo; otherwise "<not in a git repo>"); prints clean/dirty (best-effort via `git status --porcelain`).
- KR1.4: `verdict latest` subcommand — reads `<workspace>/verdict.json`; pretty-prints with: top-line `overall_pass` colored (green/red on TTY); per-dim verdicts with names + pass status + mismatch counts; engine version. `--json` flag for raw JSON output.
- KR1.5: `manifest get` subcommand — reads `<workspace>/manifest.yaml`; pretty-prints YAML or `--json` for JSON output.
- KR1.6: `table head <dim> [--limit N]` subcommand — reads `<workspace>/<dim>_output.parquet` via `parquet::file::reader::SerializedFileReader`; uses `arrow::util::pretty::pretty_format_batches` for the tabular output. Default limit: 10 rows. Supported dims: `dim_user`, `dim_account` (any other surfaces error with a useful message naming the supported set).
- KR1.7: `trace tail [--lines N]` subcommand — reads `<workspace>/trace.jsonl`; prints last N lines (default 20). Each line is JSON; pretty-print as 2-space-indented JSON. `--raw` flag for raw output.
- KR1.8: `--workspace <path>` global flag. Resolution: (a) explicit `--workspace`; (b) `$RESINK_WORKSPACE` env var; (c) walk up from cwd looking for any `synthetic_tenants/*/workspace/` directory; (d) error with named alternatives.
- KR1.9: Build green: `cargo build --release -p resink-cli` exits 0. Binary at `target/release/resink`. `target/release/resink --help` produces expected top-level help text.
- KR1.10: Tests: at least one test per subcommand. `cargo test -p resink-cli` exits 0.
- KR1.11: `docs/user-guide.md` gains a new "## Power-user CLI (`resink`)" section. Install instructions (`cargo install --path crates/resink-cli`); 5 subcommands with sample output; `--workspace` resolution explanation; cross-link from "Common commands" if natural. ~60 lines.
- KR1.12: Commit on resink-core master + push to `git@github.com:resink-ai/resink-core.git`. Commit message: `feat(cli): v1 of the resink power-user CLI`. First non-promotion content push on the new remote.
- KR1.13: Parent newbase's `repos/resink-ai/resink-core` submodule pointer bumps to the new HEAD post-push.
- KR1.14: Tenant-isolation: zero `org-os/` writes. All edits in resink-core submodule + this OKR + this loop's exec summary.

**Tasks**

- [ ] Scaffold the resink-cli crate; add to workspace.
- [ ] Implement clap structure.
- [ ] Implement each subcommand.
- [ ] Add tests.
- [ ] Run `cargo build --release -p resink-cli`; verify binary works.
- [ ] Test each subcommand against the closed_loop_v0/workspace fixture.
- [ ] Document in user-guide.md.
- [ ] Commit on resink-core master + push to new remote.
- [ ] Bump parent submodule pointer.
- [ ] Tenant-isolation dry-run.

## Risks

- **Parquet pretty-print column widths.** `arrow::util::pretty::pretty_format_batches` uses fixed column widths; long string columns may wrap awkwardly. v1 ships as-is; future loops can swap for a custom formatter.
- **Workspace resolution heuristic edge cases.** If the operator is `cd`'d outside any `synthetic_tenants/*/` ancestor, the walk-up fails. The error message names `$RESINK_WORKSPACE` and `--workspace <path>` as remedies.
- **Cross-platform behavior.** TTY colorization needs `is-terminal` or similar crate; or use `colored`/`termcolor`. v1 may ship without colorization to keep dependency count low; pretty-print still works via plain text.
- **First remote push is direct to master.** No PR workflow on the new remote yet (deferred per 2026-05-12-1254 retro P2). Force-push recovery acceptable for a one-contributor repo.

## Out of scope

- Write commands (retrain, hot-swap rollback).
- OAuth + remote API integration.
- Homebrew tap distribution.
- `workspace diff` (requires versioned workspace storage).
- Additional dims beyond `dim_user` and `dim_account`.
- CI/CD setup on the new remote.
- Cargo MSRV bump (bookkeeping).
{% endraw %}
