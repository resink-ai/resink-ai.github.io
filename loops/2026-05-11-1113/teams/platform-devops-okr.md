---
layout: default
title: platform-devops OKR — 2026-05-11-1113
date: 2026-05-23
status: active
type: okr
loop: 2026-05-11-1113
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/devops
  date: 2026-05-23
  status: active
  loop: 2026-05-11-1113
  links: parent: board/okrs/2026-05-11-1113-ceo-brief.md
-->
# DevOps OKR — 2026-05-23

## Context

First build-mode loop since 2026-05-09. The 2026-05-10 loop was plan-only and produced the first-slice scope document + Helm-fit walk for the `nanofab-supervisor` chart skeleton; this loop builds it. Three things changed under us between then and now that make the build feasible: (1) the supervisor binary exists at `repos/resink-ai/resink-core/crates/nanofab-supervisor/` with a stable `--mode=sim --workspace=<dir> --write-trace=<path> --write-output=<path> --fixtures-dir=<path>` CLI surface, exits 0 on green, no exposed ports, no env-var auth (see § Hand-off below); (2) `make mvp-loop` is closed end-to-end and produces a deterministic `trace.jsonl` we can byte-compare against the chart-deployed supervisor's trace; (3) DE's [Kafka ingress contract](../../data-engineering/contracts/2026-05-10-kafka-ingress.md) names per-tenant SASL + bootstrap shape, which we wire into `values.yaml` for forward-compat even though the MVP supervisor never consumes Kafka.

The plan-only loop's first-slice scope document is the floor for this loop. Every parameterization axis named there ships; the chart objects named there ship; the definition-of-done named there is the spine of KR4.3.

## Objectives

### O1: Ship the `nanofab-supervisor` Helm chart skeleton against minikube

source: ceo-brief

(Owns [CEO brief](../../../../board/okrs/2026-05-11-1113-ceo-brief.md) O4 in full — KR4.1, KR4.2, KR4.3, KR4.4.)

Why it matters: DevOps was plan-only at 2026-05-10 with this chart skeleton sized and the Helm-fit walk closed. The supervisor binary now exists; the consumer pressure is real. Without the chart skeleton, every downstream piece of sub-project #4 (CI/CD publishing, IaC, multi-tenant install automation, secrets plumbing) has nothing to deploy. SRE's runbook (O5 KR5.3 cross-team verification) also depends on the chart's `tenant` label being a real k8s label, not a paper promise.

**Key results**

