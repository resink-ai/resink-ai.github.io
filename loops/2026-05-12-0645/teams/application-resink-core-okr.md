---
layout: default
title: application-resink-core OKR — 2026-05-12-0645
date: 2026-05-12
status: active
type: okr
loop: 2026-05-12-0645
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-12
  status: active
  loop: 2026-05-12-0645
  links: parent: board/okrs/2026-05-12-0645-ceo-brief.md
-->
{% raw %}

# Resink Core OKR — 2026-05-12

## Context

Last loop closed ADR-2026-05-16-001 step 2 (full dlopen-plugins execution) end-to-end on developer laptops; both feature flag configurations green-bar at `cargo build`, `cargo test`, and `make mvp-loop`. **This loop is the first time the supervisor binary builds into a container image and ships to a real Kubernetes cluster.** Per CEO brief 2026-05-12-0645 § O2: build the supervisor image via the existing `deploy/charts/nanofab-supervisor/Dockerfile`, save it to a tarball for DevOps to distribute, and document the deploy path in `docs/user-guide.md`. The image tag aligns with `Chart.appVersion`; the binary crate version is `0.1.0` (per workspace `Cargo.toml`), so the canonical tag is `nanofab-supervisor:0.1.0`.

ADR-2026-05-16-001 step 3 (hot-swap correctness test) was scheduled `loop+1` relative to step 2's closure (= this loop), but the brief defers it to `loop+2` to keep this loop single-focus on deployment. Slippage record updates `docs/architecture.md § Named deviations`.

This loop also fulfills the long-open R1-R5 hand-off ack request (`requests/2026-06-06-handoff-response-ack.md`); the chart's choices have been operationally honored across the last two loops, and this loop's end-to-end deploy against the home cluster is the strongest form of ack.

## Objectives

### O1: Build + distribute the supervisor image; document the home-cluster deploy path

source: ceo-brief

Why it matters: First production image build of the supervisor binary. Per the R1 hand-off response, DevOps owns the chart-side Dockerfile but resink-core owns the binary build — this loop is the first time the Dockerfile actually compiles a real image and ships it to a target cluster. The image is the long-lived artifact that future deploys consume; the user-guide section documents the build → distribute → deploy flow so a second IC can reproduce. The Dockerfile's COPY of `synthetic_tenants/closed_loop_v0/workspace` makes `workspaceMount.enabled: false` work without external volumes — a deliberate design choice that simplifies the first home-cluster deploy.

**Key results**

