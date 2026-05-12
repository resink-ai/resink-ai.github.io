---
layout: default
title: application-resink-core OKR — 2026-06-13
date: 2026-06-13
status: active
type: okr
loop: 2026-06-13
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-06-13
  status: active
  loop: 2026-06-13
  links: parent: board/okrs/2026-06-13-ceo-brief.md
-->
# Resink Core OKR — 2026-06-13

## Context

This loop executes step 2 of the ADR-2026-05-16-001 dlopen restoration plan (loop+0; previously slipped one loop from 2026-06-06). AE's template extension landed at loop-1 — `nanofab_node_process` plus the four `NANOFAB_NODE_*` status code constants are exported on the `scd2_maintainer` template, and our supervisor's `plugin_loader.rs` already mirrors the C-ABI exactly under the `dlopen-plugins` feature flag against a no-op shim. This loop is full execution: generate a plugin crate from AE's template, build a real `.so`/`.dylib`, and exercise the full FFI roundtrip end-to-end. Per the 2026-06-06 retro's mild "what didn't" #4, the integration test is the load-bearing gate that loop-1's unit-level shim could not exercise — every named risk lands here.

## Objectives

### O1: Execute ADR-2026-05-16-001 step 2 — supervisor dlopen swap full execution

source: ceo-brief

Why it matters: This is resink-core's slice of CEO O3 — the load-bearing integration step that closes ABI Option A's named MVP deviation as a "step 2 closed; hot-swap test pending loop+1" entry. Loop-1 scaffolded the conditional load shape and unit-tested the no-op shim path; this loop builds AE's template against the `dim_user` schema fixture, loads the resulting `.so`/`.dylib` via `libloading::Library::open(...)`, invokes `nanofab_node_process` on a known event, and asserts the FFI roundtrip matches the static-linking baseline byte-for-byte. Once green, `make mvp-loop` runs under both `static-plugins` (existing baseline) and `dlopen-plugins`-on against the built artifact, with verdict.json shape unchanged. Step 3 (hot-swap correctness test) reschedules to loop+1 (= 2026-06-20) per the existing multi-loop plan.

