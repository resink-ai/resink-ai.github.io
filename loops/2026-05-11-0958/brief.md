---
layout: default
title: Brief
nav_order: 1
parent: "Loop 2026-05-11-0958"
grand_parent: "Loops"
render_with_liquid: false
date: 2026-05-16
status: active
type: okr
loop: 2026-05-11-0958
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-05-16
  status: active
  loop: 2026-05-16
  links: parent: ""
-->
# Resink.ai CEO Brief — 2026-05-16

## Context

This is a CEO-called **focus loop**. Last loop (2026-05-10) absorbed the nanofab pivot — every team's OKR is re-aimed against the new specs ([runtime](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md), [training](../../docs/superpowers/specs/2026-05-10-nanofab-training-pipeline-design.md), [sim-farm](../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md), [serving](../../docs/superpowers/specs/2026-05-10-nanofab-serving-deployment-design.md), [product-ux](../../docs/superpowers/specs/2026-05-10-nanofab-product-ux-design.md)) and ADR [2026-05-10-001](../decisions/2026-05-10-001-nanofab-runtime-is-rust.md) ratifies the Rust runtime. We have charters, plans, and one Kafka ingress contract. We do not yet have a single line of nanofab code that runs end-to-end. The vision in [`ORG.md`](../../ORG.md) is a co-design product where customers go from sample parquets to a deployed runtime; success per `board/charter.md` is "End-to-end customer journey (training → serving) works on at least one fact-table fixture." We have not crossed that bar in any form.

**The gap.** Distance from vision is a binary: there is no closed-loop test that exercises both halves of the customer journey end-to-end, even on a synthetic fixture. Every team has a plan grounded in spec sections; no team has produced a line of code that another team's code consumes. A plan-only second consecutive loop would compound the risk that the specs and plans are internally consistent but practically wrong. The cheapest way to discover what's wrong is to build the smallest closed loop that exercises every component at least once, and watch what breaks.

**The shape of this loop.** This loop builds a self-validating MVP closed loop. The customer journey, run on a developer laptop, against a single synthetic fixture:

