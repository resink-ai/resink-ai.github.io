---
layout: default
title: CEO Brief — 2026-05-11-1631
date: 2026-05-11
status: active
type: okr
loop: 2026-05-11-1631
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-11
  status: active
  loop: 2026-05-11-1631
  links: parent: ""
-->
# Resink.ai CEO Brief — 2026-06-06

## Context

Three loops produced a stable rhythm: focus (2026-05-16) → consolidation (2026-05-23) → documentation (2026-05-30). The MVP is GREEN on a two-dim fixture; the product is now self-describing via `repos/resink-ai/resink-core/CLAUDE.md` + four `docs/` files + the company's first external-facing HTML capabilities report. **What the company has not yet demonstrated is that the named MVP deviations close.** Four deviations have been carrying for two-to-three loops: (1) ABI Option A static linking instead of `dlopen` (ADR-2026-05-16-001 multi-loop plan, three-loop horizon); (2) sim-farm `user_id`-hard-coded join columns (engine 0.2.0 shape-only multi-dim, retro P1 from 2026-05-23); (3) workspace promotion deferred for three consecutive loops on the same blocker (GitHub remote doesn't exist); (4) DevOps minikube smoke deferred for two loops on the developer-machine-toolchain blocker. The documentation loop made every deviation observable but closed none.

**The gap.** Distance from vision is closure: the company built the MVP, widened it, and explained it. The next step is to deliver on the multi-loop plans that the MVP shipped under deferred. The runtime spec §4.3 hot-swap property is the load-bearing one — without it the multi-tenant supervisor architecture in `ORG.md` cannot scale. The first step of the dlopen restoration plan (AE template extension) is now due loop+1; resink-core's supervisor swap follows loop+2. This loop returns to **code-bearing build work** with multi-team active configuration.

**The shape of this loop.** A **build loop** in the focus → consolidation → documentation → build rhythm. Four teams activated; one paused with a single housekeeping ask each. No new product features; every objective closes an already-named multi-loop plan or unblocks a multi-loop plan from advancing further.

**Re P5 from the 2026-05-30 retro (documentation as recurring third loop type):** **Working answer — documentation loops are demand-driven, not pre-scheduled.** Invoked when (a) the company has shipped multiple build loops with no canonical orientation document for the primary product surface, (b) a canonical answer to "what does X do?" requires reading source, (c) an external-facing artifact is required. We do not pre-schedule them. The next documentation loop is the next loop in which these conditions accrue, not a date. The 2026-05-30 retro will revisit this answer once two more loops have shipped (target: post-2026-06-20).

**Re ADR-2026-05-16-001's multi-loop plan (the dlopen restoration):** Plan slipped one loop at 2026-05-30. This loop is the **rescheduled step 1** (AE template extension); step 2 (resink-core supervisor swap) targets `loop+1` (= 2026-06-13); step 3 (hot-swap correctness test) targets `loop+2` (= 2026-06-20). Per ADR-2026-05-30-002 (P4, ratifying this loop), this paragraph is the worked-example application of the loop+N convention — multi-loop plans henceforth use loop+N relative dating with an explicit "subject to team capacity at that loop's brief" qualifier; absolute dates only when external commitments require them.

**CEO decisions for this loop:**

- **Activate four teams + board.** Resink-core + AE + sim-farm + DevOps. Board runs ADR ratification, board-action ticket creation, and the P5 working-answer in this Context section. DE + SRE paused; each gets a one-line housekeeping ask (DE: flip the `<verified-against-rust-impl: pending>` tag in Kafka §2.1 to `verified` whenever the file next gets touched; SRE: review the supervisor swap pre-staging when the WIP lands and confirm the runbook still cross-links). No exec summary required from DE/SRE; a one-paragraph paused-team-format ack is sufficient.

- **Approve Docker Desktop on the developer machine (DevOps standing-answer carry from loop 2026-05-11-1302 § Asks).** Install Docker Desktop on the IC's Mac; run minikube with `--driver=docker`. This is one tool to install (Docker Desktop bundles the daemon) rather than two (separate daemon + minikube install). The `minikube-smoke` Helm target was authored at 2026-05-23 and unrun at 2026-05-30; this loop runs it.

