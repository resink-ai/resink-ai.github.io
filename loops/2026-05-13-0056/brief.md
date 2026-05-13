---
layout: default
title: CEO Brief — 2026-05-13-0056
date: 2026-05-13
status: active
type: okr
loop: 2026-05-13-0056
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-0056
  links: parent: ""
-->
# Resink.ai CEO Brief — 2026-05-13 (loop 2026-05-13-0056)

## Context

Four consecutive loops with **zero `org-os/` writes** by any team — a healthy stretch demonstrating that org-os process evolution should be load-bearing change via the evolution path (ADR + ratification), not housekeeping. The cost of that stretch is a growing ratification backlog: as of last loop's retro, **5 org-os items are deferred** across 2 prior retros' P-list proposals. The last retro's P3 was "strong-recommend: actually do this next loop. Backlog grows each loop deferred."

**The gap.** Distance from vision is now the org-os process surface itself. Two draft ADRs (`2026-05-12-001-reframe-vs-act-playbook.md` and `2026-05-12-002-submodule-promotion-playbook.md`) sit at `status: draft` with bodies authored but their playbook artifacts not yet written. Two pattern extensions (V-a-E scope to `runbook` artifacts; first-real-deploy pattern as a sibling section in `provisional-and-migrate.md`) sit as retro proposals waiting for in-place edits. One new playbook (`smallest-product-slice.md` per last retro's P4) waits to be drafted. **This loop ratifies the backlog as a single bundled batch.**

**The shape of this loop.** A **ratification-bundle loop** — pure org-os authoring; single team active (board); minimal cross-team work; tenant-isolation maintained throughout via per-edit dry-runs + cumulative final sweep. The pattern matches the 2026-05-11-2153 loop's 4-ADR ratification batch (and the implicit sister precedent from 2026-05-30's 3-ADR batch).

**Probe results entering the loop:**

- 2 draft ADRs in `board/decisions/`: `2026-05-12-001-reframe-vs-act-playbook.md` + `2026-05-12-002-submodule-promotion-playbook.md`. Both have bodies authored at their drafting loop; both need their **playbook artifacts** authored + their `status:` flipped to `active`.
- 1 in-place ADR body extension: `2026-05-16-003-contract-environment-verification.md` — its scope section currently names `contract` and `rfc` types; extending to admit `runbook` (and opportunistically `convention` + `playbook`).
- 1 in-place playbook edit: `org-os/playbooks/provisional-and-migrate.md` — add a "Sibling pattern: first-real-deploy as integration test" section (or similar shape) per 2026-05-12-0645 retro P5.
- 1 new playbook: `org-os/playbooks/smallest-product-slice.md` per 2026-05-12-1826 retro P4 (XS).
- 1 in-place conventions edit: `org-os/conventions.md § "contract and rfc — Verified-against-environment subsection (required)"` — update the section header + body to admit the wider scope, alongside the ADR-003 extension.
- Open accepted requests with `deferred_to_loop: 2026-05-13-0056`: **none.**
- Tenant-isolation entering: clean (only `acme.ai` placeholder in `org-os/`).

**Re ADR-2026-05-16-001 step 3 (hot-swap correctness test, was loop+4 = this loop per prior retro):** **Slipped to loop+5** (third consecutive slip). Slip reason: org-os ratification focus this loop. **THIRD consecutive slip** — per the prior retro's P2, if it slips a third time the scheduling assumption needs revision. **Surfaced in this loop's retro** as a candidate for option (b) — split into AE-only + resink-core-only sub-deliverables — or option (c) — accept indefinite slip.

**Re retro proposals carrying forward:**

