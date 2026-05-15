---
layout: default
title: Retro — 2026-05-15-0001
date: 2026-05-15
status: active
type: retro
loop: 2026-05-15-0001
owner: board
grand_parent: Loops
parent: Loop 2026-05-15-0001
nav_order: 3
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-15
  status: active
  loop: 2026-05-15-0001
  links: parent: board/exec-summaries/2026-05-15-0001.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-15-0001

## What worked

- **The contract held cleanly under implementation — zero revisions.** The strongest possible vindication of the contract-first split discipline (loop 2026-05-14-0857 ratified the NodeCtx C-ABI contract; loop 2026-05-15-0001 implemented across it with zero revisions). The `#[repr(C)] NanofabNodeCtxVTable` field order frozen by AE's co-author constraint was mirrored verbatim across the supervisor + both cdylibs + the template. JSON serialization shapes (Key JSON, Row JSON, append_current payload-only) all worked as specified. One serialization observation surfaced (event JSON uses tagged FieldValue form; Row JSON uses plain form — they're different shapes for different directions across the FFI) — documented but not a revision. **Reproducibility:** when a cross-team contract is ratified pre-implementation AND lock-step constraints (`#[repr(C)]` field-order frozen) are baked into the contract by a co-author, implementation Phase risk approaches zero. The org has now closed one full contract-first-then-implement cycle; the pattern is validated.

- **The bridge is proven on real FFI, not mocked.** `bridge_v_table_round_trips_dim_user_insert` + `bridge_v_table_close_and_append_on_update` load the real `nanofab-plugin-dim-user.dylib`, drive `RawEvent`s through `NodeRunner::process_event`, and assert the supervisor's `ShardKv` is mutated by the cdylib's `CAbiCtxShim` calling back through the v-table. Output parquets + trace records are populated by FFI callbacks. The qualitative bar this clears — "the contract is exercised end-to-end through real ABI machinery, not through mocked PluginNode shims" — is the load-bearing acceptance signal for Phase 1b. **Reproducibility:** when a contract's load-bearing surface is unsafe FFI, the Phase-acceptance gate should be real-FFI against a real artifact, not mocked. The cost of a real-FFI test is small; the value is the verified ABI conformance the artifact itself proves.

- **The in-loop re-shape worked — again.** Phase 1b → 1b + 1c is the second in-loop re-shape of this arc (Phase 1 → 1a + 1b loop 2026-05-14-0857; Phase 1b → 1b + 1c this loop). Pattern: probe surfaces brief-mis-sizing; brief is updated in-place before code lands; ADR Status + spec §9 + exec summary all narrate the re-shape end-to-end; the new sub-phase is named, sized, and scheduled before the loop closes. **Reproducibility:** brief-time probes aren't one-shot — they continue into build-time. Every concrete coupling that surfaces during implementation gets the same treatment: in-place brief update, ADR Status narration, sequencing-table refresh. The narrative gets written as it's discovered, not back-filled.

- **AE's lock-step `#[repr(C)]` constraint paid off across multiple implementation surfaces.** A single field-order rule synchronized: the supervisor's authoritative `node_runner.rs::NanofabNodeCtxVTable`, the template's inline declaration in `lib.rs.tmpl`, and TWO hand-updated cdylib variants (`nanofab-plugin-dim-user` + `-v2`). Zero drift; zero coordination cost. **Reproducibility:** for `#[repr(C)]` types declared in multiple places, the contract should freeze field order explicitly and name the lock-step protocol — same discipline as the four `NANOFAB_NODE_*` status codes that have held for five loops.

- **Tenth consecutive loop, zero process churn.** A real in-loop re-shape — the kind of mid-build finding that historically would have triggered a process retro — absorbed without a single `org-os/` edit. The brief-time-probe + in-loop-update discipline IS the process; it's already memory-fied (`feedback-spec-first-architectural-arc.md`). The retro records this as a hard data point: the pattern works at brief time AND build time.

## What didn't

- **The brief-time-probe checklist had a gap.** The probe checked the C-ABI source surface (`lib.rs.tmpl`, `plugin_loader.rs`, the `nanofab-node-abi` trait crate) and correctly sized the contract-implementation work at M+M. But the probe missed the build-system coupling: `crates/nanofab-supervisor/src/supervisor.rs:189-211` unconditionally calls `nodes::user::run` + `nodes::account::run` (not gated by `dlopen-plugins`); `crates/nanofab-supervisor/Cargo.toml` path-deps `nanofab_node_dim_user_scd2` + `nanofab_node_dim_account_scd2` by name; only `nanofab-plugin-dim-user` exists as a cdylib in-tree. Retiring `nodes.rs` + un-ignoring `supervisor_runs_three_dims` + flipping `make mvp-loop` byte-stable via the C-ABI together require Makefile + Cargo.toml + orchestrator + new-cdylib changes the M+M sizing missed. The build-time probe caught the miss; the in-loop re-shape absorbed it; but a complete brief-time probe would have authored the brief as Phase 1b + Phase 1c from the start, sized M+M + M+M. **Action: see P1.**

