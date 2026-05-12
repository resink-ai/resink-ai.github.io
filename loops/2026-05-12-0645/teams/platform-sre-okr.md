---
layout: default
title: platform-sre OKR — 2026-05-12-0645
date: 2026-05-12
status: active
type: okr
loop: 2026-05-12-0645
owner: teams/platform/sre
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/sre
  date: 2026-05-12
  status: active
  loop: 2026-05-12-0645
  links: parent: board/okrs/2026-05-12-0645-ceo-brief.md
-->
# SRE OKR — 2026-05-12

## Context

SRE's second owned-surface artifact and first happy-path runbook. The first runbook (`nanofab-supervisor-failed-validation.md`, shipped at 2026-05-23) covered the failure-mode tree for verdict mismatches; this loop's deployment SOP covers the install / verify / delete happy path. Sister to the failed-validation runbook in shape and depth; cross-link both directions. Per CEO brief 2026-05-12-0645 § O3: author `runbooks/nanofab-supervisor-deployment.md` + update `home-cluster/docs/k8s-cluster.md § Smoke-test workloads` to register the supervisor as the cluster's first product workload.

The deployment procedure is executed in-loop by DevOps (their O1) — SRE authors the SOP in parallel against the same procedure. First-iteration SOPs are exercised by their author (DevOps); SRE captures the runbook shape so a second-time operator can reproduce without re-deriving from the brief. SRE's status.md is stale relative to the last loop (`teams/platform/sre/status.md` is dated 2026-06-06 in frontmatter; the 2026-05-11-2153 review-ack didn't refresh it). This loop refreshes it forward.

## Objectives

### O1: Author the `nanofab-supervisor` deployment SOP runbook

source: ceo-brief

Why it matters: The deployment procedure DevOps exercises this loop needs a written runbook so any operator can reproduce without re-deriving the steps from this brief. The runbook also documents the failure modes specific to deployment (image-pull issues, RBAC errors, scheduler failures, PVC binding) — distinct from the existing failed-validation runbook which covers verdict mismatch and supervisor-internal panic paths. Together the two runbooks cover the deployment+execution surface end to end.

**Key results**

