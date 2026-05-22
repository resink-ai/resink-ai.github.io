---
layout: default
title: "Team: platform-devops"
parent: Teams
has_children: true
nav_order: 5
---
{% raw %}
# Team: platform-devops

## Mission

Bridge dev and ops; own deployment pipelines and cloud infra.

## Owned products

- Kubernetes manifests / Helm charts.
- Terraform / Pulumi IaC.
- CI/CD pipelines (GitHub Actions / ArgoCD).
- **Sub-project #4 (Nanofab Serving & Deployment) surfaces** — DevOps owns:
  - **Deployment topology** — k8s on AWS for cloud (EKS), minikube for local-dev; manifest portability between the two (per ADR-003). Tenant isolation enforced at the k8s layer (per-tenant `ResourceQuota`, `LimitRange`, `ServiceAccount`, labels) per serving spec §4.1.
  - **IaC** — Terraform modules under the resinkit_deployer pattern. Per-tenant IAM roles, S3 artifact buckets, Secrets Manager namespaces, IRSA bindings.
  - **CI/CD gate** — the pipeline that builds the supervisor binary, packages the Helm chart, publishes content-addressed artifacts to S3, and emits the signed manifest that the runtime coordinator validates against the Sim Farm seal (per serving spec §6.1). DevOps owns the *pipeline plumbing*; the coordinator and Sim Farm own the *validation gate*.
  - **Secrets surfaces** — AWS Secrets Manager namespacing, IRSA wiring, rotation Lambdas, and the chart-side plumbing that references externally-provisioned secrets (per serving spec §7).

## Conventions

- **Manifest tool for k8s: Helm** (charter-level default; CEO-confirmed loop 2026-05-10-2227-002). Rationale: sub-project #4's per-tenant logical-isolation pattern (per-tenant IAM, KV prefix, Kafka ACLs in shared infrastructure) is exactly the per-tenant-parameterization shape Helm charts handle well. Kustomize overlays scale poorly when every tenant needs distinct values; raw YAML is a non-starter for multi-tenant. Charts named after the binary they deploy (e.g., `nanofab-supervisor`); per-tenant values files; chart version bumps independently of the binary's image tag.
- **DevOps/SRE seam for sub-project #4:** DevOps stops at "deployable artifact + CI/CD pipeline + IaC modules + secrets plumbing." SRE picks up runbooks, on-call, alerting policy, and capacity tuning. The seam is the rendered chart + the running cluster: anything that has to render or install is DevOps; anything that has to be diagnosed or paged on is SRE.

## Interfaces

**Consumes:**
- Resink-core's supervisor-binary release: container image, entrypoint contract, required env vars, exposed ports (named hand-off section in resink-core's deliverables).
- DE's Kafka ingress contract: per-tenant SASL credential format, broker bootstrap shape, topic-naming convention (named hand-off section in DE's deliverables).
- Sim Farm's release seal contract: the seal lookup surface that the CI pipeline must reference before posting the signed manifest to the coordinator.

**Produces:**
- Deployable infra used by every team.
- Helm charts (`nanofab-supervisor`, future sibling charts), Terraform modules, CI/CD pipeline definitions, IAM role + Secrets Manager namespacing templates — consumed by SRE for operational shape and by resink-core for release plumbing.

## Success metrics

- A new service goes from PR-merged to running in production via a single CI/CD path.
- Rollback in under 5 minutes.
- A new tenant onboards via a single `helm install` (or its automation equivalent) against an existing cluster, with all per-tenant isolation surfaces (KV prefix, Kafka ACLs, IAM role, ResourceQuota) wired correctly.
- Zero unsigned or unsealed manifests ever reach the runtime coordinator (enforced at the coordinator, observed at the pipeline).

## Decision rights

- Decides unilaterally on: cloud-provider choice; CI/CD tool choice; manifest-tool choice (current default: Helm); IaC tool choice; chart layout and per-tenant values shape.
- Must escalate: cost-impacting infra changes; any change to the CI/CD gate's signature/seal requirements (joint with board + Sim Farm); BYOC (v2) topology decisions.

## Out of scope

- Runtime monitoring, alerting policy, on-call rotations, capacity tuning, runbooks (SRE).
- Application code (resink-core, sim-farm, training).
- The Sim Farm seal's *content* and the coordinator's *validator logic* — DevOps plumbs them, doesn't define them.
- Product UX (sub-project #5).
{% endraw %}