- **Phase 1b's scope re-shape narrative now lives in three places (brief, ADR Status, spec §9).** They are consistent — but each loop's re-shape adds another layer to the arc's record. Phase 1 has now been re-shaped twice: 1 → 1a + 1b (loop 2026-05-14-0857), then 1b → 1b + 1c (this loop). The ADR Status section grew substantially; the spec §9 table now has four Phase 1 rows. **Mild:** the discipline of narrating in-place keeps the record honest, but readability of the arc's history degrades each re-shape. Worth flagging for the `loop+3` org-os-process loop: should arcs with multiple in-loop re-shapes graduate to a separate "history" section, leaving Status as the rolling head? Action: queue for `loop+3`'s codification work; not urgent this loop.

- **The orchestrator's strict slot-fill bit me when I added a `{{cabi_ctx_shim_dispatch}}` placeholder.** The orchestrator's slot-fill engine treats any unfilled `{{...}}` as a hard error ("FATAL [dim_user]: slot-fill error: unfilled slot in lib.rs: cabi_ctx_shim_dispatch"). I added the placeholder as a Phase 1c marker in the template; the orchestrator failed `make mvp-loop` immediately. Fixed by removing the placeholder + adding a plain `// TODO` comment instead. **Trivial:** caught + fixed in one iteration; flagged here for the Phase 1c work — the orchestrator's slot-fill must learn to handle `{{cabi_ctx_shim_*}}` placeholders OR the convention is "no `{{...}}` markers in the template that aren't slot-filled per tenant." Action: P3 names the call.

- **`supervisor_runs_three_dims` is still `#[ignore]`'d at the end of Phase 1b.** The original Phase 1b plan had it as the acceptance gate; the re-shape makes it Phase 1c's gate. The end-state is unchanged (the test runs in Phase 1c), but a casual reader of the codebase would see a one-loop-old `#[ignore]` reason claiming "Phase 1b" — except it now claims "Phase 1c" (updated). **Trivial:** addressed in this loop's update; flagged here so the Phase 1c retro confirms it un-ignores cleanly.

## Evolution proposals

### P1: Extend the brief-time-probe checklist to cover build-system coupling (class: **org-os**)

- **Problem it solves:** "What didn't" #1. The probe checked the C-ABI source surface but missed `supervisor.rs:189-211` + `Cargo.toml` path-deps + the Makefile's static-plugins default + the missing cdylibs for dim_account + dim_widget. The brief's M+M sizing was structurally incomplete; the in-loop re-shape absorbed it, but a complete brief-time probe would have authored Phase 1c upfront.
- **Proposed change:** When the next brief authors an arc phase that involves a build-system flip (Cargo feature defaults, Cargo dependency-set changes, Makefile target changes, orchestrator pipeline changes, new artifact production), the brief-time probe must additionally check: (a) which feature/flag combination the live test gate (`make mvp-loop` / `make check`) actually exercises today; (b) what artifacts the live path depends on (compile-time, runtime); (c) what infrastructure (orchestrator, Makefile, CI) emits those artifacts. Documented as a checklist extension in `org-os/conventions.md`. Codified during the `loop+3` org-os-process loop (P2 of loop 2026-05-14-0857 retro hard-scheduled it).
- **Recommendation:** **Bundle into the `loop+3` org-os-process loop's spec-first codification work.** Not its own loop; not even its own ADR — it's a checklist extension that pairs with the spec-first codification (carried since loop 2026-05-13-1944) + the WIP-rule codification (candidate from loop 2026-05-14-0857 retro P2).
- **Review path:** Codified via the `loop+3` org-os-process loop's `org-os/conventions.md` edit (which is itself an ADR per the evolution-path discipline).
- **Owner:** board (drafted at `loop+3`).
- **Timing:** **`loop+3`** (the org-os-process loop, hard-scheduled).

### P2: Phase 1c — the static-plugins retirement (class: **tenant**, cross-team)

