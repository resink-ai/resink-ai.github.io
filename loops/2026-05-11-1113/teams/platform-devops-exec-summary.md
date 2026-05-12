---
layout: default
title: platform-devops Exec Summary — 2026-05-11-1113
date: 2026-05-11
status: active
type: exec-summary
loop: 2026-05-11-1113
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/devops
  date: 2026-05-11
  status: active
  loop: 2026-05-11-1113
  links: parent: teams/platform/devops/okrs/2026-05-11-1113-team-okr.md
-->
# DevOps Exec Summary — 2026-05-23

**Headline:** `nanofab-supervisor` Helm chart skeleton shipped (15 files); `helm lint`, `helm template`, and `helm install --dry-run --debug` all exit 0. Minikube smoke target authored but DEFERRED with a recorded blocker (no docker / minikube on the IC's developer machine this loop) — fallback `helm install --dry-run` validation path was executed and passed per OKR § Risks.

First BUILD-mode loop in 4 weeks after a 3-loop plan-only run. The 2026-05-10 scope document was the floor; every artifact named there shipped. The supervisor binary's actual surface (read from `crates/nanofab-supervisor/src/main.rs` + `Cargo.toml`) is narrower than the chart-friendly long-running shape we'd assumed: no `/healthz`, no env-var config, no SIGTERM drain, no exposed ports. The chart papers over the gap with values-gated probes (`healthz.enabled` default `false`), `restartPolicy: Never`, and CLI-flag-driven invocation — explicitly the not-long-running MVP shape. Hand-off asks back to resink-core capture this seam for post-dlopen-restoration work.

## What we shipped

All against OKR O1 (CEO brief O4 in full — KR4.1, KR4.2, KR4.3, KR4.4). `source: ceo-brief`.

### Chart artifacts (15 files; on-branch: `master-2026-05-10-2232`, in `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/`)

- **Chart skeleton root** (5 files): `Chart.yaml` (v0.1.0, appVersion = `mvp-2026-05-23`), `values.yaml` (8 OKR-named parameterization axes + structural axes), `Dockerfile` (multi-stage; `rust:1.83-slim-bookworm` → `debian:12-slim`; non-root UID 65532; pivoted off distroless per OKR § Risks fallback), `Makefile` (targets `help`, `lint`, `template`, `dry-run`, `image`, `minikube-smoke`, `clean`), `README.md` (covers all 8 axes, install paths, `tenant`-label convention, carried-forward exclusion list, hand-off notes).
- **Two named values overlays** (2 files): `values/minikube-mvp.yaml` (`tenant=mvp-tenant`, `dagVersion=v1`, `fleetColor=blue`); `values/minikube-bluegreen.yaml` (same tenant, `fleetColor=green` — proves parameterization wires through). KR1.2.
- **Eight template files** under `templates/`: `_helpers.tpl` (defines `nanofab-supervisor.tenantLabels` + `.fullname` + `.selectorLabels`); `deployment.yaml` (restartPolicy: Never, no probes by default); `service.yaml` (ClusterIP, gated on `.Values.healthz.enabled`); `configmap.yaml` (mode/workspace/manifest paths); `secret.yaml` (stub-only — renders only when no external secret name is supplied; the chart never owns production secrets per serving spec §7.2); `serviceaccount.yaml` (IRSA annotation gated on `.Values.irsa.enabled`); `role.yaml` (namespace-scoped, read own ConfigMap); `rolebinding.yaml` (binds Role to SA). KR1.1.
- **`tenantLabels` helper emits the 8-field label set SRE references in their O5 KR5.3 runbook hand-off**: `tenant`, `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`, `app.kubernetes.io/managed-by`, `helm.sh/chart`, `nanofab.resink.ai/dag-version`, `nanofab.resink.ai/fleet-color`. The strict 3-field `selectorLabels` subset (tenant + name + instance) goes into `Deployment.spec.selector.matchLabels` (immutable post-create, standard Helm convention). KR1.1.

### Validation (KR4.3 / OKR KR1.3 fallback path)

- `helm lint deploy/charts/nanofab-supervisor` (helm 4.1.4): **exit 0**, single cosmetic INFO ("icon is recommended"). `helm lint --strict`: also **exit 0**, first attempt.
- `helm template` with `values/minikube-mvp.yaml`: 190 lines, 6 kinds rendered (`ServiceAccount`, `Secret` stub, `ConfigMap`, `Role`, `RoleBinding`, `Deployment`). With `healthz.enabled=true`: 7 kinds (Service added). With external `secrets.kafkaSasl.secretRef.name` set: 5 kinds (stub Secret suppressed, Deployment gets mounted volume + envFrom).
- `helm install --dry-run --debug nanofab-supervisor . -f values/minikube-mvp.yaml`: **exit 0**, status `Dry run complete`.
- `helm template` with `values/minikube-bluegreen.yaml`: confirms `nanofab.resink.ai/fleet-color: "green"` flips across all 6 rendered kinds — parameterization end-to-end verified.
- Required-tenant assertion fires: `helm template no-tenant .` (no values file) exits non-zero with `Error: .Values.tenant is required` — the helper's `fail` guard works.

### Hand-off documentation (KR4.4 / OKR KR1.4)

- § Hand-off subsection authored in the 2026-05-23 OKR itself (the canonical hand-off text). Five resink-core asks + three DE asks filed. None blocking this loop.
- README's hand-off block restates the asks for chart readers.

## What we didn't ship and why

- **Minikube smoke execution (OKR KR1.3 mechanical run)** — DEFERRED. Reason: minikube + docker daemon are not installed on the IC's developer machine this loop. Per OKR § Risks fallback ("If you can't get minikube working locally in a reasonable budget, FALLBACK: validate the chart with `helm install --dry-run` + `helm template` and document the minikube smoke as a deferred follow-up with a recorded blocker"), the fallback path was executed and passed. The `make minikube-smoke` target itself is shipped and reviewable; the 9-step sequence (start, docker-env, image build, mount, install, wait, cp, cmp, uninstall) is authored. **Carries to 2026-05-30** pending either docker/minikube install on the dev machine or pivot to a shared CI runner / Docker-on-Mac alternative — surfacing as a board ask below.
- **Frontmatter-lint CI script** — DEFERRED again per CEO brief § Out of scope. Third consecutive loop. Manual ADR-2026-05-08-001 enforcement continues; retro reviews drift.
- **Supervisor-binary resource-budget doc** — still DEPRIORITIZED per 2026-05-10 status. Sized once a real (non-sim) workload runs; can't benchmark without one.

## Surprises

- **The supervisor binary's actual surface is narrower than chart-friendly.** Reading `crates/nanofab-supervisor/src/main.rs` + `Cargo.toml` during build confirmed: no `/healthz`, no `/readyz`, no env-var config (all `clap::Parser` CLI flags), no exposed ports, no SIGTERM drain logic, no long-running HTTP server. The chart had to default `healthz.enabled=false` and use `restartPolicy: Never` — the "not-long-running MVP shape." Anticipated at plan-time (§ Hand-off was authored in the OKR before build); confirmed by source-reading and now flagged back to resink-core for post-dlopen-restoration work (loops 2026-05-11-1302 + 2026-06-06).
- **`helm lint --strict` passed on the first attempt.** No template iteration needed for syntax / convention compliance. Cosmetic INFO ("icon is recommended") only.
- **Distroless pivot to `debian:12-slim` for the Dockerfile stage-2 base.** OKR § Risks flagged distroless as a possible glibc-symbol risk for parquet/arrow deps; the conservative MVP default is debian:12-slim. The chart's image tag is unchanged either way.
- **Dockerfile location pivot.** Authored at `deploy/charts/nanofab-supervisor/Dockerfile`, not `crates/nanofab-supervisor/Dockerfile` as KR1.3 originally named. DevOps's preference is for resink-core to own the binary's Dockerfile post-MVP given the path-dep coupling to the codegen output; the under-the-chart location is the MVP placement until resink-core's hand-off ask resolves.
- **No board ping needed on cross-team text.** All five resink-core asks + three DE asks are about *post-dlopen-restoration* shape (2026-05-30 + 2026-06-06 loops), not this-loop blockers. The chart's MVP `restartPolicy: Never` + values-gated probes mean we don't depend on any of the asks resolving to ship the skeleton.

## Asks

### To `teams/application/resink-core` (next-loop 2026-05-11-1302 deliverable; not blocking)

Confirm the five hand-off items in OKR § Hand-off to resink-core, all aimed at post-dlopen-restoration work:

1. **Dockerfile ownership** — DevOps wrote a working MVP Dockerfile under the chart; preference is resink-core owns the binary's Dockerfile post-MVP given path-dep coupling.
2. **Healthz / readyz probe surface** — DevOps proposes port `9090` once the supervisor becomes long-running.
3. **SIGTERM graceful-shutdown drain semantics** — chart's `terminationGracePeriodSeconds=30s` is a placeholder; resink-core names the actual drain behavior.
4. **Env-var convention vs. CLI flags** — DevOps's preference: keep CLI flags primary; secrets injected via mounted Secret files referenced by `--*-credentials-file=<path>`, not env vars.
5. **`nodePluginManifestUri` runtime semantics** — chart axis is wired; runtime consumer shape (env var? CLI flag? coordinator-fetched config?) decided once dlopen restores.

### To `teams/platform/data-engineering` (next-loop 2026-05-11-1302 deliverable; not blocking)

Confirm the three hand-off items in OKR § Hand-off to DE:

1. **AWS Secrets Manager key layout for Kafka SASL** — DevOps proposes JSON blob `{"username","password","mechanism":"SCRAM-SHA-512"}` at path `/tenants/<id>/ingest/kafka_sasl`.
2. **Bootstrap-servers source-of-truth** — DevOps reads DE contract §6 as "supervisor reads from coordinator at runtime, not env var"; chart's `kafkaBootstrap` is a sim-mode convenience axis only.
3. **`min 3 bootstrap entries` enforcement** — supervisor-side at startup; chart does NOT double-enforce. DE confirms.

### To `board` (this loop, follow-up)

**Approve docker + minikube install on the IC's developer machine for next-loop smoke execution, OR pivot the smoke target to a containers-via-Docker-on-Mac alternative.** Without either, OKR KR1.3's mechanical proof carries indefinitely. Recommendation: install docker + minikube on dev machine for the 2026-05-30 loop — alignment with the original OKR design and reusable for future smoke targets.

## Metrics (KR-by-KR)

| KR | Description | Status |
|---|---|---|
| **KR1.1** | Helm chart skeleton at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` with 6 chart objects + `_helpers.tpl`; `helm lint` exit 0; `helm install --dry-run` exit 0 | **shipped** (on-branch: `master-2026-05-10-2232`; 8 templates incl. helpers + Chart.yaml; `helm lint --strict` exit 0; dry-run exit 0) |
| **KR1.2** | `values.yaml` with 8 OKR-named axes + minikube defaults; `values/minikube-mvp.yaml` + `values/minikube-bluegreen.yaml` | **shipped** (on-branch: `master-2026-05-10-2232`; 8 axes wired; both overlays render; bluegreen flips `fleet-color` across all 6 kinds) |
| **KR1.3** | Minikube smoke mechanically verifiable; `make minikube-smoke` exits 0; trace byte-comparison passes; helm uninstall residual-free | **deferred** (target authored on-branch: `master-2026-05-10-2232`; smoke execution carries to 2026-05-30; FALLBACK validation `helm install --dry-run` + `helm template` executed and passed per OKR § Risks) |
| **KR1.4** | § Hand-off subsection naming consumer-side constraints to resink-core + DE; cross-linked from asks | **shipped** (on-branch: `master-2026-05-10-2232`; 5 resink-core asks + 3 DE asks filed in OKR § Hand-off; restated in README; restated as cross-team asks above) |
| **KR1.5** | README at `deploy/charts/nanofab-supervisor/README.md` documenting axes, smoke target, label convention, exclusion list | **shipped** (on-branch: `master-2026-05-10-2232`; 7.8 KB README; covers all 8 axes, install paths, tenant-label convention, 8-item carried-forward exclusion list, hand-off notes, file-layout tree) |

**KR tally:** 4 shipped / 1 deferred-with-fallback / 0 dropped.

**Exit codes recorded (KR4.3 mechanical proof):** `helm lint` = 0; `helm lint --strict` = 0; `helm template values/minikube-mvp.yaml` = 0 (190 lines, 6 kinds); `helm template values/minikube-bluegreen.yaml` = 0 (fleet-color flip verified); `helm install --dry-run --debug` = 0; missing-tenant assertion = non-zero (correct).

**Loop adherence:** on-track. First build-mode loop after 4 weeks of plan-only landed mechanically without scope creep; only deferral is the smoke execution (recorded blocker, fallback executed).
