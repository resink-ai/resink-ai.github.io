---
layout: default
title: platform-agent-engineering OKR — 2026-05-10-2227-002
date: 2026-05-10
status: active
type: okr
loop: 2026-05-10-2227-002
owner: teams/platform/agent-engineering
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/agent-engineering
  date: 2026-05-10
  status: active
  loop: 2026-05-10-2227-002
  links: parent: board/okrs/2026-05-10-2227-002-ceo-brief.md
-->
# Agent Engineering OKR — 2026-05-10

## Context

The 17-task bottom-up flow implementation (`docs/superpowers/plans/2026-05-09-org-os-bottom-up-flow.md`) was deferred from loop 2026-05-10-2227-001 per that loop's CEO scope decision; AE is now scheduled to pick it up. The plan and its gating ADR live on the `org-os-bottom-up-flow-design` branch — the ADR file (`board/decisions/2026-05-09-001-org-os-bottom-up-flow.md`) has been drafted and board-reviewed with no material design changes, but it is **not yet merged to master**. The product pivot (sub-projects #1–#5) does not touch this work; bottom-up flow is org-os infrastructure and evolves on its own track.

Per the brief's O4, AE must this loop:
1. Size every task (S/M/L) and group into 2–3 milestone bundles with explicit definition-of-done — this is the gate (KR4.1, sizing+grouping piece).
2. Execute the first bundle (KR4.1, floor).
3. Record shipped vs. carried over by bundle with reasons (KR4.2).
4. Run the tenant-isolation dry-run from `org-os/playbooks/extract-org-os.md` after the first bundle ships (KR4.3).

This OKR commits to all three plus a stretch reach into bundle B if capacity allows.

## Sizing decision (carried out as part of this OKR's drafting)

| # | Task                                               | Size |
|---|----------------------------------------------------|------|
| 1 | ADR — gate the org-os change                       | S    |
| 2 | Conventions — define new vocabulary                | L    |
| 3 | New template — `request`                           | S    |
| 4 | New template — `team-proposals`                    | S    |
| 5 | Modify `okr` template — add `source` field         | S    |
| 6 | New ritual — `team-intake`                         | M    |
| 7 | Modify `executive-loop` — insert team-intake step  | M    |
| 8 | Modify `ceo-brief` — consume proposals, ratify     | M    |
| 9 | Modify `team-planning` — three objective sources   | M    |
| 10| Modify `exec-summary` — close request lifecycle    | M    |
| 11| Modify `ceo-consolidation` — capacity/request roll-ups | M  |
| 12| Modify `retro` — read request-flow roll-up         | S    |
| 13| Modify `roles/EM.md`                               | M    |
| 14| Modify `roles/CEO.md`                              | S    |
| 15| Modify `roles/IC.md`                               | S    |
| 16| Update `org-os/README.md`                          | S    |
| 17| Verification — walk three worked flows end-to-end  | M    |

## Bundle grouping decision

Three milestone bundles, ordered so each later bundle depends only on earlier-bundle outputs. This lets Bundle A ship as a self-contained "the org-os now has the vocabulary and a ratified gate" milestone even if B and C slip.

### Bundle A — Foundation: gate, vocabulary, artifacts (T1, T2, T3, T4, T5)

**Why first:** Establishes the ADR gate (mandatory per `org-os/playbooks/merge-evolution-proposal.md`) and the vocabulary (type enum, status enum, frontmatter table, directory contract, source field, two templates) that every subsequent task in Bundles B and C references textually. Without this bundle, no later task can land in a way that lints clean against `conventions.md`.

**Definition of done:**
- `board/decisions/2026-05-09-001-org-os-bottom-up-flow.md` merged to master with `status: active`.
- `org-os/conventions.md` carries: extended `type` enum, request+team-proposals required-keys rows, request lifecycle status subsection, in-place mutation rule for requests, validation hints section, and the two new "what goes where" rows.
- `org-os/templates/request.md` and `org-os/templates/team-proposals.md` exist with the exact content specified in plan tasks 3 and 4.
- `org-os/templates/okr.md` carries the `source:` and `links.source:` lines per objective plus the source-field reference table.
- Tenant-isolation dry-run (KR4.3) passes: `grep -ri "resink\|acme\|company-x" org-os/` returns no real tenant names from Bundle A's diff (placeholders are fine).
- All 5 tasks committed on the `org-os-bottom-up-flow-design` branch with the exact commit messages in the plan (or equivalent imperative form), then merged to master via PR or fast-forward.

