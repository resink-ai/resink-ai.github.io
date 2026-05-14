---
layout: default
title: CEO Brief — 2026-05-13-0859
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-0859
owner: board
grand_parent: Loops
parent: Loop 2026-05-13-0859
nav_order: 1
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-0859
  links: parent: ""
-->
{% raw %}

# Resink.ai CEO Brief — 2026-05-13 (loop 2026-05-13-0859)

## Context

Two loops on the same calendar date — the previous loop (`2026-05-13-0056`) shipped the org-os ratification bundle: 5 org-os edits + 2 ADR `draft → active` flips + 1 ADR scope extension, all tenant-isolation-clean. Backlog cleared. Mid-day check-in: gitbook CI had been silently failing for 5 consecutive publishes (Liquid parser misreading `{{...}}` template snippets in code fences); separately, the loop-index pages linked to `.md` instead of `.html`. Both fixes shipped as out-of-loop corrective PRs (newbase #16 + #17, gitbook #7 + #8); CI now green on main, live `blog.resink.ai` deploying as expected. Both incidents are retro-worthy.

**The gap.** Distance from vision is now **operator observability of running services**. The home cluster has a deployed nanofab-supervisor pod (per loop `2026-05-12-0645`); the resink CLI gives filesystem-workspace introspection (per loop `2026-05-12-1826`); but no first-class surface exists for asking "what's the cluster doing right now?" as a first-class AI-readable artifact. Closing that gap — a Claude Code skill that emits a self-contained HTML cluster snapshot — is this loop's primary deliverable.

**The shape of this loop.** Two-objective loop, single team active (AE). **Applies the smallest-product-slice playbook from last loop:** of four candidate observability slices (k8s cluster snapshot, k8s + log aggregation, Prometheus/metrics, filesystem workspace introspection), **k8s cluster snapshot has the fewest infrastructure prerequisites** — kubectl already works against the home cluster, no Prometheus to deploy, no log parser to write. Ship that slice; let richer slices land in future loops driven by real demand.

**Plus the hot-swap step 3 split (per prior retro § P1 option b).** Third consecutive slip on the joint AE + resink-core hot-swap correctness test forced a scheduling decision. **CEO picks option (b): split into AE-only + resink-core-only sub-deliverables.** AE-side this loop: template-side v1/v2 fixture pair — a `dim_user_scd2` node template with a deliberately differing logic step between v1 and v2 (e.g., a comparison-operator change in the latest-row predicate), packaged so the eventual resink-core consumer-side load-and-swap test has a stable, byte-pinned fixture to load. Sized S; co-rides with the AE-owned observability skill.

**Probe results entering the loop:**

- Marketplace plugins at `repos/resink-ai/resink-marketplace/plugins/`: `nanofab` (with `codegen-scd2-node` skill, GREEN at MVP); no other plugins. New `observability` plugin lands as the third entry.
- AE charter "Owned products" list: 4 bullets (nanofab plugin etc.); the observability plugin lands as the 5th bullet alongside `date:` bump.
- Tenant-isolation entering: clean (only canonical `acme.ai` placeholder in `org-os/conventions.md`).
- Two corrective PRs to gitbook + parent for the publishing breakage shipped between loops (newbase #16/#17, gitbook #7/#8). Surfaces in retro as a "what didn't" item with a forward-looking action.

**Re ADR-2026-05-16-001 step 3 (hot-swap correctness test):** **CEO splits per prior retro P1 option (b).** AE-side template fixture pair lands this loop (O2). Resink-core consumer-side load-and-swap test lands at the next active-resink-core loop (bundled with retro P2 CI/CD bootstrap + retro P3 submodule-deinit-recovery doc, per prior retro's bundling recommendation).

**Re retro proposals carrying forward:**

- **2026-05-13-0056 retro P1 (hot-swap scheduling):** resolved this loop — CEO picks split (option b); AE-side ships here.
- **2026-05-13-0056 retro P2 (CI/CD bootstrap on resink-core remote):** still deferred to next active-resink-core loop (bundled with P3).
- **2026-05-13-0056 retro P3 (submodule-deinit-then-regenerate recovery):** still deferred to next active-resink-core loop.
- **2026-05-13-0056 retro P4 (5-section playbook body shape codification):** still deferred (codification-without-motivation; revisit when motivated). No deviation this loop.

**CEO decisions for this loop:**

- **Activate one team + board (light).** AE active for both objectives; board light (brief + exec + retro + any minor org-os housekeeping). All other teams paused (no review-ack — neither objective touches their surface).

- **2 objectives, both AE-owned.** O1 observability plugin scaffold + `cluster-snapshot` skill v1 (kubectl wrap → self-contained HTML report saved at `board/reports/2026-05-13-cluster-snapshot.html`). O2 hot-swap step 3 AE-side fixture pair (`templates/scd2_maintainer_v2/` sister directory alongside `templates/scd2_maintainer/`, with a single deliberate logic deviation documented in DISPATCH.md and a fixture-shape contract for the eventual resink-core consumer test).

- **Smallest-product-slice discipline.** O1's prerequisites table is required body content — write down the four candidate slices, mark the chosen one, archive the others as future-loop work. The "what's the smallest first slice?" question is the load-bearing scoping decision.

- **Tenant-isolation invariant maintained.** The new skill goes under `repos/resink-ai/resink-marketplace/plugins/observability/`, not under `org-os/`; no `org-os/` writes expected this loop. Per-edit dry-runs as standing discipline if any `org-os/` edits surface.

- **Defer further org-os process work.** No new playbooks; no new ADR drafts; no in-place ADR extensions. The four-loop org-os-zero-write stretch the prior retro celebrated resumes here. P4 (5-section body shape codification) stays deferred — uninvoked-codification waits for invocation pressure.

**Standing CEO answers:**

- **Plugin layering.** The new `observability` plugin sits beside `nanofab` and `org-os` at the marketplace's `plugins/` root. Observability is generic (cluster-state introspection, not nanofab-specific), so a sibling plugin is the right shape rather than a skill nested under nanofab. Future observability skills (log aggregation, metrics) extend the plugin in-place.

- **HTML output, not markdown.** The deliverable is `*.html` because: (i) the report is meant to be opened in a browser as a self-contained dashboard view (CSS embedded, no external assets), (ii) the publish-to-gitbook script already recognizes `board/reports/*.html` and routes them through `raw_copy=True` (bypasses the Liquid-escape wrapping fixed in the inter-loop corrective PRs), (iii) the existing capabilities report at `board/reports/2026-05-30-resink-core-capabilities.html` is the reference shape.

- **HTML report content shape.** Top: cluster fingerprint (node count, k8s version, kubectl context, snapshot UTC timestamp). Mid: per-namespace pod table (name, status, restarts, ready, age, image). Bottom: recent events (warning-class only) + node resource summary. The skill writes a single file; idempotent re-runs overwrite atomically. Future slices add metrics/logs panels in-place.

- **Pinning + reproducibility (per DISPATCH discipline).** The skill names its kubectl + jq versions in `SKILL.md`'s "Pinned tooling" section, recording the smoke-test toolchain so future invocations can detect a drift.

**Carryover load by team:**

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| AE | O1 observability plugin + `cluster-snapshot` skill (scaffold + smoke + HTML report) + O2 hot-swap AE-side template fixture pair (v1/v2 with documented deviation) + charter "Owned products" 5th bullet + DISPATCH.md addendum + AE exec summary | M | Two-objective loop; the smallest-product-slice scoping keeps O1 contained. O2 is intentionally template-side only — resink-core consumer test deferred per split decision. |
| All other teams | Paused; no review-ack ask | — | Paused, silent — no team exec summary required (paused-team-only loop for non-AE teams). |

## Objectives

### O1: New `observability` plugin + `cluster-snapshot` skill v1

source: ceo-brief

Why it matters: First operator-readable surface for "what is the cluster doing right now?" as an AI-consumable artifact. The home cluster has been running deployed workloads for one loop (since `2026-05-12-0645`'s first-real-deploy); the next time anything goes red, the on-the-keyboard agent needs a one-shot way to capture and inspect cluster state without remembering kubectl incantations. The skill is also a forcing function: it makes the smallest-product-slice question concrete (k8s vs logs vs metrics) and ratifies the multi-plugin marketplace pattern (sibling plugin, not nested skill).

Smallest-product-slice table (per `org-os/playbooks/smallest-product-slice.md`):

| Slice | Infrastructure prerequisites |
|---|---|
| **K8s cluster snapshot (chosen for v1)** | **kubectl access to home cluster — exists since 2026-05-12-0645** |
| K8s + log aggregation | Snapshot prereqs + log parser/storage |
| Metrics (Prometheus) | Prometheus deployed in-cluster (not yet) + Grafana/dashboards |
| Filesystem workspace introspection | Workspace exists locally — resink CLI v1 already provides this |

KR1.1: New plugin scaffold at `repos/resink-ai/resink-marketplace/plugins/observability/` with `.claude-plugin/plugin.json` declaring version `0.1.0`, owner AE.

KR1.2: New skill at `repos/resink-ai/resink-marketplace/plugins/observability/skills/cluster-snapshot/SKILL.md` with: when-to-invoke, the procedure (the actual kubectl calls), pinned tooling section (kubectl + jq versions), worked-example invocation, output-file contract (`<output_dir>/cluster-snapshot.html`).

KR1.3: A working `cluster-snapshot.html` template + the generator logic (whether shell + heredoc, Python, or another stdlib-only path is an implementation choice — pick the one that minimizes prerequisites). Self-contained HTML: inline CSS, no external assets, opens in a browser standalone.

KR1.4: Marketplace manifest test suite stays GREEN. New plugin row added to `marketplace.json` (third entry) + `.version-bump.json` extended with the observability row. `bash scripts/bump-version.sh --check observability` reports version match.

KR1.5: One end-to-end smoke run against the home cluster. Report saved at `board/reports/2026-05-13-cluster-snapshot.html`. Verified by opening in a browser locally. `SMOKE.md` records the toolchain + acceptance.

KR1.6: AE charter "Owned products" gains a 5th bullet naming the observability plugin + its marketplace path; `date:` bumped.

KR1.7: Tenant-isolation invariant holds. Zero `org-os/` edits this loop.

### O2: Hot-swap step 3 AE-side — template v1/v2 fixture pair

source: prior-retro

Why it matters: ADR-2026-05-16-001's three-step plan has step 3 (hot-swap correctness test) slipped a third consecutive time. The prior retro § P1 forced a scheduling decision; CEO picks split (option b). AE's contribution: a stable, byte-pinned template fixture pair — `templates/scd2_maintainer/` (v1, existing) and a new `templates/scd2_maintainer_v2/` with a single deliberate logic deviation between them (e.g., the latest-row predicate uses `>` in v1 and `>=` in v2). The deviation is small enough to fit in a single-line diff but large enough to produce observably different output rows on the same input fixture — which is exactly what the eventual consumer-side hot-swap test needs to detect.

KR2.1: New `templates/scd2_maintainer_v2/` directory mirroring `templates/scd2_maintainer/`'s file layout. Build artifact (slot-filled compile) green via `cargo build --release` on the smoke crate; identical to v1 except the documented deviation.

KR2.2: A `DEVIATION.md` (or similar) in the v2 template root naming exactly what changed, the expected behavioral delta (sample input rows + the v1-output vs v2-output difference), and an "invariants preserved" section (the C-ABI signature, the status codes, the `Node`/`NodeCtx` trait surface — all unchanged).

KR2.3: DISPATCH.md gains a "## Template variants (hot-swap fixture)" section pointing at both template paths + the deviation contract, so the eventual resink-core consumer-side test has the contract pinned.

KR2.4: Marketplace tests stay GREEN (no regression from adding the v2 template directory).

KR2.5: The resink-core consumer-side test (load v1, process N events, swap to v2, process M events, assert the row deltas diverge as documented) is **out of scope this loop**. It bundles with retro P2 CI/CD + retro P3 submodule-deinit-recovery at the next active-resink-core loop.

## What success looks like

- `cluster-snapshot.html` saved at `board/reports/2026-05-13-cluster-snapshot.html`, opens in a browser, shows the home cluster's current state (node count, pod table, recent warning events). Self-contained — no external CSS/JS fetches.
- The `observability` plugin lands as the third marketplace entry; manifest tests stay GREEN; AE charter has the 5th "Owned products" bullet.
- `templates/scd2_maintainer_v2/` sits alongside `templates/scd2_maintainer/` with the deviation documented; the v2 build is green; DISPATCH.md names both paths.
- Zero `org-os/` writes this loop. Per-edit tenant-isolation dry-runs needed only if `org-os/` is touched (don't expect to touch it).
- Retro surfaces: (i) the inter-loop gitbook CI/links recovery — what we'd do differently to catch it earlier (sticky CI verification standing-practice or a hook?); (ii) the smallest-product-slice playbook's first real application; (iii) the hot-swap split — does it actually reduce coordination cost or just defer it?

## Out of scope

- Prometheus / log aggregation slices of the observability skill (future-loop work).
- Resink-core consumer-side hot-swap test (bundles with next active-resink-core loop).
- Any org-os process evolution (no new playbooks, ADRs, conventions edits).
- Any cross-team request from non-AE teams (paused).
- A `make bootstrap` target or any other infra-recovery doc (deferred per prior retro P3).
{% endraw %}
