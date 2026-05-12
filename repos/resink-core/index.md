---
layout: default
title: "resink-core"
nav_order: 8
has_children: true
render_with_liquid: false
---
# resink-core

resink-core is the Rust Cargo workspace plus Python training-pipeline that hosts the **Nanofab Runtime** (sub-project #1) and the **Nanofab Training Pipeline** (sub-project #2) of the resink.ai product family. It is consumed by other resink.ai teams (sim-farm, data-engineering, devops, sre) and is the primary code surface owned by `teams/application/resink-core/`. The MVP closed loop ships an end-to-end demo where a synthetic tenant's CDC fact streams are routed through orchestrator-generated SCD2-maintainer nodes inside a multi-node supervisor and the resulting per-dim parquets are byte-diffed against a deterministic fixture — `make mvp-loop` exits 0 with `verdict=pass mismatches=0` on the widened 2-dim / 3-fact / 4-shard fixture as of loop 2026-05-23. This repo does not target Spark, the JVM, or a general SQL engine; analytical query is delegated to Trino/DuckDB over the (future) Iceberg CDC sink.

<!-- rit-docs-init:begin auto-generated -->

## Tech stack

- **Rust 2021 edition, MSRV 1.83** — declared in `[workspace.package]` of `Cargo.toml`. Three crates: `nanofab-node-abi` (trait surface), `nanofab-supervisor` (sim-mode driver binary), `nanofab-coordinator` (release-seal validator).
- **Cargo workspace** — resolver = 2, pinned `arrow`/`parquet` 53, `clap` 4, `serde`/`serde_json`/`serde_yaml`, `twox-hash` 1.x (xxHash64 partitioning per DE contract §2 / §2.1, `seed=0`). Release profile uses `lto = "thin"`, `codegen-units = 1`.
- **Python 3.12 (>=3.11)** — `uv`-managed environments under `training/orchestrator/.venv` and `sim-farm/.venv`. Build backend `hatchling`. Orchestrator deps: `pyarrow>=14`, `pyyaml>=6`. Sim-farm deps: `duckdb>=1.0`, `pyarrow>=14`, `pytz>=2024.1`.
- **DuckDB-via-Python** — the sim-farm diff engine (`sim_farm/diff_scd2.py`) reads both fixture and supervisor-output parquets into DuckDB and emits the verdict JSON.
- **Helm 3** — chart skeleton at `deploy/charts/nanofab-supervisor/` (15 files; 8 templates + 2 minikube values files + `Chart.yaml` + `values.yaml` + `Dockerfile` + `Makefile` + `README.md`). `helm lint --strict`, `helm template`, and `helm install --dry-run` all exit 0 as of loop 2026-05-23; minikube smoke is deferred.
- **`claude` CLI (optional)** — the orchestrator dispatches AE's `nanofab:codegen-scd2-node` skill via the local `claude` binary on PATH; when absent or unauthenticated, in-process slot-fill is used instead (see env var below).

## Common commands

Run from `repos/resink-ai/resink-core/` unless noted.

```bash
# Full MVP closed loop (the canonical green-bar gate; multi-dim, multi-shard).
# Default uses in-process slot-fill (no ANTHROPIC_API_KEY required).
cd synthetic_tenants/closed_loop_v0 && make mvp-loop

# Same, but route codegen through the real claude CLI (requires ANTHROPIC_API_KEY
# or an OAuth-authenticated claude install on PATH).
cd synthetic_tenants/closed_loop_v0 && CLAUDE_DISPATCH=1 make mvp-loop

# Remove all generated artifacts (parquets, schema JSONs, trace, verdict,
# workspace/.nanofab/, manifest.yaml, per-node target/, determinism scratch).
cd synthetic_tenants/closed_loop_v0 && make clean

# Build a single crate.
cargo build --release -p nanofab-supervisor
cargo build --release -p nanofab-coordinator

# Supervisor determinism test (whole-DAG, ignored-by-default, slow).
cargo test --release -p nanofab-supervisor --test determinism -- --include-ignored

# Sim-farm test suite (9 tests as of loop 2026-05-23).
cd sim-farm && uv run pytest -q

# Helm chart gates (all three must exit 0).
cd deploy/charts/nanofab-supervisor
helm lint --strict .
helm template release . --values values/minikube-mvp.yaml >/dev/null
helm install release . --values values/minikube-mvp.yaml --dry-run --debug >/dev/null
```

## Repository layout

```
Cargo.toml                # workspace root (members: 3 crates; excludes: synthetic_tenants/, training/, sim-farm/)
Cargo.lock                # checked in for reproducibility
README.md                 # legacy human-facing README (pre-CLAUDE.md; preserved verbatim)
.gitignore                # ignores target/, synthetic_tenants/*/workspace/, generated fixture parquets, .venv/
crates/                   # 3 workspace crates (node-abi, supervisor, coordinator)
training/                 # Python orchestrator (uv-managed); src/orchestrator/{__main__,dispatch,manifest,codegen_slotfill}.py
sim-farm/                 # Python Mode-A diff engine; sim_farm/diff_scd2.py + 9 pytest cases under tests/
synthetic_tenants/        # tenant test rigs; closed_loop_v0/ holds fixture + Makefile + workspace/ (gitignored)
deploy/                   # deployment surfaces; charts/nanofab-supervisor/ is the Helm skeleton
target/                   # cargo build output (gitignored)
```

## Where to find more

- **Architecture & system shape** → `docs/architecture.md`
- **Domain concepts & vocabulary** → `docs/concepts.md`
- **How to develop, build, run, deploy** → `docs/user-guide.md`
- **Module catalog (what each top-level module does)** → `docs/module-catalog.md`
- **Source code** — read directly only after the above are insufficient. Start at `crates/nanofab-supervisor/src/main.rs` (the MVP green-bar binary) or `synthetic_tenants/closed_loop_v0/Makefile` (the end-to-end driver).
- **Canonical specs (in parent newbase repo)** — `docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md`, `docs/superpowers/specs/2026-05-10-nanofab-training-pipeline-design.md`.
- **ABI Option A deviation ADR (in parent newbase repo)** — `board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md`.
- **SRE runbook for supervisor-failed validation (in parent newbase repo)** — `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md`.

<!-- rit-docs-init:end -->
