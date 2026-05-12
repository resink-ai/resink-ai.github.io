---
layout: default
title: Retro
nav_order: 3
parent: "Loop 2026-05-11-1631"
grand_parent: "Loops"
render_with_liquid: false
date: 2026-06-06
status: active
type: retro
loop: 2026-05-11-1631
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-06-06
  status: active
  loop: 2026-06-06
  links: parent: board/exec-summaries/2026-05-11-1631.md
-->
# Resink.ai CEO Retro — 2026-06-06

## What worked

- **The build loop pattern returns after the focus → consolidation → documentation rhythm without drift.** Loop 2026-05-11-1631 was the first code-bearing loop after the 2026-05-30 documentation loop; the rhythm shifted cleanly back without the loop-shape question recurring (P5 from the 2026-05-30 retro answered in this brief's Context section: documentation loops are demand-driven, not pre-scheduled). Four teams active + one paused + one paused-with-light-housekeeping is a quieter shape than the consolidation loop's all-teams-active but louder than the documentation loop's three-team focus. Reproducibility: when the brief opens with "the company built and documented; now we close the named deviations" framing, every team's OKR self-selects to deviation-closure work. No re-litigation of the loop shape mid-stream.

- **Three multi-loop deviations closed simultaneously in a single loop.** Engine 0.3 schema-aware columns (sim-farm, retro P1 from 2026-05-23); `dim_account` natural-column-shape lift (resink-core, was named architectural debt at 2026-05-23 and documented at 2026-05-30); ADR-2026-05-16-001 step 1 (AE template extension, retro from 2026-05-16). Three deviations carried for 2-3 loops each; all three closed this loop with no carryover. Pattern: when the brief explicitly names "close the named deviations" as the objective shape, teams converge on closure rather than feature-adding. Reproducibility: future "closure loops" should be triggered when the carryover-load table shows ≥3 multi-loop carries against a single team or against intersecting teams.

- **ADR-2026-05-30-002 (loop+N relative dating) ratified WITH its worked example demonstrated in-loop.** The brief's Context section uses loop+1 / loop+2 notation to describe the next two steps of ADR-2026-05-16-001's plan; the ADR ratifies the convention simultaneously. The "demonstrate in the ratification loop" pattern paid off — readers consuming the brief alongside the ADR have a same-loop reference for what the convention looks like in use. Reproducibility: when an ADR's content is itself a notation convention (or a similar self-describing pattern), the same loop in which it ratifies should include the first worked example in the brief or exec summary.

- **The provisional-and-migrate pattern reused successfully.** First `board/actions/` ticket filed with `type: action` (provisional) and an in-body note about the 2026-06-13 enum-extension migration. Same pattern AE used at 2026-05-30 for `rfc → role` mid-loop. The pattern is graduating from one-off coordination move to a documented standing practice (codified in AE's status.md "Carrying into next loop" as a standing practice — board should consider explicitly playing it back into a playbook).

- **First contract honors ADR-2026-05-16-003's Verified-against-environment subsection.** Sim-farm's verdict contract update at 2026-06-06 is the first cross-team contract to receive the subsection going forward (existing contracts grandfathered). The ADR ratified at 2026-05-30; the first worked example landed one loop later. Reproducibility: ADRs that mandate a future-going edit (this one), should expect first compliance in the loop after ratification, not the ratification loop itself. The discipline is observably load-bearing in one loop.

- **Build phase scheduled five teams in parallel with one in-loop coordination handshake (sim-farm ↔ resink-core).** Engine 0.3 (sim-farm) and the schema.json files + orchestrator dispatch (resink-core) needed an agreed contract; the brief defined the schema shape in its standing answers; both teams built independently; resink-core's `make mvp-loop` ran end-to-end on first attempt. The pre-agreement-in-brief discipline saved one round of cross-team negotiation. Reproducibility: when two teams have an in-loop integration, the brief's standing-answers section MUST name the contract shape — not "to be agreed in-loop."

## What didn't

- **DevOps minikube smoke deferred for the third consecutive loop.** Same shape as workspace promotion (1st pattern): a single specific human action external to the loop-tree machinery (in this case, `brew install minikube`). The CEO standing answer this loop approved Docker Desktop install, which DevOps already had via OrbStack — but the same standing answer didn't think through the minikube binary, which is a separate install. **Root cause:** the brief's standing answer named "Docker Desktop install" without specifying "and minikube binary on PATH." DevOps reasonably read the answer literally, completed the Docker side, and discovered the minikube gap at smoke time. The 3-loop pattern means this is no longer a one-off blocker — it's a recurring class of failures where a multi-step external action gets brief-coded as a single-line approval. See P1.

- **Workspace promotion remained blocked despite the new board-action ticket.** The forcing function is dated 2026-06-13 (one loop boundary); this loop the ticket fired the day it was filed and the resink-core agent verified the remote still doesn't exist via `git ls-remote`. Now a 4th-loop carry. The forcing function will pay off OR escalate at 2026-06-13's brief authoring; this loop's outcome is a known-good escalation mechanism, not a closed blocker. Not strictly a "didn't work" — the ticket worked as designed — but the blocker persists. **Root cause:** ticket → human action → human reads ticket is still a one-human-bandwidth path. If 2026-06-13 doesn't resolve, escalation needs a different shape (e.g., the brief generates the `gh repo create` command and asks the human to run it via the existing "! prefix" mechanism). See P2 (deferred to 2026-06-13 retro).

- **Sim-farm filed a request to DE for schema-JSON spec canonicalization; DE was paused but committed to act next loop.** This is fine on surface, but it's a soft pattern: paused teams accepting in-loop work commits as carry-forward without the typical request-flow lifecycle (open → accepted → fulfilled). The provisional answer ("DE will canonicalize next loop; sim-farm-embedded spec stands meanwhile") is well-formed; what's not well-formed is whether DE's commitment lives in DE's status.md, in the request file, or in both. **Root cause:** the request-flow lifecycle from ADR-2026-05-09-006 doesn't say where a paused team's "accepted with action next loop" lives. The bookkeeping is double-entered (status.md + request file) without a canonical write-down for which is source-of-truth. Minor friction; pattern emerging. See P3.

- **AE's smoke test for `nanofab_node_process` used a workaround (null ctx_ptr + thread-local MemCtx).** This works for the template smoke but doesn't fully exercise the FFI surface the supervisor will use at loop+1. The actual supervisor will pass a real `*mut c_void` pointing at its NodeCtx implementation; the template smoke can't construct that pointer from outside the supervisor crate. AE's documented this in the test comment, but the cross-process libloading::Library::open dlopen test was deferred to resink-core's loop+1 work. **Root cause:** the template smoke and the integration smoke are different exercises, and only the integration smoke validates the full FFI roundtrip. The ADR-2026-05-16-001 plan doesn't separately name the integration smoke — it's implicit in step 2. **Pattern:** multi-loop plans should explicitly name the smoke at each step, not just the deliverable. Mild — not retro-worthy alone but observed.

## Evolution proposals

### P1: Convert "install missing toolchain on developer machine" into a same-class board-action ticket pattern (class: **tenant**)

- **Problem it solves:** "What didn't" #1. The DevOps minikube smoke deferred for the third consecutive loop on a developer-machine-toolchain blocker. Same shape as workspace promotion: single specific human action external to the loop-tree machinery. The brief's CEO standing answer named "Docker Desktop install" but didn't enumerate the full toolchain (Docker Desktop + minikube CLI). Three loops of recurring deferral on small external installs is now a pattern.
- **Proposed change:** File a sibling board-action ticket at `board/actions/2026-06-06-002-install-minikube-on-dev-machine.md` with the same shape as `2026-06-06-001`: provisional `type: action`, `status: open`, `due: 2026-06-13`, body names the exact command (`brew install minikube`), forcing function names the count of downstream blockers (DevOps KR1.2 carry, supervisor swap full-execution at loop+1 will want minikube smoke as the deployment surface). The pattern: any blocker that's "human runs single command on local machine" is now a board-action ticket, not a brief-channel ask.
- **Review path:** Tenant decision. The `board/actions/` tree is already a tenant convention (admitted provisionally; canonicalization deferred to 2026-06-13 alongside P2 from 2026-05-30 retro).
- **Owner:** board (CEO + DevOps coordinate the command; CEO files the ticket).
- **Timing:** **Picked up loop 2026-05-11-2153** — file the ticket as part of brief authoring; if the human runs the install out-of-loop before then, the ticket is no-op and dropped. **OR** file in this retro as a same-loop action (smaller surface). **Recommend: file in this retro.** The action ticket is mechanical to author and the forcing function works the same way.

### P2: Extract the loop-coordination "provisional-and-migrate" pattern into a documented playbook (class: **org-os**)

- **Problem it solves:** The provisional-and-migrate pattern has now been used three times: AE's `rfc → role` migration at 2026-05-30, the `type: action` provisional admission at 2026-06-06, and (looking forward) DE's schema-JSON spec canonicalization at 2026-06-13 (if it proceeds as agreed). The pattern is reusable and graduating from one-off to standing practice; AE's status.md already names it as such. Without a documented playbook, future agents have to discover the pattern from prior loops; with a playbook, they can apply it from cold.
- **Proposed change:** Author a new `org-os/playbooks/provisional-and-migrate.md` with `type: playbook` codifying the pattern: when team A's work depends on team B's same-loop or future-loop output (typically a conventions change, contract change, or enum extension), team A files provisional artifact + in-body migration note, team B ratifies, parent/board housekeeping migrates. Three worked examples cited from prior loops (AE rfc→role at 2026-05-30; board action provisional admission at 2026-06-06; sim-farm/DE schema-JSON pending at 2026-06-13).
- **Review path:** ADR mandatory (`org-os` change adding a new playbook). New ADR placeholder at `board/decisions/2026-06-06-001-provisional-and-migrate-playbook.md`.
- **Owner:** board (ADR + playbook authoring); AE (reviewer — they invented the pattern and use it most).
- **Timing:** **Deferred to loop 2026-05-11-2153** — batches naturally with ADR-2026-05-30-001 (verify predicted syntax in OKRs, P2 from 2026-05-30) and the `action` enum extension (P2 from this retro's classification). Three org-os ADRs in one batch at 2026-06-13 is consistent with prior batching practice (2026-05-23 had a 5-type enum extension; 2026-05-30 had three ADR ratifications).

### P3: Clarify where a paused team's in-loop work commitments live — status.md, request file, or both (class: **org-os**)

- **Problem it solves:** "What didn't" #3. DE was paused this loop, accepted an in-loop request from sim-farm, and committed to act at 2026-06-13. The commitment is recorded in DE's exec summary, DE's status.md "Carrying into next loop" section, and (will be) in the request file's `links.fulfilled_by` field once filed. No single source-of-truth. ADR-2026-05-09-006's request-flow lifecycle doesn't specify where a "paused team accepts and defers" record lives. Minor bookkeeping friction; pattern emerging.
- **Proposed change:** Extend `org-os/rituals/team-intake.md` (or `team-planning.md`) to specify: paused teams accepting in-loop requests record the acceptance ONLY in the request file's frontmatter (transition `status: open → status: accepted-deferred-to-loop-<date>`); their status.md and exec summary can REFERENCE the request file by path, but the canonical write-down is in the request file. This avoids double-entry; the request file is source-of-truth. Open question: do we need a new `status: accepted-deferred` state, or does `status: accepted` cover it with a `deferred_to_loop` field?
- **Review path:** ADR likely needed (`org-os` change to team-intake.md + possible request frontmatter extension). Draft placeholder at `board/decisions/2026-06-06-002-paused-team-request-acceptance.md`.
- **Owner:** board.
- **Timing:** **Deferred to loop 2026-05-11-2153** or later. Low-urgency; the friction is bookkeeping not blocking. Bundle with P2 if the 2026-06-13 batch is otherwise small; defer further if 2026-06-13 is already heavy.

### P4: Multi-loop plans inside ADRs should explicitly name the smoke / verification gate at each step, not just the deliverable (class: **org-os**)

- **Problem it solves:** "What didn't" #4 (mild). ADR-2026-05-16-001's plan names: step 1 = template extension + smoke; step 2 = supervisor swap; step 3 = hot-swap correctness test. The smoke at step 1 is named (single-process), the test at step 3 is named (hot-swap correctness), but step 2 (supervisor swap) doesn't name what gates "swap done." Resink-core's loop+1 work will need to define this gate (e.g., "make mvp-loop green with `dlopen-plugins` feature on against AE's built artifact"). Defining the gate in the ADR rather than discovering it loop-by-loop tightens the multi-loop plan.
- **Proposed change:** Extend `org-os/templates/adr.md` "Multi-loop plan note" subsection (just added this loop) to include a "Verification gate per step" line item: each step lists what gates closure (test, smoke, contract, or "no gate — coordination only"). This is an additive edit to the template that just ratified.
- **Review path:** ADR-class change to a same-loop-just-ratified template. **Recommend: extend ADR-2026-05-30-002 in place** rather than spawn a new ADR — same author (board), same loop +1 from ratification, mechanical addition. Sister relationship to the Verified-against-environment subsection in contracts.
- **Owner:** board.
- **Timing:** **Picked up loop 2026-05-11-2153** alongside P2 + P3 + ADR-2026-05-30-001. **OR** can be folded into a corrigendum-style edit to ADR-2026-05-30-002 if board prefers a single-ADR multi-iteration pattern. Light decision.

### P5: Track DevOps's minikube smoke deferral pattern and decide between three resolution paths at 2026-06-13 (class: **tenant**)

- **Problem it solves:** DevOps's minikube smoke is a 3rd-consecutive-loop defer (parallel to workspace promotion's 4-loop carry). Same blocker shape (single-human-action externally to loop-tree). The brief's "Asks for the CEO" already named the three options: (a) `brew install minikube` resolves in next loop; (b) same-class board-action ticket; (c) accept chart skeleton as shippable without in-loop smoke and rely on production-time k8s deploy smoke instead. P1 picks (b) as the recommended path; this P5 is a meta-question: are we accumulating board-action tickets for every external-toolchain blocker (potentially many of them; Docker, minikube, jq, gh, terraform, ...), or do we need a different mechanism?
- **Proposed change:** The 2026-06-13 brief's Context section opens this question explicitly. Working answer (proposed): board-action tickets are the right mechanism for blockers that require a single specific human action on shared infrastructure (GitHub remote, IAM role, etc.); developer-machine toolchain installs are NOT board-action tickets — they're per-team Prerequisite documentation that the team running the smoke installs as part of their own build phase. If 2026-06-13 doesn't decide, this question accrues to a retro pattern.
- **Review path:** Tenant decision; lives inside next CEO brief authoring.
- **Owner:** board.
- **Timing:** **Picked up loop 2026-05-11-2153** (brief Context section).

## Decisions to record

New ADRs this loop:

- **ADR `2026-06-06-001` (P2)** "Extract the provisional-and-migrate pattern into a documented playbook." Draft placeholder at [`board/decisions/2026-06-06-001-provisional-and-migrate-playbook.md`](../decisions/2026-06-06-001-provisional-and-migrate-playbook.md). Status: `draft`. Owner: board. Picked up loop 2026-05-11-2153. Class: org-os.
- **ADR `2026-06-06-002` (P3)** "Paused teams record in-loop request acceptance canonically in the request file, not in status.md." Draft placeholder at [`board/decisions/2026-06-06-002-paused-team-request-acceptance.md`](../decisions/2026-06-06-002-paused-team-request-acceptance.md). Status: `draft`. Owner: board. Picked up loop 2026-05-11-2153 or later. Class: org-os.

(P1 needs no ADR — tenant-class, board-action ticket filing. Filed in this retro as a same-loop action — see § "Same-loop actions" below.)

(P4 will extend ADR-2026-05-30-002 in place rather than spawn a new ADR.)

(P5 needs no ADR — tenant-class, brief-context-section question.)

## Same-loop actions (filed in this retro)

- **`board/actions/2026-06-06-002-install-minikube-on-dev-machine.md`** filed per P1. Frontmatter: `type: action` (provisional, same provisional-and-migrate as `2026-06-06-001`), `status: open`, `due: 2026-06-13`. Body: `brew install minikube` is the recovery command; forcing function names DevOps KR1.2 + supervisor swap deployment-surface readiness. Cross-link to 2026-06-13 brief's "first read step."

## Carryover ADRs still on the books

- **ADR-2026-05-30-001 (P2 from 2026-05-30 retro):** Verify predicted syntax in OKRs at planning time. Still `draft`; scheduled for body authorship + ratification at 2026-06-13. Carries from prior retro unchanged.

- **P3 from 2026-05-30 retro (documentation-loop pattern playbook extension):** Still pending two-loop track record. This loop was a build loop, not a documentation loop. No data point added; carries from prior retro unchanged. Will revisit when a second documentation loop occurs (target: post-2026-06-20).

Multi-loop plan adjustments (not new ADRs, just plan-side updates):

- **ADR-2026-05-16-001 (Option A / dlopen restoration plan):**
  - Step 1 (AE template extension): ✅ closed this loop.
  - Step 2 (resink-core supervisor swap full execution): scheduled `loop+1` (= 2026-06-13). Resink-core pre-staged feature flags this loop; supervisor module compiles cleanly with both `static-plugins` and `dlopen-plugins`; loop+1 work is integration testing the dlopen path against AE's built artifact.
  - Step 3 (hot-swap correctness test): scheduled `loop+2` (= 2026-06-20). Joint between AE + resink-core.

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: **passed**. Board ran one dry-run after the `org-os/templates/adr.md` "Multi-loop plan note" subsection edit (the only org-os/ write this loop). AE made zero org-os/ writes (work in marketplace submodule). Combined: only the pre-existing `acme.ai` placeholder reference in `org-os/conventions.md` matched. The lightest org-os edit load in any single loop since 2026-05-09 (Bundle A start); the discipline of "every org-os edit gets a dry-run" held trivially. Pass.
