---
layout: default
title: Retro — 2026-05-30
date: 2026-05-30
status: active
type: retro
loop: 2026-05-30
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-30
  status: active
  loop: 2026-05-30
  links: parent: board/exec-summaries/2026-05-30.md
-->
# Resink.ai CEO Retro — 2026-05-30

## What worked

- **The documentation-loop pattern named itself by the artifacts it produced.** Three named loop patterns now exist: focus (2026-05-16), consolidation (2026-05-23), documentation (2026-05-30). Each has a distinct precondition (focus: company drifting plan-heavy; consolidation: MVP stable, multiple teams carrying; documentation: company built faster than it explained), a distinct shape (focus: 4 teams + closing-seam exit criterion; consolidation: all teams + integration anchor; documentation: 2-3 active teams + 4 paused review-ack), and a distinct deliverable type. The rhythm is starting to look like a stable 3-tuple. Reproducibility: when the brief decides the loop's shape, naming the pattern explicitly ("this is a documentation loop") lets every team configure to it (paused teams know to ack rather than build; active teams know not to ask for code work from paused teams). The new `focus-and-consolidation-loops.md` playbook should probably grow a sibling for documentation loops in the loop after next, but the 2-loop track record isn't enough to ratify yet.
- **Paused-team review-ack as a peer-review mechanism worked exactly as designed.** Four teams paused this loop wrote one-paragraph acks instead of full exec summaries. The acks are substantive: DE caught a pointer-bug (`§4` → `§2/§2.1`); sim-farm/DevOps/SRE confirmed surfaces by-the-numbers (9/9 pytest cited, 15 chart files cited, runbook cross-link paths cited). The single bug found was fixable in two `Edit` calls within the same loop. Reproducibility: paused-team review-ack is a lighter-weight pattern than "we will look at this between sprints" — it converts pause into a structured, dated, single-paragraph contribution.
- **Mid-loop frontmatter migration (`rfc → role`) coordinated cleanly between AE and board.** AE built role files knowing the canonical `type: role` was board's deliverable the same loop. They filed `type: rfc` provisionally with an in-body migration note, and the parent agent caught the migration once board's enum extension landed. Two parallel writers; one file; no merge conflict. The pattern: when team A's output's correctness depends on team B's same-loop work, team A explicitly provisional-files with an in-body migration plan; the loop's last housekeeping step is the migration. Reproducibility: name "provisional-and-migrate" as a coordination pattern in the focus-and-consolidation-loops playbook (or in a sibling playbook on cross-team-parallel work).
- **The HTML capabilities report is read-once-and-understand.** 18KB, single-file, inline CSS, no JS, no external assets, well-formed HTML5, ASCII architecture diagram, citations all resolve on disk. An outside reader can open the file by double-clicking and orient on resink-core in 3-5 minutes. The file lives at a conventions-legal path (`board/reports/`, type: `report`, audience field set) because the same loop's enum extension admitted the type. Reproducibility: when the company needs an external-facing artifact, the brief names the artifact AND the conventions extension that legitimizes its path in the same brief — they ratify together.
- **Bundle C's tenant-isolation tripwire surfaced the regex-inlining anti-pattern.** Drafting `IC.md` and `README.md`, AE initially included the literal regex string `resink|nanofab|acme\.ai` in prose explaining the tenant-isolation invariant. The dry-run flagged it (the file matched its own grep). AE rewrote to reference `conventions.md § Linking` instead of inlining the strings. The invariant did exactly what it's designed to do: surface a tripwire at edit time, before commit. Reproducibility: when documenting the tenant-isolation invariant, reference `conventions.md` rather than spelling the regex out — the meta-discipline applies to descriptions of the discipline.

## What didn't

