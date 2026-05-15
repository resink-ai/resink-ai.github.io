---
layout: default
title: Exec Summary — 2026-05-15-0001
date: 2026-05-15
status: active
type: exec-summary
loop: 2026-05-15-0001
owner: board
grand_parent: Loops
parent: Loop 2026-05-15-0001
nav_order: 2
---

<!-- original-frontmatter:
  type: exec-summary
  owner: board
  date: 2026-05-15
  status: active
  loop: 2026-05-15-0001
  links: parent: board/okrs/2026-05-15-0001-ceo-brief.md
-->
{% raw %}

# Resink.ai Loop 2026-05-15-0001 — Company Exec Summary

**Headline.** First loop on 2026-05-15. General-tables **Phase 1b** shipped (re-shaped) — the NodeCtx C-ABI callback contract wired across both sides, bridge proven end-to-end on **real FFI**. AE wired `lib.rs.tmpl` (`nanofab_node_process` accepts `ctx_ptr`; `NanofabNodeCtxVTable` declared inline, field order verbatim from the supervisor). resink-core implemented `NodeCtxBridge for ShardKv` + `NodeRunner::process_event` (build v-table → call plugin → drain mutations to trace), extended `PluginNode::process` to accept `ctx_ptr`, hand-updated the in-tree `nanofab-plugin-dim-user` + `-v2` cdylibs with concrete `CAbiCtxShim` implementations, and added two new dlopen-integration tests (`bridge_v_table_round_trips_dim_user_insert` + `bridge_v_table_close_and_append_on_update`) that drive `RawEvent`s through `NodeRunner` against the real cdylib and assert the supervisor's `ShardKv` is mutated through the FFI callbacks. **Contract held cleanly — zero revisions.** **An in-loop re-shape was narrated: Phase 1b → 1b + 1c** (a build-time probe found the brief mis-sized; static-plugins retirement requires Makefile + Cargo.toml + orchestrator changes the M+M sizing missed). `make mvp-loop` byte-stable trivially (static path untouched); 27 unit + 5 dlopen integration + 3 hot-swap + 3 streaming + 3 general-tables (1 ignored as Phase 1c gate) — ALL GREEN. Two teams active. Two CEO decisions ratified: WIP=1 + org-os-process loop at `loop+3` (post-reshape, was `loop+2`). Tenant-isolation invariant CLEAN.

## Per-objective rollup

### O1: Wire the NodeCtx C-ABI bridge + retire the hardcoded supervisor path (resink-core) — ✅ PASS (re-shaped)

8/9 KRs cleared; 2 KRs (1.4 nodes.rs deletion, 1.5 supervisor_runs_three_dims) deferred to Phase 1c by an in-loop re-shape. **The contract-implementation work — the load-bearing piece — is fully shipped.** See `teams/application/resink-core/exec-summaries/2026-05-15-0001.md` for per-KR detail. Highlights:

- **The bridge is proven on real FFI, not mocked.** Two new tests load the real `nanofab-plugin-dim-user.dylib`, drive `RawEvent`s through `NodeRunner::process_event`, and assert the supervisor's `ShardKv` is mutated by the cdylib's `CAbiCtxShim` calling back through the v-table. Output parquets + trace records are populated by FFI callbacks. The strongest possible acceptance signal — the contract isn't just declared, it's exercised.
- **In-loop re-shape narrated.** Phase 1b → 1b (this loop, contract implementation) + 1c (`loop+1`, static-plugins retirement). The brief was updated in-place before code landed; ADR-2026-05-14-001 Status records the re-shape; spec §9 adds the Phase 1c row + bumps `loop+N` targets.
- 27 supervisor unit + 5 dlopen integration + 3 hot-swap + 3 streaming + 3 general-tables (1 ignored as Phase 1c gate) all green. `make mvp-loop` `verdict=pass mismatches=0` (trivially — static path untouched).

### O2: Wire `ctx_ptr` in `lib.rs.tmpl` per the ratified contract (AE) — ✅ PASS (re-shaped + partial)

5/8 KRs cleared; 3 KRs (full per-tenant `CAbiCtxShim` slot-fill emission, end-to-end via regenerated tenant crates, slot-filled crate test) deferred to Phase 1c. **The structural template change — declares the v-table inline + accepts ctx_ptr — is fully shipped.** See `teams/platform/agent-engineering/exec-summaries/2026-05-15-0001.md`. Highlights:

- `lib.rs.tmpl::nanofab_node_process` accepts `ctx_ptr` (no longer ignored); structurally consumed; null fallback to `DefaultProcessCtx` preserved.
- `NanofabNodeCtxVTable` declared inline `#[repr(C)]` with field order **verbatim** from `crates/nanofab-supervisor/src/node_runner.rs` — AE's co-author constraint loop 2026-05-14-0857 reified.
- Per-tenant `CAbiCtxShim` slot-fill deferred to Phase 1c; in-tree `nanofab-plugin-dim-user` cdylib (resink-core's hand-update) carries the reference shape Phase 1c emits per tenant. AE accepted Phase 1c's orchestrator slot-fill extension as a named deliverable.

## Per-team rollup

### resink-core (active, primary)

One objective; 9 KRs (8 PASS, 2 re-shaped to Phase 1c, 1 KR1.6.5 added). The contract-implementation work shipped — the v-table bridge wired and proven end-to-end on real FFI. The in-loop re-shape into Phase 1b + 1c is narrated in the brief + ADR + spec + this exec summary + the resink-core exec summary. nodes.rs + Cargo.toml path-deps + supervisor.rs::match retirement carry to Phase 1c.

### agent-engineering / AE (active, light)

One objective; 8 KRs (5 PASS, 3 re-shaped to Phase 1c). The structural AE deliverable — template declares the v-table + accepts ctx_ptr with lock-step field order — shipped. The full per-tenant `CAbiCtxShim` slot-fill emission is Phase 1c (orchestrator extension); the in-tree cdylib carries the reference shape.

### board (active, accounting)

Brief + this exec summary + retro + ADR-2026-05-14-001 Status §2026-05-15 + spec §9 update + the two CEO decisions (WIP=1, org-os-process at `loop+3`). Sized S.

### All other teams (paused, silent)

DE / sim-farm / training pipeline / devops / sre are downstream of later phases; none act this loop.

## Cross-cutting wins

- **The contract held cleanly under implementation — zero revisions.** This loop is the strongest possible vindication of the contract-first split discipline (memory: `feedback-spec-first-architectural-arc.md`). The `#[repr(C)] NanofabNodeCtxVTable` field order frozen by AE's co-author constraint was mirrored verbatim across the supervisor + both cdylibs + the template. The JSON serialization shapes (Key JSON, Row JSON, append_current payload-only) all worked as specified. **Reproducibility:** when a cross-team contract is ratified pre-implementation and lock-step constraints are baked into the contract itself, implementation Phase risk approaches zero.

- **The bridge is proven on real FFI, not mocked.** `bridge_v_table_round_trips_dim_user_insert` + `bridge_v_table_close_and_append_on_update` load the real `nanofab-plugin-dim-user.dylib`, drive events through `NodeRunner::process_event`, and assert the supervisor's `ShardKv` is mutated by the cdylib's `CAbiCtxShim` calling back through the v-table. This is the load-bearing Phase 1b acceptance signal — the alternative (mocked PluginNode) would have proven nothing about the actual FFI ABI. **Reproducibility:** Phase-acceptance gates that exercise the contract's load-bearing surface end-to-end (here: real FFI through a real cdylib) are vastly stronger than gates that exercise it in isolation.

- **In-loop re-shape worked.** The brief-time probe missed a coupling (Cargo.toml + Makefile + orchestrator), surfaced during build-time, and the brief was updated *in-place* before code landed. The Phase 1c split is documented end-to-end (brief, ADR Status, spec §9, retro). **Reproducibility:** the brief-time-probe discipline extends to build-time probes — when an in-loop probe finds the brief mis-sized, the brief gets updated in-place + the new sub-phase is named, sized, and scheduled before code lands. The narrative doesn't get back-filled; it gets written as it's discovered.

- **AE's lock-step constraint paid off twice.** Loop 2026-05-14-0857 the constraint was an artifact of contract co-authorship. Loop 2026-05-15-0001 the constraint became the protocol that made parallel cdylib + supervisor + template edits trivially safe. **Reproducibility:** when a contract specifies a `#[repr(C)]` shape, freezing field order via a co-author-imposed constraint is the artifact that makes multi-side parallel implementation safe.

- **Tenth consecutive loop, zero process churn.** This loop included a real in-loop re-shape — the kind of mid-build finding that historically would have caused a process retro — and absorbed it without a single `org-os/` edit. The brief-time-probe + in-loop-update discipline IS the process; it's already memory-fied.

## Cross-cutting blockers

None.

## Asks for the CEO

- **(from resink-core + AE, next loop — Phase 1c):** Cross-team. resink-core: build cdylibs for dim_account + dim_widget (or generate via the orchestrator + cdylib emission); extend orchestrator to emit cdylib paths into the manifest; flip `make mvp-loop` default to `dlopen-plugins`; remove `Cargo.toml` path-deps on per-tenant crates; rewire `Supervisor::run` to iterate `manifest.node_specs` through `NodeRunner`; delete `nodes.rs` + the `match node_spec.table` block; un-ignore + author `supervisor_runs_three_dims`. AE: extend the orchestrator's codegen-scd2-node slot-fill to emit `CAbiCtxShim` impl `NodeCtx<K, R>` + JSON (de)serialization per slot-filled tenant. **Acceptance signal: `make mvp-loop` byte-stable through the C-ABI (verdict=pass mismatches=0 with the supervisor consuming nodes only through `PluginNode::process` + the v-table bridge).** Sized M+M.
- **(from board, awareness):** Two open arcs (general-tables Phase 1c → 2 → 3; streaming ADR-2026-05-13-001 steps 3-4) + the org-os-process loop are sequenced per WIP-1: `loop+1` Phase 1c → `loop+2` Phase 2 → `loop+3` org-os-process → `loop+4` Phase 3 → `loop+5`+ streaming reactivates. The brief's sequencing table records this; the retro confirms no swap is requested.
- **(from board, sequencing reminder):** The org-os-process loop is hard-scheduled at **`loop+3`** (post-reshape, was `loop+2`). The hard-scheduling intent — that the slot is fixed and reachable — is preserved; only the calendar moves by one.

## Decisions ratified this loop

**No new ADRs.** ADR-2026-05-14-001 gained a substantive Status section recording Phase 1b shipped + Phase 1c carved out (the second in-loop re-shape of this arc, after Phase 1 → 1a + 1b loop 2026-05-14-0857).

## Decisions filed this loop (not new ADRs)

- **(a) WIP=1 standing rule.** At most one architectural arc in active execution at a time; the second waits. General-tables holds the slot through Phase 3 closure; streaming (ADR-2026-05-13-001 steps 3-4) stays deferred carryover.
- **(b) Org-os-process loop hard-scheduled at `loop+3`** (post-reshape, was `loop+2`). Theme: arc-report-type ADR + link-existence smoke + spec-first codification + WIP-rule codification candidate. No code, no arc work. The hard-schedule intent is preserved; only the calendar shifts.
- **Build-time-probe checklist extension.** Brief-time probes for arc phases that involve a build-system flip (feature defaults, Cargo.toml deps, Makefile targets, orchestrator pipeline) must probe the build's feature/flag wiring + the artifact infrastructure, not just the source-code surface. (This loop's miss: probed C-ABI source; missed `supervisor.rs:189-211`'s unconditional call to `nodes::*::run` + `Cargo.toml`'s per-tenant path-deps.) Adopted as a brief-authoring discipline; will be codified in `org-os/conventions.md` during the `loop+3` org-os-process loop.

