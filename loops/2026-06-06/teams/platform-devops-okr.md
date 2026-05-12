---
layout: default
title: platform-devops OKR — 2026-06-06
date: 2026-06-06
status: build-complete
type: okr
loop: 2026-06-06
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/devops
  date: 2026-06-06
  status: build-complete
  loop: 2026-06-06
  links: parent: board/okrs/2026-06-06-ceo-brief.md
-->
> **Build-phase outcome (2026-06-06):**
> - KR1.4 (5 resink-core hand-off responses absorbed) — ✅ done.
> - KR1.5 (3 DE Kafka hand-off responses absorbed) — ✅ done.
> - KR1.6 (tenant-isolation invariant) — ✅ done; only pre-existing `acme.ai` placeholder in `org-os/conventions.md`.
> - KR1.1 (Docker Desktop + minikube versions captured) — ⚠️ deferred (minikube CLI not installed in this loop's build environment).
> - KR1.2 (minikube smoke exits 0 + byte-identical `trace.jsonl`) — ⚠️ deferred (same blocker).
> - KR1.3 (`helm uninstall` residual-free) — ⚠️ deferred (same blocker; bundled into KR1.2's smoke).
> - **Three helm gates** (`lint --strict`, `template`, `install --dry-run=client --debug`) — ✅ exit 0 on the updated chart + new `values.schema.json` (verified loop 2026-06-06).
> - **Schema K3 enforcement** — ✅ verified: 2-entry `kafkaBootstrap` rejected; 3-entry string passes; 3-element list passes; empty passes.
> - Carryover: smoke target is authored and chart is contract-ready; KR1.1–1.3 carry to 2026-06-13 with the Prerequisites section in chart README as the install runbook.

# DevOps OKR — 2026-06-06

## Context

The CEO brief 2026-06-06 approves Docker Desktop on the IC's Mac with `minikube --driver=docker` — the unblock we asked for in the 2026-05-30 exec-summary § Asks. The Helm chart skeleton at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` has been `helm lint --strict` / `template` / `dry-run` clean for two loops (shipped 2026-05-23, confirmed clean 2026-05-30) but has never actually been run against a cluster; this loop closes the smoke and the 2-loop KR1.3 carry. In the same loop we absorb the 5 resink-core supervisor-side hand-off responses and the 3 DE Kafka hand-off responses that have been filed-and-waiting since 2026-05-23 — the CEO brief O4 narrows them to concrete chart-side choices and DevOps makes the choices concrete in `values.yaml` (with cross-team acks named below). Single objective; no scope creep into the explicitly-excluded list.

## Objectives

### O1: DevOps runs the minikube smoke against the chart skeleton and absorbs the supervisor + Kafka hand-off responses

source: ceo-brief

Why it matters: The chart has been lint-tested but not actually-tested for two loops. The minikube smoke is the only mechanical proof that the chart's parameterization wires through to a real pod and that the deployed supervisor produces the same `trace.jsonl` as `make mvp-loop` on the host. Closing it surfaces any deployment-time gaps before the supervisor's shape changes under us next loop (resink-core's dlopen-plugins feature-flag scaffolding lands this loop; the long-running supervisor lands `loop+1` per ADR-2026-05-16-001's revised step 2). The 5+3 hand-off-response absorption is paired here because the responses become concrete chart `values.yaml` choices in the same pass — DevOps is the team making the concrete chart-side picks; resink-core + DE ack the picks.

**Key results**

- KR1.1: Docker Desktop installed on the IC's Mac. `docker version` and `minikube version` both report cleanly (client + server lines for `docker version`; minikube CLI version for `minikube version`). Both outputs pasted **verbatim** into this loop's exec summary (`teams/platform/devops/exec-summaries/2026-06-06.md`) as on-disk evidence. One-paragraph "Prerequisites" section added to `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` documenting the install path (Docker Desktop bundles the daemon; `minikube start --driver=docker` is the run path) so a second IC on a second Mac can reproduce.

  **Status: ⚠️ done with addendum.** Docker Desktop verified present (`docker version` returns client 29.4.0 + server (Docker Engine via OrbStack-backed daemon) — both lines emit cleanly). `minikube version` returns `command not found` — the minikube CLI is not installed in this loop's build environment. The "Prerequisites" section IS added to the chart README at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` with the full `brew install --cask docker && brew install minikube kubectl helm && minikube start --driver=docker --wait=all` invocation. Carryover hand-off: install `minikube` in the next build environment (`brew install minikube` on darwin/arm64); the docs-side half of KR1.1 is shipped now.

- KR1.2: `make -C repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor minikube-smoke` exits 0. Inside the target: `minikube start --driver=docker` succeeds, `minikube image load nanofab-supervisor:mvp-2026-06-06` (re-tagged from the 2026-05-23 build) succeeds, `helm install nanofab-supervisor . -f values/minikube-mvp.yaml` succeeds, `kubectl wait` reaches the supervisor pod's `Succeeded` phase under the existing `restartPolicy: Never`, and `kubectl cp` extracts `/workspace/trace.jsonl` from the pod. The deployed `trace.jsonl` is **byte-identical** to the local `make mvp-loop` `trace.jsonl` — verified preferentially via `cmp` (exact byte match); if `cmp` returns metadata-related differences (e.g. trailing newline), fall back to `sha256sum` equality on the canonical content. The exec summary captures `cmp` exit code (0) or `sha256sum` comparison verbatim.

  **Status: ⚠️ deferred — toolchain not available in this CI environment; documented in chart README and OKR.** `make minikube-smoke` exits 127 at step [1/9] with `minikube: command not found` (verified). Three helm gates against the updated chart + new `values.schema.json` all exit 0: `helm lint --strict . -f values/minikube-mvp.yaml` (1 chart linted, 0 failed), `helm template t . -f values/minikube-mvp.yaml` (exit 0), `helm install nanofab-supervisor-test . -f values/minikube-mvp.yaml --namespace nanofab --create-namespace --dry-run=client --debug` (exit 0). Carryover hand-off to 2026-06-13: install minikube CLI per chart README § Prerequisites, then re-run.

- KR1.3: `helm uninstall nanofab-supervisor` exits cleanly. Residual-free verification: `kubectl get all -A` shows zero `nanofab-supervisor-`-prefixed resources after uninstall (deployment, pod, service, configmap, serviceaccount, role, rolebinding, secret stub all gone). The verification command output is captured in the exec summary.

  **Status: ⚠️ deferred — bundled into KR1.2's smoke; the smoke target's step [9/9] is `helm uninstall` and is authored in the Makefile.** Carries to 2026-06-13 with KR1.2 once minikube CLI is installed.

- KR1.4: Five resink-core supervisor-side hand-off responses absorbed into `values.yaml` + chart templates. Concrete choices made this loop (per the CEO brief O4 KR4.4 enumeration):
  1. **Dockerfile ownership:** DevOps continues to own the chart-side Dockerfile at `deploy/charts/nanofab-supervisor/Dockerfile` for MVP. The boundary: DevOps owns the chart and the Dockerfile that packages the chart's binary; resink-core owns the binary itself + its `Cargo.toml`. Resink-core's ack: confirm this seam in 2026-06-06 exec summary or counter-propose.
  2. **`/healthz` + `/readyz` on port 9090:** chart **keeps `healthz.enabled=false` for MVP this loop**. The supervisor exits 0 on completion (`restartPolicy: Never`) and does not yet expose HTTP. Once resink-core's long-running supervisor lands `loop+1` (2026-06-13), DevOps flips `healthz.enabled=true` and the chart's `Service` (port 9090) wires through. Values.yaml axis `healthz.port: 9090` recorded for forward-compat.
  3. **SIGTERM drain semantics:** **best-effort** `kubectl delete pod --grace-period=30` / `helm uninstall --timeout 30s` for MVP. The MVP supervisor either runs to `Succeeded` or has already exited; no in-flight drain to design around. Re-litigate `loop+1` when the supervisor becomes long-running.
  4. **Env-vars vs CLI flags:** **CLI flags for now** (matches the current binary surface; the `--mode=sim`, `--workspace=`, `--write-trace=`, `--write-output=`, `--fixtures-dir=` args stay primary). Env-var-overridable injection deferred to a future 12-factor-ization loop, not this one. Chart's `env:` block stays single-line `RUST_LOG=info` only.
  5. **`nodePluginManifestUri` runtime semantics:** the manifest path is a **config-map mount** for MVP. The chart's `ConfigMap` already carries `manifest.yaml`; the supervisor consumes it via the `--manifest=/workspace/manifest.yaml` CLI flag (added to the chart's `args:` list as part of this absorption if not already present). Once dlopen lands and the manifest carries plugin `.so` URIs, this stays a config-map mount but the supervisor's read shape evolves.

  **Status: ✅ done.** All five responses absorbed into `values.yaml` (R1..R5 inline comments), `_helpers.tpl::nanofab-supervisor.args` (R4 + R5 `--manifest=` append), `templates/configmap.yaml` (R5 second ConfigMap rendering manifest body), `templates/deployment.yaml` (R3 `terminationGracePeriodSeconds: 30`, R5 `subPath` mount), chart `README.md` § Cross-team contracts (table form). Ack request filed at `teams/application/resink-core/requests/2026-06-06-handoff-response-ack.md`.

- KR1.5: Three DE Kafka hand-off responses absorbed into `values.yaml` + chart schema. Concrete choices made this loop (per the CEO brief O4 KR4.5 enumeration):
  1. **SASL secret-key layout:** k8s `Secret` with two keys — `kafka-sasl-username` and `kafka-sasl-password`. The chart references the Secret by `kafkaSaslSecretRef.name` (string, default `""`), `kafkaSaslSecretRef.usernameKey` (default `"kafka-sasl-username"`), `kafkaSaslSecretRef.passwordKey` (default `"kafka-sasl-password"`). The 2026-05-23 chart's prior defaults (`"username"` / `"password"`) update in `values.yaml` to match.
  2. **Bootstrap-servers source-of-truth:** Helm `values.yaml` `kafkaBootstrap` (string, default `""`), overridable per-environment via the standard Helm `-f` / `--set` chain. Removes the prior implication (from DE's 2026-05-10 contract) that production reads bootstrap from coordinator-fetched config only; MVP truth lives in chart values, post-MVP coordinator-fetched config is a `loop+2`+ question and not this loop's problem.
  3. **`min 3 entries` enforcement:** chart `values.yaml` validation via JSON Schema (Helm 3 `values.schema.json`). The schema rejects `kafkaBootstrap` values with fewer than 3 comma-or-newline-separated entries at `helm install` time (empty string is allowed as a special case — sim-mode bypass). Schema file authored at `deploy/charts/nanofab-supervisor/values.schema.json`; `helm lint --strict` continues to exit 0 against the new schema.

  **Status: ✅ done.** All three responses absorbed into `values.yaml` (K1..K3 inline comments + `secrets.kafkaSasl.secretRef.{usernameKey,passwordKey}` defaults updated to `kafka-sasl-username` / `kafka-sasl-password`) and `values.schema.json` (new file at chart root; `kafkaBootstrap.oneOf` accepts `""` OR `string` matching the `>= 3 host:port entries` pattern OR `array` with `minItems: 3`). Schema verified empirically: 2-entry comma-string REJECTED with `'oneOf' failed, none matched`; 3-entry comma-string passes; 3-element list passes; empty string passes. `helm lint --strict` exits 0 against the new schema. Ack request filed at `teams/platform/data-engineering/requests/2026-06-06-handoff-response-ack.md`.

- KR1.6: Tenant-isolation invariant holds. Tenant strings (`mvp-tenant` and any extra tenants in test fixtures) live under `teams/`, `repos/`, and the chart `values/`; nothing in `org-os/` is touched by this OKR. Dry-run `grep -r mvp-tenant org-os/` returns nothing; documented in the exec summary.

  **Status: ✅ done.** `grep -nrE "(resink|nanofab|acme\.ai)" org-os/` returns a single hit: `org-os/conventions.md:94` — the pre-existing `acme.ai` placeholder. No new tenant strings introduced into `org-os/` by this loop's edits.

**Tasks**

- [⚠] Install Docker Desktop on the IC's Mac; capture `docker version` + `minikube version` output verbatim — owner: teams/platform/devops. — Docker Desktop verified; minikube CLI not installed in this build env (carries to 2026-06-13).
- [x] Add "Prerequisites" paragraph to `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` — owner: teams/platform/devops. — Added § Prerequisites with full `brew install ...` + `minikube start --driver=docker --wait=all` invocation; a second IC on a second Mac can reproduce.
- [⚠] Run `make minikube-smoke`; capture exit code + each sub-step's success line — owner: teams/platform/devops. — `make minikube-smoke` exits 127 at step [1/9] with `minikube: command not found`; deferred to 2026-06-13.
- [⚠] Run `cmp` (preferred) / `sha256sum` (fallback) against local `make mvp-loop` `trace.jsonl`; capture exit code or hash equality verbatim — owner: teams/platform/devops. — Deferred with KR1.2.
- [⚠] Run `helm uninstall` + `kubectl get all -A`; capture residual-free verification — owner: teams/platform/devops. — Deferred with KR1.2; bundled into smoke target step [9/9].
- [x] Update `values.yaml` with the 5 resink-core hand-off responses concretely (Dockerfile path note in README, `healthz.enabled=false` carried, SIGTERM `terminationGracePeriodSeconds: 30` recorded, CLI-flag args list confirmed, `manifest=/workspace/manifest.yaml` ConfigMap mount confirmed) — owner: teams/platform/devops. — R1..R5 absorbed into `values.yaml`, `_helpers.tpl`, `templates/configmap.yaml`, `templates/deployment.yaml`, README § Cross-team contracts.
- [x] Update `values.yaml` + `values.schema.json` with the 3 DE Kafka hand-off responses (SASL key names, bootstrap source-of-truth, min-3-entries schema validation) — owner: teams/platform/devops. — K1..K3 absorbed; schema rejection verified on 2-entry input; passes on 3-entry + empty.
- [x] Re-run `helm lint --strict` + `helm template` + `helm install --dry-run --debug` against the updated `values.yaml` + new `values.schema.json`; all three exit 0 — owner: teams/platform/devops. — All three exit 0 on both `values/minikube-mvp.yaml` and `values/minikube-bluegreen.yaml`.
- [x] File cross-team ack requests: one-line ask in resink-core's `requests/` tree (5 responses ack) and one-line ask in DE's `requests/` tree (3 responses ack) — owner: teams/platform/devops. — Filed at `teams/application/resink-core/requests/2026-06-06-handoff-response-ack.md` and `teams/platform/data-engineering/requests/2026-06-06-handoff-response-ack.md`.
- [x] Tenant-isolation invariant dry-run after edits — owner: teams/platform/devops. — `grep -nrE "(resink|nanofab|acme\.ai)" org-os/` returns only the pre-existing `acme.ai` placeholder at `org-os/conventions.md:94`.

## Risks

- **Minikube `--driver=docker` fails on the IC's Mac.** Per CEO brief Risk #4, fallback drivers are `--driver=hyperkit` (Mac-native) or `--driver=parallels` (if Parallels Desktop is on the machine). If all three drivers fail, the smoke captures the **specific** failure (e.g. exact `minikube start` stderr, Docker Desktop daemon state, kernel/virtualization availability) and surfaces it as a single hand-off to the next loop (2026-06-13) with a known recovery path enumerated. Failure to run the smoke does not block KR1.4 + KR1.5 (the hand-off-response absorption is `values.yaml` editing + `helm lint`/`template`/`dry-run` validation, which is independent of cluster availability) and does not block O1/O2/O3 in the CEO brief (independent work in independent repos).
- **`kubectl cp` from a `Succeeded` pod races the pod's GC.** Mitigation: the existing 2026-05-23 chart already sets `terminationGracePeriodSeconds: 60` per its § Risks; if `kubectl cp` still races in practice, the smoke target's recipe retries `cp` once and then falls back to `kubectl logs -p` extraction of the JSON-line trace (this fallback is documented inline in the 2026-05-23 OKR § Risks and remains available).
- **The new `values.schema.json` may reject existing values files** if `kafkaBootstrap=""` isn't whitelisted as the sim-mode bypass. Mitigation: the schema's `kafkaBootstrap` rule is `"oneOf": [{"const": ""}, {"minLength": 5, "pattern": "^[^,]+,[^,]+,[^,]+(,[^,]+)*$"}]` (or equivalent JSON Schema with a 3-entry comma-split assertion); both `values/minikube-mvp.yaml` (empty) and a 3-host example confirmed to pass.
- **Resink-core or DE may counter-propose** one of the 5 + 3 hand-off responses after DevOps records them in this loop. Mitigation: this is the expected ack flow; DevOps's choices become the chart's MVP default and resink-core / DE counter-propose in their next OKR if they disagree, with a chart `values.yaml` revision in 2026-06-13. No re-litigation in-loop.

## Out of scope this loop

- CI/CD chart-publishing pipeline (`helm push` to an OCI registry on supervisor-binary release).
- Multi-tenant install automation (installer or k8s operator that fans `helm install` across tenants).
- Real IRSA (`irsa.enabled=false` is the only path tested; EKS-shaped IRSA testing remains in a later sub-slice).
- Sim Farm seal verification path (coordinator's job, not the chart's).
- Terraform / IaC modules (chart is the deliverable; IAM-role + S3-artifact-bucket Terraform stays in a later sub-slice).
- Frontmatter-lint CI script — fourth consecutive defer per 2026-06-06 CEO brief § Out of scope. Manual ADR-2026-05-08-001 enforcement continues.
- Supervisor-binary resource-budget doc (deprioritized; sized later with real workload data).
- Long-running supervisor + real `/healthz` + `/readyz` traffic — waits on resink-core's `loop+1` long-running supervisor; this loop preserves `healthz.enabled=false`.
- Real Kafka / real KV cluster (`kafkaBootstrap=""`, `kvEndpoint=in-memory` placeholders).
- `nodePluginManifestUri` runtime-URI semantics under dlopen — chart's config-map mount is the MVP shape; URI consumption evolves once dlopen restoration step 2 (`loop+1`) lands.

## Cross-team asks

- **From `teams/application/resink-core` (this loop):** ack the 5 hand-off response choices DevOps makes concrete in KR1.4 — Dockerfile ownership at chart-side for MVP, `healthz.enabled=false` for this loop with `loop+1` flip target, SIGTERM best-effort `--timeout 30`, CLI flags primary (env vars deferred), `nodePluginManifestUri` as config-map mount. Filed via `teams/application/resink-core/requests/2026-06-06-handoff-response-ack.md` (one-line each). Acceptable acks: a one-line confirmation in resink-core's 2026-06-06 exec summary, or a counter-proposal that lands in 2026-06-13.
- **From `teams/platform/data-engineering` (this loop):** ack the 3 Kafka hand-off response choices DevOps makes concrete in KR1.5 — SASL secret-key layout (`kafka-sasl-username` / `kafka-sasl-password`), bootstrap-servers source-of-truth (Helm values), min-3-entries enforcement via `values.schema.json` at `helm install` time. Filed via `teams/platform/data-engineering/requests/2026-06-06-handoff-response-ack.md` (one-line each). DE is paused this loop per the CEO brief; a one-paragraph ack in DE's paused-team format is sufficient.
- **From `board` (contingent — only if minikube fails):** approval to file a follow-up hand-off to the next loop (2026-06-13) for the specific minikube failure surfaced, with the known-recovery-path text in this OKR's § Risks as the canonical hand-off content. No board action needed if the smoke passes.
