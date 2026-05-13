---
layout: default
title: "resink-core: user-guide"
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
{% raw %}

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

## Plugin loading modes

The supervisor crate ships two mutually-exclusive Cargo features that select **how** the per-tenant SCD2-maintainer node code is loaded into the supervisor binary:

- **`static-plugins`** (default) — the codegen-output crates under `synthetic_tenants/<tenant>/workspace/nodes/<dim>_scd2/` are linked into the supervisor binary as Cargo path-dependencies. The supervisor is rebuilt per-tenant per-DAG-version. This is the original ABI Option A path (named in ADR-2026-05-16-001) and remains the default for `make mvp-loop`.
- **`dlopen-plugins`** — the supervisor links `libloading` and resolves `nanofab_node_new` / `nanofab_node_process` / `nanofab_node_drop` via `libloading::Library::open(...)` against an external cdylib. The canonical artifact source is the in-tree `crates/nanofab-plugin-dim-user/` workspace member, built as `target/release/libnanofab_plugin_dim_user.{so,dylib}`. ADR-2026-05-16-001 step 2 (closed loop 2026-06-13) end-to-end exercises this path; the supervisor wraps the loaded symbols in a safe `PluginNode` Rust handle (see `crates/nanofab-supervisor/src/plugin_loader.rs`).

To run the closed loop under the dlopen path, use the dedicated Make target (added 2026-06-13):

```bash
cd repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0
make mvp-loop-dlopen
```

This rebuilds the supervisor under `--no-default-features --features dlopen-plugins`, builds the in-tree `nanofab-plugin-dim-user` cdylib, then runs steps 5-7 of the closed loop (publish-dag, run supervisor, verdict). The verdict.json shape is identical to the default `make mvp-loop` run: `overall_pass: true`, `engine_version: 0.3.0`, both per-dim `pass: true` with `mismatch_count: 0`. To exercise the FFI roundtrip end-to-end without the full closed loop, run the dlopen integration test directly:

```bash
cargo test --features dlopen-plugins -p nanofab-supervisor --test dlopen_integration
```

The integration test (`crates/nanofab-supervisor/tests/dlopen_integration.rs`) builds `nanofab-plugin-dim-user` via `cargo build --release -p nanofab-plugin-dim-user`, opens the resulting cdylib via `libloading::Library::new`, looks up the three C-ABI symbols, fires a known `dim_user` insert event, and asserts `NANOFAB_NODE_OK = 0` from the FFI roundtrip.

To choose feature configurations explicitly when building the supervisor:

```bash
# Static-plugins exclusively (rollback configuration; matches MVP baseline).
cargo build --release --no-default-features --features static-plugins -p nanofab-supervisor

# Dlopen-plugins exclusively (canonical going-forward path per ADR-2026-05-16-001).
cargo build --release --no-default-features --features dlopen-plugins -p nanofab-supervisor

# Build the dlopen artifact alongside (workspace target dir).
cargo build --release -p nanofab-plugin-dim-user
```

Hot-swap (loading a new plugin version into a running supervisor without restart) is the joint AE + resink-core deliverable for ADR-2026-05-16-001 step 3 (target loop 2026-06-20).

## Deploying to the home cluster

First exercised at loop 2026-05-12-0645. The supervisor image (`nanofab-supervisor:0.3.0`) is built from this repo via the chart-owned Dockerfile, distributed to the 4 home-cluster nodes via `ctr import`, and installed via the Helm chart at `deploy/charts/nanofab-supervisor/`. The full SOP (prerequisites, install / verify / delete, failure modes) lives in the SRE runbook at `teams/platform/sre/runbooks/nanofab-supervisor-deployment.md`. Headline recipe:

