---
layout: default
title: CEO Brief — 2026-05-12-0645
date: 2026-05-12
status: active
type: okr
loop: 2026-05-12-0645
owner: board
grand_parent: Loops
parent: Loop 2026-05-12-0645
nav_order: 1
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-12
  status: active
  loop: 2026-05-12-0645
  links: parent: ""
-->
{% raw %}

# Resink.ai CEO Brief — 2026-05-12

## Context

Seven loops of structured org-os work plus the first AI-observability surface (GitBook publishing pipeline, PR #1 awaiting merge on `resink-ai/resink-ai.github.io`). Two MVP demos green-bar on developer laptops (`make mvp-loop` static + dlopen). One Helm chart authored to `helm lint --strict` + `helm template` + `helm install --dry-run` clean. **And zero workloads running anywhere a teammate could log in and look at.** ORG.md's claim of an "AI-native realtime data processing engine" presumes there is a thing to run — a binary executing on a cluster, observable to anyone with `kubectl`. Today: not yet.

**The gap.** Distance from vision is now **runnable infrastructure**. The supervisor binary builds, the chart renders, the verdict shape is byte-stable — and none of it has ever been deployed to a real Kubernetes cluster. The chart's minikube-smoke target has been carrying for 4 loops blocked on a developer-machine `brew install minikube`; at the same time, the resink-ai home cluster (4 nodes, kubeadm + Calico + MetalLB + local-path-provisioner, k8s 1.33) has been up and idle for 7+ days. **The user's framing for this loop names the pivot directly: the home cluster IS the deployment target, replacing minikube; deploy and delete `nanofab-supervisor` against it, and turn the procedure into a written SOP.**

**The shape of this loop.** A **first-real-workload loop** — code-bearing on the deployment pipeline + values overlay + SOP runbook authoring + image-distribution path. Three teams active (board light, devops + resink-core + sre primary); three paused with review-ack asks (DE, sim-farm, AE). No ADRs to ratify (none in `draft` entering the loop). No new ADRs proposed for ratification this loop. Hot-swap correctness test (ADR-2026-05-16-001 step 3) explicitly deferred to next loop — single-loop focus on deployment.

**Probe results entering the loop:**

- `kubectl config current-context` → `kubernetes-admin@kubernetes`. `kubectl get nodes` → 4 nodes Ready (`k8s-0` CP at `192.168.1.162`; `k8s-1`/`k8s-2`/`k8s-3` workers); k8s v1.33.0 server / v1.33.9 client; containerd runtime. **Cluster ready.**
- `kubectl get sc` → `local-path (default)`, `WaitForFirstConsumer`. MetalLB pool `192.168.1.200-192.168.1.250` live; Calico DaemonSet 4/4.
- `helm version` → v4.1.4. Chart already passes `helm lint --strict` + `helm template` + `helm install --dry-run`.
- `kubectl get pods -A` → only baseline workloads (calico, coredns, etcd, kube-*, metallb, local-path-provisioner). No application workloads. **The supervisor will be the cluster's first product workload.**
- `git ls-remote git@github.com:resink-ai/resink-core.git` → still `Repository not found`. Workspace promotion 6th-loop carry. Board-action `2026-06-06-001` `status: open`, `due: 2026-06-13` — not yet firing per today's date; deferred-not-due.
- `which minikube` → `not found`. **Now structurally irrelevant** — the home cluster is the deployment target, the chart's minikube section degrades to "legacy local-dev path."
- Open accepted requests with `deferred_to_loop: 2026-05-12-0645`: **none.** The one historical accepted-deferred request (`teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md`) is `status: fulfilled` since the last loop.
- Draft ADRs in `board/decisions/`: **none.** No ratifications this loop.
- Team `proposals/` directories: **empty** (no team-initiated proposals filed since the last brief).

**Re P1 from the 2026-06-13 retro (workspace-promotion ticket reframed as ready-to-execute command):** The proposal was to surface the exact `gh repo create` command in this brief's Out-of-scope when the action ticket is still `open` so the human can run it via `! prefix`. Doing that below. Forcing function is not yet firing (due date is in the future per today's calendar); the count of downstream blockers continues to grow each loop the carry persists.

**Re minikube smoke (4-loop carry):** Per the 2026-06-13 brief's working answer to retro P5, toolchain installs route through team Prerequisite docs not board-action tickets. The follow-on question — "what if the minikube install never lands because the toolchain is the wrong shape entirely?" — is answered this loop: **the home cluster is the right shape.** Minikube was a stand-in for a real cluster; the real cluster is now available. The chart's minikube-mvp values become a legacy path; the new canonical path is the home cluster.

**CEO decisions for this loop:**

