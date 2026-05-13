---
layout: default
title: Retro — 2026-05-12-1254
date: 2026-05-12
status: active
type: retro
loop: 2026-05-12-1254
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-12
  status: active
  loop: 2026-05-12-1254
  links: parent: board/exec-summaries/2026-05-12-1254.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-12-1254

## What worked

- **The 8-loop workspace-promotion saga closed.** Multi-loop blocker resolution through cumulative pattern-application: filing (2026-05-16) → board-action with forcing function (2026-06-06) → forcing function fires (2026-06-13) → reframe-vs-act applied + auto-run-command reframing (2026-05-12-0645 retro P1) → command runs (2026-05-12-0645 close) → mechanical structural execution (this loop). Each loop's iteration refined the pattern; the resolution wasn't a single insight but a sequence of partial unblockers. **Reproducibility:** multi-loop blockers may need multiple cycles of pattern refinement, not a single attempt-and-retry. The right move is to keep iterating on the FRAMING each loop the blocker carries, not to apply more pressure to the existing framing.

- **`git subtree split` was the right tool.** Faithful loop-level commit subjects preserved (7-commit linear history on the new remote; each commit's subject is "Loop YYYY-MM-DD: ..."); 19 walked → 7 split; no surprises. The alternative (fresh `git init` + single import) would have lost the loop-coupling and made future blame untraceable. **Reproducibility:** for any in-tree → submodule promotion of an active development tree, prefer `git subtree split` over fresh-init. Codifying this is worthwhile — see P1.

- **No path rewrites needed across 75 referencing files.** The submodule mounts at the same working-tree path as the in-tree directory; all relative references resolve identically. Live consumers (chart, publish-to-gitbook.py, SOP) work without modification. The 1-line footers on the capabilities report and SRE runbooks are the only live-reference acknowledgments needed. **Reproducibility:** for any structural separation that preserves the working-tree mount point, prefer additive footer-acknowledgments over bulk path-rewrite. Bulk rewrites introduce surface-area risk for zero functional gain.

- **Single-focus loop discipline.** Three retro proposals from last loop (P3 + P4 + P5) were intentionally deferred to keep this loop tight on workspace promotion. The mechanical-but-multi-step git operation landed cleanly with zero scope-creep. Trade-off: org-os process evolution slows by a loop. The trade is favorable when the loop's focus carries a clean-execution risk (one slop in the subtree-split or submodule-add could have produced a partial-state mess). **Reproducibility:** when a loop's focus is mechanical-but-multi-step and carries clean-execution risk, defer process-evolution work to the next loop. Don't bundle unrelated org-os edits with risky mechanical work.

- **Second consecutive zero-`org-os/`-writes loop.** Pure tenant-surface work; the org-os process surface is stable enough that several consecutive loops can avoid touching it. Tenant-isolation invariant held trivially. **Reproducibility:** stable org-os process surfaces are healthy — `org-os/` edits should be load-bearing change, not housekeeping. Trend toward minimal-surface impact when possible.

## What didn't

- **The capabilities report's historical body still cites in-tree paths.** Footer note acknowledges the post-promotion state, but a reader scanning the body sees "the file at `repos/resink-ai/resink-core/crates/...`" without immediately recognizing this is now a submodule. **Root cause:** the body is historical state, true at the time of authoring (2026-05-30). Bulk rewriting historical artifacts produces a "rolling truth" effect where past records are no longer faithful to their authoring moment. The footer-note approach is the right trade — historical truth preserved + current state acknowledged. Mild: any reader confused by the discrepancy reads the footer first.

- **`helm lint --strict` against default `values.yaml` has always failed.** Pre-existing chart behavior surfaced as a side-effect of DevOps re-running gates this loop. The chart's `tenant` field has no sensible default (it's required-non-empty), so the JSON schema's `minLength: 1` rejects the default `""`. This wasn't a regression; it was just newly-visible. **Root cause:** chart values' defaults don't satisfy the chart's own schema unless values overrides supply tenant. **Mild bookkeeping:** future loop could either set a placeholder default like `tenant: "REPLACE_ME"` (which then fails the schema's pattern instead of minLength — different failure mode), OR document the chart-default-fails-strict-lint behavior in chart README. The right fix depends on whether you want default-state to be auto-lintable.

