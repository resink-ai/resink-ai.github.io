---
layout: default
title: Retro — 2026-05-13-0859
date: 2026-05-13
status: active
type: retro
loop: 2026-05-13-0859
owner: board
grand_parent: Loops
parent: Loop 2026-05-13-0859
nav_order: 3
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-0859
  links: parent: board/exec-summaries/2026-05-13-0859.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-13-0859

## What worked

- **Smallest-product-slice playbook's first real application closed the v1 scope decision cleanly.** Authored last loop (2026-05-13-0056 § O5); applied this loop to choose between four candidate observability slices (k8s snapshot / k8s+logs / metrics / workspace introspection). The prerequisites table inside the brief made the decision explicit — k8s snapshot has the fewest prereqs (kubectl already worked), so v1 ships that. Rejected slices archived as future-loop work, not lost. **Reproducibility:** when scoping any v1 surface with multiple candidate slices, the prerequisites table goes IN the brief, not on a separate planning doc. The playbook's worked-example shape (the 4-row table from the prior loop's CLI scoping) reproduces directly.

- **Hot-swap split (option b) absorbed a third-consecutive slip without dedicating an entire loop.** Prior retro § P1 forced a CEO decision; option (b) — split into AE-only + resink-core-only sub-deliverables — let AE ship its half (template fixture pair) alongside the unrelated O1 observability work in this loop, while the resink-core half bundles naturally with retro P2/P3 at the next active-resink-core loop. The 3-consecutive-slip pattern is broken. **Reproducibility:** when a joint deliverable slips 3 loops with a single-focus framing, the split-into-halves-with-a-documented-contract pattern is a viable third path between "dedicate a loop to it" (option a) and "accept indefinite slip" (option c). Pre-condition: at least one half must be authorable independently — for hot-swap that was the template-side (no runtime semantics needed); for other joint deliverables, the split may not be tractable.

- **DEVIATION.md as the load-bearing artifact in the hot-swap split.** The split only works if the contract between the two halves is pinned with zero ambiguity. DEVIATION.md does that: one-line conceptual diff + source-line diff block + worked 3-row example with explicit expected-rows tables + invariants-preserved list + 6-step consumer-side acceptance contract. The resink-core author of the consumer-side test has a complete spec to implement against. **Reproducibility:** when authoring half of a split deliverable, the contract doc is the deliverable; the code is just a forcing function for the doc.

- **Inter-loop publishing reliability fixes shipped same day, out of loop.** Mid-loop check-in revealed gitbook CI had been failing 5 consecutive loops (Liquid parse errors on `{{...}}` snippets in code fences) and loop-index pages were 404-ing on `.md` extensions. Both fixes shipped as out-of-loop corrective PRs (gitbook #7/#8, parent newbase #16/#17) without disrupting the in-loop work. The publishing pipeline is now demonstrably healthy on main. **Reproducibility:** when out-of-loop infra breakage surfaces during a loop, the right answer is small, scoped corrective PRs landed in parallel — not a re-shape of the in-loop objectives. The corrective work is its own retro entry (in "what didn't"), not a reason to drop the loop's deliverables.

- **The `raw_copy=True` HTML path is now load-bearing.** The publish-to-gitbook script bypasses Liquid-wrapping for `*.html` files (`raw_copy=True`); the observability skill produces HTML at `board/reports/2026-05-13-cluster-snapshot.html`. The file shipped through the publish pipeline byte-for-byte, no Liquid escaping issues. The CSS rendered, no broken interpolations. **Reproducibility:** for any operator-readable report meant to render in a browser, HTML output is the right shape — markdown reports go through Liquid-wrap (correctly opaque to `{{...}}` patterns) but lose styling fidelity; HTML reports stay raw and look right.

## What didn't

- **Gitbook CI broke 5 consecutive loops before anyone noticed.** The publish PRs merged cleanly to main, but the post-merge Jekyll/Liquid build silently failed; live `blog.resink.ai` was frozen at the pre-failure state. Discovered mid-loop on a user check, not by any automation or post-merge discipline. Fixed via corrective PRs same day. **Mild-to-moderate:** the breakage was bounded (publishing only, no in-tree corruption), but the time-to-detection is the load-bearing miss. **Action required.** See P1.

- **Loop-index pages linked to `.md` URLs that Jekyll publishes as `.html`.** Separate from the Liquid issue. `https://blog.resink.ai/loops/<loop-id>/brief.md` 404'd; the correct path is `brief.html`. The generated index linked to `.md` because the build script was inconsistent with itself (decisions/actions/teams indexes correctly used `.html`; only the loop index was wrong). User-reported, not caught by any test. **Mild:** bug-class issue; same-day fix shipped (gitbook #8 + parent #17). **Codify a link-existence smoke test.** See P2.

- **No build smoke for the v2 template's slot-fill output.** AE shipped the v2 template + DEVIATION.md without running `cargo build` against an output crate (rustc/cargo not in the AE workstation env). The deviation is surgical (one method call's third argument changes); structurally it cannot introduce a compile error. But "structurally cannot" is a claim, not a verification. Consumer-side compile in resink-core's CI will catch any latent issue — but the gap surfaces as "the AE-side closes without independent build evidence." **Mild:** acceptable given the deviation's shape; surface in retro as a known gap to revisit when AE's workstation gets a toolchain. **No action this retro.**

- **AE charter's `date:` and the marketplace's `package.json` version are out of sync conceptually.** AE charter bumps to the current loop date as a "I worked on this team this loop" signal; package.json stays at `0.1.0` despite the new plugin landing. Both are correct per their own conventions, but a casual reader could conclude that no AE work happened since the prior bump. **Mild:** convention mismatch, not a bug. The plugin's version stays at `0.1.0` until a breaking change to the public surface; charter `date:` is a freshness signal. No action; flag for awareness.

## Evolution proposals

### P1: Post-publish CI verification as a standing practice (class: **tenant**)

- **Problem it solves:** "What didn't" #1. Five consecutive gitbook-publish loops failed silently because no one checked the CI status after the publish PR merged. Time-to-detection was a user observation, not a process artifact.
- **Proposed change:** After every `python3 scripts/publish-to-gitbook.py` + gitbook PR-merge step, the publish-to-gitbook ritual (or whatever does the publishing — currently the loop close-out in `rit-ceo-consolidation`) verifies the post-merge CI on the gitbook repo's main branch. Concretely: `cd repos/resink-ai/resink-ai.github.io && gh run list --branch main --workflow "Build and deploy Jekyll site to Pages" --limit 1 --json conclusion --jq '.[0].conclusion'` returns `success` before the loop is declared closed. If it returns anything else, the publish step is marked as a known issue in the exec summary or retro and a corrective PR is filed.
- **Recommendation:** **Codify as memory'd standing practice, not as an ADR or playbook.** The discipline is mechanical (one `gh run list` call); a memory entry on the agent's persistent store already captures it (added during the inter-loop corrective work today). If the discipline is missed in future loops, the memory triggers; if it surfaces as a load-bearing convention with multiple authoring points, then it earns a playbook or runbook entry.
- **Review path:** No ADR. Memory entry already exists; surface a one-line cross-link in the next CEO brief's "Standing CEO answers" section if the post-publish-verification pattern is invoked.
- **Owner:** memory'd; agent on the keyboard during loop close-out.
- **Timing:** **Already in force** as of this retro. Revisit if a sixth publish-CI failure surfaces undetected.

### P2: Link-existence smoke for the gitbook publish (class: **tenant**)

- **Problem it solves:** "What didn't" #2. The loop-index pages linked to `brief.md` etc., which Jekyll publishes as `brief.html`. No test caught the 404. The bug was 100% mechanical (the publish script's `build_loop_index_body` differed from the convention used in `build_decisions_index_body`, etc.) — a smoke test would have caught it.
- **Proposed change:** Add a link-existence smoke to `scripts/publish-to-gitbook.py` or as a separate `scripts/verify-gitbook-links.py` that: (i) walks the published Jekyll tree, (ii) extracts every `[text](path)` markdown link from generated index pages, (iii) resolves each path against the tree's expected output (markdown → `.html`), (iv) errors if any link points at a non-existent destination. Run as part of the publish step.
- **Recommendation:** **Sized S.** Standalone script (no Jekyll dependency — just markdown parsing + filesystem checks). Run as a pre-publish step inside the publish-to-gitbook script before commit. Adds ~30 lines.
- **Review path:** No ADR (CI-internal). Land as a parent newbase PR alongside any next publish.
- **Owner:** board (publish-script touch).
- **Timing:** **Next non-resink-core loop.** Bundle with whatever loop has a small board-side touch (or open as a standalone corrective PR if no natural loop arrives).

### P3: Resink-core consumer-side hot-swap test (class: **resink-core**)

- **Problem it solves:** Closes the second half of ADR-2026-05-16-001 step 3. AE's half closed this loop; resink-core's half is the load v1 → process e1+e2 → swap → process e3 → assert `valid_to` shifts as documented in `templates/scd2_maintainer_v2/DEVIATION.md` § "Consumer-side acceptance contract". 6-step procedure already pinned.
- **Proposed change:** Resink-core's test at `crates/nanofab-supervisor/tests/hot_swap_correctness.rs` (or similar) implements the 6-step procedure. Test loads two compiled cdylib artifacts (v1 + v2), processes synthetic events through each, captures `valid_to` of the closed predecessor for event e3, asserts v2's policy.
- **Recommendation:** **Bundle with prior retro P2 (CI/CD bootstrap on resink-core remote) + P3 (`make bootstrap` for submodule-deinit recovery) at the next active-resink-core loop.** Three resink-core items naturally fit one loop with one CI/CD + one runbook + one hot-swap test deliverable.
- **Review path:** No ADR (test-only; ADR-2026-05-16-001 already names step 3 in its plan).
- **Owner:** teams/application/resink-core.
- **Timing:** **Next active-resink-core loop.**

### P4: Observability v2 slice — schedule on real demand (class: **deferred**)

- **Problem it solves:** Anticipates the "what's the next observability slice?" question. Three candidates: logs slice, metrics slice (needs metrics-server in cluster), events-tail (streaming/follow).
- **Proposed change:** Don't schedule any of them now. The smallest-product-slice playbook's discipline is "ship the smallest viable thing and let real demand drive the next slice." When a future loop has a concrete signal — an incident where the cluster snapshot was insufficient, an operator asking for streaming events, etc. — that becomes the trigger.
- **Recommendation:** **Deferred candidate.** Revisit on demand.
- **Review path:** None unless invoked.
- **Owner:** AE.
- **Timing:** **Indefinite; demand-driven.**

## Decisions to record

New ADR placeholders this loop: **None.**

- P1 is a memory'd practice, not an ADR (codify-by-discipline rather than codify-by-document).
- P2 is a small script touch (CI-internal).
- P3 is execution of an already-active ADR plan, not a new decision.
- P4 is explicit deferral.

**Net new ADR drafts this retro: 0.** Consistent with the prior loop's "no new ADRs; backlog cleared" trend.

## Carryover ADRs / open work still on the books

- **ADR-2026-05-16-001 step 3 (resink-core consumer-side):** **AE-side closed.** Resink-core-side bundles with P2 + P3 + retro P2/P3 at next active-resink-core loop. The third-consecutive-slip pattern is now broken.
- **CI/CD setup on new resink-core remote:** prior retro P2; carries.
- **Submodule-deinit-then-regenerate recovery `make bootstrap`:** prior retro P3; carries.
- **Multi-loop-blocker-arc report-type:** 2026-05-12-1254 retro P3; revisit after hot-swap arc fully closes (i.e., after the resink-core consumer-side test lands).
- **5-section playbook body shape codification:** prior retro P4 (deferred, codification-without-motivation; revisit when motivated). No motivation surfaced this loop.
- **Link-existence smoke (P2 this retro):** next non-resink-core loop.

## Tenant-isolation dry-run

Held trivially this loop. Zero `org-os/` edits. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

The new content this loop lives entirely under `repos/resink-ai/resink-marketplace/plugins/` (observability plugin + nanofab v2 template) and `board/reports/` (HTML cluster snapshot) and `teams/platform/agent-engineering/` (charter + OKR + exec summary + status). None of these are under `org-os/`; the invariant has no edits to gate.
{% endraw %}
