---
layout: default
title: "Team: platform-sre"
---
# Team: platform-sre

## Mission

Uptime, SLOs, incident response. As of 2026-05-10, SRE also owns the **operational shape** of [Nanofab Serving & Deployment](../../../docs/superpowers/specs/2026-05-10-nanofab-serving-deployment-design.md) (sub-project #4) — runbooks, on-call, and capacity surfaces — paired with DevOps's ownership of deployment topology, IaC, CI/CD gate, and secrets storage.

## Owned products

- Monitoring / alerting configs (Prometheus / Datadog / Grafana dashboards as code).
- Runbooks (storage convention: `teams/platform/sre/runbooks/<slug>.md`, one runbook per file, with `type: runbook` frontmatter).
- Chaos-engineering scripts.
- **Sub-project #4 operational surfaces** (per CEO brief 2026-05-10 O3 KR3.2):
  - **Runbooks** for nanofab-specific failure classes, starting with supervisor failed-validation against the Sim Farm verdict layers (runtime spec §6 hot swap; sim-farm spec §6 failure handling).
  - **On-call** posture for the supervisor fleet, runtime coordinator, sim-farm coordinator, and the cross-cutting Iceberg / KV / Kafka data planes — definition and rotation TBD in a future loop.
  - **Capacity surfaces** — per-tenant `LimitRange` / `ResourceQuota` health (serving spec §4.1), cost-guardrail anomaly behavior (serving spec §8.3), per-tenant Grafana datasource shape (serving spec §4.1). SRE consumes the underlying quotas from DevOps and is accountable for the "is this tenant getting starved?" observability.
  - **Region & availability** operational response (serving spec §5) — DR drill cadence, RTO/RPO verification, failover runbooks. v1 single-region multi-AZ scope; v2 multi-region work scheduled when that SKU comes online.
  - **Secrets-rotation surfacing** (serving spec §7) — SRE owns the operational view of rotation lag and expiring-customer-secret alerting; DevOps owns the rotation Lambdas and IAM roles themselves.

## Interfaces

**Consumes:** infra from DevOps; app metrics from application teams; Sim Farm verdicts (`simfarm.verdicts` Iceberg table) and runtime coordinator hot-swap state-machine transitions as alert sources for sub-project #4.

**Produces:** alerts and runbooks. For sub-project #4 specifically, SRE produces the alerting / runbook surface keyed on per-tenant `tenant_id` labels (serving spec §4.1 — every metric and structured log line is tenant-labeled).

**Boundary with DevOps for sub-project #4:** DevOps owns deployment topology, IaC (Helm + Terraform), CI/CD gate (Sim Farm seal + manifest signature enforcement at the coordinator), and secrets storage (AWS Secrets Manager + IRSA). SRE owns runbooks, on-call, and capacity / availability response. The seam: DevOps wires the metric and log emission (per-tenant labels, Prometheus scrape, log routing); SRE owns the alert rules, dashboards, and runbook content that read from it.

## Success metrics

- Every production service has an SLO.
- Every alert has a runbook.
- MTTR trends down loop-over-loop.
- For sub-project #4: every Sim Farm verdict failure class enumerated by sim-farm has a corresponding runbook section; every hot-swap `BLOCKED` transition has a defined operator response.

## Decision rights

- Decides unilaterally on: what counts as a Sev-1; SLO targets per service; runbook structure and storage convention; alert rule shape for sub-project #4 (subject to the metric labels DevOps wires).
- Must escalate: SLO targets that affect customer commitments; on-call rotation design (HR / staffing implications); cross-team capacity tradeoffs (e.g., a tenant near a hard cap).

## Out of scope

- Building services themselves.
- CI/CD pipelines (DevOps).
- Deployment topology and IaC for sub-project #4 (DevOps).
- Secrets storage and rotation Lambda implementation (DevOps); SRE owns the *observability* of rotation health, not the mechanism.
- Sub-project #5 (Product UX) operational surfaces — ownership for #5 is deferred per CEO brief 2026-05-10.

## Executive summaries

- [2026-05-12-1826](../../loops/2026-05-12-1826/teams/platform-sre-exec-summary.html)
- [2026-05-12-1254](../../loops/2026-05-12-1254/teams/platform-sre-exec-summary.html)
- [2026-05-12-0645](../../loops/2026-05-12-0645/teams/platform-sre-exec-summary.html)
- [2026-05-11-1631](../../loops/2026-05-11-1631/teams/platform-sre-exec-summary.html)
- [2026-05-11-1302](../../loops/2026-05-11-1302/teams/platform-sre-exec-summary.html)
- [2026-05-11-1113](../../loops/2026-05-11-1113/teams/platform-sre-exec-summary.html)
- [2026-05-10-2227-002](../../loops/2026-05-10-2227-002/teams/platform-sre-exec-summary.html)
- [2026-05-10-2227-001](../../loops/2026-05-10-2227-001/teams/platform-sre-exec-summary.html)
- [2026-05-09-1715](../../loops/2026-05-09-1715/teams/platform-sre-exec-summary.html)

## OKRs

- [2026-05-12-1826](../../loops/2026-05-12-1826/teams/platform-sre-okr.html)
- [2026-05-12-1254](../../loops/2026-05-12-1254/teams/platform-sre-okr.html)
- [2026-05-12-0645](../../loops/2026-05-12-0645/teams/platform-sre-okr.html)
- [2026-05-11-1113](../../loops/2026-05-11-1113/teams/platform-sre-okr.html)
- [2026-05-10-2227-002](../../loops/2026-05-10-2227-002/teams/platform-sre-okr.html)
- [2026-05-10-2227-001](../../loops/2026-05-10-2227-001/teams/platform-sre-okr.html)
- [2026-05-09-1715](../../loops/2026-05-09-1715/teams/platform-sre-okr.html)

## Runbooks

- [nanofab-supervisor-deployment](runbooks/nanofab-supervisor-deployment.html)
- [nanofab-supervisor-failed-validation](runbooks/nanofab-supervisor-failed-validation.html)
