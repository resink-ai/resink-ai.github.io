---
layout: default
title: "resink-core: module-catalog"
date: 2026-05-30
status: active
type: doc
owner: teams/application/resink-core
parent: resink-core (overview)
---

<!-- original-frontmatter:
  type: doc
  owner: teams/application/resink-core
  date: 2026-05-30
  status: active
-->
{% raw %}

# Module Catalog — resink-core

One entry per top-level code module. Each entry names the path, the one-sentence purpose, who depends on it (resink-core internal teams + named peer teams via cross-team contracts), and the entry point file path a future agent should open first.

<!-- rit-docs-init:begin auto-generated -->

## `crates/nanofab-node-abi/`

- **Path:** `crates/nanofab-node-abi/`
- **Purpose:** Published stable trait surface for nanofab DAG nodes — `Node`, `NodeCtx<K,R>`, `Event`, `Scd2Row<R>`, `Op`, `FieldValue`, `NodeError` — kept field-for-field equivalent with AE's codegen template so generated crates can switch from inline to upstream when AE flips the template.
- **Who depends on it:** resink-core's `nanofab-supervisor` (declared as a workspace dependency in `Cargo.toml`); AE owns the codegen template that currently inlines these types verbatim and will eventually `use nanofab_node_abi::*`; the SRE runbook references the failure-mode shapes (`NodeError` variants) for triage.
- **Entry point:** `crates/nanofab-node-abi/src/lib.rs`

## `crates/nanofab-supervisor/`

- **Path:** `crates/nanofab-supervisor/`
- **Purpose:** The `--mode=sim` driver binary that reads a tenant `manifest.yaml`, loads fact parquets, routes events by `Event.table` to the two statically-linked codegen-output nodes, runs each through 4 logical shard handlers (`xxHash64(seed=0) % shard_count`), writes per-dim output parquets and a combined `trace.jsonl`.
- **Who depends on it:** resink-core's `nanofab-coordinator` validates its output; **sim-farm** consumes the per-dim output parquets against its fixtures and emits the verdict (cross-team contract — see `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`); **DE** owns the `xxHash64(seed=0)` partition contract and the in-memory event-source contract this binary implements; **DevOps** wraps this binary in the Helm chart at `deploy/charts/nanofab-supervisor/`; **SRE** owns the failed-validation runbook at `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md`.
- **Entry point:** `crates/nanofab-supervisor/src/main.rs`

## `crates/nanofab-coordinator/`

- **Path:** `crates/nanofab-coordinator/`
- **Purpose:** Release-seal validator — `publish-dag --workspace=<path>` reads `manifest.yaml` + `.nanofab/release_seal.json` and validates that stages 1 (compile) + 2 (smoke) are `passed` per node (stages 3 + 4 may be `skipped`); prints `accepted, version=<seal.version>` to stdout and exits 0 on green.
- **Who depends on it:** resink-core's `make publish-dag` recipe gates step 5 of the MVP closed loop on this binary's exit code; the `training/orchestrator/` writes the seal shape this binary parses; **DevOps** treats the `accepted, version=...` line as the deploy-gate signal.
- **Entry point:** `crates/nanofab-coordinator/src/main.rs`

## `training/orchestrator/`

- **Path:** `training/orchestrator/`
- **Purpose:** Python (`uv`-managed) orchestrator that dispatches AE's `nanofab:codegen-scd2-node` skill (or in-process slot-fills when `--skip-dispatch`), writes the per-tenant `manifest.yaml` + `.nanofab/release_seal.json`, runs per-node `cargo build` + `cargo test`, and flips seal stages — the training-pipeline entry point for the closed loop.
- **Who depends on it:** resink-core's `make orchestrate` recipe invokes this as `python -m orchestrator`; **AE** owns the `nanofab:codegen-scd2-node` skill this orchestrator dispatches (cross-team contract — see `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/DISPATCH.md`); the `nanofab-coordinator` parses the seal this orchestrator writes.
- **Entry point:** `training/orchestrator/src/orchestrator/__main__.py`

## `synthetic_tenants/closed_loop_v0/`

- **Path:** `synthetic_tenants/closed_loop_v0/`
- **Purpose:** The closed-loop test rig — a self-contained synthetic tenant carrying the fixture generators (`fixtures/generate.py`, `fixtures/derive_facts.py`), the seven-step end-to-end driver (`Makefile`), and the gitignored runtime `workspace/`; the MVP green-bar gate runs from here.
- **Who depends on it:** resink-core's `make mvp-loop` integration gate; **sim-farm** depends on the dim-fixture parquets as ground-truth; **DE** depends on the fact-stream parquet shapes matching their in-memory event-source contract; the `training/orchestrator/` consumes the schema JSONs and writes into `workspace/`.
- **Entry point:** `synthetic_tenants/closed_loop_v0/Makefile`

## `sim-farm/`

- **Path:** `sim-farm/`
- **Purpose:** Python (`uv`-managed) Mode-A SCD2 batch diff engine — takes one-or-more `(fixture.parquet, output.parquet, dim_table)` triples, runs a DuckDB-backed row diff against the engine 0.2.0 schema (`(user_id, valid_from)` join key, `(email, country, valid_to, is_current)` payload), and emits a multi-dim verdict JSON; the 9-test pytest suite (`uv run pytest -q`) is the engine's own gate.
- **Who depends on it:** resink-core's `make verdict` recipe invokes this as `python -m sim_farm.diff_scd2`; **sim-farm** team owns this engine and the verdict contract (`teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`); resink-core's `dim_account` fixture is shaped isomorphically to dim_user's join+payload columns to satisfy this engine's 0.2.0 hard-coded shape (named deviation in `docs/architecture.md`).
- **Entry point:** `sim-farm/sim_farm/diff_scd2.py`

## `deploy/charts/nanofab-supervisor/`

- **Path:** `deploy/charts/nanofab-supervisor/`
- **Purpose:** Helm 3 chart skeleton wrapping the `nanofab-supervisor` binary for MVP single-tenant, single-shard, in-memory sim-mode deployment — 15 files (8 templates: `_helpers.tpl`, `configmap.yaml`, `deployment.yaml`, `role.yaml`, `rolebinding.yaml`, `secret.yaml`, `service.yaml`, `serviceaccount.yaml`; 2 minikube values files: `minikube-mvp.yaml`, `minikube-bluegreen.yaml`; plus `Chart.yaml`, `values.yaml`, `Dockerfile`, `Makefile`, `README.md`); `helm lint --strict`, `helm template`, `helm install --dry-run` all exit 0; minikube smoke is deferred to loop 2026-06-06.
- **Who depends on it:** **DevOps** owns and maintains this chart; resink-core ships the supervisor binary the chart wraps; **SRE** consumes the deployment surface (signal handling, env-var convention, port convention) for runbook authoring; future-loop production deploy targets read this chart as the canonical wrapping shape.
- **Entry point:** `deploy/charts/nanofab-supervisor/Chart.yaml`

<!-- rit-docs-init:end -->
{% endraw %}
