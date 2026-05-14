---
layout: default
title: Retro — 2026-05-12-1826
date: 2026-05-12
status: active
type: retro
loop: 2026-05-12-1826
owner: board
grand_parent: Loops
parent: Loop 2026-05-12-1826
nav_order: 3
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-12
  status: active
  loop: 2026-05-12-1826
  links: parent: board/exec-summaries/2026-05-12-1826.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-12-1826

## What worked

- **First user-facing surface shipped in one loop's worth of work.** `resink` CLI v1: 5 subcommands; 6 unit tests; build clean; documented in user-guide. Implemented sub-project #5 (Product UX) § 3.2 directly off the canonical spec. The smallest-possible scope (read-only, filesystem-local) gave a real user experience without requiring a backend API or web stack. **Reproducibility:** for a product surface with multiple candidate slices (web app, CLI, notifications), prefer the slice with the fewest infrastructure prerequisites for v1. CLI > backend-dependent surfaces.

- **First non-promotion development cycle on the new resink-core remote.** Last loop bootstrapped the submodule with promotion-only history; this loop committed + pushed `feat(cli): v1 of the resink power-user CLI` directly to master. No surprises in the workflow. The submodule's independent development cadence is now exercised end-to-end. **Reproducibility:** the post-promotion development flow (commit on submodule master, push, bump parent pointer) works as designed; the submodule-promotion playbook (still draft, ADR-2026-05-12-002) captures the bootstrap path; the per-loop development path is the same as any other submodule.

- **Streaming parquet read pattern.** `ParquetRecordBatchReaderBuilder` with `batch_size = limit` produces one batch and stops. CLI memory footprint stays small even for large parquets. Establishes the pattern for future `resink table` extensions (`--filter`, `--query`, etc.). **Reproducibility:** for any CLI surface reading large columnar data, prefer streaming-with-early-stop over read-all-then-truncate.

- **arrow `prettyprint` feature opt-in.** The supervisor crate doesn't need it; resink-cli does. Adding `features = ["prettyprint"]` only on resink-cli's `arrow` dependency kept the supervisor's footprint unchanged. Clean isolated feature opt-in inside a Cargo workspace. **Reproducibility:** workspace-shared dependencies don't have to share their feature sets — per-crate feature overrides via `{ workspace = true, features = [...] }` are the right tool.

