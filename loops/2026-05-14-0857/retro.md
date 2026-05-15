---
layout: default
title: Retro — 2026-05-14-0857
date: 2026-05-14
status: active
type: retro
loop: 2026-05-14-0857
owner: board
grand_parent: Loops
parent: Loop 2026-05-14-0857
nav_order: 3
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-14
  status: active
  loop: 2026-05-14-0857
  links: parent: board/exec-summaries/2026-05-14-0857.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-14-0857

## What worked

- **Brief-time probing caught a mis-sized phase before team planning began.** ADR-2026-05-14-001 sized Phase 1 "resink-core; M" *from the design*. The loop+1 brief was authored only after a probe of the actual C-ABI — which found `nanofab_node_process` returns only a status code and `ctx_ptr` is unwired in the codegen template, making Phase 1 cross-team. The loop re-shaped to a contract-first split *at brief-authoring time*. The spec's §9 had pre-flagged this exact site as the likeliest first-contact revision — so the probe wasn't luck, it was following the spec's own checklist. **Reproducibility:** arc ADRs size phases from the design; implementing-loop briefs must re-size from a probe of the load-bearing interface. The spec's pre-flagged first-contact sites *are* the probe checklist — read them before sizing the brief.

- **Contract-first split at the sub-phase grain.** The org has applied spec-first discipline at arc-opener grain three times (ADR-2026-05-16-001, -13-001, -14-001). This loop applied the same discipline one level down: a single phase split into "ratify the cross-team contract (both teams co-author)" + "implement across it." The risky part — agreeing an unsafe-FFI callback ABI between AE and resink-core — is now one ratified artifact, co-authored before either side wrote FFI code. **Reproducibility:** spec/contract-first isn't only for arc openers. Any phase that needs two teams to agree an interface should split: ratify first, implement second.

- **The byte-stability-critical surface landed first, unit-tested.** The partition-key byte encoding is the one place Phase 1 could silently break `make mvp-loop`. resink-core landed `CompositeKey::encode_bytes` in Phase 1a — single-column string key → bare UTF-8 bytes, byte-identical to the current partitioner — with a unit test, even though it can't be exercised end-to-end until Phase 1b wires the runner. **Reproducibility:** in a multi-phase migration, land the byte-stability-critical encoding in the earliest phase that can hold it, unit-tested. A locked-and-tested design is far cheaper than a Phase-1b debugging session.

- **Additive-first phasing made the regression gate trivial.** Phase 1a is purely additive: `schema.rs` + `node_runner.rs` are new modules that compile but aren't wired into `Supervisor::run`. The live path is untouched, so `make mvp-loop` byte-stability didn't *pass* — it *couldn't fail*. The retirement of `nodes.rs` is its own scoped Phase 1b step. **Reproducibility:** structure a migration's early phase as "new path alongside, wired to nothing." The regression gate becomes free; the cutover is a separate, deliberate step.

- **The contract needed no C-ABI signature change.** AE's co-author review found `ctx_ptr` has been a parameter of `nanofab_node_process` since the dlopen scaffold (ADR-2026-05-16-001) — Phase 1b only stops *ignoring* it. The callback v-table behind it is an additive change. **Reproducibility:** when an FFI boundary already carries an opaque context pointer, a callback v-table behind it costs nothing at the signature level — earlier scaffolding that "left room" pays off.

- **Ninth consecutive loop, zero process churn** — and this one absorbed a cross-team code loop *with a mid-flight re-scope* without a single `org-os/` edit.

## What didn't

- **The sequencing is crowded — four open threads.** As of loop-close the org carries: (1) general-tables arc, Phases 1b/2/3 open; (2) streaming arc, ADR-2026-05-13-001 steps 3-4 deferred; (3) the org-os-process backlog (arc-report-type ADR + link-existence smoke + spec-first codification), deferred through four+ loops; (4) the visual-docs brainstorm, parked at design stage. Every one is legitimately motivated; none is abandoned; but "deferred indefinitely" and "abandoned" converge if the list only grows. **Mild now, structural if it continues.** **Action:** see P2 — the CEO needs a WIP-limit decision, not just another deferral.

