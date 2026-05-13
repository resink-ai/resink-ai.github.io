---
layout: default
title: CEO Brief — 2026-05-13-1303
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-1303
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1303
  links: parent: ""
-->
{% raw %}

# Resink.ai CEO Brief — 2026-05-13 (loop 2026-05-13-1303)

## Context

Fourth loop on the same calendar date. The earlier three closed: org-os ratification bundle (`-0056`); observability skill + hot-swap AE-side (`-0859`); resink-core bundle close-out — ADR-2026-05-16-001 retires (`-1022`). Three product / platform arcs all landed in one calendar day. The state of the work on disk is now substantial enough to **show somebody**, and the only existing showcase artifact (`board/reports/2026-05-30-resink-core-capabilities.html`) is scoped to resink-core only and was written for a technical reviewer rather than a first-time external visitor.

**The gap.** Distance from vision is now **storytelling about what's been built**. A first-time visitor to `blog.resink.ai` lands on a loop-index, not on a "this is what we do" surface. The repo contains a multi-tenant data-pipeline supervisor (deployed on a real k8s cluster), an LLM-orchestrated codegen pipeline (closed-loop verdict GREEN), a CLI surface, an observability skill, a marketplace of 4 plugins, and a closed multi-loop dlopen restoration ADR — none of which is discoverable from the gitbook landing page without reading individual loop pages.

**The shape of this loop.** Single-team loop (board active, all other teams paused). **Three coordinated deliverables** that together close the showcase gap:

- **O1: External-audience capabilities HTML report.** Mirror of the 2026-05-30 report shape, but rewritten for potential users / investors. Cover the platform as a whole (not just resink-core): runtime, training pipeline, observability, CLI, marketplace, org-os process. Self-contained HTML; publishes through `raw_copy=True`.
- **O2: Live walkthrough doc.** A narrative external-audience tour at `docs/superpowers/specs/2026-05-13-resink-ai-walkthrough.md` — what we do, how the pieces fit, the evaluator's path through the artifacts (which file to read, which command to run, what verdict to expect). Less marketing copy; more "you can verify this for yourself."
- **O3: Gitbook navigation surfacing.** The publish-to-gitbook script's landing-page builder gains a prominent "Start here" section that links to the capabilities report + walkthrough as the **first two entries**, before drilling into loops / decisions / teams. First-time visitor lands on the showcase pair.

**Probe results entering the loop:**

