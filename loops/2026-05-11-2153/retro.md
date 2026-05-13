---
layout: default
title: Retro — 2026-05-11-2153
date: 2026-05-11
status: active
type: retro
loop: 2026-05-11-2153
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-11
  status: active
  loop: 2026-05-11-2153
  links: parent: board/exec-summaries/2026-05-11-2153.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-06-13

## What worked

- **GitBook publishing pipeline shipped on first attempt.** 159 files generated, idempotent re-run produced empty diff, defense-in-depth refusal of `org-os/` writes held, manual invocation pattern proved the script's correctness without CI dependency. The pipeline opens an entire class of follow-on work (CI integration, layout iteration, scope expansion, theme refresh) without coupling them to this loop. Reproducibility: when shipping a new publishing or transformation pipeline that has a loose coupling to surface details, ship the pipeline + first-publication FIRST and defer everything else (CI integration, theme polish, advanced layouts) — those are downstream of "does this work end-to-end at all." This is the third iteration of the "ship the smallest thing that exercises every component" pattern (focus loop 2026-05-11-0958; consolidation loop 2026-05-11-1113; observability loop 2026-05-11-2153).

- **The 4-ADR ratification batch landed cleanly with no merge friction.** Six+ `org-os/` files edited across four ADRs in a single loop, plus the same-loop draft-and-ratify of ADR-2026-06-13-001. AE was paused this loop, removing the parallel-writer coordination overhead from 2026-05-30; sim-farm was also paused. Per-edit dry-run + final cumulative tenant-isolation sweep returned only the pre-existing `acme.ai` placeholder. **The 2026-05-30 retro's prediction held: serializing org-os edits within board's own work stream — when no parallel team writers are active — eliminates the merge-coordination tax.** Reproducibility: when a loop carries 3+ ADRs with mandated `org-os/` edits, prefer pausing teams that don't have non-org-os work (saves coordination); use parallel writers only when teams have substantial product-repo work that doesn't touch `org-os/`.

- **First end-to-end cross-team request lifecycle fulfilled (sim-farm → DE schema-JSON).** Filed `open` at 2026-06-06; accepted-deferred mid-loop with `deferred_to_loop: 2026-06-13`; fulfilled at 2026-06-13 by DE's new convention. The mechanism specified in ADR-2026-06-06-002 (paused-team request acceptance canonicalization, ratified mid-loop today by O2) had its first worked example land within hours of the ratification. The mid-loop sequencing (ratify ADR → file ticket per the new field → close ticket using the ratified mechanism) is unusual but legal — the ratification flips draft → active in O2's body authoring; later steps in the same loop honor the new mechanism. Reproducibility: when an ADR ratifies a process change and the change has an immediate worked example available the same loop, exercise the new mechanism within the same loop's later objectives — it's the strongest possible "verified-against-environment" check for the ADR itself.

- **The DevOps minikube smoke pattern shifted via reframing rather than acting.** Board-action 2026-06-06-002 closed `superseded` at 2026-06-13 not because minikube was installed, but because the brief's working answer to retro P5 reframed external-toolchain blockers as per-team Prerequisite docs rather than board-action tickets. The 4-loop deferral resolved structurally. **The pattern is: when a recurring blocker has the wrong shape (not the wrong remediation), reframe the blocker class, not the specific blocker.** Reproducibility: future blockers that recur 3+ loops with the same "single human action" framing should trigger a reframing-vs-acting question at the next retro (option C: change the loop machinery's expectations).

- **resink-core's dlopen swap closed without surprises.** AE's template extension (loop-1 deliverable) integrated cleanly with resink-core's pre-staged `plugin_loader.rs` scaffold (loop-1 deliverable). The integration test passed on first attempt; the wrong-table BLOCKED status code mapping matched AE's documented semantics; the byte-identical output assertion held against the static-linking path. **The 2-loop pre-stage + integrate cadence proved the right size for a multi-team FFI handoff.** Reproducibility: when team A and team B share an FFI/ABI contract, stage the contract definition (template + types) in loop N, the consumer's scaffold (feature flag + no-op shim) in the same loop N, and the integration in loop N+1 against a real artifact built from the contract. The 2-loop window means the consumer doesn't have to wait for the producer's polish; the producer doesn't have to wait for the consumer's integration.

- **The same-loop draft-and-ratify pattern reused for ADR-2026-06-13-001.** Sister precedent at 2026-05-23-001 (enum extension drafted at 2026-05-23, ratified at 2026-05-30; later extended same-loop to add `report` at the 2026-05-30 brief authoring). This loop, ADR-2026-06-13-001 was both drafted from scratch AND ratified within the same brief authoring step. Works for mechanical batched extensions (admit a single new enum value). Reproducibility: when an ADR's content is bounded, mechanical, and has clear sister precedent, prefer same-loop draft-and-ratify over deferring a draft to the next loop. The cost is small (one more ADR in the loop's batch); the benefit is closing the provisional-and-migrate cycle for any in-flight artifacts in the same loop.