- **2026-05-12-0645 retro P3 (reframe-vs-act playbook):** picked up this loop as O1.
- **2026-05-12-0645 retro P4 (V-a-E scope extension to `runbook`):** picked up as O3.
- **2026-05-12-0645 retro P5 (first-real-deploy playbook section):** picked up as O4.
- **2026-05-12-1254 retro P1 (submodule-promotion playbook):** picked up as O2.
- **2026-05-12-1254 retro P3 (multi-loop-blocker-arc report-type):** still deferred (timing trigger: revisit after another arc closes; not this loop).
- **2026-05-12-1254 retro P4 (footer-acknowledgment pattern):** bundled as a sibling section in O2's submodule-promotion playbook.
- **2026-05-12-1826 retro P1 (document submodule-deinit-then-regenerate recovery):** deferred (tenant-class; pickup at next active-resink-core loop).
- **2026-05-12-1826 retro P2 (hot-swap step 3 scheduling decision):** surfaced this loop's retro for CEO decision; not actioned this loop.
- **2026-05-12-1826 retro P3 (org-os ratification bundle):** THIS LOOP.
- **2026-05-12-1826 retro P4 (smallest-product-slice playbook):** picked up as O5.

**CEO decisions for this loop:**

- **Activate one team + board (light).** Board only. All other teams paused (no review-acks because the changes are pure `org-os/` process surface — they don't touch any team's product surface).

- **5 objectives, all `org-os/`-bearing.** O1 reframe-vs-act playbook + ratification. O2 submodule-promotion playbook + ratification (with footer-ack sibling section). O3 ADR-2026-05-16-003 scope extension (in-place body + conventions.md edit). O4 first-real-deploy sibling section in provisional-and-migrate.md (in-place). O5 smallest-product-slice playbook (new, XS).

- **Per-edit tenant-isolation dry-runs.** 5 separate `org-os/` edits this loop. Discipline: dry-run after each edit (cumulative grep) + one final sweep. The 4-ADR ratification batch at 2026-05-11-2153 set the precedent; this loop applies the same shape with 4 new playbooks/extensions + 2 ADR ratifications.

- **No new ADR drafts this loop.** All retro proposals being actioned this loop already have ADR placeholders (drafted at prior retros) OR are in-place extensions to existing ADRs. P3 from 2026-05-12-1254 (multi-loop-blocker-arc report-type) explicitly deferred per its own timing recommendation; no draft this loop.

- **Defer hot-swap step 3 (THIRD consecutive slip).** Surfaced in retro for CEO decision.

**Standing CEO answers:**

- **The bundled-ratification loop pattern.** Single-focus loops produce clean deliverables; the trade-off is org-os process evolution slows by a loop. The accumulation strategy — let backlog build for 2-3 loops, then dedicate a loop to bundling — is the right answer when the deferred items are all the same shape (org-os authoring). This loop demonstrates the pattern at scale.

