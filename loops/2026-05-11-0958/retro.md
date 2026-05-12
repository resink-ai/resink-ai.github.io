---
layout: default
title: Retro — 2026-05-11-0958
date: 2026-05-11
status: active
type: retro
loop: 2026-05-11-0958
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-11
  status: active
  loop: 2026-05-11-0958
  links: parent: board/exec-summaries/2026-05-11-0958.md
-->
# Resink.ai CEO Retro — 2026-05-16

## What worked

- **The MVP-focus framing landed exactly what it was designed to land.** A CEO-called focus loop, four activated teams, two paused teams, one synthetic single-dim fixture, one closing seam — and the loop's exit criterion (`make mvp-loop` exits 0 with `verdict=pass`) closed on the first end-to-end attempt with real LLM codegen. The decision to defer Bundle B + ADR-003 was the load-bearing constraint: it freed AE bandwidth onto the training-pipeline subagent surface, which was the integration gate the MVP needed. Reproducibility: when the company drifts into "two consecutive plan-only loops" territory, the CEO brief invokes "build the smallest closed loop that exercises every component," activates only the teams on the critical path, and pauses everyone else explicitly. See P6.
- **Template-with-LLM-fill chosen pre-emptively, not as a fallback.** AE's planning decision moved the template approach from "permitted fallback if pure-LLM proves brittle" (CEO brief Risk #2 framing) to "primary approach from the start." On first dispatch, the slot-filled `lib.rs` compiled in 1.81s with zero warnings and zero template iterations. Pure-LLM codegen would almost certainly have needed prompt-iteration loops. Reproducibility: when an AE deliverable has a well-defined output shape and the LLM's job is to fill named slots (not architect), declare template-with-fill as primary in the team OKR's "Decisions made this loop" section so the build phase doesn't waste time on retry loops.
- **Wave-1 / Wave-2 dispatch sequencing produced zero same-loop contract addenda.** AE, DE, and sim-farm built their contracts and tools in Wave 1; resink-core consumed all three in Wave 2. Every Wave-1 peer artifact resink-core consumed in Wave-2 conformed to its published contract byte-for-byte. No addendum-to-the-addendum was filed; no contract changed mid-loop. The CEO brief's framing risk "three peer hand-offs all due 2026-05-19 means a single peer slip cascades" never materialized. Reproducibility: for focus loops where contracts cross team boundaries, dispatch producers in Wave 1 and consumers in Wave 2; do not parallelize the consumer against unpublished contracts.
- **"Named permitted pivots" turned scope changes into pre-authorized choices.** Resink-core's OKR pre-named three permitted pivots (CSV fallback if parquet non-deterministic; submodule promotion deferral with recorded blocker; hand-written template if codegen brittle). Only one fired (submodule deferral); the other two never had to be exercised, but their existence meant the team could have pivoted without re-litigating scope mid-loop. AE's "template-with-fill" was the same pattern at a different level — pre-naming the approach removed retro-time questions about why an OKR commitment changed shape. Reproducibility: every non-trivial KR carrying a "what if this fails" question gets a one-line "permitted pivot" entry in the OKR; the team can land the pivot without escalating.
- **Real LLM dispatch from a Python orchestrator via the local `claude` CLI worked.** Cost: $0.77. Time: ~106s on developer laptop. STATUS: ok on first try. The training-pipeline architecture (training spec §4.1) — orchestrator dispatches per-pattern subagents via a stateless CLI surface — is now demonstrated, not aspirational. This is the most consequential MVP-side outcome after the closed loop itself: the company has a working end-to-end customer-facing codegen path on a single fixture.
- **Determinism discipline held end-to-end.** Six independent determinism guarantees stacked: (1) `generate.py` seeded byte-stable parquet; (2) `derive_facts.py` deterministic event_id; (3) AE template slot-fill byte-stable; (4) `cargo build --release` produces a stable `.rlib`; (5) supervisor sort-by-`(event_ts, event_id)` deterministic; (6) `trace.jsonl` + `dim_user_output.parquet` byte-identical across runs (verified). Every layer's determinism was a precondition for the diff to be a meaningful verdict — if any one had been wobbly, sim-farm's verdict would have been suspect. Reproducibility: determinism is a property of the whole stack, not any one component; the team OKR's "determinism smoke" test verifies the composition.

## What didn't

