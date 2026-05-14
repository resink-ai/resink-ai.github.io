---
layout: default
title: Decisions (ADRs)
has_children: true
nav_order: 3
---
{% raw %}
# Decisions (ADRs)

All architectural decision records, sorted by date (newest first). Both `draft` and `active` ADRs appear here.

| Date | # | Title | Status |
|---|---|---|---|
| 2026-06-13 | 001 | ["Admit type: action to the org-os conventions enum so board-action tickets become canonical artifacts; migrate the two existing 2026-06-06-001 and 2026-06-06-002 board-action tickets from provisional to canonical in the same loop"](2026-06-13-001-admit-action-to-conventions-enum.html) | active |
| 2026-06-06 | 002 | ["Paused teams accepting in-loop requests record the acceptance canonically in the request file's frontmatter (status field + deferred_to_loop), not in status.md; status.md may reference the request file by path but is not source-of-truth"](2026-06-06-002-paused-team-request-acceptance.html) | active |
| 2026-06-06 | 001 | ["Codify the 'provisional-and-migrate' coordination pattern as a documented org-os playbook, with three worked examples from prior loops"](2026-06-06-001-provisional-and-migrate-playbook.html) | active |
| 2026-05-30 | 002 | ["Multi-loop plans inside ADRs name dates as `loop+N` relative to the ADR's ratification loop, with an explicit 'subject to team capacity at that loop's brief' qualifier; absolute calendar dates only when external commitments require them"](2026-05-30-002-multi-loop-plan-relative-dating.html) | active |
| 2026-05-30 | 001 | ["Team OKRs that reference predicted-but-unverified syntax (CLI flags, file paths, env vars) must carry an explicit verification marker; the build phase confirms or files an addendum"](2026-05-30-001-verify-predicted-syntax-in-okrs.html) | active |
| 2026-05-23 | 001 | ["Extend the type enum in org-os/conventions.md to admit five additional owned-artifact types in one ratification batch: `runbook`, `convention`, `playbook`, `report`, and `role`; add a row to the additional-fields-per-type table for each; migrate existing files filed under workaround types to their new canonical types"](2026-05-23-001-conventions-enum-extension.html) | active |
| 2026-05-16 | 004 | ["Codify the focus-loop pattern AND its consolidation-loop sibling as a single playbook at org-os/playbooks/focus-and-consolidation-loops.md; both patterns are CEO-invocable, both have explicit preconditions, both have explicit exit criteria, and they form a paired rhythm (focus loops produce structural deviations; consolidation loops close them)"](2026-05-16-004-focus-loop-pattern.html) | active |
| 2026-05-16 | 003 | ["Every cross-team contract or RFC artifact must include a 'Verified-against-environment' subsection naming toolchain versions, OS, runtime dependencies, and auth-mode prerequisites it was exercised against; new contracts comply going forward, existing contracts grandfather"](2026-05-16-003-contract-environment-verification.html) | active |
| 2026-05-16 | 002 | ["AE's Bundle B (7 org-os rituals tasks) becomes the hard floor for loop 2026-05-11-1113; AE refuses other loop-floor work until Bundle B ships or is explicitly blocked"](2026-05-16-002-bundle-b-hard-floor.html) | active |
| 2026-05-16 | 001 | ["Ratify ABI Option A (Cargo path-dep, no dlopen) as the named MVP runtime-spec deviation; name the multi-loop plan to restore dlopen via a `nanofab_node_process` C-ABI export"](2026-05-16-001-abi-option-a-mvp-deviation.html) | active |
| 2026-05-13 | 001 | ["Name the current parquet-eager event-source as a deliberate MVP deviation from the runtime spec's streaming architecture; introduce an `EventSource` trait abstraction with Kafka as the first real implementation; commit to a four-step multi-loop restoration plan owned across resink-core + devops + sim-farm"](2026-05-13-001-streaming-event-source-rearchitecture.html) | active |
| 2026-05-12 | 002 | [Author an org-os playbook codifying the in-tree → submodule promotion pattern using `git subtree split`.](2026-05-12-002-submodule-promotion-playbook.html) | active |
| 2026-05-12 | 001 | [Author an org-os playbook codifying the "reframe-vs-act" pattern for multi-loop blockers whose framing itself may be wrong.](2026-05-12-001-reframe-vs-act-playbook.html) | active |
| 2026-05-10 | 004 | [Add `contract` to the `type` enum in `org-os/conventions.md` with required `consumers:` and `producers:` fields; migrate existing rfc-typed contracts to `type: contract` same loop.](2026-05-10-004-contract-artifact-type.html) | active |
| 2026-05-10 | 003 | [State-claims (shipped / merged / reserved on master) at exec-summary, ceo-brief, and ceo-consolidation ritual transitions must carry a verifiable git location, or be re-phrased as "shipped to branch <X>, awaiting merge."](2026-05-10-003-verify-state-claims-at-ritual-transitions.html) | active |
| 2026-05-10 | 002 | [Decompose the resink.ai product into five named nanofab sub-projects (Runtime, Training Pipeline, Sim Farm, Serving & Deployment, Product UX) and assign team ownership; resink-core stretches to own #1 and #2; sim-farm owns #3; DevOps + SRE split #4; #5 ownership is deferred to a future loop.](2026-05-10-002-nanofab-sub-project-decomposition.html) | active |
| 2026-05-10 | 001 | [The nanofab runtime is a purpose-built distributed Rust application; no JVM, no general SQL execution layer. Spark Structured Streaming (ADR-002) is superseded.](2026-05-10-001-nanofab-runtime-is-rust.html) | active |
| 2026-05-09 | 006 | [Org-os change proposals that surface mid-loop (not at retro time) route through a dedicated team-proposals path and are picked up as candidate evolution proposals in the next CEO brief.](2026-05-09-006-org-os-change-routing.html) | active |
| 2026-05-09 | 005 | [Every CEO brief includes a "Carryover load by team" tally so brief commitments visibly account for deferred work entering the loop.](2026-05-09-005-carryover-load-in-brief.html) | active |
| 2026-05-09 | 004 | [Adopt onboard-application-team.md as the canonical playbook for an application team's first product-shaped OKR.](2026-05-09-004-onboard-application-team.html) | active |
| 2026-05-09 | 003 | [Adopt minikube (single-node local k8s) as the deployment target for the first fact_sign_up.parquet demo; manifests are portable to cloud k8s when production demos land.](2026-05-09-003-deployment-target.html) | active |
| 2026-05-09 | 002 | [Adopt Spark Structured Streaming as the streaming engine for the first fact_sign_up.parquet demo; revisit if a future demo surfaces a constraint Spark cannot meet.](2026-05-09-002-streaming-engine-choice.html) | archived |
| 2026-05-09 | 001 | [Adopt the org-os bottom-up flow design (team-initiated work + cross-team requests) as specified in docs/superpowers/specs/2026-05-09-org-os-bottom-up-flow-design.md; AE owns implementation via the 17-task plan in docs/superpowers/plans/2026-05-09-org-os-bottom-up-flow.md.](2026-05-09-001-org-os-bottom-up-flow.html) | active |
| 2026-05-08 | 002 | [Tenant tree restructured — board/ for executive desk, teams/{platform,application}/ for workforce, agent-engineering/data-engineering full-name slugs, application/realtime-pipeline → teams/application/resink-core](2026-05-08-002-tenant-tree-restructure.html) | active |
| 2026-05-08 | 001 | [Org-OS docs require frontmatter validation before merge](2026-05-08-001-frontmatter-validation.html) | active |
{% endraw %}