- **Problem it solves:** Phase 1c of ADR-2026-05-14-001 (the carve-out from this loop's re-shape). The contract-implementation work shipped; Phase 1c retires the legacy path + makes `make mvp-loop` exercise the bridge end-to-end. After Phase 1c the supervisor is genuinely a single general DAG executor — the ADR Decision §1-§3 commitments are reified in `make mvp-loop`'s byte-stable verdict.
- **Proposed change:** Cross-team loop, AE + resink-core. **resink-core:** build cdylibs for dim_account + dim_widget (either hand-authored mirrors of `nanofab-plugin-dim-user` OR — preferably — driven by the orchestrator's cdylib emission for AE-slot-filled tenant crates); extend the orchestrator's `manifest.yaml` emission to include cdylib output paths per node-spec; flip `make mvp-loop` default to `dlopen-plugins` (build the cdylibs + remove the `--features static-plugins` default); remove the `Cargo.toml` path-deps on `nanofab_node_dim_user_scd2` + `nanofab_node_dim_account_scd2`; rewire `Supervisor::run` to iterate `manifest.node_specs` through `NodeRunner` (load each node-spec's cdylib via `PluginNode::load`); delete `crates/nanofab-supervisor/src/nodes.rs`; remove `pub(crate) mod nodes;` from `lib.rs`; un-ignore + author `supervisor_runs_three_dims`. **AE:** extend the orchestrator's codegen-scd2-node slot-fill to emit `CAbiCtxShim` impl `NodeCtx<K, R>` per slot-filled tenant (using the in-tree `nanofab-plugin-dim-user::CAbiCtxShim` as the reference shape; hand-rolled JSON (de)serialization, zero external deps). Acceptance: `make mvp-loop` byte-stable (`verdict=pass mismatches=0`) with the supervisor consuming nodes only through `PluginNode::process` + the v-table bridge. Sized M (AE) + M (resink-core).
- **Recommendation:** **Bake into the next CEO brief.** Natural continuation; the contract + bridge are fresh; the cdylib + manifest changes are well-scoped.
- **Review path:** No new ADR (executes against ADR-2026-05-14-001's Phase 1c row in spec §9). Phase 1c's retro records what shipped + Phase 2's `loop+N` re-anchoring.
- **Owner:** resink-core + AE.
- **Timing:** **`loop+1`.**

### P3: Decide the orchestrator's slot-fill placeholder convention (class: **tenant**, hygiene)

- **Problem it solves:** "What didn't" #3. The orchestrator's strict slot-fill treats unfilled `{{...}}` as fatal. Phase 1c will add new placeholders (`{{cabi_ctx_shim_decl}}`, `{{cabi_ctx_shim_dispatch}}`, `{{serialize_key_body}}`, `{{serialize_payload_body}}`, etc.); each must be slot-filled per tenant OR the template needs a different marker convention.
- **Proposed change:** Two options for AE's Phase 1c orchestrator extension work: (a) the orchestrator's slot-fill registers Phase-1c placeholders + emits per-tenant content for each (the "every `{{...}}` is a slot-fill" convention; consistent with today's behavior); OR (b) the template uses a different marker for "comment/anchor that doesn't get slot-filled" (e.g. `// TODO-CABI-SHIM` plain comment, not `{{...}}`) to avoid the strict-fill bite. **Recommendation: (a).** Consistent with the existing template invariant ("every `{{...}}` is slot-filled") + makes Phase 1c's slot-fill registration explicit (a slot-fill placeholder IS a slot-fill registration). Option (b) is a special case that adds template-reader cognitive load.
- **Recommendation:** **Decide (a) as part of Phase 1c's brief.** Trivial; AE's Phase 1c work registers the new placeholders explicitly.
- **Review path:** No ADR; an implementation decision within Phase 1c.
- **Owner:** AE (decision in Phase 1c brief; implementation in Phase 1c build).
- **Timing:** **`loop+1`** (Phase 1c).

### P4: Standing CEO question for `loop+1` — Phase 1c vs streaming arc swap (class: **tenant**, sequencing)

- **Problem it solves:** Standing WIP=1 enforcement. General-tables Phase 1c is the natural continuation; streaming (ADR-2026-05-13-001 steps 3-4) remains deferred carryover. The CEO has the option to swap slots in any future brief; this proposal flags that the next brief should explicitly re-confirm (or swap) the slot allocation, not silently assume continuation.
- **Proposed change:** The `loop+1` CEO brief opens with an explicit "Active arc: general-tables Phase 1c (confirmed); streaming carries" sentence — same one-line ceremony as the WIP=1 ratification this loop. Standing rule: every brief states the active arc + carryover state explicitly. Prevents accidental drift in arc sequencing.
- **Recommendation:** **Adopt as a brief-authoring discipline.** Codify in `org-os/conventions.md` during the `loop+3` org-os-process loop (bundle with P1 + the WIP-rule codification candidate).
- **Review path:** Bundled with P1 (the org-os-process loop's spec-first codification work).
- **Owner:** board.
- **Timing:** Active immediately (next brief should state active arc explicitly); codified at **`loop+3`**.

## Decisions to record

**New ADR placeholders this loop: 0.** ADR-2026-05-14-001 gained a substantive Status section recording Phase 1b shipped + Phase 1c carved out (the second in-loop re-shape of this arc). One contract artifact (`2026-05-14-nodectx-cabi-callback.md`) referenced but not edited — the contract held cleanly.

**This loop ratified two standing CEO decisions** (from loop 2026-05-14-0857 retro P2):
- **(a) WIP=1.** At most one architectural arc in active execution at a time. General-tables holds the slot.
- **(b) Org-os-process loop hard-scheduled at `loop+3`** (post-reshape; was `loop+2`). The hard-schedule intent (the slot is fixed + reachable) preserved; only the calendar shifts.

**This loop surfaced one brief-authoring discipline** for future codification: **build-time-probe checklist extension** (P1). Adopted as a brief-authoring discipline now; codified in `org-os/conventions.md` during the `loop+3` org-os-process loop.

## Carryover ADRs / open work still on the books

- **ADR-2026-05-14-001 Phase 1c (P2 this retro):** `loop+1`. Cross-team AE + resink-core; cdylibs for dim_account + dim_widget; orchestrator cdylib emission; flip `make mvp-loop` to dlopen; remove Cargo.toml path-deps; delete `nodes.rs`; un-ignore + author `supervisor_runs_three_dims`. Sized M+M.
- **ADR-2026-05-14-001 Phase 2:** `loop+2`. Cross-team resink-core + DE + AE + sim-farm; `RawFieldValue` widening to `Float64` + `Timestamp`; composite-PK end-to-end. Sized M-L.
- **Org-os-process loop (loop 2026-05-14-0857 retro P2, hard-scheduled):** **`loop+3`** (post-reshape). Bundles: arc-report-type ADR (carried since 2026-05-12-1254) + link-existence smoke (since 2026-05-13-0859) + spec-first codification (since 2026-05-13-1944) + WIP-rule codification (candidate from loop 2026-05-14-0857 retro P2) + build-time-probe checklist extension (P1 this retro) + active-arc-explicit brief-authoring discipline (P4 this retro). One dedicated, no-code, no-arc-work loop.
- **ADR-2026-05-14-001 Phase 3:** `loop+4`. Cross-team resink-core + DE + training pipeline. Sized L.
- **ADR-2026-05-13-001 streaming steps 3-4:** deferred carryover under WIP=1; re-activates after general-tables Phase 3 closes (or earlier with explicit CEO swap, per P4's discipline).
- **Local `master` divergence (loop 2026-05-14-0857 retro P3):** user-owned; user has chosen "leave `035b33e` alone" both loops it surfaced; workaround (branch from `origin/master`) is the established pattern.
- **Visual-docs brainstorm (parked):** design-stage, Approach 1 selected, nothing on disk. Subject to WIP=1; reactivates when no architectural arc active.
- **resink-marketplace `rit-executive-loop` SKILL.md edit (uncommitted):** the loop-close-conventions addition from an earlier session turn; still uncommitted in the marketplace working tree, pending user review.
- **Showcase refresh cadence:** demand-driven (loop 2026-05-13-1303 retro P1).
- **Fresh-clone verification of walkthrough recipe:** opportunistic (loop 2026-05-13-1303 retro P2).
- **Cargo CI cross-repo access** (private-compatible options documented; user-only decision).
- **Branch protection on resink-core master** (gated on GitHub Pro upgrade; user-only).
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing carry from 2026-05-23.
- **`<verified-against-rust-impl: pending>` tag in DE Kafka contract §2.1** — owner = DE.

## Tenant-isolation dry-run

Held trivially this loop. Zero `org-os/` edits. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `board/{okrs,decisions,exec-summaries,retros}/`, `docs/superpowers/specs/`, `teams/application/resink-core/`, `teams/platform/agent-engineering/`, `repos/resink-ai/resink-core/crates/`, and `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/`. None under `org-os/`.
{% endraw %}
