---
layout: default
title: "ADR 2026-05-16-001: abi option a mvp deviation"
parent: "Decisions (ADRs)"
render_with_liquid: false
date: 2026-05-16
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-16
  status: active
  decision: "Ratify ABI Option A (Cargo path-dep, no dlopen) as the named MVP runtime-spec deviation; name the multi-loop plan to restore dlopen via a `nanofab_node_process` C-ABI export"
-->
# ADR 2026-05-16-001: ABI Option A — MVP Runtime-Spec Deviation + dlopen Restoration Plan

## Context

The nanofab runtime spec ([docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md)) §4.3 describes the supervisor as `dlopen`-ing per-tenant cdylib node plugins so that node code can be hot-swapped without restarting the supervisor. During the 2026-05-16 MVP-focus loop, resink-core discovered that AE's `codegen-scd2-node` skill template exports only `nanofab_node_new` and `nanofab_node_drop` as C-ABI symbols — there is no `nanofab_node_process` symbol. Pure `dlopen` is impossible without either extending the template or shipping a Rust shim. Resink-core chose **Option A** (statically link the codegen output as a Cargo path-dep) for MVP simplicity, knowingly sacrificing the hot-swap property; the supervisor binary is rebuilt per-tenant. The MVP closed loop is green; this ADR ratifies the deviation and names the multi-loop plan to restore the spec.

## Decision

The board ratifies Option A as the named MVP-shape deviation from the runtime spec §4.3. For MVP, the supervisor binary statically links codegen output as a Cargo path-dep crate; the binary is rebuilt per-tenant per-DAG-version. Hot-swap is sacrificed knowingly: a node-version bump requires a supervisor rebuild and redeploy, not an in-place plugin reload. The deviation is time-boxed via a three-loop restoration plan, each step owner-assigned:

1. **AE (loop 2026-05-11-1302):** Extend the `templates/scd2_maintainer/lib.rs.tmpl` to export `nanofab_node_process` as a C-ABI symbol. Recommended form: `extern "C" fn nanofab_node_process(node_ptr: *mut Node, event_json: *const u8, event_json_len: usize, ctx: *mut NodeCtxC) -> i32` with explicit success / panic / blocked / retry status codes. The `nanofab-node-abi` crate gains the matching declaration. AE's deliverable is the template extension plus a single-process smoke that the new symbol is exported and callable.
2. **Resink-core (loop 2026-05-11-1631):** Swap the supervisor from Cargo path-dep to `libloading::Library::open(...)` against the codegen output's `.so` / `.dylib`. The static-linking path remains as a feature-gated fallback (`--features static-plugins`) so emergency rollback is possible without retreating to this ADR. Resink-core's deliverable is the supervisor consuming a dlopen-loaded plugin against the existing widened MVP fixture with byte-identical output.
3. **Joint, loop after 2026-06-06:** Hot-swap correctness test against a "candidate" plugin version. Demonstrate that a node-version bump can land without a supervisor restart and that in-flight events are not lost across the swap. The test fixture exercises a deliberate logic change in the codegen output between v1 and v2 of the same `dim_user_scd2` node.

Until step 2 lands, every new codegen pattern (post-`scd2_maintainer`) MUST be authored against the Option-A static-linking path; AE re-discovers the C-ABI gap per pattern is the named cost.

## Alternatives considered

- **A: Status quo (Option A permanent).** Rejected because hot-swap is a load-bearing operational property in the runtime spec (multi-tenant supervisor fleet cannot tolerate per-tenant rebuild cycles at production scale).
- **B: Extend the template this loop.** Rejected because the MVP-focus framing intentionally narrowed AE's bandwidth onto the codegen surface; a same-loop template extension would have been retry-shaped work against an already-passing MVP.
- **C: Skip dlopen entirely and standardize on static linking + per-tenant rebuild.** Rejected because the multi-tenant scale (runtime spec §3.2 invariant #1) needs a separation of plugin lifecycle from supervisor lifecycle; static linking forces them to be coupled.

## Consequences

- **Positive:** MVP loop is green; the runtime spec's deviation is documented and time-boxed; consumer teams (sim-farm, DE) have a worked example of how cross-team contracts can ship with a named limitation. Determinism is preserved (static linking is byte-stable). The multi-loop plan is concrete.
- **Negative / costs:** Hot-swap property delayed by ~3 loops. Two follow-up template/supervisor edits required. Until the swap lands, every new codegen pattern carries the same C-ABI gap (resink-core re-discovers per pattern).
- **Follow-ups required:** AE template extension (loop 2026-05-11-1302). Supervisor swap (loop 2026-05-11-1631). Hot-swap test (loop after). Update runtime spec § 4.3 with a "deviation in force until 2026-06-XX" note when the ADR ratifies.

## Links

- Triggering retro: [board/retros/2026-05-11-0958-ceo-retro.md](../retros/2026-05-11-0958-ceo-retro.md) — P1.
- Related ADRs: [2026-05-10-001-nanofab-runtime-is-rust](2026-05-10-001-nanofab-runtime-is-rust.md), [2026-05-10-002-nanofab-sub-project-decomposition](2026-05-10-002-nanofab-sub-project-decomposition.md).
- Codegen template: [`repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/lib.rs.tmpl`](../../repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/lib.rs.tmpl).
- MVP supervisor: [`repos/resink-ai/resink-core/crates/nanofab-supervisor/`](../../repos/resink-ai/resink-core/crates/nanofab-supervisor/).
