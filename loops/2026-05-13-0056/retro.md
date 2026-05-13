---
layout: default
title: Retro — 2026-05-13-0056
date: 2026-05-13
status: active
type: retro
loop: 2026-05-13-0056
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-0056
  links: parent: board/exec-summaries/2026-05-13-0056.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-13-0056

## What worked

- **Bundled ratification loop cleared the entire org-os backlog.** 5 items in one focused authoring pass: ADR-2026-05-12-001 ratification + reframe-vs-act playbook; ADR-2026-05-12-002 ratification + submodule-promotion playbook (with footer-ack sibling section); ADR-2026-05-16-003 in-place scope extension + conventions.md sync edit; first-real-deploy sibling section in provisional-and-migrate.md; new smallest-product-slice playbook. No item slipped. **Reproducibility:** when 3-4 org-os authoring items accumulate across multiple single-focus product loops, dedicate one loop to bundled ratification. The cost is 1 loop of org-os focus; the benefit is the prior 2-3 loops kept clean single-focus discipline.

- **Per-edit tenant-isolation dry-run caught 2 violations early.** The reframe-vs-act playbook's worked-example used "resink-ai home cluster" verbatim from the retros; the submodule-promotion playbook's worked-example named the actual tenant repo (`repos/resink-ai/resink-core/`); the smallest-product-slice playbook named the binary `resink`. All three caught at per-edit dry-run; all three swapped for `<TENANT>` placeholders OR generic descriptions before commit. **Reproducibility:** when authoring org-os content from retro/exec-summary sources, the source content is tenant-aware; the playbook content must NOT be. Per-edit dry-run (not just final sweep) is the right discipline because final-sweep-only would require cleanup commits after the fact. The 2026-05-11-2153 4-ADR ratification batch established this; this loop confirms it at 5-edit scale.

- **In-place playbook extensions worked.** `provisional-and-migrate.md` gained a first-real-deploy sibling section; ADR-2026-05-16-003 gained a sister-artifact-types subsection. Both kept related patterns co-located rather than spawning new playbooks or new ADRs. **Reproducibility:** for sibling patterns to an existing playbook/ADR, prefer in-place sibling-section extensions over standalone artifacts. Codified now in the smallest-product-slice playbook's anti-pattern section (re-deriving-the-prerequisites-table) and in ADR-2026-05-23-001's existing in-place-extension precedent.

- **The 4-playbook coherent cluster.** `provisional-and-migrate.md` ↔ `reframe-vs-act.md` ↔ `submodule-promotion.md` ↔ `smallest-product-slice.md` — all four playbooks cross-link as sister patterns. Future readers can navigate the cluster as a logical unit. The cluster's shared shape (when-to-invoke; the question/procedure; worked examples; anti-patterns; sister patterns) is itself a meta-pattern worth recording. **Reproducibility:** when 3+ playbooks form a logical cluster, ensure mutual cross-links + a shared body shape. Reduces cognitive load for future authoring.

- **Conventions enum has been stable since 2026-06-13** (5 months ago in the loop timeline; ~6 weeks in wall-clock). No new types added; only scope extensions to existing types' body-shape rules. The enum's stability signals that the org-os process is maturing — new patterns extend existing types rather than proliferate new ones. **Reproducibility:** treat conventions enum extensions as load-bearing (require ADR + in-place edit + ratification). Body-shape extensions are lighter (in-place edit + ADR amendment).

## What didn't

- **Hot-swap step 3 slipped a THIRD time.** Originally loop+1; slipped to loop+2 (workspace promotion focus); slipped to loop+3 (UX focus); slipped to loop+4 (org-os ratification focus); now loop+5. **Root cause unchanged across all four loops:** the test is a joint AE + resink-core deliverable; every loop with a strong single-focus has crowded it out. The scheduling assumption ("joint deliverable lands the loop after step 2 closes") needs revision. **Mild:** ADR body grandfathers under absolute-date authoring; the slip is bounded but stale. **Decision required.** See P1.

- **No CI/CD on the new resink-core remote yet.** Per 2026-05-12-1254 retro P2; carried for two loops since. The remote's master gets pushes directly (per the resink CLI v1 commit at 2026-05-12-1826). No status checks; no branch protection. **Mild:** single-contributor reality; risk is theoretical. Defer until a second contributor lands OR until a CI failure would have been catchable. See P2.

- **Submodule-deinit-then-regenerate recovery still undocumented.** Per 2026-05-12-1826 retro P1. The 2026-05-12-1826 loop's build phase surfaced the recovery pattern; this loop didn't pick it up (org-os focus). **Mild:** recoverable, mechanical. Pickup at the next active-resink-core loop.

## Evolution proposals

### P1: Decide hot-swap step 3's scheduling (class: **tenant**)

- **Problem it solves:** "What didn't" #1. Third consecutive slip on the joint AE + resink-core deliverable. Each loop's single-focus discipline has displaced it. The 2026-05-12-1826 retro § P2 named three options; this retro forces the decision.
- **Options (decision required):**
  - **(a) Dedicate next loop to hot-swap step 3 as the single focus.** Closes the ADR cleanly. Joint AE + resink-core both active.
  - **(b) Split into AE-only (template-side v1/v2 fixture pair) + resink-core-only (consumer-side load-and-swap) sub-deliverables** that can land in separate loops.
  - **(c) Accept indefinite slip until a loop where AE + resink-core are both naturally active.** ADR body grandfathers under absolute-date authoring; slip is bounded.