## What didn't

- **Workspace promotion 5th-loop carry; ticket carries another loop without resolution.** Board-action 2026-06-06-001's forcing function fired this loop (named 5 downstream blockers explicitly in the brief's Out-of-scope), and resink-core's loop+1 supervisor swap commit graph is now fully in-tree as a result — when promotion eventually happens, the cross-tree git history will need cleanup. **Root cause:** the forcing function names downstream blockers but doesn't generate ACTION on the blocker. The single human who would create the remote is the same human who reads the ticket; the brief's forcing function adds urgency text but no urgency mechanism. The 5-loop pattern is now structural — there's no in-loop machinery that resolves a blocker requiring out-of-loop human action. See P1.

- **GitBook publishing pipeline ships without verifying the rendered site.** O1 KR1.4 explicitly named "verify the rendered site at `blog.resink.ai` (or fallback `resink-ai.github.io`)" but the publishing PR is open (PR #1 on the github.io repo) and the GitHub Pages build won't trigger until the PR merges to `main`. The verification step is therefore deferred to post-merge — meaning the loop closes with the rendering verification incomplete. **Root cause:** the publishing branch was named `master-2026-06-13-publish-1` (not `main`); GitHub Pages builds from `main` only by default. The KR's verification step assumed merge would happen in-loop, but the PR-and-merge pattern is human-gated (consistent with the marketplace + parent PR-and-merge cadence). Mild — not a blocker, just a deferral. Mitigation: the next loop's brief or this loop's exec-summary should call out the post-merge verification as a checklist item so it doesn't get forgotten. See P2.

- **The integration test in resink-core's dlopen_integration.rs uses a hardcoded fixture event** (`user_id="u-001"`, etc.) rather than driving from the existing synthetic-tenant fixture parquet. The test is correct; it's just narrower than the full mvp-loop coverage. The wrong-table BLOCKED test exists, but exhaustive Op variants (Update, Delete) aren't separately exercised at the FFI boundary. **Root cause:** time-box on the loop kept the integration test minimal; the full coverage matrix would be a follow-up if the FFI surface evolves. **Pattern:** integration tests at FFI boundaries tend to grow narrow over time as the static-linking-path tests cover the same logic. Mild; reproducibility: when an FFI integration test exists alongside a static-linking-path test for the same logic, document the coverage equivalence in the test comment so future readers don't expand the FFI test redundantly.

- **The publishing scope decisions documented in `scripts/PUBLISHING-SCOPE.md` are first-iteration; reading the rendered site will surface scope adjustments.** The loop's `Out of scope this loop` already named several deferred decisions: `org-os/` engine docs publishing, `status.md` files, `proposals/` + `requests/` standalone pages, `values.schema.json`. These are deferred-not-decided. Once the rendered site exists and outside readers scan it, the question "where do I find X?" will surface gaps. **Pattern:** documentation site scope is iteratively discovered, not pre-specified. Reproducibility: the next loop's retro (or the loop after) should review the rendered site against the original scope, classifying each "couldn't find X" as an in-scope-but-missing case (script bug) vs. out-of-scope-and-needs-revisit case (scope expansion).

## Evolution proposals

### P1: Convert the workspace-promotion ticket from "ask the human" to "generate the command and run it via `! prefix`" (class: **tenant**)

- **Problem it solves:** "What didn't" #1. Workspace promotion 5-loop carry; the existing forcing function names downstream blockers but doesn't generate action. The single human reads the ticket and the urgency text, but the in-loop machinery has no escalation mechanism beyond text.
- **Proposed change:** The 2026-06-20 brief's Out-of-scope section names not just the count of blockers (existing forcing function) but also generates the EXACT command the human needs to run, formatted for `! prefix` execution: "If unblocking workspace promotion this loop, run: `! gh repo create resink-ai/resink-core --private --description 'Resink.ai nanofab runtime, training pipeline, sim-farm engine, and orchestrator'` then notify the brief in chat to re-trigger the resink-core promotion attempt." This converts the ticket from "ask" to "ready-to-execute" — the human's action becomes a one-line copy-paste rather than navigating to a GitHub UI. If the human runs the command in-loop, the brief's later read-step picks up the remote's existence and resink-core's promotion attempt fires within the same loop.
- **Review path:** Tenant decision; lives inside next CEO brief authoring. The pattern itself is general (any tracked external-action ticket where the action is a CLI command) — could later be folded into the `board/actions/<date>-<slug>.md` artifact template as a "Suggested execution" subsection.
- **Owner:** board.
- **Timing:** **Picked up loop 2026-06-20** (brief authoring includes the generated command in Out-of-scope when 2026-06-06-001 is still `open`).

### P2: GitBook publishing post-merge verification as standing post-loop checklist item (class: **org-os**)

- **Problem it solves:** "What didn't" #2. The first publication's rendered-site verification step is deferred to post-merge; without a standing checklist, future publishing loops will repeat the same gap.
- **Proposed change:** Extend `scripts/publish-to-gitbook.py`'s docstring + `scripts/PUBLISHING-SCOPE.md` to specify the **post-merge verification protocol**: after the github.io PR merges, the next loop's brief or exec-summary cites the rendered URL (e.g., `https://blog.resink.ai/loops/<date>/`) and confirms the new content is visible. If GitHub Pages build failed (the repo's Settings → Pages page would surface the build error), the next loop's brief opens an objective to fix. **OR** make the publishing script optionally drive the merge directly via `gh pr merge` after a `--auto-merge` flag — but this is heavier and couples publishing to GitHub-Pages-build state.
- **Review path:** Light — the protocol is documented in the publishing script's own docstring + README, not in `org-os/`. No ADR mandatory unless the protocol becomes a ritual (which it should NOT this loop — it's a single tenant-level publication concern, not an org-wide ritual).
- **Owner:** board (publishing script docstring + PUBLISHING-SCOPE.md update).
- **Timing:** **Picked up same loop as the github.io PR merge** — i.e., whenever the user merges PR #1, the publishing script's docs gain the post-merge verification protocol. No deferral needed.

### P3: Reading-experience review of the rendered GitBook against original scope (class: **tenant**)

- **Problem it solves:** "What didn't" #4. Publishing scope decisions are first-iteration; the rendered site will surface "couldn't find X" cases that classify as either in-scope-but-missing (script bug — fix) or out-of-scope-and-needs-revisit (scope expansion — propose).
- **Proposed change:** The first loop AFTER the github.io PR merges (target: 2026-06-20 if merged in-loop, else 2026-06-27) opens a "GitBook reading-experience review" objective. Walk the rendered site as a fresh outside reader; classify any "couldn't find X" as bug-vs-scope. Bug: file a quick fix in `scripts/publish-to-gitbook.py`. Scope: file a follow-up evolution proposal naming the desired addition.
- **Review path:** Tenant decision; lives inside next-or-after-next CEO brief authoring.
- **Owner:** board (brief authoring).
- **Timing:** **Deferred to loop 2026-06-20 or 2026-06-27** depending on merge timing.

### P4: Document the "2-loop FFI handoff pattern" in `provisional-and-migrate.md` or as a sibling playbook (class: **org-os**)

- **Problem it solves:** "What worked" #5. Resink-core's dlopen swap closed without surprises because of a 2-loop pre-stage-and-integrate cadence with AE. The pattern (loop N: producer ships contract, consumer scaffolds; loop N+1: consumer integrates against real producer artifact) is reusable for any cross-team FFI/ABI handoff. Without documentation, future cross-team work may serialize unnecessarily (consumer waits for producer's polish) or skip the scaffold step (consumer gets surprised at integration time).
- **Proposed change:** Extend the newly-ratified `org-os/playbooks/provisional-and-migrate.md` with a "Sibling pattern: 2-loop FFI handoff" section, OR author a sibling playbook at `org-os/playbooks/cross-team-ffi-handoff.md`. **Recommend: sibling section in `provisional-and-migrate.md`** — both patterns are about coordination through a contract definition, just at different scales (frontmatter convention vs. FFI/ABI). The shape: producer ships contract definition + minimal artifact (template, type signatures) in loop N; consumer scaffolds (feature flag + no-op shim against the contract) in loop N; consumer integrates against real producer artifact in loop N+1. Cite the AE (template at 2026-06-06) → resink-core (scaffold at 2026-06-06; integrate at 2026-06-13) example.
- **Review path:** ADR-class change to a same-loop-just-ratified playbook. **Recommend: extend ADR-2026-06-06-001 in place** (sister to the loop-after extension precedent ADR-2026-05-16-004 used at 2026-05-30 for consolidation-loop pattern). Same author (board), same loop +0 from ratification (this retro), mechanical addition to the playbook.
- **Owner:** board.
- **Timing:** **Picked up loop 2026-06-20** alongside the ADR-2026-06-06-001 extension. Bundle with the 2026-06-13 reading-experience review work if no other org-os work crowds the loop.

### P5: Authoring an `org-os/` engine doc-set publishing track (separate namespace from tenant gitbook) (class: **org-os**)

- **Problem it solves:** GitBook publishing intentionally excluded `org-os/` this loop (engine internals; tenant-agnostic; would need separate namespace). But `org-os/` is the most-reusable artifact in the repo — it's the entire org operating system that other tenants could adopt. Without publishing, the engine's documentation surface is locked to anyone outside this repo.
- **Proposed change:** Author a sibling publishing pipeline that publishes `org-os/` (rituals, playbooks, conventions, templates, roles, README) to a separate location — either a separate repo (e.g., `resink-ai/org-os-public`) or a separate subpath at `blog.resink.ai/engine/`. Tenant-agnostic content; no `board/`/`teams/`/tenant artifacts. The pipeline would be a sibling of `scripts/publish-to-gitbook.py` (call it `scripts/publish-org-os.py`), sharing the same Jekyll-organization patterns. **Open question:** is `org-os/` ready to be a standalone project / installable starter kit? Or is it documentation-only? The answer affects whether the publishing target is a doc site (read-only) or a usable starter (clone-and-go).
- **Review path:** ADR mandatory if standalone starter (it's a new project surface); tenant decision if docs-only (lives in a sibling submodule analogous to the gitbook one).
- **Owner:** board.
- **Timing:** **Deferred to loop 2026-06-27 or later.** Requires (a) the tenant gitbook's first publication to settle (post-merge verification + reading-experience review), (b) a CEO question on standalone-vs-docs-only, (c) bandwidth not consumed by the workspace promotion fallout if 2026-06-20 doesn't resolve it.

## Decisions to record

New ADR placeholders this loop:

- (P1 needs no ADR — tenant-class, brief's Out-of-scope edit pattern only.)
- (P2 needs no ADR — script docstring + PUBLISHING-SCOPE.md update.)
- (P3 needs no ADR — tenant-class, future brief objective.)
- **P4 will extend ADR-2026-06-06-001 in place** rather than spawn a new ADR (sister to ADR-2026-05-16-004's consolidation-loop extension precedent).
- **P5** may require a new ADR depending on the standalone-vs-docs-only answer; **draft placeholder deferred until the answer is decided** at 2026-06-27 brief.

**Net new ADRs this retro: 0.** All proposals route through tenant-class action OR same-loop-just-ratified ADR extensions. The 2026-06-20 batch is unusually small at retro-authoring time (likely a brief-only or brief-plus-one-extension batch), which is consistent with the 4-ADR batch this loop having absorbed the org-os process surface comprehensively.

## Carryover ADRs / open work still on the books

- **Workspace promotion blocker** (board-action 2026-06-06-001): 5-loop carry; ticket stays `open`. P1 from this retro proposes the `! prefix` reframing for next loop's brief.
- **GitBook publishing PR #1** (resink-ai/resink-ai.github.io): awaits user merge; post-merge verification carries to next loop or whichever loop the merge happens.
- **Sim-farm verdict-contract back-reference to DE convention:** mechanical migration, sim-farm next-loop deliverable.
- **Hot-swap correctness test (ADR-2026-05-16-001 step 3):** loop+1 = 2026-06-20; joint AE + resink-core deliverable.

Multi-loop plan adjustments (not new ADRs, just plan-side updates):

- **ADR-2026-05-16-001 (Option A → dlopen restoration):**
  - Step 1 (AE template extension): ✅ closed at loop-1 (= 2026-06-06).
  - Step 2 (resink-core supervisor swap full execution): ✅ closed at loop+0 (this loop, = 2026-06-13).
  - Step 3 (hot-swap correctness test): scheduled `loop+1` (= 2026-06-20). Joint AE + resink-core. Test fixture exercises a deliberate logic change in the codegen output between v1 and v2 of the same `dim_user_scd2` node. Demonstrates that a node-version bump can land without supervisor restart and that in-flight events are not lost across the swap.

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: **passed**. Board ran per-edit dry-runs after each of the ~6 `org-os/` edits across the 4 ADR ratifications + 1 cumulative final sweep. AE made zero `org-os/` writes (paused). Resink-core, DE, DevOps, sim-farm: zero `org-os/` writes. Combined: only the pre-existing `acme.ai` placeholder reference in `org-os/conventions.md` matched. **The largest concentration of `org-os/` writes since 2026-05-30 (Bundle C + 3 ADR ratification at that loop) completed with zero tenant-isolation violations.** The new playbook `org-os/playbooks/provisional-and-migrate.md` (authored fresh per ADR-2026-06-06-001) was authored carefully against the regex-inlining anti-pattern from 2026-05-30 (references `conventions.md § Linking` rather than spelling tenant strings out). Pass.
{% endraw %}
