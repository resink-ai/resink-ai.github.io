---
layout: default
title: CEO Brief — 2026-05-11-1302
date: 2026-05-11
status: active
type: okr
loop: 2026-05-11-1302
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-11
  status: active
  loop: 2026-05-11-1302
  links: parent: ""
-->
# Resink.ai CEO Brief — 2026-05-30

## Context

The company has shipped two consecutive build-heavy loops: 2026-05-16 (MVP closed loop GREEN on a single-dim fixture, focus-loop pattern), 2026-05-23 (widened MVP GREEN on two dims + multi-shard partitioning, consolidation-loop pattern with a 6-ADR batch ratified). The product can demonstrably run end-to-end. **What the company cannot yet demonstrate is that it can explain itself.** Resink-core's product repo carries a one-page README pointing back at specs in newbase; there is no `CLAUDE.md`, no `docs/` tree, no module catalog, no architecture summary, no user-facing capabilities report. Every new IC (human or agent) opening `repos/resink-ai/resink-core/` must reverse-engineer the product surface from the Cargo workspace + the Python orchestrator + the Helm chart + the sim-farm subtree + the synthetic-tenant fixture. The "AI-native organization" claim from [`ORG.md`](../../ORG.md) presumes that future agents can orient themselves cheaply; today they cannot.

**The gap.** Distance from vision is now meta-structural: the company builds and demonstrates capabilities faster than it explains them. The MVP shipped two loops ago; the widened MVP shipped one loop ago; and there is still no canonical answer to "what does resink-core do today?" beyond reading the verdict file. A docs-only loop is overdue. The user's framing for this loop names the deliverable directly: documentation for resink-core, plus a single-file HTML capabilities summary that an outside reader can open and understand.

**The shape of this loop.** A **documentation loop** — the third named loop pattern after the focus loop (2026-05-16) and the consolidation loop (2026-05-23). Three teams activated:

- **resink-core** owns `rit-docs-init` for the product repo, producing `CLAUDE.md` + `docs/{architecture,concepts,user-guide,module-catalog}.md`.
- **AE** ships Bundle C (T13–T17 — `roles/EM.md`, `roles/CEO.md`, `roles/IC.md`, `org-os/README.md`, three worked-flow walks), closing the bottom-up flow's 17-task plan, plus the DISPATCH.md `--bare` addendum carried from loop 2026-05-11-1113.
- **board** ratifies the 2026-05-30 ADR batch (three ADRs in draft) and authors the single-file HTML capabilities report at `board/reports/2026-05-30-resink-core-capabilities.html`.

**Paused this loop:** sim-farm, DE, DevOps, SRE. Each receives a single light "review the resink-core docs PR" ask, no other floor pull. Their carryovers (sim-farm's schema-aware join columns, DE's `<verified-against-rust-impl>` tag flip, DevOps's minikube smoke, SRE's BLOCKED observable follow-up) carry to loop 2026-05-11-1631.

**CEO decisions for this loop:**

- **Activate three teams.** Resink-core + AE + board. Sim-farm + DE + DevOps + SRE paused with a single review-ack ask each (no exec-summary requirement; a one-paragraph ack note is sufficient).
- **Ratify the 2026-05-30 ADR batch and extend the conventions enum to admit `report`.** Three ADRs in draft this loop, all of which mandate edits to `org-os/`:
  1. [`2026-05-16-003-contract-environment-verification`](../decisions/2026-05-16-003-contract-environment-verification.md) — adds "Verified-against-environment" subsection to every cross-team contract; updates `org-os/conventions.md` (or `org-os/rituals/build.md`) per the ADR's "Decision" placeholder.
  2. [`2026-05-16-004-focus-loop-pattern`](../decisions/2026-05-16-004-focus-loop-pattern.md) — codifies the focus-loop pattern, **extended this loop to also codify the consolidation-loop sibling** (per 2026-05-23 retro P5). The output is a new `org-os/playbooks/focus-and-consolidation-loops.md` (or similar) with both patterns named, paired, and worked-example-linked.
  3. [`2026-05-23-001-conventions-enum-extension`](../decisions/2026-05-23-001-conventions-enum-extension.md) — adds `runbook`, `convention`, `playbook` to the `org-os/conventions.md` type enum. **This loop extends the ADR to also add `report`** so `board/reports/` becomes a legal artifact tree and the HTML capabilities summary lives at a conventions-legal path. The four types land in one ratification rather than spawning separate ADRs.