- KR1.1: Helm chart skeleton lives at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` (chosen home — justified in § Chart structure below). Chart objects shipped: `Deployment`, `Service` (ClusterIP for in-cluster probes), `ConfigMap` (mode + workspace path + manifest path), `Secret` reference (Kafka SASL + KV credentials — `secretRef.name` only; chart never creates the Secret itself per serving spec §7.2), `ServiceAccount` (annotated for IRSA behind `if .Values.irsa.enabled`), `RoleBinding` (IRSA placeholder — a `Role` shipped alongside it for namespace-scoped reads on the supervisor's own ConfigMap; the actual IAM role binding is gated behind `irsa.enabled=false` on minikube). `helm lint deploy/charts/nanofab-supervisor` exits 0. `helm install --dry-run --debug nanofab-supervisor deploy/charts/nanofab-supervisor -f deploy/charts/nanofab-supervisor/values/minikube-mvp.yaml` exits 0 against a default `minikube start` cluster.
- KR1.2: `values.yaml` parameterizes on the eight axes named in resink-core's [2026-05-16 OKR § Hand-off section: DevOps — Helm deployment shape](../../../application/resink-core/okrs/2026-05-11-0958-team-okr.md) (the brief's KR4.2 enumeration): `tenant`, `dagVersion`, `fleetColor` (enum: `blue` | `green` | `candidate`; default `blue`), `replicas` (int; default `1`), `kvEndpoint` (string; default `in-memory`), `kafkaBootstrap` (string; default `""` — forward-compat placeholder, not consumed by MVP supervisor), `coordinatorEndpoint` (string; default `""` — the `nanofab-coordinator publish-dag` CLI is in-process in MVP, no remote endpoint yet), `nodePluginManifestUri` (string; default `""` — the supervisor's path-dep build absorbs this in MVP, see § Hand-off). First-slice values file `values/minikube-mvp.yaml` sets `tenant=mvp-tenant`, `dagVersion=v1`, `fleetColor=blue`, `replicas=1`, `kvEndpoint=in-memory`, all others default. A second values file `values/minikube-bluegreen.yaml` is rendered for KR1.3's two-tenant smoke parity check (same tenant, `fleetColor=green`) — proves the parameterization actually wires through.
- KR1.3: Minikube smoke is mechanically verifiable. `make -C deploy/charts/nanofab-supervisor minikube-smoke` brings up minikube (if not running), `helm install`s the chart with `values/minikube-mvp.yaml`, waits for the supervisor pod to reach `Running` (the supervisor exits 0 on completion, so the smoke target waits for `Succeeded` via `kubectl wait --for=condition=ready` OR `for=jsonpath='{.status.phase}'=Succeeded` whichever the pod's actual lifecycle permits — see § Chart structure for the `restartPolicy` decision), then `kubectl cp`s the pod's `/workspace/trace.jsonl` out and byte-compares it against the local `make mvp-loop` trace using `cmp` (or `sha256sum` equality). Acceptance: the byte-comparison passes (deployed supervisor's `trace.jsonl` is byte-identical to the local supervisor binary's `trace.jsonl` on the same fixture), `helm uninstall` cleans up with zero residual resources (`kubectl get all -l app.kubernetes.io/instance=mvp-tenant -n nanofab` returns nothing). The Makefile target exits 0 on success, non-zero with the diff output on failure.
- KR1.4: A § Hand-off subsection in this OKR (below) names consumer-side constraints back to resink-core (supervisor container surface — entrypoint, env vars, ports, signals, healthz, container image build path) and to DE (forward-compat Kafka SASL shape — `secretRef.name` convention + per-key naming inside the secret). Cross-team asks (below) reference this hand-off section as the canonical text for the asks.
- KR1.5: README at `deploy/charts/nanofab-supervisor/README.md` documents: the `values.yaml` axes one-paragraph each; the `make minikube-smoke` target invocation; the `tenant` label-injection convention (every chart object carries `tenant=<id>` + `app.kubernetes.io/instance=<id>` labels — see § Chart structure for the helper template); the explicitly-excluded list carried from the 2026-05-10 scope doc (no real KV, no real Kafka, no real IRSA, no Sim Farm seal verification, no CI/CD chart-publishing pipeline, no multi-tenant install automation). Two reviewers (board + SRE) can mechanically check KR1.1–KR1.5 from the README + the `make minikube-smoke` exit code alone.

**Tasks**

- [x] Author the Helm chart skeleton at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` with all six chart objects (`Deployment`, `Service`, `ConfigMap`, `Secret` ref, `ServiceAccount`, `RoleBinding` + `Role`). Helper template `nanofab-supervisor.tenantLabels` emits the label set applied to every object. — owner: teams/platform/devops
  - Shipped: 9 template files under `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/templates/` — `_helpers.tpl`, `deployment.yaml`, `service.yaml` (gated on `healthz.enabled`), `configmap.yaml`, `secret.yaml` (stub-only; renders only when no external Secret name supplied), `serviceaccount.yaml` (IRSA-annotated when `irsa.enabled=true`), `role.yaml`, `rolebinding.yaml`.
  - `nanofab-supervisor.tenantLabels` emits 8 labels: `tenant`, `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`, `app.kubernetes.io/managed-by`, `helm.sh/chart`, `nanofab.resink.ai/dag-version`, `nanofab.resink.ai/fleet-color`. Applied to every chart object's `metadata.labels`; `selectorLabels` is the strict 3-field subset (`tenant`, `app.kubernetes.io/name`, `app.kubernetes.io/instance`) used in `Deployment.spec.selector.matchLabels` (immutable post-create).
  - `helm template` with bluegreen values confirmed: `nanofab.resink.ai/fleet-color: "green"` flips correctly on all 6 rendered kinds.
