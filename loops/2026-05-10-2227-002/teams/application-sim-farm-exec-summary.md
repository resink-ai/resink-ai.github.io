---
layout: default
title: application-sim-farm Exec Summary — 2026-05-10-2227-002
date: 2026-05-10
status: active
type: exec-summary
loop: 2026-05-10-2227-002
owner: teams/application/sim-farm
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/sim-farm
  date: 2026-05-10
  status: active
  loop: 2026-05-10-2227-002
  links: parent: teams/application/sim-farm/okrs/2026-05-10-2227-002-team-okr.md
-->
# Sim Farm Exec Summary — 2026-05-10

## What we shipped

### O1: Re-aim the validation contract from Spark pipeline outputs to nanofab supervisor outputs
- Re-aimed the validation contract scope: integration target flipped from `fact_sign_up.parquet` Spark pipeline outputs to the nanofab supervisor `--mode=sim` interface per Sim Farm spec §1.3 and runtime spec §8.4 (three execution modes A / B / C). Documented in [`okrs/2026-05-10-team-okr.md`](../okrs/2026-05-10-2227-002-team-okr.md) § O1 Context and KR1.1.
- Walked the 2026-05-09 seven-failure-mode list against the nanofab supervisor: 5 survive/translate (schema drift, duplicate `event_id`, late data, out-of-range timestamp, null in required field), 1 generalizes (missing `user_id` → missing required key in `WriteRecord`), 1 obsolete (unknown `country_code` → scenarios layer, training-authored). Captured in OKR § Failure mode translation table.
- Enumerated 7 net-new supervisor-specific failure modes (panic in `--mode=sim`, write-trace socket overrun, sidecar pair-orphans, mode-A timeout, mode-B sidecar restart > 3×/5 min, DuckDB diff engine failure, coordinator crash with sidecar buffering) per Sim Farm spec §6.2–§6.6. In OKR § Failure mode translation.
- Observed: resink-core's 2026-05-10 OKR references this contract as the `--mode=sim` integration target — KR1.2 hand-off receiving endpoint confirmed.

### O3: Update sim-farm charter to reflect the new scope (generation OUT)
- Recorded the scope contraction: synthetic-data generation moved out of sim-farm to training (per Sim Farm spec §3.3, CEO brief context). Sim-farm now consumes `sim/gen_data.py` and `coverage_spec.yaml` from the workspace rather than authoring them.

## What we didn't ship and why

- **KR1.1 (full validation contract document at `contracts/2026-05-10-supervisor-mode-sim-validation-contract.md`)** — deferred. Plan-only this loop per CEO brief O2; the OKR plans the next-loop slice. Failure-mode translation captured inline in the OKR is the durable artifact.
- **KR1.3 § Hand-off to SRE (written, in agreed format)** — deferred. Format confirmation outstanding (cross-team ask to SRE pending).
- **KR1.4 supervisor-side dependencies named in contract** — partially recorded in OKR but the contract document itself is not landed.
- **O2 per-mode assertion-format spec (`contracts/2026-05-10-per-mode-assertion-format.md`, all KR2.* tasks)** — not started. Plan-only loop; this lives in the next slice. Risk noted in OKR § Risks: O2 may collapse into O1 as a subsection when both are drafted.
- **O3 charter update (KR3.1, KR3.2)** — not landed. The scope shift is documented in the OKR Context but the charter.md edit itself was not made this loop.
- **Carryover "minimal test-data generator" from 2026-05-09** — **obsolete, not a sim-farm deliverable at all** under the new spec. Removed from scope rather than deferred.
- **Carryover "schema + idempotency assertions against resink-core's first slice"** — deferred. Re-shaped under the new contract as "wire one mode-A assertion against a stub workspace" (OKR § First build slice).
- **Carryover "per-variant assertion file format"** — re-aimed into O2 (per-mode assertion-format spec); not landed this loop.

## Surprises