- **Playbook authoring shape.** All 4 new/extended playbooks (reframe-vs-act, submodule-promotion, smallest-product-slice, provisional-and-migrate's first-real-deploy section) follow the same body shape: (a) when to invoke; (b) the procedure / question to ask; (c) worked example(s); (d) anti-patterns; (e) sister-pattern relationships. Established by `provisional-and-migrate.md`; carried forward.

- **Scope-extension shape for V-a-E discipline.** ADR-2026-05-16-003's body gains a "Sister artifact types" subsection naming the four canonical scope types (`contract`, `rfc`, `convention`, `runbook`, `playbook`). The grandfathering rule stays (required on new artifacts after the original ratification; opportunistic for old). `org-os/conventions.md`'s body-shape section gets the matching scope update.

**Carryover load by team:**

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| board | 5 org-os edits batched in one loop (2 playbooks new + 2 in-place + 1 small + 2 ADR ratifications + 1 conventions edit); per-edit + final tenant-isolation dry-runs; brief + retro | L | The single-focus ratification batch; largest org-os authoring loop since 2026-05-11-2153 (which had 4 ADR ratifications). |
| All other teams | Paused; no review-ack ask (changes are pure org-os process; don't touch any team's product surface) | — | Paused, silent — no team exec summary required (paused-team-only loop) |

## Objectives

### O1: Author `org-os/playbooks/reframe-vs-act.md` + ratify ADR-2026-05-12-001

source: ceo-brief

Why it matters: The reframe-vs-act pattern (when a multi-loop blocker has the wrong shape, change the framing rather than try to clear the specific blocker) has now closed two recurring blocker classes (toolchain installs → Prerequisite docs at 2026-05-11-2153; minikube smoke → home-cluster deploy at 2026-05-12-0645). The pattern is broad enough to be reusable; the ADR draft from 2026-05-12-0645 retro P3 captured the decision. This loop authors the playbook body + flips the ADR.

**Key results**

- KR1.1: New file at `org-os/playbooks/reframe-vs-act.md` with `type: playbook`, `owner: board`, `date: 2026-05-13`, `status: active`, `invocation_trigger: "when a blocker has carried 3+ loops with the same 'specific action' framing"`.
- KR1.2: Body sections: (a) when to invoke; (b) the question to ask; (c) two worked examples (toolchain installs at 2026-05-11-2153; minikube smoke at 2026-05-12-0645); (d) anti-patterns; (e) sister pattern relationship to `provisional-and-migrate.md`.
- KR1.3: `board/decisions/2026-05-12-001-reframe-vs-act-playbook.md` flips `status: draft → status: active`. Body's "Mandated edits" subsection reads as accomplished (new playbook is in place per KR1.1).
- KR1.4: Cross-link from `org-os/conventions.md § Mutation rules` or appropriate section pointing at the new playbook. Single-line addition.
- KR1.5: Per-edit tenant-isolation dry-run after each `org-os/` edit. Final sweep at the end.

**Tasks**

- [ ] Author `org-os/playbooks/reframe-vs-act.md`.
- [ ] Flip ADR-2026-05-12-001 `status: draft → active`.
- [ ] Cross-link from `org-os/conventions.md`.
- [ ] Per-edit + final tenant-isolation dry-runs.

### O2: Author `org-os/playbooks/submodule-promotion.md` + ratify ADR-2026-05-12-002

source: ceo-brief

Why it matters: The `git subtree split` pattern from the 2026-05-12-1254 workspace promotion is reusable for any future in-tree → submodule migration. The ADR draft from that loop's retro P1 captured the decision. This loop authors the playbook body + flips the ADR. The playbook bundles in 2026-05-12-1254 retro P4 (footer-acknowledgment pattern) as a sibling section, per the P4 proposal's own bundling recommendation.

**Key results**

- KR2.1: New file at `org-os/playbooks/submodule-promotion.md` with `type: playbook`, `owner: board`, `date: 2026-05-13`, `status: active`, `invocation_trigger: "when an in-tree directory under repos/<org>/<name>/ needs to become a real git submodule"`.
- KR2.2: Body sections: (a) when to invoke; (b) the procedure (10-step `git subtree split` recipe); (c) **footer-acknowledgment sibling section** per 2026-05-12-1254 retro P4 (single-line footer pattern for structural-but-non-disruptive changes); (d) worked example (the 2026-05-12-1254 resink-core promotion); (e) anti-patterns (fresh-init + single-import; bulk-rewriting historical references; skipping working-tree diff verification); (f) sister pattern relationships.
- KR2.3: `board/decisions/2026-05-12-002-submodule-promotion-playbook.md` flips `status: draft → status: active`.
- KR2.4: Cross-link from `org-os/playbooks/provisional-and-migrate.md` § "Sister patterns" or via a Related-playbooks footer pointing at the new playbook. Single-line addition.
- KR2.5: Per-edit tenant-isolation dry-run.

**Tasks**

- [ ] Author `org-os/playbooks/submodule-promotion.md` (incl. the footer-acknowledgment sibling section).
- [ ] Flip ADR-2026-05-12-002 `status: draft → active`.
- [ ] Cross-link from `provisional-and-migrate.md`.
- [ ] Per-edit + final tenant-isolation dry-runs.

### O3: Extend ADR-2026-05-16-003 scope to admit `runbook`, `convention`, `playbook` artifacts

source: ceo-brief

Why it matters: The `Verified-against-environment` discipline has been adopted by 3 artifacts across 3 types since ratification (sim-farm verdict contract at 2026-06-06; DE schema-JSON convention at 2026-05-11-2153; SRE deployment runbook at 2026-05-12-0645). The original ADR scoped to `contract` + `rfc` only; the `convention` adoption was opportunistic; the `runbook` adoption was deliberate but not explicitly scoped. **Extending the ADR's scope closes the gap between observed pattern and codified policy.** Per 2026-05-12-0645 retro P4.

**Key results**

- KR3.1: ADR-2026-05-16-003 body gains a "Sister artifact types" subsection naming the canonical scope: `contract`, `rfc`, `convention`, `runbook`, `playbook`. Original sentences referencing only `contract or rfc` extended to name the wider set.
- KR3.2: Grandfathering rule preserved: required on new artifacts authored after the original ratification date (2026-05-30) OR materially-revised existing ones. Body-shape rule unchanged.
- KR3.3: `org-os/conventions.md § "contract and rfc — Verified-against-environment subsection (required)"` retitled to `"contract, rfc, convention, runbook, playbook — Verified-against-environment subsection (required)"`. Body text extended to name the wider scope. Sister precedent: ADR-2026-05-23-001's in-place extension at 2026-05-30 (added `report` to enum) and 2026-06-13 (added `action` to enum).
- KR3.4: Per-edit tenant-isolation dry-run.

**Tasks**

- [ ] Edit ADR-2026-05-16-003 body (Sister artifact types subsection).
- [ ] Edit `org-os/conventions.md` § body-shape rule.
- [ ] Per-edit + final tenant-isolation dry-runs.

### O4: Add first-real-deploy sibling section to `org-os/playbooks/provisional-and-migrate.md`

source: ceo-brief

Why it matters: The first-real-deploy-as-integration-test pattern surfaced at 2026-05-12-0645 (home-cluster first deploy revealed 2 latent chart bugs that dry-run testing had missed). The pattern is reusable for any chart/container/infrastructure-bearing loop. Per 2026-05-12-0645 retro P5: extend `provisional-and-migrate.md` as a sibling section rather than authoring a standalone playbook (P5's own recommendation, because both patterns are about coordination-through-shape rather than coordination-through-documentation).

**Key results**

- KR4.1: New section in `org-os/playbooks/provisional-and-migrate.md` titled "Sibling pattern: first-real-deploy as integration test." Body: (a) when to invoke (any chart/container/infrastructure-bearing loop); (b) the acceptance shape ("deploys to a real target environment, exercises canonical happy path, exits cleanly on uninstall"); (c) anti-pattern (treating dry-run gates as sufficient); (d) worked example (2026-05-12-0645's home-cluster first deploy surfacing 2 latent chart bugs); (e) sister relationship to provisional-and-migrate's main pattern.
- KR4.2: Per-edit tenant-isolation dry-run.

**Tasks**

- [ ] Add the sibling section to `provisional-and-migrate.md`.
- [ ] Per-edit + final tenant-isolation dry-runs.

### O5: Author `org-os/playbooks/smallest-product-slice.md` (XS)

source: ceo-brief

Why it matters: The 2026-05-12-1826 loop picked the CLI (no backend prereq) over dashboard tabs (backend prereq) over chat experience (LLM + backend + frontend prereqs). The choice was deliberate but ad-hoc; codifying as a playbook prevents the same multi-slice scoping question from being re-derived each time. Per 2026-05-12-1826 retro P4. XS-sized; one short page.

**Key results**

- KR5.1: New file at `org-os/playbooks/smallest-product-slice.md` with `type: playbook`, `owner: board`, `date: 2026-05-13`, `status: active`, `invocation_trigger: "when scoping a new product surface that has multiple candidate slices"`.
- KR5.2: Body sections: (a) when to invoke; (b) the question to ask ("which slice has the fewest infrastructure prerequisites?"); (c) worked example (2026-05-12-1826's CLI choice vs. dashboard/chat); (d) anti-pattern (picking the highest-fidelity slice and discovering its infrastructure dependencies mid-build).
- KR5.3: Per-edit tenant-isolation dry-run.

**Tasks**

- [ ] Author `org-os/playbooks/smallest-product-slice.md`.
- [ ] Per-edit + final tenant-isolation dry-runs.

## Risks

- **5 separate `org-os/` edits in one loop is the largest org-os surface concentration since 2026-05-30 (Bundle C: 5 ritual + role edits) or 2026-05-11-2153 (4-ADR ratification + 6 mandated org-os edits).** Mitigation: per-edit tenant-isolation dry-run (proven shape at 2026-05-11-2153); final cumulative sweep; all other teams paused so no parallel `org-os/` writers create merge friction.
- **Two playbook draft ADRs flip same-loop; bundled-ratification could mask one's quality.** Mitigation: ratify in order (O1 then O2); read each playbook end-to-end before flipping its ADR's status; the existing `provisional-and-migrate.md` is the reference shape.
- **The first-real-deploy section in `provisional-and-migrate.md` is the FIRST sibling-pattern section in an existing playbook.** Sets the precedent for future playbook extensions (alternative was a standalone playbook). Mitigation: P5's own recommendation framed this; the sibling-section shape mirrors the discipline used in conventions.md's "additional fields per type" extension pattern.
- **Hot-swap step 3 slips a third time.** Bounded slip (ADR body grandfathers under absolute-date authoring) but third-consecutive-slip on a multi-loop ADR-mandated test is a scheduling-assumption signal. Mitigation: surfaced in this loop's retro for CEO decision per the prior retro's P2 options.
- **Conventions.md section header rename.** Renaming `"contract and rfc — Verified-against-environment subsection (required)"` could break any external link or future automated lint. Mitigation: keep the renamed section in the same place in conventions.md (no anchor change beyond the header text); cross-reference both old and new in the ADR body if any artifact cross-links it.

## Out of scope this loop

- **Hot-swap correctness test** (ADR-2026-05-16-001 step 3) — slipped to **loop+5** (third consecutive slip). Joint AE + resink-core deliverable. Decision point surfaced in this loop's retro.
- **2026-05-12-1254 retro P3 (multi-loop-blocker-arc report-type)** — deferred per its own timing recommendation; revisit after another arc closes.
- **2026-05-12-1826 retro P1 (document submodule-deinit-then-regenerate recovery)** — deferred to next active-resink-core loop (tenant-class).
- **CI/CD setup on the new resink-core remote** — per 2026-05-12-1254 retro P2. Future loop.
- **Resink CLI v2** (write commands; Homebrew distribution; OAuth + backend) — future loop after backend exists.
- **Job-kind chart variant** — per 2026-05-12-0645 retro P2. Future deployment-focused loop.
- **In-cluster image registry** — home-cluster roadmap Phase 1.
- **pyinfra wrapper at `services/nanofab_supervisor/`** — Phase 1.1.
- **Observability stack** — Phase 1.3.
- **Cargo MSRV declaration sync; Chart.appVersion drift** — bookkeeping.
- **GitBook publishing of `org-os/` engine docs** — separate publishing track per 2026-06-13 retro P5.
- **All product-surface work this loop (no team activation beyond board).**

## Ratifications this loop

- **ADR-2026-05-12-001 (reframe-vs-act playbook)** flips `draft → active`. Mandated edit: new `org-os/playbooks/reframe-vs-act.md` (per O1).
- **ADR-2026-05-12-002 (submodule-promotion playbook)** flips `draft → active`. Mandated edit: new `org-os/playbooks/submodule-promotion.md` (per O2).
- **ADR-2026-05-16-003 scope extension** — in-place body edit; no separate ratification ADR (sister precedent: ADR-2026-05-23-001's in-place extensions at 2026-05-30 + 2026-06-13). Mandated edit: `org-os/conventions.md § "..." subsection header + body` (per O3).
- **No contract migrations this loop.**
- **No new conventions enum additions this loop.**

**Multi-loop plan slippage absorbed (not new ratifications):**

- **ADR-2026-05-16-001 step 3 (hot-swap correctness test):** was loop+4 (= this loop), **slipped to loop+5** (= the loop after this one). **Third consecutive slip.** Decision point surfaced in this loop's retro per 2026-05-12-1826 retro P2's options.