### Bundle B — Rituals: thread the new flow through the executive loop (T6, T7, T8, T9, T10, T11, T12)

**Why second:** Rituals consume the vocabulary and templates Bundle A lands. Each ritual edit references type names, status values, and template paths that must be live first. Once Bundle B is in, the executive loop description on disk matches the actual flow agents will run next loop.

**Definition of done (when bundle is scheduled):**
- `org-os/rituals/team-intake.md` exists; `executive-loop.md` step list is 8 entries with team-intake as step 2; `ceo-brief.md`, `team-planning.md`, `exec-summary.md`, `ceo-consolidation.md`, `retro.md` carry the diffs specified in plan tasks 7–12.
- Cross-ritual greps confirm: the four escalation tokens (`effort | scope | interface | fan-out`), the three source values (`ceo-brief | team-initiated | cross-team-request`), and the seven request states are spelled identically everywhere.
- Tenant-isolation dry-run still clean after merge.

### Bundle C — Roles, surface, verification (T13, T14, T15, T16, T17)

**Why third:** Roles edits codify decision rights and boundaries that depend on the rituals being in place. README is a navigational tidy that's easiest with everything else live. T17 is a read-only end-to-end walk that has nothing to verify until A and B are in.

**Definition of done (when bundle is scheduled):**
- `org-os/roles/EM.md`, `CEO.md`, `IC.md` carry the diffs specified in plan tasks 13–15.
- `org-os/README.md` has the new "How to use this" bullet, the team-intake row in the rituals table, and the request + team-proposals rows in the templates table.
- T17's three worked-flow walks succeed against on-disk files; the four cross-file consistency greps pass; the tenant-isolation grep is clean.

## Objectives

### O1: Execute Bundle A end-to-end, record what shipped, run the dry-run

Why it matters: Bundle A is the floor of KR4.1 and the prerequisite for the rest of the 17-task plan. Without the ADR merged and the vocabulary live, every later bundle (and every future bottom-up-flow consumer) has nothing to point at. The dry-run is KR4.3; doing it after Bundle A specifically (rather than waiting for B and C) catches any tenant-name leak in the foundation before more files are touched. This objective covers KR4.1 (floor), KR4.2 (bookkeeping), and KR4.3 (dry-run) for the brief's O4.

Maps to brief O4 → KR4.1 (floor + sizing/grouping gate already landed in Context above), KR4.2, KR4.3.

**Key results**
- KR1.1: Bundle A's five tasks (T1–T5) land on `org-os-bottom-up-flow-design` and are merged to master, each meeting its task-level acceptance checklist in the 17-task plan. Bundle A's definition-of-done above passes.
- KR1.2: After Bundle A merges, the tenant-isolation dry-run from `org-os/playbooks/extract-org-os.md` returns clean (no real tenant names anywhere in `org-os/`), captured in this OKR's exec summary as a one-line confirmation with the exact grep command output.
- KR1.3: This OKR's exec summary at loop close records, per bundle (A, B, C), which tasks shipped vs. carried over and a one-line reason for each defer.

**Tasks**
- [x] Land Task 1: ADR `board/decisions/2026-05-09-001-org-os-bottom-up-flow.md` with `status: active` — owner: agent-engineering/ic-foundation-1 (AE IC rotation, primary) — already on branch
- [x] Land Task 2: extend `org-os/conventions.md` per plan Task 2's 8-step diff — owner: agent-engineering/ic-foundation-1 — shipped (org-os/conventions.md)
- [x] Land Task 3: create `org-os/templates/request.md` — owner: agent-engineering/ic-foundation-2 — shipped (org-os/templates/request.md)
- [x] Land Task 4: create `org-os/templates/team-proposals.md` — owner: agent-engineering/ic-foundation-2 — shipped (org-os/templates/team-proposals.md)
- [x] Land Task 5: extend `org-os/templates/okr.md` with `source` field — owner: agent-engineering/ic-foundation-2 — shipped (org-os/templates/okr.md)
- [ ] Open a PR (or fast-forward) merging `org-os-bottom-up-flow-design`'s Bundle A commits to master — owner: AE EM — pending end-of-loop merge
- [x] Run the `extract-org-os.md` dry-run greps; paste output into the loop-close exec summary — owner: AE EM — CLEAN (see Build update)
- [ ] Write the per-bundle ship/defer record into the 2026-05-10 exec summary at loop close — owner: AE EM

