---
layout: default
title: Exec Summary — 2026-05-12-0645
date: 2026-05-12
status: active
type: exec-summary
loop: 2026-05-12-0645
owner: board
---

<!-- original-frontmatter:
  type: exec-summary
  owner: board
  date: 2026-05-12
  status: active
  loop: 2026-05-12-0645
  links: parent: board/okrs/2026-05-12-0645-ceo-brief.md
-->
{% raw %}

# Resink.ai Loop 2026-05-12-0645 — Company Exec Summary

**Headline.** Resink.ai's first product workload shipped to real Kubernetes infrastructure. `nanofab-supervisor:0.3.0` built cross-arch (linux/amd64 from darwin/arm64), distributed to all 4 home-cluster nodes via `sudo ctr -n=k8s.io images import`, installed via Helm against `kubernetes-admin@kubernetes`, executed successfully (`supervisor: ok`, 21 dim_user + 18 dim_account rows written), and uninstalled cleanly with zero residual objects. **The 4-loop minikube smoke carry is structurally superseded** — the chart's canonical deploy target is now the home cluster; minikube is preserved as a legacy local-dev path. Two latent chart bugs surfaced and were patched in-loop (`restartPolicy: Never` in Deployment; missing `/fixtures` bake in Dockerfile). SRE's deployment SOP runbook is the **third adopter** of ADR-2026-05-16-003's `Verified-against-environment` discipline and the **first runbook-type adopter** (after sim-farm's verdict contract and DE's schema-JSON convention). Two cross-team requests closed end-to-end (resink-core's R1–R5 ack + DE's K1–K3 ack); **the org-os tree now has three fulfilled cross-team request lifecycles.** Workspace promotion: 7th-loop carry; the brief's named `gh repo create` command was not run in-loop. ADR-2026-05-16-001 step 3 (hot-swap correctness test) explicitly deferred to loop+2 per the brief's deployment focus.

## Per-team rollup

### board (active, light)

Authored brief + this exec summary + retro. No ADR ratifications this loop (no draft ADRs entering the loop). No `org-os/` writes by design — code-bearing on tenant surfaces only. Tenant-isolation invariant held trivially.

### teams/platform/devops (active, primary)

O1 fully shipped: new `values/home-cluster-mvp.yaml`, image distributed to all 4 nodes, `helm install` succeeded, supervisor ran successfully, `helm uninstall` + namespace delete clean. Surfaced and patched 2 latent chart bugs (`restartPolicy: Never` + missing fixtures bake); also bumped the Dockerfile's rust base from 1.83 to 1.85 (forced by `clap_lex 1.1.0` requiring edition2024). Chart README gained a canonical "Install (home cluster)" section and a Limitations entry for the Deployment-vs-Job kind mismatch. [full summary](../../teams/platform/devops/exec-summaries/2026-05-12-0645.md)

### teams/application/resink-core (active, primary)

O1 fully shipped: image built (154 MB, amd64/linux), saved, handed off to DevOps for distribution. `docs/user-guide.md` gained "Deploying to the home cluster" section. `docs/architecture.md` Named-deviations entry updated for ADR-2026-05-16-001 step 3 slip-by-one. O2: R1–R5 ack closure — `requests/2026-06-06-handoff-response-ack.md` flipped `accepted → fulfilled`. **Workspace promotion 7th-loop carry**; opportunistic-only; not actioned in-loop. Cargo MSRV vs lockfile drift surfaced (workspace declares 1.83; Cargo.lock pins `clap_lex 1.1.0` needing 1.85); future-loop bookkeeping. [full summary](../../teams/application/resink-core/exec-summaries/2026-05-12-0645.md)

### teams/platform/sre (active, primary)

O1 fully shipped: new `runbooks/nanofab-supervisor-deployment.md` (`type: runbook`, severity_tiers, 6 failure-mode subsections F1–F6, `Verified-against-environment` subsection — first SRE adopter, third overall). Home-cluster operator runbook gained the supervisor as the cluster's first product workload. Cross-links closed all 4 directions. Status.md refreshed (was 2026-06-06; now 2026-05-12). [full summary](../../teams/platform/sre/exec-summaries/2026-05-12-0645.md)

### teams/platform/data-engineering (paused, review-ack + K1-K3 fulfillment)

