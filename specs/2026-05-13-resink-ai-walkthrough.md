---
layout: default
title: "Spec: 2026-05-13-resink-ai-walkthrough"
date: 2026-05-13
status: active
type: spec
owner: board
parent: Design specs
---

<!-- original-frontmatter:
  type: spec
  owner: board
  date: 2026-05-13
  status: active
-->
{% raw %}

# Resink.ai — live walkthrough

> A 30-minute reader-followable tour of what we've built, written for someone evaluating the platform for the first time. Companion to the [2026-05-13 capabilities report](../../../board/reports/2026-05-13-resink-ai-capabilities.html), which is the higher-level "what runs today" view. This document is the **how to verify it yourself** view.

---

## 1. What is Resink.ai? (3-minute read)

Resink.ai is an **AI-managed data-pipeline platform**. You give us fact-data samples (or a description of the dim tables you need); the system uses an LLM-driven orchestrator to generate the per-dim Rust maintainer code, compiles + tests it, runs it under a multi-tenant Rust supervisor, and deploys it to a real Kubernetes cluster. The output is a per-tenant streaming-data pipeline whose behavior is reproducible against its specification and whose every run produces a verifiable `verdict.json`.

The differentiator: it's not "use AI to write code." It's **AI-generated code under a deterministic, verifiable, multi-tenant runtime**. The generated Rust is byte-identical for identical inputs; the runtime is Rust (no JVM, no GC); the verdict file is the operator's source of truth.

The product surface today is concrete:

- A Rust workspace at `repos/resink-ai/resink-core/`: 5 crates including a supervisor binary, a coordinator, and a user-facing CLI.
- A Python training pipeline that drives codegen and verdict-emission.
- A Helm chart that's been `helm install`-ed on a real 4-node Kubernetes cluster.
- A marketplace of 4 cross-platform plugins (Claude Code / Codex / Gemini / OpenCode), one of which is the codegen skill itself.
- An org-os process that authors the company's own operations — 24 ADRs, 12+ executive loops, all reviewable in the gitbook at `blog.resink.ai`.

If you want to verify anything in this document, the rest of the walkthrough shows you how.

---

## 2. Run it yourself (operator recipe)