- **Workspace promotion deferred for the third consecutive loop.** Same blocker every time: the `resink-ai/resink-core` GitHub remote doesn't exist. Same resolution every time: the human founder needs to create the empty repo. **Root cause:** the brief's "ask" to the board has been routed-through-the-brief for three loops, but the brief is read by the same human who would create the remote — so the ask is, mechanically, the human asking themselves. There's no escalation step that bumps the urgency. **Pattern risk:** any blocker that requires a single specific human action external to the loop-tree machinery indefinitely accrues. See P1.
- **The `claude --skill` flag was predicted in three OKRs and continues not to exist.** Loops 2026-05-10 / 2026-05-16 / 2026-05-23 all referenced `claude --skill` as the dispatch invocation form; AE caught at build time that the local CLI doesn't expose it; the actual form is `claude --bare --plugin-dir <plugin> --print "/<plugin>:<skill> {json}"`. This loop's DISPATCH.md addendum closes the `--bare` half of the gap. But the predicted-syntax pattern itself remains: planning artifacts can reference syntax that doesn't exist if no one verifies before committing the OKR text. ADR-2026-05-16-003 (contract-environment-verification, ratified this loop) addresses the contract-side of this pattern but doesn't fully address the planning-artifact side. **Root cause:** OKRs are authored against ideal-shape and verified only at build time; verifying at planning time would catch this earlier. See P2.
- **dlopen restoration plan slipped one loop.** ADR-2026-05-16-001's multi-loop plan had AE template extension scheduled for 2026-05-30. AE's full bandwidth this loop was Bundle C; the template extension reschedules to 2026-06-06. This isn't an ADR re-litigation (the ADR is fine; the plan inside it slips by one loop), but it's the first slip of a board-ratified multi-loop plan. **Root cause:** ADR-2026-05-16-001 named dates in absolute form (2026-05-30, 2026-06-06) without accounting for cumulative carryover (Bundle C took AE's full bandwidth at 2026-05-30; that was foreseeable when the ADR was authored). Pattern risk: multi-loop plans inside ADRs assume linear team capacity; they don't account for the team being held by another commitment.

## Evolution proposals

### P1: Convert "create GitHub remote" from a brief-channel ask into a board-action ticket with a forcing function (class: **tenant**)

- **Problem it solves:** "What didn't" #1. Three loops of workspace promotion deferral, same blocker, same resolution. The brief-channel ask doesn't escalate because it routes through the same human who would resolve it.
- **Proposed change:** The 2026-06-06 brief creates a board-action ticket as a separate tracked artifact (or, simpler, the retro this loop creates one). Format: a single dated artifact at `board/actions/<date>-<slug>.md` (new artifact tree; another conventions-enum candidate — `action`) with `status: open | done`. The ticket is read first at every CEO brief authoring step (per ADR-2026-05-09-005's carryover-load discipline) until it flips to `done`. The forcing function: if not resolved by 2026-06-13 (one more loop boundary), the brief's "Out of scope this loop" section names the implication ("workspace promotion now blocking N other deliverables" — name them explicitly).
- **Review path:** Tenant decision. The new `board/actions/` tree is a tenant convention; admitting `action` to the org-os conventions enum is an org-os change with an ADR.
- **Owner:** board (human founder creates the remote; org-os change for the new artifact type if we go that route).
- **Timing:** **Picked up loop 2026-06-06** if board doesn't create the GitHub remote out-of-loop before then. If the remote IS created out-of-loop, the proposal is no-op and dropped from carryover.

### P2: Add a "Verified-against-environment" check to OKR authoring, not just contract authoring (class: **org-os**)

- **Problem it solves:** "What didn't" #2. ADR-2026-05-16-003 (ratified this loop) requires contracts to carry a Verified-against-environment section. OKRs that reference external syntax (CLI flag names, file paths, command shapes) are not subject to the same discipline. The `claude --skill` flag prediction pattern recurred in three OKRs because no one verified the syntax at planning time.
- **Proposed change:** Extend ADR-2026-05-16-003's discipline to team OKR authoring: when an OKR mentions a CLI flag, file path, or environment variable that doesn't yet exist on the team's working branch, the OKR carries an explicit "not yet verified — predicted shape" marker. The team-planning ritual's Acceptance criteria already requires every task to have an owner; extend it to require every predicted-syntax reference to have a verification marker. The build phase converts the marker into a confirmed reference or files an addendum.
- **Review path:** ADR mandatory (`org-os` change to `org-os/rituals/team-planning.md`). Draft placeholder at [`board/decisions/2026-05-30-001-verify-predicted-syntax-in-okrs.md`](../decisions/2026-05-30-001-verify-predicted-syntax-in-okrs.md).
- **Owner:** board (ADR authoring); AE (the natural reviewer since OKR authoring is the team-planning ritual AE's Bundle B finished landing).
- **Timing:** **Deferred to loop 2026-06-13.** The 2026-06-06 batch already has the sim-farm schema-aware join columns + dlopen restoration prep + workspace promotion attempt; adding one more org-os rituals edit would multiply merge friction with already-active ritual files. P2 batches naturally at 2026-06-13.

### P3: Document the documentation-loop pattern in the focus-and-consolidation-loops playbook (class: **org-os**)

- **Problem it solves:** Three named loop patterns now exist with worked examples; the playbook ratified this loop only names two. Future CEOs reading the playbook may not recognize documentation as a third invocation mode.
- **Proposed change:** Extend `org-os/playbooks/focus-and-consolidation-loops.md` (ratified this loop) to also codify documentation loops. Definition: invoked when (a) the company has shipped multiple build loops with no canonical orientation document for the primary product surface, (b) a canonical answer to "what does X do?" requires reading source, (c) external audiences would benefit from an externally-shippable artifact. Shape: 2-3 active teams + paused-team review-ack mechanism + one cross-loop closure artifact. Names this loop (2026-05-30) as the canonical worked example.
- **Review path:** ADR-class change to an `org-os/` playbook. Treat as a same-class extension of ADR-2026-05-16-004 (which already extended once this loop to add the consolidation sibling), OR file as a new ADR. **Recommend: extend ADR-2026-05-16-004 in place** rather than spawn a new ADR — the playbook is one document with three patterns; one ADR can govern the whole document.
- **Owner:** board.
- **Timing:** **Deferred to loop 2026-06-13.** Two-loop track record of documentation-pattern observation (this loop is the first; a second occurrence would strengthen the case). If no documentation loop recurs by 2026-06-20, ratify as P3 the 2026-06-13-or-after loop.

### P4: Multi-loop plans inside ADRs should name dates as "loop+N" rather than absolute calendar dates (class: **org-os**)

- **Problem it solves:** "What didn't" #3. ADR-2026-05-16-001's multi-loop plan named absolute dates (2026-05-30, 2026-06-06) that don't account for cumulative team-capacity carryover. When AE's Bundle C took 2026-05-30 in full, the template extension date slipped without a recorded mechanism.
- **Proposed change:** When an ADR records a multi-loop plan, it names dates as "loop+1", "loop+2" relative to the ADR's ratification loop, with an explicit "subject to team capacity at that loop's brief" qualifier. Absolute-date plans recur only when an external commitment (customer milestone, conference date, regulatory deadline) requires them. The org-os change is to `org-os/templates/adr.md` — add a "Multi-loop plan note" section noting the loop+N convention.
- **Review path:** ADR mandatory (`org-os` template change). Draft placeholder at [`board/decisions/2026-05-30-002-multi-loop-plan-relative-dating.md`](../decisions/2026-05-30-002-multi-loop-plan-relative-dating.md).
- **Owner:** board.
- **Timing:** **Picked up loop 2026-06-06** — small change, naturally batches with the 2026-06-06 work.

### P5: Open a CEO question on documentation as a third recurring loop type (class: **tenant**)

- **Problem it solves:** Two loops separated by 14 days produced a focus loop + a consolidation loop + a documentation loop. If the rhythm stabilizes at a 3-tuple, the cadence is "build → consolidate → document, repeat." If documentation loops only happen when documentation debt accrues, the cadence is "build → consolidate → build → consolidate → … → document (rare)." This shape question affects how teams plan their carryovers (do they hold back doc work for the next docs loop, or expect it inside every loop?).
- **Proposed change:** The 2026-06-06 brief's Context section opens this question explicitly: "Do we expect documentation loops to recur regularly, or only when documentation debt accrues?" The brief proposes a working answer (e.g., "documentation loops are demand-driven — invoked when canonical answers require source-reading; we do not pre-schedule them"). Subsequent retros reaffirm or revise.
- **Review path:** Tenant decision; lives inside next CEO brief authoring.
- **Owner:** board.
- **Timing:** **Picked up loop 2026-06-06** (brief Context section).

## Decisions to record

New ADRs this loop:

- **ADR `2026-05-30-001` (P2)** "Verify predicted syntax in OKRs at planning time, not build time." Draft placeholder at [`board/decisions/2026-05-30-001-verify-predicted-syntax-in-okrs.md`](../decisions/2026-05-30-001-verify-predicted-syntax-in-okrs.md). Status: `draft`. Owner: board. Picked up loop 2026-06-13. Class: org-os.
- **ADR `2026-05-30-002` (P4)** "Multi-loop plans inside ADRs use loop+N relative dating." Draft placeholder at [`board/decisions/2026-05-30-002-multi-loop-plan-relative-dating.md`](../decisions/2026-05-30-002-multi-loop-plan-relative-dating.md). Status: `draft`. Owner: board. Picked up loop 2026-06-06. Class: org-os.

(P1 needs no ADR — tenant-class, board-action ticketing. If the new `board/actions/` tree is admitted as a conventions extension, that would spawn an enum-extension ADR sibling — but not unless P1 escalates to needing a structural answer.)
(P3 will extend ADR-2026-05-16-004 in place if/when the documentation pattern recurs.)
(P5 needs no ADR — tenant-class, brief-context-section question.)

## Carryover ADRs still on the books

Three ADRs ratified to `active` this loop (covered in detail in the company exec summary). The 2026-05-30 batch is now empty; new ADR work begins with the 2026-06-06 + 2026-06-13 placeholders above.

Multi-loop plan adjustments (not new ADRs, just plan-side updates):

- **ADR-2026-05-16-001 (Option A / dlopen restoration plan):** AE template extension slips from 2026-05-30 to 2026-06-06; supervisor swap re-targets 2026-06-13; hot-swap correctness test 2026-06-20-or-later.

## Tenant-isolation dry-run

`org-os/playbooks/extract-org-os.md` dry-run: **passed**. Board ran dry-runs after each of the 3 ADR-mandated `org-os/` edits + the new focus-and-consolidation-loops playbook + the conventions enum extension. AE ran dry-runs after each of the 7 Bundle C tasks + a final post-loop sweep. Combined: only the pre-existing `acme.ai` placeholder reference in `org-os/conventions.md` matched. **The largest concentration of `org-os/` writes in any single loop completed with zero tenant-isolation violations.** Two self-induced tripwires (regex inlining in `IC.md` + `README.md` during AE's Bundle C) surfaced and were fixed before commit — the invariant did exactly what it's designed to do. Pass.