- **Author the HTML capabilities summary at `board/reports/2026-05-30-resink-core-capabilities.html`.** Self-contained (inline CSS, no JS), shippable to readers as a single file. Audience: technical reviewer (engineer, investor, prospective hire) who wants a concise overview of what resink-core can do *today* — not aspirations, not roadmap, just present-tense observable capabilities. Sources: resink-core's new `docs/` tree (produced earlier this loop), the on-disk state of `crates/`, `training/orchestrator/`, `synthetic_tenants/`, `sim-farm/`, `deploy/charts/`, plus the two verdict files (`workspace/verdict.json` from each loop). Owner: board.
- **Workspace promotion (resink-core KR2.1) is NOT in scope this loop.** Third consecutive defer. The board has not yet created the `resink-ai/resink-core` GitHub remote; until the remote exists, the promotion is blocked exclusively on a non-tenant action. Resink-core's docs work this loop produces a documentation surface that partially serves the same goal — making resink-core observable from outside the immediate repo tree — but does not substitute for the structural separation submodule promotion would provide. **Carry to loop 2026-05-11-1631 with a fourth-loop deferral flag for the retro.**

**Standing CEO answers (closing carryover loops):**

- **HTML capabilities summary location:** `board/reports/2026-05-30-resink-core-capabilities.html`. The `board/reports/` directory is new this loop; ADR `2026-05-23-001`'s extended ratification adds `report` to the conventions type enum so this artifact tree is convention-legal at write time.
- **Bundle C scoping:** T13 `org-os/roles/EM.md`, T14 `org-os/roles/CEO.md`, T15 `org-os/roles/IC.md`, T16 `org-os/README.md`, T17 three worked-flow walks (suggested in the plan: a `ceo-brief → team-planning → build → exec-summary → ceo-consolidation → retro` walk; a team-intake → request-lifecycle walk; an out-of-retro org-os change walk per the new ADR-2026-05-09-006 playbook). Five tasks; AE picks the placement of the worked-flow walks (likely under `org-os/playbooks/walks/` or appended to `org-os/README.md` — AE decides during planning).
- **`--bare` addendum:** AE adds a new section to `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/DISPATCH.md` documenting the OAuth-vs-env-var auth deviation that led resink-core to drop `--bare` in the orchestrator. Sized S; lands alongside Bundle C. Sourced from loop 2026-05-11-0958 retro P3-shaped ask; deferred at 2026-05-23 per ADR-002 strict floor.

**Carryover load by team** (per ADR-2026-05-09-005, now ratified for two loops):

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| resink-core | rit-docs-init outputs (CLAUDE.md + 4 docs); peer-team review-ack for the docs PR | M | Primary loop deliverable |
| AE | Bundle C (5 tasks: T13–T17), DISPATCH.md `--bare` addendum | M | Strict ADR-002 lineage — Bundle B → Bundle C succession |
| board | 3 ADR ratifications + mandated edits, HTML capabilities report | M | Highest-leverage loop work (the report is read by external audience) |
| sim-farm | Schema-aware join columns (retro P1 from 2026-05-23), Modes B/C scoping, sim-farm own repo decision | M | All paused; review-ack only this loop |
| DE | `<verified-against-rust-impl: pending>` → `verified` tag flip in Kafka contract §2.1; next-slice event-source contracts when consumer pressure arises | XS | Paused; review-ack only |
| DevOps | Minikube smoke execution (KR4.3 carryover); supervisor-side hand-off responses (5 items) | M | Paused; review-ack only |
| SRE | BLOCKED observable verification follow-up; Mode-B/C runbook expansion | M | Paused; review-ack only |
| resink-core (workspace promotion) | Third consecutive defer | — | Carries to 2026-06-06; retro flag |

## Objectives

### O1: resink-core ships rit-docs-init outputs for the product repo

source: ceo-brief

Why it matters: Resink-core has the largest in-tree product surface (3 Rust crates + 1 Python orchestrator + 1 fixture + 1 sim-farm subtree + 1 Helm chart skeleton + ~40 files) and no canonical orientation document. The vision in `ORG.md` presumes that every IC opening the repo can find the right pointer to the right file within a minute. Today that takes ten minutes of code archaeology. `rit-docs-init` directly closes this gap.

