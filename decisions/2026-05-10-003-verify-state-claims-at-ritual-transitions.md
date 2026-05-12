---
layout: default
title: "ADR 2026-05-10-003: verify state claims at ritual transitions"
date: 2026-05-10
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-10
  status: active
  decision: State-claims (shipped / merged / reserved on master) at exec-summary, ceo-brief, and ceo-consolidation ritual transitions must carry a verifiable git location, or be re-phrased as "shipped to branch <X>, awaiting merge."
-->
# ADR 2026-05-10-003: Verify state-claims at ritual transitions

## Context

In loop 2026-05-10-2227-002, AE flagged that ADR `2026-05-09-001-org-os-bottom-up-flow` — recorded as "merged to master via the design branch's PR" with "the slot reserved on master" in the 2026-05-09 exec summary — was on no branch's `board/decisions/`. The design branch's PR had not landed. AE absorbed the mid-loop gate slip; the board wrote the ADR inline. The exec-summary ritual today accepts textual "shipped" claims without verification. The CEO brief at loop start reads exec summaries and treats them as truth, then commits new work assuming the referenced artifacts are present on the working branch. When a "shipped" claim turns out to be "shipped on the author's branch, never merged," the next loop's brief loses the artifact mid-step and the cost lands on whichever team owns the dependent work. The discipline failure is not the slip itself — branches don't always merge cleanly — but the language: textual "shipped" without git-state verification creates an implicit truth claim the rituals don't enforce.

## Decision

Three rituals gain explicit state-verification steps:

1. **`org-os/rituals/exec-summary.md` step 1** — items claimed as "shipped to master," "merged," or "reserved on master" MUST carry a verifiable git location: either `verified-at: <ref/sha>` (the artifact is on master at that ref) or `on-branch: <name>` (the artifact is on a named branch, awaiting merge). Items that cannot produce either form are recorded as "shipped to branch `<X>`, awaiting merge" — the truthful framing — and surface in the team's asks list if the merge dependency is cross-team.
2. **`org-os/rituals/ceo-brief.md` step 2** — when scoring the gap, the CEO verifies that every artifact referenced as an input is present on the brief's working branch. Missing artifacts surface as a named "merge / cherry-pick action" in the brief context with an explicit owner (the team responsible for landing the merge or the CEO for cross-team coordination). The brief does not commit dependent work to a team without the input artifact in branch.
3. **`org-os/rituals/ceo-consolidation.md` step 1** — when reading each team's exec summary, the CEO walks per-team summaries and flags any unverified "shipped to master" claim before publishing the company exec summary. Flagged claims are either resolved (the team produces the ref) or rewritten in the company summary as "shipped to branch, awaiting merge."

The light-touch text convention is the chosen path; automated git-state lint via a script is deferred to Phase 2 tooling.

## Alternatives considered

- **A: Status quo + retro-only catch.** Rejected because retro-only catch means the consumer team has already paid the cost; the 2026-05-10 loop is the existence proof of the cost.
- **B: Automated git-state check via a pre-merge script.** Deferred to Phase 2 — adds tooling burden during Phase 1 and is not strictly needed for a small team that authors a small number of state-claims per loop. The text convention is sufficient if the rituals enforce it.
- **C: Convert all "shipped" claims to PR links by convention.** Considered; the verifiable-git-location form subsumes this (a PR link is a valid `verified-at` value if it points at a merge commit) without forcing PR-shape language for direct-to-master commits.

## Consequences

- **Positive:** Cross-loop trust in exec-summary claims becomes verifiable. The CEO brief's input layer is closed against drift between author intent and branch state. Cross-team dependency hand-offs gain a forcing function (the consumer team can verify the producer's branch state before committing dependent work). The mid-loop ADR-gate slip pathology that triggered this ADR is structurally prevented.
- **Negative / costs:** Marginal authoring overhead per shipped item (one git ref or branch name). The CEO brief authoring step grows by a verification sub-step (read the working branch, check every referenced artifact). Teams must keep their working-branch state legible to the CEO at brief time — surfaces a discipline cost on branch hygiene.
- **Follow-ups required:** Update exec-summary, ceo-brief, and ceo-consolidation ritual files (this loop). Loop 2026-05-11-1113 is the first loop the convention applies to authoring; loop 2026-05-11-1302 retro evaluates whether the text convention has caught any near-misses or whether Phase 2 tooling needs to escalate.

## Links

- Triggering retro: [board/retros/2026-05-10-2227-002-ceo-retro.md](../retros/2026-05-10-2227-002-ceo-retro.md) (P1)
- Driving incident: loop 2026-05-10-2227-002 mid-loop ADR-001-on-master gate slip — see [board/exec-summaries/2026-05-10-2227-002.md](../exec-summaries/2026-05-10-2227-002.md) § "Cross-cutting blockers"
- Rituals updated: [org-os/rituals/exec-summary.md](../../org-os/rituals/exec-summary.md), [org-os/rituals/ceo-brief.md](../../org-os/rituals/ceo-brief.md), [org-os/rituals/ceo-consolidation.md](../../org-os/rituals/ceo-consolidation.md)
- Related ADRs: [2026-05-09-005-carryover-load-in-brief](2026-05-09-005-carryover-load-in-brief.md), [2026-05-09-006-org-os-change-routing](2026-05-09-006-org-os-change-routing.md), [2026-05-10-004-contract-artifact-type](2026-05-10-004-contract-artifact-type.md), [2026-05-16-003-contract-environment-verification](2026-05-16-003-contract-environment-verification.md)
