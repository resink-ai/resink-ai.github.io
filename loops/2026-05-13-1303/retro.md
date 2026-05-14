---
layout: default
title: Retro — 2026-05-13-1303
date: 2026-05-13
status: active
type: retro
loop: 2026-05-13-1303
owner: board
grand_parent: Loops
parent: Loop 2026-05-13-1303
nav_order: 3
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1303
  links: parent: board/exec-summaries/2026-05-13-1303.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-13-1303

## What worked

- **External-audience tonal translation succeeded without losing precision.** The capabilities HTML report and the walkthrough markdown both rewrite the same body of engineering work that the 2026-05-30 technical-reviewer report covered, but for a different audience. Concrete moves that worked: (i) lead with vision in plain language ("an AI-managed data-pipeline platform" — not "a Rust multi-tenant supervisor with LLM-driven codegen"), (ii) make every claim evidence-backed with a file pointer the reader can verify, (iii) surface deferrals visibly in a dedicated section (8 specific deferrals named) rather than burying them. **Reproducibility:** for any external-audience artifact, the trick is the same — translate jargon inline at first use, cite evidence as file paths, name the boundary explicitly. The "honest deferrals" section is positioning, not weakness; readers respect "we shipped X and named Y as deferred follow-up" more than "we have everything."

- **Showcase-pair pattern works.** HTML + markdown specialize: the HTML is the executive-summary view (10-minute read; opens in a browser; raw_copy=True through publish); the markdown is the reader-followable tour (30-minute review; includes the clone-and-run recipe; lives in the design-specs path). The pair is more useful than either alone — the HTML tells you what runs, the markdown tells you how to verify it. **Reproducibility:** for future showcase artifacts (per-quarter snapshot, post-mortem, multi-loop arc report), default to the pair: a polished executive view + a longer reader-followable companion. They specialize cleanly.

- **`SHOWCASE_ENTRIES` constants block generalizes.** The publish-script change introduces a small dataclass-style list at module scope: `(label, src_path, href, description)` tuples. Existence-checked at build time; missing entries silently skipped. Future showcase artifacts add a 4-line entry; no emitter re-shape required. **Reproducibility:** when extending a generated-page builder for a new artifact class, prefer a constants block over inline conditionals — keeps the emitter clean and the artifact-set self-documenting.

- **Landing-page reframe was 4 lines of code + the biggest UX change of the day.** Renaming "Navigate" → "Deep dive" and introducing "Start here" as the first content block sounds cosmetic but is the load-bearing UX edit. A first-time visitor's path changes from "land on a loop-index" to "land on what we do" with a one-click drilldown to per-loop history if they want it. **Reproducibility:** the load-bearing UX change often costs the least code; resist the urge to over-engineer the first iteration. Add SHOWCASE_ENTRIES + a 6-line section emit; done.