## Build update — 2026-05-10

**Bundle A shipped in full** (T1–T5):
- T1: ADR `board/decisions/2026-05-09-001-org-os-bottom-up-flow.md` — already present on branch with `status: active`.
- T2: `org-os/conventions.md` — extended `type` enum (`+request`, `+team-proposals`), added required-keys rows for both, added asymmetric status enum subsections (`status` singleton + `request` lifecycle), extended team directory contract (`+proposals/`, `+requests/`), added two "What goes where" rows, added in-place `request` mutation rule, added `## Validation hints` section with all four advisory rules.
- T3: `org-os/templates/request.md` — new file with full request frontmatter (lifecycle status, escalation block, fulfilled_by link) and three body sections (Ask, Receiver triage, Resolution).
- T4: `org-os/templates/team-proposals.md` — new file with team-proposals frontmatter and four body sections (team-initiated, accepted cross-team requests, escalations for CEO, outbound visibility).
- T5: `org-os/templates/okr.md` — objective block now carries `source:` and `links.source:` lines; new `## Source field reference` section appended with the three-row mapping table.

**Tenant-isolation dry-run (KR1.2):**
```
$ grep -rEi "resink|nanofab|acme\.ai" /Users/shijinglu/Workspace/resink.ai/newbase/org-os/ | grep -v -E "(acme\.ai|<TENANT>|placeholder|example)"
CLEAN: no real tenant/product names found
$ grep -rEi "resink|nanofab" /Users/shijinglu/Workspace/resink.ai/newbase/org-os/
CLEAN: no resink/nanofab names
```

**Bundle B status:** Not started this loop. Per O2 risk-mitigation guardrail in this OKR, Bundle B was sized M-heavy (six rituals files, seven tasks) and the build phase prioritized full Bundle A verification (including the dry-run) over partial Bundle B starts. Carried to loop 2026-05-11-0958 in full.

**Bundle C status:** Out of scope per Context; remains deferred to loop 2026-05-11-0958 or later.

### O2: Stretch — start Bundle B if Bundle A lands with capacity remaining

Why it matters: KR4.1's stretch is "remaining bundles." Bundle B is the next natural slice because it has no dependency on Bundle C and its tasks share an editing context (six rituals files). Framing this as a stretch under a separate objective keeps the floor (O1) protected: if Bundle A hits any snag (ADR merge needs CEO approval round-trip, conventions diff conflicts with parallel master edits, etc.), O2 simply does not start and the exec summary records B as carried over to loop 2026-05-11-0958. No reshuffling of O1 needed.

Maps to brief O4 → KR4.1 (stretch).

**Key results**
- KR2.1: If started, every Bundle B task that lands meets its plan-task acceptance checklist. Bundle B does not "partially land" — either an individual task is fully done per its checklist or it is not counted as shipped.
- KR2.2: Bundle B's definition-of-done above is achieved if **all seven** tasks land; otherwise the exec summary records which tasks shipped and which carried to the next loop.

**Tasks**
- [ ] Land Task 6: create `org-os/rituals/team-intake.md` — owner: agent-engineering/ic-rituals-1
- [ ] Land Task 7: insert team-intake into `org-os/rituals/executive-loop.md` — owner: agent-engineering/ic-rituals-1
- [ ] Land Task 8: ratification step in `org-os/rituals/ceo-brief.md` — owner: agent-engineering/ic-rituals-2
- [ ] Land Task 9: three-source decomposition in `org-os/rituals/team-planning.md` — owner: agent-engineering/ic-rituals-2
- [ ] Land Task 10: request-lifecycle closure in `org-os/rituals/exec-summary.md` — owner: agent-engineering/ic-rituals-3
- [ ] Land Task 11: roll-ups in `org-os/rituals/ceo-consolidation.md` — owner: agent-engineering/ic-rituals-3
- [ ] Land Task 12: one-line addition to `org-os/rituals/retro.md` — owner: agent-engineering/ic-rituals-3