- **Third consecutive zero-`org-os/`-writes loop.** Pure tenant-surface work; org-os process surface is stable enough that several consecutive loops can avoid touching it. The pattern (loop 2026-05-11-2153 zero, 2026-05-12-0645 zero, 2026-05-12-1254 zero, 2026-05-12-1826 zero — actually that's FOUR consecutive zero-writes loops counting 2026-05-12-1254 as the prior loop). Trend: org-os should be edited via the evolution path (ADR + ratification), not via housekeeping; the four consecutive zero-writes loops demonstrate the discipline.

## What didn't

- **Submodule-deinit-then-regenerate pattern was unobvious.** During this loop's startup, the resink-core submodule's gitignored workspace artifacts were missing (because the prior loop's recovery from the PR #13 merge state included `git submodule deinit -f`, which wiped the gitignored runtime files). The supervisor crate's path-deps on `workspace/nodes/dim_*_scd2/` couldn't resolve. **Root cause:** gitignored runtime artifacts aren't restored by `git submodule update --init`. Recovery this loop: re-create venvs (`uv sync --extra dev` in 2 locations) + `make mvp-loop` to regenerate. **Mild:** recoverable but unobvious. Mitigation: documentation OR a `make bootstrap` target. See P1.

- **Hot-swap step 3: second consecutive slip.** Originally loop+1; slipped to loop+2 (workspace promotion focus); slipped to loop+3 (UX focus); now slipped to **loop+4**. **Root cause:** the test is a "joint AE + resink-core" deliverable that requires both teams active; every loop with a strong single-focus has crowded it out. Mitigation: either dedicate a next loop to hot-swap (single-focus), OR pick a loop where AE + resink-core are both naturally active and bundle the test. **If it slips a third time, evaluate scheduling assumption.** See P2.

- **Org-os ratification backlog growing.** 4 items deferred across 2 retros (ADR-2026-05-12-001 reframe-vs-act playbook body+ratify; ADR-2026-05-12-002 submodule-promotion playbook body+ratify; ADR-2026-05-16-003 scope extension to runbook; first-real-deploy playbook section in provisional-and-migrate.md). Each deferral was correct in isolation (single-focus loops); cumulatively the backlog builds. **Root cause:** the 4 items are all org-os authoring work, naturally batched in a single loop. The right move: dedicate a next loop to the bundle. **Mild:** each item is small individually; the bundle is sized at one M loop. See P3.

- **The org-os ratification backlog is the only "what didn't" that's CEO-attention-worthy.** The other items (submodule recovery; hot-swap slip) are bounded; the backlog growth is a scheduling decision the CEO should make explicit at the next loop's brief.

## Evolution proposals

### P1: Document submodule-deinit-then-regenerate recovery (class: **tenant**)

- **Problem it solves:** "What didn't" #1. After a submodule deinit, the gitignored runtime artifacts (`workspace/nodes/`, fixtures parquet) are gone and the supervisor's path-deps can't resolve. Recovery is `uv sync --extra dev` in 2 places + `make mvp-loop`. Unobvious to a future operator hitting it cold.
- **Proposed change:** Either (a) add a `make bootstrap` target to the closed_loop_v0 Makefile that runs the venv-create + workspace-regen in one shot; OR (b) add a "Troubleshooting" subsection to `docs/user-guide.md` documenting the recovery pattern. **Recommend (a) — automation over documentation** when the recovery is mechanical and repeatable.
- **Review path:** Tenant decision (resink-core repo Makefile + docs). No ADR.
- **Owner:** teams/application/resink-core.
- **Timing:** **Picked up next loop's resink-core build phase** if resink-core is active.

### P2: Evaluate hot-swap step 3 scheduling assumption (class: **tenant**)

- **Problem it solves:** "What didn't" #2. Second consecutive slip on a joint AE + resink-core deliverable. Each loop's single-focus discipline has crowded out the joint deliverable. The pattern suggests the scheduling assumption ("joint deliverable lands the loop after step 2 closes") may be wrong.
- **Proposed change:** Either (a) **dedicate next loop to hot-swap step 3** as the single focus; OR (b) **split the test** into AE-only (template-side: produce v1/v2 fixture pair) + resink-core-only (consumer-side: load v2 plugin without supervisor restart, verify in-flight events not dropped) deliverables that can land in separate loops; OR (c) **accept the slip indefinitely** until a loop where AE + resink-core are both naturally active. **Recommend (a) — dedicate the next loop** if the user wants the test closed; otherwise (c) — the slip is bounded as long as ADR-2026-05-16-001's body is grandfathered under absolute-date authoring.
- **Review path:** CEO decision at next brief authoring.
- **Owner:** board (decides); AE + resink-core (executes).
- **Timing:** **Decision: next loop's brief.** Execution: whenever the CEO picks (a) or (c).

### P3: Pick up the org-os ratification backlog as a single bundled loop (class: **org-os**)

- **Problem it solves:** "What didn't" #3. 4 deferred items across 2 prior retros: ADR-2026-05-12-001 + ADR-2026-05-12-002 (draft playbooks, each needs body authoring + ratification); ADR-2026-05-16-003 scope extension to `runbook` artifacts (in-place); first-real-deploy playbook section (in-place sibling to provisional-and-migrate.md). The backlog is sized at one M loop if bundled.
- **Proposed change:** **Dedicate the next loop to the org-os ratification batch.** 4-objective loop (or 3, if P1 first-real-deploy bundles with ADR-2026-05-12-002 submodule-promotion playbook as sibling sections). Single team active (board); minimal cross-team work; tenant-isolation maintained throughout.
- **Review path:** CEO decision at next brief authoring.
- **Owner:** board.
- **Timing:** **Decision: next loop's brief.** Strong-recommend timing: actually do this next loop. Backlog grows each loop deferred.

### P4: Codify the "smallest-scope-with-no-infrastructure-prereqs" pattern for product UX surfaces (class: **org-os**)

- **Problem it solves:** "What worked" #1. This loop picked the CLI (no backend prereq) over the dashboard tabs (backend prereq) over the chat experience (LLM + backend + frontend prereqs). The choice was deliberate but ad-hoc. Future product UX work could benefit from a documented pattern.
- **Proposed change:** Add a short playbook at `org-os/playbooks/smallest-product-slice.md` (`type: playbook`, `invocation_trigger: "when scoping a new product surface that has multiple candidate slices"`). Body: (a) when to invoke (multi-slice product surface; v1 scope decision); (b) the question to ask ("which slice has the fewest infrastructure prerequisites?"); (c) worked example (this loop's CLI choice vs. dashboard/chat). Lighter-weight than the other playbooks but still useful.
- **Review path:** ADR-class change to `org-os/`. Draft a placeholder ADR alongside the next loop's bundled org-os work (P3). Sized XS within the P3 bundle.
- **Owner:** board.
- **Timing:** **Bundled with P3** at next-loop org-os batch.

## Decisions to record

New ADR placeholders this loop: **None.** P4's playbook is small enough to be drafted+ratified inside the P3 bundle without a separate ADR.

## Carryover ADRs / open work still on the books

- **ADR-2026-05-12-001 (reframe-vs-act playbook):** still `status: draft`. Target ratification: next loop's org-os bundle (per P3).
- **ADR-2026-05-12-002 (submodule-promotion playbook):** still `status: draft`. Target ratification: next loop's org-os bundle.
- **ADR-2026-05-16-003 scope extension to `runbook`:** prior-prior retro P4. Target: next loop's org-os bundle.
- **First-real-deploy playbook section** in `provisional-and-migrate.md`: prior-prior retro P5. Target: next loop's org-os bundle.
- **Hot-swap correctness test (ADR-2026-05-16-001 step 3):** slipped to loop+4. Second consecutive slip. P2 names the decision options.
- **CI/CD setup on new resink-core remote:** prior-prior retro P2. Future loop.
- **`Chart.appVersion` ↔ supervisor crate version alignment:** bookkeeping.
- **Cargo MSRV declaration sync to Cargo.lock:** bookkeeping.

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: **passed**. **Fourth consecutive loop with zero `org-os/` writes by any team.** Final tenant-isolation grep returns only the canonical `acme.ai` placeholder at `org-os/conventions.md:121`. Pass.
{% endraw %}
