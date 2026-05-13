---
layout: default
title: Exec Summary — 2026-05-13-1303
date: 2026-05-13
status: active
type: exec-summary
loop: 2026-05-13-1303
owner: board
---

<!-- original-frontmatter:
  type: exec-summary
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1303
  links: parent: board/okrs/2026-05-13-1303-ceo-brief.md
-->
{% raw %}

# Resink.ai Loop 2026-05-13-1303 — Company Exec Summary

**Headline.** Fourth loop on the same calendar date. After three product loops (org-os ratification → observability skill + hot-swap AE-side → resink-core bundle close-out), this loop closes the showcase gap: the work on disk is now substantial enough to show somebody, and the gitbook landing page needs to surface what the platform does before drilling into loops. **Three coordinated deliverables shipped:** (O1) an external-audience capabilities HTML report at `board/reports/2026-05-13-resink-ai-capabilities.html` (mirror of the 2026-05-30 shape, rewritten for potential users / investors); (O2) a live walkthrough markdown at `docs/superpowers/specs/2026-05-13-resink-ai-walkthrough.md` (a reader-followable 30-minute tour with a clone-and-run recipe); (O3) `scripts/publish-to-gitbook.py`'s landing-page builder now emits a prominent "Start here" block with the showcase pair as the first nav entry, before "Deep dive" (loops, decisions, actions, reports, teams). Live `blog.resink.ai` landing reframes from "Resink.ai operations" (internal-operator) to "Resink.ai" with the showcase-first nav. Tenant-isolation invariant CLEAN. Single team active (board); all other teams paused.

## Per-objective rollup

### O1: External-audience capabilities HTML report — ✅ PASS