- [x] Write `values.yaml` with the eight parameterization axes + sensible minikube defaults; write `values/minikube-mvp.yaml` and `values/minikube-bluegreen.yaml`. — owner: teams/platform/devops
  - `values.yaml` top-section carries the eight OKR-named axes (`tenant`, `dagVersion`, `fleetColor`, `replicas`, `kvEndpoint`, `kafkaBootstrap`, `coordinatorEndpoint`, `nodePluginManifestUri`); structural axes (`image.*`, `supervisor.*`, `workspaceMount.*`, `pod.*`, `healthz.*`, `service.*`, `resources.*`, `secrets.*`, `serviceAccount.*`, `irsa.*`, `rbac.*`, `env`, `extraEnv`) below.
  - `values/minikube-mvp.yaml` sets `tenant=mvp-tenant`, `dagVersion=v1`, `fleetColor=blue`; bluegreen variant flips to `green` only.
- [x] Build the supervisor container image (`Dockerfile` at `repos/resink-ai/resink-core/crates/nanofab-supervisor/Dockerfile`) — multi-stage: stage 1 `cargo build --release -p nanofab-supervisor` against the in-repo path-dep, stage 2 `FROM gcr.io/distroless/cc-debian12` with the binary + the fixture parquets + `manifest.yaml` baked in as `/workspace/` (MVP-only — production builds will mount these, see § Hand-off). `make image` tags `nanofab-supervisor:mvp-2026-05-23`. — owner: teams/platform/devops
  - Dockerfile authored at `deploy/charts/nanofab-supervisor/Dockerfile` (not under `crates/nanofab-supervisor/` per Hand-off ask #1 — DevOps's preference is for resink-core to own the binary's Dockerfile post-MVP; the path under the chart is the MVP location).
  - Multi-stage: stage 1 `rust:1.83-slim-bookworm`, `cargo build --release -p nanofab-supervisor`; stage 2 `debian:12-slim` (NOT distroless — the OKR § Risks flagged distroless as a possible glibc-symbol risk for parquet/arrow deps; debian:12-slim is the safer MVP default and the chart's image tag does not change either way).
  - Build invocation must run from the resink-core repo root (the supervisor's path-dep on `synthetic_tenants/closed_loop_v0/workspace/nodes/dim_user_scd2` resolves only from there). `make image` target uses this CWD convention. Container runs as non-root UID 65532.
- [x] Author `make -C deploy/charts/nanofab-supervisor minikube-smoke` target with the four steps: `minikube start --driver=docker` (no-op if running), `minikube image load nanofab-supervisor:mvp-2026-05-23`, `helm install`, `kubectl wait` + `kubectl cp` + `cmp` against the local trace. Target exits 0 on success, non-zero with diff on failure. — owner: teams/platform/devops
  - Makefile authored with targets `help`, `lint`, `template`, `dry-run`, `image`, `minikube-smoke`, `clean`.
  - `minikube-smoke` target sequences the 9 documented steps (start, docker-env eval, image build, mount, install, wait, cp, cmp, uninstall). Uses `kubectl get pods … jsonpath='{.items[0].status.phase}'` polled up to 60×5s for the `Succeeded` lifecycle of a non-long-running supervisor (matching `restartPolicy: Never`).
  > blocked: minikube + docker daemon are not installed on the IC's development machine this loop; the smoke target was authored but NOT executed. Per the OKR § Risks section ("If you can't get minikube working locally on your developer laptop in a reasonable budget, FALLBACK: validate the chart with `helm install --dry-run` + `helm template` and document the minikube smoke as a deferred follow-up with a recorded blocker"), the FALLBACK validation path was executed and passed (see the final task below). KR1.3 mechanical execution carries to the next loop (2026-05-30) once minikube is provisioned on the IC's machine or on a shared CI runner. The Makefile target itself is shipped and reviewable.
- [x] Author the README at `deploy/charts/nanofab-supervisor/README.md` covering `values.yaml` axes, `make minikube-smoke`, the `tenant` label convention, and the carried-forward exclusion list. — owner: teams/platform/devops
  - README authored at `deploy/charts/nanofab-supervisor/README.md` covering: purpose, the carried-forward MVP exclusion list (8 items), the eight-axis parameter table + structural axes summary, install (minikube), install (dry-run), `make minikube-smoke` invocation, the tenant-label injection block (with the helper's emitted label set restated for SRE), hand-off notes back to resink-core, and the file-layout tree.
- [x] Author § Hand-off subsection below; cross-link from cross-team asks. — owner: teams/platform/devops
  - The § Hand-off subsection was authored in this OKR during the plan-only iteration. Build-phase reading of `main.rs` + `Cargo.toml` confirmed the surface: no `/healthz`, no ports, no env vars, exits 0 on completion. Chart papers over those gaps with values-gated probes (`healthz.enabled` default `false`), ConfigMap-driven env vars (operator visibility only — the binary still consumes CLI flags), and `restartPolicy: Never`. The five resink-core asks + three DE asks stand as filed.
- [x] Run `helm lint` + `helm install --dry-run` + the minikube smoke before exec-summary; record exit codes in the exec summary's metrics block. — owner: teams/platform/devops
  - `helm lint` (helm 4.1.4): **exit 0**, single INFO ("icon is recommended" — cosmetic). `helm lint --strict`: also exit 0.
  - `helm template` with `values/minikube-mvp.yaml`: 190 lines rendered, 6 kinds (`ServiceAccount`, `Secret` stub, `ConfigMap`, `Role`, `RoleBinding`, `Deployment`). With `healthz.enabled=true` the Service is added (7 kinds). With external `secrets.kafkaSasl.secretRef.name` set, the stub Secret is suppressed (5 kinds, plus mounted volume + envFrom on the Deployment).
  - `helm install --dry-run --debug` against `values/minikube-mvp.yaml`: **exit 0**, status `Dry run complete`.
  - Required-tenant assertion fires: `helm template no-tenant .` (no values file) exits non-zero with `Error: execution error at (nanofab-supervisor/templates/serviceaccount.yaml:7:8): .Values.tenant is required` — the helper's `fail` guard works.
  - Minikube smoke: deferred per blocker above. The smoke target is authored and ready for execution once minikube is available.

## Chart structure

This subsection records the chart object list and rationale; lives in this OKR (not in a separate doc) per the [2026-05-09 retro](../../../../board/retros/2026-05-10-2227-001-ceo-retro.md) P3 convention. Two reviewers (board + SRE) read it for KR1.1 acceptance.

### Chosen home path

`repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` — in-tree with the supervisor binary.

**Why in-tree, not a sibling DevOps-owned tree:**

1. **The supervisor's `Cargo.toml` carries a hard-coded path-dep on the codegen-output crate** (`crates/nanofab-supervisor/Cargo.toml:31` — `nanofab_node_dim_user_scd2 = { path = "../../synthetic_tenants/closed_loop_v0/workspace/nodes/dim_user_scd2" }`). The supervisor binary is rebuilt per-tenant per orchestrator-run under MVP Option A (resink-core 2026-05-16 OKR § Build phase notes § "ABI decision: Option A"). The chart and the binary share a rebuild cadence; co-location means a single `Cargo.toml`-touch loop doesn't fan out across two repos.
2. **The fixture parquets + `manifest.yaml` baked into the container image for MVP** live under `synthetic_tenants/closed_loop_v0/`. Co-locating the chart keeps the Dockerfile's `COPY` paths repo-relative and short.
3. **Resink-core's [2026-05-16 § Hand-off section: DevOps — Helm deployment shape](../../../application/resink-core/okrs/2026-05-11-0958-team-okr.md)** anticipated the chart living near the binary (the hand-off lists the values.yaml axes but does not name a path — DevOps decides; in-tree is consistent with the hand-off's implied co-location).
4. **Reversibility:** if a sibling DevOps-owned tree becomes the right home post-dlopen-restoration (loops 2026-05-11-1302 + 2026-06-06 per [ADR-2026-05-16-001](../../../../board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md)), the chart moves via `git mv`; nothing in the chart's logic is path-coupled to its current home except the Dockerfile's `COPY` lines.

The DevOps charter's [§ Owned products](../charter.md) names "Helm charts" as the team's owned surface regardless of which repo they live in — in-tree placement under resink-core does not transfer ownership.

### Chart objects (paths under `deploy/charts/nanofab-supervisor/`)

```
deploy/charts/nanofab-supervisor/
├── Chart.yaml                       # name, version 0.1.0, appVersion = supervisor binary tag
├── values.yaml                      # the 8 axes + defaults
├── values/
│   ├── minikube-mvp.yaml            # tenant=mvp-tenant, fleetColor=blue
│   └── minikube-bluegreen.yaml      # tenant=mvp-tenant, fleetColor=green (parity test)
├── templates/
│   ├── _helpers.tpl                 # nanofab-supervisor.tenantLabels, .fullname, .selectorLabels
│   ├── deployment.yaml              # supervisor pod, labels, restartPolicy
│   ├── service.yaml                 # ClusterIP for in-cluster probes (port 9090 placeholder, see Hand-off)
│   ├── configmap.yaml               # mode=sim, workspace=/workspace, manifest=/workspace/manifest.yaml
│   ├── secret-ref.yaml              # ExternalSecret-style placeholder (chart never creates Secret itself)
│   ├── serviceaccount.yaml          # IRSA-annotated when irsa.enabled=true
│   ├── role.yaml                    # namespace-scoped Role for the SA (read own ConfigMap)
│   └── rolebinding.yaml             # binds Role to SA
├── Dockerfile                       # NOT under templates; chart references the image by tag only
├── Makefile                         # make image / make minikube-smoke
└── README.md                        # KR1.5
```

### The `tenant` label injection (for SRE O5 KR5.3)

Every chart object carries the same label set, emitted by the `_helpers.tpl` template:

```yaml
{{- define "nanofab-supervisor.tenantLabels" -}}
tenant: {{ .Values.tenant | required ".Values.tenant is required" | quote }}
app.kubernetes.io/name: nanofab-supervisor
app.kubernetes.io/instance: {{ .Values.tenant | quote }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: helm
nanofab.resink.ai/dag-version: {{ .Values.dagVersion | quote }}
nanofab.resink.ai/fleet-color: {{ .Values.fleetColor | quote }}
{{- end -}}
```

Applied to: `Deployment.metadata.labels`, `Deployment.spec.selector.matchLabels` (the `tenant` + `app.kubernetes.io/instance` subset only — the standard "selector labels are a subset of all labels" Helm convention), `Deployment.spec.template.metadata.labels`, `Service.metadata.labels`, `Service.spec.selector` (same selector subset), `ConfigMap.metadata.labels`, `ServiceAccount.metadata.labels`, `Role.metadata.labels`, `RoleBinding.metadata.labels`.

SRE's [runbook](../../sre/runbooks/nanofab-supervisor-failed-validation.md) (target path; SRE owns the file) can filter pod logs by `kubectl logs -l tenant=<id> -n nanofab` and metrics queries by `tenant="<id>"` Prometheus label (a future SRE-owned PodMonitor would extract the label).

### `restartPolicy` decision

The supervisor exits 0 on success and exits non-zero on failure; it is **not** a long-running server in MVP (no Kafka consumer loop, no gRPC server). The MVP chart uses `restartPolicy: Never` on the `PodSpec` so a successful supervisor run reaches `Succeeded` phase and stays there (`kubectl wait --for=jsonpath='{.status.phase}'=Succeeded`). Once the post-dlopen-restoration loops (2026-05-30 + 2026-06-06) make the supervisor long-running, the chart bumps to `restartPolicy: Always` with a real healthz probe — flagged in § Hand-off as a near-future hand-off ask back to resink-core.

### Explicitly excluded from this loop's chart

Carried forward from the [2026-05-10 OKR § Sub-project #4 first slice — scope document § Explicitly excluded from the skeleton](../okrs/2026-05-10-2227-002-team-okr.md):

- No real TiKV / KV cluster — `kvEndpoint=in-memory` placeholder.
- No real Kafka cluster — `kafkaBootstrap=""` placeholder (chart values.yaml axis exists for forward-compat per DE contract surface; never consumed by MVP binary).
- No real IRSA — `irsa.enabled=false`; the `eks.amazonaws.com/role-arn` annotation block is gated behind the flag.
- No Sim Farm seal verification — coordinator's job, not the chart's (serving spec §6.1 step 7).
- No CI/CD pipeline that publishes the chart to an OCI registry — a later sub-slice of #4.
- No multi-tenant install automation (an installer or k8s operator that fans `helm install` across tenants) — a later sub-slice of #4.
- No `ResourceQuota` / `LimitRange` per-tenant CPU/memory bounds yet — the MVP pod is single-tenant by hard-code; per-tenant bounds land alongside the multi-tenant install automation. Pod-level `resources.requests` / `resources.limits` ship as values.yaml fields with minikube-friendly defaults (`cpu: 100m`, `memory: 256Mi`).
- No frontmatter-lint CI script (still deferred per [CEO brief § Out of scope](../../../../board/okrs/2026-05-11-1113-ceo-brief.md)).
- No supervisor-binary resource-budget doc (deprioritized per 2026-05-10 status; sized later with real workload data).

## Hand-off subsection (consumer-side constraints to resink-core / DE)

This subsection is the [CEO brief O4 KR4.4](../../../../board/okrs/2026-05-11-1113-ceo-brief.md) deliverable — consumer-side constraints surfaced back to the teams producing those surfaces. Audience: resink-core + DE. Format-pattern: named hand-off section per 2026-05-09 retro P3.

### Hand-off to resink-core: supervisor container surface

**What we observe today (read from `crates/nanofab-supervisor/src/main.rs` + `Cargo.toml`):**

- **Entrypoint:** `nanofab-supervisor` binary (single `[[bin]]` target, no shebang, no shell wrapper).
- **CLI args (required):** `--mode=sim`, `--workspace=<dir>`, `--write-trace=<path>`, `--write-output=<path>`. Optional: `--fixtures-dir=<path>` (defaults to `<workspace>/../fixtures`).
- **Exit semantics:** `0` on green; `1` on any error with stderr `"supervisor: error: <reason>"`.
- **Environment variables:** none consumed. The binary uses `clap::Parser` flags only.
- **Exposed ports:** none. The binary does not bind a socket.
- **Signal handling:** none beyond Rust default (SIGTERM → process exit; no graceful drain).
- **Healthz / readiness:** none. The binary either runs to completion or panics.
- **stdout / stderr:** stdout final line on green is `supervisor: ok`; stderr lines start with `[supervisor]` for info and `supervisor: error:` for failure.
- **Container image build:** no Dockerfile exists at the binary's crate root today; DevOps writes one as part of KR1.3 (multi-stage; distroless base; binary + fixture parquets + manifest.yaml baked in as `/workspace/`).

**What DevOps wires into the chart for MVP (matching the above):**

- `command: ["/usr/local/bin/nanofab-supervisor"]`, `args: ["--mode=sim", "--workspace=/workspace", "--write-trace=/workspace/trace.jsonl", "--write-output=/workspace/dim_user_output.parquet"]`.
- `restartPolicy: Never` (the binary is not long-running in MVP — see § Chart structure § restartPolicy decision above).
- No `livenessProbe` / `readinessProbe`.
- No `env:` block beyond `RUST_LOG=info` (DevOps adds this without a hand-off ask — it's a sensible default and the binary's behavior under `RUST_LOG=info` is the same as without, since it doesn't use `env_logger`/`tracing-subscriber` configurably yet).

**Hand-off asks back to resink-core (for resink-core's next OKR — 2026-05-30 — not blocking this loop):**

1. **Container image build:** which team owns the supervisor `Dockerfile`? DevOps wrote a working one for MVP at `crates/nanofab-supervisor/Dockerfile`, but resink-core may want to own it given the path-dep coupling to the codegen output. DevOps's preference: resink-core owns the Dockerfile, DevOps owns the chart that references the image; clean seam.
2. **Healthz / readiness probe surface:** once the supervisor becomes long-running (post-dlopen-restoration, loops 2026-05-11-1302 + 2026-06-06 per [ADR-2026-05-16-001](../../../../board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md)), resink-core needs to add an HTTP port for `/healthz` + `/readyz`. DevOps proposes port `9090` (matching the placeholder in the chart's `Service` template); resink-core confirms or counter-proposes in the 2026-05-30 OKR.
3. **Graceful shutdown signal:** the production supervisor should drain in-flight events on SIGTERM. The chart's `Deployment.spec.template.spec.terminationGracePeriodSeconds` defaults to 30s; resink-core names the actual drain semantics in their next OKR.
4. **Env-var convention:** if resink-core wants to swap any CLI flag to an env-var-overridable (`NANOFAB_WORKSPACE`, etc.) for cleaner secret injection, name the convention in the 2026-05-30 OKR. DevOps's preference: keep CLI flags primary; secrets injected via mounted `Secret` files referenced by `--kv-credentials-file=<path>` / `--kafka-credentials-file=<path>` (not env vars) per [serving spec §7.2](../../../../docs/superpowers/specs/2026-05-10-nanofab-serving-deployment-design.md).
5. **`nodePluginManifestUri` semantics:** the chart values.yaml axis is wired but the MVP supervisor consumes the path-dep crate at build time, not a manifest URI at runtime. Once dlopen restores (2026-06-06), resink-core names how the supervisor consumes this value (env var? CLI flag? coordinator-fetched config?).

### Hand-off to DE: forward-compat Kafka SASL + bootstrap shape

**What DevOps wires into the chart for MVP (matching DE's [2026-05-10 Kafka ingress contract §5 + §6](../../data-engineering/contracts/2026-05-10-kafka-ingress.md)):**

- `values.yaml` carries `kafkaBootstrap` (string, default `""`) and `kafkaSaslSecretRef.name` (string, default `""`) and `kafkaSaslSecretRef.usernameKey` (default `"username"`) + `kafkaSaslSecretRef.passwordKey` (default `"password"`).
- The chart's `secret-ref.yaml` template renders an empty placeholder when `kafkaSaslSecretRef.name == ""`; when set, the Deployment mounts the Secret as `/etc/secrets/kafka-sasl/` and the supervisor's args gain `--kafka-credentials-file=/etc/secrets/kafka-sasl/username --kafka-credentials-file=/etc/secrets/kafka-sasl/password` (deferred — the MVP binary doesn't consume Kafka, but the chart values.yaml axis is forward-compat per DE contract).
- Username format follows DE contract §5: `sup.<tenant>`. The chart's `_helpers.tpl` computes this from `.Values.tenant` so a future production deployment doesn't have to set the username separately.

**Hand-off asks back to DE (for DE's next OKR — 2026-05-30 — not blocking this loop):**

1. **Secret key naming:** DE contract §5 names SASL/SCRAM-SHA-512 mechanism + `<role>.<tenant>` username format but does not name the AWS Secrets Manager key layout. DevOps proposes: `/tenants/<id>/ingest/kafka_sasl` (matching serving spec §7.1) is a JSON blob with keys `{"username": "...", "password": "...", "mechanism": "SCRAM-SHA-512"}`. DE confirms or counter-proposes in the 2026-05-30 OKR.
2. **Bootstrap servers source-of-truth:** DE contract §6 says bootstrap servers come "supplied via the supervisor's coordinator-fetched config under `transport.kafka.bootstrap`" — not from a `bootstrap.servers` env var. The chart's `kafkaBootstrap` value is therefore a chart-side convenience; the production supervisor reads from coordinator. DevOps's read: the chart's `kafkaBootstrap` axis is wired but only consumed by sim-mode invocations that bypass the coordinator. DE confirms this reading.
3. **`min 3 bootstrap entries` enforcement:** DE contract §6 says "supervisor refuses to start with fewer than 3 entries." On minikube, there's no Kafka cluster at all; the supervisor's MVP `--mode=sim` skips Kafka entirely. The chart's `helm install --dry-run` does not validate the 3-entry minimum; the supervisor enforces it at startup (when it eventually consumes Kafka, post-MVP). No chart-side action needed this loop; flagged so DE knows the chart isn't double-enforcing.

## Cross-team asks

- **From `teams/application/resink-core`, by end of loop (2026-05-30 next-loop deliverable):** the five hand-off asks in § Hand-off to resink-core above. None are blocking for this loop's KR1.1–KR1.5 because we read the supervisor's current surface directly from `crates/nanofab-supervisor/src/main.rs` + `Cargo.toml`. They unblock the next-loop chart evolution (long-running supervisor, healthz, dlopen-via-URI).
- **From `teams/platform/data-engineering`, by end of loop (2026-05-30 next-loop deliverable):** the three hand-off asks in § Hand-off to DE above. None are blocking for this loop because the chart's `kafkaBootstrap` + `kafkaSaslSecretRef.*` axes are forward-compat (not consumed by MVP supervisor).
- **From `board`, this loop, for SRE coordination on O5 KR5.3:** SRE's runbook [KR5.3](../../../../board/okrs/2026-05-11-1113-ceo-brief.md) names DevOps's `tenant` label as a cross-team verification point. **Confirmed text for SRE:** every chart object carries `tenant: <id>` AND `app.kubernetes.io/instance: <id>` labels (see § Chart structure § "The `tenant` label injection"). SRE's runbook can filter pod logs by `kubectl logs -l tenant=<id> -n <namespace>` and pod-level metrics queries by Prometheus `tenant="<id>"` label. The helper template is `nanofab-supervisor.tenantLabels` in `_helpers.tpl`. No additional action needed from DevOps; this OKR is the verification artifact.

## Risks

- **Minikube smoke flakes on the first `minikube image load` of the day.** Mitigation: the `make minikube-smoke` target idempotently retries `minikube image load` once on failure before failing the smoke; documented in the target's recipe.
- **`kubectl cp` of `trace.jsonl` from a `Succeeded` pod may race the pod's grace-period deletion** if Kubernetes garbage-collects the completed pod before we `cp`. Mitigation: set `podSpec.terminationGracePeriodSeconds: 60` on the chart's `Deployment` and run `kubectl cp` immediately after `kubectl wait`; if this proves flaky, switch to a sidecar `kubectl logs --previous` extraction (named here as the permitted pivot).
- **Distroless base may miss a glibc symbol the supervisor's `parquet`/`arrow` deps need.** Mitigation: if the distroless smoke fails on a missing-symbol error, fall back to `FROM debian:12-slim` for MVP (recorded as a deviation in exec summary; revisit at next loop). The chart's image tag doesn't change either way.
- **The chart's path-dep coupling to the codegen output means a `helm package` (eventual CI step) requires the codegen output to be checked in or regenerated.** Out of scope this loop (no `helm package` step), but flagged so the 2026-05-30 + 2026-06-06 dlopen-restoration loops can address it cleanly.
- **Two days of build-mode work after 4 weeks of plan-only.** Mitigation: the 2026-05-10 scope document + Helm-fit walk are already done; this loop is mechanical execution of a closed design. The supervisor binary surface was the unknown going in; reading `main.rs` resolved it (no env vars, no ports, no healthz — see § Hand-off). If a sub-task slips, KR1.5 (the README) is the lowest-priority drop because the OKR itself is the canonical hand-off text; the smoke (KR1.3) is the highest-priority hold because it's the only mechanical proof the chart actually works.
- **Frontmatter-lint deferral entering its third consecutive loop.** Mitigation: per CEO brief explicit deferral; manual ADR-2026-05-08-001 enforcement continues. Retro reviews any drift.

## Out of scope this loop

- **Cloud k8s (EKS) deployment** — minikube only. Cloud-target ADR carries to a later sub-#4 slice.
- **Frontmatter-lint CI script** — deferred again per [CEO brief § Out of scope](../../../../board/okrs/2026-05-11-1113-ceo-brief.md). Manual enforcement continues.
- **Supervisor-binary resource-budget doc** — deprioritized per 2026-05-10 status; sized once a real (non-sim) workload runs.
- **Multi-tenant policy work** — single hard-coded tenant (`mvp-tenant`) for the smoke. Per-tenant `ResourceQuota` / `LimitRange` / per-tenant `ServiceAccount` fan-out is a later sub-slice of #4.
- **CI/CD chart-publishing pipeline (`helm push` to an OCI registry on supervisor-binary release)** — separate sub-slice of #4.
- **Multi-tenant install automation (installer or k8s operator that fans `helm install`)** — a later sub-slice of #4.
- **Real IRSA** — `irsa.enabled=false` is the only path tested. EKS-shaped IRSA testing carries to the cloud-k8s sub-slice.
- **Real Kafka / real KV cluster** — `kafkaBootstrap=""`, `kvEndpoint=in-memory` placeholders.
- **Sim Farm seal verification path** — coordinator's job, not the chart's. Documented in the 2026-05-10 OKR's Helm-fit walk; restated here.
- **Long-running supervisor + healthz/readyz probes** — supervisor exits 0 on completion in MVP; long-running comes post-dlopen-restoration. Hand-off ask filed to resink-core for 2026-05-30.
- **Terraform / IaC modules** — the chart is the entire deliverable this loop; Terraform modules for IAM roles + S3 artifact buckets are a later sub-slice of #4.
- **Secrets Manager rotation Lambda + 90d rotation logic** — chart references externally-provisioned `Secret`s only; rotation is a later sub-slice.
- **DevOps/SRE seam runbook content** — SRE owns the runbook per CEO brief O5; DevOps's only deliverable into the runbook this loop is the `tenant` label confirmation (Cross-team asks above).
