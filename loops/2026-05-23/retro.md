---
layout: default
title: Retro
nav_order: 3
parent: "Loop 2026-05-23"
grand_parent: "Loops"
render_with_liquid: false
date: 2026-05-23
status: active
type: retro
loop: 2026-05-23
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-23
  status: active
  loop: 2026-05-23
  links: parent: board/exec-summaries/2026-05-23.md
-->
# Resink.ai CEO Retro — 2026-05-23

## What worked

- **The consolidation loop pattern is reproducible.** This loop was the converse of 2026-05-16's focus loop: stable MVP shape, six teams active, every team shipping its carryover in parallel, one closing seam (the widened verdict) anchoring integration risk. The pattern's tell: a "Carryover load by team" table in the brief that everyone reads first, plus team-level OKRs that operate against a settled ADR landscape (six ADRs land this loop, but the planning was done against their drafts). Reproducibility: when MVP-equivalent surfaces stabilize and pre-committed retro items are due, the next CEO brief activates everyone and gates on the largest outstanding cross-team contract (the widened verdict, this loop). Consolidation loops follow focus loops; the rhythm is alternating.
- **The two-wave merge plan held with zero conflict.** Board edited `org-os/rituals/{ceo-brief, exec-summary, ceo-consolidation, team-planning}.md` per ADR-003/005/006; AE edited the same files plus three more per Bundle B; no merge collision. The mechanism: board occupied step 1 + sub-steps 2a/2b/2c, AE placed its additions at non-overlapping positions (T8 new step 5 in ceo-brief.md, T10 new step 5 in exec-summary.md, T11 new step 4 in ceo-consolidation.md, T9 new subsection in team-planning.md). The Wave-2 mid-loop sync gate (2026-05-27) didn't even fire because board's edits were already on the working branch when AE started Wave 2. Reproducibility: when two teams have parallel work on the same ritual files, the brief names section-placement explicitly (e.g., "board edits step 2, AE adds a new step 5 elsewhere") so coordination is structural rather than negotiated.
- **ADR-002 hard-floor discipline worked exactly as predicted.** AE refused all non-Bundle-B floor pulls; the DISPATCH.md `--bare` addendum deferred cleanly; no codegen pattern was expanded; the 3-loop Bundle B slip pattern broke in one loop. The mechanism: ADR-002 was both the activation gate (ratified earlier in the same loop) AND the discipline AE honored end-to-end. The brief's "Out of scope this loop" section named every deferred item with a forward-pointer to 2026-05-30. Reproducibility: when a team has accumulated multi-loop carryover and external pressure won't relent, a hard-floor ADR with a single-loop activation window is the load-bearing tool — but it requires the brief to name the floor as the team's *only* deliverable AND to defer every conflict explicitly.
- **xxHash64 byte-stable verification across languages worked first-attempt.** DE computed the worked example in Python (`xxhash.xxh64(b"u-001", seed=0).intdigest() == 6277523119516880885`); resink-core's Rust implementation (`twox-hash::xxh64::xxh64(b"u-001", 0)`) matches exactly. The fixture (canonical UTF-8 bytes of a single primary-key value, seed=0) is small enough to round-trip the discipline without ambiguity. Reproducibility: cross-language byte-stable invariants are testable; the canonical test is "DE publishes worked example in one language; consumer reproduces in their language; CI gates on equality." This pattern generalizes to any cross-language data-shape invariant (e.g., the manifest YAML's structural-stability under YAML library variation; the verdict JSON's field ordering).
- **Real LLM codegen against a real ABI surface, scaled to two dims, with zero codegen-side changes.** AE's existing `nanofab:codegen-scd2-node` skill (from loop 2026-05-16) produced compilable Rust for both `dim_user` and `dim_account` schemas without any template edit; the orchestrator dispatched twice (once per dim) and got STATUS.json `status: ok` both times. This is the most consequential observation about the codegen system's robustness: the template-with-LLM-fill approach generalizes — slot-fill against schema JSON works for two schemas as cleanly as for one. Reproducibility: future pattern-library entries (`scd1_first_event`, `window_stats_with_decrement`, etc.) inherit this property; their templates are hand-authored once, then handle every schema that fits the pattern.
- **All Wave-1 peer contracts held under real Wave-2 consumption — again.** This is now the **third consecutive loop** the "named hand-off sections in one document" P3 convention has paid off (loops 2026-05-10, 2026-05-16, 2026-05-23). DevOps's chart was authored against resink-core's published 2026-05-10 OKR Helm hand-off section; SRE's runbook was authored against sim-farm's verdict contract; resink-core's supervisor partition was authored against DE's `xxHash64(seed=0)` worked example. **Zero same-loop addenda were filed**. Reproducibility: the convention is now a load-bearing organizational property, not a one-off observation.

