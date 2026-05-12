---
layout: default
title: Brief
nav_order: 1
parent: "Loop 2026-05-23"
grand_parent: "Loops"
render_with_liquid: false
date: 2026-05-23
status: active
type: okr
loop: 2026-05-23
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-23
  status: active
  loop: 2026-05-23
  links: parent: ""
-->
# Resink.ai CEO Brief — 2026-05-23

## Context

Loop 2026-05-16 crossed the binary `board/charter.md` success metric: `make mvp-loop` exits 0 with `verdict=pass mismatches=0` against `synthetic_tenants/closed_loop_v0/`, demonstrated end-to-end with real LLM codegen on a single-dim fixture. The state of the company entering this loop is therefore qualitatively different from any prior loop — we have a closed customer-journey path that *runs*, not just *exists in design*. The next gap from [`ORG.md`](../../ORG.md) and the runtime spec ([docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md)) is the difference between "works on one trivial fixture" and "works on a fixture realistic enough to put weight on." Every named MVP deviation from last loop (ABI Option A; `--bare` auth; `pytz` transitive dep; multi-shard placeholder hash) is still on the books.

**The gap.** Distance from vision is now structural rather than binary. The MVP fixture exercises exactly one node, one dim, one shard, in-memory everything — the smallest possible exercise of every component. To put weight on the architecture, we need a fixture that exercises:

- **Two dims maintained simultaneously** by the same supervisor (two `Node` instances, two `dlopen`-or-static-link paths, two state-layer streams) — surfaces whether the orchestrator + supervisor have any single-node assumptions baked in.
- **Three fact streams routed to the right dim node** (`fact_sign_up` + `fact_profile_update` → `dim_user_scd2`; `fact_account_open` → `dim_account_scd2`) — surfaces whether the event-source contract supports table-discriminator routing in a multi-source setup.
- **Cross-dim referential semantics** — `dim_account.user_id` references `dim_user.user_id`. The codegen + supervisor don't need cross-key transactional joins this loop, but the fixture exercises referential consistency at the diff seam.
- **Multi-shard partitioning activated** — `shard_count = 4` per DE contract §4 (`xxHash64(seed=0)`). The placeholder fold-and-mod from last loop must be swapped to a real hash so producer/consumer agree byte-stably; the supervisor must handle per-shard event ordering correctly even when the MVP runs single-process.

**The shape of this loop.** A consolidation loop. Six activated teams (DevOps and SRE resume per the post-MVP commitment), one widened fixture, and the largest ADR ratification batch the company has authored to date: six ADRs flip from `status: draft` to `status: active` this loop, locking in the patterns that surfaced in the prior three loops.

**CEO decisions for this loop:**

