---
layout: default
title: platform-devops OKR — 2026-05-10
date: 2026-05-10
status: active
type: okr
loop: 2026-05-10
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/devops
  date: 2026-05-10
  status: active
  loop: 2026-05-10
  links: parent: board/okrs/2026-05-10-ceo-brief.md
-->
# DevOps OKR — 2026-05-10

## Context

The product vision pivoted: the runtime is now purpose-built Rust (sub-project #1), not Spark. Sub-project #4 (Nanofab Serving & Deployment) lands under DevOps + SRE per the [CEO brief](../../../../board/okrs/2026-05-10-ceo-brief.md) O3. DevOps owns deployment topology, IaC, CI/CD gate, and secrets surfaces; SRE owns operational shape, runbooks, on-call, capacity. The board's open question from our 2026-05-09 exec summary is answered: **Helm** is the preferred manifest tool. ADR-003 (minikube) survives the pivot — the serving spec is k8s-shaped end-to-end. This loop is **plan-only**: we update the charter, design the first sub-project-#4 slice on paper, and ship no chart code or Terraform. Implementation starts next loop.

## Objectives

### O1: Claim sub-project #4 ownership at the charter level

Why it matters: Per [CEO brief](../../../../board/okrs/2026-05-10-ceo-brief.md) O3 KR3.1, DevOps's charter must explicitly declare ownership of the deployment topology, IaC, CI/CD gate, and secrets surfaces for sub-project #4, and record Helm as the manifest-tool default. Without the charter-touch, next-loop work has no standing scope to reference.

**Key results**
- KR1.1: `teams/platform/devops/charter.md` updated (in-place per conventions §"Mutation rules") to add sub-project #4 ownership covering: deployment topology (k8s on AWS / minikube for local), IaC (Terraform per spec §3.1), CI/CD gate (Helm chart publishing + the Sim Farm seal + manifest-signature path per spec §6.1), and secrets surfaces (AWS Secrets Manager + IRSA per spec §7). Helm recorded as charter-level default for k8s manifests. Existing valid charter content (mission, decision rights, success metrics) preserved.
- KR1.2: Charter explicitly delineates the DevOps/SRE seam for sub-project #4 — DevOps stops at "deployable artifact + CI/CD pipeline + IaC modules + secrets plumbing"; SRE owns runbooks, on-call, capacity tuning, alerting policy. Reduces drift risk next loop.

**Tasks**
- [ ] Edit `teams/platform/devops/charter.md` to add sub-project #4 ownership block + Helm preference + DevOps/SRE seam — owner: teams/platform/devops

### O2: Design the first sub-project-#4 slice (plan-only): Helm chart skeleton for the supervisor binary

Why it matters: Per [CEO brief](../../../../board/okrs/2026-05-10-ceo-brief.md) O3 KR3.3 + KR3.4, DevOps produces a plan-only first-slice OKR this loop. The brief suggests "Helm chart skeleton for the supervisor binary deployment, parameterized for tenant + version, against minikube." This OKR scopes that slice for **next-loop build**. Plan-only framing matches O2/O3 of the brief and is a conscious capacity choice — no chart code, no Terraform code lands this loop.

**Key results**
- KR2.1: A scope document for the next-loop Helm chart skeleton exists in this OKR (sections below), naming: chart name, the kubernetes objects the chart will produce (`Deployment`, `ServiceAccount`, `ConfigMap`, `Service`, `ResourceQuota`, `LimitRange` — per serving spec §4.1 and §7.2), the parameterization axes (`tenant_id`, `dag_version`, `supervisor_image_tag`, `kv_prefix`, `kafka_sasl_secret_ref`, `iam_role_arn`), the target environment (minikube per ADR-003), and what is **explicitly excluded** from the skeleton (no real KV cluster, no real Kafka, no IRSA on minikube — those are stubbed/mocked).
- KR2.2: A "Helm-fit validation" note exists below that walks the serving spec sections most likely to surface a constraint Helm cannot meet (§4.1 isolation, §6.1 CI/CD, §6.3 plugin distribution, §7.2 IRSA), records the verdict (fit / fit-with-caveat / blocker), and — if any blocker — raises it as a cross-team ask to the board this loop. Per the brief's risks section, "the answer is a default, not a lock."
- KR2.3: The next-loop deliverable is named with a definition-of-done that two reviewers (board + SRE) can mechanically check: chart renders against minikube via `helm template`, `helm install --dry-run` succeeds for ≥ 2 distinct tenant values files, and the resulting manifests pass `kubectl apply --dry-run=server` against a minikube cluster.

**Tasks**
- [ ] Write the first-slice scope document (chart objects, parameterization axes, exclusions) into the section below — owner: teams/platform/devops
- [ ] Walk the four serving-spec sections (§4.1, §6.1, §6.3, §7.2) against Helm capability; record verdict — owner: teams/platform/devops
- [ ] Write the definition-of-done for the next-loop deliverable — owner: teams/platform/devops
- [ ] Raise any Helm blocker as a cross-team ask to the board (this loop) — owner: teams/platform/devops

### O3: Carryover triage

Why it matters: Our 2026-05-09 `status.md` carries three items into this loop: frontmatter-lint script, manifest-tool choice, resource-budget docs. The brief resolves one (Helm) and explicitly defers another (frontmatter-lint, again). Resource-budget docs were an ADR-003 follow-up shaped for Spark workloads; under the pivot the relevant resource budget is the Rust supervisor's, which we cannot size until the first build slice runs. Triage them so `status.md` reflects post-pivot reality, not pre-pivot debt.

**Key results**
- KR3.1: `teams/platform/devops/status.md` updated to: drop "manifest-tool choice" (resolved); keep "frontmatter-lint script" carrying with the same deferred framing (per brief: "deferred again"); re-aim "resource-budget documentation" to "supervisor-binary resource budget — sized once the first chart deploy runs next loop or the one after" and lower its priority explicitly.

**Tasks**
- [ ] Update `teams/platform/devops/status.md` per KR3.1 — owner: teams/platform/devops

## Sub-project #4 first slice — scope document (per O2 KR2.1)

This section is the **plan-only** deliverable for the next-loop build. No code lands this loop.

### Chart name and shape
- Chart name: `nanofab-supervisor` (singular; one chart per binary, parameterized).
- Chart version: starts at `0.1.0`; bumps independently of the supervisor image tag.
- Repository location (next loop): TBD between `repos/resink-ai/resink-core/deploy/charts/` (co-located with the binary) and `websites/resinkit_deployer/charts/` (co-located with existing IaC). DevOps decides at build time; the brief does not mandate.

### Kubernetes objects produced by the skeleton
Driven by serving spec §4.1 (logical isolation) and §7.2 (IRSA):
- `Deployment` — supervisor pods, replicas parameterized.
- `ServiceAccount` — annotated for IRSA (annotation gated behind a `if .Values.irsa.enabled` block so minikube path skips it).
- `ConfigMap` — non-secret runtime config (`tenant_id`, `kv_prefix`, `dag_version`, broker addresses).
- `Service` — ClusterIP for supervisor's coordination ports.
- `ResourceQuota` + `LimitRange` — per-tenant CPU/memory bounds (serving spec §4.1 row "Compute").
- `Secret` reference (not the Secret itself — chart references an externally-provisioned secret name) — for Kafka SASL credentials.

### Parameterization axes (values.yaml shape)
- `tenant.id` — string, required, propagates to `kv_prefix`, ResourceQuota name, ServiceAccount name, label `tenant=<id>` on every object.
- `dag.version` — string, required, becomes a ConfigMap value and a pod label.
- `supervisor.image.tag` — string, required, the container image tag.
- `supervisor.image.repository` — string, default `nanofab/supervisor`.
- `kv.prefix` — string, default `t/<tenant.id>/` (computed if not set).
- `kafka.sasl.secretRef.name` — string, required when Kafka is real; mock when target is minikube.
- `iam.roleArn` — string, optional (only set when `irsa.enabled=true`).
- `irsa.enabled` — bool, default `false` (minikube path); `true` on EKS.
- `resources.requests` / `resources.limits` — standard k8s shape, with sane minikube-friendly defaults.

### Target environment
- Minikube per ADR-003. The chart MUST render and install cleanly against a default `minikube start` cluster.
- All paths that require AWS-specific resources (IRSA, Secrets Manager) gated behind a feature flag in values (default off for minikube).

### Explicitly excluded from the skeleton (deferred to later slices)
- No real TiKV / KV cluster — `kv.endpoint` points at a placeholder.
- No real Kafka cluster — the chart references a `kafka.bootstrapServers` value pointing at a placeholder.
- No real IRSA — `irsa.enabled=false` is the only path tested this loop's-plus-one.
- No Sim Farm seal verification — that's the *coordinator's* gate (serving spec §6.1 step 7), not the chart's.
- No CI/CD pipeline that publishes the chart — separate sub-slice; this skeleton is the chart only.
- No multi-tenant deploy automation — one `helm install` per tenant in the skeleton; an installer/operator that fans out is later.

### Definition-of-done for the next-loop build
A reviewer can mechanically verify:
1. `helm lint charts/nanofab-supervisor` exits 0.
2. `helm template charts/nanofab-supervisor -f values/tenant-a.yaml` and `-f values/tenant-b.yaml` both produce non-empty, distinct manifests.
3. `helm install --dry-run --debug` succeeds for both tenant values files.
4. `kubectl apply --dry-run=server` against a running minikube applies cleanly for both.
5. Rendered manifests show `tenant=<id>` label on every object (validation that the parameterization wires through).
6. README at the chart root explains values.yaml axes and shows the two-tenant rendering example.

## Helm-fit validation against the serving spec (per O2 KR2.2)

Walk the four spec sections most likely to surface a Helm-incompatibility blocker. Each entry: section reference, what Helm has to do, verdict.

### Serving spec §4.1 — logical isolation in shared infrastructure
- **Demand on Helm:** parameterize every k8s object with `tenant_id`; per-tenant `ResourceQuota` + `LimitRange`; per-tenant `ServiceAccount`; per-tenant labels for cost tagging and metric/log filtering.
- **Verdict:** **fit.** This is exactly the per-tenant-parameterization shape Helm charts handle well (per the brief's rationale). Each tenant gets its own `helm install <tenant-id> nanofab-supervisor -f values/<tenant-id>.yaml`. No constraint surfaced.

### Serving spec §6.1 — CI/CD gate (Sim Farm seal + manifest signature)
- **Demand on Helm:** none directly. The gate is enforced by the *runtime coordinator's* `PublishDagVersion` endpoint validating the signed manifest + seal — it does not touch k8s manifests. The chart only deploys the supervisor binary; the *DAG manifest* it consumes is fetched at supervisor startup from the coordinator (spec §6.3, §4.2).
- **Verdict:** **fit.** Helm and the AI-codegen CI/CD gate are orthogonal: Helm packages the supervisor binary; the seal + signature gate the DAG manifest the binary consumes. No constraint surfaced. (Adjacent: the chart's own publishing pipeline — `helm push` to an OCI registry on supervisor-binary release — is a separate CI lane and does not interact with the seal.)

### Serving spec §6.3 — plugin distribution to supervisors
- **Demand on Helm:** none. Plugin `.so` files are fetched by the supervisor at startup from S3, not deployed via k8s. The chart provides the IAM role + S3 bucket-name config; the *fetching* is the supervisor's runtime concern.
- **Verdict:** **fit.** Helm doesn't need to know about plugins. The chart exposes `s3.artifactBucket` and the supervisor handles the rest.

### Serving spec §7.2 — IRSA per-tenant role binding
- **Demand on Helm:** template the `ServiceAccount` with an `eks.amazonaws.com/role-arn` annotation, parameterized per tenant. Standard EKS pattern; widely used in production Helm charts.
- **Verdict:** **fit-with-caveat.** Caveat: the `iam.roleArn` value must be set by whatever provisions the chart install (Terraform-generated values file is the natural pattern). The chart itself doesn't create the IAM role — Terraform does. This is a *boundary*, not a blocker: Terraform outputs feed Helm values. Documented in the scope doc above as `irsa.enabled` + `iam.roleArn`.

### Net verdict on Helm
**No blocker surfaced.** Helm fits the serving spec's v1 demands. The CEO answer stands. No push-back to the board this loop. Adjacent recommendation for SRE: when SRE writes the operational shape, the per-tenant Grafana datasource pattern (spec §4.1 row "Logs / metrics") is also a Helm-friendly per-tenant template; SRE may want to co-locate that with this chart or split it into a sibling chart.

## Cross-team asks

- **From board:** none this loop. The Helm-fit walk found no blocker; the manifest-tool answer (Helm) stands.
- **From SRE:** none required this loop. **Coordination hand-off for next loop:** when DevOps starts building the chart skeleton, SRE writes the runbook for "supervisor pod fails to start under chart-rendered manifests" against the same chart. Co-locate or split — SRE's call. Captured here to surface the seam, not as a blocking ask.
- **From resink-core:** **named hand-off section** in resink-core's next-loop deliverables — the supervisor binary's expected container-image entrypoint, required env vars, exposed ports, and graceful-shutdown signal. Without these, the chart's `Deployment` shape is guesswork. Per the 2026-05-09 retro P3, the hand-off lives as a *named section* in one resink-core document, not a separate doc.
- **From DE:** **named hand-off section** in DE's Kafka ingress contract (per [CEO brief](../../../../board/okrs/2026-05-10-ceo-brief.md) O2 KR2.4) — the per-tenant SASL credential format, broker bootstrap address shape, and topic-naming convention DevOps must wire into the chart's `kafka.*` values. Same retro P3 pattern.

## Risks

- **Helm-fit walk done by DevOps alone may miss spec details.** Mitigation: the verdict above is "no blocker"; if SRE finds one when writing operational shape, raise it next loop as an addendum rather than relitigating now.
- **The "explicitly excluded" list in the scope doc may grow when implementation starts.** Mitigation: that's expected; the list is the floor of exclusions, not the ceiling. Next-loop OKR closes it.
- **Frontmatter drift over a second consecutive deferred-lint loop.** Mitigation: per 2026-05-09 OKR's risk note, retro will review any drift. Manual ADR-001 enforcement continues.
- **Resource-budget doc re-aim may need a third re-aim if the chart skeleton doesn't run anything real.** Mitigation: KR3.1 explicitly lowers priority and rebinds it to "sized once the first chart deploy runs" — a conscious "we'll know more later" deferral, not a forget.

## Out of scope this loop

- Any chart code, any values.yaml file, any Helm template — plan-only this loop per [CEO brief](../../../../board/okrs/2026-05-10-ceo-brief.md) O3 KR3.4.
- Any Terraform code or module sketch — same plan-only constraint.
- The CI/CD pipeline that publishes the chart to an OCI registry — separate slice.
- Multi-tenant install automation (an installer or k8s operator that fans `helm install` across tenants) — a later sub-slice of #4.
- Frontmatter-lint script — deferred again per brief.
- Cloud k8s deployment target ADR — lives inside #4 work, not this loop.
- The DevOps/SRE seam for runbooks and on-call — surfaced in O1 KR1.2 at the charter level, but the operational shape is SRE's OKR, not ours.