## What didn't

- **Sim-farm's diff engine hard-codes `user_id` as join column.** Surfaced during resink-core's Wave-2 build of `dim_account`. Resink-core's workaround was an isomorphic compromise (account_id literals in the `user_id` column, account_type|status compound in `country` column). SCD2 + partitioning exercised correctly; only the column-name *semantics* are renamed. **Root cause**: the diff engine was authored against a single-dim MVP fixture (loop 2026-05-16) and the multi-dim extension (this loop) bumped the verdict format additively but didn't generalize the SQL projection. The engine reads schema-less parquets and assumes the canonical column layout matches `dim_user`. **Pattern risk**: any third dim with a primary key other than `user_id` triggers the same workaround; multi-table support is structurally incomplete until the engine becomes schema-aware. **Highest-priority retro item.** See P1.
- **Workspace promotion deferred for the second consecutive loop.** The OKR's permitted pivot was used (recorded blocker: `resink-ai/resink-core` GitHub remote doesn't exist). **Root cause**: the brief assumed the remote existed; nobody on the loop is in a position to create the GitHub repo (board-level / external action). **Pattern risk**: every loop that adds substantial in-tree content (three crates last loop, two more this loop, plus deploy charts, plus pytz conventions) compounds the eventual migration cost. **Mitigation already in place**: the retro carries this to 2026-05-30 with explicit board action. See P2.
- **DevOps minikube smoke deferred** because no docker/minikube on the IC's developer machine. **Root cause**: the brief assumed the standard developer-laptop toolchain (`rustup`, Python, `make`, `claude` CLI) but minikube + docker were never on that list. **Pattern risk**: chart-skeleton-builds-but-can't-be-deployed becomes a recurring DevOps-shaped friction. **Mitigation**: P3 — either provision docker/minikube on the canonical dev machine OR pivot the smoke to Docker-Desktop-on-Mac (which is already installed).
- **`type: runbook` not in conventions enum forced a workaround on SRE's runbook.** SRE filed `type: runbook` as a "charter-grounded SRE-local extension" with workaround documented inline. **Root cause**: ADR-2026-05-10-004 (contract type addition) addressed `contract` but didn't generalize. The conventions enum grows one type at a time. **Pattern risk**: every new owned-artifact-type that doesn't fit the enum repeats the workaround. **Mitigation**: P4 — small ADR to add `runbook` (and possibly anticipate other types like `convention`, `worked-example`, `playbook`) in one batched evolution.

## Evolution proposals

### P1: Sim Farm diff engine becomes schema-aware (class: **product**)

