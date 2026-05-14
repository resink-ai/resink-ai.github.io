---
layout: default
title: CEO Brief — 2026-05-13-1022
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-1022
owner: board
grand_parent: Loops
parent: Loop 2026-05-13-1022
nav_order: 1
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1022
  links: parent: ""
-->
{% raw %}

# Resink.ai CEO Brief — 2026-05-13 (loop 2026-05-13-1022)

## Context

Third loop on the same calendar date. The earlier loops shipped (i) the org-os ratification bundle (`2026-05-13-0056`), (ii) the observability skill + hot-swap step 3 AE-side (`2026-05-13-0859`). The retro for the AE loop named the resink-core bundle as the single highest-leverage carryover — three items that travel together because they all touch the same product surface and one team. **This loop closes the bundle.**

**The gap.** Distance from vision is now **resink-core operational maturity**. AE's half of the hot-swap correctness test (ADR-2026-05-16-001 step 3) closed last loop with a pinned `DEVIATION.md` consumer-side contract; the contract has no implementation. The new resink-core remote (split-out 2026-05-12-1254) has no CI workflow, no branch protection — pushes to master land unverified, with a single-contributor risk profile that is bounded today but unbounded the moment a second contributor lands. The submodule-deinit-then-regenerate recovery (the operator step that wipes gitignored runtime artifacts after `git submodule deinit -f`) is mechanical but undocumented — every operator who hits it re-derives the `uv sync --extra dev` + `make mvp-loop` recipe from memory.

**The shape of this loop.** Single-team loop (resink-core active; board light). Three bounded objectives sized S + S + M; all touch the resink-core submodule's product surface. **The bundling rationale matches the prior retro's recommendation:** when 2-3 carryover items are all in the same submodule's product surface and none of them is large alone, bundling reduces the per-loop coordination cost (one PR set, one CI bootstrap run, one operator workstation toolchain context).

**Probe results entering the loop:**