This recipe takes a clean clone of `resink-ai/newbase` to a green-bar `verdict=pass mismatches=0` end-to-end run on a developer laptop. **No cloud credentials required**; **no LLM API key required for the default path** (the orchestrator's `--skip-dispatch` flag uses in-process deterministic slot-fill).

### Prerequisites

- A POSIX shell (macOS or Linux). The maintainers test on macOS Darwin 24+; Linux works.
- `git` 2.30+ for submodule support.
- `cargo` (Rust 1.83+; we ship pinned to 1.85 for the production Docker image).
- `uv` (the Python venv/package tool). Install via `curl -LsSf https://astral.sh/uv/install.sh | sh`.
- `make`.

### Recipe

```bash
# 1. Clone (with submodules — resink-core, resink-marketplace, gitbook).
git clone --recurse-submodules <newbase-url>
cd newbase

# 2. Bootstrap the Python venvs the training pipeline needs.
cd repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0
make bootstrap

# 3. Run the closed loop. This:
#    (i)   generates the dim_user + dim_account fixtures (deterministic, seeded);
#    (ii)  derives the three fact parquets (sign_up, profile_update, account_open);
#    (iii) dispatches the codegen skill (in-process slot-fill by default;
#          set CLAUDE_DISPATCH=1 with ANTHROPIC_API_KEY for real LLM dispatch);
#    (iv)  cargo builds + cargo tests each codegen-output crate;
#    (v)   cargo builds the supervisor binary against the per-dim path-deps;
#    (vi)  publishes the DAG via the coordinator;
#    (vii) runs the supervisor against the fact streams;
#    (viii) runs the sim-farm diff engine on the supervisor output vs the fixture;
#    (ix)  emits verdict.json.
make mvp-loop
```

### Expected output (last line of stdout)

```
verdict=pass mismatches=0
```

Inspect `workspace/verdict.json` for the structured result. The `overall_pass: true` line is the operator-readable promise; the `verdicts[]` array contains per-dim records.

### Useful one-liners after `make mvp-loop` succeeds

```bash
# Inspect the workspace via the resink CLI:
cargo run --release -p resink-cli -- workspace status
cargo run --release -p resink-cli -- verdict latest
cargo run --release -p resink-cli -- manifest get
cargo run --release -p resink-cli -- table head dim_user --limit 10
cargo run --release -p resink-cli -- trace tail --limit 50

# Diff your dim_user output against the fixture:
duckdb -c "SELECT * FROM read_parquet('workspace/dim_user_output.parquet') LIMIT 5"

# Take a snapshot of the cluster your kubectl context points at:
# (Requires kubectl + python3; saves a self-contained HTML report.)
python3 repos/resink-ai/resink-marketplace/plugins/observability/skills/cluster-snapshot/generate.py \
  --output-path /tmp/cluster-snapshot.html
open /tmp/cluster-snapshot.html
```

### Recovery if something goes wrong

If you ran `git submodule deinit -f` (which is the recovery dance for some merge-conflict scenarios), the gitignored Python venvs and codegen output crates are wiped. Recovery is one command:

```bash
cd repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0
make bootstrap        # regenerates the venvs
make mvp-loop         # regenerates the codegen output + runs the loop
```

The full troubleshooting guide is at `repos/resink-ai/resink-core/docs/user-guide.md` § "Troubleshooting."

---

## 3. The closed loop, narrated

The thing you just ran is a 7-step pipeline. Each step is an independent piece you can inspect, modify, or replace.

### Step 1: Fixture generation

`fixtures/generate.py` produces a deterministic, seeded dim_user + dim_account fixture pair (parquet). The fixture is the customer-shaped "truth" the supervisor's output will be diffed against.

**Read:** `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/fixtures/generate.py`

### Step 2: Fact derivation

`fixtures/derive_facts.py` derives three fact streams from the fixture: `fact_sign_up`, `fact_profile_update`, `fact_account_open`. The supervisor consumes these in `--mode=sim`.

**Read:** `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/fixtures/derive_facts.py`

### Step 3: Codegen dispatch (the AI step)

The Python orchestrator dispatches the `nanofab:codegen-scd2-node` Claude Code skill once per dim. The skill takes a `(schema_json, pattern_name, output_dir)` triple and writes `Cargo.toml` + `src/lib.rs` + `STATUS.json` into the output directory. The default path uses in-process deterministic slot-fill (no API key); `CLAUDE_DISPATCH=1` opts in to LLM-driven dispatch.

The generated Rust is **byte-identical** for identical `(schema, pattern_name)` inputs — the content-hash dedup property the runtime relies on.

**Read:**
- Orchestrator: `repos/resink-ai/resink-core/training/orchestrator/src/orchestrator/`
- Codegen skill: `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/`
- The generated output (after a run): `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/workspace/nodes/dim_user_scd2/src/lib.rs`

### Step 4: Compile + test the generated crates

The orchestrator runs `cargo build --release` + `cargo test` on each generated crate. Failures write `STATUS.json` with `status: error` and the cargo stderr; no DAG is published until every per-dim crate is green.

### Step 5: Coordinator publishes the DAG

`cargo run --release --bin nanofab-coordinator -- publish-dag --workspace=...` validates the release seal and writes the DAG-ready signal. The supervisor only starts when the seal is valid.

**Read:** `repos/resink-ai/resink-core/crates/nanofab-coordinator/`

### Step 6: Supervisor processes events

`cargo run --release --bin nanofab-supervisor -- --mode=sim` reads the fact-event parquets, routes events to per-dim node instances across 4 logical shards using `xxh64(key_bytes, seed=0)`, and emits `dim_*_output.parquet` + a full `trace.jsonl` of lifecycle events.

The supervisor supports two plugin-loading modes:

- `--features static-plugins` (default): per-dim node crates are Cargo path-deps; the supervisor binary is rebuilt per-tenant.
- `--features dlopen-plugins`: per-dim node crates are loaded as cdylibs via `libloading::Library`; hot-swap-capable.

**Read:**
- Supervisor entrypoint: `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs`
- Plugin loader: `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/plugin_loader.rs`
- Per-dim node modules: `crates/nanofab-supervisor/src/nodes.rs`

### Step 7: Diff engine emits the verdict

`sim-farm/sim_farm/diff_scd2.py` reads both the fixture and the supervisor's output into DuckDB, byte-compares them, and emits per-dim PASS/FAIL with mismatch records (`missing`, `extra`, `diverged` types).

**Read:**
- Diff engine: `repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py`
- Test suite: `repos/resink-ai/resink-core/sim-farm/tests/test_diff_scd2_smoke.py` (9/9 passing)
- Verdict shape: `repos/resink-ai/resink-core/sim-farm/sim_farm/verdict.py`

The verdict file is the operator's source of truth for whether the pipeline behaved correctly.

---

## 4. The components in one sentence each

If you have 5 minutes and want a code tour, here's the navigation:

| Component | One sentence | Path |
|---|---|---|
| Runtime (Rust workspace) | 5 crates that together are the per-tenant data pipeline | `repos/resink-ai/resink-core/crates/` |
| Supervisor | The driver binary; routes events, loads plugins, emits the trace | `crates/nanofab-supervisor/` |
| Codegen output (per dim) | The Rust crate AE's skill produces for one SCD2-maintainer node | `crates/nanofab-plugin-dim-user/src/lib.rs` (reference variant) |
| Coordinator | Validates the release seal + signals DAG ready | `crates/nanofab-coordinator/` |
| Resink CLI | The `resink` power-user binary | `crates/resink-cli/` |
| Training pipeline (Python) | The orchestrator + the diff engine | `training/orchestrator/`, `sim-farm/` |
| Synthetic tenants | End-to-end driver Makefile + fixtures | `synthetic_tenants/closed_loop_v0/` |
| Helm chart | Per-tenant cluster deployment | `deploy/charts/nanofab-supervisor/` |
| Marketplace | 4 cross-platform plugins | `repos/resink-ai/resink-marketplace/plugins/` |
| Codegen skill | The LLM-driven codegen entrypoint | `plugins/nanofab/skills/codegen-scd2-node/` |
| Observability skill | The cluster-snapshot HTML generator | `plugins/observability/skills/cluster-snapshot/` |
| Org-os | The internal operations process | `org-os/` (rituals, conventions, playbooks) |

---

## 5. How the company operates: the org-os process (200 words)

The company runs on a structured operations process called **org-os**, scaffolded over three engineering loops in May–June 2026 (ADR-2026-05-09-001's bottom-up flow, 17 tasks across three bundles).

Every two-week-equivalent slice of work is a **loop**:

1. The **CEO brief** opens the loop with the objectives, sizing, and decisions. (`board/okrs/<loop-id>-ceo-brief.md`)
2. **Team OKRs** decompose the brief into per-team key results. (`teams/<layer>/<team>/okrs/<loop-id>-team-okr.md`)
3. **Build phase** lands code; the team OKR is updated in place with status.
4. **Team exec summaries** roll up what shipped. (`teams/<layer>/<team>/exec-summaries/<loop-id>.md`)
5. **CEO consolidation** rolls up all team summaries into the company exec summary. (`board/exec-summaries/<loop-id>.md`)
6. The **retro** classifies evolution proposals and surfaces what to address next. (`board/retros/<loop-id>-ceo-retro.md`)

Architectural changes go through an **ADR** (Architectural Decision Record) under `board/decisions/`. Process changes go through the org-os evolution path (an ADR + a ratified playbook). Every artifact is dated, owned, status-tracked, and tenant-isolated.

The complete current state — every brief, exec, retro, ADR, action ticket, and capability report — is published to `blog.resink.ai` by the publish-to-gitbook script.

---

## 6. The home cluster (real Kubernetes)

The Helm chart at `deploy/charts/nanofab-supervisor/` has been `helm install`-ed on a real 4-node Kubernetes cluster (a "home cluster" the maintainers operate). The first real deploy (loop 2026-05-12-0645) surfaced two latent chart bugs that all the dry-run gates (helm lint, helm template, helm install --dry-run) had passed cleanly — the value of exercising the actual runtime is real. Both bugs were fixed in flight and became failure-mode entries in the SRE deployment runbook.

The cluster's current state is captured at:

- **HTML snapshot:** `board/reports/2026-05-13-cluster-snapshot.html` (4 nodes ready, 21 pods running, 0 warning events at snapshot time).
- **Live regeneration:** `python3 .../observability/skills/cluster-snapshot/generate.py --output-path /tmp/cluster-snapshot.html` (against whatever kubectl context you point it at).

---

## 7. Honest deferrals

Named explicitly so the boundary is clear:

- **Real Kafka / event-stream ingestion** — supervisor reads parquet event-source files; Kafka contract is specified, consumer code does not ship.
- **Persistent KV state backend** — SCD2 state held in-process; Redis/DynamoDB integration spec'd, not implemented.
- **Multi-tenant supervisor sharing** — one supervisor process per tenant today.
- **Cross-swap state observability** — the hot-swap test verifies the load + unload + reload cycle through the production code path. The richer "read back v2's deviation across the swap" assertion in `DEVIATION.md` requires a supervisor-side NodeCtx bridge — named follow-up, demand-driven.
- **Sim-farm Modes B and C** — Mode A (file-vs-file diff) is shipped; Modes B (streaming Rust differ) and C (sidecar comparison) are spec'd, not implemented.
- **Customer-facing UI** — system is exercised via CLI + Makefile + the `resink` binary today.
- **Cargo CI cross-repo step** — supervisor's codegen path-deps depend on the marketplace repo as a sibling; CI clone needs a deploy key, scoped PAT, or in-repo test fixtures. Three private-compatible options documented; no public-repo conversion required.
- **Branch protection on private engineering remote** — helm CI runs on every PR; making it a required check requires a paid GitHub plan. Tracked as a contributor-count-driven decision.

Every deferral above corresponds to a named follow-up in our internal artifacts (ADRs, retros, exec summaries). Hidden deferrals turn into hidden surprises; we surface them.

---

## 8. Further reading

- **Capabilities report (companion):** [`board/reports/2026-05-13-resink-ai-capabilities.html`](../../../board/reports/2026-05-13-resink-ai-capabilities.html). Same evidence, executive-summary tone.
- **Design specs:**
  - [Runtime](2026-05-10-nanofab-runtime-design.md) — per-tenant compute, the supervisor's contract with node code.
  - [Training pipeline](2026-05-10-nanofab-training-pipeline-design.md) — the orchestrator + codegen flow.
  - [Serving & deployment](2026-05-10-nanofab-serving-deployment-design.md) — multi-tenant operations.
  - [Sim Farm](2026-05-10-nanofab-sim-farm-design.md) — the diff engine.
  - [Product UX](2026-05-10-nanofab-product-ux-design.md) — the customer-facing surfaces (CLI today; UI in roadmap).
  - [Org-OS design](2026-05-08-ai-native-org-os-design.md) — how the company itself operates.
- **Closed engineering arcs:**
  - [ADR-2026-05-16-001](../../../board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md) — dlopen restoration (closed 2026-05-13-1022).
  - [ADR-2026-05-09-001](../../../board/decisions/2026-05-09-001-org-os-bottom-up-flow.md) — bottom-up flow scaffolding (closed 2026-05-30).
- **Recent loops** (in chronological order of work):
  - [2026-05-13-1022](../../../board/exec-summaries/2026-05-13-1022.md) — resink-core bundle close-out (CI/CD + hot-swap test + `make bootstrap`).
  - [2026-05-13-0859](../../../board/exec-summaries/2026-05-13-0859.md) — observability skill v1 + hot-swap AE-side.
  - [2026-05-13-0056](../../../board/exec-summaries/2026-05-13-0056.md) — org-os ratification bundle (5 playbooks + 3 ADR ratifications).
- **All ADRs:** `board/decisions/` — every architectural decision has its own document. Start with [ADR-2026-05-10-001](../../../board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md) (Rust runtime choice) for the foundational tech-stack decision.

---

## Asking questions

The maintainers can be reached at the email in any `Cargo.toml`'s author field. For specific questions about a particular component, the file paths above each contain a `CLAUDE.md` or `docs/` directory with internal-engineering documentation.
{% endraw %}