- **No CI/CD on the new resink-core submodule.** Empty repo with no GitHub Actions; future contributors land changes with only local-build verification. **Root cause:** out of scope for this loop's promotion focus. **Sized small:** an initial cargo-build + helm-lint workflow is ~20 lines of YAML. Pickup at next resink-core-touching loop. See P2.

- **No branch protection rules on the new remote.** Anyone with org access could force-push to master. **Mild:** the only contributor currently is the loop author; risk is theoretical until contributors land. **Sized small:** GitHub UI click-through (or `gh api` call). Pickup alongside CI/CD or as a one-off. See P2.

## Evolution proposals

### P1: Author `org-os/playbooks/submodule-promotion.md` codifying the `git subtree split` pattern (class: **org-os**)

- **Problem it solves:** "What worked" #2. This loop's workspace promotion used `git subtree split` — the right tool, but the choice was based on the loop author's prior knowledge, not on a documented playbook. Future in-tree → submodule promotions (e.g., sim-farm extracting to its own repo when first non-Python component lands; potential `services/<name>/` from home-cluster roadmap Phase 1) would benefit from a written playbook with the worked example anchor.
- **Proposed change:** Author `org-os/playbooks/submodule-promotion.md` with `type: playbook`, `invocation_trigger: "when an in-tree directory under repos/<org>/<name>/ needs to become a real git submodule"`. Body sections: (a) when to invoke (in-tree development is constraining cadence; a separate repo would unblock CI/CD, branch protection, contributor flow); (b) the procedure (subtree split → push → rm --cached → mv stash → submodule add → diff --brief → restore gitignored artifacts → footer acknowledgments); (c) worked example (this loop's resink-core promotion); (d) anti-patterns (don't fresh-init + single-import — loses history; don't bulk-rewrite historical references — produces rolling truth).
- **Review path:** ADR-class change. Draft a placeholder ADR `2026-05-13-001-submodule-promotion-playbook.md` (or whatever next loop's date is) for next-loop ratification. Sister precedent: ADR-2026-05-12-001 (reframe-vs-act playbook, drafted last retro, still pending ratification) — bundling submodule-promotion-playbook + reframe-vs-act ratification in the same next-loop org-os batch is efficient (both are org-os playbook authoring).
- **Owner:** board.
- **Timing:** **Picked up next loop alongside P3/P4/P5 from prior retro's deferred batch.** The bundle becomes 4 org-os edits: reframe-vs-act playbook body + ADR-2026-05-12-001 ratification (was P3 last retro); ADR-2026-05-16-003 scope extension to runbook (was P4); first-real-deploy playbook section (was P5); submodule-promotion playbook + new ADR draft (this retro's P1). Sized M.

### P2: Set up initial CI/CD + branch protection on the new resink-core submodule (class: **tenant**)

- **Problem it solves:** "What didn't" #3 + #4. The new resink-core remote has no CI workflows and no branch protection. Both are minimal additions but they matter once a second contributor lands.
- **Proposed change:** In the new resink-core repo's master branch, add `.github/workflows/ci.yml` with two jobs: (a) `cargo build --release -p nanofab-supervisor` (gate the Rust workspace); (b) `helm lint --strict deploy/charts/nanofab-supervisor --values deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml` + `helm template` + `helm install --dry-run` (gate the chart). Plus branch protection via GitHub UI (or `gh api`): require PRs to master, require status checks to pass, require at least 1 review. **Optional later:** add a `cargo test` job; add a sim-farm pytest job; add cross-arch docker build verification.
- **Review path:** Tenant decision (chart-internal; new repo's CI). No ADR. Single PR on the new resink-core repo. Sized S.
- **Owner:** teams/application/resink-core.
- **Timing:** **Picked up next loop's resink-core build phase**, OR opportunistically the loop after.

### P3: Establish a "multi-loop blocker arc" report-type for retros (class: **org-os**)

- **Problem it solves:** "What worked" #1. This loop's retro names the 8-loop arc of workspace promotion as a pattern reflection. The arc is reusable knowledge — "what does multi-loop blocker resolution look like when it works?" But the reflection only appears in retro bodies, not in any reusable artifact type. Future retros might or might not surface arc-level reflections; no template structure exists.
- **Proposed change:** Add a new artifact type `blocker-arc-report` to the conventions enum, OR extend the existing `report` type to admit arc-shape reports as a subset. The artifact captures: (a) the blocker's name + originating loop; (b) the per-loop pattern-evolution (filing → forcing function → reframe → ...); (c) the resolution loop; (d) the reusable pattern (when this shape recurs, the approach is...). Lives under `board/reports/` per existing `report` type's directory.
- **Review path:** ADR-class change (conventions enum extension). Defer to a later loop — this is the LOWEST priority of the proposals; codifying retro reflection patterns is meta-process work that should wait until 2+ arcs have closed to provide multi-example anchoring. **Recommend: revisit in 3-4 loops** after another multi-loop blocker arc closes (e.g., hot-swap step 3 multi-loop saga, or the GitBook publishing post-merge verification arc).
- **Owner:** board.
- **Timing:** **Deferred** 3-4 loops out.

### P4: Document a "footer-acknowledgment" pattern for structural-but-non-disruptive changes (class: **org-os**)

- **Problem it solves:** "What worked" #3. This loop used 1-line footer notes (on the capabilities report + 2 SRE runbooks) to acknowledge the structural change WITHOUT rewriting historical content. The pattern is reusable for any structural change that preserves the working-tree surface (mount point unchanged; relative paths still resolve). Future structural changes (e.g., a future sim-farm extraction; a future docs reorganization) would benefit from the same pattern.
- **Proposed change:** Extend `org-os/playbooks/submodule-promotion.md` (if authored per P1) with a sibling section "Sibling pattern: footer-acknowledgment for non-disruptive structural changes." OR author a standalone short playbook. The shape: (a) when to invoke (structural change preserves working-tree surface); (b) the format (single-line footer with date + change description + cross-link to triggering loop's exec summary); (c) anti-pattern (don't bulk-rewrite historical body content — produces rolling truth).
- **Review path:** Sibling section in P1's playbook is cleanest. No separate ADR (extension to P1's same playbook).
- **Owner:** board.
- **Timing:** **Bundled with P1's playbook authoring** at next loop.

## Decisions to record

New ADR placeholders this loop:

- **P1 generates a draft ADR placeholder** at `board/decisions/2026-05-13-001-submodule-promotion-playbook.md` (or appropriate next-loop date) for next-loop ratification. Bundled with ADR-2026-05-12-001 (reframe-vs-act playbook, prior retro's draft) — both playbook-shaped, both org-os, both pickup-able in the same next-loop batch.

Actually — I'm drafting this ADR placeholder *this* loop so the next loop's brief has it in `status: draft` to ratify.

- **P2 needs no ADR** (tenant-class, new repo's CI setup).
- **P3 deferred** — no ADR placeholder yet; revisit after another arc closes.
- **P4 extends P1's playbook in place** — no separate ADR.

**Net new ADR drafts this retro: 1** (P1's submodule-promotion playbook ADR).

## Carryover ADRs / open work still on the books

- **Hot-swap correctness test (ADR-2026-05-16-001 step 3):** was loop+2 (= this loop), **slipped to loop+3** (the loop after next). Joint AE + resink-core deliverable. Single-loop slip; bounded.
- **ADR-2026-05-12-001 (reframe-vs-act playbook):** still `status: draft` from prior retro's P3. Body authoring + ratification target: next loop's bundled org-os batch.
- **ADR-2026-05-16-003 (Verified-against-environment) scope extension to `runbook`:** prior retro's P4. In-place extension; target: next loop's bundle.
- **First-real-deploy playbook section in `provisional-and-migrate.md`:** prior retro's P5. In-place extension; target: next loop's bundle.
- **Workspace promotion:** ✅ CLOSED. Action ticket `2026-06-06-001` already at `status: done`. The 8-loop arc resolves.
- **`Chart.appVersion` ↔ supervisor crate version alignment:** bookkeeping.
- **Cargo MSRV declaration sync to Cargo.lock:** bookkeeping.

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: **passed**. Final tenant-isolation grep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns only the canonical `acme.ai` placeholder at `org-os/conventions.md:121`. **Second consecutive loop with zero `org-os/` writes by any team.** Pass.
{% endraw %}
