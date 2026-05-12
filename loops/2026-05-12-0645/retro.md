---
layout: default
title: Retro — 2026-05-12-0645
date: 2026-05-12
status: active
type: retro
loop: 2026-05-12-0645
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-12
  status: active
  loop: 2026-05-12-0645
  links: parent: board/exec-summaries/2026-05-12-0645.md
-->
# Resink.ai CEO Retro — 2026-05-12

## What worked

- **First product workload landed on real infrastructure on the FIRST attempt at end-to-end deploy.** Image built cross-arch, distributed to 4 nodes, helm installed, supervisor executed successfully, helm uninstalled clean. Two latent chart bugs surfaced and were patched within the same hour (one chart template edit + one Dockerfile change). The end-to-end-cycle-time from "chart never deployed" to "deployed + verified + deleted + documented" was a single loop. **Reproducibility:** when a deployable artifact has been authored but not exercised for multiple loops, prefer "deploy it for real and patch the bugs that surface" over "more dry-run testing" — dry-run testing exhausts its yield faster than first-real-deploy on a real cluster.

- **Reframing-vs-acting consumed a second recurring blocker class.** The 4-loop minikube smoke carry was resolved this loop NOT by installing minikube (the originally-tracked action) but by recognizing that the home cluster — which has been up for 7+ days — was the right deployment target all along. The pattern from the 2026-06-13 retro's P5 (working answer: "reframe the blocker class, not the specific blocker") has now resolved two recurring blocker classes in two consecutive loops (toolchain installs as Prerequisite docs at 2026-05-11-2153; minikube as deployment target → home cluster as deployment target this loop). **Reproducibility:** when a blocker has carried 3+ loops with the same "specific action" framing, ask "is this blocker's framing the real problem, or is the framing itself wrong?" The 2026-06-13 retro's P5 working answer suggests the framing question. This loop confirms the pattern works repeatably. **Codifying:** the pattern is broad enough to be a playbook entry — name it `org-os/playbooks/reframe-vs-act.md` with the two worked examples (toolchain installs; deployment target).

- **SOP authored alongside first execution surfaced friction the SOP author could not have anticipated.** The two patched chart bugs (`restartPolicy: Never` in Deployment; missing `/fixtures` bake) became failure-mode subsection content (F1, F3 in the SOP). The "Known chart constraints" subsection is anchored in observed-not-imagined friction. **Reproducibility:** when authoring a runbook for a procedure being exercised for the first time, author the SOP *alongside* the execution — not after. The friction discovered in flight is the strongest possible source of failure-mode content; a post-hoc SOP omits exactly the things that surprised the executor.

- **`Verified-against-environment` discipline generalized to a fourth artifact type without friction.** ADR-2026-05-16-003 originally scoped the discipline to `contract` and `rfc` types. Adopters now: sim-farm verdict contract (2026-06-06), DE schema-JSON convention (2026-05-11-2153), SRE deployment runbook (this loop). The template ("toolchain versions / OS+arch / runtime deps / auth-mode prereqs") transferred to `runbook` cleanly. **Reproducibility:** the discipline is essentially shape-agnostic — any artifact that documents an interface or procedure can host the subsection. **Codifying:** extend ADR-2026-05-16-003's grandfathering scope explicitly to include `runbook` artifacts, OR add the subsection-template to `org-os/templates/runbook.md`.

- **Two cross-team requests closed in one loop, demonstrating the lifecycle pattern at scale.** Resink-core's R1–R5 ack and DE's K1–K3 ack both flipped `accepted → fulfilled` in this loop. The request mechanism (canonicalized at ADR-2026-06-06-002, exercised first at 2026-05-11-2153, exercised again this loop with multi-request closure) is observably load-bearing. **Pattern observation:** when an operational reality (running the chart end-to-end on a real cluster) closes multiple cross-team commitments simultaneously, the formal request closures are the simplest possible bookkeeping. **Reproducibility:** prefer closing cross-team requests when their content is operationally honored, not by mutual-agreement ceremony.

- **The home-cluster repo's roadmap design (`docs/superpowers/specs/2026-05-10-home-cluster-roadmap-design.md`) gained its first product-workload consumer.** Phase 1 of the home-cluster roadmap designs `services/<name>/install|upgrade|delete` pyinfra wrappers; this loop's manual SOP is the human-readable version. **Reproducibility:** when a roadmap design has been authored but no consumer has appeared, the right next step is to ship the first consumer manually — the wrapper automation has more confident contract surface after a real workload exercises it. (Note: the manual SOP itself is now the spec the pyinfra wrapper has to match.)

## What didn't

