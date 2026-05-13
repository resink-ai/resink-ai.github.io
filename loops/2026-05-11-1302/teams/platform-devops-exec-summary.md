---
layout: default
title: platform-devops Exec Summary — 2026-05-11-1302
date: 2026-05-11
status: active
type: exec-summary
loop: 2026-05-11-1302
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/devops
  date: 2026-05-11
  status: active
  loop: 2026-05-11-1302
  links: parent: board/okrs/2026-05-11-1302-ceo-brief.md
-->
{% raw %}

# DevOps Exec Summary — 2026-05-30 (paused team — review-ack)

**Loop status:** paused per CEO brief 2026-05-30. No build deliverables this loop; single light "review the resink-core docs PR" ask.

## Review-ack

Read `repos/resink-ai/resink-core/CLAUDE.md` + `docs/{architecture,concepts,user-guide,module-catalog}.md`. DevOps's surface is **accurately represented.** Specifics checked:

- **`module-catalog.md` § `deploy/charts/nanofab-supervisor/`** correctly identifies DevOps as the owner of the Helm 3 chart skeleton, correctly counts the **15 files** (8 templates: `_helpers.tpl`, `configmap.yaml`, `deployment.yaml`, `role.yaml`, `rolebinding.yaml`, `secret.yaml`, `service.yaml`, `serviceaccount.yaml`; 2 minikube values files: `minikube-mvp.yaml`, `minikube-bluegreen.yaml`; plus `Chart.yaml`, `values.yaml`, `Dockerfile`, `Makefile`, `README.md`), correctly characterizes the gate state (`helm lint --strict`, `helm template`, `helm install --dry-run` all exit 0), and correctly defers the minikube smoke to loop 2026-05-11-1631. Entry point `Chart.yaml` is correct.
- **`architecture.md` § "System shape"** correctly summarizes the chart skeleton ("15 files total (8 templates under `templates/`, 2 minikube values files under `values/`, plus `Chart.yaml`, `values.yaml`, `Dockerfile`, `Makefile`, `README.md`)") and explicitly attributes the deferred minikube smoke to DevOps as a 2026-06-06 follow-up.
- **`architecture.md` § "Named deviations"** correctly carries "DevOps minikube smoke" with owner DevOps and lift target loop 2026-05-11-1631.
- **`CLAUDE.md` § "Tech stack"** correctly cites Helm 3, names all three gate commands (`helm lint --strict`, `helm template`, `helm install --dry-run` exit 0), and correctly defers the minikube smoke.
- **`CLAUDE.md` § "Common commands"** carries the three Helm chart gates (lint/template/install --dry-run) using `values/minikube-mvp.yaml` — matches the on-disk overlay.
- **`module-catalog.md` § `crates/nanofab-coordinator/`** correctly notes "DevOps treats the `accepted, version=...` line as the deploy-gate signal" — accurate characterization of the gate semantic.
- **No claim is made about `/healthz`, SIGTERM drain, env-var convention, or `nodePluginManifestUri` semantics** in resink-core's docs — correctly, since these are unresolved hand-off responses pending resink-core's post-dlopen-restoration work (not DevOps deliverables).

No corrections needed. The chart's MVP shape (`restartPolicy: Never`, `healthz.enabled=false`, CLI flags rather than env vars) is correctly absent from the "what works today" docs and is also correctly out-of-scope for the HTML capabilities report per CEO brief KR3.2's "real Kafka, real KV, multi-tenant, hot-swap" not-in-scope clause.

## Carryover unchanged

- **Minikube smoke execution (OKR KR1.3 mechanical run)** — target authored; carries to loop 2026-05-11-1631 pending docker/minikube install on the dev machine **or** pivot to a shared CI runner / Docker-Desktop-on-Mac path. Acceptance bar unchanged: `make minikube-smoke` exits 0; deployed supervisor's `trace.jsonl` byte-identical to local `make mvp-loop` trace via `cmp` / `sha256sum`; `helm uninstall` residual-free.
- **Supervisor-side hand-off responses from resink-core (5 items)** — Dockerfile ownership, `/healthz` + `/readyz` on port 9090, SIGTERM drain semantics, env-var convention vs. CLI flags, `nodePluginManifestUri` runtime semantics. **Post-dlopen-restoration work for resink-core, not DevOps deliverables**; chart absorbs the responses once resink-core surfaces them (likely 2026-06-06 + 2026-06-13 per ADR-2026-05-16-001's multi-loop plan).
- **DE hand-off items** — Kafka SASL secret-key layout, bootstrap-servers source-of-truth, min-3-entries enforcement boundary. Forward-compat `kafkaBootstrap` + `kafkaSaslSecretRef.*` axes already wired. Carries.
- **Frontmatter-lint CI script** — deferred again per CEO brief (now fourth consecutive loop). Manual ADR-2026-05-08-001 enforcement continues. Retro flags drift if any. Carries.
- **Supervisor-binary resource-budget doc** — still deprioritized; sized once a real (non-sim) workload runs. Carries.

## Asks

**Board action on KR1.3 closure path.** The minikube smoke has now carried two loops (2026-05-23 → 2026-05-30 → 2026-06-06) on the same dev-machine-toolchain blocker. Filing the explicit board ask: **approve docker + minikube install on the IC's dev machine, OR pivot to Docker-Desktop-on-Mac (the smoke target rewrites trivially), OR provision a shared CI runner with docker.** A board decision in either direction unblocks KR1.3 closure in loop 2026-05-11-1631; continued deferral risks a third-loop slip. (Filed as a single ask to roll into the CEO consolidation 2026-05-30.)
{% endraw %}