**Key results**
- KR1.1: `repos/resink-ai/resink-core/CLAUDE.md` exists at the repo root, ≤500 lines, follows the rit-docs-init template (One-paragraph summary; Tech stack; Common commands; Repository layout; Where to find more — pointers to the 4 `docs/` files). `AGENTS.md` exists as a symlink to `CLAUDE.md`.
- KR1.2: `repos/resink-ai/resink-core/docs/architecture.md` exists with sections covering: system shape (the three crates + orchestrator + sim-farm subtree + Helm chart), the MVP closed-loop seven-step flow, the ABI Option A static-linking shape (with forward-pointer to ADR-2026-05-16-001), the multi-shard partitioning model. Skeleton-but-non-trivial — each section has at minimum a one-paragraph body, not a TODO.
- KR1.3: `repos/resink-ai/resink-core/docs/concepts.md` exists naming the core vocabulary (Node, NodeCtx, Event, Scd2Row, manifest, release_seal, verdict, fixture, synthetic_tenant, workspace, shard, partition_key, dag) with one-paragraph definitions per term. Plain prose, no marketing.
- KR1.4: `repos/resink-ai/resink-core/docs/user-guide.md` exists covering: prerequisites (rustup, Python 3.12, `make`, `claude` CLI), `make mvp-loop` step-by-step, `make clean`, environment variables (`CLAUDE_SKIP_DISPATCH=1`), troubleshooting common failure modes (cargo-build failure, verdict file shape, parquet determinism). Commands match what's actually in the Makefile.
- KR1.5: `repos/resink-ai/resink-core/docs/module-catalog.md` exists with one entry per top-level module (`crates/nanofab-node-abi/`, `crates/nanofab-supervisor/`, `crates/nanofab-coordinator/`, `training/orchestrator/`, `synthetic_tenants/closed_loop_v0/`, `sim-farm/`, `deploy/charts/nanofab-supervisor/`). Each entry has: path, one-sentence purpose, who depends on it (resink-core teams + peer teams), entry point file path.
- KR1.6: Cross-team review-ack landed from sim-farm, DE, DevOps, SRE. Each reviews the docs PR and posts a one-paragraph ack in their `<team>/exec-summaries/2026-05-11-1302.md` (paused-team format) confirming their own surfaces in the docs are accurate.

**Tasks**
- [ ] Invoke `rit-docs-init` against `repos/resink-ai/resink-core/` — owner: teams/application/resink-core (skill writes the scaffold; resink-core fills the bodies).
- [ ] Fill the architecture.md body — owner: teams/application/resink-core.
- [ ] Fill the concepts.md body — owner: teams/application/resink-core.
- [ ] Fill the user-guide.md body against the actual Makefile — owner: teams/application/resink-core.
- [ ] Fill the module-catalog.md body from the actual directory tree — owner: teams/application/resink-core.
- [ ] Coordinate review-ack from the four paused teams — owner: teams/application/resink-core.

### O2: AE ships Bundle C and the DISPATCH.md `--bare` addendum

source: ceo-brief

Why it matters: Bundle C closes the 17-task bottom-up flow plan from ADR-2026-05-09-001. The org-os surface (rituals, roles, README, worked-flow walks) becomes fully self-describing — a new IC reading `org-os/README.md` can orient on the whole operating system in one document. The `--bare` addendum closes the loop-2026-05-16 DISPATCH.md gap that resink-core's orchestrator has been working around for two loops.

**Key results**
- KR2.1: `org-os/roles/EM.md` exists with `status: active` describing the Engineering Manager role: responsibilities (team-intake, OKR ownership, build coordination, exec summary authoring), authority bounds, escalation paths to CEO.
- KR2.2: `org-os/roles/CEO.md` exists with `status: active` describing the CEO role: brief authoring, decision rights per charter §"Decision rights", ratification of ADRs, retro convening.
- KR2.3: `org-os/roles/IC.md` exists with `status: active` describing the IC role (human or agent): task execution, OKR task-level ownership, blocker surfacing, tenant-isolation discipline, the per-loop "no codegen pattern expansion under hard-floor" pattern.
- KR2.4: `org-os/README.md` exists with `status: active`: one-paragraph overview of org-os; the rituals → playbooks → conventions → templates layout; pointer to the focus-and-consolidation-loops playbook (ratified this loop per ADR-2026-05-16-004 extension).
- KR2.5: Three worked-flow walks shipped. Suggested placement at `org-os/playbooks/walks/` (AE decides). Walk #1: `ceo-brief → team-planning → build → exec-summary → ceo-consolidation → retro` end-to-end (the canonical executive loop, using loop 2026-05-11-1113 as the worked example). Walk #2: `team-intake → request lifecycle` end-to-end (using a synthetic cross-team request). Walk #3: out-of-retro org-os change per ADR-2026-05-09-006's playbook.
- KR2.6: `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/DISPATCH.md` gains a "Auth modes" section documenting: when `--bare` is safe (env-var auth, CI environments); when to drop `--bare` (developer-laptop OAuth keychain); what determinism guarantees are preserved either way (template slot-fill is deterministic by content; auth mode does not affect codegen output). Marketplace tests pass (`bash tests/run-all.sh`).
- KR2.7: Tenant-isolation invariant holds across all Bundle C edits; dry-run after each task.