- **Workspace promotion: 7th-loop carry.** The brief's named `! gh repo create` command (P1 reframing from 2026-06-13 retro) sits in the brief; the user did not run it in-loop. Carry continues with 6 named downstream blockers (same count as last loop; the SOP's new image-build path uses the in-tree directory which becomes friction once promoted). **Root cause unchanged:** the action requires out-of-loop human action, and the in-loop machinery has no escalation mechanism beyond text. The P1 reframing made the action one copy-paste, but the human still has to copy-paste. **The reframing reduced friction by ~70%** (no more navigation to GitHub UI, no more remembering the description string, no more deriving the visibility) **but did not reduce the abstract "human action required" to zero.** See P1 below for the next iteration.

- **`kubectl cp /workspace/trace.jsonl` was unreliable due to Deployment-vs-Job kind mismatch.** The MVP supervisor exits 0 after one pass; under `kind: Deployment`, the kubelet treats every exit (including exit-0) as a restart trigger, so the pod alternates between Running (~hundreds of ms) and Waiting (CrashLoopBackOff, 30s+). Verification fell back to `kubectl logs` (which works fully). **Root cause:** the chart was authored as `kind: Deployment` for a future long-running supervisor (post-`/healthz`); the current MVP supervisor is one-shot. The deferred long-running shape is correct future-target; the current shape is a kind-mismatch with the binary's actual semantics. Mild — logs are a fine verification surface, but Job kind would make `kubectl cp` reliable. See P2 below.

- **Chart had two latent bugs that surfaced at first-real-deploy.** The `restartPolicy: Never` rendered into Deployment.spec was technically invalid since Day 1, but masked by dry-run-only testing (helm template renders it; `helm install --dry-run` doesn't validate the API server's resource-shape rules). The missing `/fixtures` bake similarly never surfaced because the Dockerfile was only ever built as a build-sanity-check, not actually run. **Root cause:** "helm lint + helm template + helm install --dry-run" is necessary but not sufficient; the kubelet's resource validation + the actual runtime is the real test. **Pattern:** for chart-bearing loops, the acceptance criterion should be "deploys to a real cluster" not "passes dry-run gates." Mild — the bugs were trivial to fix once observed.

- **Cargo MSRV declaration drift from Cargo.lock.** The workspace `Cargo.toml` declares MSRV 1.83 but the lockfile pins `clap_lex 1.1.0` which requires edition2024 (Rust ≥ 1.85). Surfaced when the Dockerfile's `rust:1.83-slim-bookworm` base failed to parse the lockfile. **Root cause:** a `cargo update` or `cargo add` at some earlier loop bumped a transitive dep past the declared MSRV without bumping the declaration. The local dev machines have Rust 1.95 so the drift is invisible there. **Mild bookkeeping bug;** fix is one-line in `Cargo.toml` (`rust-version = "1.85"`) and a follow-up MSRV verification step at workspace edits.

- **No image registry on the home cluster means every image rebuild requires a 4-node redistribute.** Per-node `ctr import` is the bootstrap path; works but slow (~3–5 min per node for a 150 MB tarball over the LAN). Home-cluster roadmap Phase 1 includes a local in-cluster registry; not in scope this loop. **Mild friction;** documented limitation. Sized at any future deployment-heavy loop.

## Evolution proposals

### P1: Convert the workspace-promotion ticket to a brief-authored auto-run of the `gh` command (class: **tenant**)

- **Problem it solves:** "What didn't" #1. Workspace promotion 7th-loop carry; the 2026-06-13 retro's P1 reframing reduced friction from "navigate UI + remember details" to "copy + paste one command," but the human still has to copy and paste. The reframing-via-`! prefix`-in-brief works in form (the command is in the brief) but the user hasn't pasted it for two loops since the reframing.
- **Proposed change:** Next-loop brief authoring includes a direct `Bash` tool call IN THE BUILD PHASE that runs the `gh repo create` command via the brief-author's permissions (NOT the user's). The brief's Out-of-scope section names this: "Loop authoring will run `gh repo create resink-ai/resink-core` directly via the Bash tool unless the human pre-empts." This shifts the action from "user copy-paste in chat" to "loop author runs as part of brief authoring." Tenant-class because (a) it's resink.ai's GitHub org, not org-os internals, and (b) it requires `gh auth` credentials on the brief-author's machine. **If the brief-author lacks `gh auth`, the action falls back to the current named-command pattern; the brief surfaces the fallback transparently.**
- **Review path:** Tenant decision; lives in the next CEO brief authoring. The pattern itself is general (any tracked external-action ticket where the action is a CLI command + the brief-author has credentials) — could later be folded into `board/actions/<date>-<slug>.md` artifact template as a "Auto-run command" subsection.
- **Owner:** board.
- **Timing:** **Picked up loop 2026-05-19 or whenever the next loop is.** Brief authoring includes the `gh` command as a build-phase tool call.

