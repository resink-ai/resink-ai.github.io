---
layout: default
title: platform-devops Exec Summary — 2026-05-12-0645
date: 2026-05-12
status: active
type: exec-summary
loop: 2026-05-12-0645
owner: teams/platform/devops
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/devops
  date: 2026-05-12
  status: active
  loop: 2026-05-12-0645
  links: parent: board/okrs/2026-05-12-0645-ceo-brief.md
  team_okr: teams/platform/devops/okrs/2026-05-12-0645-team-okr.md
-->
{% raw %}

# DevOps Exec Summary — Loop 2026-05-12-0645

**Headline.** First product workload deployed to the home cluster end-to-end. The `nanofab-supervisor:0.3.0` image was built cross-arch (linux/amd64 from darwin/arm64 via `docker buildx`), distributed to all 4 nodes via `sudo ctr -n=k8s.io images import`, installed via Helm against `kubernetes-admin@kubernetes`, verified via `kubectl logs` (`supervisor: ok`), and uninstalled cleanly. Deploy → verify → delete cycle complete. **Two latent chart bugs surfaced and patched in-loop:** (1) `restartPolicy: Never` rendered into Deployment.spec.template.spec is invalid for Deployment kind (removed the unconditional render; values knob retained for future Job-shape variant); (2) Dockerfile was missing the `/fixtures` bake step (added; the supervisor's default fixtures-dir is `/fixtures`, not `<workspace>/../fixtures` as the chart comment claimed). The 4-loop minikube smoke carry is **structurally superseded** by the home-cluster deploy.

## Per-KR rollup

| KR | Outcome | Notes |
|---|---|---|
| KR1.1 (`values/home-cluster-mvp.yaml`) | **PASS** | New file; sister to `values/minikube-mvp.yaml`. `helm lint --strict` + `helm template` + `helm install --dry-run` all exit 0. |
| KR1.2 (image distribution) | **PASS** | `scp` + `sudo ctr -n=k8s.io images import` succeeded on all 4 nodes; `crictl images` confirms same image ID (`0b11a5c085874`) per node. Idempotent (re-import is a no-op). |
| KR1.3 (`helm install`) | **PASS** (after 2 chart patches) | First install failed on `restartPolicy: Never` (chart bug); patch landed (deployment.yaml line 21 removed). Second install failed on missing `/fixtures` directory (Dockerfile bug); patch landed (Dockerfile COPY synthetic_tenants/closed_loop_v0/fixtures /fixtures). Third install: `STATUS: deployed`. |
| KR1.4 (output verification) | **PASS via logs; documented divergence on cp** | `kubectl logs` confirms `[supervisor] [dim_user] wrote 21 rows...`, `[supervisor] [dim_account] wrote 18 rows...`, `supervisor: ok`. `kubectl cp /workspace/trace.jsonl` races BackOff (Deployment-vs-Job kind mismatch); documented in SOP as F-area limitation. Future Job-kind variant proposed. |
| KR1.5 (delete path) | **PASS** | `helm uninstall` returns 0; `kubectl get all -n nanofab` returns "No resources found"; `kubectl delete namespace nanofab` succeeds; final `kubectl get ns \| grep nanofab` empty. |
| KR1.6 (chart README update) | **PASS** | New "## Install (home cluster)" section (canonical) before legacy minikube section; minikube section gets a top-line blockquote note; Limitations gained a "Deployment vs Job kind mismatch" entry. Cross-links to SRE SOP. |
| KR1.7 (tenant-isolation) | **PASS** | Final `grep -nrE "(resink\|nanofab\|home-cluster\|acme\.ai)" org-os/` returns only the canonical `acme.ai` placeholder at `org-os/conventions.md:121`. |

## Chart patches landed this loop (latent bugs)

1. `deploy/charts/nanofab-supervisor/templates/deployment.yaml` line 21 — removed unconditional `restartPolicy: {{ .Values.pod.restartPolicy }}` (Deployment kind only admits `Always`; the values knob is preserved for a future Job variant). Added an explanatory comment naming this loop and pointing at the SOP's "Known chart constraints" subsection.
2. `deploy/charts/nanofab-supervisor/Dockerfile` — added `COPY --from=rust-builder /build/synthetic_tenants/closed_loop_v0/fixtures /fixtures` + chown the directory to uid 65532. Without this the supervisor exited `open /fixtures/fact_sign_up.parquet: No such file or directory`.
3. `deploy/charts/nanofab-supervisor/Dockerfile` (separate, build-time) — bumped `rust:1.83-slim-bookworm` to `rust:1.85-slim-bookworm` because `Cargo.lock`'s pinned `clap_lex 1.1.0` requires edition2024 (Rust ≥ 1.85). The workspace's declared MSRV is 1.83 but the lockfile moved past it.

## Cross-team coordination

- **Resink-core (O2):** image build + distribution path coordinated. resink-core's `docs/user-guide.md` gained a "Deploying to the home cluster" section + `docs/architecture.md` Named-deviations entry updated for the ADR-2026-05-16-001 step 3 slip-by-one.
- **SRE (O3):** authored the deployment SOP runbook in parallel with this build phase; cross-links closed in both directions.

## Open carries (for next loop)

- **Job-kind chart variant.** Surfaced this loop; sized by DevOps or SRE next loop. The cleanest fix for the one-shot supervisor's `kubectl cp` race.
- **In-cluster image registry.** Per-node `ctr import` is the current bootstrap path; a future home-cluster Phase 1 loop adds a local registry. Until then, every image rebuild re-distributes.
- **pyinfra wrapper at `services/nanofab_supervisor/`.** Home-cluster roadmap Phase 1.1. The manual SOP this loop is the first iteration; the wrapper is iteration 2.
- **`/healthz` + `/readyz`** — depends on resink-core's long-running supervisor.
- **Frontmatter-lint CI script** — seventh consecutive defer.

## Tenant-isolation invariant

Held throughout. Zero `org-os/` writes by DevOps this loop. All edits landed under `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` (values file new, README updated, deployment.yaml + Dockerfile patched), and `teams/platform/devops/okrs/` + `teams/platform/devops/exec-summaries/`. Final tenant-isolation dry-run clean.
{% endraw %}
