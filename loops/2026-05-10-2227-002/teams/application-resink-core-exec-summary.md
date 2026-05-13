---
layout: default
title: application-resink-core Exec Summary — 2026-05-10-2227-002
date: 2026-05-10
status: active
type: exec-summary
loop: 2026-05-10-2227-002
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/resink-core
  date: 2026-05-10
  status: active
  loop: 2026-05-10-2227-002
  links: parent: teams/application/resink-core/okrs/2026-05-10-2227-002-team-okr.md
-->
{% raw %}

# Resink Core Exec Summary — 2026-05-10

## What we shipped

### O1 — Re-aim resink-core against the new specs; pick one narrow first build slice

- Re-aimed, product-shaped OKR landed declaring ownership of **sub-project #1 (Nanofab Runtime)** and **sub-project #2 (Nanofab Training Pipeline)**, citing runtime spec §1.4 / §4 and training-pipeline spec §1.2 / §4 with zero Spark references — see [§ Plan ownership statement](../okrs/2026-05-10-2227-002-team-okr.md).
- First build slice named: **`trace-path-skeleton-v0`** — end-to-end stub through both sub-projects. Slice contents: `nanofab-supervisor` boots and processes 3 hand-written events from an in-memory source; `nanofab-node-abi` defines a minimal `Node` trait + one `extract_stub` cdylib; `nanofab-coordinator publish-dag` validates a stub manifest; Python `training/orchestrator/` writes a hand-coded `manifest.yaml` + `release_seal.json` with all four gate stages `skipped: true`; one `make smoke` target asserts `trace.jsonl` has exactly 3 records. Acceptance is a clean `cargo build` + `cargo test` + `make smoke` exit 0. See [§ First build slice](../okrs/2026-05-10-2227-002-team-okr.md#first-build-slice-kr12--trace-path-skeleton).
- Three named hand-off sections written in the OKR (P3 convention): **DE** (Kafka ingress contract — topic naming, partitioning, consumer-group naming, manual-offset-commit + at-least-once, DLQ/quarantine), **sim-farm** (supervisor `--mode=sim` interface — CLI flags, trace JSONL schema, `reset_sim()`, determinism, panic capture), and **DevOps** (Helm chart shape — one Deployment per (tenant, DAG version, fleet color); blue/green/candidate; sidecar slot; local-dev parity via minikube). See [§ Hand-off sections](../okrs/2026-05-10-2227-002-team-okr.md#hand-off-section-de--kafka-ingress-contract-kr13).
- Deferred-to-next-loop subsection enumerated: submodule promotion of `repos/resink-ai/resink-core/`, first-slice implementation, workspace `members = []` wiring, LLM provider abstraction decision, first non-stub node generation (`scd2_counter_maintainer`), and KV vendor pick (TiKV vs FoundationDB). See [§ Deferred](../okrs/2026-05-10-2227-002-team-okr.md#deferred-to-next-loop-kr14).

### O2 — Stand up product repo skeleton at `repos/resink-ai/resink-core/`

- [`Cargo.toml`](../../../../repos/resink-ai/resink-core/Cargo.toml) — workspace root, `resolver = "2"`, empty `members = []`, placeholders for `[workspace.package]` + `[workspace.dependencies]`.
- [`README.md`](../../../../repos/resink-ai/resink-core/README.md) — names the long-term Rust/Cargo layout, links back to the runtime + training-pipeline specs in newbase, explicitly states "no Spark, no JVM, no embedded SQL engine," and flags that submodule promotion is deferred.
- [`crates/.gitkeep`](../../../../repos/resink-ai/resink-core/crates/.gitkeep) — placeholder for the per-component crates that land in future loops.

## What we didn't ship and why

- **`git submodule add` for `repos/resink-ai/resink-core/`** — explicitly deferred per the OKR (KR2.4); this loop the directory lives in-tree, and the README flags it. Picked up in the next-loop OKR. Recorded in § Deferred.
- **Pre-pivot first-build-slice (Spark Job → `dim_user_signup` on minikube)** — **cancelled** per the CEO brief and ADR `2026-05-10-001-nanofab-runtime-is-rust`. The Spark-shaped plan from loop 2026-05-10-2227-001 is replaced by `trace-path-skeleton-v0`, not carried.
- **Production Rust code for any of the new specs** — out of scope this loop per the CEO brief; this loop was plan-only.

## Surprises

- The "named hand-off sections in one document" P3 convention worked even better than expected: the DE Kafka-ingress contract **landed the same loop** at [`teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md`](../../../platform/data-engineering/contracts/2026-05-10-kafka-ingress.md), well ahead of its 2026-05-13 mid-loop target. Citing spec section anchors (rather than paraphrasing) appears to be the leverage point.
- Picking option (c) — end-to-end trace path through both #1 and #2 — instead of either single-side stub turned out cheap to scope precisely because we forced ourselves to write what was *out*. The "explicitly NOT in the first slice" list is longer than the slice itself, and that's the right shape.
- The team charter already says "owns Training + Serving" — which encompasses sub-projects #1 + #2 unchanged. No charter edit was needed this loop; ADR-002's supersession (and the new ADR `2026-05-10-002-nanofab-sub-project-decomposition`) records the formal ownership change without us having to touch `charter.md`.
- Submodule promotion deferral was noisier than expected — it now appears in three places (OKR § Deferred, README, next-loop carryover) because we couldn't find a single canonical home for "deferred infra plumbing." Retro candidate.

## Asks

- **DE — Kafka ingress contract by 2026-05-13 (mid-loop).** **FULFILLED THIS LOOP** at [`teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md`](../../../platform/data-engineering/contracts/2026-05-10-kafka-ingress.md). No further action from CEO; closing out.
- **Sim-farm — confirm supervisor `--mode=sim` surface (runtime spec §8.4) by 2026-05-13.** Outstanding. We need sim-farm to ratify the CLI flags + trace JSONL schema before our next-loop first-slice supervisor hard-codes them. CEO ask: nudge if not landed by mid-loop.
- **DevOps — initial Helm chart shape by end of loop 2026-05-11-0958.** Outstanding; stub values acceptable. CEO ask: confirm DevOps has capacity, or surface a specific blocker.
- **(New) ADR-002 supersession status.** Hand-off sections cite the new specs directly, but a few hand-off readers will look for the ADR. CEO ask: confirm ADR `2026-05-10-001-nanofab-runtime-is-rust` lands cleanly so we can backfill citations next loop.

## Metrics

- **KR1.1** (ownership declared, no Spark): **met** — § Plan ownership statement names #1 + #2 with spec-section citations; zero Spark references in the OKR.
- **KR1.2** (first build slice named, small, spec-grounded): **met** — `trace-path-skeleton-v0` named with explicit in/out, code surface, and acceptance; cited runtime spec §4.1–4.7 + §8.4 and training spec §4.2 / §4.12.
- **KR1.3** (three named hand-off sections): **met** — DE, sim-farm, DevOps sections written in-document; DE contract already landed.
- **KR1.4** (deferred-to-next-loop subsection): **met** — six concrete deferrals enumerated.
- **KR2.1** (`Cargo.toml` workspace root): **met**.
- **KR2.2** (README with layout + spec links + "no Spark/JVM/SQL engine"): **met**.
- **KR2.3** (`crates/.gitkeep`): **met**.
- **KR2.4** (submodule promotion deferred, no `.gitmodules` edits): **met** — README flags it; no edits to newbase `.gitmodules`.
{% endraw %}