- **The codegen template's C-ABI is incomplete (`_new` + `_drop` but no `_process`).** Pure `dlopen` is impossible. Resink-core had to pivot to Option A (static linking via Cargo path-dep), which sacrifices the runtime spec's "hot-swap node code without restarting supervisor" property for MVP simplicity. **Root cause:** the gap was findable at the SKILL.md / DISPATCH.md design stage — the template authors knew the supervisor would need to call `process()`, but the C-ABI export set in the template stopped at object lifecycle. The skill's "Acceptance contract" subsection committed only to "the produced crate compiles," not to "the produced crate is loadable by the supervisor and drives events." Pattern risk: any future codegen pattern can ship a contract that compiles in isolation but doesn't cooperate with the consumer's loading path. See P1 (ratify Option A + multi-loop dlopen plan) and P4 (template completeness checks).
- **Contracts referenced idealized environments, not verified ones.** AE's DISPATCH.md prescribes `claude --bare` for determinism; the local `claude 2.1.138 --bare` enforces env-only auth and refuses OAuth — so resink-core's orchestrator dropped `--bare` and the contract leans on conventions documented elsewhere. Sim-farm's verdict contract implicitly promised "no extra installs" but the DuckDB Python driver requires `pytz` at runtime for TIMESTAMPTZ — sim-farm caught it mid-build. **Root cause:** contracts were specified against the spec, not the developer-laptop environment they were exercised in. Pattern risk: every cross-team contract becomes a hidden dependency on the consumer's environment matching the producer's. See P5 (contract-against-environment validation).
- **Two consecutive Bundle B deferrals are a real pattern, not a one-off slip.** 2026-05-10 deferred B for a mid-loop ADR-gate slip; 2026-05-16 deferred B for explicit CEO MVP-focus. Different causes, same surface effect. AE absorbed each deferral silently — no explicit board-level decision either time that "B has slipped its third loop" carries different consequences than the previous loops' slippage. **Root cause:** Bundle B is internal org-os work with no external consumer pressure, so it loses to any CEO-priority pull. The status quo (B keeps slipping a loop at a time) is an attractor. See P2 (Bundle B hard floor vs re-slice decision).
- **The `claude --skill` flag predicted in three team OKRs doesn't exist.** AE, resink-core, and the CEO brief all referenced `claude --skill nanofab:codegen-scd2-node` as the dispatch shape. The actual CLI form is `claude --bare --plugin-dir <plugin> --print "/<plugin>:<skill> {json}"`. AE caught it during build and DISPATCH.md documents the corrected form, but the OKR claim about the dispatch surface was loose for a full loop's planning artifact. **Root cause:** the predicted flag was placeholder text that survived three rounds of OKR authoring without being checked against the actual CLI. Pattern risk: predicted-but-unverified syntax in planning artifacts. Mitigated by P5.

## Evolution proposals

### P1: ADR — Ratify ABI Option A as the MVP runtime-spec deviation; name the multi-loop `dlopen` restoration plan (class: **product**)