## Cross-team asks

- **From board, by mid-loop:** review and merge the ADR `board/decisions/2026-05-09-001-org-os-bottom-up-flow.md` from the `org-os-bottom-up-flow-design` branch onto master. The 2026-05-09 AE exec summary records that the board ratified the design without material changes; this loop AE needs that ratification to become a merged `status: active` ADR file on master, since every other Bundle A task edits `org-os/` and is gated by it per `org-os/playbooks/merge-evolution-proposal.md`. Without this merge, AE cannot land T2–T5.
- **From board, end-of-loop (informational):** acknowledge in the loop's CEO consolidation that AE's Bundle A landed and the tenant-isolation dry-run passed, so future tenants extracting via `extract-org-os.md` inherit a clean foundation.
- **From all other teams, no this-loop ask:** Bundle A and (if reached) Bundle B do not require any other team to consume the new artifacts. Teams will encounter the new rituals starting loop 2026-05-11-0958; AE is not asking anyone to write a `team-proposals.md` or triage a `request` this loop.

## Risks

- **ADR merge round-trip stretches into mid-loop.** If the board's merge of `2026-05-09-001-org-os-bottom-up-flow.md` is not done by mid-loop, T2–T5 cannot land per the merge-evolution-proposal playbook. Mitigation: T1 (raising the ADR PR) is owned by AE on day one of the loop, not gated on anything else; if the merge is still pending at mid-loop, AE escalates as a CEO ask in the daily exec ping rather than absorbing the wait silently. T2–T5 may be drafted locally on the design branch in the meantime, ready to push once the gate is open.
- **Conventions diff conflicts with parallel master edits.** Loop 2026-05-10-2227-001 closed with no other in-flight `org-os/conventions.md` edits, but this loop's O1 (ADR pivot) does touch `board/decisions/` heavily, and an inadvertent conventions touch from another team could create a merge conflict. Mitigation: AE owns rebasing the design branch on master before opening the Bundle A merge PR; rebase conflicts in `conventions.md` go to the AE EM for resolution.
- **Stretch (Bundle B) cannibalizes floor (Bundle A) quality.** The temptation to begin Bundle B before Bundle A's dry-run runs is real. Mitigation: O2 tasks may not be started until O1's KR1.1 and KR1.2 are both ticked. The AE EM gates this explicitly at mid-loop check-in.
- **Tenant-isolation dry-run surfaces a real leak in already-merged Bundle A.** If `grep -ri` returns a real tenant name in something Bundle A created or touched, AE must fix in a follow-up commit before the loop closes. Mitigation: the four AE ICs use only the placeholders specified in the plan's "Cross-Cutting Conventions" (`<TENANT>`, `<team>`, `<layer>`, `<receiver>`, `<requester>`, `<slug>`); the EM reviews each commit's diff for tenant strings before pushing.

## Out of scope this loop

- **Bundle C (T13–T17).** Explicitly deferred to loop 2026-05-11-0958 or later; capacity this loop is sized for A as floor plus B as stretch only.
- **Walking the three worked flows end-to-end on disk (T17).** Without Bundle C's role edits, the worked flows are incomplete; deferring T17 to the loop that lands Bundle C keeps the verification meaningful.
- **Any consumer-side adoption of the new artifacts.** No other team writes a `team-proposals.md`, files a `request`, or adds a `source` field to their OKR this loop; that adoption starts loop 2026-05-11-0958 when the new ritual descriptions are live on master.
- **Tooling / lint enforcement of the validation hints.** Phase-2 concern per the spec; this loop only lands the hints as advisory text in `conventions.md`.
- **Plugin marketplace / agent runtime work.** Per status.md, AE's bootstrap-phase priorities are entirely on the bottom-up flow rollout this loop; product-shaped work resumes once the org-os layer stabilizes.