- **No `org-os/` edits, no new ADRs, tenant-isolation trivially held.** Fourth consecutive loop with zero `org-os/` writes (counting back from this loop: -1303, -1022, -0859, -0056 all zero). The org-os surface is stable; product / showcase / external-visibility work runs cleanly under the existing process without authoring new process. **Reproducibility:** the org-os process is now in its "use it, don't extend it" phase — extensions need real motivation (a new pattern that won't fit existing shapes), not aspirational codification.

## What didn't

- **The walkthrough's "Run it yourself" recipe is theoretical until a fresh-clone reader verifies it.** I authored the recipe from working-tree knowledge; the steps are the ones I'd type, but I didn't actually `rm -rf newbase && git clone --recurse-submodules` and run them on a clean filesystem. **Mild:** the recipe is informed by the running tree and the commands are the actual `Makefile` targets; a real failure would be an environmental gap (`uv` not on PATH; cargo MSRV mismatch; submodule URL wrong for the cloning machine). **Action:** none this loop; flag for a future loop's "fresh-clone-verify" pattern. See P2.

- **The capabilities report's "How to read further" links assume some path conventions.** Specifically: the "../decisions/2026-05-16-001-..." style relative-from-reports paths. Jekyll publishes everything under one root, so `../decisions/` from `/reports/foo.html` resolves correctly to `/decisions/`. But that's not obvious from reading the HTML source; if a future loop changes the gitbook directory layout, the links break silently. **Trivial:** structural assumption; current layout has been stable for 12+ loops. No action; awareness.

- **No fresh-clone test of the bootstrap → mvp-loop recipe.** Same root as #1 above. The walkthrough's recipe is documented but the operator's fresh-clone path is not exercised in CI; relying on the existing local development environment. **Mild:** a future contributor's onboarding session is the natural pressure test.

- **Cross-cutting timing artifact: this is the fourth loop on the same calendar date.** Loop IDs (-0056, -0859, -1022, -1303) preserve uniqueness via the `HHMM` component, but four loops in 12 hours of wall-clock time is a different cadence than the convention's "two-week-equivalent" framing implies. The org-os process scales down to this density cleanly (no process surface changes needed); but the convention's language under-specifies the actual cadence. **Trivial:** loop-ID convention works; the conceptual framing in conventions.md is loose enough that this isn't a violation. Surface as awareness only.

## Evolution proposals

### P1: Showcase-pair refresh cadence (class: **deferred**)

- **Problem it solves:** The 2026-05-13 showcase pair captures state at a point in time. As more loops ship, the capabilities-as-of-2026-05-13 framing decays. Eventually the "what we've built" report needs to be re-published with current state.
- **Proposed change:** Establish a refresh trigger — e.g., "republish the capabilities pair every Nth loop, OR whenever a major engineering arc closes." No specific schedule.
- **Recommendation:** **Deferred candidate.** Wait until at least one of the artifacts feels visibly stale (e.g., a major component lands that isn't named in the capabilities report) before scheduling a refresh. The decay rate is unknown until it surfaces.
- **Review path:** None unless invoked.
- **Owner:** board.
- **Timing:** **Demand-driven.**

### P2: Fresh-clone verification of the walkthrough's recipe (class: **tenant**)

- **Problem it solves:** "What didn't" #1. The walkthrough's "Run it yourself" recipe was authored from working-tree knowledge; a real fresh-clone hasn't validated the exact step sequence.
- **Proposed change:** As a one-off, do a clean clone + execute the recipe on a fresh filesystem; capture any gaps. Update the walkthrough's recipe section with whatever the fresh-clone surface actually exposes.
- **Recommendation:** **Sized XS** (one clean clone + one mvp-loop run; ~30 minutes). Schedule when convenient; ideally before showing the showcase pair to an external evaluator.
- **Review path:** No ADR (operational verification).
- **Owner:** board.
- **Timing:** **Opportunistic; ideally before first external share.**

### P3: Multi-loop-blocker-arc report-type ADR (class: **org-os**)

- **Problem it solves:** Prior retro § P1 (2026-05-13-1022). With ADR-2026-05-16-001 now closed and narrated in this loop's capabilities report's "closed engineering arcs" section, the worked example for the arc-report-type pattern exists in two forms (the ADR's closing-section + the capabilities report's narrative). The codification is observation-not-decision.
- **Proposed change:** New ADR drafted as `board/decisions/<date>-NNN-multi-loop-blocker-arc-report-type.md`. Body proposes a new artifact type (`arc-report`) emitted when a multi-loop ADR closes. Contents: timeline (loop dates per step), slip record, what-shipped per step, what-deferred + carryover, what-the-arc-validated-or-broke.
- **Recommendation:** **Draft + ratify next loop.** The worked example is now in two artifacts; codification is straightforward. Pair with P4 (link-existence smoke) for a non-resink-core loop.
- **Review path:** ADR-class.
- **Owner:** board.
- **Timing:** **Next loop OR opportunistic.**

### P4: Link-existence smoke for `publish-to-gitbook.py` (class: **tenant**, carryover)

- **Problem it solves:** Prior retro § P2 (2026-05-13-0859). The publish script generates index pages that link to other pages; a link-existence smoke would catch the `.md`-vs-`.html` class of bug at publish time instead of post-merge.
- **Proposed change:** Sized S. Walk the published tree; extract `[text](path)` links from generated index pages; resolve each path; error if any link points at a non-existent destination.
- **Recommendation:** **Bundle with P3 next non-resink-core loop.**
- **Review path:** No ADR (CI-internal).
- **Owner:** board.
- **Timing:** **Next non-resink-core loop.**

## Decisions to record

**New ADR placeholders this loop: None.**

- P1 (showcase refresh cadence) is demand-driven; no ADR.
- P2 (fresh-clone verification) is operational; no ADR.
- P3 (arc-report-type) is the candidate ADR for next loop.
- P4 (link-existence smoke) is CI-internal; no ADR.

**Net new ADR drafts this retro: 0.** Fifth consecutive zero-ADR retro since the 2026-05-13-0056 ratification bundle. Org-os process surface continues to be stable; product / showcase / external-visibility work fits the existing shape without authoring new process.

## Carryover ADRs / open work still on the books

- **Multi-loop-blocker-arc report-type ADR (P3 this retro; first proposed 2026-05-12-1254 retro P3):** **Now actionable** — worked example exists in tree (ADR-2026-05-16-001's closing-section + this loop's capabilities report § "Closed engineering arcs"). Next non-resink-core loop.
- **Link-existence smoke for `publish-to-gitbook.py` (P4 this retro; carryover from 2026-05-13-0859 retro P2):** Next non-resink-core loop. Bundles with P3.
- **Showcase refresh cadence (P1 this retro):** Demand-driven; revisit when the showcase pair feels stale.
- **Fresh-clone verification of the walkthrough recipe (P2 this retro):** Opportunistic; ideally before first external share of the showcase pair.
- **Supervisor-side NodeCtx bridge** (step 3.5 follow-up to ADR-2026-05-16-001; demand-driven).
- **Cargo CI cross-repo access** (private-compatible options documented; user-only decision when to act).
- **Branch protection on resink-core master** (gated on GitHub Pro upgrade decision; user-only).
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing carry from 2026-05-23.
- **`<verified-against-rust-impl: pending>` tag in DE Kafka contract §2.1** — owner = DE.

## Tenant-isolation dry-run

Held trivially this loop. Zero `org-os/` edits. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `board/{okrs,reports,exec-summaries,retros}/`, `docs/superpowers/specs/`, and `scripts/`. None of these are under `org-os/`; the invariant has no edits to gate.
{% endraw %}