- **Activate three teams + board (light).** DevOps + resink-core + SRE active. board owns brief + retro + ratification-step (none this loop). Sim-farm + DE + AE paused; one-paragraph review-ack each (sim-farm reviews the home-cluster deployment for verdict-contract surface impact; DE reviews the supervisor's runtime config for any schema-JSON consumer surface that surfaces; AE reviews the deployed binary for ABI/plugin contract impact under containerized execution).

- **Headline deliverable: deploy `nanofab-supervisor` to the home cluster, then delete it.** End-to-end happy path: build supervisor image → distribute to nodes → `helm install` against `kubernetes-admin@kubernetes` → pod reaches `Succeeded` → `kubectl cp` the trace out and verify byte-equivalence against the local `make mvp-loop` trace → `helm uninstall` → confirm zero residual objects in the namespace → tear down. The deployment is owned by DevOps; the image is owned by resink-core.

- **Companion deliverable: deployment SOP.** SRE authors a new runbook documenting the procedure end-to-end with prerequisites, install / upgrade / delete steps, verification, and failure-mode subsections. Sister to the existing `nanofab-supervisor-failed-validation.md` runbook; type: `runbook`. Lives at `teams/platform/sre/runbooks/nanofab-supervisor-deployment.md`. Cross-link from chart README's new "Install (home cluster)" section.

- **No image-registry setup this loop.** First deployment uses a per-node `ctr -n=k8s.io images import` distribution path (build image locally, `docker save`, `scp` to each node, import on each). Documented in the SOP. A proper in-cluster registry (option B in the home-cluster roadmap's Phase 1) is OUT of scope this loop — sized as a follow-up when image-deploy friction warrants it. The chart's `image.pullPolicy: IfNotPresent` makes per-node pre-load work cleanly.

- **No pyinfra wrapper this loop.** The home-cluster roadmap's Phase 1.1 (`services/_lib/` shared helm-runner + `services/nanofab_supervisor/` install/upgrade/delete deploys) is OUT of scope. First we prove the manual SOP works end-to-end; the wrapper is the second iteration once the procedure stabilizes. Sized at next loop or beyond.

- **No real workload-config this loop.** The deploy uses the existing `values/minikube-mvp.yaml` shape adapted to the home cluster (new file `values/home-cluster-mvp.yaml`): `tenant: mvp-tenant`, `fleetColor: blue`, in-memory KV, empty Kafka. Sim-mode supervisor runs once against the baked-in workspace, exits 0, leaves a trace. This is a deployment-mechanism demonstration, not a multi-tenant or production rollout.

- **Defer hot-swap correctness test (ADR-2026-05-16-001 step 3).** Step 3 was scheduled "loop+1" relative to step 2's closure (last loop). Deployment focus crowds it out this loop. New schedule: **loop+2** relative to step 2 (= the loop after this one). Slippage recorded in this loop's exec summary per ADR-2026-05-30-002's loop+N convention.

- **Sim-farm verdict-contract back-reference to DE convention** — opportunistic XS this loop if the sim-farm team has bandwidth during their paused-review ack. Mechanical one-line edit. Not a KR; carries again to next loop if it doesn't land.

**Standing CEO answers (closing carryover loops):**

- **Home cluster as deployment target — canonical:** The home cluster (4-node kubeadm cluster at `192.168.1.162` CP) is the canonical deployment target for pre-prod resink.ai workloads going forward. EKS / cloud deployment is a later sub-slice (sub-project #4 § serving-deployment spec). Minikube remains a legacy local-dev path documented in the chart README but is not the load-bearing deploy target — the home cluster is. The chart's `values/home-cluster-mvp.yaml` is the new canonical values file; `values/minikube-mvp.yaml` and `values/minikube-bluegreen.yaml` are preserved as legacy.

- **Image-distribution path (manual, first iteration):** Build the supervisor image with `docker build -f deploy/charts/nanofab-supervisor/Dockerfile -t nanofab-supervisor:0.1.0 .` from `repos/resink-ai/resink-core/`. Distribute to nodes via `docker save -o /tmp/nanofab-supervisor.tar nanofab-supervisor:0.1.0` then `scp` + `sudo ctr -n=k8s.io images import /tmp/nanofab-supervisor.tar` on each of the 4 nodes (k8s-0..k8s-3, ssh as `lsj` with key `~/.ssh/id_ed25519_hwmgmt`). Set `image.pullPolicy: IfNotPresent` so the kubelet uses the imported image. Documented in the SOP. **Not load-bearing forever** — Phase 1 of the home-cluster roadmap calls for a local registry; this is the bootstrap path.

- **Storage for the supervisor workspace:** `local-path-provisioner` with `WaitForFirstConsumer`. PVC binds to whichever node the pod schedules on. The MVP supervisor exits after one pass; PVC's lifetime is the release's; `helm uninstall` removes the PVC unless `persistence.keep: true` is set (not set this loop). For the first deployment, the simpler shape is: bake the workspace into the image at build time (Dockerfile already does this via `COPY --from=rust-builder /build/synthetic_tenants/closed_loop_v0/workspace /workspace`) and set `workspaceMount.enabled: false`. PVC-backed workspace is sized at a later loop when state needs to outlive a single pod.

- **Tenant labels and observability:** Every chart object already carries the canonical label set (`tenant=mvp-tenant`, `app.kubernetes.io/instance=...`, `nanofab.resink.ai/dag-version=v1`, `nanofab.resink.ai/fleet-color=blue` etc.) per the existing `_helpers.tpl::tenantLabels` template. `kubectl logs -l tenant=mvp-tenant` works out of the box; no Grafana or Prometheus this loop (that's home-cluster roadmap Phase 1.3, not in scope).

- **Service Type:** No `Service` rendered this loop. `healthz.enabled: false` (the MVP supervisor exposes no HTTP port); when resink-core ships the long-running supervisor with `/healthz` + `/readyz`, devops flips the flag and a ClusterIP Service renders. MetalLB LoadBalancer comes when we have a real client. Not this loop.

- **Delete semantics:** `helm uninstall nanofab-supervisor-mvp -n nanofab` removes Deployment, ServiceAccount, ConfigMap(s), RBAC, and (when rendered) Service. The release's PVC is deleted with the release in the current `Recreate` strategy / no-explicit-finalizer shape. **A successful delete is part of the SOP's acceptance criteria** — the user's framing names "deploy and delete" together; the SOP confirms both directions.

**Carryover load by team** (per ADR-2026-05-09-005):

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| board | brief + retro + ratification-step (none); workspace-promotion forcing-function status (deferred-not-due); GitBook PR #1 merge awaits human; multi-loop plan absorption for ADR-2026-05-16-001 step 3 slip-by-one | S | Light loop for board; brief + retro only |
| devops | Home-cluster deployment first execution (build / distribute / install / verify / delete); new `values/home-cluster-mvp.yaml`; chart README new "Install (home cluster)" section; minikube smoke now superseded structurally (no longer a carry); GitBook scope responses already absorbed | L | Headline deliverable; first product workload on the cluster |
| resink-core | Supervisor image build + per-node distribution; `docs/user-guide.md` § new "Deploying to the home cluster" subsection; workspace promotion (opportunistic only, 6-loop carry, board-action `2026-06-06-001` still open) | M | Image build + distribution; opportunistic carry on workspace promotion |
| sre | Deployment SOP authoring (new `runbooks/nanofab-supervisor-deployment.md`); home-cluster `docs/k8s-cluster.md` § "Smoke-test workloads" small edit to add the supervisor as first product workload; companion failure-mode subsections (image-pull-back-off, RBAC errors, scheduler failures, PVC binding) | M | New runbook + small home-cluster docs edit |
| sim-farm | Paused; review-ack only (review the home-cluster deployment for verdict-contract surface impact); **opportunistic XS** verdict-contract back-reference to DE convention (long-standing carry) | XS | Paused; review-ack + optional 1-line back-reference |
| DE | Paused; review-ack only (review the supervisor's runtime config for any schema-JSON consumer surface that surfaces during deployment); duckdb.md migration to `type: convention` continues to carry | XS | Paused; review-ack only |
| AE | Paused; review-ack only (review the deployed binary for ABI/plugin contract impact under containerized execution — same ABI under dlopen-plugins-off as under dlopen-plugins-on) | XS | Paused; review-ack only |

## Objectives

### O1: DevOps deploys `nanofab-supervisor` to the home cluster end-to-end

source: ceo-brief

Why it matters: The chart has been authored, dry-run-validated, and minikube-smoke-deferred for 4 loops. The 4-loop deferral pattern resolves structurally this loop by replacing minikube with the real home cluster as the deployment target. Closing this objective is the company's first product workload on real Kubernetes infrastructure — observable to anyone with `kubectl`, deletable with `helm uninstall`, and reproducible by a second IC with kubeconfig access. It is the foundation for every later deployment exercise (sub-project #4 § serving-deployment), every per-tenant install in Phase 2 of the home-cluster roadmap, and every observability + SLO authoring step in SRE's backlog.

**Key results**
- KR1.1: New file at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml`. Sister to `values/minikube-mvp.yaml`. Sets `tenant: mvp-tenant`, `fleetColor: blue`, `replicas: 1`, `kvEndpoint: in-memory`, `kafkaBootstrap: ""`, `coordinatorEndpoint: ""`, `nodePluginManifestUri: ""`. Sets `workspaceMount.enabled: false` (workspace baked into image). Sets `image.repository: nanofab-supervisor`, `image.tag: 0.1.0`, `image.pullPolicy: IfNotPresent` (image is pre-loaded on each node via `ctr import`). Sets `pod.restartPolicy: Never` (MVP supervisor exits 0). Sets `healthz.enabled: false` (no HTTP port). Comments inline name the home-cluster context (4-node kubeadm, local-path-provisioner, MetalLB available but no LB needed this loop).
- KR1.2: Image distribution executed: `cd repos/resink-ai/resink-core && docker build -f deploy/charts/nanofab-supervisor/Dockerfile -t nanofab-supervisor:0.1.0 .` (exits 0); `docker save nanofab-supervisor:0.1.0 -o /tmp/nanofab-supervisor.tar` (one tarball); for each node `k8s-0..k8s-3`: `scp -i ~/.ssh/id_ed25519_hwmgmt /tmp/nanofab-supervisor.tar lsj@<ip>:/tmp/` then `ssh -i ~/.ssh/id_ed25519_hwmgmt lsj@<ip> 'sudo ctr -n=k8s.io images import /tmp/nanofab-supervisor.tar'` (all 4 exits 0). Verify via `ssh ... 'sudo crictl images | grep nanofab-supervisor'` returning one row per node. **Idempotent:** re-importing on a node that already has the image is a no-op.
- KR1.3: `helm install nanofab-supervisor-mvp deploy/charts/nanofab-supervisor --values deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml --namespace nanofab --create-namespace` (run from `repos/resink-ai/resink-core/`) exits 0 with `STATUS: deployed`. `kubectl get all -n nanofab` shows: 1 Deployment (1/1 ready), 1 ReplicaSet, 1 Pod (Status `Succeeded` after the supervisor runs and exits — `restartPolicy: Never`), 1 ServiceAccount, 1 ConfigMap (operator-visibility env mirror), 0 Services (healthz disabled), 1 Role + 1 RoleBinding. **Pod reaches `Succeeded` within 2 minutes** (image is small, supervisor is fast on the baked-in workspace).
- KR1.4: Output verification: `kubectl cp -n nanofab <pod>:/workspace/trace.jsonl /tmp/home-cluster-trace.jsonl` (run while pod is `Succeeded`) succeeds; `cmp /tmp/home-cluster-trace.jsonl <local-mvp-loop-trace>` produces no output (byte-equivalent). **Optionally** also `kubectl cp -n nanofab <pod>:/workspace/dim_user_output.parquet /tmp/home-cluster-dim_user.parquet` and verify parquet shape matches the local fixture. **Either** byte-equivalence holds **or** the SOP documents the expected divergence (e.g., timestamp fields if any) — first iteration is "captured + documented," not "byte-stable across runs."
- KR1.5: Delete path executed: `helm uninstall nanofab-supervisor-mvp -n nanofab` exits 0 with `release "nanofab-supervisor-mvp" uninstalled`. Within 30 seconds, `kubectl get all -n nanofab` returns "No resources found in nanofab namespace." Pod is gone; ReplicaSet is gone; Deployment is gone; ServiceAccount is gone; ConfigMap is gone; RBAC objects are gone; namespace remains (Helm doesn't delete the namespace). **`kubectl delete namespace nanofab`** as a final clean-up step is part of the SOP.
- KR1.6: Chart README gains a new "Install (home cluster)" section after "Install (minikube MVP)". Names the prerequisites (kubeconfig at `~/.kube/<config>` or env `KUBECONFIG`, helm ≥ 3.x or 4.x, `docker` for image build, ssh access to k8s-0..k8s-3 as `lsj` with the hwmgmt key), the four-step install (build → distribute → install → verify), and the two-step delete (uninstall → namespace delete). Cross-links the SRE SOP at `teams/platform/sre/runbooks/nanofab-supervisor-deployment.md`. Notes that the minikube section is preserved as a legacy local-dev path.
- KR1.7: Tenant-isolation invariant holds; no `org-os/` edits this objective. All edits land under `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` and `teams/platform/devops/`.

**Tasks**
- [ ] Author `values/home-cluster-mvp.yaml` — owner: teams/platform/devops.
- [ ] Build supervisor image; verify image size + layer count is reasonable — owner: teams/application/resink-core (delegated; see O2).
- [ ] Distribute image to 4 nodes via `docker save | scp | sudo ctr -n=k8s.io images import` — owner: teams/platform/devops.
- [ ] Run `helm install` against `kubernetes-admin@kubernetes`; verify pod reaches `Succeeded`; capture event log via `kubectl get events -n nanofab` — owner: teams/platform/devops.
- [ ] `kubectl cp` the trace; verify against local mvp-loop trace OR document expected divergence — owner: teams/platform/devops.
- [ ] Run `helm uninstall`; verify zero residual objects in the namespace; `kubectl delete namespace nanofab` — owner: teams/platform/devops.
- [ ] Update chart README with "Install (home cluster)" section + cross-link to SRE SOP — owner: teams/platform/devops.
- [ ] Tenant-isolation dry-run — owner: teams/platform/devops.

### O2: resink-core builds + distributes the supervisor image and documents the deploy path in `docs/user-guide.md`

source: ceo-brief

Why it matters: The Dockerfile lives in the devops-owned chart directory but the binary is owned by resink-core. Per the existing R1 hand-off response, DevOps owns the Dockerfile; resink-core owns the binary build. This loop is the first time the Dockerfile actually compiles a production image and ships it to a target cluster. resink-core owns the image build + the user-guide documentation of the deploy path so future developers can reproduce. DevOps owns the chart-side install and SOP authoring; resink-core owns the image-side and the per-developer reproducibility documentation.

**Key results**
- KR2.1: `docker build -f deploy/charts/nanofab-supervisor/Dockerfile -t nanofab-supervisor:0.1.0 .` (run from `repos/resink-ai/resink-core/`) exits 0. The image size is reasonable (multi-stage build with debian:12-slim runtime; expect < 200 MB). Inspect with `docker inspect nanofab-supervisor:0.1.0 --format='{{.Size}}'` and record the size in the exec summary. **The Dockerfile's COPY of `synthetic_tenants/closed_loop_v0/workspace` is intentional** — bakes the MVP workspace into the image so `workspaceMount.enabled: false` works without external volumes.
- KR2.2: Image tag aligned with chart's `Chart.appVersion: "0.1.0"`. If the supervisor binary's crate version differs from `0.1.0`, either bump `Chart.appVersion` to match (preferred — single source of truth on the crate) or tag the image to match the crate version. **Record the choice and the resulting tag in the exec summary** so future loops know the convention.
- KR2.3: `docker save nanofab-supervisor:0.1.0 -o /tmp/nanofab-supervisor.tar`; record tarball size (expect 100-200 MB). Hand off the tarball to DevOps for distribution (DevOps owns the `scp` + `ctr import` steps per O1 KR1.2).
- KR2.4: `repos/resink-ai/resink-core/docs/user-guide.md` gains a new "## Deploying to the home cluster" section (~30 lines). Names the build command, the save command, the per-node distribute command, the helm install command, the verification + delete commands. Cross-links the SRE deployment SOP. Notes the once-per-build flow vs the multi-pod stream-deployment shape (post-dlopen, post-long-running-supervisor, post-registry — all out of scope this loop).
- KR2.5: `repos/resink-ai/resink-core/docs/architecture.md` § "Named deviations" gains a one-line entry under ABI Option A's "Step 3 (hot-swap correctness test)" updating the schedule: **was loop+1 (= the loop just-completed-relative-to-step-2), now loop+2 (= the loop after this one)**, with the slippage reason "deployment focus crowds out hot-swap test this loop." This is the loop+N convention's slippage record per ADR-2026-05-30-002.
- KR2.6: Tenant-isolation invariant holds; no `org-os/` edits this objective.
- KR2.7: Workspace promotion **opportunistic only** — if the human creates the `resink-ai/resink-core` remote in-loop (via `! gh repo create resink-ai/resink-core --private --description "Resink.ai nanofab runtime, training pipeline, sim-farm engine, and orchestrator"`), resink-core picks up the submodule promotion as an in-loop sub-task. No KR; 6-loop carry continues if not.

**Tasks**
- [ ] Build the image; record size + tag — owner: teams/application/resink-core.
- [ ] Save the image to a tarball; hand off path to DevOps — owner: teams/application/resink-core.
- [ ] Update `docs/user-guide.md` with the home-cluster deploy section — owner: teams/application/resink-core.
- [ ] Update `docs/architecture.md` Named deviations entry for ADR-2026-05-16-001 step 3 schedule slip — owner: teams/application/resink-core.
- [ ] Tenant-isolation dry-run — owner: teams/application/resink-core.
- [ ] (Opportunistic) Workspace promotion if the GitHub remote is created in-loop — owner: teams/application/resink-core.

### O3: SRE authors the deployment SOP runbook and updates the home-cluster operator runbook

source: ceo-brief

Why it matters: The deployment procedure exercised by DevOps in O1 needs a written runbook so any operator can reproduce it without re-deriving the steps from this brief. Sister to the existing failed-validation runbook (which SRE shipped at 2026-05-23). This is SRE's second owned-surface artifact and the first that covers a happy-path procedure rather than a failure-mode tree. It also establishes the home-cluster operator runbook (`repos/resink-ai/home-cluster/docs/k8s-cluster.md`) as the discoverable index of cluster workloads — currently it documents bootstrap + smoke-test only; this loop adds nanofab-supervisor as the first product workload.

**Key results**
- KR3.1: New file at `teams/platform/sre/runbooks/nanofab-supervisor-deployment.md` with `type: runbook`, `severity_tiers: [S1, S2, S3]`, `status: active`. Sections: (a) Purpose and scope (one paragraph: deploys nanofab-supervisor MVP to the home cluster); (b) Prerequisites (kubeconfig at the operator's machine with the `kubernetes-admin@kubernetes` context active; `helm` ≥ 3.x or 4.x; `docker` for image build; ssh access to k8s-0..k8s-3 as `lsj` with the hwmgmt key at `~/.ssh/id_ed25519_hwmgmt`); (c) Install procedure (the 6-step recipe: build → save → distribute → install → verify → done); (d) Delete procedure (the 2-step recipe: helm uninstall → namespace delete); (e) Verification (kubectl get all + kubectl logs + kubectl cp trace + cmp); (f) Failure modes — `ImagePullBackOff` (image not loaded on the scheduled node), `ErrImagePull` (typo in image name/tag), `CreateContainerConfigError` (ConfigMap missing or RBAC missing), `Pending` for > 5 min (PVC binding failure if hostPath/PVC misconfigured; or no schedulable node), `CrashLoopBackOff` (supervisor binary crashes — but `restartPolicy: Never` makes this surface as a single failed-pod log, see the existing failed-validation runbook for the diagnosis tree), `RBAC denied` on the supervisor's ServiceAccount; (g) Cross-team notes (cross-link to chart README, SRE failed-validation runbook, home-cluster k8s-cluster docs); (h) `Verified-against-environment` per ADR-2026-05-16-003 — naming `kubectl` v1.33.x, `helm` v4.x, `docker` ≥ 20.10, `ssh` with the hwmgmt key, macOS or Linux operator, 4-node home cluster at `192.168.1.0/24`, `kubernetes-admin@kubernetes` context.
- KR3.2: `repos/resink-ai/home-cluster/docs/k8s-cluster.md` § "Smoke-test workloads" gains a new subsection "### nanofab-supervisor (first product workload)" with a one-paragraph description, an `ls deploy/charts/nanofab-supervisor/` pointer, the canonical `helm install ...` invocation, and a cross-link to the SRE SOP. **Small edit** (~15 lines); no restructuring of the existing doc.
- KR3.3: Cross-link both ways: chart README "Install (home cluster)" section cross-links the SOP (handled by DevOps in O1 KR1.6); the SOP cross-links the chart README + the home-cluster doc. The home-cluster doc cross-links the SOP. No circular links — the home-cluster doc and SOP each link the other as a pointer, not as a dependency.
- KR3.4: Tenant-isolation invariant holds; no `org-os/` edits this objective. All edits land under `teams/platform/sre/runbooks/` and `repos/resink-ai/home-cluster/docs/`.
- KR3.5: Refresh `teams/platform/sre/status.md` from the 2026-06-06 baseline to reflect: this loop's runbook + home-cluster docs edit; the supervisor swap delivered last loop closed cleanly under both feature flags (review-ack landed at 2026-05-11-2153 exec summary, but the status.md was not refreshed past 2026-06-06 — bring it forward); `BLOCKED` observable verification remains forward-pointed.

**Tasks**
- [ ] Author `runbooks/nanofab-supervisor-deployment.md` per the standing-answer shape — owner: teams/platform/sre.
- [ ] Update `repos/resink-ai/home-cluster/docs/k8s-cluster.md` § "Smoke-test workloads" with the supervisor subsection — owner: teams/platform/sre.
- [ ] Cross-link the SOP, chart README, and home-cluster doc (coordinate with DevOps's O1 KR1.6) — owner: teams/platform/sre.
- [ ] Refresh `teams/platform/sre/status.md` — owner: teams/platform/sre.
- [ ] Tenant-isolation dry-run — owner: teams/platform/sre.

## Risks

- **Image build could fail at `cargo build --release -p nanofab-supervisor`** inside the Dockerfile due to glibc/openssl mismatches or missing system dependencies. Mitigation: the multi-stage build uses `rust:1.83-slim-bookworm` (matches the MSRV declared in the workspace `Cargo.toml`) and installs `pkg-config + libssl-dev + ca-certificates` in stage 1. If it fails, try `rust:1.83-bookworm` (full) or pin a specific patch version. Document the working tag in the exec summary.
- **Image distribution could fail at `ctr -n=k8s.io images import`** if the namespace flag is wrong (kubeadm uses `k8s.io`, not the default) or if `sudo` is misconfigured on the node. Mitigation: passwordless `sudo` is documented in the home-cluster repo as a baseline guarantee; `k8s.io` is the canonical containerd namespace for kubeadm clusters. If a node fails to import, the SOP names the diagnosis (`sudo ctr namespaces ls`, `sudo crictl images`).
- **Pod fails to schedule** if image is missing on the only candidate node (no `imagePullSecrets`, no `pullPolicy: Always` retry). Mitigation: import on ALL 4 nodes, not just one — the scheduler may pick any. If only one node has the image and the scheduler picks a different one, the pod sits in `ImagePullBackOff` until the image is imported on the actual scheduled node OR a `nodeSelector` constrains scheduling. SOP documents both diagnoses; first-iteration distributes to all nodes by default.
- **Pod completes but trace divergence from local mvp-loop** if the containerized run sees different timestamps, different ordering, different floating-point representations, or different parquet writer settings than the local run. Mitigation: O1 KR1.4 admits either byte-equivalence OR documented divergence. The MVP supervisor's trace is line-delimited JSON; the diff is human-readable. If divergence is expected (e.g., wall-clock fields), the SOP documents which fields to ignore.
- **Helm uninstall could leave residual objects** if a Helm hook or a finalizer is added to the chart in a future iteration. Mitigation: current chart has no hooks and no finalizers; uninstall is mechanical. Verify `kubectl get all -n nanofab` returns empty after uninstall (KR1.5 acceptance criterion).
- **Workspace promotion 6th-loop carry surfaces escalation pressure on the retro.** Per the 2026-06-13 retro's P1, this brief generates the exact `! gh repo create` command in Out-of-scope. If the human runs it in-loop, resink-core picks up the promotion as a sub-task in O2 KR2.7 (opportunistic). If not, the carry continues and the next retro should evaluate whether the forcing-function mechanism needs a different shape (e.g., a deeper escalation to "stop opening new loops that depend on resink-core paths until this is resolved" — drastic, but it shows the cost). Mitigation: this loop names the command and the count of downstream blockers; escalation is the retro's call if the carry persists.
- **The SOP could prove insufficient on second iteration** — first-iteration SOPs always do. Mitigation: first execution is by the SOP author (DevOps), not by a third-party operator; the friction surfaces during authoring, not after. Iteration is expected at next loop's retro if reading-experience feedback warrants it. The SOP is "correctness," not "polish."
- **`local-path-provisioner` with `WaitForFirstConsumer`** delays PVC binding until pod scheduling. Not in scope this loop (workspace is baked into the image, no PVC), but flagged for the SOP's "Pending for > 5 min" diagnosis: if a future deploy uses a PVC, the binding waits for the scheduler. SOP names this as a known-shape, not a bug.
- **No image registry means no rollback to a prior image without re-importing.** First iteration accepts this; the SOP names the limitation. Phase 1 of the home-cluster roadmap adds a local registry; until then, rollback = "import the prior tarball again on each node." Mild risk; mitigation is documentation, not a chart change.

## Out of scope this loop

- **Workspace promotion (`resink-ai/resink-core` GitHub remote creation)** — 6th-consecutive-loop defer. **Per board-action 2026-06-06-001's forcing function (not yet firing per today's date; due 2026-06-13): this section names the count of downstream blockers explicitly, as the 2026-06-13 retro's P1 proposed.** Downstream blockers known at 2026-05-12:
  1. Workspace promotion itself (resink-core, original carry).
  2. ADR-2026-05-16-001 step 2 commit graph: lands last loop in-tree only; future bisect/blame across the runtime separation gains friction.
  3. ADR-2026-05-16-001 step 3 (hot-swap correctness test, target loop+2 = the loop after this one): same friction multiplied.
  4. The HTML capabilities report at `board/reports/2026-05-30-resink-core-capabilities.html` cites in-tree paths; once promoted, paths become submodule-relative and need a one-paragraph update.
  5. The GitBook publishing pipeline's resink-core docs tree integration: lives in the in-tree path `repos/resink-ai/resink-core/docs/` this loop; once promoted, the publishing script needs a one-line submodule-walk update.
  6. **NEW this loop:** The deployment SOP's image-build step cites the in-tree path `cd repos/resink-ai/resink-core && docker build ...`; once promoted, the path becomes submodule-relative and the SOP's prerequisites section needs a one-line update.
  Total: **6 named downstream blockers**, increasing by 1 each loop (this loop adds #6, the SOP path). **Ready-to-execute command (per P1 reframing):** If unblocking this loop, run `! gh repo create resink-ai/resink-core --private --description "Resink.ai nanofab runtime, training pipeline, sim-farm engine, and orchestrator"` and then notify in chat to re-trigger the resink-core promotion attempt as an opportunistic sub-task under O2.
- **Hot-swap correctness test** — ADR-2026-05-16-001 step 3, was target loop+1, **now target loop+2** (= the loop after this one). Joint AE + resink-core deliverable. Deployment focus crowds it out this loop. Slippage recorded per ADR-2026-05-30-002's loop+N convention; ADR body grandfathers under absolute-date authoring.
- **Image registry on the home cluster** — Phase 1 of the home-cluster roadmap (likely a `services/registry/` Helm install). Out of scope this loop; per-node `ctr import` is the bootstrap path. Sized at next loop or whenever image-deploy friction warrants.
- **pyinfra wrapper at `services/nanofab_supervisor/`** — Phase 1.1 of the home-cluster roadmap. Out of scope this loop; the first SOP is manual to surface friction before automating. Sized at the loop after the SOP is exercised by a second operator.
- **DevOps minikube smoke full execution** — **structurally superseded** by the home-cluster deployment this loop. The chart's minikube-mvp values file is preserved as a legacy local-dev path; the canonical deploy target is the home cluster going forward. No new board-action ticket; the 4-loop carry is now closed by reframing, not by acting. Chart README's "Install (minikube MVP)" section stays in place for any developer who wants a local single-node smoke; not load-bearing for the company's deployment story.
- **Sim Farm Modes B + C** — out of scope per long-standing posture; trigger is post-multi-dim, post-dlopen, post-deployment.
- **Sim-farm own product repo decision** — trigger condition not met (first non-Python sim-farm component).
- **Three-layer verdict** — out of scope until training-pipeline gate stage 3 adopts coverage-spec-driven verdicts.
- **Future codegen patterns** (`scd1_first_event`, `window_stats_with_decrement`, etc.) — out of scope; AE paused this loop.
- **GitBook publishing PR #1 merge + post-merge verification** — awaits user merge on `resink-ai/resink-ai.github.io`. Once merged, the next loop's brief or retro picks up the rendered-site verification per the 2026-06-13 retro's P2. Not in scope this loop's authorship; happens at whatever loop the merge occurs.
- **GitBook reading-experience review** (2026-06-13 retro P3) — deferred to the first loop AFTER PR #1 merges. Not in scope this loop.
- **GitBook publishing of `org-os/` engine docs** (2026-06-13 retro P5) — deferred per the 2026-06-13 retro. Requires PR #1 merge + reading-experience review to land first.
- **Resink-core long-running supervisor with `/healthz` + `/readyz`** — out of scope this loop; ADR-2026-05-16-001 doesn't schedule it. Surfaces when post-dlopen step 3 + a real Kafka-driven workload exists.
- **DE schema-JSON v2 (additive fields)** — out of scope per DE's status; minimal shape is intentional, future cross-team request triggers v2.
- **DE duckdb.md migration from `type: rfc` to `type: convention`** — out of scope this loop (low priority bookkeeping; the `convention` type is admitted since 2026-05-30 but the migration is not load-bearing).
- **SRE BLOCKED observable verification** — multi-loop wait per SRE status; depends on resink-core surfacing a structured BLOCKED state from the runtime coordinator.
- **SRE `dlopen-plugins`-specific failure-mode subsections** — depends on the dlopen path being exercised in CI; the home-cluster deployment exercises `dlopen-plugins-off` (default `static-plugins`) this loop, so the dlopen failure modes are not surfaced. Carries to the loop that runs dlopen-on in CI.
- **SRE on-call rotation policy, alert paging wiring, per-supervisor-pod SLOs** — all deferred-not-dropped per SRE status; depend on Phase 1.3 of the home-cluster roadmap (observability stack).
- **Frontmatter-lint CI script** — sixth consecutive defer per the 2026-06-13 brief § Out of scope. Manual ADR-2026-05-08-001 enforcement continues.
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing housekeeping.
- **Multi-tenant install automation** (sub-project #4 § serving-deployment) — out of scope; first single-tenant install validates the deployment shape.
- **`ResourceQuota` / `LimitRange` per-tenant bounds** — out of scope per chart's existing posture; pod-level requests/limits suffice for MVP.
- **CI/CD chart publishing to an OCI registry** — out of scope; first install is from the local chart directory.

## Ratifications this loop

**None this loop.** No ADRs entered the loop in `status: draft`; no draft ADRs are proposed for ratification. No contracts are migrating types this loop. No `org-os/` convention edits this loop. The loop is code-bearing on product surfaces (chart values, image, SOP, docs) with zero `org-os/` writes by design. Tenant-isolation invariant therefore holds trivially across the loop on the board's side (no `org-os/` edits to dry-run).

The 2026-06-13 retro's classified evolution proposals:
- **P1** (workspace-promotion `! prefix` reframing) — **picked up this loop** in Out-of-scope; ready-to-execute command surfaced per the proposal.
- **P2** (GitBook post-merge verification) — picked up the loop PR #1 merges (not this loop).
- **P3** (GitBook reading-experience review) — picked up the loop after PR #1 merges (not this loop).
- **P4** (2-loop FFI handoff playbook extension) — deferred to a future loop alongside other org-os authoring (not this loop).
- **P5** (org-os engine-docs publishing track) — deferred to loop 2026-06-27 or later per the retro's own timing.

## Multi-loop plan slippage absorbed (not new ratifications)

- **ADR-2026-05-16-001 (Option A → dlopen restoration):**
  - Step 1 (AE template extension): ✅ closed loop-2 (= 2026-05-11-1631).
  - Step 2 (resink-core supervisor swap full execution): ✅ closed loop-1 (= 2026-05-11-2153).
  - Step 3 (hot-swap correctness test): was `loop+1`, **now `loop+2`** (= the loop after this one). Slippage reason: deployment focus crowds out hot-swap this loop. Joint AE + resink-core deliverable; AE awaits scheduling signal in the next CEO brief.
{% endraw %}
