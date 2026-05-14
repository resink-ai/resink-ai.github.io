---
layout: default
title: "ADR 2026-05-12-002: submodule promotion playbook"
date: 2026-05-12
status: active
type: adr
owner: board
parent: Decisions (ADRs)
nav_order: 13
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-12
  status: active
  decision: Author an org-os playbook codifying the in-tree → submodule promotion pattern using `git subtree split`.
  links: parent: board/retros/2026-05-12-1254-ceo-retro.md
-->
{% raw %}

# ADR 2026-05-12-002 — Submodule-promotion playbook

## Status

**Ratified at loop 2026-05-13-0056** alongside the org-os ratification bundle. Drafted at loop 2026-05-12-1254 retro § P1. The mandated edit landed: [`org-os/playbooks/submodule-promotion.md`](../../org-os/playbooks/submodule-promotion.md) authored with `type: playbook`, `status: active`. The playbook includes the **footer-acknowledgment sibling section** per 2026-05-12-1254 retro § P4 (codifies the 1-line footer pattern for structural-but-non-disruptive changes).

## Context

This loop (2026-05-12-1254) executed the resink-core workspace promotion using `git subtree split --prefix=repos/resink-ai/resink-core master`. The split walked 19 newbase commits and produced a clean 7-commit linear history of resink-core-only changes; pushed to the new remote as `master`; parent's index removed via `git rm -r --cached`; submodule re-attached via `git submodule add`. Working-tree integrity verified via `diff -r --brief` (only gitignored runtime artifacts differed; restored from stash). The promotion preserved loop-level commit subjects on the new repo's standalone timeline ("Loop YYYY-MM-DD: ..." for each prior loop that touched the resink-core path) — future git-blame traces directly back to the originating loop.

The procedure is mechanical-but-multi-step. The pattern is reusable for any in-tree → submodule promotion:
- Sim-farm extracting to its own repo when first non-Python component lands.
- `services/<name>/` extracts under the home-cluster roadmap's Phase 1.
- Any future `repos/<org>/<name>/` directory currently in-tree.

Without a documented playbook, future promotions risk choosing the wrong tool (e.g., fresh `git init` + single import commit loses history) or skipping non-obvious steps (e.g., the working-tree stash + diff-verification step). Codifying the procedure preserves the loop-coupling on future migrations.

## Decision (draft)

Author a new playbook at `org-os/playbooks/submodule-promotion.md` with `type: playbook` and `invocation_trigger: "when an in-tree directory under repos/<org>/<name>/ needs to become a real git submodule"`.

Body sections:

1. **When to invoke.** In-tree development is constraining the team's cadence (independent CI/CD, branching, contributor flow blocked by parent-tree coupling). A separate repo would unblock structural separation without forcing source-code changes.

2. **The procedure.** Step-by-step:
   1. `git subtree split --prefix=repos/<org>/<name> -b promote-<name> master` from parent root → produces clean linear history branch.
   2. Verify split history: `git log --oneline promote-<name>` shows expected commit count (touch-count of the prefix path in parent's history).
   3. Push to new remote: `git push <remote-url> promote-<name>:master` (explicit master target — GitHub may auto-create `main` otherwise).
   4. Parent index removal: `git rm -r --cached repos/<org>/<name>` (cached form preserves working tree).
   5. Stash working tree: `mv repos/<org>/<name> /tmp/<name>-stash` (for integrity diff).
   6. Add submodule: `git submodule add <remote-url> repos/<org>/<name>` (re-clones from remote).
   7. Verify integrity: `diff -r /tmp/<name>-stash repos/<org>/<name> --brief --exclude=target --exclude=.venv --exclude=__pycache__ --exclude=.git`. Expected: only gitignored runtime artifacts differ (because the remote was clone-only).
   8. Restore gitignored runtime artifacts from stash if operator needs working-tree continuity: `cp <stash-paths> <submodule-paths>`.
   9. Cleanup stash: `rm -rf /tmp/<name>-stash`.
   10. Verify `git submodule status` shows the new submodule at the expected SHA.

3. **Footer acknowledgments.** Add 1-line post-promotion footer notes to (a) any report citing in-tree paths (e.g., capabilities report); (b) any runbook citing submodule-relative paths (e.g., deployment SOP). Format: "**Post-promotion note (YYYY-MM-DD-HHMM):** `repos/<org>/<name>/` is now a git submodule; all cited paths continue to resolve at the same working-tree mount point." DON'T bulk-rewrite historical body content — that produces rolling truth.

4. **Worked example.** This loop's resink-core promotion (2026-05-12-1254). Cite the exec summary + the resulting `.gitmodules` entry + the submodule's `master` SHA at promotion time.

5. **Anti-patterns.**
   - **Fresh `git init` + single-import commit.** Loses history; future blame untraceable. Use `git subtree split` instead.
   - **Bulk-rewriting historical references.** Produces rolling truth — past records lose authoring-time accuracy. Use footer-acknowledgment pattern instead.
   - **Skipping the working-tree diff verification.** Risks silent data loss (gitignored files exist in working tree but not in remote — operator confusion if not restored).
   - **Omitting explicit branch name in push.** GitHub default-branch auto-creation may surprise; specify `:master` (or `:main`) explicitly.

6. **Sibling section: footer-acknowledgment pattern.** Per retro P4 — extend this playbook with a section codifying the 1-line footer pattern for structural-but-non-disruptive changes that preserve the working-tree mount point. Pattern is broader than submodule promotion (applies to any structural change that preserves surface).

## Mandated edits (if ratified)

- New file: `org-os/playbooks/submodule-promotion.md` (`type: playbook`).
- Cross-link from `org-os/playbooks/provisional-and-migrate.md` § "Sister patterns" (if it has such a section) OR add a one-line "Related playbooks" footer pointing at it.

## Alternatives considered

1. **Don't codify; let future operators re-derive the procedure each time.** Rejected because the procedure has non-obvious steps (the working-tree stash + diff verification; the `--cached` form of `git rm`; the explicit `:master` in push). Re-deriving risks tool-choice errors.

2. **Codify as a section in `provisional-and-migrate.md` rather than a sibling playbook.** Considered. Reason for sibling: the two patterns are independent (one is about artifact-type-coordination; the other is about repo-structural separation). Sibling preserves discoverability.

3. **Codify in `org-os/rituals/` rather than `org-os/playbooks/`.** Rejected because rituals are recurring procedures; submodule promotion is an event-driven action that may never recur for some teams. Playbooks are the right shape for invocable patterns.

## Risks

- **Pattern drift.** As future promotions land, the worked-example body grows; future loops might find the playbook's examples don't match their case shape. Mitigation: each invocation appends its own worked example.
- **Over-codification.** If only resink-core ever promotes, the playbook is a single-example artifact that took authoring effort. Mitigation: even single-example playbooks pay off in onboarding (future operator reading the playbook avoids re-deriving the procedure).

## Ratification path

One-loop-out draft-and-ratify. This ADR's `status: draft` at 2026-05-12-1254; the next loop's brief authoring includes an O-level objective to author the playbook body + flip this ADR to `status: active`. Sister precedents: ADR-2026-06-06-001 (provisional-and-migrate playbook, drafted at 2026-06-06 + ratified at 2026-06-13); ADR-2026-05-12-001 (reframe-vs-act playbook, drafted at 2026-05-12-0645 + still pending ratification — bundling both at the same next-loop batch is efficient).

## Decision

**(Draft.)** Author `org-os/playbooks/submodule-promotion.md` per the body shape above; ratify at next loop alongside the other deferred playbooks (P3/P4/P5 from prior retro + P1 from this retro).
{% endraw %}
