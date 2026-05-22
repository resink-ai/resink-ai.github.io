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
| 2026-05-21 | 001 | [The executive loop is not complete at retro -- the rit-executive-loop skill gains an explicit loop-close step requiring loop artifacts and build-phase code to be committed, reviewed, merged, and (for publishing tenants) deploy-verified, with a fixed leaves-before-root submodule integration order.](2026-05-21-001-loop-close-integration-and-publishing.html) | active |
| 2026-05-16 | 004 | ["Codify the focus-loop pattern AND its consolidation-loop sibling as a single playbook at org-os/playbooks/focus-and-consolidation-loops.md; both patterns are CEO-invocable, both have explicit preconditions, both have explicit exit criteria, and they form a paired rhythm (focus loops produce structural deviations; consolidation loops close them)"](2026-05-16-004-focus-loop-pattern.html) | active |
| 2026-05-16 | 003 | ["Every cross-team contract or RFC artifact must include a 'Verified-against-environment' subsection naming toolchain versions, OS, runtime dependencies, and auth-mode prerequisites it was exercised against; new contracts comply going forward, existing contracts grandfather"](2026-05-16-003-contract-environment-verification.html) | active |
| 2026-05-16 | 002 | ["AE's Bundle B (7 org-os rituals tasks) becomes the hard floor for loop 2026-05-11-1113; AE refuses other loop-floor work until Bundle B ships or is explicitly blocked"](2026-05-16-002-bundle-b-hard-floor.html) | active |
| 2026-05-16 | 001 | ["Ratify ABI Option A (Cargo path-dep, no dlopen) as the named MVP runtime-spec deviation; name the multi-loop plan to restore dlopen via a `nanofab_node_process` C-ABI export"](2026-05-16-001-abi-option-a-mvp-deviation.html) | active |
| 2026-05-14 | 001 | [Name the supervisor's hardcoded dim_user/dim_account handling as a deliberate MVP deviation from the runtime spec's general-DAG-executor design; restore generality via a generic schema-driven node runner over a three-phase, multi-loop plan.](2026-05-14-001-general-tables-rearchitecture.html) | active |
| 2026-05-13 | 001 | ["Name the current parquet-eager event-source as a deliberate MVP deviation from the runtime spec's streaming architecture; introduce an `EventSource` trait abstraction with Kafka as the first real implementation; commit to a four-step multi-loop restoration plan owned across resink-core + devops + sim-farm"](2026-05-13-001-streaming-event-source-rearchitecture.html) | active |
{% endraw %}