```bash
# 1. Build (cross-arch from macOS/arm64 to linux/amd64).
docker buildx build --platform linux/amd64 --load \
  -f deploy/charts/nanofab-supervisor/Dockerfile \
  -t nanofab-supervisor:0.3.0 .

# 2. Save and distribute to all 4 nodes (k8s-0..k8s-3).
docker save nanofab-supervisor:0.3.0 -o /tmp/nanofab-supervisor.tar
for ip in 192.168.1.162 192.168.1.160 192.168.1.163 192.168.1.164; do
  scp -i ~/.ssh/id_ed25519_hwmgmt /tmp/nanofab-supervisor.tar lsj@$ip:/tmp/
  ssh -i ~/.ssh/id_ed25519_hwmgmt lsj@$ip \
    'sudo ctr -n=k8s.io images import /tmp/nanofab-supervisor.tar'
done

# 3. Install + verify via kubectl logs (Deployment kind makes kubectl cp
#    unreliable for one-shot supervisor; logs carry the success marker).
helm install nanofab-supervisor-mvp deploy/charts/nanofab-supervisor \
  --values deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml \
  --namespace nanofab --create-namespace
kubectl logs -n nanofab -l app.kubernetes.io/name=nanofab-supervisor --tail=10
# Expected: "[supervisor] [...] wrote N rows..." x 2, "supervisor: ok".

# 4. Delete.
helm uninstall nanofab-supervisor-mvp -n nanofab
kubectl delete namespace nanofab
```

**Image-tag convention:** the image tag aligns with `crates/nanofab-supervisor/Cargo.toml` `[package].version` (currently `0.3.0`). When the supervisor crate's version bumps, rebuild + redistribute with the new tag and update `deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml`'s `image.tag`. The chart's `Chart.appVersion` is informational; the values' `image.tag` is the authoritative pull target.

**Deployment vs Job kind:** the current chart renders a `kind: Deployment` resource. The MVP supervisor exits 0 after one pass, so the Deployment cycles between Running and BackOff. This is the expected MVP shape; a future loop ships a `kind: Job` variant for clean one-shot semantics + reliable `kubectl cp` of `/workspace/trace.jsonl`. Until then, treat `kubectl logs` as the canonical verification surface.

**No image registry yet:** the per-node `ctr import` distribution is the current bootstrap path. A future home-cluster loop adds an in-cluster registry (per the home-cluster roadmap Phase 1); until then, every image rebuild re-distributes.

## Power-user CLI (`resink`)

A Rust binary that gives operators a friendly read-only view of the local filesystem workspace produced by `make mvp-loop`. Sub-project #5 (Product UX) per the spec at `docs/superpowers/specs/2026-05-10-nanofab-product-ux-design.md` § 3.2. Shipped v1 at loop 2026-05-12-1826.

### Install

```bash
cargo install --path crates/resink-cli
# binary lands at $CARGO_HOME/bin/resink (typically ~/.cargo/bin/resink)
```

Or for in-tree use without install:

```bash
cargo build --release -p resink-cli
# binary at target/release/resink
```

### Workspace resolution

`resink` needs to locate a workspace directory (the one `make mvp-loop` writes to: `synthetic_tenants/closed_loop_v0/workspace/`). Resolution order:

1. Explicit `--workspace <path>` flag.
2. `$RESINK_WORKSPACE` env var.
3. Walk up from cwd looking for `synthetic_tenants/<tenant>/workspace/`.
4. Error with a helpful message naming the alternatives.

### Subcommands (v1)

All v1 commands are read-only.

#### `resink workspace status`

Prints the workspace path, its git revision (if available), and clean/dirty status, plus a checklist of expected artifacts.

```
$ resink workspace status
workspace: /.../synthetic_tenants/closed_loop_v0/workspace
git rev: f8b5f5da994f723d739e080434ebb894bb0ff3f8
status: clean

artifacts:
  ✓ manifest.yaml
  ✓ verdict.json
  ✓ trace.jsonl
  ✓ dim_user_output.parquet
  ✓ dim_account_output.parquet
```

