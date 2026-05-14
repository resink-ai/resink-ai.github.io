---
layout: default
title: platform-devops Exec Summary — 2026-05-10-2227-002
date: 2026-05-10
status: active
type: exec-summary
loop: 2026-05-10-2227-002
owner: teams/platform/devops
grand_parent: Loops
parent: Loop 2026-05-10-2227-002
nav_order: 19
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/devops
  date: 2026-05-10
  status: active
  loop: 2026-05-10-2227-002
  links: parent: teams/platform/devops/okrs/2026-05-10-2227-002-team-okr.md
-->
{% raw %}

# DevOps Exec Summary — 2026-05-10

## What we shipped

This was a **plan-only** loop per [CEO brief](../../../../board/okrs/2026-05-10-2227-002-ceo-brief.md) O3 KR3.4. All deliverables are documents, not code.

### O1 — Charter claim for sub-project #4
- Charter updated in-place (`teams/platform/devops/charter.md`) to claim sub-project #4 ownership across the four surfaces: deployment topology (k8s on AWS / minikube local), IaC (Terraform per serving spec §3.1), CI/CD gate (Helm-chart publishing + Sim Farm seal + manifest-signature path per §6.1), and secrets surfaces (AWS Secrets Manager + IRSA per §7).
- Helm recorded as charter-level default for k8s manifests, per the brief.
- DevOps/SRE seam delineated: DevOps stops at "deployable artifact + CI/CD pipeline + IaC modules + secrets plumbing"; SRE owns runbooks, on-call, capacity, alerting policy.

### O2 — First sub-project-#4 slice, designed on paper
- Scope document for the `nanofab-supervisor` Helm chart skeleton landed inside the OKR (`teams/platform/devops/okrs/2026-05-10-2227-002-team-okr.md` "Sub-project #4 first slice — scope document"): chart name, k8s objects (`Deployment`, `ServiceAccount`, `ConfigMap`, `Service`, `ResourceQuota`, `LimitRange`, `Secret` ref), parameterization axes (`tenant.id`, `dag.version`, `supervisor.image.tag`, `kv.prefix`, `kafka.sasl.secretRef.name`, `iam.roleArn`, `irsa.enabled`), minikube target per ADR-003, and an explicit excluded-from-skeleton list (no real TiKV/Kafka/IRSA/Sim-Farm-seal/publishing pipeline this slice).
- Helm-fit validation walk across serving spec §4.1, §6.1, §6.3, §7.2 recorded in the OKR: three "fit", one "fit-with-caveat" (§7.2 IRSA — Terraform feeds Helm values; this is a boundary, not a blocker). **Net verdict: no blocker. Helm stands as the default.**
- Definition-of-done for the next-loop build named mechanically: `helm lint` exits 0, `helm template` produces non-empty distinct manifests for ≥2 tenant values files, `helm install --dry-run --debug` succeeds, `kubectl apply --dry-run=server` against minikube applies cleanly, every rendered object carries `tenant=<id>`, and a chart README explains values.yaml + the two-tenant rendering example.

### O3 — Carryover triage
- Status refreshed in-place to reflect the pivot: manifest-tool ask resolved (Helm per brief and validated this loop); frontmatter-lint script kept as a carry with the same deferred framing (per brief, "deferred again"); resource-budget docs re-aimed and explicitly de-prioritized to "supervisor-binary resource budget — sized once the first chart deploy runs."

## What we didn't ship and why

- **No chart code, no `values.yaml`, no Helm template.** Plan-only loop per CEO brief O3 KR3.4 — by design, not a miss.
- **No Terraform code or module sketch.** Same plan-only constraint.
- **No CI/CD pipeline for chart publishing.** Out of scope for the first slice; separate sub-slice of #4.
- **No multi-tenant install automation (installer/operator).** Later sub-slice; the skeleton is one-tenant-one-`helm install`.
- **Frontmatter-lint CI script.** Deferred again per [CEO brief](../../../../board/okrs/2026-05-10-2227-002-ceo-brief.md). Manual ADR-001 enforcement continues.
- **Cloud k8s deployment target ADR (EKS vs. alternative).** Lives inside sub-project #4 build work, not this planning loop.
- **DevOps/SRE seam for runbooks/on-call detail.** Charter-level seam declared (O1 KR1.2); operational shape is SRE's OKR, not ours.

## Surprises

- **Helm-fit walk completed cleanly on the first pass.** Expected to surface at least one caveat large enough to need a board ping; only §7.2 raised a boundary (Terraform feeds Helm values) and that boundary is industry-standard, not novel. No push-back to board needed.
- **Spec §6.1 (CI/CD seal + manifest signature) turned out to be orthogonal to Helm.** Going into the walk we assumed the seal would land somewhere in chart-publishing or chart-install; in fact it is the *runtime coordinator's* `PublishDagVersion` gate, decoupled from the supervisor binary's k8s packaging. That simplifies the chart skeleton meaningfully — the chart deploys the supervisor; the supervisor fetches and validates the signed DAG manifest at runtime.
- **Spec §6.3 (plugin distribution) is similarly chart-agnostic.** Plugins are S3-fetched at supervisor startup; the chart only exposes `s3.artifactBucket`. Pre-walk we expected a CSI or initContainer pattern; turns out neither is required for the v1 shape.
- **Resource-budget carryover re-aim needed twice.** Originally a Spark-shaped doc, then re-aimed at the pivot to "Rust supervisor budget"; this loop re-aimed again to "sized once first chart deploys" because plan-only means we cannot benchmark. Pattern worth flagging at retro — carryovers that depend on real workload data should be marked as such at creation, not at second re-aim.

## Asks

- **From resink-core (next loop):** a *named section* (per 2026-05-09 retro P3) in one resink-core document specifying the supervisor binary's container-image entrypoint, required env vars, exposed ports, and graceful-shutdown signal. Without these, the chart's `Deployment` shape is guesswork.
- **From DE (next loop):** a *named section* in DE's Kafka ingress contract (per [CEO brief](../../../../board/okrs/2026-05-10-2227-002-ceo-brief.md) O2 KR2.4) giving the per-tenant SASL credential format, broker bootstrap address shape, and topic-naming convention DevOps must wire into the chart's `kafka.*` values.
- **From CEO/board:** none this loop. The Helm-fit walk closed the open manifest-tool question; no new blockers surfaced.
- **For SRE coordination (not blocking, surfacing the seam):** when DevOps starts building the chart skeleton next loop, SRE writes the runbook for "supervisor pod fails to start under chart-rendered manifests" against the same chart. Co-locate or split is SRE's call.

## Metrics

- **O1 KR1.1** — charter updated in-place with sub-project #4 ownership block + Helm preference + DevOps/SRE seam: **done**.
- **O1 KR1.2** — charter explicitly delineates the DevOps/SRE seam: **done**.
- **O2 KR2.1** — scope document for the Helm chart skeleton (chart objects, parameterization axes, exclusions, target environment): **done** (lives in the OKR).
- **O2 KR2.2** — Helm-fit validation note across §4.1, §6.1, §6.3, §7.2 with verdicts: **done**; net verdict no blocker, no board push-back.
- **O2 KR2.3** — definition-of-done for the next-loop deliverable, mechanically checkable: **done**.
- **O3 KR3.1** — `status.md` updated per the post-pivot triage rules: **done** (this loop).
- Plan-only loop adherence: **on-track** — no scope creep into code.
{% endraw %}