- **The org-os-process backlog has been deferred four+ times for the same circular reason.** It keeps being slotted "after the next phase" — but there is always a next phase. The deferral reason ("don't bundle org-os ADRs into a code/arc loop") is sound, but the *consequence* — the backlog never runs — is now the actual problem, not the bundling. **Action:** P2 hard-schedules it.

- **Recurring git-divergence friction — second consecutive loop.** Local `master` carries an unpushed `035b33e`; for the second loop running, work had to start by branching from `origin/master` instead of local `master`, and this loop the friction surfaced *mid-build* (an Edit failed because the loop-0742 ADR wasn't in the local working tree). It was recoverable both times, but it's a papercut that now costs real attention. **Action:** see P3.

- **Phase 1a's central deliverable — the contract — is unverified end-to-end.** AE confirmed implementability against `lib.rs.tmpl`'s shape, but no code crossed the C-ABI this loop. The contract's first real test is Phase 1b; it may need a revision once both sides implement against it. The contract's `Verified-against-environment` section is honest about this. **Mild:** this is the inherent nature of contract-first — the contract is a hypothesis until implemented. The honest `Verified-against-environment` note is the right mitigation. No action; flagged for Phase 1b's retro.

- **`write_output` is pure-Rust-but-unexercised.** It landed this loop (it's FFI-free, so it fit Phase 1a) but first runs in Phase 1b. If its schema-driven column layout doesn't reproduce the current per-table writers' byte layout, Phase 1b's byte-stability gate catches it — a Phase-1b debugging surface, not a Phase-1a one. **Trivial:** flagged in the resink-core exec summary; Phase 1b's brief should name it as a watch item.

## Evolution proposals

### P1: General-tables Phase 1b — wire the NodeCtx bridge (class: **tenant**, cross-team)

- **Problem it solves:** Phase 1b of ADR-2026-05-14-001. The contract is ratified; Phase 1b implements across it and completes the de-hardcoding.
- **Proposed change:** Cross-team loop, AE + resink-core. **AE:** extend `lib.rs.tmpl`'s `nanofab_node_process` to interpret `ctx_ptr` as `*mut NanofabNodeCtxVTable`, process against a `CAbiCtxShim` (null-`ctx_ptr` falls back to `DefaultProcessCtx` for the template's smoke test). **resink-core:** implement `NodeCtxBridge for &mut ShardKv`, wire `NodeRunner::process_event` + the generic runner into `Supervisor::run`, retire `nodes.rs`'s `mod user`/`mod account` + `supervisor.rs`'s `match node_spec.table`, un-ignore `supervisor_runs_three_dims`. Acceptance: the third synthetic dim runs end-to-end + the byte-stability gate (generic runner output == frozen golden of the current hand-written output) passes. Sized M (AE) + M (resink-core).
- **Recommendation:** **Bake into the next CEO brief.** It is the natural continuation; the contract is fresh.
- **Review path:** No new ADR (executes against ADR-2026-05-14-001 + the contract). Phase 1b's retro records what shipped + any contract revision.
- **Owner:** resink-core + AE.
- **Timing:** **`loop+1`.**

### P2: WIP-limit decision + hard-schedule the org-os-process loop (class: **org-os**)

- **Problem it solves:** "What didn't" #1 and #2. Four open threads; the org-os-process backlog has been deferred four+ times for a circular reason. This is no longer "schedule it eventually" — it is "decide the WIP limit, or the backlog is de facto abandoned."
- **Proposed change:** The CEO makes two explicit decisions in an upcoming brief: (a) **a WIP limit** — e.g. "at most one architectural arc in active execution at a time; the second waits" — recorded as a standing sequencing rule; (b) **a hard calendar slot for the org-os-process loop** — not "after the next phase" (which never arrives) but a specific loop: recommended **`loop+2`** (immediately after Phase 1b), theme = org-os-process backlog, no code, no arc work. That loop drafts+ratifies the arc-report-type ADR, ships the link-existence smoke, and adds the spec-first codification to `org-os/conventions.md`.
- **Recommendation:** **Decide (a) in the next brief; hard-schedule (b) for `loop+2`.** The recurring deferral is itself the evidence that a soft "recommended slot" does not work.
- **Review path:** (a) is a standing sequencing rule — brief-level, but worth an ADR if it edits `org-os/` process docs. (b) is a scheduling commitment.
- **Owner:** board (CEO).
- **Timing:** **Next brief (decision); `loop+2` (the scheduled loop).**

### P3: Reconcile the local `master` divergence (class: **tenant**, hygiene)

- **Problem it solves:** "What didn't" #3. Local `master` carries the unpushed `035b33e` ("publish-to-gitbook: emit just-the-docs tree-nav frontmatter"); for two consecutive loops this forced a "branch from `origin/master`, not local `master`" workaround, and this loop it surfaced mid-build as a failed Edit. The workaround is reliable but it costs attention every loop and is a foot-gun for anyone less careful.
- **Proposed change:** One of: (a) the user reconciles — `git rebase origin/master` on local `master` then push `035b33e` (it is real, coherent work; its code is already in active use by every publish); or (b) if `035b33e` is intentionally held back, save the "branch from `origin/master`" workaround as a memory entry so it is not re-discovered each loop. (a) is strongly preferred — it removes the papercut entirely rather than institutionalizing it.
- **Recommendation:** **Surface to the user as a decision.** This is user-owned state (their unpushed commit); the loop machinery should not unilaterally rebase/push it. Flag it at loop-close.
- **Review path:** User decision; no ADR.
- **Owner:** user (with board flagging it).
- **Timing:** **Flagged at this loop's close; resolve before `loop+1` if possible.**

## Decisions to record

**New ADR placeholders this loop: 0.** ADR-2026-05-14-001 gained a Status section (Phase 1 re-shape + Phase 1a closure). One new **contract** artifact — `2026-05-14-nodectx-cabi-callback.md` — which is `type: contract`, not an ADR.

**P2's WIP-limit rule + the org-os-process loop are the candidate org-os work for `loop+2`** — but per the no-bundling discipline they are NOT drafted in this retro (whose loop was a code loop). P2 hard-schedules the loop that drafts them.

## Carryover ADRs / open work still on the books

- **ADR-2026-05-14-001 Phase 1b (P1 this retro):** `loop+1`. Cross-team AE + resink-core; wire the bridge, retire `nodes.rs`.
- **ADR-2026-05-14-001 Phases 2-3:** `loop+3` onward (shifted by one for the 1a/1b split).
- **ADR-2026-05-13-001 streaming steps 3-4:** deferred carryover; subject to P2's WIP-limit decision.
- **Org-os-process backlog (P2 this retro):** arc-report-type ADR (carried since 2026-05-12-1254) + link-existence smoke (since 2026-05-13-0859) + spec-first codification (since 2026-05-13-1944). P2 hard-schedules a dedicated loop at `loop+2`.
- **Local `master` divergence (P3 this retro):** the unpushed `035b33e`; user-owned; flag at loop-close.
- **Visual-docs brainstorm (parked):** design-stage, Approach 1 selected, nothing on disk. Resumable; subject to P2's WIP limit.
- **resink-marketplace `rit-executive-loop` SKILL.md edit (uncommitted):** the loop-close-conventions addition from an earlier session turn; still uncommitted in the marketplace working tree, pending user review.
- **Showcase refresh cadence (2026-05-13-1303 retro P1):** demand-driven.
- **Fresh-clone verification of the walkthrough recipe (2026-05-13-1303 retro P2):** opportunistic.
- **Cargo CI cross-repo access** (private-compatible options documented; user-only decision).
- **Branch protection on resink-core master** (gated on GitHub Pro upgrade; user-only).
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing carry from 2026-05-23.
- **`<verified-against-rust-impl: pending>` tag in DE Kafka contract §2.1** — owner = DE.

## Tenant-isolation dry-run

Held trivially this loop. Zero `org-os/` edits. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `board/{okrs,decisions,exec-summaries,retros}/`, `docs/superpowers/specs/`, `teams/application/resink-core/`, `teams/platform/agent-engineering/`, and `repos/resink-ai/resink-core/crates/nanofab-supervisor/`. None under `org-os/`; the invariant has no edits to gate.
{% endraw %}
