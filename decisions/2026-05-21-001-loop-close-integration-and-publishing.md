---
layout: default
title: "ADR 2026-05-21-001: loop close integration and publishing"
date: 2026-05-21
status: active
type: adr
owner: board
parent: Decisions (ADRs)
nav_order: 7
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-21
  status: active
  decision: The executive loop is not complete at retro -- the rit-executive-loop skill gains an explicit loop-close step requiring loop artifacts and build-phase code to be committed, reviewed, merged, and (for publishing tenants) deploy-verified, with a fixed leaves-before-root submodule integration order.
-->
{% raw %}

# ADR 2026-05-21-001: Loop-close integration & publishing

## Context

The `rit-executive-loop` skill (`plugins/org-os/skills/rit-executive-loop/SKILL.md` in the resink-marketplace skills repo) orchestrates the CEO-led cadence and, until now, treated a loop as complete once `rit-retro` had written `board/retros/<loop-id>-ceo-retro.md`. In practice a loop leaves uncommitted changes scattered across several repos — the parent tenant repo, one or more product/code submodules, and any published-docs mirror — and the skill said nothing about integrating them. Two recurring failures resulted: (1) submodule pointers bumped to feature-branch commits that a later squash-merge orphaned, leaving the parent repo referencing an unreachable commit — the pathology behind the repeated `submodule: bump … to squash-merged …` cleanup commits; and (2) "the retro is written, so we're done" — loops declared complete while their artifacts sat unmerged or, for publishing tenants, while the deploy CI had not been verified green. The friction is freshly visible at loop-close and the skill was silent on it.

## Decision

Adopt the loop-close integration & publishing discipline as a permanent part of the `rit-executive-loop` skill. The skill gains a step 8, **"Integrate & publish"**, plus a dedicated **"Loop-close: integration & publishing"** section and an **"Operational habits"** section. The substantive rules:

1. **One branch name, every repo** — `loop-<loop-id>-<slug>` across the parent tenant repo and every submodule and mirror the loop touches, so the loop's footprint is greppable.
2. **Publish before committing the parent** — if the tenant publishes loop artifacts, run the publish step first; its output is itself a change to commit.
3. **Submodules: leaves before root** — commit + PR + merge inside each submodule's own repo first, then bump the submodule pointer in the parent. Merge order: published-docs mirror → code submodules → parent tenant repo. The parent's submodule pointers must always reference commits reachable on the submodule's default branch.
4. **A squash-merge orphans the feature-branch commit** — after a submodule PR is squash-merged, re-sync the submodule to the squash-merged commit on its default branch, then make a dedicated `submodule: bump <name> to squash-merged <loop-id>` commit on the parent. Never fold the bump into another commit.
5. **Companion PRs cross-reference** — each PR body links its companions; the parent PR states the explicit merge order.
6. **Verify the deploy, not just the merge** — for a CI-built publish target, poll the deploy run to completion and confirm it is green before declaring the loop done.

Codified alongside as operational habits: verify-before-PASS (a green-gate KR is not PASS until the command ran and the output confirmed it), track each loop phase as a task, run the tenant-isolation sweep before done, and end every loop by naming the next one.

This ADR is the mandatory org-os gate (`org-os/playbooks/merge-evolution-proposal.md` step 4) for the change. It is a CEO-initiated out-of-band change rather than a retro-classified one, ratified directly under the CEO's authority over org-os evolution.

## Alternatives considered

- **A: Leave it to operator habit / per-loop instruction.** Rejected — the orphaned-pointer cleanup commits and premature "loop done" claims are the existence proof that habit did not hold. The discipline has to live in the skill the operator actually reads at loop-close.
- **B: Encode it in `org-os/rituals/executive-loop.md` only, not the skill.** Rejected as the primary home — the skill is the agent-facing operating manual that gets run; the ritual spec is consulted less often at loop-close. The ritual spec should be brought into sync as a follow-up (below), but the skill is where the change must land to take effect.
- **C: Build tooling (a loop-close script / CI check) instead of written discipline.** Deferred — same reasoning as ADR-2026-05-10-003 alternative B: tooling is a Phase 2 escalation. For a small team, written discipline in the skill is sufficient if the skill enforces it.

## Consequences

- **Positive:** Loop-close stops orphaning submodule pointers; the parent repo always references reachable commits. "Loop done" becomes a verifiable claim (merged + deploy-green), not an assertion. The integration order is fixed and greppable. The operational-habits section folds in the verify-before-PASS discipline that ADR-2026-05-10-003 established at ritual transitions.
- **Negative / costs:** Loop-close grows a multi-repo integration step with real sequencing cost. Operators must hold the leaves-before-root order and the squash-merge re-sync step — non-obvious until learned.
- **Follow-ups required:**
  - Sync `org-os/rituals/executive-loop.md` — its step 8 ends at "Run retro" and its Acceptance criteria list only on-disk artifact existence; both should gain the integrate-and-publish step so the ritual spec and the skill do not diverge. Named here, not done in this change.
  - Reference this ADR in the next CEO brief so the change propagates awareness to all teams (per `merge-evolution-proposal.md` step 4).

## Links

- Org-os change gate: [org-os/playbooks/merge-evolution-proposal.md](../../org-os/playbooks/merge-evolution-proposal.md) step 4 (Org-OS class)
- Out-of-retro routing: [org-os/playbooks/out-of-retro-org-os-change.md](../../org-os/playbooks/out-of-retro-org-os-change.md)
- Changed artifact: `plugins/org-os/skills/rit-executive-loop/SKILL.md` in the resink-marketplace skills repo (submodule `repos/resink-ai/resink-marketplace`)
- Ritual spec to sync (follow-up): [org-os/rituals/executive-loop.md](../../org-os/rituals/executive-loop.md)
- Related ADRs: 2026-05-10-003-verify-state-claims-at-ritual-transitions, 2026-05-12-002-submodule-promotion-playbook
{% endraw %}