## Multi-loop plan slippage absorbed

ADR-2026-05-14-001's Phase 1 split into 1a + 1b + 1c (was 1a + 1b after loop 2026-05-14-0857; now 1a + 1b + 1c after this loop's in-loop re-shape). Recorded in the ADR Status + spec §9. **This is a re-shape, not a slip:** the probe found the original sizing was wrong; the plan corrected at the earliest possible in-loop point. Phase 2 + Phase 3 + the org-os-process loop targets shift by one accordingly.

## Request-flow metrics this loop (per ADR-2026-05-09-006)

| Metric | Count | Note |
|---|---|---|
| `cross-team-request` objectives `fulfilled` | 0 | The AE↔resink-core coordination ran through the brief (O1 + O2), not the request inbox. |
| `dropped` | 0 | None. |
| `declined` | 0 | None. |
| `escalated` | 0 | None. |
| In-loop requests filed | 0 | The bridge co-implementation was brief-mandated, not request-filed. |

## Tenant-isolation invariant

Held throughout. Zero `org-os/` writes this loop. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `board/{okrs,decisions,exec-summaries,retros}/`, `docs/superpowers/specs/`, `teams/application/resink-core/`, `teams/platform/agent-engineering/`, `repos/resink-ai/resink-core/crates/`, and `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/`. None under `org-os/`.

## Notes for the retro

Four patterns worth recording:

1. **The contract held cleanly under implementation.** Vindicates the contract-first split (loop 2026-05-14-0857 ratified; loop 2026-05-15-0001 implemented zero-revision). The lock-step `#[repr(C)]` field-order constraint was the protocol; AE's co-author imposition the trigger. Worth recording as a general pattern: contract-first + lock-step `repr(C)` constraint = zero-revision multi-side implementation.

2. **The bridge is proven on real FFI.** The strongest possible acceptance signal — the v-table is exercised through a real loaded cdylib. Phase 1b's gate is qualitatively stronger than a mock-based test would have been. Worth recording: when a contract's load-bearing surface is FFI, the acceptance gate should be real-FFI, not mocked.

3. **In-loop re-shape worked, again.** Phase 1b → 1b + 1c is the second re-shape of this arc (after Phase 1 → 1a + 1b). The brief-time-probe + in-loop-update discipline is now applied at two grain levels: brief-authoring time AND build-time. The retro should record this as a generalization — probes happen continuously, not once.

4. **The brief-time-probe checklist had a gap.** The probe checked the C-ABI source surface (lib.rs.tmpl, plugin_loader.rs, the trait crate) but missed the build-system coupling (Cargo.toml path-deps, supervisor.rs unconditional calls to nodes.rs, Makefile static-plugins default). Worth recording as a checklist extension — and worth bundling into the `loop+3` org-os-process loop's spec-first codification work.
{% endraw %}
