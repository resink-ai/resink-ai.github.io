---
layout: default
title: application-sim-farm Exec Summary — 2026-05-11-2153
date: 2026-05-11
status: active
type: exec-summary
loop: 2026-05-11-2153
owner: teams/application/sim-farm
grand_parent: Loops
parent: Loop 2026-05-11-2153
nav_order: 13
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/sim-farm
  date: 2026-05-11
  status: active
  loop: 2026-05-11-2153
  links: parent: board/okrs/2026-05-11-2153-ceo-brief.md
-->
{% raw %}

# Sim Farm Exec Summary — 2026-06-13 (paused, review-ack)

**Loop status.** Paused per CEO brief 2026-06-13. Single light review-ack ask in the brief covering resink-core's dlopen swap (board O3) and DE's new schema-JSON conventions doc (board O4). No build-phase work; no carryovers requiring code-bearing this loop. Sim-farm's engine 0.3.0 baseline holds (last touched 2026-06-06; 13/13 pytest green; verdict shape stable).

## Review-ack

- **Resink-core dlopen swap (board O3 / ADR-2026-05-16-001 step 2 closure):** Reviewed `crates/nanofab-supervisor/src/plugin_loader.rs` (real dlopen impl now exercises `libloading::Library::new` against the built `nanofab-plugin-dim-user` cdylib, with a clean mutual-exclusion gate so `cargo test --features dlopen-plugins` compiles when default `static-plugins` is also active — dlopen wins when both on); `crates/nanofab-supervisor/tests/dlopen_integration.rs` (FFI roundtrip test passes on first attempt; asserts `NANOFAB_NODE_OK = 0` for `dim_user` insert + byte-identical output state vs the static-linking path; cross-checks the wrong-table → `NANOFAB_NODE_BLOCKED = 2` mapping survives the dlopen path); and `make mvp-loop` under both feature configurations (default `static-plugins` and the new `make mvp-loop-dlopen`). **Verdict.json shape is byte-identical across feature flags** — `overall_pass: true`, `engine_version: 0.3.0`, both per-dim `pass: true`, both `mismatch_count: 0`. **Sim-farm's engine 0.3 contract held under the swap** — the supervisor's plugin-loading mechanism does not affect the verdict shape sim-farm authored; no corrections needed.
- **DE schema-JSON convention (board O4):** Reviewed `teams/platform/data-engineering/conventions/dim-schema-json.md`. Shape matches sim-farm's inlined spec from the 2026-06-06 verdict contract update under "Schema-aware columns (0.3.0+)" — `{key_columns: [string], payload_columns: [string]}`, flat object, additivity discipline (additional keys ignored), SQL-identifier-safe column names, ordering rules, the two worked examples (`dim_user_schema.json` + `dim_account_schema.json`) cite the same canonical files at `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/sim-farm-schemas/{dim_user,dim_account}.json`. The `Verified-against-environment` subsection per ADR-2026-05-16-003 is the second adopter of the discipline (after sim-farm's own 2026-06-06 adoption). **Sim-farm commits to back-reference the convention next loop** (= 2026-06-20): replace the inline "Schema-aware columns (0.3.0+)" subsection in `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` with a one-line link to DE's convention as the canonical owner; preserve the rest of the contract body (Invocation contract, backward-compat clauses, Verified-against-environment subsection). Mechanical migration; no semantic change; sim-farm picks the depth (link-only vs link+retained engine-side documentation).

## Carrying into next loop

- **Verdict-contract back-reference to DE convention** (2026-06-20, mechanical) — replace inline `Schema-aware columns (0.3.0+)` shape spec with a one-line link to `teams/platform/data-engineering/conventions/dim-schema-json.md`.
- **Modes B (per-node shadow) + C (DAG blue/green warmup)** — long-standing carry; trigger condition (post-multi-dim, post-dlopen) closer than before; remain primary candidates for the next major engine extension.
- **Sim-farm own product repo decision** — long-standing carry; trigger ("first non-Python sim-farm component") not met this loop.
- **Three-layer verdict** (Sim Farm spec §4.7) — long-standing carry; load-bearing when training-pipeline gate stage 3 adopts coverage-spec-driven verdicts.
- **Broader §6 failure-mode coverage** (panic events, write-trace overrun, sidecar pair-orphans, mode-A timeout, sidecar restart, coordinator crash) — long-standing carry; properties of the missing control plane.
{% endraw %}
