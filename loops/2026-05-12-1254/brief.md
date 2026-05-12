---
layout: default
title: CEO Brief — 2026-05-12-1254
date: 2026-05-12
status: active
type: okr
loop: 2026-05-12-1254
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-12
  status: active
  loop: 2026-05-12-1254
  links: parent: ""
-->
# Resink.ai CEO Brief — 2026-05-12 (loop 2026-05-12-1254)

## Context

The previous loop closed at 2026-05-12-0645 with `nanofab-supervisor:0.3.0` running on the home cluster end-to-end. Mid-loop after build phase closed, the 7-loop workspace-promotion action ticket `2026-06-06-001` resolved: the `resink-ai/resink-core` GitHub remote was created via `gh repo create` (executed directly by the brief author per the 2026-05-12-0645 retro's P1 reframing). The retro P1 pattern — brief author runs the named `gh` command directly rather than rely on user copy-paste — worked on its first application. **The blocker that gated 6 named downstream items is now cleared.**

**The gap.** The remote exists but is empty. The in-tree `repos/resink-ai/resink-core/` (100 source files, 7 parent commits in its history) is still embedded in newbase's working tree. Until it's pushed to its new remote and re-attached as a submodule, the cleared blocker is only theoretical. **This loop's deliverable: actually execute the workspace promotion** — split the resink-core history out of newbase, push it to its new remote as `master`, re-attach as a git submodule. The 8-loop saga ends.

**The shape of this loop.** A **structural-separation loop** — mostly mechanical git work + a few cross-reference touches. Single objective (the promotion itself). Three teams active light (board + resink-core + devops); SRE supports on cross-reference review; three paused (AE, sim-farm, DE).

**Probe results entering the loop:**

- `git ls-remote git@github.com:resink-ai/resink-core.git` → repo exists, **empty** (no refs). Created at 2026-05-12-0945 by the prior loop's mid-loop action close.
- `git log --oneline -- repos/resink-ai/resink-core | wc -l` → **7 commits** touch the in-tree path. Earliest is the initial in-tree placement; latest is from the deployment loop's chart bug fixes.
- `find repos/resink-ai/resink-core -type f` → **100 files** (excluding target/, .venv/, etc.) — 3 Rust crates + the Python orchestrator + sim-farm subtree + chart + docs + synthetic_tenants.
- `grep -rl "repos/resink-ai/resink-core" board/ teams/ scripts/ docs/` → **75 parent files** reference the path. Most are historical (exec summaries, OKRs, retros) and DO NOT need to be rewritten — they are point-in-time records. Live consumers: `scripts/publish-to-gitbook.py`, the capabilities HTML report, and the SRE deployment SOP — all reference the path which resolves identically post-submodule-add (the mount point is unchanged at `repos/resink-ai/resink-core/`).
- Open accepted requests with `deferred_to_loop: 2026-05-12-1254`: **none.**
- Draft ADRs in `board/decisions/`: **one** — `2026-05-12-001-reframe-vs-act-playbook.md` (drafted in the previous retro's P3; **NOT ratified this loop** — body authoring + ratification deferred so this loop stays single-focus on workspace promotion).

**Re ADR-2026-05-16-001 step 3 (hot-swap correctness test, scheduled loop+2 = this loop per the prior retro's slip record):** **Deferred again** to the next loop. Slip reason: the workspace promotion crowds out hot-swap this loop. Joint AE + resink-core deliverable; new target is loop+3 (the loop after this one). Recorded as plan-side slippage per ADR-2026-05-30-002's loop+N convention; the ADR body grandfathers under absolute-date authoring.

**Re retro carry items from 2026-05-12-0645:**

- **P1** (brief auto-runs gh repo create): ✅ closed mid-loop at 2026-05-12-0645 close; this loop is the consequent action.
- **P2** (Job-kind chart variant): deferred to a future deployment-focused loop; not in scope this loop.
- **P3** (reframe-vs-act playbook): draft ADR `2026-05-12-001` exists but body authoring + ratification deferred to next loop (single-focus discipline).
- **P4** (extend ADR-2026-05-16-003 scope to runbook): deferred to next loop alongside P3 (bundle org-os work).
- **P5** (first-real-deploy playbook section): deferred to next loop alongside P3 + P4.

**CEO decisions for this loop:**

- **Activate three teams + board.** resink-core (primary; owns the actual subtree split + push + submodule re-attach), devops (supports: chart paths reference verification), board (light: brief + retro). SRE supports on the SOP cross-reference check.

- **Single objective: workspace promotion.** No second objective; no deferred retro ratifications. The discipline of a single-focus loop is the right shape for a mechanical-but-multi-step git operation that needs to land cleanly with no slop.

- **Promotion mechanism: `git subtree split` preserves history.** The 7 commits touching `repos/resink-ai/resink-core/` get split into a standalone branch, pushed to the new remote as `master`. The parent's history (newbase) is unchanged — old commits still reference in-tree paths, which is correct for point-in-time history.

- **Post-promotion, the working-tree path is identical** (`repos/resink-ai/resink-core/` resolves to the same filesystem location whether in-tree or submodule). All live consumers (publish-to-gitbook.py, capabilities HTML, SOP) continue to work without path changes. The structural change is: there's now a `.git` directory at `repos/resink-ai/resink-core/.git` and `.gitmodules` gains a fourth entry.

- **Capabilities report touch-up.** The HTML at `board/reports/2026-05-30-resink-core-capabilities.html` and its companion MD cite in-tree paths for evidence. These continue to resolve, but the report's footer would benefit from a one-line note acknowledging the post-2026-05-12-1254 submodule status. **Recommend:** light update only — don't rewrite the report; add a single line.

- **Hot-swap step 3 slip-by-one.** Per the loop+N convention, recorded in this loop's exec summary. Joint AE + resink-core deliverable now targets loop+3 relative to the original step 2 closure (= loop after this one). No retro escalation; the slip is bounded.

**Standing CEO answers:**

- **Submodule URL convention.** `git@github.com:resink-ai/resink-core.git` (SSH). Consistent with `repos/resink-ai/resink-ai.github.io` which uses SSH; the other two (marketplace + home-cluster) use HTTPS. Either works for reading; SSH is preferred for the active-development repos to avoid PAT prompts.

- **Branch name on the new remote.** `master` (matches the in-tree branch name and prior repo convention; the gitbook submodule uses `main`, the marketplace uses `master`).

- **Submodule pointer source-of-truth.** Parent newbase commits the SHA of resink-core's master HEAD at promotion time; future loops bump the pointer as resink-core evolves independently.

**Carryover load by team** (per ADR-2026-05-09-005):

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| board | brief + retro; no ADR ratifications; multi-loop plan absorption for ADR-2026-05-16-001 step 3 slip-by-one (now loop+3) | XS | Light loop for board |
| resink-core | Workspace promotion full execution (subtree split → push → submodule add → verify); capabilities report footer light-touch | M | Headline deliverable; mechanical but multi-step |
| devops | Verify chart paths still resolve under submodule; light review of SOP cross-references | XS | Light supporting role |
| sre | Verify SOP cross-references still resolve under submodule; confirm runbook still consistent | XS | Light supporting role |
| AE | Paused; no review-ack ask this loop (the promotion doesn't touch AE's marketplace surface) | — | Paused, silent |
| sim-farm | Paused; no review-ack ask (verdict contract unaffected by parent-tree submodule structure) | — | Paused, silent |
| DE | Paused; no review-ack ask (DE artifacts don't reference resink-core paths) | — | Paused, silent |

## Objectives

### O1: Execute the resink-core workspace promotion end-to-end

source: ceo-brief

Why it matters: The 7-loop carry on resink-core's structural separation closes with this objective. Until executed, the prior loop's mid-loop unblock (remote creation) is only theoretical — the repo is empty and resink-core's commit graph is still embedded in newbase's history. The split preserves the 7 in-tree commits as a clean linear history on the new remote, making future bisect/blame/branching on resink-core development possible without parent-newbase coupling. All live consumers (publish-to-gitbook.py, capabilities report, SRE deployment SOP) continue to work because the working-tree mount point is unchanged.

**Key results**

- KR1.1: `git subtree split --prefix=repos/resink-ai/resink-core -b promote-resink-core master` (run from `/Users/shijinglu/Workspace/resink.ai/newbase/`) produces a branch `promote-resink-core` carrying ONLY the 7 commits touching the resink-core path. Branch HEAD is at the latest in-tree state (matches working tree). Verify via `git log --oneline promote-resink-core | wc -l` returning a small number consistent with the path-touching commit count.
- KR1.2: Push the split branch to the new remote as `master`: from a fresh clone OR via `git push git@github.com:resink-ai/resink-core.git promote-resink-core:master`. Verify via `git ls-remote git@github.com:resink-ai/resink-core.git` returning `refs/heads/master <sha>`. The remote is no longer empty.
- KR1.3: Remove the in-tree directory from parent newbase: `git rm -r --cached repos/resink-ai/resink-core` (uncached works too; we want index removal so the next `git submodule add` works). DON'T delete the working-tree files — they get re-placed by the submodule add.
- KR1.4: Move the working-tree directory aside (`mv repos/resink-ai/resink-core /tmp/resink-core-stash`), add as submodule: `git submodule add git@github.com:resink-ai/resink-core.git repos/resink-ai/resink-core`. This re-clones from the remote at the pushed master SHA. Verify the submodule directory has a `.git` file (pointing into `.git/modules/...`).
- KR1.5: Verify working-tree files are identical to the pre-promotion state. Compare via `diff -r /tmp/resink-core-stash repos/resink-ai/resink-core --brief` (excluding `target/`, `.venv/`, `__pycache__/`, etc.). Expected output: no differences. If any difference surfaces, investigate before proceeding (likely a gitignored file the parent tracked but the new repo doesn't, OR vice versa). Cleanup: `rm -rf /tmp/resink-core-stash` after verification passes.
- KR1.6: `.gitmodules` updated: gains a fourth submodule entry for `repos/resink-ai/resink-core` with URL `git@github.com:resink-ai/resink-core.git`. The other three entries (marketplace HTTPS, home-cluster HTTPS, gitbook SSH) are untouched.
- KR1.7: Verify `git submodule status` shows all four submodules at their correct SHAs. Verify `git status` shows the working tree clean (apart from the submodule-add changes which are staged).
- KR1.8: Light update to `board/reports/2026-05-30-resink-core-capabilities.html` AND `.md` — add a one-line footer note: "**Updated 2026-05-12-1254:** resink-core promoted from in-tree to git submodule; all paths cited above continue to resolve at the same working-tree mount point." Single-line addition; no path rewrites.
- KR1.9: Tenant-isolation invariant holds. No `org-os/` writes by this objective.

**Tasks**

- [ ] `git subtree split --prefix=repos/resink-ai/resink-core -b promote-resink-core` from newbase root.
- [ ] Push `promote-resink-core` to the new remote as `master`.
- [ ] `git rm -r --cached repos/resink-ai/resink-core` (parent index removal).
- [ ] Move working-tree resink-core to `/tmp/resink-core-stash` for diff baseline.
- [ ] `git submodule add git@github.com:resink-ai/resink-core.git repos/resink-ai/resink-core`.
- [ ] `diff -r /tmp/resink-core-stash repos/resink-ai/resink-core --brief` confirms no working-tree changes.
- [ ] Clean up `/tmp/resink-core-stash`.
- [ ] Verify `git submodule status` shows all 4 submodules at correct SHAs.
- [ ] Light footer update to capabilities report (HTML + MD).
- [ ] Tenant-isolation dry-run.

### O2: SRE + DevOps light-review cross-references resolve under submodule

source: ceo-brief

Why it matters: Most parent-repo references to `repos/resink-ai/resink-core/` are historical (exec summaries, OKRs, retros — point-in-time records that stay correct). The live consumers (publish-to-gitbook.py, SRE deployment SOP, chart README) reference paths that resolve identically post-submodule-add. This objective is a one-pass verification — confirm everything still works, don't rewrite anything that doesn't have to change.

**Key results**

- KR2.1: DevOps verifies `helm lint --strict` + `helm template` + `helm install --dry-run` still exit 0 against the chart at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` post-submodule-add. Single check; if green, no action needed.
- KR2.2: SRE verifies the deployment SOP's commands still work — `cd repos/resink-ai/resink-core` resolves to the submodule's working tree; `docker build -f deploy/charts/nanofab-supervisor/Dockerfile ...` runs as before. Spot-check; no full re-execution needed (the supervisor was deployed last loop; the deploy mechanism is unchanged).
- KR2.3: `scripts/publish-to-gitbook.py` was confirmed in the brief author's mental-model walk-through to walk into the submodule's filesystem identically to in-tree. No script changes expected; flagged for re-confirmation when O1 KR1.4 completes.
- KR2.4: Tenant-isolation invariant holds across O2 — zero new `org-os/` writes.

**Tasks**

- [ ] DevOps: run `helm lint --strict` + `helm template` + `helm install --dry-run` against the chart post-submodule-add; record outcome in DevOps exec summary.
- [ ] SRE: spot-check that the SOP's commands still resolve; record outcome in SRE exec summary.
- [ ] (Board, opportunistic) Re-run publish-to-gitbook.py to confirm the gitbook publish still works under the new submodule shape. If it does, the gitbook submodule pointer bumps as a side effect; bundle that into the parent commit.

## Risks

- **`git subtree split` produces unexpected history.** Subtree split walks the parent's history and includes any commit that touched the prefix. If a commit touched both `repos/resink-ai/resink-core/` AND another path, only the resink-core portion lands in the split. Result: the new repo's history may have empty merges or path-only commits. Mitigation: this is expected behavior; the split produces a clean linear history of resink-core-only changes. If the result looks wrong, alternative is `git subtree split --rejoin` or a fresh `git init` + single import commit (loses history but simpler).
- **Working-tree diff after submodule-add shows unexpected differences.** Possible causes: (a) `.gitignore` files differ between parent and submodule (some files tracked in parent that wouldn't be tracked in submodule, or vice versa); (b) line-ending differences if git config differs; (c) symlinks rendering differently. Mitigation: KR1.5 makes the diff an explicit acceptance criterion; surfaces issues before commit. If differences exist, investigate per-file before proceeding.
- **Submodule URL credentials.** The new remote uses SSH (`git@github.com:resink-ai/resink-core.git`). The submodule-add operation requires the operator's ssh key be authorized for the resink-ai org. Mitigation: prior loop already used SSH for gitbook submodule push; same auth applies.
- **Push of split branch fails on the new (empty) remote.** A brand-new GitHub repo defaults to no branch protection but may have a default branch named `main` rather than `master`. Mitigation: explicit ref naming in the push: `git push <remote> promote-resink-core:master`. If GitHub auto-creates `main` on first push, rename via API or accept `main` as the branch name (and update `.gitmodules` `branch =` if we declare one).
- **Cross-reference rot.** 75 parent files reference the in-tree path. Most are historical and SHOULD NOT be rewritten — they're correct at the time they were written. If a future reader expects every reference to be current state, they may be confused. Mitigation: the capabilities report footer update names the promotion date; future-loop briefs and exec summaries reference the post-promotion state naturally. No bulk rewrite this loop.
- **gitbook publish-to-gitbook.py walks submodule filesystem differently.** The script uses `pathlib.Path.glob` and `shutil.copy2` — these work transparently on submodule directories. The submodule's gitignored files (target/, .venv/) are already filtered by the script's scope. Risk is theoretical; KR2.3 confirms in practice.

## Out of scope this loop

- **Hot-swap correctness test** (ADR-2026-05-16-001 step 3) — slipped to **loop+3** (originally loop+1, slipped to loop+2 last loop, slipped to loop+3 this loop). Slip reason: workspace promotion focus this loop. Recorded in this loop's exec summary; ADR body grandfathers under absolute-date authoring per ADR-2026-05-30-002.
- **Retro proposal ratifications (P3 reframe-vs-act playbook; P4 ADR-2026-05-16-003 scope extension; P5 first-real-deploy playbook).** All three are `org-os` work; bundling them in a single deferred-loop after this one's single-focus discipline. Target: next loop or the one after.
- **Job-kind chart variant** (retro P2 from 2026-05-12-0645) — deferred to a future deployment-focused loop.
- **In-cluster image registry** — Phase 1 of the home-cluster roadmap; deferred.
- **pyinfra wrapper at `services/nanofab_supervisor/`** — home-cluster roadmap Phase 1.1; deferred.
- **Observability stack** (kube-prometheus-stack + Loki + Vector) — home-cluster roadmap Phase 1.3; deferred.
- **GitBook publishing of `org-os/` engine docs** — separate publishing track per 2026-06-13 retro P5; deferred.
- **GitBook reading-experience review** — pickup after PR #1 merges + reading-experience review window opens; not this loop.
- **Cargo MSRV declaration sync to Cargo.lock** (workspace says 1.83; lockfile needs 1.85) — bookkeeping; pickup at next resink-core workspace-touch.
- **`Chart.appVersion` ↔ supervisor crate version alignment** — bookkeeping; pickup at next chart revision.
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing; deferred (will look at this opportunistically once the resink-core submodule is in place).
- **`BLOCKED` observable verification** — multi-loop wait on resink-core surfacing structured BLOCKED state from the runtime coordinator.
- **SLO authoring + on-call rotation + alert paging** — depend on Phase 1.3 of the home-cluster roadmap.
- **Frontmatter-lint CI script** — seventh consecutive defer.
- **Resink-core long-running supervisor with `/healthz` + `/readyz`** — out per ADR-2026-05-16-001 not scheduling it.
- **Future codegen patterns** — out per AE's paused status.

## Ratifications this loop

**None.** No draft ADRs flip to active this loop. The one draft ADR (`2026-05-12-001-reframe-vs-act-playbook.md`) carries its `status: draft` into the next loop; body authoring + ratification deferred per single-focus discipline.

**Multi-loop plan slippage absorbed** (not a new ratification):
- **ADR-2026-05-16-001 step 3 (hot-swap correctness test):** was loop+2 (= this loop), **slipped to loop+3** (= the loop after this one). Joint AE + resink-core deliverable; slip reason: workspace promotion focus this loop. Per ADR-2026-05-30-002's loop+N convention; ADR body grandfathers under absolute-date authoring.
