---
layout: default
title: "ADR 2026-05-09-003: deployment target"
date: 2026-05-09
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-09
  status: active
  decision: Adopt minikube (single-node local k8s) as the deployment target for the first fact_sign_up.parquet demo; manifests are portable to cloud k8s when production demos land.
-->
# ADR 2026-05-09-003: Deployment target for the first demo

## Context

Carryover from loop 2026-05-09-1715: DevOps enumerated three deployment-target options (local Docker, minikube, cloud) but did not commit a recommendation. The CEO brief for 2026-05-09 confirms this is on the critical path for resink-core's first product-shaped OKR. Resink-core's surfaced deployment-shape requirements (in its OKR § Plan) are: a single Spark job in a pod, outputs to a persistent volume, manifest-driven so the same pipeline definition runs in dev and (eventually) production.

## Decision

Adopt **minikube** (single-node local Kubernetes) as the deployment target for the first demo. Manifests written against minikube must be portable to cloud k8s (EKS/GKE/AKS) without rewrites; only ingress, storage class, and secrets-provisioner differ at the cloud edge. The ADR commits to the target *for the first demo*, not for production.

## Alternatives considered

- **A: Local Docker (docker-compose)** — rejected because it provides no k8s semantics; we would outgrow it on the second loop when we add a second pipeline or any orchestration. Migration cost from compose to k8s would be paid before any production demo.
- **B: Cloud (EKS / GKE / AKS)** — rejected for the first demo because no team is set up for it yet (no IAM, no VPC, no shared cluster), and per-loop AWS/GCP cost is unjustified for a single-pipeline demo. Re-considered when the production demo timeline lands.

## Consequences

- Positive: Realistic developer loop on a laptop; manifests are portable; no cloud spend; no shared-resource contention during early iteration.
- Negative / costs: minikube on a laptop has resource limits (~8GB RAM minimum for a Spark workload); no shared dev cluster means each developer carries their own copy; pipeline-on-laptop is not multi-user.
- Follow-ups required:
  - DevOps documents the manifest structure choice (Helm vs. raw YAML vs. Kustomize) in a follow-up note alongside this ADR.
  - DevOps documents resource budgets per pipeline so multi-pipeline demos stay within laptop limits.
  - When a production demo lands, file a follow-up ADR for the cloud target with the same migration-path framing.

## Links

- Triggering retro: [2026-05-08-ceo-retro](../retros/2026-05-09-1715-ceo-retro.md) (DevOps deployment-target carryover)
- Related ADRs: [2026-05-09-002-streaming-engine-choice](2026-05-09-002-streaming-engine-choice.md)
- Surfaced constraints: [resink-core OKR § Plan](../../teams/application/resink-core/okrs/2026-05-10-2227-001-team-okr.md)