**Key results**
- KR1.1: A built plugin artifact exists in the resink-core working tree under `target/release/lib<crate>.{so,dylib}` (or `target/debug/...` if release fails first; build command documented in resink-core's docs). The plugin is generated from a crate produced by AE's `codegen-scd2-node` skill against the `dim_user` table fixture (use the existing `synthetic_tenants/closed_loop_v0/` fixture's schema as the contract source). [from brief KR3.1]
- KR1.2: A new integration test at `crates/nanofab-supervisor/tests/dlopen_integration.rs` (or equivalent) builds the plugin via `cargo build` (or references a pre-built path), loads the `.so`/`.dylib` via `libloading::Library::open(...)`, looks up the `nanofab_node_process` symbol, invokes it on a known event JSON, asserts return code `NANOFAB_NODE_OK = 0`, and verifies the output node state matches the static-linking-path output byte-for-byte. Test passes on first attempt (the loop-1 shim test pre-locked the symbol shape). [from brief KR3.2]
- KR1.3: `cargo build --no-default-features --features dlopen-plugins -p nanofab-supervisor` exits 0; `cargo test --features dlopen-plugins -p nanofab-supervisor` exits 0 (existing tests + the new integration test). The `static-plugins` baseline remains green: `cargo build --no-default-features --features static-plugins -p nanofab-supervisor` exits 0 and `cargo test --no-default-features --features static-plugins -p nanofab-supervisor --bins` continues to pass 5/5. [from brief KR3.3]
- KR1.4: `make mvp-loop` from `synthetic_tenants/closed_loop_v0/` runs GREEN under BOTH feature configurations: (a) default `static-plugins` (existing baseline), (b) `dlopen-plugins`-on against the built plugin artifact. Verdict.json shape unchanged in both runs (`overall_pass: true`, both per-dim `pass: true`, `mismatch_count: 0`, `engine_version: 0.3.0`). The `dlopen-plugins`-on invocation pattern (which env vars / paths point the supervisor at the built artifact) is documented in `repos/resink-ai/resink-core/docs/user-guide.md` § "Common commands" as a one-paragraph addition. [from brief KR3.4]
- KR1.5: ADR-2026-05-16-001 multi-loop plan slippage record updated in this loop's exec summary: step 2 → done at loop+0 (= 2026-06-13); step 3 → loop+1 (= 2026-06-20) per the existing schedule. ADR body grandfathers under absolute-date authoring; the slippage record lives in the brief/exec summary per ADR-2026-05-30-002's loop+N convention. [from brief KR3.5]
- KR1.6: `repos/resink-ai/resink-core/docs/architecture.md` § "Named deviations" updated: ABI Option A entry moves from "still in place" to "step 2 closed; hot-swap test pending loop+1"; the entry surfaces the `dlopen-plugins` feature flag invocation as the new canonical path under `--features dlopen-plugins`. The carried entries from loop-1's status (the deferred architecture.md flips for `dim_account` lift + `claude --bare` reintroduction) also land cleanly in this same edit. [from brief KR3.6]
- KR1.7: Tenant-isolation invariant holds; no `org-os/` edits this objective. All edits land under `repos/resink-ai/resink-core/` and `teams/application/resink-core/`. Post-edit verification: `grep -nrE "(resink|nanofab)" org-os/` returns no resink-core writes. [from brief KR3.7]

**Tasks**
- [ ] Generate a plugin crate from AE's `codegen-scd2-node` template against the `dim_user` schema (use `synthetic_tenants/closed_loop_v0/sim-farm-schemas/dim_user.json` as the contract source); record the generation invocation in `docs/user-guide.md` — owner: teams/application/resink-core.
- [ ] Build the `.so`/`.dylib` artifact (`cargo build --release -p <generated-crate>` or debug fallback); confirm symbols exported via `nm -D target/.../lib<crate>.{so,dylib} | grep nanofab_node_` (or platform equivalent) before declaring KR1.1 — owner: teams/application/resink-core.
- [ ] Author `crates/nanofab-supervisor/tests/dlopen_integration.rs` exercising the full FFI roundtrip against the built artifact (load → lookup → invoke → assert OK + byte-identical output) — owner: teams/application/resink-core.
- [ ] Verify `cargo build --no-default-features --features dlopen-plugins -p nanofab-supervisor` and `cargo test --features dlopen-plugins -p nanofab-supervisor` both exit 0; re-verify the `static-plugins` baseline still green — owner: teams/application/resink-core.
- [ ] Run `make mvp-loop` under both feature configurations; capture verdicts; confirm verdict.json shape unchanged (`overall_pass: true`, `engine_version: 0.3.0`, both per-dim `pass: true`, `mismatch_count: 0`) — owner: teams/application/resink-core.
- [ ] Update `docs/user-guide.md` § "Common commands" with the `dlopen-plugins`-on invocation pattern (build artifact path, env vars or CLI flags) — owner: teams/application/resink-core.
- [ ] Update `docs/architecture.md` § "Named deviations": ABI Option A entry → "step 2 closed; hot-swap test pending loop+1"; close the carried-from-loop-1 deferred flips for `dim_account` lift + `claude --bare` reintroduction — owner: teams/application/resink-core.
- [ ] Tenant-isolation dry-run post-edits; verify `grep -nrE "(resink|nanofab)" org-os/` returns no resink-core writes — owner: teams/application/resink-core.

## Cross-team asks

- **From `teams/application/agent-engineering`, review-ack only (paused this loop):** confirm the ABI contract held under real dlopen — i.e., review our consumption of the template extension once KR1.2 lands. The four `NANOFAB_NODE_*` status code values + the three function signatures (`nanofab_node_new` / `nanofab_node_process` / `nanofab_node_drop`) match what loop-1's `status_codes_match_ae_template` test pre-locked. **Reason:** AE owns the template; one-paragraph review-ack in AE's paused-team note is the lightest sufficient handshake.
- **From `teams/application/sim-farm`, no coordination needed (paused this loop):** verdict.json shape is unchanged across both feature configurations; engine 0.3 contract is unaffected by the supervisor's plugin-loading mechanism. No request, no review needed beyond sim-farm's paused-team review-ack of the dlopen swap landing GREEN.
- **From `board`, opportunistic only (no KR):** if the `resink-ai/resink-core` GitHub remote is created in-loop per board-action `2026-06-06-001`, notify resink-core so we attempt the workspace promotion opportunistically. **Reason:** the probe at brief-authoring time still returns "Repository not found" (5th-loop carry); the brief's Out-of-scope section names this loop's 5 downstream blockers. Per the brief, this is opportunistic only — no KR, no penalty for non-firing.

## Risks

- **FFI design surprises at integration time** (brief Risk #4): the unit-level shim path that loop-1 exercised could not surface issues like `ctx_ptr: *mut c_void` semantics under a real load, panic-boundary interaction with the supervisor's existing panic handlers, or library-load symbol versioning. Per the 2026-06-06 retro's mild "what didn't" #4, only this loop's integration test exercises the load-bearing path. **Mitigation:** if surprises surface, file an in-loop ADR addendum to ADR-2026-05-16-001 naming the actual signature/semantics (don't re-litigate the original); resink-core proceeds against the actual built artifact's exported symbols. The "verify-against-environment" discipline (ADR-2026-05-16-003) applies — confirm the actual exported symbols (`nm -D` or platform equivalent) match the template's declared exports before declaring KR1.2.
- **Build artifact location discovery** — the template-generated crate may emit its `.so`/`.dylib` under an unexpected target subdirectory (cross-compilation paths, workspace target sharing, `[lib] crate-type = ["cdylib"]` placement). **Mitigation:** the integration test discovers the artifact path at runtime via `cargo metadata` or an env-var override, not a hardcoded path. Document the discovery convention in `docs/user-guide.md` so future codegen patterns inherit it.
- **`make mvp-loop` under `dlopen-plugins`-on requires the supervisor to know which artifact to load.** If the orchestrator's existing dispatch path doesn't expose a plugin-path config, this loop adds it (a small revision to dispatch.py or an env var read by the supervisor at startup). **Mitigation:** document the chosen mechanism in `docs/user-guide.md` § "Common commands" alongside the dlopen-plugins-on invocation; KR1.4 verifies it works end-to-end via verdict.json shape.
- **Carried architecture.md flips from loop-1 could conflict with this loop's "Named deviations" edit.** Loop-1's status named two deferred flips (the `dim_account` isomorphic-shape lift closure + `claude --bare` reintroduction closure); KR1.6 lands all three in one pass. **Mitigation:** sequence the architecture.md edit AFTER KR1.5 lands the slippage record so the loop-1 carries close cleanly; one final read-through to confirm no contradictions before commit.

## Out of scope this loop

- **Workspace promotion** — 5th-consecutive-loop defer; carry-only opportunistic per the brief (no KR; gated on board-action `2026-06-06-001` remote firing in-loop). The brief's Out-of-scope section names the 5 downstream blockers explicitly; we do not attempt unless `git ls-remote git@github.com:resink-ai/resink-core.git` flips from "Repository not found" mid-loop.
- **Hot-swap correctness test** — ADR-2026-05-16-001 step 3, target loop+1 (= 2026-06-20). Joint AE + resink-core deliverable next loop; this loop closes step 2 only.
- **Future codegen patterns** (`scd1_first_event`, `window_stats_with_decrement`, etc.) — out of scope per the consolidation framing; AE paused this loop. Once step 2 closes, the C-ABI gap stops being re-discovered per pattern (the named cost from ADR-2026-05-16-001's "Until step 2 lands" clause), but no new patterns this loop.
- **Third dim** — engine 0.3 unlocks the option but no new dim lands this loop.
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing; carried since 2026-05-23.
- **SRE BLOCKED observable verification** — multi-loop wait; this loop closes step 2 but not step 3.
- **Modes B / C, real Kafka, real KV, multi-tenant deployment** — unchanged from prior loops.
- **New ADRs from resink-core** — board owns the 4-ADR ratification batch this loop; resink-core has no proposal to add (any FFI-design surprises file as an addendum to ADR-2026-05-16-001, not a new ADR).
