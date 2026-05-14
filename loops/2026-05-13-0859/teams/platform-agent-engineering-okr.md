---
layout: default
title: platform-agent-engineering OKR — 2026-05-13-0859
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-0859
owner: teams/platform/agent-engineering
grand_parent: Loops
parent: Loop 2026-05-13-0859
nav_order: 10
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/agent-engineering
  date: 2026-05-13
  status: active
  loop: 2026-05-13-0859
  links: parent: board/okrs/2026-05-13-0859-ceo-brief.md
-->
{% raw %}

# Agent Engineering OKR — 2026-05-13 (loop 2026-05-13-0859)

## Context

Two AE-owned objectives this loop, both bounded. O1 ships the first cluster-observability surface — a new `observability` marketplace plugin and its `cluster-snapshot` skill — applying last loop's smallest-product-slice playbook to pick the k8s-snapshot slice over richer alternatives (logs, metrics). O2 closes AE's half of the hot-swap step 3 split (per prior retro § P1 option b): a v1/v2 template fixture pair with one deliberate logic deviation, so the eventual resink-core consumer-side load-and-swap test has a stable contract to load.

The two objectives are independent at the file-tree level (O1 touches the marketplace's `plugins/observability/` subtree + AE charter; O2 touches the marketplace's `plugins/nanofab/skills/codegen-scd2-node/templates/` subtree + DISPATCH.md). No coordination cost between them; AE can sequence in either order.

## Objectives

### O1: Ship `observability` plugin + `cluster-snapshot` skill v1

source: ceo-brief