- **Ratify ADR-2026-05-30-002 (multi-loop plan relative dating).** Flip from `status: draft` to `status: active`. Mandated edit: add a "Multi-loop plan note" subsection to `org-os/templates/adr.md` capturing the loop+N convention with the "subject to team capacity at that loop's brief" qualifier. Existing ADRs with absolute-date plans grandfather; future ADRs comply. The ADR's worked-example is ADR-2026-05-16-001's plan, which is itself being rescheduled in this brief — so the convention is demonstrated in-loop.

- **File a board-action ticket for workspace promotion (P1 from loop 2026-05-11-1302 retro).** The third-consecutive-loop pattern needs a forcing function. New artifact at `board/actions/2026-06-06-001-create-resink-core-github-remote.md`. This is a **tenant-level** decision (the `board/actions/` tree is a tenant convention specific to resink.ai; admitting `action` to the org-os conventions enum is a separate deferred-to-2026-06-13 org-os question — same-class follow-up to ADR-2026-05-23-001's enum batch). Frontmatter: `type: action`, `owner: board`, `date: 2026-06-06`, `status: open`, `due: 2026-06-13` (forcing function: if not resolved by then, the 2026-06-13 brief's "Out of scope" section names the implication explicitly with the count of downstream blockers). The ticket is read first at every CEO brief authoring step until it flips `status: open → status: done`.

- **Workspace promotion (resink-core KR2.1) remains attempt-not-required this loop.** If board creates the `resink-ai/resink-core` GitHub remote in-loop (the board action ticket fires this), resink-core attempts the submodule promotion. If not, the promotion stays carry; resink-core's primary work is the supervisor-swap pre-staging (KR1.2 below) which does not require the remote.

**Standing CEO answers (closing carryover loops):**