**Tasks**
- [ ] Author `org-os/roles/{EM,CEO,IC}.md` (T13–T15) — owner: teams/platform/agent-engineering.
- [ ] Author `org-os/README.md` (T16) — owner: teams/platform/agent-engineering.
- [ ] Author three worked-flow walks at the chosen path (T17) — owner: teams/platform/agent-engineering.
- [ ] Update DISPATCH.md with Auth modes section + run marketplace tests — owner: teams/platform/agent-engineering.
- [ ] Tenant-isolation dry-run after each task; cumulative record — owner: teams/platform/agent-engineering.

### O3: Board ratifies the 2026-05-30 ADR batch and authors the HTML capabilities report

source: ceo-brief

Why it matters: Three ADRs have been in `draft` across the last two retros (one from loop 2026-05-11-1113, two from loop 2026-05-11-0958). Ratification closes the org-os evolution discipline gap. The HTML capabilities report is the loop's headline external-facing deliverable: one single-file HTML page that an outside reader can open and understand what resink-core does today. Authoring is board-owned because (a) it spans multiple teams' surfaces and (b) the report is for external audiences, not internal docs.

**Key results**
- KR3.1: All three ADRs flip `status: draft → status: active` with full body content per `org-os/templates/adr.md` shape. The mandated edits land alongside:
  - **ADR-2026-05-16-003 (contract-environment-verification)**: Edit `org-os/conventions.md` (or `org-os/rituals/build.md`) to require every cross-team contract artifact to include a "Verified-against-environment" subsection naming toolchain versions, OS, runtime deps, auth-mode prerequisites. Retroactive verification not required this loop (existing contracts grandfather; new contracts comply).
  - **ADR-2026-05-16-004 (focus-loop pattern, extended)**: Author `org-os/playbooks/focus-and-consolidation-loops.md` (or chosen name) naming both patterns side-by-side. Focus loop: build-the-smallest-thing-that-exercises-every-component (worked example: loop 2026-05-11-0958). Consolidation loop: every-team-active-with-one-closing-seam (worked example: loop 2026-05-11-1113). The playbook names invocation preconditions, deferral discipline, exit criteria, and the rhythm "focus loops produce structural deviations; consolidation loops close them."
  - **ADR-2026-05-23-001 (conventions enum extension)**: Add `runbook`, `convention`, `playbook`, AND `report` to the `org-os/conventions.md` `type` enum. Add an "Additional fields per type" row for each. Existing files filed under workaround types (`type: rfc`) migrate to their new canonical types: SRE's `nanofab-supervisor-failed-validation.md` → `type: runbook`; DE's `conventions/duckdb.md` → `type: convention`; the new `org-os/playbooks/out-of-retro-org-os-change.md` from loop 2026-05-11-1113 → `type: playbook`; the new `board/reports/2026-05-30-resink-core-capabilities.html` → `type: report`.
- KR3.2: `board/reports/2026-05-30-resink-core-capabilities.html` exists. Single-file HTML with inline CSS, no JavaScript, no external assets. Content sections: (a) one-paragraph product summary (what resink-core does); (b) current capabilities (what works end-to-end today, with the verdict.json contents from loops 2026-05-11-0958 + 2026-05-23 as evidence); (c) architecture diagram (ASCII or inline SVG) showing the seven-step flow from fixture → orchestrator → cargo build → coordinator → supervisor → diff → verdict; (d) component catalog (each crate / module with one-line purpose); (e) named deviations + multi-loop restoration plan (Option A → dlopen, sim-farm schema-aware join columns, workspace promotion); (f) what's NOT in scope yet (real Kafka, real KV, multi-tenant, hot-swap, Modes B/C). Frontmatter: HTML `<meta name="type" content="report">`, `<meta name="owner" content="board">`, `<meta name="date" content="2026-05-30">`, `<meta name="status" content="active">` so the conventions enum validation hooks (when they exist) can verify.
- KR3.3: A companion `board/reports/2026-05-30-resink-core-capabilities.md` exists with the same content in markdown form (for grep-ability + diff-ability). The HTML is the canonical rendered form; the markdown is the source-of-truth for content edits. Both `status: active`.
- KR3.4: Tenant-isolation invariant holds: the ADR-mandated edits to `org-os/` introduce no new tenant/product strings. Dry-run after each ADR's mandated edit.

