---
layout: default
title: application-resink-core OKR — 2026-05-13-1022
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-1022
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1022
  links: parent: board/okrs/2026-05-13-1022-ceo-brief.md
-->
{% raw %}

# Resink Core OKR — 2026-05-13 (loop 2026-05-13-1022)

## Context

Three-objective bundle close-out loop. All three items carried from prior retros, all three touch the resink-core submodule's product surface. The bundling rationale (single submodule, single CI bootstrap, single workstation toolchain context) is the load-bearing scoping decision; per the prior retro's "Bundling reduces per-loop coordination cost when 2-3 carryover items are all in the same submodule," this loop applies the pattern.

O1 closes ADR-2026-05-16-001's three-step plan (step 3 consumer-side hot-swap correctness test). O2 turns the resink-core remote from "direct-push-to-master" into "PR-gated with CI". O3 turns the unwritten submodule-deinit recovery recipe into a `make bootstrap` target.

## Objectives

### O1: Hot-swap supervisor-integrated test — close ADR-2026-05-16-001 step 3

source: ceo-brief

Why it matters: Closes the five-loop, three-step dlopen restoration plan. The supervisor-integrated test validates the real production code path (`plugin_loader::PluginNode`) against AE's v1/v2 fixture pair per the consumer-side acceptance contract in `templates/scd2_maintainer_v2/DEVIATION.md`. Once it passes, the dlopen Option-A deviation is fully resolved at runtime and the supervisor's hot-swap invariant is verified, not asserted.

Maps to brief O1 → KR1.1 (codegen produces two crates), KR1.2 (supervisor-integrated test), KR1.3 (passes under both feature flags), KR1.4 (regression-free against mvp-loop), KR1.5 (ADR closing-section).

**Key results** (KR numbering preserves traceability to brief O1)

- KR1.1: Codegen-produce two crates with the same `dim_user` schema. Invoke the codegen-scd2-node skill twice (`pattern_name="scd2_maintainer"`, then `pattern_name="scd2_maintainer_v2"`); each produces a cdylib crate at the named `output_dir`. `cargo build --release` against each produces `lib<crate>.{so,dylib}` at the expected `target/release/` location.
- KR1.2: New test at `crates/nanofab-supervisor/tests/hot_swap_correctness.rs`. Reuses `plugin_loader::PluginNode` (does not re-implement libloading wiring). Test body follows DEVIATION.md § "Consumer-side acceptance contract" verbatim: (1) load v1 cdylib via `PluginNode::load(v1_path)`; (2) process event e1 (Insert t=100 user_id=u1) + e2 (Update t=200 same key); (3) drop v1 handle; (4) load v2 via `PluginNode::load(v2_path)`; (5) process e3 (Update t=300 same key); (6) read back the closed predecessor row for e3 and assert `valid_to == 299` (not 300), confirming v2 is active.
- KR1.3: Test passes under both Cargo feature flags. Under `static-plugins` (current default; no actual dlopen), the hot-swap path is logically a no-op and the test exercises the trait pass-through. Under `dlopen-plugins`, the test is the full FFI roundtrip with library handle swap.
- KR1.4: Test integrates into `cargo test --workspace --release` (no special test harness needed; standard integration-test layout). `make mvp-loop` regression-free vs the prior baseline (`verdict=pass`).
- KR1.5: `board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md` body gains a closing "## Status (2026-05-13)" section naming this loop's step 3 close-out. ADR `status:` stays `active` (the ADR documents a plan; the plan is now complete; closing-section is the convention).

**Tasks**

- [ ] Locate v1 + v2 templates in the marketplace submodule; verify v2 template's `pattern_name="scd2_maintainer_v2"` is accepted by the codegen skill (per the SKILL.md change last loop)
- [ ] Invoke codegen-scd2-node twice with the dim_user schema to produce two output crates; cargo build --release each; verify both `.dylib` artifacts exist at the expected paths
- [ ] Author `tests/hot_swap_correctness.rs` reusing `plugin_loader::PluginNode`; assert per DEVIATION.md's 6-step contract
- [ ] Run `cargo test --release -p nanofab-supervisor --features dlopen-plugins hot_swap_correctness`; verify pass
- [ ] Run same test under default features (`static-plugins`); verify pass
- [ ] Run `make mvp-loop`; verify `verdict=pass`
- [ ] Append closing-section to ADR-2026-05-16-001

### O2: CI/CD bootstrap on the resink-core remote

source: ceo-brief

Why it matters: The resink-core remote was split out at 2026-05-12-1254 and has been receiving direct pushes to master with no verification. The risk is theoretical today (single contributor) but flips load-bearing the moment a second contributor lands. Minimum viable CI workflow + branch protection closes the gap without over-engineering.

Maps to brief O2 → KR2.1 (ci.yml workflow), KR2.2 (branch protection), KR2.3 (first run green on master), KR2.4 (docs reference).

**Key results** (KR numbering preserves traceability to brief O2)

- KR2.1: `.github/workflows/ci.yml` lands at the resink-core repo root with two jobs:
  - **cargo:** Sets up Rust 1.85 (matches the Dockerfile), runs `cargo build --workspace --release`, runs `cargo test --workspace --release`.
  - **helm:** Sets up helm v3, runs `helm lint --strict deploy/charts/nanofab-supervisor`; runs `helm template deploy/charts/nanofab-supervisor -f deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml`; runs `helm install --dry-run --debug nanofab-supervisor-test deploy/charts/nanofab-supervisor -f deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml`.
  - Triggers: `on: pull_request` + `on: push: branches: [master]`.