- KR1.1: `docker build -f deploy/charts/nanofab-supervisor/Dockerfile -t nanofab-supervisor:0.1.0 .` (run from `repos/resink-ai/resink-core/`) exits 0. Image size recorded via `docker inspect nanofab-supervisor:0.1.0 --format='{{.Size}}'` (expect 100–200 MB, multi-stage build with debian:12-slim runtime). The image's CMD is the existing Dockerfile's CMD (`--mode=sim --workspace=/workspace --write-trace=/workspace/trace.jsonl --write-output=/workspace/dim_user_output.parquet`), and the chart's `templates/_helpers.tpl::nanofab-supervisor.args` overrides via `args:` in the Deployment spec.
- KR1.2: Image tag aligned with `Chart.appVersion`. Chart.yaml currently declares `appVersion: "0.1.0"`; the workspace `Cargo.toml` declares `[workspace.package].version = "0.1.0"` (verified). Tag `nanofab-supervisor:0.1.0` is the canonical first-deploy tag. Naming convention recorded in the exec summary for future loops.
- KR1.3: `docker save nanofab-supervisor:0.1.0 -o /tmp/nanofab-supervisor.tar`. Tarball size recorded (expect ~100–200 MB compressed). Hand off path `/tmp/nanofab-supervisor.tar` to DevOps for distribution per their O1 KR1.2. Cleanup of `/tmp/nanofab-supervisor.tar` on the dev machine: deferred to next-loop housekeeping.
- KR1.4: `repos/resink-ai/resink-core/docs/user-guide.md` gains a new top-level section "## Deploying to the home cluster" after the existing "Common commands" section. ~30 lines. Names: (a) the build command (verbatim from KR1.1); (b) the save command (verbatim from KR1.3); (c) the distribute command (DevOps's `scp` + `sudo ctr -n=k8s.io images import` loop — cite the SRE SOP for the per-node detail); (d) the helm install command; (e) the verification commands (`kubectl get pods -n nanofab`, `kubectl logs`, `kubectl cp` the trace); (f) the delete commands (`helm uninstall`, `kubectl delete namespace`). Cross-link the SRE SOP at `teams/platform/sre/runbooks/nanofab-supervisor-deployment.md` and the chart README's new "Install (home cluster)" section.
- KR1.5: `repos/resink-ai/resink-core/docs/architecture.md` § "Named deviations" — the ABI Option A entry's "Step 3 (hot-swap correctness test)" sub-bullet updated: was "loop+1" (relative to step 2's closure at 2026-05-11-2153), now "loop+2" with one-line slippage reason "deployment focus this loop crowds out hot-swap test." Single sub-bullet edit; no other deviations affected.
- KR1.6: Cross-team-request fulfillment — `teams/application/resink-core/requests/2026-06-06-handoff-response-ack.md` flips `status: accepted → status: fulfilled`. `links.fulfilled_by` populated with this OKR's path. One-line resolution note appended at file bottom citing the home-cluster deploy as the operational ack of all five R1-R5 chart-side choices.
- KR1.7: Tenant-isolation invariant holds. All edits land under `repos/resink-ai/resink-core/` and `teams/application/resink-core/`; no `org-os/` writes. Final tenant-isolation dry-run: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns only the canonical `acme.ai` placeholder.
- KR1.8: **Opportunistic** — workspace promotion (6-loop carry): if the `resink-ai/resink-core` GitHub remote is created in-loop (e.g., by the human running the brief's named `! gh repo create ...` command), resink-core executes the submodule promotion in this loop's build phase as an unscored sub-task. No KR penalty if not.

**Tasks**

- [ ] Run `docker build` for the supervisor image; record size + tag; capture build log for the exec summary.
- [ ] `docker save` to `/tmp/nanofab-supervisor.tar`; hand off path to DevOps.
- [ ] Update `docs/user-guide.md` with "Deploying to the home cluster" section (~30 lines); cross-link the SRE SOP + chart README.
- [ ] Update `docs/architecture.md § Named deviations` with the step-3 slip-by-one.
- [ ] Flip `requests/2026-06-06-handoff-response-ack.md` to `status: fulfilled`; populate `links.fulfilled_by`; append resolution note.
- [ ] Tenant-isolation dry-run.
- [ ] (Opportunistic) Workspace promotion if the GitHub remote is created in-loop.

### O2: Fulfill the R1–R5 chart hand-off ack request

source: cross-team-request
links.source: teams/application/resink-core/requests/2026-06-06-handoff-response-ack.md

Why it matters: The request has been open since 2026-06-06 and operationally honored across the last two loops (chart's R1–R5 absorption in 2026-05-11-1631 + dlopen step 2 closure against the chart's ConfigMap-mount semantics in 2026-05-11-2153). This loop's end-to-end deploy against the home cluster is the strongest possible form of ack — the chart's chosen mechanisms (Dockerfile ownership split, deferred /healthz, 30s SIGTERM drain, CLI flags via args, ConfigMap-mounted manifest) all execute end-to-end without surfacing a counter-proposal. Closing the request to `status: fulfilled` is the request-lifecycle complement of O1.

**Key results**

- KR2.1: One-line ack per response (R1, R2, R3, R4, R5) lands in this loop's exec summary at `teams/application/resink-core/exec-summaries/2026-05-12-0645.md`. Format: "**R1 (Dockerfile ownership):** accepted; DevOps owns Dockerfile, resink-core owns binary build — exercised this loop." Same shape for R2-R5.
- KR2.2: Request file mutated in place: `status: accepted → status: fulfilled`; `links.fulfilled_by` populated with `teams/application/resink-core/okrs/2026-05-12-0645-team-okr.md`; one-line resolution note appended at file bottom. (Folded into O1 KR1.6 mechanically; named separately here for request-lifecycle accounting per ADR-2026-06-06-002.)

**Tasks**

- [ ] Author the 5-line R1–R5 ack in `exec-summaries/2026-05-12-0645.md` (lands at exec-summary time).
- [ ] Mutate `requests/2026-06-06-handoff-response-ack.md` per KR2.2 (folded into O1 KR1.6).

## Risks

- **`docker build` failure on first attempt:** glibc/openssl mismatches between `rust:1.83-slim-bookworm` and `debian:12-slim` runtime stages; missing system deps. Mitigation: the Dockerfile already installs `pkg-config + libssl-dev + ca-certificates` in stage 1 and `ca-certificates + libssl3` in stage 2. If a symbol-not-found error surfaces at container start, fall back to `rust:1.83-bookworm` (full) or pin a specific patch version; document the working configuration.
- **Image tag drift between Chart.yaml and the binary crate:** Chart.yaml's `appVersion: "0.1.0"` currently matches the workspace `Cargo.toml`'s `[workspace.package].version = "0.1.0"`. If a future loop bumps the crate without bumping Chart.appVersion, the tag mismatch surfaces as an `ErrImagePull` (image not loaded at the expected tag). Mitigation: KR1.2 names the convention in the exec summary; future loops follow.
- **The R1–R5 ack is operationally retroactive.** The request was open for two loops; closing it now signals "you've been operating against our chosen mechanisms without counter-proposal for long enough that we treat it as accepted." If a counter-proposal surfaces in a later loop, the ack doesn't prevent it — it just records the current state. Mild risk; no mitigation needed.
- **Workspace promotion 6th-loop carry** — opportunistic-only; no KR penalty. Documented in this loop's exec summary either way.

## Out of scope this loop

- **Hot-swap correctness test (ADR-2026-05-16-001 step 3)** — deferred to `loop+2` per CEO brief; joint AE + resink-core deliverable; AE awaits scheduling signal in the next CEO brief.
- **Long-running supervisor with `/healthz` + `/readyz`** — depends on a Kafka-driven workload; no carry this loop.
- **Workspace promotion (full submodule push to the new remote)** — opportunistic only; 6-loop carry continues if the remote doesn't materialize in-loop.
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing.
- **Supervisor determinism test under containerization** — out of scope; deferred to whichever loop wants to ratify the trace's byte-stability discipline across hosts.
- **In-cluster image registry consumer integration** — out per brief's "no image registry this loop" framing; future loop concern.
- **Future codegen patterns / template extensions** — out per AE's paused status.
{% endraw %}
