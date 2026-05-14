---
layout: default
title: "Action 2026-06-06-002: install minikube on dev machine"
date: 2026-06-06
status: superseded
type: action
owner: board
due: 2026-06-13
parent: Actions
nav_order: 1
---

<!-- original-frontmatter:
  type: action
  owner: board
  date: 2026-06-06
  status: superseded
  due: 2026-06-13
  links: triggering_retro: ../retros/2026-05-11-1631-ceo-retro.md
-->
{% raw %}

# Action 2026-06-06-002: Install minikube CLI on the developer machine

## Problem

DevOps's `make minikube-smoke` deferred for the third consecutive loop (2026-05-23 → 2026-05-30 → 2026-06-06). Each time, same shape: chart skeleton is `helm lint --strict` / `helm template` / `helm install --dry-run` green, but the in-cluster smoke is blocked on missing dev-machine toolchain. This loop the CEO approved Docker Desktop install (already present via OrbStack on the IC's Mac), but the same standing answer didn't name `minikube CLI` as part of the toolchain. `make minikube-smoke` exits 127 at step [1/9] (`minikube: command not found`).

**Root cause:** the brief's standing answer named "Docker Desktop install" without specifying "and `minikube` binary on PATH." DevOps reasonably read the answer literally, completed the Docker side via OrbStack (which provides the `docker` daemon), and discovered the `minikube` gap at smoke time. The 3-loop pattern means this is no longer a one-off blocker — it's the same class as workspace promotion (`board/actions/2026-06-06-001`): a single specific human action external to the loop-tree machinery.

## Action required

Install the `minikube` CLI on the developer machine. One command:

```sh
brew install minikube
```

After install, verify:

```sh
minikube version    # should print the installed version
minikube start --driver=docker --wait=all   # one-time cluster bootstrap
```

The chart's `Makefile` `minikube-smoke` target then runs the in-cluster smoke per DevOps's loop 2026-05-11-1631 KR1.2.

## Forcing function

**If this ticket has not flipped to `status: done` by 2026-06-13** (one loop boundary), the 2026-06-13 brief's "Out of scope this loop" section MUST name the implication: DevOps's minikube smoke is a 4-loop carry; the supervisor swap full execution at `loop+1` (per ADR-2026-05-16-001 step 2) cannot exercise the production deployment surface; production-time deploy smoke becomes the only validation path for chart shape changes.

The 2026-06-13 brief authors verify this ticket's status at brief authoring step 1 (re-read state, including any `status: open` `board/actions/`). If `status: open` at that time, the brief invokes the forcing function as described.

## Downstream blockers known at filing time

1. **DevOps KR1.2 (this loop):** `make minikube-smoke` exits 0 + trace.jsonl byte-identical to local — carried to 2026-06-13.
2. **dlopen restoration step 2 (loop+1 = 2026-06-13):** resink-core's supervisor swap full execution wants minikube as the in-loop production-deployment-surface smoke; without it, supervisor swap is validated only via cargo unit tests, not k8s deploy.
3. **DevOps KR1.3 (helm uninstall residual-free):** depends on KR1.2 having run.
4. **DevOps KR1.4 + KR1.5 ack-from-resink-core / DE:** filed as requests this loop, ack independent of the smoke running; only the smoke itself is blocked.

## Resolution

When `minikube` is installed:

1. Flip frontmatter `status: open → status: done`.
2. Add a "## Resolution" subsection below with date, who installed, minikube version output.
3. DevOps picks up `make minikube-smoke` in the 2026-06-13 build phase.

## Resolution

Superseded 2026-06-13 per CEO brief 2026-06-13's working answer to retro P5: toolchain installs route through team Prerequisite documentation, not board-action tickets. Chart README at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md § Prerequisites` already names `brew install minikube` as the recovery command. Future minikube smoke attempts pick up the toolchain via the chart's own surface.

## Read order

Read at every CEO brief authoring step 1 until `status: open → status: done`. Closed actions remain in the directory as a historical record.

## Same-class siblings

- `board/actions/2026-06-06-001-create-resink-core-github-remote.md` — same shape (single human action, external to loop-tree); both filed this loop. Together they comprise the canonical test of whether the `board/actions/` ticket pattern reduces multi-loop blocker recurrence.

## Note on the pattern

P5 from the 2026-06-06 retro asks the meta-question: should every external-toolchain-install blocker get a board-action ticket, or is there a different mechanism? The proposed working answer (to be decided at 2026-06-13's brief Context section): board-action tickets are right for blockers on **shared infrastructure** (GitHub remotes, IAM roles, secrets); developer-machine toolchain installs should be **per-team Prerequisite documentation** that the team running the smoke installs as part of their own build phase, not a separate board-action ticket. If that working answer is adopted at 2026-06-13, this ticket becomes the last toolchain-install board-action; future toolchain blockers route through `<chart>/README.md § Prerequisites` instead.

## Links

- Triggering retro: [board/retros/2026-05-11-1631-ceo-retro.md](../retros/2026-05-11-1631-ceo-retro.md) — P1.
- Worked-example blocker: 3-loop carry in [teams/platform/devops/status.md](../../teams/platform/devops/status.md).
- Sibling ticket: [board/actions/2026-06-06-001-create-resink-core-github-remote.md](2026-06-06-001-create-resink-core-github-remote.md).
- Chart prerequisites doc: [repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md](../../repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md) — § Prerequisites already names `brew install minikube`.
{% endraw %}
