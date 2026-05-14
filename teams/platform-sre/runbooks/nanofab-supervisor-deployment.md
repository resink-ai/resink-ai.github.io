---
layout: default
title: "platform-sre runbook: nanofab-supervisor-deployment"
date: 2026-05-12
status: active
type: runbook
owner: teams/platform/sre
grand_parent: Teams
parent: "Team: platform-sre"
---

<!-- original-frontmatter:
  type: runbook
  owner: teams/platform/sre
  date: 2026-05-12
  status: active
  severity_tiers: [S1, S2, S3]
-->
{% raw %}

# nanofab-supervisor deployment runbook (home cluster)

Sister to [`nanofab-supervisor-failed-validation.md`](nanofab-supervisor-failed-validation.md) — that
runbook covers the supervisor's runtime failure modes (verdict mismatches,
manifest validation, cargo-build gate). This runbook covers the
deployment+lifecycle surface: install / verify / delete against the
4-node home cluster.

First exercised at loop 2026-05-12-0645 with the chart-author (DevOps) as
the first operator. Second-operator validation is a future loop's call.

## Purpose and scope

Deploys the `nanofab-supervisor` MVP to the home cluster. Covers:

- **Install** — image build, image distribution to all 4 nodes, `helm install`,
  expected pod lifecycle.
- **Verify** — `kubectl logs` (the canonical verification surface; the
  one-shot supervisor under Deployment kind makes `kubectl cp` unreliable).
- **Delete** — `helm uninstall`, namespace cleanup, residual-object check.
- **Failure modes** — six common deployment-time failures, each with
  Symptom / Diagnosis / Mitigation / Severity.

Out of scope: runtime failure modes (covered by the sister failed-validation
runbook); minikube installs (legacy local-dev path).

## Prerequisites

| Item | Verification command | Source of truth |
|---|---|---|
| kubeconfig with `kubernetes-admin@kubernetes` context active | `kubectl config current-context` → `kubernetes-admin@kubernetes` | Home-cluster operator setup; see `repos/resink-ai/home-cluster/docs/k8s-cluster.md` § "Set up kubectl on your laptop" |
| 4 nodes Ready | `kubectl get nodes` returns 4 Ready (k8s-0..k8s-3) | Home-cluster baseline |
| `helm` ≥ 3.x | `helm version` ≥ v3.0 (verified on v4.1.4) | Operator-installed |
| `docker` with buildx | `docker buildx ls` lists `linux/amd64` | Operator-installed (Docker Desktop on macOS, native Docker on Linux) |
| ssh access to all 4 nodes as `lsj` | `ssh -i ~/.ssh/id_ed25519_hwmgmt lsj@192.168.1.162 'hostname'` returns `k8s-0` | Home-cluster baseline; key seeded by `install_01_users_ssh` |
| Passwordless `sudo` on all 4 nodes | `ssh ... 'sudo -n which ctr'` returns `/usr/bin/ctr` | Home-cluster baseline |

## Install procedure

### Step 1: Build the image (cross-arch from macOS/arm64)

From the resink-core repo root:

```bash
cd repos/resink-ai/resink-core
docker buildx build --platform linux/amd64 --load \
  -f deploy/charts/nanofab-supervisor/Dockerfile \
  -t nanofab-supervisor:0.3.0 .
```

The tag aligns with the supervisor crate version
(`crates/nanofab-supervisor/Cargo.toml` `[package].version`). On a Linux/amd64
host, `docker build` (no buildx) also works.

**Expected outcome:** `docker images | grep nanofab-supervisor` shows
`nanofab-supervisor:0.3.0` with size ~150 MB. The architecture is
`amd64/linux` (verify via `docker inspect nanofab-supervisor:0.3.0 --format='{{.Architecture}}/{{.Os}}'`).

### Step 2: Save the image to a tarball

```bash
docker save nanofab-supervisor:0.3.0 -o /tmp/nanofab-supervisor.tar
ls -lh /tmp/nanofab-supervisor.tar
```

**Expected outcome:** tarball ~150 MB at `/tmp/nanofab-supervisor.tar`.