- The existing `2026-05-30-resink-core-capabilities.html` is 372 lines of self-contained HTML with inline CSS. Shape: header + lede + TOC + 7 sections (product summary, what works today with evidence, architecture diagram, component catalog, named deviations, what's NOT in scope, how to read further). Audience header reads "technical reviewer" — this loop's report is for a different audience.
- `scripts/publish-to-gitbook.py`'s `build_landing_body` function (lines 581-607) emits a `## Vision` block (from `ORG.md`) + a `## Navigate` block (loops, decisions, actions, reports, teams). Adding a "## Start here" subsection above `## Navigate` is the minimum-shape edit.
- `raw_copy=True` HTML pipeline confirmed working from prior loops (cluster-snapshot report shipped through it cleanly).
- Tenant-isolation entering: clean (single canonical `acme.ai` placeholder at `org-os/conventions.md:121`).

**Re recent retro carryovers:**

- **2026-05-13-1022 retro P1 (multi-loop-blocker-arc report-type ADR):** the prior loop's recommended-next-loop work. **Defer to next non-resink-core loop** AFTER this showcase loop — the showcase pair is the higher-leverage deliverable when distance-from-vision is "external visibility" rather than "internal process codification."
- **2026-05-13-0859 retro P2 (link-existence smoke for `publish-to-gitbook.py`):** still deferred. This loop touches publish-to-gitbook (O3); could bundle, but the showcase deliverables are M-sized each and adding the smoke would push it to L. **Defer to next loop.**
- **2026-05-13-1022 retro P5 (cargo CI cross-repo access):** demand-driven; private-compatible options listed (deploy key / scoped PAT / in-repo fixtures). No action this loop.
- **2026-05-13-1022 retro P6 (branch protection on private repo):** indefinite defer while single-contributor.
- **2026-05-13-1022 retro P2 (supervisor-side NodeCtx bridge):** demand-driven.

**CEO decisions for this loop:**

- **Activate one team + board.** Board active for all three objectives; all other teams paused (no review-ack — neither deliverable touches their product surface beyond pulling existing artifacts as evidence).

- **3 objectives, all board-owned.** O1 HTML report (M). O2 walkthrough markdown (M). O3 publish script + nav (S). Total sized M-L; fits one loop because the deliverables share a content-research phase (one read-through of the codebase + ORG.md + key ADRs informs both artifacts).

- **External audience tone shift.** Both O1 and O2 are written for **potential users / investors** per CEO selection. Implications:
  - Outcomes-focused: lead with what runs, what verdict landed, what cluster deployed; not with internal process language.
  - Evidence-cited: every claim links to a checked-in artifact (verdict.json line, test count, deploy timestamp, ADR closure).
  - No internal-only jargon. "Loop" → "release"; "retro" → "post-release review"; "rit-*" → "ritual" with explanation. Use the existing technical terms where they're the actual product surface (e.g., "supervisor," "scd2-maintainer," "helm chart") — those are valid technical product names.
  - Honest scope: name deviations explicitly (the hot-swap test landed at the supervisor-load-cycle level; the supervisor-side NodeCtx bridge is named follow-up). Don't oversell.
  - No claims about market traction or customers (we have none).

- **Surface in gitbook landing.** "Start here: [capabilities report] · [walkthrough]" is the first navigation block on `blog.resink.ai`. The existing loop-index / decisions / actions / teams nav stays below as "Deep dive."

- **Tenant-isolation invariant maintained.** New artifacts under `board/reports/` and `docs/superpowers/specs/`; both are tenant-bearing, not under `org-os/`. The publish-script edit is also outside `org-os/`. Zero `org-os/` edits expected.

- **No new ADR drafts.** All three objectives are content + script touch; none mandates an `org-os/` change. Smallest-product-slice playbook framing: this loop's "slice" is the showcase pair; richer slices (interactive demo, pitch deck, video walkthrough) wait for real demand signal.

**Standing CEO answers:**

- **HTML report shape.** Mirror `board/reports/2026-05-30-resink-core-capabilities.html`'s structure (header + lede + TOC + sections) but rewrite the content for the external audience. Sections this loop: (1) what resink.ai does (vision + concrete); (2) what's running today (with verdicts); (3) the architecture (one diagram); (4) component catalog (per-team / per-repo); (5) closed engineering arcs (ADR-2026-05-16-001's dlopen restoration; ADR-2026-05-09-001's bottom-up flow); (6) deferred / honest deviations; (7) how to read further. Inline CSS; no external assets.

- **Walkthrough doc shape.** A reader following the doc top-to-bottom can: (i) understand what the platform does in <3 minutes; (ii) clone the repo and run `make mvp-loop` (operator-readable recipe); (iii) understand each major subsystem with one paragraph + a file pointer; (iv) understand the org-os process (briefs / ADRs / loops) without reading the full conventions. Sized M; ~600-900 lines.

- **Both artifacts live in gitbook nav as the primary entry points.** The landing page changes from "Resink.ai operations" framing (internal-operator) to "**What is Resink.ai? · How does it work? · Then: loops, decisions, teams**" framing (external-first, internal-deep-dive-second).

- **No public-repo proposal anywhere.** Per memory `feedback-no-public-repo.md`: the showcase artifacts go to the existing private-publishing-to-public-gitbook pipeline (which is how `blog.resink.ai` already works — gitbook submodule pushes a static site). Both source repos stay private. The publish pipeline (newbase → gitbook → blog.resink.ai) is the existing external surface; this loop populates it with showcase content, not changes its mechanics.

**Carryover load by team:**

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| board | O1 capabilities HTML (M) + O2 walkthrough (M) + O3 landing nav (S) + brief + exec + retro | M-L | Single-focus showcase loop; deliverables share content-research phase so the total fits one loop. |
| All other teams | Paused; no review-ack ask | — | Paused, silent — no team exec summary required. AE / resink-core / sre / devops content is pulled into the showcase artifacts as evidence (their existing files); they don't need to author anything this loop. |

## Objectives

### O1: External-audience capabilities HTML report

source: ceo-brief

Why it matters: First first-time-visitor-readable surface for what the platform is and what runs today. The 2026-05-30 report is technical-reviewer-shaped (assumes you know what an ABI is, what dlopen is, what a Cargo workspace is); this loop authors the same body of evidence in a tone an investor or potential user can read without prior context.

KR1.1: New file at `board/reports/2026-05-13-resink-ai-capabilities.html`. Self-contained: inline `<style>`; no external `<link>` / `<script>` / `<img>`. 7 sections per the standing-answer shape; TOC nav.

KR1.2: Content scope. Cover (i) vision in 1-paragraph form; (ii) what's running today with evidence (closed-loop verdict; home-cluster deploy; hot-swap test landed; observability snapshot taken; CLI v1 shipped; marketplace at 4 plugins); (iii) one architecture diagram (ASCII or SVG-inline); (iv) component catalog (per-product / per-team); (v) closed engineering arcs (dlopen restoration; bottom-up flow); (vi) named deferred work; (vii) "how to read further" pointing at the walkthrough + relevant ADRs.

KR1.3: External-audience tone. No internal-only jargon without inline explanation. Evidence links use file paths the reader can verify post-clone. No market-traction claims; no customer count claims.

KR1.4: Publishes through `raw_copy=True` to `https://blog.resink.ai/reports/2026-05-13-resink-ai-capabilities.html`. Verified by opening locally in a browser + by post-merge gitbook CI green.

### O2: Live walkthrough doc

source: ceo-brief

Why it matters: The HTML report tells you **what** runs; the walkthrough tells you **how to verify it yourself**. A first-time evaluator can clone the repo, run `make bootstrap`, run `make mvp-loop`, see `verdict=pass`, and feel confident the claims in the report hold. Without the walkthrough, the report is just text.

KR2.1: New file at `docs/superpowers/specs/2026-05-13-resink-ai-walkthrough.md`. Frontmatter: `type: spec`, `owner: board`, `date: 2026-05-13`, `status: active`. Body sections: (i) what is Resink.ai (3-minute read); (ii) run it yourself (operator recipe — clone, bootstrap, mvp-loop, verdict); (iii) the closed loop (what just happened); (iv) the components (per-product paragraph + file pointer); (v) the org-os process (briefs / ADRs / loops in 200 words); (vi) the home cluster (k8s deploy state); (vii) deferred / honest deviations; (viii) further reading (ADRs + key OKRs).

KR2.2: Operator recipe in §ii is reproducible. The exact commands work against the current main branch; the expected output is literally `verdict=pass mismatches=0`. If a step requires `ANTHROPIC_API_KEY` it's labeled as optional with the in-process fallback documented.

KR2.3: File-pointer-rich. Every component paragraph names the exact directory the reader navigates to. No "see the codebase"; instead "see `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs`."

KR2.4: Publishes through the `docs/superpowers/specs/` path that the publish-to-gitbook script already handles (lines 408-413 + 522-531). Verified by post-merge gitbook CI green.

### O3: Gitbook landing nav surfaces the showcase pair

source: ceo-brief

Why it matters: An artifact that no one finds is the same as one that doesn't exist. The publish-to-gitbook script's landing-page builder currently buries the existing capabilities report inside the "Reports" nav block. This loop hoists the showcase pair to the top of the landing page so a first-time visitor lands on them, not on a loop-index.

KR3.1: `scripts/publish-to-gitbook.py`'s `build_landing_body` function gains a "## Start here" section as the **first content block after the vision / lede**. The section emits two prominent links: capabilities report + walkthrough.

KR3.2: The existing nav (loops, decisions, actions, reports, teams) is preserved but relabelled as "## Deep dive" so the structure reads as "showcase → deep dive."

KR3.3: The script change is generic. Showcase paths derived from a small dataclass (or constants block) so future showcase artifacts can be added without re-shaping the landing emitter.

KR3.4: Live gitbook landing at `https://blog.resink.ai/` shows the new layout post-merge.

## What success looks like

- `board/reports/2026-05-13-resink-ai-capabilities.html` exists, opens in a browser, renders self-contained, 7 sections, external-audience tone.
- `docs/superpowers/specs/2026-05-13-resink-ai-walkthrough.md` exists; operator recipe is reproducible from a clean clone.
- `https://blog.resink.ai/` lands a first-time visitor on "Start here: capabilities report + walkthrough" before the loops nav.
- Post-merge gitbook CI green per standing practice.
- Tenant-isolation invariant: clean. Zero `org-os/` edits.
- Retro surfaces: (i) external-audience tonal shift — what was hard about it (translating internal language without losing precision); (ii) the showcase-pair pattern — does landing-page nav need a sustained shape for future showcase artifacts? (iii) the walkthrough's "run it yourself" recipe — what shape verified-against-environment proof should take for an outsider.

## Out of scope

- Pitch deck / video / interactive demo. Future-loop slices if real demand surfaces.
- Customer-facing landing on a different domain than blog.resink.ai. Out of scope.
- Marketing copy beyond what's needed to be readable. We're not selling; we're showcasing built work.
- Public-repo conversion of source repos (per memory `feedback-no-public-repo.md`).
- New ADR drafts. Org-os process surface is stable; no codification pressure this loop.
- Multi-loop-blocker-arc report-type ADR (prior retro P1) — defers to next non-resink-core loop.
- Link-existence smoke for publish-to-gitbook (prior retro P2) — defers.
{% endraw %}
