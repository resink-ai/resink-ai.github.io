---
layout: default
title: "ADR 2026-05-12-001: reframe vs act playbook"
date: 2026-05-12
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-12
  status: active
  decision: Author an org-os playbook codifying the "reframe-vs-act" pattern for multi-loop blockers whose framing itself may be wrong.
  links: parent: board/retros/2026-05-12-0645-ceo-retro.md
-->
# ADR 2026-05-12-001 — Reframe-vs-act playbook

## Status

**Ratified at loop 2026-05-13-0056** alongside the org-os ratification bundle. Drafted at loop 2026-05-12-0645 retro § P3; one-loop-out ratification per the bundled-org-os-batch pattern. The mandated edit landed: `org-os/playbooks/reframe-vs-act.md` authored with `type: playbook`, `status: active`.

## Context

Two consecutive loops have closed multi-loop blockers not by acting on the originally-tracked action but by recognizing the action's framing was wrong:

1. **Loop 2026-05-11-2153** — Toolchain installs (minikube) as board-action tickets. The 4-loop carry on `2026-06-06-002` resolved as `superseded` because the brief's working answer to retro P5 reframed the blocker class: developer-machine toolchain installs are per-team Prerequisite docs, not shared-infrastructure board-action tickets. The chart README already named `brew install minikube`; the supersession was citation-of-existing-state rather than new authoring.

2. **Loop 2026-05-12-0645** — Minikube smoke as the deployment target. The chart had carried a "deferred minikube smoke" for 4 loops blocked on toolchain availability. This loop resolved the carry by recognizing the home cluster (`kubernetes-admin@kubernetes`; 4-node kubeadm; up 7+ days) is the actual deployment target; minikube was a stand-in. The chart's `values/home-cluster-mvp.yaml` became the canonical values file; `values/minikube-mvp.yaml` became a legacy local-dev path.

In both cases, the recurring blocker carried multiple loops with the same-shape framing. The framing was the problem; the blocker had the wrong shape. **The pattern is general enough to be codified as a playbook** so future blockers carrying 3+ loops trigger a reframing-question, not just an act-harder-question.

## Decision (draft)

Author a new playbook at `org-os/playbooks/reframe-vs-act.md` with `type: playbook` and `invocation_trigger: "when a blocker has carried 3+ loops with the same 'specific action' framing"`.

Body sections:

1. **When to invoke.** A blocker that has carried 3+ loops AND has the same-shape "specific action" framing each time AND no new information about why the action hasn't happened. Examples: "install X tool" (4-loop carry); "deploy to Y target" (4-loop carry); "ack pattern Z" (potentially).

2. **The question to ask.** "Is the blocker's framing itself the problem?" — i.e., does the action assume a shape (a specific tool, a specific target, a specific actor) that another shape could replace?

3. **Worked examples** (two from the prior loops; future-loop examples appendable):
   - Toolchain installs → Prerequisite docs (loop 2026-05-11-2153).
   - Minikube smoke → home-cluster deploy (loop 2026-05-12-0645).

4. **Anti-patterns.** Don't reframe when the action is genuinely blocking AND has a clear path. Reframing is for "wrong-shape blockers" not "right-shape-but-stuck blockers." Distinguishing requires: (a) the action has been tried and failed for non-action-related reasons; (b) a different shape exists that bypasses the blocker entirely; (c) the different shape was either already available OR is now available.

5. **Sister pattern: provisional-and-migrate.** Both patterns address coordination-shape decisions. Provisional-and-migrate handles "the artifact's type isn't admitted yet"; reframe-vs-act handles "the blocker's action isn't the right shape." Both produce worked-example-anchored playbooks.

## Mandated edits (if ratified)

- New file: `org-os/playbooks/reframe-vs-act.md` (type: playbook).
- Cross-link from `org-os/conventions.md` § Mutation rules or § Linking — TBD at ratification time.
- Sister-pattern note added to `org-os/playbooks/provisional-and-migrate.md` referencing this playbook as a related coordination-shape playbook.

## Alternatives considered

1. **Don't codify; let the pattern surface organically at each retro.** Rejected because retro authoring is loop-specific and the pattern needs to be reachable from inside-the-loop (when a blocker is carrying its 3rd loop, the brief authoring needs to consult something).

2. **Codify as a section in `provisional-and-migrate.md` rather than a sibling playbook.** Considered. Reason for sibling: the two patterns are independent (one is about artifact-type coordination; the other is about action-shape coordination). Sibling preserves discoverability and lets each playbook be invoked separately.

3. **Codify in the CEO brief ritual (`org-os/rituals/ceo-brief.md`) as a step.** Rejected because the trigger is "blocker carries 3+ loops" — a stateful condition that the ritual doesn't easily observe. A playbook invocable by retro authoring (or by IC authoring in build phase) is more flexible.

## Risks

- **Overuse of reframing.** The pattern could be invoked when the right answer is "just do the action" — e.g., for a 4-loop carry where the action has never been attempted. The "Anti-patterns" section guards against this; future-loop retros validate.
- **Pattern drift.** As more worked examples accumulate, the playbook body grows; future loops might find the playbook's examples don't match their case shape. The grandfathering posture: each invocation adds its own worked example, keeping the playbook reflective of current practice.

## Ratification path

One-loop-out draft-and-ratify. This ADR's `status: draft` at 2026-05-12-0645; the next loop's brief authoring includes O-level objective to author the playbook body + flip this ADR to `status: active`. Sister precedent: ADR-2026-06-06-001's draft-at-2026-06-06 + ratify-at-2026-06-13.

## Decision

**(Draft.)** Author `org-os/playbooks/reframe-vs-act.md` per the body shape above; ratify at next loop.
