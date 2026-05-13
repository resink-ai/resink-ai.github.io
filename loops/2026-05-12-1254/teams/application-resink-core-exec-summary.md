---
layout: default
title: application-resink-core Exec Summary — 2026-05-12-1254
date: 2026-05-12
status: active
type: exec-summary
loop: 2026-05-12-1254
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/resink-core
  date: 2026-05-12
  status: active
  loop: 2026-05-12-1254
  links: parent: board/okrs/2026-05-12-1254-ceo-brief.md
  team_okr: teams/application/resink-core/okrs/2026-05-12-1254-team-okr.md
-->
{% raw %}

# Resink Core Exec Summary — Loop 2026-05-12-1254

**Headline.** The 8-loop saga ends. `repos/resink-ai/resink-core/` is now a real git submodule pointing at `git@github.com:resink-ai/resink-core.git` (master HEAD `f8b5f5d`). `git subtree split --prefix=repos/resink-ai/resink-core master` walked 19 newbase commits and produced a clean 7-commit linear history of resink-core-only changes, pushed to the new remote as `master`. Parent newbase index removed the in-tree directory and re-attached it via `git submodule add`. Working-tree integrity verified — only the gitignored runtime artifacts (fixtures parquet, workspace directory) differed; restored those for operator continuity. `.gitmodules` gained a fourth entry. No path rewrites required across the 75 parent files that reference the submodule's path — they all resolve identically.

## Per-KR rollup (O1)

| KR | Outcome | Notes |
|---|---|---|
| KR1.1 (`git subtree split`) | **PASS** | 19 commits walked, 7-commit clean linear history produced on branch `promote-resink-core`. HEAD: `f8b5f5d` (matches working-tree state). Each commit corresponds to a prior loop's resink-core-touching changes. |
| KR1.2 (push to new remote) | **PASS** | `git push git@github.com:resink-ai/resink-core.git promote-resink-core:master` succeeded. `git ls-remote` confirms `refs/heads/master` at `f8b5f5d`. The empty-from-creation remote now has its initial content. |
| KR1.3 (parent index removal) | **PASS** | `git rm -r --cached repos/resink-ai/resink-core` removed 100+ files from parent index. Working tree untouched (—cached form). |
| KR1.4 (submodule re-attach) | **PASS** | Working tree moved to `/tmp/resink-core-stash`; `git submodule add git@github.com:resink-ai/resink-core.git repos/resink-ai/resink-core` cloned fresh from new remote. Submodule's `.git` file points into `.git/modules/repos/resink-ai/resink-core`. |
| KR1.5 (working-tree integrity) | **PASS (with expected gitignored deltas)** | `diff -r /tmp/resink-core-stash repos/resink-ai/resink-core --brief --exclude=target --exclude=.venv --exclude=__pycache__ --exclude=.git --exclude=.pytest_cache` showed only gitignored runtime artifacts missing (7 fixture parquet/JSON files in `synthetic_tenants/closed_loop_v0/fixtures/` + the entire `workspace/` directory — all matched by the source repo's `.gitignore`). Restored those from stash for working-tree continuity; final diff is empty. Stash cleaned. |
| KR1.6 (`.gitmodules` updated) | **PASS** | Fourth entry added: `[submodule "repos/resink-ai/resink-core"]` with `url = git@github.com:resink-ai/resink-core.git`. Marketplace + home-cluster + gitbook entries untouched. |
| KR1.7 (`git submodule status`) | **PASS** | All 4 submodules at correct SHAs: home-cluster `134a44d`, gitbook `9b39735`, resink-core `f8b5f5d`, marketplace `98780a7`. `git status` shows the parent's staged submodule-add + the in-tree deletions ready to commit. |
| KR1.8 (capabilities report footer) | **PASS** | Both `board/reports/2026-05-30-resink-core-capabilities.md` and `.html` gained a 1-line post-promotion note. No path rewrites in the body. |
| KR1.9 (tenant-isolation) | **PASS** | `grep -nrE "(resink\|nanofab\|home-cluster\|acme\.ai)" org-os/` returns only the canonical `acme.ai` placeholder at `org-os/conventions.md:121`. Zero `org-os/` writes by this objective. |

## Build phase signal-of-interest

- **`git subtree split` is fast and faithful.** Walked 19 commits in seconds; produced 7-commit linear history (one commit per loop that touched the resink-core path). Loop-name commit subjects preserved verbatim: "Loop 2026-05-09 + 2026-05-10: ...", "Loop 2026-05-16: MVP closed loop is GREEN", through "Loop 2026-05-12-0645: home-cluster first deploy". Future git-blame on resink-core's master traces directly into the loop-level commit subject — the loop-coupling is preserved on the new repo's standalone timeline.
- **No path rewrites needed across 75 referencing files.** The working-tree mount point is unchanged; all relative-path references continue to resolve. Most references are historical (exec summaries, OKRs, retros — point-in-time records that stay correct). The 1-line footer on the capabilities report is the only live-reference acknowledgment of the structural change.
- **Gitignored runtime artifacts persist in working tree.** Fresh-cloning the submodule would not produce a runnable mvp-loop state — the operator would need to run the orchestrator to regenerate `fixtures/*.parquet` + `workspace/`. For this loop's stash-and-restore flow, runtime continuity is preserved.

## Cross-team coordination

- **DevOps verified chart gates** — `helm lint --strict --values values/home-cluster-mvp.yaml` + `helm template` + `helm install --dry-run` all exit 0 post-submodule-add. No chart changes needed. (See DevOps exec summary.)
- **SRE verified SOP path resolution** — `cd repos/resink-ai/resink-core` resolves to submodule's working tree; SOP commands work as before. Footer notes added to both runbooks. (See SRE exec summary.)

## Open carries (for next loop)

- **Independent CI/CD for the new submodule** — the new repo has no CI workflows yet. Future loop adds GitHub Actions for cargo build, helm gates, sim-farm pytest.
- **Branch protection rules on the new remote** — should set up before any contributors land changes. Future loop or operator-deliberate action.
- **Hot-swap correctness test (ADR-2026-05-16-001 step 3)** — slipped to **loop+3** (originally loop+1, slipped to loop+2 last loop, slipped to loop+3 this loop). Joint AE + resink-core deliverable.
- **Cargo MSRV declaration sync** — workspace `Cargo.toml` says 1.83 but Cargo.lock needs 1.85 (clap_lex). Bookkeeping; pickup at next workspace-touch.
- **`Chart.appVersion` ↔ crate version alignment** — Chart.appVersion is `0.1.0`, crate is `0.3.0`. Bookkeeping.
- **`teams/application/resink-core/resink-core/` symlink** — non-load-bearing housekeeping; pickup opportunistically.
- **Long-running supervisor + `/healthz`** — out per ADR posture.

## Tenant-isolation invariant

Held throughout. Zero `org-os/` writes by resink-core this loop. All edits landed under `repos/resink-ai/resink-core/` (no parent-relative `org-os/` paths touched), parent newbase index, `.gitmodules`, `board/reports/`, and `teams/application/resink-core/`. Final dry-run clean.
{% endraw %}
