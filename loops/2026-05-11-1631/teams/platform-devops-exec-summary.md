---
layout: default
title: platform-devops Exec Summary — 2026-05-11-1631
date: 2026-05-11
status: active
type: exec-summary
loop: 2026-05-11-1631
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/devops
  date: 2026-05-11
  status: active
  loop: 2026-05-11-1631
  links: parent: board/okrs/2026-05-11-1631-ceo-brief.md
-->
# DevOps Exec Summary — 2026-06-06

**Headline:** Five resink-core supervisor-side (R1..R5) + three DE Kafka (K1..K3) hand-off responses absorbed into `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` (`values.yaml`, `_helpers.tpl`, `templates/configmap.yaml`, `templates/deployment.yaml`, new `values.schema.json`, README § Prerequisites / § Limitations / § Cross-team contracts). All three Helm gates exit 0 on both overlays; K3 schema rejection empirically verified. Minikube smoke (KR1.2 + KR1.3) **DEFERRED a third loop** — Docker Desktop is present on this build host via OrbStack, but the `minikube` CLI is not installed (`make minikube-smoke` exits 127 at step [1/9] with `minikube: command not found`). Recovery is a single command, `brew install minikube`, documented in chart README § Prerequisites.

## KR outcomes

- **KR1.1 — Docker + minikube versions captured:** **PARTIAL.** `docker version` returns clean client + server lines (client 29.4.0; daemon via OrbStack-backed Docker Engine). `minikube version` returns `command not found`. README § Prerequisites shipped with full `brew install --cask docker && brew install minikube kubectl helm && minikube start --driver=docker --wait=all` invocation; the docs-side half of KR1.1 is done now.
- **KR1.2 — `make minikube-smoke` exits 0 + byte-identical `trace.jsonl`:** **DEFERRED.** Toolchain incomplete (no `minikube` CLI). Smoke target's authored shape is unchanged; deferred recovery path documented in chart README. Carries to 2026-06-13.
- **KR1.3 — `helm uninstall` residual-free verification:** **COULDN'T-VERIFY.** Depends on KR1.2 having run; bundled into the Makefile's step [9/9]. Carries to 2026-06-13 with KR1.2.
- **KR1.4 — 5 resink-core supervisor-side hand-off responses absorbed:** **PASS.** R1..R5 absorbed into `values.yaml` (inline-commented), `_helpers.tpl` (R4 + R5 `--manifest=` arg append), `templates/configmap.yaml` (R5 second ConfigMap for manifest body), `templates/deployment.yaml` (R3 `terminationGracePeriodSeconds: 30`, R5 `subPath` mount), README § Cross-team contracts (table). Concrete choices: R1 Dockerfile owned chart-side; R2 `healthz.enabled=false` for MVP (flip target `loop+1`); R3 `terminationGracePeriodSeconds: 30` best-effort; R4 CLI flags primary; R5 `nodePluginManifestUri` as ConfigMap mount via `--manifest=/workspace/manifest.yaml`. Ack request filed at `teams/application/resink-core/requests/2026-06-06-handoff-response-ack.md`.
- **KR1.5 — 3 DE Kafka hand-off responses absorbed:** **PASS.** K1..K3 absorbed into `values.yaml` (`secrets.kafkaSasl.secretRef.{usernameKey,passwordKey}` defaults updated to `kafka-sasl-username` / `kafka-sasl-password`) and new `values.schema.json` (`kafkaBootstrap.oneOf` accepts `""` OR `>=3`-entry comma-string OR list with `minItems: 3`). K3 verified empirically: 2-entry comma-string REJECTED (`'oneOf' failed, none matched`); 3-entry comma-string passes; 3-element list passes; empty string passes. Ack request filed at `teams/platform/data-engineering/requests/2026-06-06-handoff-response-ack.md`.
- **Tenant-isolation invariant:** **PASS.** `grep -nrE "(resink|nanofab|acme\.ai)" org-os/` returns a single pre-existing hit (`org-os/conventions.md:94` `acme.ai` placeholder). No new tenant strings in `org-os/`. Dry-run also clean on `helm install --dry-run=client --debug` output — only chart-tree paths emitted.

## Shipped this loop