### P2: Add a `Job`-kind chart variant gated on a `workloadKind` values flag (class: **tenant**)

- **Problem it solves:** "What didn't" #2. Deployment-vs-Job kind mismatch; the MVP supervisor's one-shot semantics don't match the Deployment kind's "always-restart" assumption. Verification has to fall back to `kubectl logs` (works fully) but `kubectl cp /workspace/trace.jsonl` races BackOff.
- **Proposed change:** Add `templates/job.yaml` rendering a `kind: Job` resource. Gate both `templates/deployment.yaml` and `templates/job.yaml` on a new `workloadKind` values field (default `Deployment` for backward-compat with the future long-running supervisor; set to `Job` in `values/home-cluster-mvp.yaml`). The Job template reuses the same `_helpers.tpl::tenantLabels`, `args`, ConfigMap/Secret references, ServiceAccount/RBAC. Restoring `kubectl cp` reliability is the main behavioral benefit.
- **Review path:** Tenant decision; chart-author lives in DevOps. No ADR required (chart-internal change). Future loop's brief authoring can include this as a single objective; sized M (~30 min of templating + chart values update + SOP update).
- **Owner:** teams/platform/devops.
- **Timing:** **Pickup at next deployment-focused loop**, OR whenever DevOps has bandwidth between bigger deliverables. Not blocking; current Deployment-kind shape is documented as the MVP behavior.

### P3: Author `org-os/playbooks/reframe-vs-act.md` codifying the reframing pattern (class: **org-os**)