### Step 3: Distribute the image to all 4 nodes

```bash
for ip in 192.168.1.162 192.168.1.160 192.168.1.163 192.168.1.164; do
  scp -i ~/.ssh/id_ed25519_hwmgmt /tmp/nanofab-supervisor.tar lsj@$ip:/tmp/
  ssh -i ~/.ssh/id_ed25519_hwmgmt lsj@$ip \
    'sudo ctr -n=k8s.io images import /tmp/nanofab-supervisor.tar'
done
```

**Note the `-n=k8s.io` flag:** kubeadm uses the `k8s.io` containerd namespace,
not the default. Omitting it imports the image into a namespace the kubelet
doesn't see.

**Expected outcome:** each node prints `Importing... elapsed: <N>s` and
returns 0. Verify via:

```bash
for ip in 192.168.1.162 192.168.1.160 192.168.1.163 192.168.1.164; do
  ssh -i ~/.ssh/id_ed25519_hwmgmt lsj@$ip \
    'sudo crictl images | grep nanofab-supervisor'
done
```

Each node shows the image with the same image ID.

### Step 4: Install via Helm

From the resink-core repo root:

```bash
helm install nanofab-supervisor-mvp deploy/charts/nanofab-supervisor \
  --values deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml \
  --namespace nanofab --create-namespace
```

**Expected outcome:**
```
NAME: nanofab-supervisor-mvp
LAST DEPLOYED: <timestamp>
NAMESPACE: nanofab
STATUS: deployed
REVISION: 1
```

## Verification

### Pod inventory

```bash
kubectl get all -n nanofab
```

**Expected object set:** 1 Deployment, 1 ReplicaSet, 1 Pod (cycling through
phases — see § Known chart constraints below), 0 Services (healthz disabled
in MVP values). Plus 1 ServiceAccount, 1 ConfigMap, 1 Role, 1 RoleBinding
(use `kubectl get sa,cm,role,rolebinding -n nanofab` to see them).

### Supervisor execution success

```bash
kubectl logs -n nanofab -l app.kubernetes.io/name=nanofab-supervisor --tail=10
```

**Expected output:**
```
[supervisor] [dim_user] loaded N events from M streams
[supervisor] [dim_account] loaded N events from M streams
[supervisor] [dim_user] wrote N rows to /workspace/dim_user_output.parquet
[supervisor] [dim_account] wrote N rows to /workspace/dim_account_output.parquet
supervisor: ok
```

The `supervisor: ok` line is the success marker. If absent, see § Failure modes.

### (Optional) Pod state inspection

```bash
kubectl describe pod -n nanofab -l app.kubernetes.io/name=nanofab-supervisor | tail -30
```

Shows the supervisor's exit code (0 for success), the container's state
transitions, and any kubelet events. With the current Deployment kind, the
container alternates between `Running` (briefly) and `Waiting (CrashLoopBackOff)`
because the kubelet restarts the container after every exit, including exit-0.
This is the expected MVP shape — see § Known chart constraints.

## Delete procedure

```bash
helm uninstall nanofab-supervisor-mvp -n nanofab
```

**Expected:** `release "nanofab-supervisor-mvp" uninstalled`. Within 30s,
`kubectl get all -n nanofab` returns "No resources found in nanofab namespace."

```bash
kubectl delete namespace nanofab
```

**Expected:** `namespace "nanofab" deleted`. Verify with
`kubectl get ns | grep nanofab` (empty).

## Failure modes

### F1: `ImagePullBackOff` — image not loaded on the scheduled node

**Severity:** S2 (deploy fails; cluster otherwise healthy)

**Symptom:** `kubectl get pods -n nanofab` shows `STATUS: ImagePullBackOff`.
`kubectl describe pod ...` shows event `Failed to pull image
"nanofab-supervisor:0.3.0": failed to resolve reference`.

**Diagnosis:**
```bash
NODE=$(kubectl get pod -n nanofab -l app.kubernetes.io/name=nanofab-supervisor \
       -o jsonpath='{.items[0].spec.nodeName}')
NODE_IP=$(kubectl get node $NODE -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
ssh -i ~/.ssh/id_ed25519_hwmgmt lsj@$NODE_IP \
    'sudo crictl images | grep nanofab-supervisor'
```

