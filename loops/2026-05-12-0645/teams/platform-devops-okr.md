---
layout: default
title: platform-devops OKR — 2026-05-12-0645
date: 2026-05-12
status: active
type: okr
loop: 2026-05-12-0645
owner: teams/platform/devops
grand_parent: Loops
parent: Loop 2026-05-12-0645
nav_order: 12
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/devops
  date: 2026-05-12
  status: active
  loop: 2026-05-12-0645
  links: parent: board/okrs/2026-05-12-0645-ceo-brief.md
-->
{% raw %}

# DevOps OKR — 2026-05-12

## Context

The home cluster (`kubernetes-admin@kubernetes`; 4 nodes; k8s 1.33; MetalLB + local-path-provisioner) has been up and idle for 7+ days. The `nanofab-supervisor` chart has been authored, dry-run-clean for 4 loops, and blocked on a developer-machine minikube install that's now structurally irrelevant. **This loop is the chart's first execution against real infrastructure.** Per CEO brief 2026-05-12-0645 § O1: deploy + delete `nanofab-supervisor` to the home cluster end-to-end; document the procedure in the chart README's new "Install (home cluster)" section. The chart values are nearly minikube-compatible; the principal delta is `workspaceMount.enabled: false` (workspace baked into image) and `image.pullPolicy: IfNotPresent` (image pre-loaded on nodes via `ctr import`).

The image build itself is delegated to resink-core (their O1 KR2.1-2.3); DevOps consumes the image tarball, distributes it, runs `helm install`, verifies, and runs `helm uninstall`. SRE authors the companion SOP runbook in parallel (their O3); DevOps cross-links it from the chart README.

## Objectives

### O1: Deploy `nanofab-supervisor` to the home cluster, then delete it cleanly

source: ceo-brief

Why it matters: First product workload on real Kubernetes infrastructure. Validates the chart end-to-end (image distribution → install → pod-reaches-Succeeded → trace extraction → uninstall → zero residual objects). Establishes the canonical deploy target (home cluster, replacing minikube) and the canonical image-distribution path (per-node `ctr import` for now; in-cluster registry deferred to home-cluster roadmap Phase 1). Cross-links into the SRE SOP and resink-core user-guide for a complete documentation surface.

**Key results**