- **Problem it solves:** "What worked" #2. The reframing-vs-acting pattern (when a multi-loop blocker has the wrong shape, change the framing rather than try to clear the specific blocker) has now closed two recurring blocker classes in two consecutive loops (toolchain installs as Prerequisite docs at 2026-05-11-2153; minikube smoke as home-cluster deploy at 2026-05-12-0645). The pattern is broad enough to be reusable across many blocker types — but it's currently only an in-retro observation, not a documented playbook.
- **Proposed change:** Author a new playbook at `org-os/playbooks/reframe-vs-act.md` with `type: playbook`, `invocation_trigger: "when a blocker has carried 3+ loops with the same 'specific action' framing"`. Body sections: (a) when to invoke (3+ loop carry + same-shape action); (b) the question to ask ("is the blocker's framing itself the problem?"); (c) two worked examples (toolchain installs → Prerequisite docs at 2026-05-11-2153; minikube smoke → home-cluster deploy at 2026-05-12-0645); (d) anti-patterns (don't reframe when the action is genuinely blocking AND has a clear path; reframing is for "wrong-shape blockers" not "right-shape-but-stuck blockers"); (e) sister-pattern relationship to `provisional-and-migrate.md` (both are about coordination-shape decisions; both have worked-example anchoring).
- **Review path:** ADR-class change to `org-os/` (new playbook). Draft a placeholder ADR at `board/decisions/2026-05-12-001-reframe-vs-act-playbook.md` this loop; ratify at the next loop (one-loop-out, sister precedent: ADR-2026-06-06-001's same-loop draft-and-ratify happened with different timing). Recommended same-loop-draft + next-loop-ratify here because: (a) the playbook body needs more authoring than fits in this retro's authoring step; (b) one more loop of pattern-application would strengthen the body's worked examples.
- **Owner:** board.
- **Timing:** **Draft placeholder ADR this loop; ratify next loop.** Same shape as ADR-2026-06-06-001's draft-at-N + ratify-at-N+1 pattern.

### P4: Extend ADR-2026-05-16-003 (`Verified-against-environment` discipline) scope to `runbook` artifacts (class: **org-os**)

- **Problem it solves:** "What worked" #4. The discipline has now been adopted by three artifacts across three types (`contract`, `convention`, `runbook`). The ADR's original scope was `contract` + `rfc`; the `convention` adoption (DE at 2026-05-11-2153) was opportunistic; the `runbook` adoption (SRE this loop) was deliberate but not explicitly scoped in the ADR. **Extending the ADR's scope explicitly closes the gap between observed pattern and codified policy.**
- **Proposed change:** Extend ADR-2026-05-16-003's body to admit `convention`, `runbook`, and `playbook` artifacts to the discipline's scope (matching the pattern of opportunistic adoption seen across the last 3 loops). The grandfathering rule stays: required on new artifacts authored after the original ratification date OR on materially-revised existing artifacts. The body gains a "Sister artifact types" subsection listing the four canonical scope types (`contract`, `rfc`, `convention`, `runbook`, `playbook`). No `org-os/conventions.md` enum changes — the artifact-type enum already admits all five.
- **Review path:** Sister precedent: ADR-2026-05-23-001 was extended same-loop at 2026-05-30 (added `report`) and same-loop again at 2026-06-13 (added `action`). Pattern: extend an ADR in-place rather than spawn a new one when the change is additive to the same discipline. Recommend: **extend at next loop's brief authoring**, no separate ADR needed.
- **Owner:** board.
- **Timing:** **Picked up next loop alongside any other brief-time extension batch.**

### P5: Document the "first-real-deploy as integration test" pattern (class: **org-os**)

- **Problem it solves:** "What didn't" #3 + "What worked" #1. Helm lint + helm template + helm install --dry-run are necessary but not sufficient — they pass while latent bugs (Deployment-spec validation rules; runtime path dependencies) remain undiscovered. First-real-deploy on a live cluster is the deterministic acceptance gate. Without codification, future chart-bearing loops might exit the dry-run gates and consider the work "done" prematurely.
- **Proposed change:** Extend `org-os/playbooks/provisional-and-migrate.md` (the now-canonical playbook for coordination patterns) with a sister section "Sibling pattern: first-real-deploy as integration test." Names: (a) the dry-run-vs-real-deploy distinction; (b) when to invoke (any chart, container, or infrastructure-bearing loop); (c) the acceptance shape ("deploys to a real target environment, exercises the canonical happy path, exits cleanly on uninstall"); (d) worked example from this loop (`nanofab-supervisor` first home-cluster deploy + 2 latent bugs surfaced). Alternative: separate playbook at `org-os/playbooks/first-real-deploy.md`; recommend sibling-section because the discipline is about coordination through real-exercise vs. simulated checks (same shape as provisional-and-migrate's coordination-through-frontmatter-pattern).
- **Review path:** ADR-class change to a same-loop-just-ratified-three-loops-ago playbook. Recommend extending in place. **Recommend: bundle with P3** (which also authors an `org-os/playbooks/` file) — both are playbook-class extensions; bundle the two `org-os/` edits in a single authoring batch for tenant-isolation efficiency.
- **Owner:** board.
- **Timing:** **Picked up loop after P3's playbook lands** (so 2 loops out from this one).

## Decisions to record

New ADR placeholders this loop:

- **P1 needs no ADR** — tenant-class; brief's build-phase tool-call pattern only.
- **P2 needs no ADR** — chart-internal; values-flag-gated variant addition.
- **P3 generates a draft ADR placeholder at `board/decisions/2026-05-12-001-reframe-vs-act-playbook.md`** for next-loop ratification (one-loop-out pattern, sister to ADR-2026-06-06-001's draft-and-ratify cycle).
- **P4 extends ADR-2026-05-16-003 in-place** (sister to ADR-2026-05-23-001's same-loop extensions at 2026-05-30 and 2026-06-13).
- **P5 extends `org-os/playbooks/provisional-and-migrate.md` in-place** alongside P3's playbook authoring.

**Net new ADRs this retro: 1 (draft placeholder for P3).** P4 + P5 are in-place extensions to existing artifacts.

## Carryover ADRs / open work still on the books

- **Workspace promotion blocker** (board-action 2026-06-06-001): 7-loop carry; ticket stays `open`. P1 from this retro proposes the brief-author auto-runs the `gh` command.
- **GitBook publishing PR #1** (resink-ai/resink-ai.github.io): awaits user merge; post-merge verification carries to next loop or whichever loop the merge happens.
- **Sim-farm verdict-contract back-reference to DE convention:** mechanical migration; sim-farm next-active-loop deliverable.
- **Hot-swap correctness test (ADR-2026-05-16-001 step 3):** was loop+1; **slipped to loop+2** per this loop's deployment focus. Joint AE + resink-core deliverable.
- **`Chart.appVersion` ↔ supervisor crate version alignment:** bookkeeping; update at next chart revision.
- **Cargo MSRV declaration sync to Cargo.lock:** bookkeeping; bump workspace `rust-version = "1.85"` at next workspace-touch.

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: **passed**. Final tenant-isolation grep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns only the canonical `acme.ai` placeholder at `org-os/conventions.md:121`. **This loop achieved the smallest possible org-os surface impact** — zero `org-os/` writes by ANY team, by design (no ADR ratifications, no convention edits, no template changes). The discipline holds trivially because there's nothing to dry-run beyond reading. Pass.

This is the second consecutive loop with zero `org-os/` writes from product teams (2026-05-11-2153 had board-mandated 4-ADR ratification edits; this loop has none). The minimum-surface trend is healthy — `org-os/` edits should be load-bearing change, not housekeeping.