- Resink-core repo has `crates/{nanofab-coordinator,nanofab-node-abi,nanofab-plugin-dim-user,nanofab-supervisor,resink-cli}` plus `deploy/`, `docs/`, `training/`, `sim-farm/`, `synthetic_tenants/`. cargo workspace builds clean as of the last touched loop.
- `crates/nanofab-supervisor/src/plugin_loader.rs` is the established dlopen path; `crates/nanofab-supervisor/tests/dlopen_integration.rs` is the existing FFI roundtrip test (passes per the prior loop's review-ack). The new hot-swap test sits alongside as `tests/hot_swap_correctness.rs`, reusing `plugin_loader`'s real load path.
- Marketplace v2 template at `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer_v2/` shipped last loop with `DEVIATION.md` pinning the 6-step consumer acceptance contract. The codegen skill accepts `pattern_name: "scd2_maintainer_v2"` and produces a parallel crate from the same schema.
- No `.github/workflows/` in resink-core remote today. No branch protection on master.
- `synthetic_tenants/closed_loop_v0/Makefile` exists with a `mvp-loop` target driving the full closed-loop verdict. No `bootstrap` target.
- Tenant-isolation entering: clean (single canonical placeholder at `org-os/conventions.md:121`).

**Re ADR-2026-05-16-001:** **Closes this loop.** Step 1 (template C-ABI export) closed 2026-06-06; step 2 (supervisor swap) closed 2026-05-11-2153 (per AE's prior status review-ack); step 3 has been the third-consecutive slip until last loop's split. AE-side closed last loop; resink-core-side closes here. The three-step plan ends.

**Re retro proposals carrying forward into this loop:**

- **2026-05-13-0056 retro P2 (CI/CD bootstrap on resink-core remote):** picked up this loop as O2.
- **2026-05-13-0056 retro P3 (`make bootstrap` for submodule-deinit recovery):** picked up this loop as O3.
- **2026-05-13-0859 retro P3 (resink-core consumer-side hot-swap test):** picked up this loop as O1.
- **2026-05-13-0859 retro P2 (link-existence smoke for publish-to-gitbook):** still deferred (next non-resink-core loop).
- **2026-05-13-0859 retro P4 (observability v2 slices):** still deferred (demand-driven).
- **2026-05-12-1254 retro P3 (multi-loop-blocker-arc report-type):** revisit after this loop closes the hot-swap arc — the arc closes here, so next-loop retro is the natural trigger.

**CEO decisions for this loop:**

- **Activate one team + board (light).** resink-core active; board light (brief + exec + retro + this consolidation). All other teams paused (none of the three objectives touches their surface).

- **3 objectives, all resink-core-bearing.** O1 hot-swap step 3 supervisor-integrated test (sized M). O2 CI/CD workflow bootstrap on the resink-core remote (sized S). O3 `make bootstrap` target in `synthetic_tenants/closed_loop_v0/Makefile` (sized S).

- **Hot-swap test goes against the supervisor's real load path.** Per CEO choice this loop (option from scope question): new `tests/hot_swap_correctness.rs` reuses `plugin_loader.rs`'s dlopen code so the test validates the production code path, not a mock. Authoritative test; closes the ADR cleanly.

- **CI bootstrap stays minimal-but-real.** Cargo build (workspace-wide), cargo test (workspace-wide; includes the new hot-swap test if the operator workstation can compile both templates, otherwise gated by feature flag), helm gates (`helm lint --strict` + `helm template` + `helm install --dry-run` against `home-cluster-mvp.yaml`). Branch protection on master: require PR + 1 status-check pass. No release automation, no auto-deploy, no PR labels logic this loop.

- **`make bootstrap` is one Makefile target, not a runbook.** Automation over documentation when the recovery is mechanical. The target encapsulates the two `uv sync --extra dev` calls + the `make mvp-loop` regen. A 6-line addition to the existing Makefile.

- **No new ADR drafts.** All three objectives execute existing ADRs / retro proposals. The hot-swap arc closes ADR-2026-05-16-001's plan; the CI/CD work is tenant-class (no ADR per prior retro); the Makefile target is tenant-class (no ADR).

- **Tenant-isolation invariant maintained.** All three objectives sit under `repos/resink-ai/resink-core/`; zero `org-os/` edits expected.

**Standing CEO answers:**

- **Compile target for hot-swap.** Operator workstation is macOS (`.dylib` artifacts). The supervisor's `plugin_loader.rs` already supports `.dylib` on Darwin (per existing dlopen smoke). CI runs Linux (`.so`); the hot-swap test runs in CI too, building both v1 and v2 cdylibs from `pattern_name=scd2_maintainer{,_v2}` codegen outputs. Same test source; platform-specific build artifact path. The codegen-skill's smoke (run by AE) already validates the slot-fill against macOS; resink-core's CI validates Linux compile.

- **Hot-swap test event fixture.** Reuse the 3-row fixture from `DEVIATION.md` § "Worked example" verbatim: e1 Insert t=100 user_id=u1; e2 Update t=200 same key; e3 Update t=300 same key. The expected-row tables in DEVIATION.md (v1 vs v2) are the test's assertion source — no re-derivation. This is the load-bearing point of the contract being pinned.

- **CI workflow scope: build/test only, no deploy.** Auto-deploy on push-to-master is out of scope this loop (separate ADR-class concern; the home cluster's only deployer today is the operator's `helm install` from the workstation). The workflow file at `.github/workflows/ci.yml` runs cargo + helm gates and reports pass/fail; a future loop can add deploy automation if and when demand arises.

- **Branch protection scope.** Minimum viable: require PR + 1 passing check. No "require review" rule (single-contributor reality means review requirements would block all merges). When a second contributor lands, the protection tightens — but that's a future-loop ask.

- **`make bootstrap` invariants.** Idempotent (running twice produces the same end state). Fails loudly (each `uv sync` exits non-zero if the venv layout is broken). No destructive operations (does not `rm -rf` anything; the target only adds files).

**Carryover load by team:**

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| resink-core | O1 hot-swap supervisor-integrated test + O2 CI/CD workflow + branch protection + O3 `make bootstrap` target + team OKR + exec summary | M+S+S ≈ M-L | Three-item bundle in one loop; co-located in the resink-core submodule. The bundling rationale (single submodule, single CI bootstrap, single workstation context) holds. |
| All other teams | Paused; no review-ack ask | — | Paused, silent. AE made the hot-swap fixture last loop; nothing AE-side this loop. |

## Objectives

### O1: Hot-swap supervisor-integrated test — close ADR-2026-05-16-001 step 3

source: prior-retro

Why it matters: This is the closing deliverable for ADR-2026-05-16-001's three-step plan, which has now run for five loops (step 1 = 2026-06-06; step 2 = 2026-05-11-2153 review-ack; step 3 = three consecutive slips and one AE-side split-shipment). Once it lands, the dlopen Option-A deviation is fully resolved at runtime — the supervisor can load a plugin, process events, swap plugin libraries, and continue processing — and that's the operational invariant the entire multi-tenant deployment story depends on. The test is the verification artifact that proves the hot-swap path works against the real production code (plugin_loader.rs), not against a mock.

KR1.1: Codegen-produce two crates from the existing v1 + v2 templates. Invoke `codegen-scd2-node` skill twice — once with `pattern_name: "scd2_maintainer"`, once with `pattern_name: "scd2_maintainer_v2"` — against the same `dim_user` schema. Each output produces a `cdylib` crate; `cargo build --release` against each produces `lib<crate>.{so,dylib}`.

KR1.2: New test at `crates/nanofab-supervisor/tests/hot_swap_correctness.rs` reusing `plugin_loader::PluginNode`. Test flow follows `DEVIATION.md` § "Consumer-side acceptance contract" verbatim: (1) load v1 cdylib via plugin_loader, (2) process e1 (Insert t=100) + e2 (Update t=200), (3) drop v1 library handle, (4) load v2 cdylib via plugin_loader, (5) process e3 (Update t=300), (6) assert the closed predecessor for e3 has `valid_to == 299` (not 300) — confirming v2 is active.

KR1.3: Test passes against both `static-plugins` (no-op stub) and `dlopen-plugins` features. Under `static-plugins` the hot-swap is a logical no-op (still validates the trait pass-through); under `dlopen-plugins` it's the full FFI roundtrip with library swap.

KR1.4: Test added to the workspace's `cargo test --release` invocation. `make mvp-loop` regression-free vs prior baseline.

KR1.5: ADR-2026-05-16-001 body gains a "## Status (2026-05-13)" closing section naming this loop's completion of step 3; `status:` stays `active` (the ADR documents a multi-step plan, all steps now done; closing convention is per the existing two-stage status pattern used elsewhere).

### O2: CI/CD bootstrap on the resink-core remote

source: prior-retro

Why it matters: The resink-core remote was split out at 2026-05-12-1254 and has been receiving direct pushes to master with no automated verification. Single-contributor reality keeps the risk theoretical, but the moment a second contributor lands the unverified-master invariant breaks. This loop closes the gap with the minimum viable CI workflow — build + test + helm gates — and turns on branch protection so the workflow becomes load-bearing.

KR2.1: New `.github/workflows/ci.yml` at the resink-core repo root with two jobs: (a) `cargo` (runs `cargo build --workspace --release` + `cargo test --workspace --release`), (b) `helm` (runs `helm lint --strict deploy/charts/nanofab-supervisor` + `helm template deploy/charts/nanofab-supervisor -f deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml` + `helm install --dry-run --debug nanofab-supervisor deploy/charts/nanofab-supervisor -f .../home-cluster-mvp.yaml`). Triggers: `on: pull_request` + `on: push: branches: [master]`. Runner: `ubuntu-latest`.

KR2.2: Branch protection on master via `gh api`. Minimum viable: require PR before merge; require at least 1 status check passing. No "require review" (single-contributor reality).

KR2.3: One CI run completes successfully on master after the PR with the new workflow merges. Failure modes documented inline as workflow comments.

KR2.4: `docs/` gains a short README section (in an existing doc file, not a new file) naming the CI workflow's gates and how to interpret a failed status.

### O3: `make bootstrap` target for submodule-deinit recovery

source: prior-retro

Why it matters: Every operator who runs `git submodule deinit -f` (e.g., during a parent-newbase merge-conflict recovery; happened twice in prior loops) discovers that the gitignored runtime artifacts get wiped — `training/orchestrator/.venv` and `sim-farm/.venv` are both gone, and `make mvp-loop` fails until they're regenerated. The recipe is mechanical (`uv sync --extra dev` in two locations + `make mvp-loop`) but unobvious. Automation over documentation when the recovery is mechanical.

KR3.1: New `bootstrap` target in `synthetic_tenants/closed_loop_v0/Makefile`. Sequence: (1) `uv sync --extra dev` in `$(ORCH_DIR)` (training/orchestrator); (2) `uv sync --extra dev` in `$(SIMFARM_DIR)`; (3) emit a "ready; run `make mvp-loop` to verify" message. Does NOT auto-run `mvp-loop` (operator's choice — they may want to inspect the venv state first).

KR3.2: Idempotent. Running `make bootstrap` twice in a row produces the same end state; no errors on re-run; no destructive operations.

KR3.3: Verified by simulating the recovery scenario: delete one of the `.venv` directories, run `make bootstrap`, run `make mvp-loop`, assert `verdict=pass`. Records the verification in the team OKR's smoke section.

## What success looks like

- `cargo test --release -p nanofab-supervisor` passes the new `hot_swap_correctness` test under both `static-plugins` and `dlopen-plugins` features. Test loads v1 + v2 cdylibs through `plugin_loader::PluginNode` and asserts the `valid_to` shift per DEVIATION.md's contract.
- `make mvp-loop` regression-free (`verdict=pass`).
- `.github/workflows/ci.yml` lands on master via PR; first post-merge run on master is GREEN; branch protection is active (a manual push to master is rejected).
- `make bootstrap` exists; one verified recovery cycle (deinit-style scenario simulated; `verdict=pass` after `make bootstrap` + `make mvp-loop`).
- ADR-2026-05-16-001 closing-section added; the multi-loop plan ends.
- Tenant-isolation invariant: clean. Zero `org-os/` edits.
- Retro surfaces: (i) ADR-2026-05-16-001 arc retrospective — five loops, three steps, one split, one closing loop; what did the multi-loop-blocker-arc report-type idea need (a real arc just closed)? (ii) CI bootstrap as a forcing function — what would change about how the resink-core remote is operated now that master is gated? (iii) `make bootstrap` as the canonical recovery-automation shape.

## Out of scope

- Release automation (auto-publish, auto-tag, auto-deploy on push). Future-loop concern.
- "Require review" branch protection. Single-contributor reality; future-loop tightening when a second contributor lands.
- Hot-swap test for other dim tables (`dim_account`, etc.). v1 + v2 templates are `dim_user`-flavored per the worked example in DEVIATION.md; multi-table hot-swap is a future-loop slice.
- Observability v2 slices (logs, metrics). Demand-driven; nothing scheduled this loop.
- Org-os process evolution. No new playbooks, ADRs, conventions edits. Brief is explicit.
- Multi-loop-blocker-arc report-type ADR. Surface as a retro proposal candidate now that the hot-swap arc just closed; not a this-loop deliverable.
{% endraw %}