- KR1.1: New file at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml`. Sister to `values/minikube-mvp.yaml`. Sets `tenant: mvp-tenant`, `fleetColor: blue`, `replicas: 1`, in-memory KV, empty Kafka + coordinator, `workspaceMount.enabled: false`, `image.pullPolicy: IfNotPresent`, `pod.restartPolicy: Never`, `healthz.enabled: false`. Inline comments name the home-cluster context. Helm gates: `helm lint --strict` + `helm template ... --values values/home-cluster-mvp.yaml` + `helm install --dry-run` all exit 0 against the new values file.
- KR1.2: Image distribution to all 4 nodes executed successfully. For each of k8s-0 (192.168.1.162), k8s-1 (192.168.1.160), k8s-2 (192.168.1.163), k8s-3 (192.168.1.164): `scp -i ~/.ssh/id_ed25519_hwmgmt /tmp/nanofab-supervisor.tar lsj@<ip>:/tmp/` + `ssh -i ~/.ssh/id_ed25519_hwmgmt lsj@<ip> 'sudo ctr -n=k8s.io images import /tmp/nanofab-supervisor.tar'` exits 0. Verification: `ssh lsj@<ip> 'sudo crictl images | grep nanofab-supervisor'` returns one row per node with the same image ID. Idempotency: a second `ctr import` on any node is a no-op (returns the existing image reference). Cleanup of `/tmp/nanofab-supervisor.tar` on each node: deferred to SOP's tear-down checklist.
- KR1.3: `helm install nanofab-supervisor-mvp deploy/charts/nanofab-supervisor --values deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml --namespace nanofab --create-namespace` (run from `repos/resink-ai/resink-core/`) exits 0 with `STATUS: deployed`. `kubectl get all -n nanofab` shows the expected object set (1 Deployment, 1 ReplicaSet, 1 Pod, 1 ServiceAccount, 1 ConfigMap, 0 Services, 1 Role, 1 RoleBinding). Pod reaches phase `Succeeded` within 2 minutes (`restartPolicy: Never`; supervisor binary exits 0 after one pass). `kubectl get events -n nanofab` captured for the exec summary.
- KR1.4: Output verification: `kubectl cp -n nanofab <pod-name>:/workspace/trace.jsonl /tmp/home-cluster-trace.jsonl` succeeds. EITHER `cmp /tmp/home-cluster-trace.jsonl <local-mvp-loop-trace>` produces no output (byte-equivalent) OR the divergence is documented in the SOP and exec summary (e.g., timestamp fields). The MVP supervisor's trace is line-delimited JSON; the divergence (if any) is human-readable.
- KR1.5: Delete path executed: `helm uninstall nanofab-supervisor-mvp -n nanofab` exits 0. Within 30 seconds, `kubectl get all -n nanofab` returns "No resources found in nanofab namespace." `kubectl delete namespace nanofab` exits 0. Final state: `kubectl get ns | grep nanofab` returns nothing.
- KR1.6: Chart README at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` gains a new "## Install (home cluster)" section between the existing "## Install (minikube MVP)" and "## Install (dry-run, no cluster needed)" sections. Names the prerequisites (kubeconfig with `kubernetes-admin@kubernetes` context, helm ≥ 3.x/4.x, docker, ssh to k8s-0..k8s-3 as `lsj` with hwmgmt key), the 4-step install (build → distribute → install → verify), the 2-step delete (uninstall → namespace delete), and cross-links the SRE deployment SOP at `teams/platform/sre/runbooks/nanofab-supervisor-deployment.md`. The existing "## Install (minikube MVP)" section is preserved verbatim with a top-line blockquote note: "**Legacy local-dev path.** The canonical deploy target is the home cluster (see § Install (home cluster) above); this section is retained for single-node smoke loops on a developer machine."
- KR1.7: Tenant-isolation invariant holds. Edits land under `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` and `teams/platform/devops/`; no `org-os/` writes. Final `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns only the canonical `acme.ai` placeholder.

**Tasks**

- [ ] Author `values/home-cluster-mvp.yaml` — verify with `helm lint --strict` + `helm template` + `helm install --dry-run`.
- [ ] Coordinate image hand-off from resink-core (their O1 KR2.3 produces `/tmp/nanofab-supervisor.tar`).
- [ ] Distribute image to k8s-0..k8s-3 via `scp` + `sudo ctr -n=k8s.io images import`; verify with `crictl images`.
- [ ] Run `helm install` against `kubernetes-admin@kubernetes` namespace `nanofab` (created via `--create-namespace`); wait for pod to reach `Succeeded`.
- [ ] `kubectl cp` the trace and verify against local mvp-loop trace (or document divergence).
- [ ] Run `helm uninstall`; confirm zero residual objects; `kubectl delete namespace nanofab`.
- [ ] Update chart README with "Install (home cluster)" section + legacy note on minikube section + cross-link to SRE SOP.
- [ ] Tenant-isolation dry-run.

## Risks

- **Image distribution friction:** 4-node per-node `ctr import` is slow (~3–5 min per node for a 100–200 MB tarball over the LAN). If a node fails to import (sudo misconfigured, disk full, etc.), the SOP names the diagnosis. Mitigation: the home-cluster baseline guarantees passwordless `sudo` for `lsj`; `k8s.io` containerd namespace is the kubeadm canonical.
- **Scheduler picks a node without the image:** If the kubelet's image cache is per-node and only one node has the image, the scheduler might pick a different node and ImagePullBackOff results. Mitigation: distribute to ALL 4 nodes (KR1.2 acceptance is all-4-succeed). The SOP names the `nodeSelector` workaround if ever needed.
- **Helm `--create-namespace` race:** Helm creates the namespace on first install. If the namespace already exists in a non-empty state (e.g., from a prior failed install), `helm install` may fail. Mitigation: include `kubectl get ns nanofab` in the pre-install check; if it exists, `helm list -n nanofab` confirms no prior release, OR run `kubectl delete namespace nanofab` first.
- **Trace divergence from local mvp-loop:** Containerized run might have different wall-clock fields than the local run. KR1.4 admits either byte-equivalence OR documented divergence; first-iteration is "captured + understood," not "byte-stable."
- **`workspaceMount.enabled: false` requires the Dockerfile to bake the workspace in.** The existing Dockerfile does this (`COPY --from=rust-builder /build/synthetic_tenants/closed_loop_v0/workspace /workspace`). If a future iteration moves to a PVC-backed workspace, this values file needs updating; flagged but not in scope this loop.

## Out of scope this loop

- **In-cluster image registry** (home-cluster roadmap Phase 1.X) — per-node `ctr import` is the bootstrap path. Carry to a later loop.
- **pyinfra `services/nanofab_supervisor/` wrapper** (home-cluster roadmap Phase 1.1) — manual SOP first; automation second iteration.
- **MetalLB LoadBalancer Service for the supervisor** — no HTTP port to expose; flip when `/healthz` lands.
- **`/healthz` + `/readyz` flip** — depends on resink-core's long-running supervisor; no carry this loop (forward-compat values pre-wired).
- **Minikube smoke** — structurally superseded by the home-cluster deploy; minikube section in chart README marked legacy.
- **Multi-tenant fanout, ResourceQuota/LimitRange, IRSA, CI/CD chart publishing** — out per long-standing chart posture.
- **Re-attempt of board-action 2026-06-06-002 minikube install** — closed `superseded` last loop; not re-opened.
{% endraw %}