- KR2.2: Branch protection on master enabled via `gh api`. Settings: require PR before merge; require at least 1 status check passing (the workflow's combined `cargo` + `helm` aggregate). No "require review" rule.
- KR2.3: First post-merge run on master is GREEN (proves the workflow itself is well-formed and the gates pass against the current codebase). Failure-mode notes captured inline as workflow comments where non-obvious.
- KR2.4: `docs/` gains a short section (in an existing doc file like `docs/user-guide.md` or `docs/operations.md` — append, don't create a new doc) naming the CI workflow's gates and how to interpret a failed status.

**Tasks**

- [ ] Author `.github/workflows/ci.yml` with cargo + helm jobs
- [ ] Set up rust toolchain action with Rust 1.85 to match the production Dockerfile
- [ ] Set up helm action; verify the lint/template/dry-run commands work locally first (operator's machine)
- [ ] Commit and push to a `ci/bootstrap` branch in the resink-core remote
- [ ] Open PR; verify the workflow runs on the PR (PR-trigger path)
- [ ] Merge PR; verify the workflow runs on master (push-trigger path); confirm GREEN
- [ ] Enable branch protection on master via `gh api` (require PR + 1 status check)
- [ ] Verify branch protection by attempting `git push origin master` directly; expect rejection
- [ ] Append CI section to an existing docs file (don't create a new one)

### O3: `make bootstrap` target for submodule-deinit recovery

source: ceo-brief

Why it matters: Two prior loops hit the submodule-deinit-then-regenerate scenario (after merge-conflict recovery against the parent newbase). Each time the operator re-derived the recovery recipe (`uv sync --extra dev` in two dirs + `make mvp-loop`) from memory. Automation over documentation when the recovery is mechanical.

Maps to brief O3 → KR3.1 (target exists), KR3.2 (idempotent), KR3.3 (verified by simulating recovery).

**Key results** (KR numbering preserves traceability to brief O3)

- KR3.1: New `bootstrap` target in `synthetic_tenants/closed_loop_v0/Makefile`. Sequence: (1) `cd $(ORCH_DIR) && uv sync --extra dev`; (2) `cd $(SIMFARM_DIR) && uv sync --extra dev`; (3) `@echo "bootstrap complete — run \`make mvp-loop\` to verify"`. Does not auto-run mvp-loop (operator's choice).
- KR3.2: Idempotent. `make bootstrap` run twice in a row produces the same end state; no errors on re-run; no destructive operations (no `rm`, no `find -delete`).
- KR3.3: Verification recipe in the team OKR's smoke section: delete one of the `.venv` dirs (e.g., `rm -rf $(SIMFARM_DIR)/.venv`); run `make bootstrap`; assert the deleted venv is regenerated; run `make mvp-loop`; assert `verdict=pass`.

**Tasks**

- [ ] Edit `synthetic_tenants/closed_loop_v0/Makefile` — add `bootstrap` target with the 3-step sequence; add it to the `.PHONY` list
- [ ] Simulate the recovery scenario: `rm -rf sim-farm/.venv`; run `make bootstrap`; verify the venv exists post-run
- [ ] Run `make mvp-loop` after bootstrap; verify `verdict=pass`
- [ ] Test idempotence: run `make bootstrap` again with the venv present; verify no errors

## Risks

- **Codegen for v2 not yet exercised against `cargo build` on the workstation.** AE's exec summary last loop noted "no in-tree v2 build" — the v2 template is logically clean (single arithmetic change vs v1) but has never produced a cargo-built artifact. Mitigation: this loop's O1 is the first cargo build of a v2 output; if anything surfaces (unlikely given the surgical shape of the deviation), file an in-flight fix and update DEVIATION.md.
- **CI runner may not have the home-cluster's actual k8s context, so `helm install --dry-run` differs from production.** Mitigation: dry-run is sufficient for syntactic validation; full deploy-and-verify is operator-driven, not CI-driven. The workflow's helm gate catches template-render failures and chart-lint regressions; it doesn't try to substitute for end-to-end deploy.
- **Branch protection may surprise an operator who has habit-pushed to master.** Mitigation: announce in the team OKR's exec summary; the protection is "require PR" not "require review," so the operator can self-merge their own PR with the CI check.
- **`uv sync` toolchain pinning** — the recovery target inherits whatever uv version is installed; if the lockfile is incompatible with a newer uv, bootstrap fails. Mitigation: document the uv version in the Makefile target as a comment; the existing `make mvp-loop` already relies on the same tool, so the failure mode is identical to today's.
- **Hot-swap test under `static-plugins` is a logical no-op** — it may not catch any real regression because the swap mechanism is a stub. That's intentional (static-plugins is a fallback build mode); the load-bearing assertion runs under `dlopen-plugins`. Document this explicitly in the test's doc-comment.

## Out of scope

- Multi-table hot-swap test (e.g., dim_user + dim_account simultaneously). v1+v2 templates are dim_user-flavored per DEVIATION.md's worked example; multi-table is future-loop.
- CI release automation (publish, tag, deploy). Cargo-deploy-on-tag, helm-release-on-tag — all future-loop.
- Comprehensive failure-mode runbook for the CI workflow. The retro proposal P2 will surface as a candidate but this loop ships only the workflow itself + an inline-docs reference, not a full runbook.
- Cross-platform CI (Windows runner). Linux runner only this loop.
- Per-crate CI (sharded by crate). Workspace-wide build + test only.
- Observability into the CI itself (metrics, retry rates). Out of scope.

## Smoke verification recipe (for O3)

```bash
cd repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0
rm -rf $(realpath ../../sim-farm)/.venv
make bootstrap                             # regenerates .venv
test -d $(realpath ../../sim-farm)/.venv   # asserts presence
make mvp-loop                              # verdict=pass expected
```
{% endraw %}