**Tasks**
- [ ] Author full body for ADR-2026-05-16-003; apply mandated edits to `org-os/conventions.md` (or `rituals/build.md`) — owner: board.
- [ ] Author full body for ADR-2026-05-16-004 (extended with consolidation-loop sibling); author `org-os/playbooks/focus-and-consolidation-loops.md` — owner: board.
- [ ] Author full body for ADR-2026-05-23-001 (extended with `report` type); edit `org-os/conventions.md` adding 4 types; migrate existing `type: rfc` files (SRE runbook, DE convention, ADR-2026-05-09-006 playbook) — owner: board.
- [ ] Author the markdown source `board/reports/2026-05-30-resink-core-capabilities.md` — owner: board.
- [ ] Render the HTML companion `board/reports/2026-05-30-resink-core-capabilities.html` — owner: board.
- [ ] Tenant-isolation dry-run after each ADR's mandated edit — owner: board.

## Risks

- **Three documentation deliverables in one loop is concentrated authoring work** (resink-core docs, AE Bundle C, board HTML report). Mitigation: the work is parallel rather than serial — resink-core's docs don't depend on AE's roles, AE's roles don't depend on the HTML, the HTML depends on resink-core's docs (Wave A: resink-core + AE; Wave B: board after Wave A). Within Wave A each team is independent.
- **Workspace promotion deferred for the third consecutive loop.** Mitigation already known: the docs surface this loop partially substitutes (resink-core becomes observable via CLAUDE.md + docs/ + HTML report from outside the immediate file tree). Not a substitute for structural separation but it lowers the urgency. Flagged for retro.
- **The HTML report could become a marketing artifact that drifts from reality.** Mitigation: KR3.2 names "what resink-core can do *today* — not aspirations, not roadmap, just present-tense observable capabilities." The verdict.json contents from loops 2026-05-11-0958 + 2026-05-23 are cited verbatim as evidence; if the report claims a capability that doesn't have a verdict-or-test backing, the claim doesn't ship.
- **ADR-2026-05-23-001 extension (adding `report` to the enum at the same time as `runbook`/`convention`/`playbook`) was a same-brief addition rather than a retro-class proposal.** Mitigation: this is an `org-os` change but the addition is mechanical and the rationale (HTML report needs a conventions-legal type) is documentable in the ADR body. The ADR is the gate; ratification this loop with the extended scope is consistent with the placeholder body.
- **Paused-team review-ack risk:** four teams are asked to review the docs PR but produce no other artifact. They may not engage if there's no failure mode for not engaging. Mitigation: KR1.6 is a hard KR (review-ack landed from each); if a team doesn't engage by mid-loop, resink-core escalates to the board. The ack format is one paragraph; the bar is "I read it; my surface is described accurately" or "I read it; section X is wrong, here's the correction."

## Out of scope this loop

- **Workspace promotion to a real git submodule** — deferred to loop 2026-05-11-1631; blocker is board action (creating the `resink-ai/resink-core` GitHub remote).
- **Real Kafka, real KV, real hot-swap, Modes B/C** — all carry forward unchanged.
- **Sim-farm schema-aware join columns** (P1 from loop 2026-05-11-1113 retro) — deferred to loop 2026-05-11-1631; the dim_account isomorphic-shape compromise persists this loop.
- **DevOps minikube smoke** — deferred to loop 2026-05-11-1631; the chart skeleton is documented in the HTML report as "shipped, smoke deferred."
- **`<verified-against-rust-impl: pending>` tag flip** in DE Kafka contract §2.1 — pure housekeeping; deferred.
- **AE codegen pattern expansion** (`scd1_first_event`, `window_stats_with_decrement`, etc.) — out of scope; the documentation work is AE's full bandwidth this loop.
- **Retroactive verified-against-environment sections** for the 4 existing contracts — ADR-2026-05-16-003 ratifies the requirement going-forward; existing contracts grandfather.
- **`dlopen` restoration execution** — ADR-2026-05-16-001's multi-loop plan stays on schedule (AE template extension targeted 2026-05-30, but the plan reschedules to 2026-06-06 since AE's full bandwidth this loop is Bundle C; this is a brief-level adjustment to the plan, not a re-litigation of the ADR).
- **Sim-farm Modes B/C** — full deferral; the HTML report names these explicitly as "not in scope yet."
- **Frontmatter-lint CI script** (DevOps) — still deferred per loop 2026-05-09-1715 ADR-001.
- **Resink-core stretch capacity decision** (sub-project #5 Product UX team standup vs internal split) — still deferred; revisit when the next MVP-class deliverable surfaces.