If the scheduled node doesn't have the image, the kubelet has nothing to pull
(`image.pullPolicy: IfNotPresent` + no registry = no fallback).

**Mitigation:** Re-run install Step 3 on the affected node (or all 4 to be
safe). Verify with the diagnosis command above before retrying `helm install`.

### F2: `ErrImagePull` — typo in image name/tag

**Severity:** S3 (deploy fails; trivially recoverable)

**Symptom:** `kubectl describe pod ...` shows event `Failed to pull image
"nanofab-supervisor:0.3.1": image not found` (note the wrong tag).

**Diagnosis:** Check `deploy/charts/nanofab-supervisor/values/home-cluster-mvp.yaml`
`image.tag` against the actual loaded image tag (`sudo crictl images | grep nanofab`).

**Mitigation:** Either rebuild + redistribute the image with the values' tag,
or update the values' `image.tag` to match the loaded image.

### F3: `CreateContainerConfigError` — ConfigMap or Secret missing

**Severity:** S2 (deploy fails; chart bug or partial install)

**Symptom:** `kubectl describe pod ...` shows event `couldn't find key X in
ConfigMap Y` or `secret Z not found`.

**Diagnosis:**
```bash
kubectl get cm,secret -n nanofab
kubectl get cm nanofab-supervisor-mvp -n nanofab -o yaml
```

**Mitigation:** If the ConfigMap is missing, `helm uninstall` and reinstall —
something interrupted the first install. If a Secret is missing, check
`values.secrets.kafkaSasl.secretRef.name` — if non-empty, the named Secret
must be provisioned out-of-band before install (chart never creates Secrets
with credentials).

### F4: Pod `Pending` for > 5 minutes — no schedulable node

**Severity:** S2 (deploy stalled; cluster may be saturated)

**Symptom:** `kubectl get pods -n nanofab` shows `STATUS: Pending` indefinitely.

**Diagnosis:**
```bash
kubectl describe pod -n nanofab -l app.kubernetes.io/name=nanofab-supervisor
kubectl get nodes
kubectl describe nodes | grep -A 3 "Allocated resources"
```

Common causes: (a) no node has the image + scheduling constraint; (b) all
nodes resource-constrained; (c) tolerations/affinity misconfigured in values.

**Mitigation:** If the cluster is the constraint, free resources (delete
unused workloads). If the chart's scheduling rules are at fault, set
`nodeSelector: {}` and `tolerations: []` in values.

### F5: `RBAC denied` — ServiceAccount can't read its ConfigMap

**Severity:** S2 (supervisor crashes after start)

**Symptom:** Supervisor logs show `403 Forbidden` accessing the K8s API.

**Diagnosis:**
```bash
kubectl auth can-i get configmap -n nanofab \
  --as=system:serviceaccount:nanofab:nanofab-supervisor-mvp
```

Should return `yes`. If `no`, the chart's Role/RoleBinding didn't apply.

**Mitigation:** `helm uninstall` and reinstall. If the issue persists, check
`values.rbac.create` (should be `true`) and inspect the rendered Role +
RoleBinding via `helm template ...`.

### F6: `helm uninstall` leaves residual objects

**Severity:** S2 (cleanup incomplete; future installs fail with "name in use")

**Symptom:** After `helm uninstall`, `kubectl get all -n nanofab` still shows
some objects, OR `helm install` fails with `cannot reuse a name that is still
in use`.

**Diagnosis:**
```bash
helm list -n nanofab --all                  # shows release in any state
kubectl get all,cm,sa,role,rolebinding,secret -n nanofab
```

**Mitigation:** Current chart has no hooks or finalizers, so residual objects
should not occur. If they do: manually delete them via
`kubectl delete <kind>/<name> -n nanofab`, then `kubectl delete namespace nanofab`.
For a release in `failed` state, `helm uninstall <release> -n <ns>` is idempotent.

## Known chart constraints

Surfaced at loop 2026-05-12-0645 (home-cluster first deploy):

