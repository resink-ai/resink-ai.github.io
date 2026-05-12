---
layout: default
title: application-sim-farm OKR — 2026-05-10
date: 2026-05-10
status: active
type: okr
loop: 2026-05-10
owner: teams/application/sim-farm
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/sim-farm
  date: 2026-05-10
  status: active
  loop: 2026-05-10
  links: parent: board/okrs/2026-05-10-ceo-brief.md
-->
# Sim Farm OKR — 2026-05-10

## Context

The 2026-05-09 product-shaped OKR delivered a validation contract for `fact_sign_up.parquet` against a Spark pipeline. The 2026-05-10 CEO brief retires that target: the runtime is now a purpose-built Rust supervisor (sub-project #1), and sim-farm is sub-project #3 of the nanofab family per the Sim Farm spec (`docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md`). Two consequences for this loop:

1. **Scope contracts**: synthetic-data generation moves to training (Sim Farm spec §2, §3.3). The 2026-05-09 carryover "minimal test-data generator" is no longer ours. Sim Farm consumes `sim/gen_data.py` and `coverage_spec.yaml` from the workspace; it does not author them.
2. **Integration target changes**: the validation target is the supervisor `--mode=sim` interface (runtime spec §8.4, Sim Farm spec §1.3) — three caller-facing execution modes (A: pre-deploy, B: per-node shadow, C: blue/green warmup). Not a Spark Job; not a parquet pipeline.

This loop is **plan-only** per CEO brief O2 — the validation contract is a written artifact. The differ libraries (`nanofab-simfarm-batch-diff`, `nanofab-simfarm-stream-diff`) are not implemented in this loop. The contract document grounds resink-core's next-loop runtime work, DevOps's CI/CD gate design, and SRE's runbook scope.

## Objectives

### O1: Re-aim the validation contract from Spark pipeline outputs to nanofab supervisor outputs

Why it matters: Sim Farm's verdict is the seal that gates training stage 3, the runtime coordinator's hot-swap state machine, and DevOps's CI/CD promotion gate. Without a re-aimed contract grounded in the supervisor `--mode=sim` interface and the three-layer coverage spec (node coverage / scenarios / tolerances), every downstream artifact next loop is suspect. Maps to CEO brief KR2.2.

**Key results**

- KR1.1: A written validation contract exists at `teams/application/sim-farm/contracts/2026-05-10-supervisor-mode-sim-validation-contract.md` that specifies — for each of the three execution modes (A pre-deploy, B per-node shadow, C blue/green warmup) — (a) what "valid supervisor output" means in terms of the write-trace and KV outputs the supervisor emits in `--mode=sim`, (b) the set of failure modes Sim Farm intends to detect (translated from the 2026-05-09 list against the new supervisor; see § Failure mode translation below), (c) the three-layer verdict structure (node coverage, scenarios, tolerances) consumed by training and the runtime coordinator. The contract uses **named hand-off sections** per the P3 retro convention — one document, sections for each downstream consumer (resink-core, SRE, DevOps).
- KR1.2: The contract document is referenced from resink-core's 2026-05-10 product-shaped OKR plan as the integration target for sub-project #1's `--mode=sim` interface design. Hand-off completed by mid-loop (2026-05-14).
- KR1.3: The translated failure-mode list (KR1.1.b) is delivered to SRE in writing, named under § Hand-off to SRE in the contract, to seed SRE's deferred "demo pipeline failed validation" runbook re-aim against nanofab supervisor failure modes (per CEO brief O3 KR3.3). Hand-off completed by mid-loop (2026-05-14).
- KR1.4: The contract document names the supervisor-side dependencies Sim Farm needs from resink-core: `--mode=sim` flag, `--write-trace=<path>` flag, control gRPC `EnableShadow(node_id, candidate_version)`, panic-event recording in trace.jsonl (per runtime spec §8.4 and Sim Farm spec §6.2). These are listed in § Cross-team asks and § Surfaces required from resink-core in the contract.

**Tasks**

- [ ] Walk the 2026-05-09 seven-failure-mode list against the nanofab supervisor; classify each as survives / translates / obsolete; record the result in the contract's § Failure mode translation table — owner: teams/application/sim-farm
- [ ] Enumerate new supervisor-specific failure modes not present in the Spark contract (supervisor panic in `--mode=sim`, write-trace socket overrun, sidecar pair-orphans, mode-A timeout, mode-B sidecar restart, DuckDB diff engine failure) — owner: teams/application/sim-farm
- [ ] Specify the three-layer verdict structure (node coverage / scenarios / tolerances) in contract terms — owner: teams/application/sim-farm
- [ ] Write the § Hand-off to resink-core section (named hand-off): the `--mode=sim` interface contract, write-trace schema requirements, control gRPC surfaces Sim Farm needs — owner: teams/application/sim-farm
- [ ] Write the § Hand-off to SRE section (named hand-off): the translated failure-mode list with per-mode severity (which trigger runtime rollback vs operator alert) — owner: teams/application/sim-farm
- [ ] Write the § Hand-off to DevOps section (named hand-off): the verdict-shape Sim Farm will produce for the CI/CD promotion gate (gate stage 3 verdict structure, expected latency envelope per mode) — owner: teams/application/sim-farm
- [ ] Reference the contract from resink-core's OKR by mid-loop (KR1.2) — owner: teams/application/sim-farm

### O2: Re-aim the per-variant assertion file format against the three execution modes

Why it matters: The 2026-05-09 carryover "per-variant assertion file format" was scoped to per-table assertions on Spark pipeline outputs. Under the new spec the equivalent artifact is the **per-mode coverage-spec evaluation rule set** — what Sim Farm asserts on a verdict's three layers (node coverage, scenarios, tolerances) for each of modes A / B / C. Without this re-aim, sim-farm next loop has nothing to ground its assertion-wiring work against.

**Key results**

- KR2.1: A written assertion-format spec exists at `teams/application/sim-farm/contracts/2026-05-10-per-mode-assertion-format.md` describing — for each of modes A / B / C — the structure of a Sim-Farm-evaluable assertion file: which verdict layers it touches, what tolerances it consults from `coverage_spec.yaml`, how PASS / FAIL / TIMEOUT / INFRA_FAILURE / DEGRADED are decided per layer. This is the per-tenant template that training will populate per workspace.
- KR2.2: The spec names the **boundary** between the workspace-owned `coverage_spec.yaml` (training authors it; sim-farm consumes it) and the **sim-farm-owned** assertion-evaluation logic. This is the explicit answer to "what does Sim Farm decide vs what does the customer/training declare?"
- KR2.3: The spec includes one fully worked example per mode (mode A: a pre-deploy gate; mode B: a per-node shadow session; mode C: a blue/green warmup verification) showing — for a hypothetical tenant — the input coverage spec and the expected verdict shape. Examples are illustrative, not implemented.

**Tasks**

- [ ] Draft the per-mode assertion-format spec for mode A (pre-deploy) — owner: teams/application/sim-farm
- [ ] Draft the per-mode assertion-format spec for mode B (per-node shadow) — owner: teams/application/sim-farm
- [ ] Draft the per-mode assertion-format spec for mode C (blue/green warmup) — owner: teams/application/sim-farm
- [ ] Write the boundary section: workspace-owned vs Sim-Farm-owned — owner: teams/application/sim-farm
- [ ] Write one worked example per mode — owner: teams/application/sim-farm

### O3: Update sim-farm charter to reflect the new scope (generation OUT)

Why it matters: The current charter (2026-05-08) still lists "Realistic data generators (per fact-table type)" as a sim-farm owned product. Per the new Sim Farm spec §3.3 ("Not a synthetic-data generator — training owns that") and CEO brief context, generation has moved to training. Leaving the charter stale will cause next-loop confusion about ownership boundaries (especially with resink-core, who owns training under sub-projects #1 and #2).

**Key results**

- KR3.1: `teams/application/sim-farm/charter.md` updated to (a) remove "Realistic data generators" from owned products, (b) name the new scope: execute + diff + verdict authority for nanofab supervisor DAGs across modes A/B/C, (c) declare `nanofab-supervisor --mode=sim` as the integration substrate, (d) reference the new Sim Farm spec path, (e) update the "Consumes" interface to include the workspace's `sim/gen_data.py` and `coverage_spec.yaml` (authored by training), and the "Produces" interface to name the typed `Verdict` consumed by training (stage 3 gate) and the runtime coordinator (hot-swap state machine).
- KR3.2: Charter date bumped to 2026-05-10; status remains `active`; previous charter content not overwritten but re-aimed.

**Tasks**

- [ ] Update charter.md per KR3.1 — owner: teams/application/sim-farm

## Cross-team asks

- **From `teams/application/resink-core`, by 2026-05-14 (mid-loop):** a **stub supervisor that exposes `--mode=sim`** (per Sim Farm spec §1.3 and runtime spec §8.4) — at minimum, the CLI surface (`--mode=sim`, `--workspace`, `--write-trace`, `--metrics-port`) documented with the expected I/O shape, even if the implementation behind it is a no-op for this loop. Reason: KR1.1 cannot specify the validation contract precisely without the surface shape it validates against. If a stub is not feasible this loop, an interface-only Markdown spec covering the CLI flags, the write-trace schema (`(node_id, event_id, key, before, after, event_ts)` per runtime spec §8.4), and the control gRPC `EnableShadow(node_id, candidate_version)` is acceptable.
- **From `teams/application/resink-core`, by 2026-05-14 (mid-loop):** confirmation that resink-core's 2026-05-10 product-shaped OKR will reference Sim Farm's validation contract (this loop's KR1.1 artifact) as the integration target for `--mode=sim`. Reason: KR1.2 hand-off needs a defined receiving endpoint in resink-core's OKR.
- **From `teams/platform/sre`, by 2026-05-14 (mid-loop):** confirmation of which contract format SRE wants for the failure-mode hand-off (free-form markdown vs structured YAML/JSON), so KR1.3's § Hand-off to SRE section lands in the format SRE will actually consume. Reason: P3 retro convention says one document with named sections for multiple consumers; the format SRE prefers shapes the section.
- **From `teams/platform/devops`, by 2026-05-14 (mid-loop):** the intended position of the Sim Farm verdict in the CI/CD promotion gate (synchronous block vs async signal; required vs advisory for the first iteration), so KR1.1 § Hand-off to DevOps describes a verdict shape DevOps can consume in its O3 first slice. Reason: DevOps's CEO-brief O3 work depends on a verdict-shape they can wire into the Helm-based deployment gate.

## Failure mode translation (preview; full table lives in the contract document)

The 2026-05-09 OKR enumerated seven failure modes against a Spark `fact_sign_up.parquet` pipeline. Translation against the nanofab supervisor in `--mode=sim`:

| 2026-05-09 mode | Status under new spec | Translated form |
|---|---|---|
| Schema drift (extra/missing column, type change) | **Survives, translated** | Write-trace `WriteRecord` schema drift between v1 and v2 plugin outputs (mode B) or between supervisor sim output and the DuckDB reference (mode A). Detected by the DuckDB diff engine (batch) and the streaming Rust differ (live shadow). |
| Duplicate `event_id` | **Survives, translated** | Supervisor's per-shard idempotency log (runtime spec) must dedupe; sim asserts no duplicate KV writes per `event_id` in the write-trace. |
| Late data (timestamp < watermark) | **Survives, translated** | Per-node watermark per runtime spec; events older than the watermark must be diverted to dead-letter or handled per node policy. Detected via write-trace inspection. |
| Missing `user_id` | **Translated, generalized** | Generic form: missing required key in `WriteRecord`. The specific key depends on the tenant's coverage spec, not Sim Farm's baseline contract. |
| Out-of-range `timestamp` | **Survives, translated** | `event_ts` validation at the node boundary — pre-epoch or future-dated events. Surfaced via the write-trace. |
| Null in required field | **Survives, translated** | Schema validation at the `WriteRecord` boundary. Per-tenant; coverage spec's tolerances layer encodes which columns are required. |
| Unknown `country_code` | **Obsolete as a baseline** | Table-specific to `fact_sign_up`. Generalized into the **scenarios layer** of the coverage spec (training-authored, per-tenant), not the Sim Farm baseline contract. Removed from the cross-tenant validation contract. |

**New failure modes introduced by the supervisor and Sim Farm architecture** (not present in the 2026-05-09 contract; added to the new contract per CEO brief mention of "seven Sim-Farm-enumerated failure modes"):

1. Supervisor panic in `--mode=sim` (recorded as `panic_event` in trace.jsonl per Sim Farm spec §6.2).
2. Write-trace socket overrun (`trace_dropped > 0` per Sim Farm spec §6.5) — degrades verdict to `SHADOW_DEGRADED`.
3. Sidecar pair-orphans (LRU eviction before partner arrives, > 1% rate per Sim Farm spec §4.4).
4. Mode-A timeout (per Sim Farm spec §5.1, partial-layer verdict surfaced).
5. Mode-B sidecar restart > 3× / 5 min (session marked UNRELIABLE per Sim Farm spec §6.4).
6. DuckDB diff engine failure (`Verdict{overall: DIFF_ENGINE_FAILURE}` per Sim Farm spec §6.3).
7. Coordinator crash with sidecar still buffering (per Sim Farm spec §6.6).

This is the working failure-mode list for the contract; final list lives in `contracts/2026-05-10-supervisor-mode-sim-validation-contract.md`.

## Risks

- **Resink-core has no stub supervisor this loop.** If the cross-team ask above is not met by mid-loop, KR1.1 falls back to a CLI-surface-only specification (acceptable per the ask's fallback clause). Mitigation: the ask is named with an explicit fallback; nothing in our plan blocks on a running binary.
- **Per-mode assertion-format spec (O2) may collapse into a subsection of the validation contract (O1).** O1 and O2 share the same coverage-spec substrate; if drafting reveals they are one document, we will consolidate and document the merge in the exec summary rather than fork artificially. Mitigation: this is a writing choice, not a scope reduction — every KR2.* still gets written; the only question is whether in one file or two.
- **The new failure-mode list grows beyond seven.** The 2026-05-09 contract had seven; the new architecture introduces at least seven additional infrastructure-shaped failure modes. SRE's runbook (CEO brief O3 KR3.3, plan-only this loop) must absorb both categories; the contract surfaces both clearly in § Hand-off to SRE.
- **Generation-removed scope creates a vacuum.** Until training (resink-core under sub-project #2) ships `sim/gen_data.py` patterns, sim-farm has no testable input even in unit tests. Mitigation: out-of-scope this loop per CEO brief plan-only framing; testing Sim Farm itself is a next-loop concern (Sim Farm spec §7).
- **DevOps may push back on the verdict shape.** If DevOps's mid-loop confirmation comes back with a CI/CD gate requirement Sim Farm cannot meet (e.g., synchronous gate with tight latency), the contract's § Hand-off to DevOps becomes a discussion artifact rather than a final spec. Mitigation: explicit cross-team ask above; named in risks for surface visibility.

## Out of scope this loop

- Building any Sim Farm component code (`nanofab-simfarm-coordinator`, batch runner, warm pool, sidecar, DuckDB diff library, streaming Rust differ, verdict crate, Iceberg writer). Per CEO brief: plan-only this loop.
- Authoring `sim/gen_data.py` or `coverage_spec.yaml` per-tenant templates. Generation is owned by training under sub-project #2 (Sim Farm spec §2, §3.3); the per-tenant coverage spec template is also a training concern.
- Stand-up of any Iceberg infrastructure (`simfarm.verdicts`, `simfarm.diffs`, `simfarm.runs`) — falls to the DevOps/SRE Serving & Deployment workstream once it exists.
- Sim Farm's own product repo creation (no P4-equivalent yet for sim-farm — defer to a loop after the validation contract is consumed by resink-core and DevOps).
- Validation for any specific tenant or fact-table type — the contract is mode-shaped and tenant-shape-agnostic per Sim Farm spec invariants §3.2.
- Drift monitoring (Sim Farm spec §4.9, §5.4) — lives in training's nightly canary, not Sim Farm; out of scope by spec.

## First build slice (next loop)

Per `org-os/playbooks/onboard-application-team.md` step 4, the smallest committable next-loop chunk:

**Wire one mode-A assertion against a stub workspace.** Specifically: pull a hand-built fixture workspace (resink-core or sim-farm authors it next loop), run a no-op stub supervisor binary in `--mode=sim`, capture the write-trace, run one DuckDB diff against a trivial reference query, produce a `Verdict` JSON conforming to the verdict schema sketched in this loop's contract. Single mode, single layer (tolerances), single dim table. This is the runtime-equivalent of the 2026-05-09 "minimal generator + first assertions" slice that was deferred — re-aimed at the nanofab supervisor and scoped to a single mode-A run.

This slice depends on: (a) resink-core having shipped or stubbed `--mode=sim` (this loop's cross-team ask), (b) a tiny fixture workspace existing somewhere (next-loop work, can be in sim-farm's directory). It does not depend on Iceberg, the coordinator, or the streaming differ.