1. We **generate a fixture** — a `dim_user` table with SCD2 history. Deterministic, seeded, reproducible.
2. We **derive synthetic fact streams** from the fixture: `fact_sign_up` (one row per user, at the user's effective-from `valid_from`) and `fact_profile_update` (one row per non-initial SCD2 version, at its `valid_from`). The derivation is the inverse of the SCD2 maintainer the runtime will run.
3. The **training pipeline**, implemented as a Claude Code skill + subagent dispatch, reads the fact parquets and emits a real (not stub) `manifest.yaml` plus the source for one SCD2-maintainer node targeting `dim_user`.
4. The **nanofab supervisor** executes the generated DAG against an in-memory event source replaying the synthetic fact streams in `event_ts` order, writes the resulting `dim_user` table to parquet.
5. **Sim Farm** diffs the output dim against the input fixture; verdict is `PASS` iff every SCD2 row matches.

If the loop closes, the company can show a real customer the same journey on real data. If it doesn't, we know exactly which seam broke first.

**CEO decisions for this loop:**

- **Activate four teams; pause two.**
  - **Activated:** `teams/application/resink-core` (owns the runtime + training-pipeline glue), `teams/application/sim-farm` (owns the closing diff/verdict, step 5), `teams/platform/data-engineering` (owns the in-memory/filesystem event-source contract for the MVP — this loop is *not* a Kafka loop), `teams/platform/agent-engineering` (provides the Claude Code skill + subagent scaffolding the training pipeline dispatches).
  - **Paused this loop:** `teams/platform/devops`, `teams/platform/sre`. The MVP runs on a developer laptop. No Helm chart, no runbook, no on-call surface is on the critical path. Both teams resume in the loop after the MVP closes (target: 2026-05-23 if this loop ships).
- **The first-slice sub-project is "both #1 and #2, minimum viable."** This is the brief-time decision pre-committed by the 2026-05-10 retro's P3 — the retro's default was "Runtime first," and that ordering still holds within the slice (the supervisor must boot before the codegen output can target it), but the loop's exit criterion is the closed-loop test, which requires both sub-projects on the path. This explicitly supersedes resink-core's prior `trace-path-skeleton-v0` plan: the new slice is bigger (real codegen, real diff) but tightly bounded (single dim, two facts, in-memory everything).
- **Defer the pre-committed retro items to a later loop.** ADR-003 (verify-state-claims at ritual transitions, scheduled for this loop per the 2026-05-10 retro) is **deferred to loop 2026-05-11-1113**. AE's Bundle B (7 rituals tasks, scheduled for this loop per their status carryover) is **deferred to loop 2026-05-11-1113**. Reason: both are org-os ritual edits, both touch the same files AE would be touching, and both compete for AE's bandwidth with the training-pipeline subagent scaffolding the MVP requires. The MVP-focus framing wins. ADR placeholder [`board/decisions/2026-05-10-003`](../decisions/2026-05-10-003-verify-state-claims-at-ritual-transitions.md) and AE's Bundle B both carry forward unchanged; status of both stays `draft` / `pending`. ADR-004, ADR-005, ADR-006 remain on their existing 2026-05-23 batch slot.

**Carryover load entering this loop** (informal — per ADR-005's still-deferred status):

- **resink-core** — `trace-path-skeleton-v0` plan from the 2026-05-10 OKR. **Superseded by O1 KR1.1–KR1.5 of this brief.** The MVP slice subsumes it: the supervisor stub from `trace-path-skeleton-v0` is now expected to consume real (LLM-generated) node code, against derived events from a real fixture, with sim-farm closing the loop. The architectural primitives are the same; the surfaces are exercised end-to-end instead of stubbed.
- **sim-farm** — validation contract document and per-mode assertion-format spec. **Re-anchored** to the MVP closed-loop diff: instead of the full `nanofab-supervisor --mode=sim` validation contract for all three modes (A/B/C) and seven failure modes, this loop ships the **Mode-A subset** needed by the MVP — the batch diff between the runtime's output dim and the fixture dim, with a verdict format. Modes B/C and the failure-mode coverage carry to the loop after MVP.
- **DE** — Kafka ingress contract is already landed. **No revision required this loop.** A consumer-side constraint may surface during MVP build (the in-memory event source's per-event shape must align with what the Kafka contract specifies for the eventual production path); if so, DE files an addendum, not a rewrite. The new MVP-shaped deliverable is a small in-memory-event-source contract, scoped under O4.
- **AE** — Bundle B (7 rituals tasks, M-heavy). **Deferred per the explicit decision above.** This loop AE pivots to training-pipeline subagent scaffolding under O3. Bundle B picks up loop 2026-05-11-1113.
- **DevOps** — Helm chart skeleton work, originally targeted for this loop. **Paused per the explicit decision above.** Picks up the loop after MVP.
- **SRE** — `nanofab-supervisor-failed-validation.md` runbook draft, originally targeted for this loop. **Paused per the explicit decision above.** Picks up the loop after MVP.

**Spec citations in this brief.** Section anchors are stable; keep team OKRs grounded in the same anchors:

- Runtime spec: §1.2 (skip SQL execution layer), §3.2 (invariants — stateless supervisor, hash-by-PK partitioning), §4.2 (supervisor binary), §4.3 (node plugin ABI), §4.4 (state layer), §5.4 (event shape), §8.2 (fake clock in sim), §8.4 (sim CLI surface and trace format).
- Training spec: §4.1 (orchestrator), §4.2 (workspace layout), §4.10 (pattern library — `scd2_counter_maintainer` and friends), §4.11 (CI builder), §4.12 (runtime-handoff path), §7.3 (synthetic-tenant fixtures).
- Sim Farm spec: §1.3 (verification authority), §4.2 (batch-runner), §4.5 (DuckDB batch diff), §6 (failure modes — Mode-A subset only this loop).

## Objectives

### O1: Close the MVP loop end-to-end on a single-dim fixture

source: ceo-brief

Why it matters: This is the entire point of the loop. Every team's deliverable below is a sub-component; this objective is the closing seam. A green `make mvp-loop` is the only acceptable proof. A red one is acceptable too — it tells us where the next loop's work goes — but a not-attempted is a loop failure.

**Key results**
- KR1.1: A `make mvp-loop` target (or equivalent single-command entry point — name to be settled by resink-core in their OKR) lives at `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/Makefile` (or equivalent path under that fixture directory). Running it from a clean checkout, with only `rustup`, Python 3.12, `make`, and the local `claude` CLI on PATH, exits 0 on the green path and prints a `PASS` line containing `verdict=pass`. Exits non-zero on the red path with a clear `FAIL` line referencing the verdict file.
- KR1.2: The synthetic fixture `synthetic_tenants/closed_loop_v0/` exists. It contains: (a) `fixtures/dim_user_fixture.parquet` — a `dim_user` table with SCD2 columns (`user_id`, `email`, `country`, `valid_from`, `valid_to`, `is_current`) and ≥3 distinct users, ≥2 of which have ≥1 SCD2 update each (so the fixture exercises both initial-insert and version-update); (b) `fixtures/generate.py` — the deterministic, seeded generator that produces the dim parquet; (c) `fixtures/derive_facts.py` — the inverse-SCD2 derivation that reads the dim and writes `fact_sign_up.parquet` (initial inserts) and `fact_profile_update.parquet` (subsequent versions). Generator + derivation are pure-Python (no LLM). Re-running them must produce byte-identical parquets given the same seed.
- KR1.3: The training pipeline successfully runs against the two fact parquets, dispatches at least one Claude Code skill via subagent (per O3), and emits to the workspace `synthetic_tenants/closed_loop_v0/workspace/`: (a) a `manifest.yaml` describing one DAG with one SCD2-maintainer node bound to `dim_user`, naming `event_id`, `event_ts`, partitioning key (`user_id`), and node-version; (b) the Rust source for that node under `nodes/dim_user_scd2/src/lib.rs` plus a per-node `Cargo.toml`; (c) a `release_seal.json` with the four gate stages — for the MVP, stages 1 (compile) and 2 (smoke) must be `passed`; stages 3 (Sim Farm) and 4 (deploy) may be `skipped: true`.
- KR1.4: The supervisor binary executes the generated DAG. Specifically: `nanofab-supervisor --mode=sim --workspace=<closed_loop_v0/workspace> --write-trace=<workspace/trace.jsonl> --write-output=<workspace/dim_user_output.parquet>`. The supervisor reads the two fact parquets via the in-memory event source (per O4), `dlopen`s the LLM-generated `dim_user_scd2.so`, drives the events through it in `event_ts` order, and writes the resulting `dim_user` SCD2 table to `dim_user_output.parquet`. Trace JSONL records every state mutation per the runtime spec §8.4 schema.
- KR1.5: Sim-farm's diff engine reads `dim_user_fixture.parquet` and `dim_user_output.parquet`, runs a DuckDB-based row-by-row SCD2 equivalence check (per Sim Farm spec §4.5), and writes a verdict to `workspace/verdict.json` with the shape defined in O2 KR2.2. The MVP loop succeeds iff `verdict.pass == true` AND `verdict.mismatch_count == 0`.

**Tasks** (cross-team; the per-team tasks live in each team's OKR)
- [ ] Resink-core owns the `make mvp-loop` driver, the supervisor + node-ABI + coordinator crates, the in-memory event source, the workspace layout, and the synthetic fixture (generator + derivation are owned by resink-core because they are coupled to the SCD2-maintainer pattern the codegen emits) — owner: teams/application/resink-core.
- [ ] Sim-farm owns the diff engine, the verdict format spec, and integration into `make mvp-loop` — owner: teams/application/sim-farm.
- [ ] DE owns the in-memory event-source contract addendum and the `Event` shape — owner: teams/platform/data-engineering.
- [ ] AE owns the training-pipeline skill + subagent scaffolding the orchestrator dispatches (per O3) — owner: teams/platform/agent-engineering.

### O2: Sim Farm ships the Mode-A diff/verdict that closes step 5

source: ceo-brief

Why it matters: The loop is open-ended without a verdict. Sim-farm's spec scope is the verification authority across three modes; the MVP needs only Mode-A (batch diff). Shipping the format + the engine + the integration this loop also gives the team a worked example to extend to Modes B/C in the loop after MVP.

**Key results**
- KR2.1: A diff engine binary or Python script (sim-farm's choice) lives in the sim-farm-owned tree (likely under `repos/resink-ai/resink-core/sim-farm/` or a submodule TBD by sim-farm in their OKR) that takes two parquet paths and produces a verdict JSON. Implementation uses DuckDB per Sim Farm spec §4.5. Handles the SCD2-equivalence cases: row exists in both with same payload (match), row exists in fixture but not output (missing), row exists in output but not fixture (extra), row exists in both with payload mismatch (diverged).
- KR2.2: `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` exists with `status: active` and defines the verdict JSON schema: `{verdict_id, mode: "A", dim_table, fixture_path, output_path, pass: bool, mismatch_count: int, mismatches: [{type: missing|extra|diverged, key: {...}, fixture_row?, output_row?}], ran_at, engine_version}`. Filed as `type: rfc` per the existing `contract` workaround (DE's standing ask, not unblocking this loop).
- KR2.3: The diff engine + verdict are wired into `make mvp-loop`'s final step. On red path, the verdict file lists the mismatches; the operator can read it without grepping the trace.
- KR2.4: Sim-farm's `charter.md` is updated to note "Mode-A batch diff shipped 2026-05-16 against the MVP loop; Modes B/C remain scoped per Sim Farm spec §1.3."

**Tasks**
- [ ] Pick the diff-engine implementation language (DuckDB CLI script vs Python via `duckdb` package) — owner: teams/application/sim-farm.
- [ ] Author the verdict-format contract document — owner: teams/application/sim-farm.
- [ ] Implement the SCD2 equivalence diff with the four mismatch types — owner: teams/application/sim-farm.
- [ ] Wire into `make mvp-loop` (coordinate with resink-core on invocation shape) — owner: teams/application/sim-farm.
- [ ] Charter touch-up — owner: teams/application/sim-farm.

### O3: AE ships the Claude Code skill + subagent scaffolding the training pipeline dispatches

source: ceo-brief

Why it matters: O1 KR1.3's "real codegen, not stub" requires a deterministic-enough surface on which the training-pipeline orchestrator can dispatch one Claude Code skill (or subagent) and get back a compilable Rust cdylib node. AE's core competence is agent infrastructure; resink-core's is the runtime + glue. This split keeps the codegen surface owned by the team that builds agent infrastructure, and the orchestration surface owned by the team that builds the training pipeline.

**Key results**
- KR3.1: A Claude Code skill `nanofab-codegen-scd2` lives at `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/SKILL.md` (or near-equivalent path AE picks; named in AE's OKR). The skill takes as input: the dim-table schema (column names, types, primary key, SCD2 column names) and the pattern name (`scd2_maintainer` for the MVP). It produces as output: Rust source for `lib.rs` + `Cargo.toml` for a cdylib that, when compiled and `dlopen`ed by the supervisor, processes events of the runtime spec §5.4 shape and emits SCD2-maintained rows for that dim.
- KR3.2: A thin subagent dispatch helper lives in the same plugin tree (e.g., `dispatch_subagent.py` or a Markdown spec the orchestrator reads) that documents how the training-pipeline orchestrator invokes the skill. The contract is: orchestrator calls some function/CLI with `(schema_json, pattern_name, output_dir)`, gets back a path to compiled-or-source artifacts and a status.
- KR3.3: The skill produces a **compilable** Rust crate for the MVP fixture's `dim_user` schema. Acceptance: `cargo build --release` in the produced crate directory exits 0. (Functional correctness of the codegen is gated by O1's end-to-end run; KR3.3 is just "the source compiles.")
- KR3.4: Tenant-isolation invariant holds: AE runs `grep -rEi "resink|nanofab|acme\.ai" org-os/` after this loop's edits and reports clean. (No `org-os/` edits are in scope this loop, but the discipline is reaffirmed.)

**Tasks**
- [ ] Pick the skill location + name; document in AE's OKR — owner: teams/platform/agent-engineering.
- [ ] Author the codegen skill (SKILL.md + any worked-example template files) — owner: teams/platform/agent-engineering.
- [ ] Document the dispatch contract resink-core's orchestrator will call — owner: teams/platform/agent-engineering.
- [ ] Smoke-test the codegen on the MVP fixture's `dim_user` schema — owner: teams/platform/agent-engineering.
- [ ] Run the tenant-isolation dry-run — owner: teams/platform/agent-engineering.

### O4: DE ships the in-memory event-source contract addendum the MVP needs

source: ceo-brief

Why it matters: The Kafka ingress contract assumes a real Kafka. The MVP runs on a laptop with parquet files. Resink-core's supervisor needs a single, named in-memory event-source shape to consume — and DE owns event-source contracts. A short addendum (or sibling doc) named with the same `kafka-ingress` framing means future contracts compose cleanly.

**Key results**
- KR4.1: `teams/platform/data-engineering/contracts/2026-05-16-in-memory-event-source.md` exists with `status: active` (filed as `type: rfc` per the still-open `contract` enum ask). Defines the `Event` record shape per runtime spec §5.4: `{table, key: {<pk>: <value>}, op: insert|update|delete, before?: <row>, after?: <row>, event_ts, event_id}`. Defines the in-memory source's contract: events arrive in `event_ts` order, partitioned by `hash(key) % shard_count`, no replay, no DLQ — the file path is the topic.
- KR4.2: The contract names how the in-memory source maps to the production Kafka contract: every field in the in-memory `Event` record has a one-to-one mapping to the Kafka contract's payload fields. The mapping is documented in a "Production parity" section so the swap from in-memory to Kafka is mechanical.
- KR4.3: Per-fact partitioning convention (hash function name + width) is restated for the MVP; resink-core's in-memory source uses the same hash so the swap is byte-stable.

**Tasks**
- [ ] Draft the in-memory event-source addendum — owner: teams/platform/data-engineering.
- [ ] Cross-check with the existing Kafka ingress contract that the field shapes align — owner: teams/platform/data-engineering.
- [ ] Surface any consumer-side constraints back to resink-core via a named hand-off section — owner: teams/platform/data-engineering.

## Risks

- **Scope: MVP is ambitious for one loop with four teams.** Mitigation: the fixture is deliberately the smallest one that exercises both halves (single dim, two facts, no cross-shard, no real KV/Kafka). If the loop slips, the failure surface is the most-informative possible signal — far more useful than another plan-only loop. We accept the risk knowingly.
- **Codegen quality: the LLM may produce non-compiling Rust.** Mitigation: O3 KR3.3 gates on `cargo build` passing; the skill author iterates the prompt until it does. The MVP doesn't need novel codegen — only the SCD2 maintainer pattern, which is in the runtime/training spec. If LLM-only proves brittle, the skill may fall back to template-with-LLM-fill (named explicitly in AE's OKR as a permitted pivot, not a stretch goal).
- **Determinism: the fixture must be byte-stable.** Mitigation: O1 KR1.2 makes byte-stable parquets a hard KR; the generator + derivation are seeded; resink-core verifies in their smoke. If parquet itself proves non-deterministic across runs, fall back to CSV (small enough) — named in resink-core's OKR as a permitted pivot.
- **Coordination cost: 4 teams in a focus loop is more than 0 teams.** Mitigation: every cross-team hand-off is named explicitly here (resink-core ↔ AE on the dispatch contract; resink-core ↔ sim-farm on the verdict invocation; resink-core ↔ DE on the `Event` shape) and is small. No team is awaiting another team's deliverable for >2 days mid-loop. If a hand-off slips, the consumer team uses the named placeholder shape and the producer files an addendum.
- **AE pivots from Bundle B to training-pipeline subagent work.** Mitigation: deferral is explicit and dated (2026-05-23). AE retains the ritual-editing context; nothing is lost. The training-pipeline skill is small (one node pattern, one dim) and within AE's competence. AE may push back via team-planning if their read says the MVP-shaped scope is wrong.
- **The closed loop may pass for the wrong reasons.** A trivial codegen output could happen to reproduce the dim if the test fixture is too easy. Mitigation: O1 KR1.2 mandates ≥2 SCD2 updates so the maintainer has to actually carry version state forward correctly. Sim-farm's diff distinguishes missing/extra/diverged so a partial pass shows up.
- **DevOps + SRE pause may surface need.** If MVP build reveals an operational dependency (e.g., the supervisor needs a Helm chart to run on a CI runner the team picks for verification), we revisit during this loop's mid-loop status check rather than waiting for retro. Both teams are paused, not unreachable.

## Out of scope this loop

- Real Kafka (in-memory event source only — DE owns the future swap path).
- Real KV (in-memory `HashMap` per shard — TiKV/FoundationDB not picked yet; second-slice work).
- DLQ / quarantine / panic recovery beyond `panic::catch_unwind` skeleton (covered by sim-farm Modes B/C in the loop after MVP).
- Multi-tenant isolation — single hard-coded tenant.
- Hot swap, blue/green, shadow sidecar — sim-farm Mode-B/C work; not on the MVP path.
- Helm chart skeleton (DevOps paused).
- Runbooks and on-call posture (SRE paused).
- AE Bundle B (7 rituals tasks) — deferred to 2026-05-23.
- ADR-003 (verify-state-claims at ritual transitions) — deferred to 2026-05-23.
- ADR-004 / ADR-005 / ADR-006 — remain on their existing 2026-05-23 batch slot.
- Sub-project #5 (Product UX) — ownership still deferred.
- Sub-project #4 (Serving & Deployment) implementation — DevOps + SRE paused; no work this loop.
- Codegen patterns beyond `scd2_maintainer` — pattern-library expansion is post-MVP.
- More than one dim table or more than two fact streams in the fixture.
- Production-shape error handling, retries, observability beyond what's needed for the MVP smoke and verdict.