- **Recommendation:** **(a) — dedicate next loop.** Reason: the bundled-ratification pattern worked for org-os authoring this loop; the analog (bundled-deliverable pattern) works for joint deliverables. The "dedicate a loop" approach has precedent (2026-05-12-1254's workspace-promotion-only loop). The cost is 1 loop of single-focus on a 3-loop-slipped deliverable; the benefit is closing the ADR cleanly.
- **Review path:** CEO decision at next brief authoring. The brief picks one of (a)/(b)/(c) and commits.
- **Owner:** board (decides); AE + resink-core (executes if (a) or (b)).
- **Timing:** **Decision: next loop's brief.** Execution: per the chosen option.

### P2: CI/CD bootstrap on the new resink-core remote (class: **tenant**)

- **Problem it solves:** "What didn't" #2. The new remote has no CI workflows; no status checks; no branch protection. 2026-05-12-1254 retro P2 captured this; carried for 2 loops.
- **Proposed change:** Bootstrap minimal CI on the resink-core remote:
  - `.github/workflows/ci.yml` with cargo build + cargo test + helm gates (lint --strict + template + dry-run with home-cluster values).
  - Branch protection on master: require PR + at least 1 status check pass.
  - Sized S; one PR on the resink-core remote.
- **Recommendation:** **Bundle with P1's hot-swap loop if (a) is picked.** The hot-swap loop would touch resink-core anyway; adding CI bootstrap is a small additional objective. If (a) isn't picked, defer to next active-resink-core loop.
- **Review path:** Tenant decision; no ADR (chart-internal + CI-internal).
- **Owner:** teams/application/resink-core (CI workflows); board (branch protection via gh api).
- **Timing:** **Next active-resink-core loop**; bundled with P1 if (a).

### P3: Document submodule-deinit-then-regenerate recovery (class: **tenant**)

- **Problem it solves:** 2026-05-12-1826 retro P1 carry. The gitignored runtime artifacts get wiped by `git submodule deinit -f`; recovery is `uv sync --extra dev` (in 2 locations) + `make mvp-loop`. Unobvious.
- **Proposed change:** Add a `make bootstrap` target to `synthetic_tenants/closed_loop_v0/Makefile` that runs the venv-create + workspace-regen in one shot. OR add a "Troubleshooting" subsection to `docs/user-guide.md`. **Recommend the Makefile target** — automation over documentation when the recovery is mechanical.
- **Recommendation:** **Bundle with P2** at next active-resink-core loop.
- **Review path:** Tenant decision.
- **Owner:** teams/application/resink-core.
- **Timing:** **Next active-resink-core loop**.

### P4: Codify the "5-section body shape" for playbooks (class: **org-os**)

- **Problem it solves:** "What worked" #4. The 4-playbook cluster shares a body shape: when-to-invoke; the question / procedure; worked examples; anti-patterns; sister patterns. Codifying this as the canonical playbook shape reduces friction for future playbook authoring.
- **Proposed change:** Add a "Playbook body shape" subsection to `org-os/conventions.md § "Body-shape rules"` naming the 5 canonical sections. Author future playbooks against the shape; grandfather existing ones (they all already follow it). XS-sized; one paragraph + a 5-bullet list.
- **Review path:** ADR-class change to conventions.md. Draft + ratify next loop OR same-loop (the change is mechanical). **Recommend: drafted as ADR-2026-05-13-001 this loop; ratified next loop alongside any P1/P2/P3 work.**
- **Owner:** board.
- **Timing:** **Draft this loop; ratify next loop.** (Or defer entirely if next loop is hot-swap-focused per P1 (a) — the codification can wait.)

Actually — **defer the ADR draft to next retro** rather than draft-this-loop. Reasoning: the codification is observation-not-decision (we're describing a pattern we already follow); deferring avoids unnecessary loop-state weight. If a future playbook authoring tries to deviate from the 5-section shape, the deviation itself motivates the ADR. Move P4 to **deferred candidate**, revisit when motivated.

## Decisions to record

New ADR placeholders this loop: **None.**

- P1 needs no ADR (tenant scheduling decision).
- P2 needs no ADR (CI-internal).
- P3 needs no ADR (Makefile target).
- P4 deferred (codification-without-motivation; revisit when motivated).

**Net new ADR drafts this retro: 0.** Consistent with the org-os process maturity signal — bundled ratifications cleared the backlog; no new authoring debt this loop.

## Carryover ADRs / open work still on the books

- **Hot-swap step 3 (ADR-2026-05-16-001):** loop+5. **Third consecutive slip.** P1 forces the decision.
- **CI/CD setup on new resink-core remote:** prior-prior retro P2. P2 this retro recommends bundling.
- **Submodule-deinit-then-regenerate recovery doc:** prior retro P1. P3 this retro recommends bundling.
- **Multi-loop-blocker-arc report-type:** 2026-05-12-1254 retro P3 — revisit after another arc closes.
- **`Chart.appVersion` ↔ supervisor crate version alignment:** bookkeeping.
- **Cargo MSRV declaration sync to Cargo.lock:** bookkeeping.

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: **passed**. Final tenant-isolation grep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns only the canonical `acme.ai` placeholder at `org-os/conventions.md:121`. **Largest org-os write concentration since 2026-05-30 (Bundle C: 5 ritual + role edits) completed with zero tenant-isolation violations on commit.** Two violations were caught and fixed at per-edit dry-run stages (worked-example tenant names in playbooks); the per-edit discipline pays off at this scale. Pass.
{% endraw %}
