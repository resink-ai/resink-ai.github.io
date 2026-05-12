---
layout: default
title: application-resink-core OKR — 2026-05-12-1254
date: 2026-05-12
status: active
type: okr
loop: 2026-05-12-1254
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-12
  status: active
  loop: 2026-05-12-1254
  links: parent: board/okrs/2026-05-12-1254-ceo-brief.md
-->
# Resink Core OKR — 2026-05-12 (loop 2026-05-12-1254)

## Context

The 7-loop carry on resink-core's workspace promotion (in-tree → submodule) ends this loop. The `resink-ai/resink-core` GitHub remote was created mid-loop at 2026-05-12-0645 (retro P1 reframing — brief author runs `gh repo create` directly). Remote exists but is empty. This loop executes the mechanical promotion: subtree split, push, submodule add, verify.

The promotion preserves the 7 in-tree commits as a clean linear history on the new remote (via `git subtree split`). The parent newbase keeps its full history intact (point-in-time references stay correct). The working-tree mount point at `repos/resink-ai/resink-core/` is unchanged — all live consumers (chart, publish-to-gitbook.py, SOP) continue to resolve paths identically. The structural change is: a `.git` file appears at the submodule root + `.gitmodules` gains a fourth entry.

## Objectives

### O1: Execute workspace promotion + verify working-tree integrity

source: ceo-brief

Why it matters: Closes the 8-loop saga (7 prior carries + this loop's execution). Unblocks resink-core's independent development cadence (independent CI/CD, branching, releases). Preserves history for git-blame and bisect on the new repo's standalone timeline.

**Key results**

- KR1.1: `git subtree split --prefix=repos/resink-ai/resink-core -b promote-resink-core master` from newbase root produces a clean branch carrying only the resink-core-touching commits.
- KR1.2: Push to new remote as `master`: `git push git@github.com:resink-ai/resink-core.git promote-resink-core:master`. `git ls-remote` post-push shows `refs/heads/master` at the pushed SHA.
- KR1.3: Parent index removal: `git rm -r --cached repos/resink-ai/resink-core` (cached form preserves working tree).
- KR1.4: Working-tree stash + submodule re-attach: `mv repos/resink-ai/resink-core /tmp/resink-core-stash` → `git submodule add git@github.com:resink-ai/resink-core.git repos/resink-ai/resink-core`. Re-clones from new remote.
- KR1.5: Working-tree integrity: `diff -r /tmp/resink-core-stash repos/resink-ai/resink-core --brief` (excluding target/, .venv/, __pycache__/, .git) returns no differences. Cleanup `/tmp/resink-core-stash`.
- KR1.6: `.gitmodules` gains a fourth entry for `repos/resink-ai/resink-core` with the SSH URL.
- KR1.7: `git submodule status` shows 4 submodules at correct SHAs; `git status` shows working tree clean apart from staged submodule-add changes.
- KR1.8: `board/reports/2026-05-30-resink-core-capabilities.html` + `.md` get a 1-line footer note acknowledging the post-2026-05-12-1254 submodule status. No path rewrites elsewhere.
- KR1.9: Tenant-isolation invariant holds; no `org-os/` writes.

**Tasks**

- [ ] Run `git subtree split` from newbase root.
- [ ] Push split branch to new remote as master.
- [ ] `git rm -r --cached` resink-core in parent index.
- [ ] Stash working-tree to /tmp/ for diff baseline.
- [ ] `git submodule add` from new remote.
- [ ] Verify working-tree integrity via `diff -r --brief`.
- [ ] Verify `git submodule status` shows 4 submodules.
- [ ] Add 1-line footer to capabilities report (HTML + MD).
- [ ] Tenant-isolation dry-run.
- [ ] Cleanup stash directory.

## Risks

- **`git subtree split` history shape.** If the 7 commits touched both resink-core and other paths, only the resink-core portions land in the split. Verify the split history looks sensible before pushing.
- **Working-tree diff surprises.** If `diff -r --brief` shows unexpected differences, investigate per-file. Likely cause: gitignored-in-one-but-not-the-other.
- **SSH key auth for the new remote.** Operator's SSH key must be authorized for resink-ai org. Confirmed via prior loop's PR pushes.
- **GitHub default branch.** Empty repos sometimes default to `main`. The push specifies `:master` explicitly to override.

## Out of scope this loop

- Hot-swap step 3 (slipped to loop+3).
- Independent CI/CD for the new submodule (future loop).
- Branch protection rules on the new remote (future loop).
- Bulk rewrite of historical references (intentional non-change; references stay point-in-time-correct).