- **Generation is no longer ours.** The Sim Farm spec explicitly removes synthetic-data generation from sim-farm's scope (§3.3 "Not a synthetic-data generator — training owns that"). The 2026-05-09 carryover "minimal test-data generator" — which was the headline first build slice — is now training's deliverable, not ours. This is a clean scope reduction but invalidates a chunk of the prior loop's carry-forward planning.
- **The failure-mode list ~doubled.** Seven modes carried over (5 translate, 1 generalizes, 1 obsolete), and 7 net-new supervisor-shaped modes appeared. The contract has to absorb both categories, and SRE's runbook (CEO brief O3 KR3.3, plan-only) inherits a larger surface than implied by the 2026-05-09 framing.
- **Validation target shape changed substrate.** Moving from a Spark/parquet pipeline to a Rust supervisor `--mode=sim` interface with three execution modes (A pre-deploy / B per-node shadow / C blue/green warmup) means the verdict is not table-shaped — it's three-layer (node coverage / scenarios / tolerances) per-mode. Re-aim was clean but the assertion-format work (O2) is now mode-shaped instead of table-shaped.
- **resink-core's OKR already references our contract.** Cross-team confirmation arrived effectively for free — their OKR cites this loop's KR1.1 artifact as the integration target for sub-project #1's `--mode=sim` interface design. No follow-up needed for KR1.2.

## Asks

All asks below dated **2026-05-14 (mid-loop)** and carried forward unchanged from the OKR § Cross-team asks:

- **`teams/application/resink-core`** — a **stub supervisor that exposes `--mode=sim`** (CLI surface, write-trace schema, control gRPC `EnableShadow(node_id, candidate_version)`). Interface-only Markdown spec is the acceptable fallback. Needed because KR1.1 (next loop) cannot specify the validation contract precisely without the surface it validates against.
- **`teams/application/resink-core`** — confirmation that their 2026-05-10 OKR will reference Sim Farm's validation contract as the `--mode=sim` integration target. **STATUS: observed satisfied** — their OKR already references it. Marking closed pending CEO consolidation.
- **`teams/platform/sre`** — preferred format for the failure-mode hand-off (free-form markdown vs structured YAML/JSON), so KR1.3's § Hand-off to SRE lands in the format SRE will consume. P3 retro convention requires named hand-off sections; format choice shapes the section.
- **`teams/platform/devops`** — intended position of the Sim Farm verdict in the CI/CD promotion gate (sync block vs async signal; required vs advisory for first iteration), so the contract's § Hand-off to DevOps describes a verdict shape DevOps can wire into the Helm deployment gate.

## Metrics

OKR key results, end-of-loop state:

- **KR1.1** (validation contract document at `contracts/2026-05-10-supervisor-mode-sim-validation-contract.md`): **not landed** — failure-mode translation table delivered inline in the OKR; full contract document deferred to next loop (plan-only loop).
- **KR1.2** (contract referenced from resink-core's OKR by 2026-05-14): **observed satisfied** — resink-core's 2026-05-10 OKR already references the contract as integration target.
- **KR1.3** (translated failure-mode list delivered to SRE in writing, in agreed format, by 2026-05-14): **not landed** — translated list exists in OKR; format confirmation from SRE outstanding.
- **KR1.4** (contract names supervisor-side dependencies Sim Farm needs from resink-core): **partial** — dependencies listed in OKR text and § Cross-team asks; not yet in a contract document.
- **KR2.1 / KR2.2 / KR2.3** (per-mode assertion-format spec): **not landed** — full deferral to next loop.
- **KR3.1 / KR3.2** (charter update): **not landed** — scope shift documented in OKR Context only; charter.md edit pending.

Loop adherence: **slipping** against KR landing thresholds, but **on-track** against the CEO brief's plan-only framing — the durable artifacts (failure-mode walkthrough, scope contraction, re-aimed integration target) are captured in the OKR and ready to flow into next loop's contract document.
