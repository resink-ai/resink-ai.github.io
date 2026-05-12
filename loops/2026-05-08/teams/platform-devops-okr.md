---
layout: default
title: platform-devops OKR — 2026-05-08
date: 2026-05-08
status: active
type: okr
loop: 2026-05-08
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/devops
  date: 2026-05-08
  status: active
  loop: 2026-05-08
  links: parent: board/okrs/2026-05-08-ceo-brief.md
-->
# DevOps Team OKR — 2026-05-08

## Context

DevOps is bootstrapping. No infrastructure needs to be provisioned this loop, but two decisions need to be ready by the start of the next loop: which deployment target the first demo will use, and whether a basic CI gate exists to keep `org-os/` artifacts honest. Committing to both now avoids a situation where realtime-pipeline picks tools that pin the deployment answer without DevOps having had input.

## Objectives

### O1: Identify the deployment target for the first demo

Why it matters: DevOps must commit to a deployment target before realtime-pipeline picks tools, because the engine, networking model, and resource constraints all depend on where the pipeline runs. A local Docker environment, a minikube cluster, and a cloud provider each carry different tradeoffs in reproducibility, cost, and operator burden. Without a documented recommendation, realtime-pipeline will either stall waiting for DevOps or make a de-facto decision that DevOps then has to reverse. A single ADR this loop eliminates that dependency entirely.

**Key results**
- KR1.1: ADR drafted comparing local Docker vs. minikube vs. cloud, with a recommendation and rationale for the first demo deployment target.

**Tasks**
- [ ] Draft deployment-target ADR (local Docker vs. minikube vs. cloud) — owner: teams/platform/devops
- [ ] Stub a CI workflow that lints markdown frontmatter (carry to next loop if blocked by bootstrap overhead) — owner: teams/platform/devops

## Risks

- The cloud-target option in the ADR may stall on cost decisions that require CEO or finance input not available this loop.
- The markdown-lint CI stub may slip if more urgent infrastructure scoping work surfaces once the realtime-pipeline team joins.

## Out of scope this loop

- Provisioning real infrastructure of any kind.
- Production CI/CD pipelines.
- Container image builds or registries.
- Any work for application teams (`resink-core`, `sim-farm`) — deferred to next loop.