- New file at `board/reports/2026-05-13-resink-ai-capabilities.html` — 7 sections (what we do; what's running today with evidence; architecture in one diagram; component catalog; closed engineering arcs; honest deferrals; how to read further). Self-contained: inline CSS, no external `<link>` / `<script>` / `<img>`.
- External-audience tone: no internal-only jargon without inline explanation; every claim links to a checked-in artifact; no market-traction or customer-count claims. The "honest deferrals" section names 8 specific deferrals (Kafka ingestion, persistent KV, multi-tenant supervisor sharing, cross-swap state observability, sim-farm Modes B/C, customer UI, cargo CI cross-repo step, branch protection on private repo) — surfaced visibly so the boundary is clear to an external reader.
- Closed engineering arcs section narrates both ADR-2026-05-16-001 (dlopen restoration, 5 loops / 3 steps) and ADR-2026-05-09-001 (org-os bottom-up flow, 17 tasks / 3 bundles) as completed multi-loop work — concrete evidence of the named-deviation + time-boxed-restoration discipline working in practice.
- Companion to the technical-reviewer-shaped 2026-05-30 resink-core capabilities report; the new report cross-links back as predecessor.

### O2: Live walkthrough doc — ✅ PASS

- New file at `docs/superpowers/specs/2026-05-13-resink-ai-walkthrough.md` — 8 sections (what is Resink.ai; run it yourself; the closed loop narrated; the components in one sentence each; the org-os process in 200 words; the home cluster; honest deferrals; further reading).
- Operator recipe in §2 ("Run it yourself") is reproducible from a clean clone: `git clone --recurse-submodules`; `make bootstrap`; `make mvp-loop`; expect `verdict=pass mismatches=0`. Names each step in the 7-step pipeline with a file pointer.
- File-pointer-rich. Every component paragraph names the exact directory; every "further reading" link resolves to a real artifact.
- The §2 recipe includes the `CLAUDE_DISPATCH=1` optionality (real LLM vs in-process slot-fill) so a reader without an `ANTHROPIC_API_KEY` can still run the loop.

### O3: Gitbook landing nav surfaces the showcase pair — ✅ PASS

- `scripts/publish-to-gitbook.py`'s `build_landing_body` function gained a "## Start here" section emitted **before** the existing "## Navigate" nav (now relabelled "## Deep dive"). The showcase pair is the first nav entry a visitor sees.
- The showcase entries are driven by a small `SHOWCASE_ENTRIES` constants block — a list of `(label, src_path, href, desc)` tuples. Existence-checked at build time; missing entries silently skipped. Future showcase artifacts add by appending to the list, not re-shaping the emitter.
- Landing-page title reframed from "Resink.ai operations" (internal-operator) to "Resink.ai" (external-first); H1 + tagline rewritten to match.
- Verified locally: `python3 scripts/publish-to-gitbook.py` produces the expected `index.md` shape; the showcase block renders above the deep-dive nav.

## Per-team rollup

### board (active, primary)

Three coordinated deliverables shipped: one HTML report (M), one markdown walkthrough (M), one publish-script + nav change (S). All landed cleanly without scope creep. Tenant-isolation invariant CLEAN (single canonical `acme.ai` placeholder at `org-os/conventions.md:121`; zero `org-os/` edits).

### All other teams (paused, silent)

No team's surface was touched. The showcase artifacts **reference** other teams' work (AE's codegen skill, resink-core's supervisor, the home cluster, the observability plugin) but don't require new authoring from those teams. Their existing artifacts are pulled in as evidence.

## Cross-cutting wins

- **First first-time-visitor-readable surface for the platform.** Before this loop, an external visitor landed on the loop-index — useful for internal navigation but opaque to someone evaluating the platform. The new "Start here" pair surfaces what we do, with verifiable evidence, in two reader-followable artifacts. The deep-dive nav stays one click away for the technical reviewer who wants the per-loop history.
- **Audience-tonal shift worked.** The 2026-05-30 capabilities report was technical-reviewer-shaped; this loop's report is external-audience-shaped. Both bodies of evidence are the same underlying engineering work — the tonal translation succeeded without losing precision. The same dlopen ADR closure that the 2026-05-30 report could only foreshadow is now narrated as a completed multi-loop arc.
- **Walkthrough's "run it yourself" recipe is the load-bearing evidence-citation pattern.** The HTML report tells you what runs; the walkthrough tells you how to verify it. The recipe (`git clone --recurse-submodules` → `make bootstrap` → `make mvp-loop` → expect `verdict=pass mismatches=0`) is the operator-readable promise. Without it, the report is just text; with it, an evaluator can reproduce the green-bar in <30 minutes on a developer laptop.
- **Honest-deferrals section is positioning, not weakness.** Naming 8 specific deferrals visibly (Kafka, KV, multi-tenant, cross-swap observability, sim-farm modes, UI, cargo CI, branch protection) demonstrates engineering discipline rather than scope. Each deferral corresponds to a named follow-up in our internal artifacts; the boundary between "shipped" and "named-but-not-yet" is clear.
- **Publish-script generalizes for future showcase artifacts.** The `SHOWCASE_ENTRIES` constants block makes adding a new showcase artifact a 4-line edit (label, source path, href, description) — no emitter re-shape required. Future loops that produce a new external-audience artifact (e.g., a video walkthrough, a per-quarter capability snapshot) can add an entry without touching the rest of the script.
- **Tenant-isolation invariant trivially held.** No `org-os/` edits this loop. The showcase artifacts live under `board/reports/` and `docs/superpowers/specs/`; the publish-script change is outside `org-os/`. The grep returns only the canonical `acme.ai` placeholder.

## Cross-cutting blockers

None. Prior retro carryovers all addressed appropriately (deferred or marked demand-driven).

## Asks for the CEO

- **(from board, future-loop):** Multi-loop-blocker-arc report-type ADR (prior retro § P1). With ADR-2026-05-16-001 now closed and narrated in the capabilities report's "closed engineering arcs" section, the worked example for the arc-report-type pattern exists in tree. Surface as a candidate for the next non-resink-core loop.
- **(from board, future-loop):** Link-existence smoke for `publish-to-gitbook.py` (prior retro § P2). Still pending; this loop touched publish-to-gitbook but didn't bundle the smoke (would have pushed the loop from M-L to L). Carries forward.
- **(from resink-core, demand-driven):** Cargo CI cross-repo access (deploy key / scoped PAT / in-repo fixtures); supervisor-side NodeCtx bridge (step 3.5 follow-up). All demand-driven; no scheduling pressure.

## Decisions ratified this loop

**Zero new ADRs.** No `org-os/` edits. The brief's "no new org-os process work" decision held cleanly.

## Decisions filed this loop (not new ADRs)

- **External-audience tone for showcase artifacts.** Plain-language framing without losing technical precision; every claim evidence-backed; no market-traction or customer claims; honest deferrals surfaced explicitly. Pattern is now established; future showcase artifacts inherit the same tone discipline.
- **Showcase pair as load-bearing nav entries.** Landing page reframes from operations-shaped to "Start here → Deep dive." The pair lives at the top; the existing nav (loops / decisions / actions / reports / teams) drops one level.
- **`SHOWCASE_ENTRIES` constants block.** Future showcase additions append; no emitter re-shape.

## Multi-loop plan slippage absorbed

None this loop. The prior loop closed ADR-2026-05-16-001's arc; the work since has been single-loop deliverables. No multi-loop plans in flight.

## Request-flow metrics this loop (per ADR-2026-05-09-006)

| Metric | Count | Note |
|---|---|---|
| `cross-team-request` objectives `fulfilled` | 0 | No cross-team requests filed in or out this loop. |
| `dropped` | 0 | None. |
| `declined` | 0 | None. |
| `escalated` | 0 | None. |
| In-loop requests filed | 0 | None. |

## Tenant-isolation invariant

Held trivially. Zero `org-os/` writes this loop. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `board/{okrs,reports,exec-summaries,retros}/`, `docs/superpowers/specs/`, and `scripts/`. None of these are under `org-os/`.

## Notes for the retro

Three patterns worth recording:

1. **External-audience artifacts need a tonal-shift discipline, not just content reuse.** The capabilities report's body is the same underlying engineering work as the 2026-05-30 technical-reviewer report, but the framing — what's the first sentence; what's the lede; what claims need inline explanation; what counts as evidence vs jargon — required a deliberate tonal translation. The pair-it-with-the-companion-walkthrough pattern (HTML report + markdown walkthrough) lets each artifact specialize: the HTML is the "what runs" view (executive-summary-shaped); the markdown is the "how to verify" view (reader-followable).
2. **Showcase-pair pattern + landing-page hoist.** Three deliverables (HTML + markdown + publish-script change) form a coherent unit. Without the script change, the pair lives in tree but isn't discoverable; without the artifacts, the script change has nothing to surface. Future showcase artifacts can add to `SHOWCASE_ENTRIES` and inherit the existing landing-page hoist.
3. **The publish-to-gitbook nav reshape is the load-bearing UX change.** Renaming "Navigate" → "Deep dive" and introducing "Start here" as the first content block sounds cosmetic but is the most consequential edit: it changes what a first-time visitor sees from "loop-index" to "what we do." The technical change was 4 lines (the SHOWCASE_ENTRIES list + the new section emission); the UX impact is the lede of the landing page itself.
{% endraw %}