- KR1.1: New file at `teams/platform/sre/runbooks/nanofab-supervisor-deployment.md` with `type: runbook`, `severity_tiers: [S1, S2, S3]`, `status: active`, `owner: teams/platform/sre`, `date: 2026-05-12`. Frontmatter follows the existing failed-validation runbook's shape.
- KR1.2: Body sections in order: (a) **Purpose and scope** — one paragraph: deploys `nanofab-supervisor` MVP to the home cluster (4-node kubeadm, kubernetes-admin context); covers install + verify + delete; companion to the failed-validation runbook for runtime failure modes. (b) **Prerequisites** — kubeconfig with `kubernetes-admin@kubernetes` context active, `helm` ≥ 3.x or 4.x, `docker` for image build (resink-core side), ssh access to k8s-0..k8s-3 as `lsj` with `~/.ssh/id_ed25519_hwmgmt`, the image tarball at `/tmp/nanofab-supervisor.tar`. (c) **Install procedure** — 6 steps: build (delegate to resink-core's user-guide), save (delegate), distribute (the `scp` + `ctr import` loop with explicit commands per node), `helm install --create-namespace`, `kubectl wait` for pod `Succeeded`, `kubectl cp` the trace. (d) **Verification** — `kubectl get all -n nanofab` expected object set; `kubectl logs <pod> -n nanofab` shows the supervisor's structured log lines; `kubectl cp` the trace and verify against the local mvp-loop trace (or document divergence). (e) **Delete procedure** — `helm uninstall nanofab-supervisor-mvp -n nanofab` exits 0; `kubectl get all -n nanofab` returns "No resources found"; `kubectl delete namespace nanofab`. (f) **Failure modes** — six subsections: `ImagePullBackOff` (image not loaded on scheduled node), `ErrImagePull` (typo in image name/tag), `CreateContainerConfigError` (ConfigMap or RBAC missing), `Pod Pending > 5 min` (no schedulable node OR PVC binding failure if a future iteration uses a PVC), `RBAC denied` on supervisor's ServiceAccount (chart's Role/RoleBinding missing or wrong namespace), `helm uninstall leaves residual objects` (forward-pointed; current chart has no hooks/finalizers so this should not occur; documents the diagnosis if it ever does). Each subsection follows Symptom / Diagnosis / Mitigation / Severity shape consistent with the failed-validation runbook. (g) **Cross-team notes** — cross-link to chart README's "Install (home cluster)" section, the failed-validation runbook, and the home-cluster docs. (h) **`Verified-against-environment`** per ADR-2026-05-16-003 — naming `kubectl` v1.33.x (client v1.33.9 / server v1.33.0), `helm` v4.1.4, `docker` ≥ 20.10, `ssh` with hwmgmt key, macOS or Linux operator, 4-node home cluster at 192.168.1.0/24, `kubernetes-admin@kubernetes` context, containerd `k8s.io` namespace.
- KR1.3: `repos/resink-ai/home-cluster/docs/k8s-cluster.md` § "Smoke-test workloads" gains a new subsection "### nanofab-supervisor (first product workload)" with: one-paragraph description (resink.ai's nanofab supervisor running in `sim` mode against a baked-in synthetic-tenant workspace; not long-running; exits `Succeeded`); the canonical `helm install` invocation; the canonical `helm uninstall` invocation; a cross-link to `teams/platform/sre/runbooks/nanofab-supervisor-deployment.md` and the chart README. ~15 lines; no restructuring of the existing doc.
- KR1.4: Cross-links closed: SOP runbook cross-links chart README + failed-validation runbook + home-cluster docs. Home-cluster docs cross-links the SOP. Chart README (per DevOps O1 KR1.6) cross-links the SOP. No circular dependencies — each pointer is informational.
- KR1.5: `teams/platform/sre/status.md` refreshed: bring `date:` frontmatter to 2026-05-12; "Current focus" section reflects the new SOP authoring + the prior loop's dlopen review-ack closure; "Recent shipments" includes the new SOP runbook; "Carrying into next loop" updated to drop the closed-now items and surface the `dlopen-plugins`-specific failure-mode subsections as the next opportunistic add (still depends on dlopen path being exercised in CI, not this loop).
- KR1.6: Tenant-isolation invariant holds. All edits land under `teams/platform/sre/` and `repos/resink-ai/home-cluster/`; no `org-os/` writes. Final tenant-isolation dry-run: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns only the canonical `acme.ai` placeholder.

**Tasks**

- [ ] Author `runbooks/nanofab-supervisor-deployment.md` per the KR1.2 section layout.
- [ ] Add "### nanofab-supervisor (first product workload)" subsection to `repos/resink-ai/home-cluster/docs/k8s-cluster.md § Smoke-test workloads`.
- [ ] Cross-link SOP ↔ chart README (coordinate with DevOps O1 KR1.6) ↔ home-cluster docs ↔ failed-validation runbook.
- [ ] Refresh `teams/platform/sre/status.md` (frontmatter `date: 2026-05-12`; Current focus; Recent shipments; Carrying).
- [ ] Tenant-isolation dry-run.

## Risks

- **SOP could prove insufficient on second iteration.** First-iteration SOPs always do. Mitigation: first execution is by the SOP author (DevOps), not by a third-party operator; friction surfaces during authoring. Iteration is expected at next loop's retro if a second operator hits friction.
- **`Verified-against-environment` discipline:** this is the runbook's first adoption of ADR-2026-05-16-003 (sim-farm's verdict contract was first, DE's schema-JSON convention second). The runbook isn't a `contract` or `rfc`, but the discipline transfers — naming the specific tool versions in the runbook prevents "works on my machine" friction. Mild risk if a tool version drifts; runbook acknowledges the timestamp of last verification.
- **Failure-mode subsections might miss the actual failures encountered.** First-iteration coverage is generic (image-pull, RBAC, scheduling). If DevOps's O1 surfaces a failure shape not in the runbook (e.g., a specific PVC binding bug, a containerd k8s.io namespace issue, a calico CNI hiccup), the runbook gains a new subsection in a follow-up edit. Mitigation: KR1.2's failure-mode subsections list is intentionally extensible; future loops add as needed.

## Out of scope this loop

- **`BLOCKED` observable verification** — depends on resink-core surfacing a structured BLOCKED state from the runtime coordinator; multi-loop wait, not this loop's scope.
- **`dlopen-plugins`-specific failure-mode subsections** — depend on the dlopen path being exercised in CI; this loop exercises `static-plugins-only` (default) in the home-cluster deploy.
- **SLO authoring (per-supervisor-pod)** — depends on Phase 1.3 of the home-cluster roadmap (observability stack).
- **On-call rotation policy, alert paging wiring** — depend on staffing decisions + metric-emission seam being wired.
- **Refresh pass on the existing failed-validation runbook** — non-load-bearing this loop; the new SOP is the additive deliverable.
- **Re-execution of the SOP by a second operator** — first iteration is author-executes; second-operator validation is a future loop's call.