Why it matters: The home cluster has been running deployed workloads for one loop (since `2026-05-12-0645`'s first-real-deploy). The next time anything goes red, the agent on the keyboard needs a one-shot "what is the cluster doing?" surface that produces an AI-readable artifact — not a sequence of remembered kubectl incantations. The smallest viable shape is a snapshot HTML report; this loop ships exactly that and nothing more.

Maps to brief O1 → KR1.1 (plugin scaffold), KR1.2 (SKILL.md), KR1.3 (HTML template + generator), KR1.4 (marketplace tests + manifest), KR1.5 (smoke run + report committed), KR1.6 (charter bullet), KR1.7 (tenant-isolation).

**Key results** (KR numbering preserves traceability to brief O1)

- KR1.1: New plugin directory at `repos/resink-ai/resink-marketplace/plugins/observability/` with `.claude-plugin/plugin.json` (version `0.1.0`, owner AE, the standard manifest shape used by `nanofab` + `org-os`).
- KR1.2: New skill at `repos/resink-ai/resink-marketplace/plugins/observability/skills/cluster-snapshot/SKILL.md`. Body shape: when to invoke, the procedure (the actual kubectl commands wrapped), pinned tooling section (`kubectl`, `jq` versions captured), worked-example invocation, output-file contract.
- KR1.3: A generator implementation that produces a self-contained `cluster-snapshot.html` (inline CSS; no external assets). Implementation language chosen to minimize prerequisites — almost certainly a shell wrapper around `kubectl get -o json` + `jq` + heredoc HTML emission, or a Python stdlib script if the shell-heredoc gets unwieldy. Pick the lower-prereq path. The HTML body has three sections (top fingerprint, mid pod table, bottom events + node summary) per the brief's "HTML report content shape" guidance.
- KR1.4: Marketplace manifest test suite stays GREEN. `repos/resink-ai/resink-marketplace/.claude-plugin/marketplace.json` extends with `observability` as the third `plugins[]` entry (source `./plugins/observability`, version `0.1.0`). `.version-bump.json` extends with the observability row. `bash scripts/bump-version.sh --check observability` reports version match. `bash tests/run-all.sh` regression-free vs the prior 98/0/3 baseline (new plugin will likely add 1+ manifest tests to the pass count).
- KR1.5: One end-to-end smoke against the home cluster. Output committed at `board/reports/2026-05-13-cluster-snapshot.html`. Verified by opening locally in a browser; the report opens, renders, and shows the actual cluster state at snapshot time (not a mocked template). `SMOKE.md` in the skill directory records: toolchain versions, the actual invocation command, the snapshot timestamp, acceptance — file exists + is non-trivial size + opens cleanly.
- KR1.6: AE charter "Owned products" gains a 5th bullet naming the observability plugin + path; `date:` bumped to `2026-05-13`.
- KR1.7: Tenant-isolation invariant holds. Zero `org-os/` edits this loop. Final post-loop sweep is GREEN.

**Tasks**

- [ ] Scaffold `repos/resink-ai/resink-marketplace/plugins/observability/.claude-plugin/plugin.json` (mirror the nanofab plugin's shape; version `0.1.0`, owner `agent-engineering`)
- [ ] Author `plugins/observability/skills/cluster-snapshot/SKILL.md` (frontmatter + body)
- [ ] Author the generator (`generate.sh` or `generate.py`) that wraps kubectl + jq and emits self-contained HTML
- [ ] Implement the HTML template (three sections per the brief; inline CSS)
- [ ] Run the skill against the home cluster; save output to `board/reports/2026-05-13-cluster-snapshot.html`
- [ ] Open the report in a browser locally to verify it renders; record acceptance in `SMOKE.md`
- [ ] Extend `marketplace.json` with the third plugin row; extend `.version-bump.json`; run `bash scripts/bump-version.sh --check observability`
- [ ] Run `bash repos/resink-ai/resink-marketplace/tests/run-all.sh`; expect regression-free vs 98/0/3
- [ ] Add 5th bullet to AE charter "Owned products"; bump `date:` to 2026-05-13
- [ ] Tenant-isolation grep post-edit + final loop-close sweep

### O2: Hot-swap step 3 AE-side — v1/v2 template fixture pair

source: prior-retro

Why it matters: Third consecutive slip on the joint AE + resink-core hot-swap test forced the prior retro § P1 decision. CEO picked split (option b). AE's half is template-side and is the easier of the two halves — a directory mirror of the existing template with exactly one documented logic deviation. The deviation needs to be small (single-line conceptual diff) but observable (different output rows on the same input). The eventual resink-core consumer-side test loads v1, processes N events, swaps to v2, processes M more, and asserts the v2-processed rows diverge from a "v1 kept running" baseline as documented. AE doesn't write that test — but AE pins the contract it has to satisfy.

Maps to brief O2 → KR2.1 (v2 directory), KR2.2 (DEVIATION.md), KR2.3 (DISPATCH.md addendum), KR2.4 (marketplace tests regression-free).

**Key results** (KR numbering preserves traceability to brief O2)

- KR2.1: New directory `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer_v2/` mirroring `templates/scd2_maintainer/`'s file layout. Single deliberate logic deviation — proposed: latest-row predicate uses `>` (strict) in v1, `>=` (non-strict) in v2. Effect: when two events arrive with identical timestamps, v1 keeps the older row latest; v2 promotes the newer row latest. Observable in row output; single-line diff in the predicate template; preserves the C-ABI signature, status codes, and `Node`/`NodeCtx` trait surface byte-for-byte.
- KR2.2: `templates/scd2_maintainer_v2/DEVIATION.md` (or `README-deviation.md`) names: (i) the exact source-line diff from v1, (ii) a 3-row input fixture + the v1 vs v2 expected-output difference, (iii) the "invariants preserved" list (C-ABI signature, status codes, ABI trait surface unchanged).
- KR2.3: `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/DISPATCH.md` gains a new "## Template variants (hot-swap fixture)" section pointing at both template paths + the deviation contract. Cross-link to ADR-2026-05-16-001 step 3 + this OKR.
- KR2.4: Marketplace tests stay GREEN. Both templates compile via the existing slot-fill smoke (`cargo build --release` exit 0; existing template smoke not broken; new v2 template gets its own smoke or shares the v1 smoke pattern).

**Tasks**

- [ ] Copy `templates/scd2_maintainer/` to `templates/scd2_maintainer_v2/`; apply the latest-row predicate deviation
- [ ] Smoke-build `scd2_maintainer_v2/` (slot-fill against a test input; `cargo build --release` green); record toolchain in SMOKE.md (or extend the existing SMOKE.md to cover both)
- [ ] Author `DEVIATION.md` in the v2 root with the diff + 3-row fixture + invariants-preserved list
- [ ] Add "## Template variants (hot-swap fixture)" section to DISPATCH.md
- [ ] Run marketplace test suite; expect regression-free
- [ ] Tenant-isolation grep — neither v2 template nor DEVIATION.md nor DISPATCH.md should contain `acme.ai`, `resink`, `nanofab` references inside `org-os/` (they live under marketplace, not `org-os/`, so this is a sanity check rather than a violation surface)

## Risks

- **HTML generator implementation choice over-runs.** Shell + heredoc is the smallest-prereqs path but gets unwieldy when escaping JSON-derived strings for HTML. Python stdlib (`html.escape` + f-strings) is one more prerequisite but vastly simpler. Mitigation: start with shell; if escaping pain shows up before the basic table renders, switch to Python stdlib. Time-box: 60 minutes of shell before switching.
- **kubectl context unset on the IC's machine.** If `kubectl` isn't configured for the home cluster, the smoke run can't execute. Mitigation: the skill's first step is `kubectl config current-context` (fail-fast with a clear error). If the IC's environment doesn't have kubectl context, the smoke is deferred to the next loop; the skill scaffolding still ships (this is preferable to inventing fake data).
- **Latest-row predicate deviation may already be a single-line change in the existing template** that doesn't require a separate v2 directory. Mitigation: if v1's template uses a `template-slot`-style placeholder for the comparison operator (so the same template can render both v1 and v2 by switching a slot value), the "two directories" shape is wrong and a single template + two slot-fill variants is the right shape. Surface this in the exec summary; if encountered, fix the OKR's interpretation rather than force the directory mirror.
- **Marketplace test regression from adding v2 template files.** The 98/0/3 baseline tests manifest shape, not file-tree shape; a new directory under `templates/` is unlikely to regress. If it does, AE updates whatever fixture pins the file count, in the same commit.
- **DEVIATION.md naming collision** with future patterns. If multiple template variants accumulate, "deviation.md" doesn't scale. Mitigation: this is the first v2 fixture; the naming refines if and when a third arrives. Don't pre-engineer.

## Out of scope this loop

- **Richer observability slices.** Log aggregation, Prometheus/metrics, distributed tracing — all future-loop work driven by real demand (per the brief's smallest-slice discipline).
- **Resink-core consumer-side hot-swap test.** That's the resink-core half of the split. Bundles with retro P2 CI/CD + retro P3 submodule-deinit-recovery at the next active-resink-core loop.
- **Observability skill scheduled/cron variant.** The skill is on-demand only this loop; scheduled invocation lands when a real "we want hourly snapshots" demand surfaces.
- **The cluster-snapshot HTML report being a polished dashboard.** v1 is an operator-readable table; no charts, no real-time updates, no auto-refresh. Future slices add fidelity.
- **Org-os process evolution.** No new playbooks, ADRs, conventions edits. The brief is explicit.
{% endraw %}