1. **`kubectl cp` of `/workspace/trace.jsonl` is unreliable.** The MVP supervisor
   exits 0 after one pass; under `kind: Deployment`, the kubelet restarts the
   container after every exit. The pod alternates between `Running` (a few
   hundred ms) and `Waiting (CrashLoopBackOff)` (30s+). `kubectl cp` races
   the BackOff state and fails with `container not found ("supervisor")`.
   **Workaround:** use `kubectl logs` to capture the supervisor's stdout (which
   names the row counts and the `supervisor: ok` success marker). **Future
   fix:** a sibling `kind: Job` chart variant gated on a `workloadKind` values
   flag — Job-kind pods stay in `Completed` state after exit-0, making
   `kubectl cp` reliable. Sized at a future loop.

2. **`pod.restartPolicy` values knob is documentation-only.** Deployment.spec.template.spec
   only admits `restartPolicy: Always`. The chart's deployment.yaml omits the
   field; the values knob is reserved for the future Job-shape variant.

3. **Image distribution is per-node `ctr import`, not a registry pull.** Every
   image rebuild requires re-running the distribute loop. The home-cluster
   roadmap's Phase 1 includes a local in-cluster registry; until then, treat
   the distribute step as part of every deploy.

4. **`Chart.appVersion` is informational.** The authoritative pull target is
   `values.image.tag`. When the supervisor binary's crate version changes,
   update the values file and rebuild the image.

## Cross-references

- Chart source: [`repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/`](../../../repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/)
- Chart README: [`repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md`](../../../repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md)
- Resink-core user-guide deploy section: [`repos/resink-ai/resink-core/docs/user-guide.md`](../../../repos/resink-ai/resink-core/docs/user-guide.md) § "Deploying to the home cluster"
- Home-cluster operator runbook: [`repos/resink-ai/home-cluster/docs/k8s-cluster.md`](../../../repos/resink-ai/home-cluster/docs/k8s-cluster.md) § "Smoke-test workloads"
- Companion runbook (runtime failures): [`teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md`](nanofab-supervisor-failed-validation.md)

## Verified-against-environment

Per ADR-2026-05-16-003 — first runbook to honor the discipline.

- **Toolchain versions** (verified at loop 2026-05-12-0645):
  - `kubectl` client `v1.33.9` / server `v1.33.0`
  - `helm` `v4.1.4`
  - `docker` client `v29.4.0` (OrbStack on macOS arm64)
  - `docker buildx` `v0.33.0`
  - ssh OpenSSH (system default; key `id_ed25519`)
- **OS + architecture:**
  - Operator: macOS 24.6.0 arm64 (Apple Silicon) OR Linux x86_64
  - Cluster nodes: Ubuntu 24.04.4 LTS x86_64 (`uname -m` returns `x86_64`)
- **Runtime dependencies:**
  - Cluster: kubeadm-bootstrapped, Calico CNI, MetalLB L2, local-path-provisioner default StorageClass
  - containerd `v2.2.3` on every node, namespace `k8s.io` for kubelet-visible images
  - sudo passwordless for `lsj` on every node (baseline guarantee)
- **Auth-mode prerequisites:**
  - kubeconfig with `kubernetes-admin@kubernetes` context; if absent, copy
    `/etc/kubernetes/admin.conf` from k8s-0 per home-cluster runbook
  - ssh key at `~/.ssh/id_ed25519_hwmgmt` (the hwmgmt-fleet key); failure mode
    if absent: `Permission denied (publickey)` on the `scp` step
- **Cross-arch build (macOS arm64 operator only):**
  - Docker buildx must support `linux/amd64` (verify `docker buildx ls`)
  - QEMU emulation is used; full clean build is ~2 min, incremental ~10s

---

**Post-promotion note (2026-05-12-1254):** `repos/resink-ai/resink-core/` is now a git submodule (URL `git@github.com:resink-ai/resink-core.git`). All cited paths continue to resolve at the same working-tree mount point. Future runbook revisions may shift to submodule-relative paths if external consumers need disambiguation.
{% endraw %}