#### `resink verdict latest [--json]`

Pretty-prints the latest `verdict.json`. `--json` outputs the raw JSON for `jq`-friendly piping.

```
$ resink verdict latest
overall: PASS
engine_version: 0.3.0

per-dim:
  PASS dim_user             mismatches=0
  PASS dim_account          mismatches=0
```

#### `resink manifest get [--json]`

Pretty-prints `manifest.yaml`. `--json` converts to JSON.

```
$ resink manifest get
version: 1
shard_count: 4
dags:
  - name: closed_loop_dag
    nodes:
      - id: dim_user_scd2
        table: dim_user
        pk: [user_id]
        ...
```

#### `resink table head <dim> [--limit N]`

Prints the first N rows of a dim-table parquet (default limit: 10). Supported dims in v1: `dim_user`, `dim_account`.

```
$ resink table head dim_user --limit 3
+---------+-------------------+------------+---------+---------------+---------------+
| country | email             | is_current | user_id | valid_from    | valid_to      |
+---------+-------------------+------------+---------+---------------+---------------+
| US      | alice@example.com | false      | u-001   | 1715300000000 | 1715700000000 |
| CA      | alice@example.com | true       | u-001   | 1715700000000 |               |
| GB      | bob@example.com   | false      | u-002   | 1715300000000 | 1715400000000 |
+---------+-------------------+------------+---------+---------------+---------------+

(3 rows shown)
```

#### `resink trace tail [--lines N] [--raw]`

Prints the last N lines of `trace.jsonl` (default: 20), pretty-formatted JSON unless `--raw`.

```
$ resink trace tail --lines 1
{
  "after": { ... },
  "before": null,
  "event_id": "fact_account_open:a-015:...",
  "key": { "account_id": "a-015" },
  "node_id": "dim_account_scd2_maintainer",
  ...
}
```

### Out of scope for v1

Write commands (`resink retrain ...`, `resink hot-swap rollback`), remote commands + OAuth, Homebrew tap distribution, additional dims beyond `dim_user`/`dim_account` — all future-loop work. v1 ships the smallest scope that gives a real user experience without requiring a backend API.

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

## Continuous integration

GitHub Actions workflow at `.github/workflows/ci.yml` (added 2026-05-13, loop `2026-05-13-1022`) runs on every PR and every push to master. Two jobs gate merges:

- **cargo** — `cargo build --workspace --release` + `cargo test --workspace --release`, then a second pass with `--no-default-features --features dlopen-plugins` for the supervisor crate so the `hot_swap_correctness` integration test exercises the real libloading path.
- **helm** — `helm lint --strict`, `helm template`, and `helm install --dry-run --debug` against `deploy/charts/nanofab-supervisor` with `values/home-cluster-mvp.yaml`.

Branch protection on master requires both jobs pass before merge (configured via `gh api`; single-status-check rule, no required reviewer). Direct pushes to master are rejected. To merge, open a PR; the workflow runs automatically on the PR branch; once both jobs are green, the PR is mergeable.

If a job fails:

- **cargo build / test failure.** Inspect the run log; the workspace builds clean as of the bootstrap loop, so a failure points at a regression in the changed code. The `dlopen-plugins` pass requires both v1 + v2 cdylib codegen artifacts to exist on disk (see `crates/nanofab-supervisor/tests/hot_swap_correctness.rs` for the setup).
- **helm lint failure.** Most common cause: `values.schema.json` schema enforcement (the chart's `tenant` field has `minLength: 1`; the CI passes `-f values/home-cluster-mvp.yaml` so the lint sees the actual tenant value). To reproduce locally: `helm lint --strict -f deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml deploy/charts/nanofab-supervisor`.
- **helm template / dry-run failure.** Typically a template-render bug or a Kubernetes API schema rejection. Reproduce locally with the same `-f` arguments the workflow uses.

<!-- rit-docs-init:end -->
{% endraw %}