Reviewed home-cluster deployment for schema-JSON consumer surface impact — none (supervisor runtime doesn't consume schema-JSON from chart values; baked into image at orchestrator-time). DE's K1–K3 request closed (`requests/2026-06-06-handoff-response-ack.md` `accepted → fulfilled`; paused-team format per ADR-2026-06-06-002). [full summary](../../teams/platform/data-engineering/exec-summaries/2026-05-12-0645.md)

### teams/application/sim-farm (paused, review-ack)

Reviewed home-cluster deployment for verdict-contract surface impact — none (sim-farm engine is Python, not part of the chart's supervisor binary; verdict.json shape unaffected by deployment mechanism). Verdict-contract back-reference to DE convention remains carry. [full summary](../../teams/application/sim-farm/exec-summaries/2026-05-12-0645.md)

### teams/platform/agent-engineering (paused, review-ack)

Reviewed deployed binary for ABI/plugin contract impact — none (this loop exercises `static-plugins` only; `dlopen-plugins` containerization concerns flagged for future loop). Step 3 hot-swap test re-scheduled to loop+2 per brief. [full summary](../../teams/platform/agent-engineering/exec-summaries/2026-05-12-0645.md)

## Cross-cutting wins

- **First product workload shipped to real Kubernetes infrastructure.** Seven loops of structured org-os work, two MVP demos green-bar on developer laptops, one Helm chart authored over 4 loops dry-run clean — and now, finally, a pod running on a real cluster. The deployment surface is exercised end-to-end (build → distribute → install → verify → delete) with documented procedure + failure-mode runbook + chart README + operator runbook. Anyone with kubeconfig access can reproduce.
- **4-loop minikube-smoke carry structurally resolved by reframing the deployment target, not by acting.** The pattern from the 2026-06-13 retro (working-answer to P5: "reframe the blocker class, not the specific blocker") generalized — minikube was the wrong shape; the home cluster is the right shape. **The pattern has now resolved two recurring blocker classes in two consecutive loops** (toolchain installs as Prerequisite docs at 2026-05-11-2153; deployment target as home cluster at 2026-05-12-0645). Reproducibility: when a multi-loop blocker has the wrong premise, ask whether a different shape exists rather than trying to clear the specific blocker.
- **Two latent chart bugs surfaced + fixed in-loop, without scope creep.** The chart was helm-template-clean and helm-install --dry-run-clean for 4 loops but never actually applied. First real `helm install` against a live cluster surfaced both bugs within minutes; both fixes were single-line edits; both got documented in chart README + SOP for future readers. **Reproducibility:** dry-run testing is necessary but not sufficient — first-real-deploy is when latent bugs become manifest. SOPs authored alongside first-deploy (not after) are the strongest possible fresh-eyes review.
- **Three end-to-end fulfilled cross-team request lifecycles** on the org-os tree (cumulative). First was DE's schema-JSON at 2026-05-11-2153 (open → accepted-deferred → fulfilled across two loops). Second + third this loop: resink-core's R1–R5 (chart hand-off ack, operationally honored across 3 loops, formally closed) + DE's K1–K3 (Kafka chart hand-off ack, paused-team format). The request mechanism (canonicalized at ADR-2026-06-06-002) is observably load-bearing.
- **`Verified-against-environment` discipline generalizes to a fourth artifact type.** ADR-2026-05-16-003 was originally scoped to `contract` and `rfc` types. Adopters so far: sim-farm verdict contract (2026-06-06), DE schema-JSON convention (2026-05-11-2153), SRE deployment runbook (this loop). The pattern transferred without grammatical or structural friction — toolchain versions, OS+arch, runtime deps, auth-mode prereqs. Retro candidate to extend the ADR's scope explicitly.
- **The home-cluster repo's roadmap (`docs/superpowers/specs/2026-05-10-home-cluster-roadmap-design.md`) gained its first product-workload consumer.** Phase 1 designs `services/<name>/install|upgrade|delete` pyinfra wrappers; this loop's manual SOP is the human-readable version of what those wrappers will automate. The roadmap design becomes load-bearing.

## Cross-cutting blockers

- **Workspace promotion 7th-loop carry.** The brief's named `! gh repo create resink-ai/resink-core ...` command was not run in-loop. Total downstream blockers now: 6 (added the SOP's image-build step citing the in-tree path). The 6-loop-tally has stayed unchanged for one loop but the carry continues. **Retro escalation candidate:** if 8th-loop carry materializes, propose a different mechanism (e.g., a brief that refuses to author downstream work until the remote exists — drastic, but tested by precedent in the 2026-06-13 retro's P1 proposal which already reframed once).
- **Deployment-vs-Job kind mismatch.** The Deployment-kind chart cycles the one-shot supervisor through BackOff after every exit-0; `kubectl cp /workspace/trace.jsonl` races the BackOff state and is unreliable. Workaround: `kubectl logs` for verification (works fully). Fix: a `kind: Job` chart variant gated on a `workloadKind` values flag. Sized at next loop or whenever DevOps takes it.
- **No image registry on the home cluster.** Every image rebuild requires re-running the 4-node distribute loop. Home-cluster roadmap Phase 1 calls for a local in-cluster registry; not in scope this loop. Mild friction; documented limitation.

## Asks for the CEO

Deduplicated from per-team summaries:

- **(from board / scheduling):** Hot-swap correctness test (ADR-2026-05-16-001 step 3) re-scheduled to loop+2 (the loop after this one); joint AE + resink-core deliverable. AE awaits scheduling signal in the next CEO brief.
- **(from resink-core + carried 7-loop):** Create `resink-ai/resink-core` GitHub remote. Brief's named command stands: `! gh repo create resink-ai/resink-core --private --description "Resink.ai nanofab runtime, training pipeline, sim-farm engine, and orchestrator"`. The pattern (P1 reframing from 2026-06-13 retro) is working in form but the user hasn't run the command yet; retro should consider whether the form needs further evolution.
- **(from devops + sre / scheduling):** Job-kind chart variant — proposes `workloadKind` values flag with `Deployment` (default; future long-running supervisor) and `Job` (one-shot MVP). Sized at next loop or whenever resink-core's long-running supervisor with `/healthz` lands (whichever comes first).
- **(from devops):** In-cluster image registry on the home cluster — Phase 1 of the home-cluster roadmap. Pickup at any future deployment-focused loop.
- **(from resink-core):** Workspace MSRV declaration is 1.83 but Cargo.lock pins `clap_lex 1.1.0` requiring edition2024 (Rust ≥ 1.85). Bookkeeping: bump MSRV declaration to 1.85 or downgrade pinned deps.

## Decisions ratified this loop

**None.** No draft ADRs entered the loop; no new ADRs proposed for ratification. Zero `org-os/` writes this loop (verified by final tenant-isolation dry-run).

## Decisions filed this loop (not new ADRs)

- **Chart deployment.yaml patched.** `restartPolicy: Never` was rendered into Deployment.spec.template.spec — invalid for Deployment kind. Removed the unconditional render; the `pod.restartPolicy` values knob is retained for a future Job-shape variant. Captured in chart README Limitations + SRE SOP § "Known chart constraints" + this exec summary. Surfaced at first real `helm install`; previously masked by dry-run-only testing.
- **Chart Dockerfile patched (3 changes):** (1) `rust:1.83` → `rust:1.85` base for edition2024 compat; (2) added `COPY synthetic_tenants/closed_loop_v0/fixtures /fixtures` because supervisor's default fixtures-dir is `/fixtures`, not `<workspace>/../fixtures`; (3) chown `/fixtures` to uid 65532 alongside `/workspace`.

## Multi-loop plan slippage absorbed (not new ratifications)

- **ADR-2026-05-16-001 (Option A → dlopen restoration):**
  - Step 1 (AE template extension): ✅ closed loop-3 (= 2026-05-11-1631).
  - Step 2 (resink-core supervisor swap full execution): ✅ closed loop-1 (= 2026-05-11-2153).
  - Step 3 (hot-swap correctness test): was `loop+1` (= this loop), **slipped to `loop+2`** (= the loop after this one). Slip reason: deployment focus crowded out the hot-swap test. Joint AE + resink-core deliverable.

## Request-flow metrics this loop (per ADR-2026-05-09-006)

| Metric | Count | Note |
|---|---|---|
| `cross-team-request` objectives `fulfilled` | 2 | resink-core's R1–R5 ack (active-team format) + DE's K1–K3 ack (paused-team format per ADR-2026-06-06-002). Both fulfilled in-loop. |
| `dropped` | 0 | None. |
| `declined` | 0 | None. |
| `escalated` | 0 | None. |
| In-loop requests filed (not yet ack'd) | 0 | No new cross-team requests filed this loop. |

## Tenant-isolation invariant

Held across the loop. **Zero `org-os/` writes by any team** (designed-in: code-bearing on tenant surfaces only, no ADR ratifications). Final tenant-isolation dry-run: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns only the canonical `acme.ai` placeholder at `org-os/conventions.md:121`. **The smallest org-os surface impact since 2026-05-11-2153** (which also had zero `org-os/` writes from product teams but did have board-mandated edits for the 4-ADR ratification batch). This loop achieves the absolute minimum: zero `org-os/` edits anywhere. Pass.

## Notes for the retro

Three patterns worth recording for next-loop authoring:
1. **First-real-deploy as the actual integration test.** Helm dry-run + helm template + helm lint --strict are necessary but not sufficient. First real `helm install` against a live cluster is the only deterministic surface for latent chart bugs.
2. **SOP-author-alongside-first-execution generates better SOPs than after-the-fact.** The two patched chart bugs became failure-mode-subsection content immediately; the SOP's Known-chart-constraints subsection is anchored in observed-not-anticipated friction.
3. **Reframing-vs-acting consumed a second recurring blocker class.** The pattern from 2026-06-13 retro (P5) — when a blocker has the wrong shape, reframe the class — handled the minikube carry this loop (replaced minikube with the home cluster as the deployment target). Two consecutive loops of pattern application; candidate for codifying as a playbook.
{% endraw %}