- **Schema-aware join columns shape (sim-farm + DE + resink-core coordination):** Sim-farm's `diff_scd2.py` engine 0.3 reads join + payload columns from a per-dim schema reference. Shape: a `schema_ref` argument on each `--fixture/--output/--dim-table/--schema-ref` quadruple (paired by argument order, same pattern as engine 0.2.0), pointing at a JSON file containing at minimum `{"key_columns": [...], "payload_columns": [...]}`. DE owns the canonical schema-JSON spec (a small `conventions/dim-schema-json.md` or appended section to the existing in-memory event-source contract). Resink-core's orchestrator writes the schema JSON next to each fixture/output parquet and passes the `--schema-ref` path. The `dim_account` isomorphic column-shape compromise (NAMED in resink-core's `docs/architecture.md § Named deviations`) lifts when this lands. Backward-compat: omitted `--schema-ref` falls back to engine 0.2.0 hard-coded columns (legacy path preserved).

- **AE template extension scope:** Extend `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/lib.rs.tmpl` to export `nanofab_node_process` as a C-ABI symbol per ADR-2026-05-16-001 §1's recommended form (`extern "C" fn nanofab_node_process(node_ptr: *mut Node, event_json: *const u8, event_json_len: usize, ctx: *mut NodeCtxC) -> i32` with explicit status codes for success / panic / blocked / retry). The `nanofab-node-abi` crate gains the matching `extern "C"` declaration. Smoke test: a single-process test that `dlopen`-loads a built `.so` / `.dylib` and successfully invokes `nanofab_node_process` against a known event. AE's deliverable is template + ABI declaration + smoke + marketplace tests pass.

- **Resink-core supervisor swap pre-staging (NOT execution):** Resink-core scaffolds the `libloading::Library` swap behind a feature flag (`--features dlopen-plugins`, default off; the existing Cargo path-dep path is `--features static-plugins`, default on). The supervisor binary gains the conditional load shape but does not exercise it on master this loop — full execution lands `loop+1` (= 2026-06-13) once AE's template has propagated. Resink-core's smoke this loop: cargo-build succeeds with both feature flags toggled; the unit test suite passes for both configurations. **No fixture run with `dlopen-plugins`** this loop — that requires AE's template `.so` build artifact, which AE produces this loop but resink-core integrates next loop.

- **DevOps minikube smoke scope:** `make minikube-smoke` against the existing chart skeleton at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/`. Acceptance: target exits 0; deployed supervisor's `trace.jsonl` byte-identical to local `make mvp-loop` trace via `cmp` / `sha256sum`; `helm uninstall` residual-free. Run minikube with `--driver=docker`. No CI/CD chart publishing, no multi-tenant install, no real IRSA — just the smoke.

**Carryover load by team** (per ADR-2026-05-09-005):

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| resink-core | Workspace promotion (4th-loop carry; gated on board action ticket); supervisor swap pre-staging (dlopen feature-flag scaffolding); `dim_account` isomorphic-shape compromise (lifts when sim-farm 0.3 lands this loop); `claude --bare` reintroduction (AE DISPATCH addendum landed 2026-05-30, can re-introduce now) | M | Primary code-bearing work; coordinates with AE + sim-farm in-loop |
| AE | Template extension (ADR-2026-05-16-001 step 1, rescheduled from 2026-05-30); future-codegen-pattern requests (no this-loop carry — only on demand) | M | Strict ADR-002 lineage — single bounded ADR-mandated deliverable |
| sim-farm | Schema-aware join columns (engine 0.3, retro P1 from 2026-05-23); forced-red-path smoke (2-loop carry); Modes B/C scoping (deferred); own-product-repo decision (deferred trigger) | M | Engine 0.3 lift is the headline; forced-red-path absorbs into the engine 0.3 build naturally |
| DevOps | Minikube smoke execution (KR1.3 carry, 2-loop); supervisor-side hand-off responses (5 items); DE hand-off responses (3 items); frontmatter-lint CI script (4-loop carry, defer) | M | Now unblocked by CEO Docker Desktop approval |
| board | ADR-2026-05-30-002 ratification (loop+N relative dating); board-action ticket P1 (workspace promotion forcing function); multi-loop plan slippage absorption for ADR-2026-05-16-001 | S | Smaller than 2026-05-30's three-ADR batch; focused on ratification + forcing function |
| DE | `<verified-against-rust-impl: pending>` → `verified` tag flip in Kafka §2.1 (XS, optional housekeeping); `dim-schema-json` spec coordination with sim-farm 0.3 (S, on-demand when sim-farm asks) | XS–S | Paused unless sim-farm 0.3 surfaces a contract question; one-paragraph ack |
| SRE | BLOCKED observable verification (post-dlopen; multi-loop wait); Mode-B/C runbook expansion (deferred); `main.rs` line-ref pin to commit SHA (low priority) | XS | Paused; one-paragraph ack confirming runbook still cross-links after the supervisor pre-staging lands |

## Objectives

### O1: AE ships the template extension for `nanofab_node_process` C-ABI export

source: ceo-brief

Why it matters: This is step 1 of the dlopen restoration plan from ADR-2026-05-16-001. Step 1 was originally scheduled for 2026-05-30 but slipped one loop when AE's full bandwidth went to Bundle C (the 17-task bottom-up flow closure). Step 1 must land this loop or the entire three-loop plan slides; resink-core's supervisor swap (step 2) cannot begin pre-staging until the template defines the symbol shape. Closing this step unblocks step 2 next loop and step 3 the loop after.

**Key results**
- KR1.1: `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/lib.rs.tmpl` exports `nanofab_node_process` as a C-ABI symbol per ADR-2026-05-16-001 §1's recommended form. Status codes documented in the template comments and in the ABI crate.
- KR1.2: `crates/nanofab-node-abi/src/lib.rs` (or equivalent ABI declarations file in the marketplace's reference workspace) gains the matching `extern "C"` declaration with the four status code constants (`NANOFAB_NODE_OK = 0`, `NANOFAB_NODE_PANIC = 1`, `NANOFAB_NODE_BLOCKED = 2`, `NANOFAB_NODE_RETRY = 3`).
- KR1.3: Single-process smoke test: `dlopen`-loads a built `.so` / `.dylib` from the template's reference compilation and successfully invokes `nanofab_node_process` against a known event JSON; expected return code `0`; output node state matches the static-linking-path output byte-for-byte.
- KR1.4: Marketplace tests pass: `bash repos/resink-ai/resink-marketplace/tests/run-all.sh` exits 0 with no regressions vs the 98 pass / 0 fail / 3 skip baseline from loop 2026-05-11-1302.
- KR1.5: Tenant-isolation invariant holds; dry-run after the template + ABI crate edits.

**Tasks**
- [ ] Extend `lib.rs.tmpl` with the `nanofab_node_process` C-ABI export — owner: teams/platform/agent-engineering.
- [ ] Add `extern "C"` declaration + status code constants to the ABI crate — owner: teams/platform/agent-engineering.
- [ ] Author the single-process smoke that `dlopen`-loads the built artifact — owner: teams/platform/agent-engineering.
- [ ] Run `bash tests/run-all.sh` and verify regression-free — owner: teams/platform/agent-engineering.
- [ ] Tenant-isolation dry-run — owner: teams/platform/agent-engineering.

### O2: Resink-core pre-stages the supervisor swap (dlopen feature flag) and re-introduces `claude --bare`

source: ceo-brief

Why it matters: Step 2 of the dlopen restoration plan executes next loop (loop+1 = 2026-06-13), but resink-core must scaffold the `libloading::Library` path now so the integration is a thin lift once AE's template is available. The `claude --bare` reintroduction closes the orchestrator-side dispatch shape that has been on a workaround for three loops (since the AE DISPATCH.md addendum landed at 2026-05-30 documenting when `--bare` is safe).

**Key results**
- KR2.1: `crates/nanofab-supervisor/Cargo.toml` gains two feature flags: `static-plugins` (default on; current Cargo path-dep behavior preserved) and `dlopen-plugins` (default off; `libloading::Library` path). Both feature flags compile cleanly; the unit test suite passes with each toggled.
- KR2.2: `crates/nanofab-supervisor/src/plugin_loader.rs` (or equivalent module) gains the conditional load shape: under `static-plugins`, calls the path-dep entrypoint; under `dlopen-plugins`, calls `libloading::Library::open(...)` against the codegen output's `.so` / `.dylib` path. No runtime use of `dlopen-plugins` against a real plugin this loop — that's loop+1 work. `dlopen-plugins`-on cargo-test exercises a placeholder against a no-op shim.
- KR2.3: `training/orchestrator/dispatch.py` re-introduces `--bare` to the `claude` invocation per the auth-mode rule documented in AE's DISPATCH.md (env-var auth + CI environments use `--bare`; developer-laptop OAuth keychain mode drops `--bare`). The orchestrator detects the environment automatically (presence of `ANTHROPIC_API_KEY` env var → `--bare` path; absence → drop `--bare`). `make mvp-loop` exits 0 with both env states tested locally.
- KR2.4: Workspace promotion attempted IF the `resink-ai/resink-core` GitHub remote exists at loop start. If not, resink-core does not attempt the promotion and surfaces the blocker on the board-action ticket created this loop (P1).
- KR2.5: `dim_account` isomorphic-shape compromise lifted in resink-core's orchestrator once sim-farm engine 0.3 lands (O3 below) — orchestrator writes the per-dim `schema.json` next to each fixture/output parquet and passes the `--schema-ref` arg to `sim_farm.diff_scd2 run_diff_multi`. The `dim_account` fixture/output parquets stop carrying synthetic `user_id`/`email`/`country` columns and use their natural `account_id` + `account_type` + `status` shapes.
- KR2.6: `make mvp-loop` against the two-dim fixture continues to pass under the updated shape (`overall_pass: true`, both per-dim `pass: true`, both `mismatch_count: 0`). The verdict file at `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/workspace/verdict.json` matches the canonical shape from 2026-05-23.

**Tasks**
- [ ] Add `static-plugins` and `dlopen-plugins` Cargo feature flags to the supervisor — owner: teams/application/resink-core.
- [ ] Scaffold `plugin_loader.rs` conditional load against both paths — owner: teams/application/resink-core.
- [ ] Re-introduce `--bare` to `dispatch.py` per AE's DISPATCH.md auth-mode rule — owner: teams/application/resink-core.
- [ ] Coordinate with sim-farm on schema-JSON shape + write per-dim schema.json files + pass `--schema-ref` in orchestrator — owner: teams/application/resink-core.
- [ ] Drop `dim_account` isomorphic-shape compromise; restore natural columns — owner: teams/application/resink-core.
- [ ] Verify `make mvp-loop` green under new shape — owner: teams/application/resink-core.
- [ ] Attempt workspace promotion IF GitHub remote exists; otherwise skip and surface on P1 ticket — owner: teams/application/resink-core.

### O3: Sim-farm ships engine 0.3 with schema-aware join columns

source: ceo-brief

Why it matters: This is retro P1 from loop 2026-05-11-1113 — the highest-priority architectural-debt item. Engine 0.2.0 hard-codes `KEY_COLUMNS = ("user_id", "valid_from")` and payload columns `(email, country, valid_to, is_current)`, which forced resink-core to use an isomorphic column-shape compromise for `dim_account`. The multi-dim widening at 2026-05-23 was shape-only; semantic multi-table support requires this lift. Closing it (a) restores semantic correctness to resink-core's `dim_account`, (b) closes a third-loop carry, (c) unblocks any future third dim with a different PK name.

**Key results**
- KR3.1: `repos/resink-ai/resink-core/sim-farm/sim_farm/diff_scd2.py` engine bumped `0.2.0 → 0.3.0`. New entrypoint signature: `run_diff_multi(pairs, verdict_path)` where each pair is `(fixture_path, output_path, dim_table_name, schema_ref_path)`. Legacy 0.2.0 hard-coded-column path preserved as fallback when `schema_ref_path is None` (backward compat).
- KR3.2: `--schema-ref` CLI arg added; repeated per `--fixture`/`--output`/`--dim-table` triple in argument-order pairing. Schema JSON format documented inline in module docstring: `{"key_columns": [...], "payload_columns": [...]}`.
- KR3.3: `tests/test_diff_scd2_smoke.py` gains tests: `test_multi_dim_schema_aware_both_pass`, `test_multi_dim_schema_aware_mismatched_pk`, `test_schema_ref_omitted_falls_back_to_legacy_hardcoded`, `test_engine_version_constant_is_0_3_0`. All pass. Existing 9 tests continue to pass (no regressions): 13/13 total green.
- KR3.4: Verdict contract at `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` updated to describe the schema-aware shape (still backward compatible with 0.2.0 verdict file shape). `Verified-against-environment` subsection added per ADR-2026-05-16-003 (this is the first cross-team contract to gain the section going forward; existing contracts grandfather).
- KR3.5: Coordinated with resink-core (O2 KR2.5) — sim-farm receives resink-core's per-dim schema.json files at the agreed path and engine 0.3 consumes them in `make mvp-loop` end-to-end with the lifted `dim_account` natural-column shape.
- KR3.6: Forced-red-path smoke against `make mvp-loop` driver: the schema-aware engine returns non-zero exit code with `overall_pass: false` against an intentionally-corrupted dim_account fixture. Two-loop carryover closes.

**Tasks**
- [ ] Add `--schema-ref` CLI arg + `schema_ref_path` param to `run_diff_multi` — owner: teams/application/sim-farm.
- [ ] Read key + payload columns from schema JSON, fall back to legacy hard-coded constants when omitted — owner: teams/application/sim-farm.
- [ ] Bump engine version `0.2.0 → 0.3.0` in module constant + `pyproject.toml` — owner: teams/application/sim-farm.
- [ ] Add 4 new smoke tests; verify 13/13 green — owner: teams/application/sim-farm.
- [ ] Update verdict contract with schema-aware shape + Verified-against-environment subsection — owner: teams/application/sim-farm.
- [ ] Coordinate schema-JSON shape with DE (canonical spec lives in DE's `conventions/` tree if DE agrees; otherwise embedded in sim-farm contract) — owner: teams/application/sim-farm.
- [ ] Pair-test with resink-core's orchestrator `make mvp-loop` end-to-end — owner: teams/application/sim-farm.
- [ ] Run forced-red-path smoke against corrupted dim_account fixture — owner: teams/application/sim-farm.

### O4: DevOps runs the minikube smoke against the chart skeleton

source: ceo-brief

Why it matters: The Helm chart skeleton landed at 2026-05-23 with `helm lint --strict` / `template` / `dry-run` all exit 0; the minikube smoke has been deferred for two consecutive loops on the developer-machine-toolchain blocker. Per the CEO standing answer above (Docker Desktop approval), DevOps installs Docker Desktop on the IC's machine and runs the smoke. Closing it makes the chart actually-tested (not just lint-tested) and surfaces any deployment-time gaps before the dlopen-plugin loading shape changes the supervisor binary's shape next loop.

**Key results**
- KR4.1: Docker Desktop installed on the IC's Mac; `docker version` and `minikube version` both report cleanly. Installation steps documented in `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` (one-paragraph "Prerequisites" section). On-disk evidence: `docker version` and `minikube version` output pasted verbatim into the loop's exec summary.
- KR4.2: `make minikube-smoke` from `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` exits 0. The deployed supervisor produces a `trace.jsonl` byte-identical to `make mvp-loop`'s local `trace.jsonl` via `cmp` (preferred) or `sha256sum` (fallback if `cmp` fails on metadata).
- KR4.3: `helm uninstall` exits cleanly with no residual resources (verified via `kubectl get all -A` showing no `nanofab-supervisor-` prefixed resources after uninstall).
- KR4.4: Five resink-core supervisor-side hand-off responses absorbed: Dockerfile ownership (likely DevOps for chart's Dockerfile, resink-core for the binary build), `/healthz` + `/readyz` on port 9090 (deferred until supervisor exposes them; chart keeps `healthz.enabled=false` for this loop), SIGTERM drain semantics (best-effort `kill --timeout 30` for MVP), env-var vs CLI flag convention (CLI flags for now; switch to env vars when 12-factor-ization arrives), `nodePluginManifestUri` runtime semantics (manifest path is a config-map mount for MVP).
- KR4.5: Three DE Kafka hand-off responses absorbed: SASL secret-key layout (k8s Secret with `kafka-sasl-username` + `kafka-sasl-password` keys), bootstrap-servers source-of-truth (Helm values, overridable per-environment), min-3-entries enforcement (chart `values.yaml` validation: empty/wrong-count rejected at `helm install` time via JSON schema).

**Tasks**
- [ ] Install Docker Desktop on dev machine; document prerequisites — owner: teams/platform/devops.
- [ ] Run `make minikube-smoke` and capture trace.jsonl — owner: teams/platform/devops.
- [ ] Compare deployed trace.jsonl with local — owner: teams/platform/devops.
- [ ] Verify `helm uninstall` residual-free — owner: teams/platform/devops.
- [ ] Absorb resink-core hand-off responses into chart values — owner: teams/platform/devops.
- [ ] Absorb DE Kafka hand-off responses into chart values — owner: teams/platform/devops.

### O5: Board ratifies ADR-2026-05-30-002 and files the workspace-promotion forcing-function ticket

source: ceo-brief

Why it matters: Two retro-class carries from 2026-05-30 land structurally this loop. ADR-2026-05-30-002 (P4) closes the absolute-date-plan-drift pattern by making loop+N relative dating the convention for multi-loop plans; the worked example is ADR-2026-05-16-001's rescheduling demonstrated in this brief's Context section. The board-action ticket for workspace promotion (P1) converts a recurring brief-channel ask into a tracked artifact with a forcing function — third-consecutive-loop carry transmutes into a dated tenant-tracked deliverable.

**Key results**
- KR5.1: ADR `2026-05-30-002` body filled per `org-os/templates/adr.md` shape; `status: draft → status: active`. Loop+N convention named explicitly with the "subject to team capacity at that loop's brief" qualifier. Worked example: ADR-2026-05-16-001's plan (step 1 = loop+1, step 2 = loop+2, step 3 = loop+3 — relative to ADR-2026-05-16-001's ratification loop, even though the absolute dates were drifted by the 2026-05-30 slip).
- KR5.2: `org-os/templates/adr.md` gains a "Multi-loop plan note" subsection (placed under Consequences, or as a new section between Consequences and Links) capturing the loop+N convention. Existing ADRs grandfather; future ADRs comply.
- KR5.3: `board/actions/2026-06-06-001-create-resink-core-github-remote.md` filed. Frontmatter: `type: action`, `owner: board`, `date: 2026-06-06`, `status: open`, `due: 2026-06-13`. Body: problem (three-loop carry, single specific human action external to loop-tree machinery), action required (create empty repo at `https://github.com/resink-ai/resink-core` with `master` branch only), forcing function (if not resolved by 2026-06-13, the 2026-06-13 brief's "Out of scope" section names the count of downstream deliverables this is blocking), downstream blockers known (workspace promotion, supervisor swap full execution at loop+1, hot-swap test at loop+2). The `board/actions/` directory is new; the directory is a tenant convention and lives in the tenant tree (not `org-os/`). The same loop, the brief's first read-step (per ADR-2026-05-09-005's carryover-load discipline) includes reading every `status: open` `board/actions/` ticket.
- KR5.4: `org-os/conventions.md` enum extension to add `action` is **NOT** ratified this loop (deferred to 2026-06-13 alongside ADR-2026-05-30-001 P2 to batch the org-os-enum changes naturally). For 2026-06-06, the action ticket is filed under a tenant-local provisional convention (frontmatter `type: action`, no enum membership yet); migration to the canonical `type: action` after the 2026-06-13 ADR ratification follows the same provisional-and-migrate pattern AE used at 2026-05-30 for `rfc → role`. The action ticket body includes an in-body note: "Frontmatter `type: action` is provisional pending board enum extension at 2026-06-13."
- KR5.5: Multi-loop plan slippage absorbed in-brief for ADR-2026-05-16-001: step 1 was scheduled for 2026-05-30, slipped to 2026-06-06 (this loop); step 2 was scheduled for 2026-06-06, reschedules to 2026-06-13 (loop+1); step 3 was scheduled for "loop after 2026-06-06", reschedules to 2026-06-20 (loop+2). No ADR re-litigation; the brief's Context section records the slippage; the new loop+N convention frames the going-forward dating. ADR-2026-05-16-001 body stays as-authored; the slippage record lives in this brief.
- KR5.6: Tenant-isolation invariant holds: ADR-mandated edit to `org-os/templates/adr.md` introduces no tenant strings; new `board/actions/` tree is tenant-only.

**Tasks**
- [ ] Author full body for ADR-2026-05-30-002; flip `status: draft → active` — owner: board.
- [ ] Edit `org-os/templates/adr.md` adding "Multi-loop plan note" subsection — owner: board.
- [ ] Author `board/actions/2026-06-06-001-create-resink-core-github-remote.md` with provisional `type: action` frontmatter — owner: board.
- [ ] Brief read-step audit: the loop's exec summary cites the open action ticket as carryover-load — owner: board.
- [ ] Tenant-isolation dry-run after the `org-os/templates/adr.md` edit — owner: board.

## Risks

- **AE template extension hits a C-ABI design surprise at smoke-test time.** Status code constants, FFI pointer lifetime semantics, or the `NodeCtxC` struct shape could differ from ADR-2026-05-16-001 §1's recommendation. Mitigation: AE files an in-loop addendum to the ADR (or a sibling ADR) naming the divergence rather than re-litigating the original; resink-core's pre-staging (O2) absorbs the actual signature in its `plugin_loader.rs` scaffold rather than the ADR's recommended one. The "verify-against-environment" discipline applies — AE confirms the actual built artifact's exported symbols match the template's exported symbols before declaring KR1.3.
- **Sim-farm schema-JSON spec coordination with DE is the cross-team dependency.** Engine 0.3 cannot ship without an agreed schema-JSON shape, and DE is paused this loop. Mitigation: sim-farm authors the spec embedded in its own contract update (KR3.4) and notifies DE via a one-line `<team>/requests/` filing; if DE doesn't object by mid-loop, the sim-farm-embedded spec stands. If DE objects, the spec migrates to DE's `conventions/dim-schema-json.md` next loop without breaking engine 0.3's contract.
- **Resink-core's `--bare` reintroduction (O2 KR2.3) could regress `make mvp-loop` if the env-var auto-detect logic is wrong.** Mitigation: KR2.6 explicitly verifies `make mvp-loop` green under the updated shape; if it regresses, drop `--bare` to the previous workaround and file a follow-up addendum. The DISPATCH.md addendum from 2026-05-30 documents both auth modes — the workaround is well-documented.
- **The DevOps minikube smoke (O4) could fail in ways the chart lint did not catch.** Mitigation: per OKR § Risks discipline, document the fallback path explicitly in DevOps's planning OKR — if minikube driver=docker fails, try `--driver=hyperkit` or `--driver=parallels`; if all drivers fail, the smoke captures the specific failure and surfaces as a hand-off to next loop with a known recovery path. Failure does not block O1/O2/O3 (independent work).
- **Workspace promotion attempt (O2 KR2.4) is gated on the board-action ticket firing in-loop.** If the board doesn't create the GitHub remote in the same loop as the ticket fires, KR2.4 is no-op — but the ticket's `due: 2026-06-13` forcing function then fires next loop. Mitigation: this is by design (the forcing function is the point); KR2.4 carries forward without scope creep.
- **Board ADR ratification (O5) is the smallest among this loop's deliverables.** Risk is unintentional drift in `org-os/templates/adr.md` that breaks the template for existing readers. Mitigation: the "Multi-loop plan note" subsection is additive only; existing ADRs grandfather without re-reading; no existing template field is removed or renamed.
- **The `board/actions/` tree is a new artifact path that hasn't been admitted to the org-os enum yet.** Risk: tenant-local convention drift if other tenants adopt different `actions/` shapes before the enum extension ratifies. Mitigation: the provisional-and-migrate discipline (in-body note + 2026-06-13 batch with the P2 enum extension) constrains the drift to one loop. The body of the ticket is the same shape it would have under the canonical `type: action` — only the enum membership pends.

## Out of scope this loop

- **Workspace promotion full execution** (resink-core KR2.4 only attempted if remote exists; otherwise carry).
- **`dlopen`-plugins full execution against a real plugin** — resink-core scaffolds only this loop; full run is loop+1 (= 2026-06-13).
- **Hot-swap correctness test** — ADR-2026-05-16-001 step 3, target loop+2 (= 2026-06-20). Far out.
- **Sim-farm Modes B (per-node shadow) + C (DAG blue/green warmup)** — out of scope per long-standing posture; trigger is post-multi-dim, post-dlopen.
- **Sim-farm own product repo decision** — trigger condition not yet met (first non-Python sim-farm component).
- **Three-layer verdict** (Sim Farm spec §4.7) — out of scope until training-pipeline gate stage 3 adopts coverage-spec-driven verdicts.
- **Future codegen patterns** (`scd1_first_event`, `window_stats_with_decrement`, etc.) — out of scope per the consolidation framing; AE's full bandwidth this loop is the template extension.
- **ADR-2026-05-30-001 (verify predicted syntax in OKRs)** — deferred to loop 2026-05-11-2153 per 2026-05-30 retro P2 timing.
- **`org-os` conventions enum extension to admit `action`** — deferred to loop 2026-05-11-2153; same-class batch with P2.
- **P3 (documentation-loop playbook extension)** — deferred to loop 2026-05-11-2153 or later per 2026-05-30 retro timing; requires a second documentation-loop occurrence for the two-loop track record.
- **Real Kafka, real KV, multi-tenant deployment** — all out of scope unchanged.
- **Frontmatter-lint CI script** — fourth consecutive defer per loop 2026-05-09-1715 ADR-001 framing.
- **Resink-core stretch capacity decision** (sub-project #5 Product UX team standup vs internal split) — still deferred; revisit at next MVP-class deliverable surfacing.
- **DE next-slice event-source contracts** — deferred-not-dropped; DE authors only when a consumer surfaces specific pressure.
- **SRE BLOCKED observable verification** — multi-loop wait on dlopen restoration step 2/3; this loop is step 1 only.
- **SRE Modes B/C runbook expansion** — wait on sim-farm Modes B/C.
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing housekeeping.
- **`EnterWorktree` discipline restoration** — situational; re-runs from a worktree remain straightforward when load-bearing work resumes.

## Ratifications this loop

- **ADR-2026-05-30-002** (multi-loop plan relative dating) flips `draft → active`. Mandated edit: `org-os/templates/adr.md` "Multi-loop plan note" subsection.
- **No contract migrations this loop** (all `type: rfc → contract` migrations completed at 2026-05-23; all `type: rfc → role`/`playbook`/`runbook`/`convention` migrations completed at 2026-05-30).
- **No conventions enum changes this loop** (`action` deferred to 2026-06-13).
- **Multi-loop plan slippage absorption (not a new ratification):** ADR-2026-05-16-001 plan dates slide one loop: step 1 = 2026-06-06, step 2 = 2026-06-13, step 3 = 2026-06-20. Recorded here; the ADR body stays as-authored (per the convention being ratified this loop, this is the kind of slippage that loop+N dating would have prevented going-forward).