- **Problem it solves:** Resink-core shipped Option A (Cargo path-dep, static linking) because the codegen template lacks a `nanofab_node_process` C-ABI export. The runtime spec's "hot-swap node code without restarting supervisor" property is sacrificed for MVP; the supervisor binary is rebuilt per-tenant. This is a real spec deviation that future ICs reading the runtime spec need to see documented, with a multi-loop plan to restore the `dlopen` surface.
- **Proposed change:** ADR records: (a) Option A is the MVP-shape and remains active until the multi-loop plan completes; (b) AE updates the template to export `nanofab_node_process(node_ptr, event_json_bytes, event_json_len, ctx_callback_ptr) -> result_code` (or equivalent C-ABI surface — exact shape TBD in the ADR's "Decision" section); (c) resink-core's supervisor swaps from Cargo path-dep to `libloading::Library::open(...)` once the template ships the new symbol; (d) the swap lands in a loop where neither AE's template nor the supervisor is on the critical path for another deliverable, so the swap can be measured for hot-swap correctness without contention.
- **Review path:** Product change; ADR voluntary but recommended for a deviation this consequential. Filed as ADR `2026-05-16-001` to record the decision and the multi-loop plan.
- **Owner:** board (ratify); AE owns the template extension; resink-core owns the supervisor swap.
- **Timing:** **ADR lands loop 2026-05-11-1113** (ratification). Multi-loop plan: AE template extension targeted 2026-05-30; supervisor swap 2026-06-06. Hot-swap test against a "candidate" plugin version: loop after the swap lands.

### P2: ADR — Bundle B becomes a hard floor at loop 2026-05-11-1113 (class: **org-os**)

- **Problem it solves:** Bundle B has now slipped two consecutive loops (2026-05-10 mid-loop ADR-gate; 2026-05-16 explicit CEO MVP-focus). Each slip was individually justified but the cumulative carry compounds: B is foundational org-os ritual work that, until it lands, holds back team-intake and the formal request-lifecycle closure surfaces. The status quo of single-loop slips is no longer acceptable as an implicit answer; AE flagged it as a pattern in their exec summary.
- **Proposed change:** The 2026-05-23 CEO brief explicitly names Bundle B as AE's primary floor work. AE refuses other loop-floor pulls (no codegen patterns, no MVP-shaped deliverables) until Bundle B's 7 tasks (T6 `team-intake.md`, T7 `executive-loop.md` insert, T8 `ceo-brief.md` ratification, T9 `team-planning.md` three-source decomposition, T10 `exec-summary.md` request-lifecycle closure, T11 `ceo-consolidation.md` roll-ups, T12 `retro.md` one-line) are shipped or explicitly blocked on a non-AE dependency. Alternative considered: re-slice B into 2-3 smaller chunks that absorb alongside MVP-shaped work. Rejected because (a) re-slicing is itself work, (b) the existing 7-task bundle is already small (one ritual file per task), (c) two consecutive loops without B means the cost of further slip exceeds the cost of explicit prioritization.
- **Review path:** ADR mandatory (`org-os` change to AE's loop-floor commitment). Draft placeholder at [`board/decisions/2026-05-16-002-bundle-b-hard-floor.md`](../decisions/2026-05-16-002-bundle-b-hard-floor.md).
- **Owner:** board.
- **Timing:** **Picked up loop 2026-05-11-1113** — the loop the ADR ratifies is the same loop the floor activates.

### P3: ADR — Contract-against-environment verification (class: **org-os**)

- **Problem it solves:** "What didn't" #2. Cross-team contracts this loop (AE's DISPATCH.md `--bare` prescription; sim-farm's verdict contract's `pytz` omission; the predicted-but-nonexistent `claude --skill` flag in three OKRs) referenced idealized specs, not the actual developer-laptop environment they were exercised in. The consumer team picked up each gap mid-build; the contract authors had no per-loop discipline forcing them to verify against the runtime environment.
- **Proposed change:** Update `org-os/conventions.md` (or `org-os/rituals/build.md`): every cross-team contract or RFC artifact filed under a team's `contracts/` directory must include a "Verified-against-environment" subsection naming: (a) the toolchain versions (`cargo --version`, `rustc --version`, `python --version`, `claude --version`, etc.) the contract was exercised against; (b) the OS + arch; (c) any runtime dependencies the contract assumes are available; (d) any auth-mode prerequisites (env vars, OAuth, etc.). The consumer team reads this section first; if their environment differs, they surface the delta in their OKR's cross-team asks before consuming the contract. Sister proposal to ADR `2026-05-10-003` (verify-state-claims at ritual transitions): same family, different scope.
- **Review path:** ADR mandatory. Draft placeholder at [`board/decisions/2026-05-16-003-contract-environment-verification.md`](../decisions/2026-05-16-003-contract-environment-verification.md).
- **Owner:** board (with input from AE on the section template shape).
- **Timing:** **Deferred to loop 2026-05-11-1302.** Same-loop bandwidth conflict with ADR `2026-05-10-003` (verify-state-claims) — both touch the same conventions/rituals files and AE's Bundle B is the dominant org-os work at 2026-05-23. P3 pairs naturally with ADR-003 at 2026-05-30 once Bundle B lands.

### P4: Template completeness checks before declaring a codegen skill ready (class: **product**)

- **Problem it solves:** "What didn't" #1's root cause. The `nanofab_node_process` C-ABI gap was findable at template design time; the SKILL.md / DISPATCH.md "Acceptance contract" subsection committed only to "the produced crate compiles," not "the produced crate is loadable and drivable by the consumer." Future codegen patterns risk shipping the same shape: compiles, doesn't cooperate.
- **Proposed change:** AE's codegen-skill design discipline gains a "Completeness check" subsection in DISPATCH.md per pattern: a list of every C-ABI symbol the supervisor (or other consumer) needs, with a smoke-test that verifies their presence via `nm`/`objdump`/`readelf -Ws`. The orchestrator or `make mvp-loop` reads the symbol manifest and rejects codegen output missing any required symbol. The smoke is run as part of AE's KR-acceptance, not deferred to the consumer's integration loop.
- **Review path:** Product change; lives inside the AE plugin's per-skill DISPATCH.md template. No ADR required.
- **Owner:** AE.
- **Timing:** **Picked up loop 2026-05-11-1113** — folded into AE's DISPATCH.md `--bare` addendum work (P3 dependency surface).

### P5: Codify the "smallest-thing-that-exercises-everything" focus-loop pattern (class: **org-os**)

- **Problem it solves:** This loop's MVP-focus framing produced more architectural learning in one loop than the prior two combined: discovered the ABI Option A gap, surfaced the `--bare` auth deviation, validated template-with-fill, exercised the four-team Wave-1/Wave-2 sequence, demonstrated determinism end-to-end. The pattern is reproducible but it's currently un-named in `org-os/`. Future CEOs (human or agent) reading the rituals shouldn't have to re-derive the framing from scratch when the company drifts into plan-heavy territory.
- **Proposed change:** Add a section to `org-os/rituals/ceo-brief.md` (or a new playbook at `org-os/playbooks/focus-loop.md`) naming the pattern: "When the company has shipped two consecutive plan-only loops, or when team OKRs depend on cross-team contracts that have not yet been exercised in code, the CEO brief MAY invoke a focus-loop. A focus-loop: (a) activates only teams on the critical path of one specific closing seam; (b) explicitly pauses other teams with dated resumption commitments; (c) defers pre-committed retro items if they conflict with the focus; (d) the loop's exit criterion is a single observable, mechanical artifact (a green test, a passing verdict, a shipped contract — not 'a plan is written')." Names this loop (2026-05-16) as the canonical worked example.
- **Review path:** ADR mandatory (`org-os` change to ritual or playbook surface). Draft placeholder at [`board/decisions/2026-05-16-004-focus-loop-pattern.md`](../decisions/2026-05-16-004-focus-loop-pattern.md).
- **Owner:** board.
- **Timing:** **Deferred to loop 2026-05-11-1302.** Pairs with P3 and the existing 2026-05-23 ADR batch (ADR-003 + ADR-004 + ADR-005 + ADR-006). Five org-os edits at 2026-05-23 multiplies merge friction; P5 + P3 land cleanly at 2026-05-30 after Bundle B opens the rituals surface.

### P6: `pytz` dependency hygiene on DuckDB-with-TIMESTAMPTZ (class: **product**)

- **Problem it solves:** Sim-farm caught mid-build that DuckDB's Python driver requires `pytz` at runtime to materialize TIMESTAMPTZ rows; the verdict contract's "no extra installs" promise leans on transitive availability. As more product code consumes DuckDB-with-TIMESTAMPTZ (sim-farm Modes B/C, potentially the training pipeline's profiling stage), this re-discovers.
- **Proposed change:** Sim-farm folds `pytz` into `repos/resink-ai/resink-core/sim-farm/pyproject.toml` as an explicit runtime dep (already done mid-build). DE owns adding a one-paragraph "DuckDB conventions" entry in `teams/platform/data-engineering/` (in a new `conventions/` directory or as a section in the Kafka contract — DE's call) that names: `pytz` is required for TIMESTAMPTZ; whenever DE introduces a DuckDB-consuming surface, the entry is updated.
- **Review path:** Product change; no ADR.
- **Owner:** DE (conventions doc); sim-farm (their own pyproject is already done).
- **Timing:** **Picked up loop 2026-05-11-1113** — sized XS, no dependencies.

### P7: Resink-core stretch capacity sustainability check (class: **tenant**)

- **Problem it solves:** Resink-core owned sub-projects #1 (Runtime) and #2 (Training Pipeline) for one loop's MVP slice and shipped 14/15 KRs. The stretch held under a narrow fixture (single dim, two facts, single shard). As the fixture broadens (multi-dim, multi-fact, real KV/Kafka, real codegen patterns beyond SCD2), the stretch may not. The "sustainability" question named as a risk in the 2026-05-10 retro hasn't been answered — it's been deferred behind the MVP success signal.
- **Proposed change:** The 2026-05-23 brief (or the loop after) explicitly names a capacity-check on resink-core. Options: (a) start a Product UX team standup using `org-os/playbooks/onboard-application-team.md`, taking sub-project #5 off resink-core's mental load even though resink-core doesn't own it (it's currently unowned, and "unowned" is a load on resink-core because they get questions about it); (b) name a resink-core internal split (`runtime/` and `training-pipeline/` as separate squads inside the same team); (c) accept the stretch and revisit at next MVP-class deliverable.
- **Review path:** Tenant decision; lives inside the next-or-after-next CEO brief. No ADR required.
- **Owner:** board (brief authoring).
- **Timing:** **Deferred — not committed to a specific loop yet.** Revisit when the next MVP-class deliverable (multi-dim fixture? real Kafka path?) is in flight; the capacity question is observable then, abstract now.

## Decisions to record

New ADRs this loop:

- **ADR `2026-05-16-001` (P1)** "ABI Option A — MVP runtime-spec deviation + dlopen restoration plan." Draft placeholder at [`board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md`](../decisions/2026-05-16-001-abi-option-a-mvp-deviation.md). Status: `draft`. Owner: board. Picked up loop 2026-05-11-1113 (ratification). Class: product (ADR voluntary but recommended).
- **ADR `2026-05-16-002` (P2)** "Bundle B hard-floor at loop 2026-05-11-1113." Draft placeholder at [`board/decisions/2026-05-16-002-bundle-b-hard-floor.md`](../decisions/2026-05-16-002-bundle-b-hard-floor.md). Status: `draft`. Owner: board. Picked up loop 2026-05-11-1113 (the loop the ADR activates the floor on). Class: org-os.
- **ADR `2026-05-16-003` (P3)** "Contract-against-environment verification." Draft placeholder at [`board/decisions/2026-05-16-003-contract-environment-verification.md`](../decisions/2026-05-16-003-contract-environment-verification.md). Status: `draft`. Owner: board. Picked up loop 2026-05-11-1302. Class: org-os.
- **ADR `2026-05-16-004` (P5)** "Focus-loop pattern — codify the smallest-thing-that-exercises-everything framing." Draft placeholder at [`board/decisions/2026-05-16-004-focus-loop-pattern.md`](../decisions/2026-05-16-004-focus-loop-pattern.md). Status: `draft`. Owner: board. Picked up loop 2026-05-11-1302. Class: org-os.

(P4 needs no ADR — product-class, lives in AE's DISPATCH.md template discipline.)
(P6 needs no ADR — product-class, small.)
(P7 needs no ADR — tenant-class, lives in future brief authoring.)

## Carryover ADRs still on the books

- **ADR-003 (P1 from 2026-05-10 retro):** "Verify state-claims at ritual transitions." Draft at [`board/decisions/2026-05-10-003-verify-state-claims-at-ritual-transitions.md`](../decisions/2026-05-10-003-verify-state-claims-at-ritual-transitions.md). Was scheduled for this loop; deferred per CEO brief. **Now scheduled for loop 2026-05-11-1113** (no further deferral — pairs naturally with P2's Bundle B floor since both edit `org-os/rituals/`).
- **ADR-004 (P2 from 2026-05-10 retro):** "Contract (or unified hand-off-document) artifact type." Draft at [`board/decisions/2026-05-10-004-contract-artifact-type.md`](../decisions/2026-05-10-004-contract-artifact-type.md). Triple-confirmed as needed this loop (sim-farm, DE, AE all filed contracts as `type: rfc` under the workaround). **Still scheduled for 2026-05-23** batch.
- **ADR-005 (P1 from 2026-05-09 retro):** "Carryover load tally added to CEO brief authoring step." Draft. **Still scheduled for 2026-05-23.**
- **ADR-006 (P2 from 2026-05-09 retro):** "Out-of-retro routing path for org-os change proposals." Draft. **Still scheduled for 2026-05-23.**

The 2026-05-23 batch is now: ADR-003 + ADR-004 + ADR-005 + ADR-006 + this loop's ADR-002 (Bundle B floor) = 5 org-os ADRs. ADR-001 (Option A) is product-class but ratifies the same loop. This is a heavy batch; the 2026-05-30 batch is ADR-003 sister (P3, contract-environment) + ADR-004 sister (P5, focus-loop). Spreading across two loops is the explicit decision.

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: **passed.** AE ran `grep -rEi "resink|nanofab|acme\.ai" org-os/` at the end of the loop; the only match is the pre-existing `acme.ai` placeholder in `org-os/conventions.md`. No `org-os/` files were edited this loop (Bundle B + ADR-003 deferred per CEO brief). No new ADRs from this retro touch `org-os/` content; all four are placeholder files in `board/decisions/`. Pass.