- **Problem it solves:** "What didn't" #1. The engine hard-codes `user_id` as join column and `(email, country, valid_to, is_current)` as payload, blocking genuine multi-table support. Resink-core's `dim_account` shipped via an isomorphic compromise; a third dim or a primary key other than `user_id` cannot use the engine without renaming.
- **Proposed change:** Sim-farm extends the diff engine CLI to accept a `--schema <path>` flag per `--fixture/--output/--dim-table` triple, OR a sibling `<fixture-name>.schema.json` discovery convention. The schema-JSON shape is the same one AE's codegen consumes (the `dim_<table>.schema.json` files resink-core's `generate.py` emits) — `{table, primary_key, scd2_columns, columns}`. The engine reads the primary_key to construct the join condition and reads the columns to construct the payload projection. Backward compat: if no schema is passed and only one pair is given, fall back to current hard-coded behavior with a deprecation note in stderr. Engine_version bumps to `0.3.0` (additive).
- **Review path:** Product change; no ADR required (sim-farm's product surface). The contract update is additive to `2026-05-16-mvp-loop-verdict.md` (engine_version field bumps; no field renaming).
- **Owner:** sim-farm, with interface input from DE (canonical schema-JSON shape — `org-os/templates/` or a new convention?) and resink-core (orchestrator passes the schema).
- **Timing:** **Picked up loop 2026-05-30.** Sized M; sim-farm's main loop work plus a 2-week-old retro item should fit comfortably alongside Mode-B scoping start. Resink-core's dim_account fixture migrates back to its canonical shape (account_id PK, real `account_type`/`status` columns) once the engine is schema-aware.

### P2: Approve the `resink-ai/resink-core` GitHub remote so workspace promotion completes (class: **tenant**)

- **Problem it solves:** "What didn't" #2. Two consecutive loops of deferred workspace promotion compound in-tree drift; resink-core's product code grows monotonically and the eventual migration cost grows with it.
- **Proposed change:** Board (the human founder, per `board/charter.md` "CEO held by the human founder during Phase 1") creates `https://github.com/resink-ai/resink-core.git` (empty repo) before loop 2026-05-30 opens. resink-core's first 2026-05-30 KR is the submodule promotion: `git submodule add` + `.gitmodules` entry + initial-commit-on-submodule from the in-tree contents + README deferral note removal.
- **Review path:** Tenant decision (no `org-os/` change); lives in the human's GitHub admin workflow. No ADR required.
- **Owner:** board (human founder); resink-core executes the submodule promotion downstream.
- **Timing:** **Picked up loop 2026-05-30** (resink-core's KR2.1 third attempt).

### P3: Provision developer-machine docker/minikube OR pivot DevOps smoke to Docker-Desktop-on-Mac (class: **tenant**)

- **Problem it solves:** "What didn't" #3. DevOps's chart-skeleton smoke target is reviewable but unexecuted because the canonical developer machine lacks docker/minikube. Without a working smoke, the chart's `helm install --dry-run` exit code is the only signal; live deployment is unverified.
- **Proposed change:** Three options, board picks one:
  - **(a) Install docker + minikube** on the canonical developer machine (`brew install docker minikube`; configure Docker Desktop OR use minikube's built-in driver). Most direct; matches the spec's minikube assumption.
  - **(b) Pivot the chart smoke to Docker-Desktop-on-Mac.** Docker Desktop is already available. The smoke becomes `docker build + docker run` (skipping the k8s layer) — exercises the binary but not the chart's deployment semantics.
  - **(c) Provision a shared minikube target** (a board-owned VM with docker/minikube pre-installed; DevOps SSHes in for smoke runs). Highest setup cost; useful if multiple consumers want a shared k8s sandbox eventually.
- **Review path:** Tenant decision; no `org-os/` change.
- **Owner:** board (chooses); DevOps executes the chosen pivot.
- **Timing:** **Picked up loop 2026-05-30** so DevOps's KR4.3 closes.

### P4: Extend conventions enum to admit `runbook` (and possibly batch other anticipated types) (class: **org-os**)

- **Problem it solves:** "What didn't" #4. SRE's runbook filed as `type: runbook` with workaround documented inline. The conventions enum should admit owned-artifact-types that teams produce rather than forcing one ADR per type-addition.
- **Proposed change:** Add `runbook` to the type enum in `org-os/conventions.md`. Add an "Additional fields per type" row: `runbook` | `severity_tiers: [<list>]` (or appropriate fields per SRE's runbook structure). Consider batching: also add `convention` (DE's `conventions/duckdb.md` could use this — currently `type: rfc`) and `playbook` if the new `org-os/playbooks/out-of-retro-org-os-change.md` would benefit. Each addition is one line in the enum + one row in the additional-fields table.
- **Review path:** ADR mandatory (`org-os` change). Draft placeholder at [`board/decisions/2026-05-23-001-conventions-enum-extension.md`](../decisions/2026-05-23-001-conventions-enum-extension.md).
- **Owner:** board (with input from SRE on `runbook` fields, DE on `convention` fields).
- **Timing:** **Picked up loop 2026-05-30** alongside the existing 2026-05-30 batch (ADR-2026-05-16-003 contract-environment-verification, ADR-2026-05-16-004 focus-loop pattern, AE's DISPATCH.md `--bare` addendum, AE's Bundle C). Sized XS.

### P5: Document the consolidation-loop-follows-focus-loop rhythm (class: **org-os**)

- **Problem it solves:** This loop demonstrated a coherent organizational mode that is the natural sibling to 2026-05-16's focus loop — and the pair together (focus → consolidation) is a more powerful framing than either alone. Without naming the rhythm, future CEOs may not recognize when consolidation is the right move.
- **Proposed change:** Update `org-os/playbooks/focus-loop.md` (drafted in ADR-2026-05-16-004, scheduled to ratify 2026-05-30) to name the converse: **consolidation loop**. Definition: invoked when (a) a focus loop's MVP-equivalent has stabilized, (b) multiple teams carry deferred work from the focus, (c) the next gate is verifying the MVP works at scale. Shape: every team active, pre-committed retro items ratify, the largest cross-team contract anchors integration risk, no codegen-side / spec-side change is required. Names this loop (2026-05-23) as the worked example, paired with 2026-05-16 as the focus-loop example.
- **Review path:** Folds into ADR-2026-05-16-004's ratification at 2026-05-30 (extend rather than separate ADR). No new ADR file; the existing placeholder body grows.
- **Owner:** board.
- **Timing:** **Picked up loop 2026-05-30** as part of ADR-2026-05-16-004 ratification work.

## Decisions to record

New ADRs this loop:

- **ADR `2026-05-23-001` (P4)** "Extend conventions type enum (runbook + possibly batched additions)." Draft placeholder at [`board/decisions/2026-05-23-001-conventions-enum-extension.md`](../decisions/2026-05-23-001-conventions-enum-extension.md). Status: `draft`. Owner: board. Picked up loop 2026-05-30. Class: org-os.

(P1 needs no ADR — product-class, lives in sim-farm's contract evolution.)
(P2 needs no ADR — tenant-class, GitHub admin action.)
(P3 needs no ADR — tenant-class, developer-machine provisioning.)
(P5 needs no ADR — extends the existing ADR-2026-05-16-004 placeholder rather than spawning a new one.)

## Carryover ADRs still on the books

Six ADRs ratified to `active` this loop (covered in detail in the company exec summary). Three ADRs remain `draft` carrying into the 2026-05-30 batch:

- **ADR `2026-05-16-003` (P3 from 2026-05-16 retro):** "Contract-against-environment verification." Draft. **2026-05-30 batch.**
- **ADR `2026-05-16-004` (P5 from 2026-05-16 retro):** "Codify the focus-loop pattern" — now extends with the consolidation-loop sibling per this retro's P5. Draft. **2026-05-30 batch.**
- **ADR `2026-05-23-001` (P4 this retro):** "Extend conventions type enum." Draft. **2026-05-30 batch.**

The 2026-05-30 batch is now: ADR-2026-05-16-003 + ADR-2026-05-16-004 (extended) + ADR-2026-05-23-001 = 3 ADRs. AE's Bundle C (T13–T17 — 5 ritual / role files) + DISPATCH.md `--bare` addendum also fold into 2026-05-30. Lighter than this loop's 6-ADR batch.

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: **passed**. The board ran `grep -rEi "resink|nanofab|acme\.ai" org-os/` after each ADR-mandated ritual edit; AE ran it after each of the 7 Bundle B task edits. Combined: only the pre-existing `acme.ai` placeholder reference in `org-os/conventions.md` matched. Zero new tenant/product names introduced anywhere in `org-os/` this loop. The new `org-os/playbooks/out-of-retro-org-os-change.md` (created per ADR-2026-05-09-006) and the new `org-os/rituals/team-intake.md` (created per AE Bundle B T6) both inspected as part of the dry-run. Pass.