- `values.yaml` rewritten with R1..R5 + K1..K3 inline comments; `pod.terminationGracePeriodSeconds: 30` (R3); `nodePluginManifest.{mountPath,content}` axes added (R5).
- `values.schema.json` NEW at chart root — `kafkaBootstrap.oneOf` enforces empty OR `>=3`-entry shape for both comma-string and list inputs (K3).
- `templates/_helpers.tpl` extended: `nanofab-supervisor.args` appends `--manifest=<mountPath>` when set (R4 + R5); new `manifestKey` / `manifestDir` helpers.
- `templates/configmap.yaml` extended: renders second ConfigMap `<release>-manifest` carrying manifest body keyed by basename of mount path (R5).
- `templates/deployment.yaml` extended: `subPath` volumeMount + `node-plugin-manifest` volume (R5); 30s grace flows via values (R3).
- `values/minikube-mvp.yaml` + `values/minikube-bluegreen.yaml`: `terminationGracePeriodSeconds: 30` + inline R2/R3 carryover comments.
- `README.md`: § Prerequisites (Docker Desktop + minikube `brew install` runbook), § Limitations (R2 deferral + smoke deferral + R4 env-var note + schema-bypass note), § Cross-team contracts (two tables enumerating R1..R5 + K1..K3).
- Cross-team ack requests: `teams/application/resink-core/requests/2026-06-06-handoff-response-ack.md`, `teams/platform/data-engineering/requests/2026-06-06-handoff-response-ack.md`.
- On-disk gate evidence: `helm lint --strict` (1 chart linted, 0 failed) ×2 overlays; `helm template` exit 0 ×2 overlays; `helm install --dry-run=client --debug` exit 0; K3 schema verification (2-entry rejected; 3-entry / 3-element / empty all pass).

## Cross-team requests filed

- **resink-core:** ack the 5 supervisor-side hand-off response choices (R1..R5) — `teams/application/resink-core/requests/2026-06-06-handoff-response-ack.md`. Acceptable acks: one-line confirmation in resink-core's 2026-06-06 exec summary OR counter-proposal landing 2026-06-13.
- **DE (paused this loop):** ack the 3 Kafka hand-off response choices (K1..K3) — `teams/platform/data-engineering/requests/2026-06-06-handoff-response-ack.md`. One-paragraph paused-team-format ack sufficient.

## Asks for CEO consolidation

**Three-loop deferral pattern on the minikube smoke — same blocker shape as workspace promotion.** KR1.2 + KR1.3 have now slipped 2026-05-23 → 2026-05-30 → 2026-06-06, all on a single-human-action-external-to-the-loop-tree blocker (dev-machine toolchain install). The 2026-05-30 § Asks asked for board approval on Docker Desktop, and the 2026-06-06 CEO brief granted it — yet this loop still slipped because Docker arrived on the host via OrbStack rather than Docker Desktop, and the minikube CLI was never separately installed. The single-command recovery (`brew install minikube`) is documented and not in dispute; the pattern is what bears the cost. **Request: treat the missing `minikube` CLI as a same-class board-action ticket (siblng to `board/actions/2026-06-06-001-create-resink-core-github-remote.md`) with a `due: 2026-06-13` forcing function, OR accept that the chart skeleton is shippable without the in-loop smoke and rely on production-time smoke at the runtime k8s deploy.** This is a retro candidate — the loop's mechanical-toolchain-blocker pattern resembles P1's workspace-promotion pattern from 2026-05-30 closely enough that the two should be considered together at the 2026-06-06 retro.

## Tenant-isolation invariant — held

- `grep -nrE "(resink|nanofab|acme\.ai)" org-os/` → single pre-existing hit at `org-os/conventions.md:94` (`acme.ai` placeholder). No new tenant strings introduced.
- `helm install nanofab-supervisor-test . -f values/minikube-mvp.yaml --namespace nanofab --create-namespace --dry-run=client --debug` exit 0; rendered manifests confine `nanofab-supervisor` strings to chart-tree paths and `nanofab` namespace metadata (tenant-scoped, never `org-os/`).
- Chart's two values overlays (`minikube-mvp`, `minikube-bluegreen`) live under `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/values/` — tenant tree, not `org-os/`.

## Carrying into 2026-06-13

- **KR1.1 minikube version capture + KR1.2 smoke exit 0 + KR1.3 residual-free uninstall** — install `minikube` per chart README § Prerequisites (`brew install minikube`), then `make -C repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor minikube-smoke`. Acceptance bar unchanged: byte-identical `trace.jsonl` via `cmp` / `sha256sum`; `helm uninstall` residual-free.
- **R2 `/healthz` + `/readyz` flip target** — when resink-core's `loop+1` long-running supervisor lands (2026-06-13 per ADR-2026-05-16-001 revised step 2), DevOps flips `healthz.enabled=true` in `values.yaml` defaults; the `Service` block then renders. Forward-compat values already wired (`port: 9090`, `livenessPath: /healthz`, `readinessPath: /readyz`).
- **Frontmatter-lint CI script** — fourth consecutive defer; manual ADR-2026-05-08-001 enforcement continues.
- **Supervisor-binary resource-budget doc** — deprioritized; sized when a real (non-sim) workload runs.