- **Activate all six teams.** Per the 2026-05-16 retro's post-MVP commitment, DevOps and SRE resume — both teams' last status files date to 2026-05-10 and carry concrete first-slice scope (DevOps: nanofab-supervisor Helm chart skeleton; SRE: nanofab-supervisor-failed-validation runbook). resink-core, sim-farm, DE, AE remain active. AE's activation is strictly Bundle B per ADR-002 (see below).
- **Honor ADR-002: Bundle B is AE's hard floor.** ADR-002 ratifies this loop (O2 below). AE's primary objective is the 7 rituals tasks of Bundle B — `team-intake.md` + edits to `executive-loop.md`, `ceo-brief.md`, `team-planning.md`, `exec-summary.md`, `ceo-consolidation.md`, `retro.md`. **Strict floor:** AE does not take codegen pattern work, DISPATCH.md addenda, or any other floor pull. The `--bare` addendum (carried from 2026-05-16's retro P3-shaped ask) defers to loop 2026-05-30 alongside Bundle C. ADR-003 (verify-state-claims) also lands this loop and touches `org-os/rituals/` files — AE's Bundle B authoring coordinates merge ordering with the ADR-003 ritual edits so the two don't collide.
- **Widen the MVP fixture: two dims, three facts.** Resink-core's primary objective (O1 below) is `synthetic_tenants/closed_loop_v0_two_dim/` (or a v1 of the existing fixture — resink-core decides the naming during team planning). The closed-loop test exercises **two SCD2 dims maintained concurrently** by one supervisor process, **three fact streams** routed to the correct dim node, **`shard_count = 4` partitioning** activated end-to-end (no longer single-shard short-circuit). Both dims use the same `scd2_maintainer` codegen pattern → **no new pattern needed from AE, consistent with ADR-002's strict floor.** The orchestrator dispatches the existing `nanofab:codegen-scd2-node` skill twice (once per dim), with different `schema_json` payloads.
- **Ratify the 2026-05-23 ADR batch.** Six ADRs flip to `active` this loop, ordered to minimize merge friction:
  1. [`2026-05-09-005-carryover-load-in-brief`](../decisions/2026-05-09-005-carryover-load-in-brief.md) — adds a "Carryover load by team" section to the CEO brief authoring step.
  2. [`2026-05-09-006-org-os-change-routing`](../decisions/2026-05-09-006-org-os-change-routing.md) — out-of-retro routing path for org-os change proposals.
  3. [`2026-05-10-003-verify-state-claims-at-ritual-transitions`](../decisions/2026-05-10-003-verify-state-claims-at-ritual-transitions.md) — verifiable git location for "shipped to master" / "merged" claims.
  4. [`2026-05-10-004-contract-artifact-type`](../decisions/2026-05-10-004-contract-artifact-type.md) — adds `contract` to the `type` enum in `org-os/conventions.md` (three loops of contracts-filed-as-RFC end here).
  5. [`2026-05-16-001-abi-option-a-mvp-deviation`](../decisions/2026-05-16-001-abi-option-a-mvp-deviation.md) — ratifies Option A; names the multi-loop `dlopen` restoration plan (AE template extension loop 2026-05-30; resink-core supervisor swap loop 2026-06-06; hot-swap test loop after).
  6. [`2026-05-16-002-bundle-b-hard-floor`](../decisions/2026-05-16-002-bundle-b-hard-floor.md) — Bundle B as AE's hard floor for this loop (the very ADR this loop honors).
- **DevOps + SRE: build-mode this loop, not plan-only.** Both teams were plan-only at 2026-05-10. The supervisor binary exists; the diff engine exists; the verdict contract exists. DevOps's Helm chart skeleton has a real consumer (the supervisor) and resink-core surfaces concrete entrypoint/env/ports/shutdown-signal sections in the consolidation work. SRE's runbook has a real failure-mode catalog to reference (sim-farm's four MVP cases plus the supervisor `panic::catch_unwind` path).

**Carryover load by team** (formal section per ADR-005's ratification this loop — first brief to carry the structured tally):

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| resink-core | Workspace promotion (git submodule), `xxHash64` partition swap, multi-dim fixture widening, peer-confirmation closures (AE `--bare`, sim-farm `pytz`) | M-heavy | All folded into this loop's O1 + O6 consolidation work |
| sim-farm | Multi-dim verdict, Mode-B scoping prep, three-layer verdict spec, sim-farm own product repo decision | M | Multi-dim verdict is O1 KR; Modes B/C are stretch / next loop |
| DE | Multi-tenant + multi-shard contract addenda, ADR-004 ratification, `pytz` conventions doc, `fact_account_open` schema | S-M | Schema is O1 KR; conventions doc is O6 sub-item |
| AE | Bundle B (7 ritual tasks), Bundle C, `--bare` addendum, ADR-003 drafting, future codegen patterns | L (Bundle B only this loop) | Strict floor per ADR-002 |
| DevOps | Helm chart skeleton, frontmatter-lint CI script (still deferred), resource-budget doc, cross-team asks (resink-core supervisor surface + DE Kafka SASL) | M | Helm chart is O4 |
| SRE | nanofab-supervisor-failed-validation runbook, three verification points (sim-farm verdict-layer names, runtime `BLOCKED` observable, DevOps `tenant_id` labels), one-vs-seven runbook granularity decision | M | Runbook is O5; granularity = "one runbook" per this brief |

**Standing CEO answers (closing carryover loops):**

- **Runbook granularity (from 2026-05-10 SRE ask):** **One runbook** covering the failure modes — both sim-farm's four MVP SCD2-equivalence cases AND the seven post-MVP modes from sim-farm spec §6 — with named subsections per mode. Searchability is solved by section anchors, not by separate files.
- **DISPATCH.md `--bare` addendum (from 2026-05-16 retro):** **Deferred to 2026-05-30 alongside Bundle C.** Strict reading of ADR-002 keeps AE off all non-Bundle-B work this loop. Resink-core's consolidation work continues to use the non-`--bare` invocation form; the deviation is named in resink-core's status as a known-not-yet-fixed item rather than a blocker.
- **`pytz` dependency hygiene (from 2026-05-16 retro):** DE authors a one-paragraph "DuckDB conventions" entry under `teams/platform/data-engineering/conventions/` (new directory) this loop. Sim-farm's pyproject keeps `pytz` as a hard dep (already shipped); no upstream push.
- **Sim-farm own product repo trigger (from 2026-05-16 retro):** **Trigger condition is "first non-Python sim-farm component."** Mode-B's streaming Rust differ (`nanofab-simfarm-stream-diff`, sim-farm spec §4.6) is the natural trigger; until that component is in scope, sim-farm subtree stays under `repos/resink-ai/resink-core/sim-farm/`.

## Objectives

### O1: Widen the MVP closed loop to two dims, three facts, multi-shard partitioning

source: ceo-brief

Why it matters: A single-dim MVP closes the binary "does it run end-to-end?" question. A two-dim MVP closes "does it run when the supervisor has to coordinate?" — which is the precondition for any production-shaped customer fixture. This objective is the loop's primary deliverable; everything else is consolidation around it. The work concretely exercises the existing ADR-001 deviation (Option A static linking with two crate path-deps), the multi-shard partition function (DE contract §4 ratification), and sim-farm's verdict on multi-dim output.

**Key results**
- KR1.1: A widened fixture exists at `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v1/` (resink-core may pick a different name during planning — `closed_loop_v0` may be widened in-place if that's cleaner). The fixture contains: (a) `dim_user_fixture.parquet` extended to ≥10 users with ≥2 SCD2 updates per user (≥20 dim_user rows total); (b) `dim_account_fixture.parquet` — a new SCD2 dim with columns `(account_id, user_id, account_type, status, valid_from, valid_to, is_current)` and ≥15 accounts (some users have ≥2 accounts; ≥3 SCD2 updates total); (c) three fact parquets: `fact_sign_up.parquet`, `fact_profile_update.parquet`, `fact_account_open.parquet`. Deterministic generator + inverse-SCD2 derivation; byte-stable on re-run with the same seed.
- KR1.2: The orchestrator (`training/orchestrator/`) dispatches AE's existing `nanofab:codegen-scd2-node` skill **twice** (once per dim) against two `schema_json` payloads (`dim_user.schema.json` + `dim_account.schema.json` — both emitted by `generate.py`), produces two cdylib crates under `<workspace>/nodes/{dim_user_scd2,dim_account_scd2}/`, and writes a `manifest.yaml` declaring a DAG with two nodes routed by `table` discriminator. Stages 1 (compile) + 2 (smoke) gate both crates independently; one's failure does not silently pass the other.
- KR1.3: The supervisor binary runs two `Node` instances in one process. The in-memory event source materializes events from all three fact parquets in global `(event_ts, event_id)` order and routes each event to the correct node by the `Event.table` discriminator (runtime spec §5.4). Output: two parquets, `dim_user_output.parquet` + `dim_account_output.parquet`, plus a single combined `trace.jsonl` covering both nodes. **Determinism preserved:** running the supervisor twice on the same fixture produces byte-identical output parquets and trace JSONL.
- KR1.4: **Multi-shard partitioning activated.** Supervisor `partition()` swaps from the placeholder fold-and-mod (last loop's named gap) to `twox-hash::xxh64::xxh64(key_bytes, 0)` per DE contract §4 ratification (O2 sub-item). `shard_count = 4` in the manifest; supervisor instantiates 4 logical shard handlers (single-process — no IPC), each maintains its own KV partition, all 4 write to the shared in-memory output buffer. The shard assignment for a given primary key is byte-stable across runs and matches what a future Kafka producer would produce.
- KR1.5: Sim-farm's diff engine extends to accept **two fixture/output pairs** and emit a **single combined verdict** with per-dim mismatch breakdowns: `{verdicts: [{dim_table: "dim_user", pass, mismatch_count, mismatches}, {dim_table: "dim_account", pass, mismatch_count, mismatches}], overall_pass: bool, ...}`. The MVP loop's PASS criterion is `overall_pass == true` AND every per-dim `pass == true` AND every per-dim `mismatch_count == 0`. The Mode-A verdict contract gains an `engine_version` bump (additive — existing single-dim consumers unaffected).
- KR1.6: `make mvp-loop` exits 0 with `overall_verdict=pass` against the widened fixture. `make clean && make mvp-loop` also exits 0 (no hidden state leakage). Determinism smoke (`cargo test --release -p nanofab-supervisor --test determinism -- --include-ignored`) passes on the widened fixture.

**Tasks**
- [ ] Resink-core widens the fixture (generator + inverse-SCD2 derivation for `dim_account`; deterministic byte-stability check) — owner: teams/application/resink-core.
- [ ] Resink-core extends the orchestrator to dispatch the codegen skill per dim and write a multi-node manifest — owner: teams/application/resink-core.
- [ ] Resink-core extends the supervisor for multi-node execution + multi-shard partitioning (`xxh64` swap) — owner: teams/application/resink-core.
- [ ] Sim-farm extends the diff engine for multi-dim verdicts and bumps the verdict contract additively — owner: teams/application/sim-farm.
- [ ] DE confirms `xxHash64(seed=0)` for the partition swap (already in contract §4; ratify byte-stably with a worked example tagged for resink-core's swap) — owner: teams/platform/data-engineering.
- [ ] DE drafts the `fact_account_open` schema fragment (additive to the existing Kafka contract; cross-link from in-memory event-source contract) — owner: teams/platform/data-engineering.

### O2: Ratify the 2026-05-23 ADR batch

source: ceo-brief

Why it matters: Six ADRs have been on `status: draft` across the last three retros; consolidation means landing them. The batch covers structural ritual evolution (carryover load, out-of-retro routing, verify-state-claims) plus product-class deviations (Option A, contract enum) plus this loop's activation gate (Bundle B floor). Ratification means the board authors the ADR bodies, flips `status: draft → active`, and updates the relevant ritual/convention files where the ADR mandates an edit. Coordinated with O3 (AE Bundle B) on merge ordering so the rituals edits don't collide.

**Key results**
- KR2.1: All six ADRs in this loop's batch flip from `status: draft` to `status: active` with full body content (Context, Decision, Alternatives, Consequences, Links sections per `org-os/templates/adr.md`).
- KR2.2: Each ADR's mandated edit lands: (a) ADR-2026-05-09-005 adds the "Carryover load by team" subsection to `org-os/rituals/ceo-brief.md` (this brief already demonstrates the section — the ratification adds the rule); (b) ADR-2026-05-09-006 adds the out-of-retro routing entry to `org-os/playbooks/` (new path: `org-os/playbooks/out-of-retro-org-os-change.md` or similar); (c) ADR-2026-05-10-003 adds the verifiable-git-location step to `org-os/rituals/exec-summary.md` + `org-os/rituals/ceo-brief.md` + (optionally) `org-os/rituals/ceo-consolidation.md`; (d) ADR-2026-05-10-004 adds `contract` to the `type` enum in `org-os/conventions.md` and an "additional fields per type" row; (e) ADR-2026-05-16-001 records the multi-loop dlopen plan as an ADR; (f) ADR-2026-05-16-002 records the Bundle B floor as an ADR.
- KR2.3: Tenant-isolation invariant holds: `grep -rEi "resink|nanofab|acme\.ai" org-os/` returns only the pre-existing `acme.ai` placeholder in `conventions.md`. Re-run after each org-os edit.
- KR2.4: All three retro carryover ADRs that DE/sim-farm/AE filed contracts against (`type: rfc` workaround) get retroactive `type: contract` migration in the same loop ADR-2026-05-10-004 lands. Existing contract files (`teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md`, `…/2026-05-16-in-memory-event-source.md`, `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`, and the DISPATCH.md inside the marketplace submodule — though DISPATCH.md lives outside the conventions enum scope) update their frontmatter from `type: rfc` to `type: contract`. The workaround closes.

**Tasks**
- [ ] Author full ADR bodies for all six placeholders — owner: board.
- [ ] Land the mandated ritual / convention / playbook edits — owner: board, coordinated with AE Bundle B authoring (see O3 merge-ordering note).
- [ ] Retroactive `type: contract` migration on the three existing contract files — owner: board (or DE/sim-farm if they want to own their own; the board orchestrates).
- [ ] Tenant-isolation dry-run after each org-os edit; record output — owner: board.

### O3: AE ships Bundle B as the activated hard floor per ADR-002

source: ceo-brief

Why it matters: ADR-002 ratifies this loop with Bundle B as AE's hard floor (the very ADR Bundle B is unblocking the ratification of). Three loops of single-loop slips end here. The bundle is 7 atomic ritual edits implementing the bottom-up flow design from ADR `2026-05-09-001`. Discipline: AE refuses all other floor pulls; ADR-003 (verify-state-claims) ratification this loop also touches `org-os/rituals/` files — AE coordinates merge ordering with the board (O2 KR2.2) so the two streams don't collide on the same files.

**Key results**
- KR3.1: All 7 Bundle B tasks ship: T6 `team-intake.md` ritual created; T7 `executive-loop.md` insert team-intake step; T8 `ceo-brief.md` ratification step added; T9 `team-planning.md` three-source decomposition added; T10 `exec-summary.md` request-lifecycle closure added; T11 `ceo-consolidation.md` request roll-ups added; T12 `retro.md` one-line request closure note added.
- KR3.2: Each shipped ritual file passes its own existence + frontmatter + section check per `org-os/conventions.md`. Tenant-isolation dry-run after each task; record output cumulatively.
- KR3.3: Merge ordering with O2 KR2.2: AE Bundle B edits land in two waves. **Wave 1:** T6 + T7 + T9 + T10 + T11 + T12 (six files that don't overlap with the ADR-003 verify-state-claims edits). **Wave 2:** T8 (`ceo-brief.md`) lands AFTER O2's ADR-003 edits to `ceo-brief.md` + O2's ADR-005 carryover-load edits to `ceo-brief.md`, so the same file is edited once with both the bundle's change and both ratifications' changes folded in. AE coordinates the wave-2 timing via a mid-loop sync with the board.
- KR3.4: AE's status / charter unchanged this loop (no new owned surfaces); the loop's hygiene work is concentrated in Bundle B.

**Tasks**
- [ ] Bundle B Wave 1: T6, T7, T9, T10, T11, T12 — owner: teams/platform/agent-engineering.
- [ ] Coordinate Wave 2 timing with board (O2 KR2.2 ADR-003 + ADR-005 edits to `ceo-brief.md`) — owner: teams/platform/agent-engineering.
- [ ] Bundle B Wave 2: T8 `ceo-brief.md` ratification step (post-ADR-003 + ADR-005 landing) — owner: teams/platform/agent-engineering.
- [ ] Tenant-isolation dry-run after each task; record cumulatively — owner: teams/platform/agent-engineering.

### O4: DevOps ships the nanofab-supervisor Helm chart skeleton

source: ceo-brief

Why it matters: DevOps was plan-only at 2026-05-10 with a concrete first-slice scope (the supervisor Helm chart). The supervisor binary now exists; the consumer pressure is real. The first-slice scope from DevOps's 2026-05-10 OKR is the floor for this loop. Two consumer-side hand-offs are required from resink-core (supervisor container entrypoint/env/ports/shutdown signal) and DE (Kafka SASL credential format + broker bootstrap shape + topic naming convention).

**Key results**
- KR4.1: Helm chart skeleton lives at a path DevOps picks during planning (likely `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/` or a sibling DevOps-owned tree; DevOps's OKR commits to the location). Chart objects include: `Deployment`, `Service`, `ConfigMap` (manifest path, supervisor mode), `Secret` ref (Kafka SASL + KV credentials — empty stub for MVP), `ServiceAccount`, `RoleBinding` (IRSA placeholder per serving spec §6). The chart deploys against minikube; `helm install --dry-run` validates with stub `values.yaml`.
- KR4.2: `values.yaml` parameterizes on the axes named in resink-core's 2026-05-10 OKR § Helm hand-off: `tenant`, `dagVersion`, `fleetColor` (enum: blue|green|candidate), `replicas`, `kvEndpoint`, `kafkaBootstrap`, `coordinatorEndpoint`, `nodePluginManifestUri`. First-slice values: `kvEndpoint=in-memory`, `kafkaBootstrap=""`, single-tenant placeholder.
- KR4.3: Minikube smoke: `make -C deploy/charts/nanofab-supervisor/ minikube-smoke` (or equivalent) brings up the supervisor in `--mode=sim` against a hand-written manifest; the chart-deployed supervisor produces a `trace.jsonl` confirming it's the same binary as the locally-run MVP. Acceptance: deployed supervisor exits 0 on the same fixture and a `helm uninstall` cleans up.
- KR4.4: A named hand-off section in DevOps's OKR (or a contract under `teams/platform/devops/contracts/`) surfaces consumer-side constraints back to resink-core and DE: any container surface (entrypoint shape, signal handling, env var convention) that DevOps requires from resink-core, and any Kafka surface that DevOps requires from DE for the post-MVP Kafka path.

**Tasks**
- [ ] Draft the Helm chart skeleton with all chart objects — owner: teams/platform/devops.
- [ ] Parameterize `values.yaml` per the 2026-05-10 OKR § Helm hand-off axes — owner: teams/platform/devops.
- [ ] Minikube smoke — verify deployed supervisor matches local supervisor binary — owner: teams/platform/devops.
- [ ] Author the consumer-side constraints hand-off section — owner: teams/platform/devops.

### O5: SRE drafts the nanofab-supervisor-failed-validation runbook

source: ceo-brief

Why it matters: SRE's 2026-05-10 OKR named the runbook as next-loop work; the supervisor binary now exists, sim-farm's diff engine + verdict contract are stable, and there's a real failure surface to catalog. CEO answer to the runbook-granularity ask: **one runbook**, with named subsections per failure mode. Cross-team verifications (sim-farm verdict-layer names, runtime `BLOCKED` observable, DevOps `tenant_id` labels) land in the same loop because all three teams are now active.

**Key results**
- KR5.1: `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` exists with `status: active`. Single document with named subsections per failure mode. Covers: (a) the four sim-farm MVP SCD2-equivalence cases (`missing`, `extra`, `diverged`, plus the `engine_failure` exit code); (b) the supervisor `panic::catch_unwind` path (codegen-generated `process()` panics); (c) the manifest-validation failure path (coordinator `publish-dag` rejects a malformed seal); (d) the cargo-build failure path (codegen output that doesn't compile — stage-1 gate). Modes B/C failure modes (per sim-farm spec §6: write-trace overrun, sidecar pair-orphans, mode-A timeout, sidecar restart, coordinator crash) are stubbed with a "post-Mode-B/C" note — full coverage when those modes ship.
- KR5.2: Each subsection follows a common shape: symptom (what the operator sees), diagnosis (what to grep for in `trace.jsonl` / `verdict.json` / supervisor logs), mitigation (rollback / pause / re-dispatch path), and severity tier. Severity uses a 3-tier scale (S1 supervisor stuck, S2 wrong data emitted, S3 cosmetic / observability gap).
- KR5.3: The three cross-team verification points land: (a) **sim-farm verdict-layer names confirmed** — SRE references the exact `Verdict.mismatches[].type` enum values (`missing` / `extra` / `diverged`) from sim-farm's verdict contract; (b) **runtime `BLOCKED` observable confirmed** — resink-core supplies a one-line note on how the supervisor exposes its "blocked because sim verdict failed" state for SRE to query in production (placeholder is acceptable if the production observable doesn't exist yet; SRE marks `TODO: verify against impl` with a forward-pointer to the loop the observable is implemented); (c) **DevOps `tenant_id` label injection confirmed** — DevOps's Helm chart values include a `tenant` parameter that becomes a Kubernetes label on the Deployment; SRE references this in the runbook's "filter logs by tenant" guidance.
- KR5.4: A backlink in SRE's `status.md` once the runbook lands; the runbook is the team's primary owned-surface artifact through loop 2026-05-30.

**Tasks**
- [ ] Draft the runbook with the four MVP failure modes plus the two supervisor-side paths — owner: teams/platform/sre.
- [ ] Confirm the three cross-team verification points; place TODO markers if any cannot confirm by mid-loop — owner: teams/platform/sre.
- [ ] Add status.md backlink to the runbook on landing — owner: teams/platform/sre.

### O6: Close consolidation hygiene items distributed across teams

source: ceo-brief

Why it matters: A consolidation loop's value is in turning prior-loop named deviations + hygiene items into closed artifacts. Each sub-item below has a clear owner; each is small relative to the team's O1–O5 commitment but the cumulative effect is closing the deviation backlog.

**Key results**
- KR6.1: **Workspace promotion to a real git submodule** (resink-core). `repos/resink-ai/resink-core/` becomes a `git submodule add` with a real remote URL; `.gitmodules` in newbase carries the entry. Promotion attempted early in the loop so a slip surfaces with margin. README updated to drop the deferral note. (Carries from 2026-05-16 KR3.5 permitted pivot.)
- KR6.2: **`pytz` DuckDB conventions doc** (DE). New file `teams/platform/data-engineering/conventions/duckdb.md` (or similar — DE picks during planning) with a one-paragraph entry: DuckDB's Python driver requires `pytz` at runtime to materialize TIMESTAMPTZ rows; consumers must declare `pytz` as a hard dep in their `pyproject.toml`. Sim-farm's existing `pyproject.toml` declaration is the canonical example.
- KR6.3: **`xxHash64(seed=0)` partition swap** (resink-core, ratified by DE). Already covered as O1 KR1.4 — restated here as a consolidation deliverable so it's visible to retro.
- KR6.4: **Multi-loop `dlopen` restoration plan recorded in ADR-001** (O2 KR2.1 sub-item). The restoration is multi-loop work (template extension at 2026-05-30, supervisor swap at 2026-06-06); this loop the plan is documented in the ADR body. No code lands this loop.
- KR6.5: **Forced-red-path smoke against the live `make mvp-loop` driver** (resink-core + sim-farm). The 2026-05-16 sim-farm exec summary flagged KR2.2 as partial because only the engine-side red paths were verified, not the driver-level red propagation. This loop, run `make mvp-loop` once with a deliberately corrupted output (e.g., delete one SCD2 row before the diff) and verify the Makefile propagates the non-zero exit. Small; folds into resink-core's consolidation work.

**Tasks**
- [ ] Workspace promotion attempt + remote URL + `.gitmodules` entry — owner: teams/application/resink-core.
- [ ] `pytz` DuckDB conventions doc — owner: teams/platform/data-engineering.
- [ ] Forced-red-path smoke through `make mvp-loop` — owner: teams/application/resink-core, verified by sim-farm.
- [ ] (See O1, O2 for the other consolidation sub-items.)

## Risks

- **Six teams active for the first time means coordination cost compounds.** The MVP-focus loop's "every Wave-1 peer contract held under real Wave-2 consumption with zero addenda" property doesn't automatically extend; six teams * cross-team hand-offs grows quadratically. Mitigation: cross-team asks named explicitly in each team's OKR; O3 KR3.3's merge ordering with O2 KR2.2 is the worked example; if a hand-off mid-loop slip is detected, the consumer team uses placeholder + named-for-mid-loop-reconciliation pattern (proven last loop).
- **Multi-dim fixture surfaces orchestrator + supervisor single-dim assumptions.** The MVP supervisor was written for one node; the supervisor binary may need surgery beyond what O1 KR1.3 anticipates (e.g., shared mutable state across nodes, output-buffer concurrency, manifest schema multi-node). Mitigation: O1 KR1.3 names "single-process — no IPC" so the concurrency is limited to whatever the supervisor needs to manage internally; if it surfaces real concurrency needs, narrow O1 to one dim + flag for next loop.
- **AE Bundle B hard floor leaves the `--bare` addendum unowned this loop.** Resink-core's consolidation work continues to use the non-`--bare` invocation form. Mitigation: the brief explicitly defers the addendum to 2026-05-30; resink-core's status names the deviation as known-not-fixed-yet rather than blocking; ADR-002 is the named source of the deferral so it's defensible.
- **ADR batch ratification is the largest org-os authoring chunk the board has taken.** Six ADRs * full bodies * mandated edits = real work. Mitigation: the batch is ordered (ADR-005 → 006 → 003 → 004 → 016-001 → 016-002) so each ratification's mandated edit lands before the next; merge ordering with AE Bundle B is named in O3 KR3.3; if the batch slips, ADR-005 / 006 (oldest carryovers) are highest priority and ADR-016-001 / 002 (this-loop) are lowest in the ratification queue (they may carry to 2026-05-30 if the batch capacity-busts).
- **DevOps + SRE first build-mode loops in 4 weeks.** Both teams have plan-only history since 2026-05-08 (DevOps last shipped infrastructure work at 2026-05-09 KR3 deployment-target ADR; SRE's last shipment is plan-only artifacts). Returning to build mode means picking up their carryover load with real consumers. Mitigation: O4 KR4.3 (minikube smoke) and O5 KR5.3 (three verification points) are concrete enough to fail-fast if there's a real blocker; both teams ran a plan-only loop already against the same scope, so the scope is well-understood.
- **Workspace promotion may surface remote-URL / permissions issues mid-loop.** Carry from 2026-05-16 KR3.5. Mitigation: attempt early; permitted pivot is to defer with a single recorded blocker; the rest of O1 doesn't depend on promotion.

## Out of scope this loop

- Real Kafka path (still single-shard in-memory; multi-shard partitioning is logically activated but runs single-process).
- Real KV (in-memory `HashMap` per shard, four shards; TiKV/FoundationDB pick still deferred — the multi-loop dlopen plan in ADR-001 doesn't depend on it).
- `dlopen` restoration itself (the plan is recorded in ADR-001; execution is 2026-05-30 template extension + 2026-06-06 supervisor swap).
- AE codegen pattern expansion beyond `scd2_maintainer` (Bundle B is the strict floor; `scd1_first_event` / `window_stats_with_decrement` / etc. carry to post-Bundle-B loops).
- AE DISPATCH.md `--bare` addendum (deferred to 2026-05-30 alongside Bundle C — strict ADR-002 floor).
- Sim Farm Modes B (per-node shadow) and C (DAG blue/green warmup) — O1 KR1.5 extends the Mode-A verdict additively; Modes B/C carry to the loop after the dlopen restoration lands.
- Sim Farm three-layer verdict (coverage / scenarios / tolerances) — single-layer this loop too.
- Sim Farm own product repo — trigger condition is "first non-Python sim-farm component"; not this loop.
- AE Bundle C (T13–T17 — roles + README + worked-flow walks) — post-Bundle-B.
- Resink-core stretch capacity decision (sub-project #5 Product UX team standup vs internal split) — retro-class; not loop work.
- Product UX (sub-project #5) — ownership still deferred.
- Multi-tenant isolation — single hard-coded tenant remains.
- Frontmatter-lint CI script (DevOps) — still deferred; manual ADR-2026-05-08-001 enforcement continues.
- DevOps supervisor-binary resource-budget doc — deprioritized per 2026-05-10 status.
- Production-shape error handling, retries, observability beyond `panic::catch_unwind` + JSONL trace + DuckDB verdict.
