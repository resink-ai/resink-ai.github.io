---
layout: default
title: application-resink-core OKR — 2026-05-10
nav_exclude: true
render_with_liquid: false
date: 2026-05-10
status: active
type: okr
loop: 2026-05-10-2227-002
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-10
  status: active
  loop: 2026-05-10
  links: parent: board/okrs/2026-05-10-2227-002-ceo-brief.md
-->
# Resink Core OKR — 2026-05-10

## Context

The product pivot lands this loop: ADR-002 (Spark Structured Streaming) is being superseded by ADR `2026-05-10-001-nanofab-runtime-is-rust`, and ADR `2026-05-10-002-nanofab-sub-project-decomposition` formally assigns sub-projects #1 (Nanofab Runtime) and #2 (Nanofab Training Pipeline) to this team. Last loop's first-build-slice (a Spark Job → `dim_user_signup` on minikube) is **cancelled**. This loop is plan-only against the new specs ([runtime](../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md), [training](../../../docs/superpowers/specs/2026-05-10-nanofab-training-pipeline-design.md)); no production Rust code yet. Deliverables are (1) a re-aimed plan that names one narrow first slice grounded in the new specs, (2) the product repo skeleton at `repos/resink-ai/resink-core/` (this loop's P4 from the 2026-05-09 retro), and (3) explicit, dated hand-off sections for DE, sim-farm, and DevOps using the "named hand-off sections in one document" convention (P3 from 2026-05-09 retro).

## Objectives

### O1: Re-aim resink-core against the new specs; pick one narrow first build slice

Parent: `board/okrs/2026-05-10-2227-002-ceo-brief.md` — O2 KR2.1.

Why it matters: Every prior planning artifact (the 2026-05-09 OKR, status.md, the exec summary) is Spark-shaped. Until this team writes down what it owns under the new specs and names the smallest committable next-loop chunk, the next loop has to start from scratch. The first-build-slice subsection grounds next-loop planning in the new specs.

**Key results**
- KR1.1: This OKR declares ownership of sub-project #1 (Nanofab Runtime) and sub-project #2 (Nanofab Training Pipeline), citing the runtime spec §1.4 / §4 and training-pipeline spec §1.2 / §4. No Spark references anywhere.
- KR1.2: The first build slice is named in § Plan and is small enough to land in one loop after planning. The slice is grounded in named sections of the runtime + training specs.
- KR1.3: Cross-team hand-offs are written as **named sections in this single document** (P3 convention): one section each for DE (Kafka/KV contract surfaces), sim-farm (supervisor `--mode=sim` interface), and DevOps (Helm deployment shape).
- KR1.4: A "deferred to next loop" subsection lists the work this team scopes for loop 2026-05-11-1113 — including submodule promotion of `repos/resink-ai/resink-core/`, the actual implementation of the first build slice, and any next-cut design questions surfaced this loop.

**Tasks**
- [ ] Draft the re-aimed plan (§ Plan below) — owner: teams/application/resink-core
- [ ] Name the first build slice with explicit in/out, code surface touched, and acceptance — owner: teams/application/resink-core
- [ ] Write the three named hand-off sections (DE, sim-farm, DevOps) in this document — owner: teams/application/resink-core
- [ ] Enumerate deferred-to-next-loop work — owner: teams/application/resink-core

### O2: Stand up the product repo skeleton at `repos/resink-ai/resink-core/`

Parent: `board/okrs/2026-05-10-2227-002-ceo-brief.md` — O2 KR2.3.

Why it matters: P4 from the 2026-05-09 retro creates the home for the runtime + training-pipeline code. The skeleton must be a Rust Cargo workspace (no Spark/JVM scaffolding) with a README that links back to the canonical specs, so any IC opening the repo next loop sees the right pointers immediately. Real git-submodule promotion is deferred — this loop the directory lives in-tree alongside `repos/resink-ai/resink-marketplace/`.

**Key results**
- KR2.1: `repos/resink-ai/resink-core/Cargo.toml` exists as a workspace root (resolver = "2"; empty `members = []` for now; workspace-level `[workspace.package]` and `[workspace.dependencies]` placeholders).
- KR2.2: `repos/resink-ai/resink-core/README.md` exists, names the long-term layout, links back to the runtime + training-pipeline specs in newbase, and explicitly states "no Spark, no JVM, no embedded SQL engine."
- KR2.3: `repos/resink-ai/resink-core/crates/.gitkeep` exists as a placeholder for the per-component crates that land in future loops.
- KR2.4: Submodule promotion is **not** done this loop. The README states this; the next-loop OKR picks it up. No edits to newbase's `.gitmodules`.

**Tasks**
- [x] Create `repos/resink-ai/resink-core/Cargo.toml` (workspace root) — owner: teams/application/resink-core
- [x] Create `repos/resink-ai/resink-core/README.md` — owner: teams/application/resink-core
- [x] Create `repos/resink-ai/resink-core/crates/.gitkeep` — owner: teams/application/resink-core
- [ ] Defer git-submodule promotion (no `.gitmodules` edits this loop) — owner: teams/application/resink-core (recorded in § Deferred)

## Cross-team asks

These restate the named hand-off sections below as ownerful, dated asks. The hand-off sections themselves are the read-by-other-teams artifact (P3 convention).

- **From `teams/platform/data-engineering`, by mid-loop (2026-05-13):** Kafka ingress contract for the nanofab runtime, formatted as a named hand-off section in DE's own OKR or in a single shared doc DE owns. Must cover topic naming (per-source fact topic + `nanofab.internal.<dag_version>.<edge>` for cross-shard hops per runtime spec §4.5 and §5.3), partitioning convention (hash by the table's primary key), consumer-group naming convention (per supervisor fleet, per DAG version), at-least-once + manual-offset-commit contract, and dead-letter / quarantine topic naming per runtime spec §7.4 / §7.5. **Reason:** without this contract, our first-build-slice supervisor stub cannot fix its Kafka call shapes; replaces the deferred "shared streaming primitive contract" from 2026-05-09 KR2.1.
- **From `teams/application/sim-farm`, by mid-loop (2026-05-13):** confirmation that the supervisor's `--mode=sim` interface as specified in runtime spec §8.4 (`--mode=sim`, `--write-trace=path`, `--workspace=path`, control-gRPC `EnableShadow`, `reset_sim()` between jobs per sim-farm spec §4.3) is the exact integration surface they will consume in their batch-runner and warm-pool. Any deltas surfaced as a named hand-off section in sim-farm's OKR. **Reason:** locks the runtime side of the contract before our first-slice supervisor stub hard-codes the CLI surface.
- **From `teams/platform/devops`, by end of loop (2026-05-16):** an initial Helm-chart shape for the supervisor binary, parameterized on (tenant, DAG version, supervisor count, KV endpoint, Kafka bootstrap, coordinator endpoint). Stub values acceptable. Per the CEO brief, Helm is the default; DevOps may push back with a specific blocker. **Reason:** lets our next-loop first-slice implementation plug into a tenant-shaped chart instead of inventing one.

## Risks

- **Scope creep on the first build slice.** Both #1 and #2 are huge. Mitigation: § Plan caps the slice at the trace-path skeleton (§ First build slice below); novel patterns, real KV, and real Kafka are explicitly out.
- **Hand-off section content drifts from DE/sim-farm/DevOps's own OKR sections.** Mitigation: this OKR cites the runtime/sim-farm/training specs by section anchor, so the source of truth is the spec, not the hand-off paraphrase. If a peer team's OKR contradicts a spec section, that's a spec-update ask, not a hand-off mismatch.
- **In-tree repo (vs real submodule) hides cross-repo concerns we'll hit at submodule promotion.** Mitigation: README explicitly flags this; submodule promotion is the *first* deferred task.
- **ADR-002 supersession lands late.** If ADR `2026-05-10-001` slips, our O1 KR1.1 still completes because the brief itself names the pivot, but the hand-off sections need a stable ADR to cite. Mitigation: cite the spec paths directly and add the ADR reference once it lands.

## Out of scope this loop

- Any production Rust code (this is plan-only per the CEO brief).
- Real `git submodule add` for `repos/resink-ai/resink-core/` and any `.gitmodules` edit in newbase (deferred to next loop).
- Sub-project #4 (Serving & Deployment) — DevOps and SRE own.
- Sub-project #3 (Sim Farm) internals — sim-farm owns.
- Sub-project #5 (Product UX) — ownership deferred per the CEO brief.
- Customer-facing chat surface, design-fork rendering, audit view — that's sub-project #5.
- The pre-pivot Spark Job → `dim_user_signup` first slice — cancelled; no Spark anywhere this loop.

## Plan

### Ownership statement (KR1.1)

Resink-core owns:

- **Sub-project #1 — Nanofab Runtime** (per `docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md`). Specifically: the supervisor binary (§4.2), the node plugin ABI (§4.3), the state layer (§4.4), the transport layer (§4.5), the CDC sink (§4.6), the query gateway (§4.7), and the coordinator (§4.1). These ship as separate crates in the Cargo workspace at `repos/resink-ai/resink-core/`.
- **Sub-project #2 — Nanofab Training Pipeline** (per `docs/superpowers/specs/2026-05-10-nanofab-training-pipeline-design.md`). Specifically: the orchestrator (§4.1), every sub-agent (§4.3–4.9), the pattern library (§4.10), the CI builder (§4.11), and the runtime-handoff path (§4.12). Surfaced as a `training/` sibling tree in the same workspace (per README layout).

Out of scope for this team (restated for clarity): Sim Farm internals; serving/deployment topology; product UX.

### First build slice (KR1.2) — "Trace path skeleton"

**Name:** `trace-path-skeleton-v0`.

**Picked from three candidates** considered (a) supervisor binary stub with one stub cdylib node, (b) training-pipeline orchestrator stub that emits a stub `manifest.yaml` + `release_seal.json` from one parquet sample, (c) end-to-end trace-path through both halves with everything stubbed. Picked (c) because it exercises the boundary between #1 and #2 — which is the surface that bit us most in the Spark plan — at the lowest possible code cost.

**What ships next loop (loop 2026-05-11-1113):**

- A `nanofab-supervisor` crate that boots, parses `--mode=sim --workspace=<path> --write-trace=<path>`, reads a `manifest.yaml` from the workspace, `dlopen`s one stub cdylib node from the workspace's `nodes/<id>/target/`, feeds it three hand-written synthetic events from an in-memory source (no Kafka), records one stub SCD2-write line per event to an in-memory KV (no TiKV/FDB), and emits a `trace.jsonl` with one record per state mutation in the runtime spec §8.4 schema (`node_id`, `event_id`, `key`, `before`, `after`, `event_ts`). Exits 0.
- A `nanofab-node-abi` crate that defines the `Node` trait + `NodeCtx` skeleton from runtime spec §4.3 (only the surface needed by the stub node — `get`/`put_scd2`/`emit` as no-op-returning trait methods is acceptable). One `cdylib` example crate in `nodes/extract_stub/` implementing `Node`.
- A `nanofab-coordinator` crate with one binary, `nanofab-coordinator publish-dag`, that reads a hand-written `manifest.yaml` + `release_seal.json` from a workspace and prints "accepted, version v1" to stdout. No gRPC server, no Raft, no state store — just the manifest-validation surface from training spec §4.12.
- A `training/orchestrator/` Python stub that accepts one parquet sample (no profiling — just confirms the file exists), writes a hand-coded `manifest.yaml` referencing the `extract_stub` node, emits a stub `release_seal.json` with all four gate stages marked `skipped: true`, and writes them into the workspace layout from training spec §4.2. No LLM. No sub-agent dispatch. No real validation.
- One synthetic-tenant fixture in `synthetic_tenants/blog_minimal/` (per training spec §7.3) with one `fact_post_publish.parquet`, the hand-coded `manifest.yaml`, the `extract_stub` cdylib source, and a make-target that runs the orchestrator stub + supervisor and asserts `trace.jsonl` has exactly three records.

**Explicitly NOT in the first slice:**

- No real Kafka (in-memory event source only).
- No real KV (in-memory map only; no TiKV, no FoundationDB).
- No actual codegen — the `extract_stub` node is hand-written, not LLM-generated.
- No DuckDB equivalence stage, no Sim Farm dispatch — the orchestrator marks all four gate stages `skipped: true` in the seal.
- No CI builder (`cargo build` is run by hand; no content-addressed `.so` store).
- No Iceberg sink, no query gateway, no hot-swap, no cross-shard.
- No multi-tenant isolation — single hard-coded tenant.
- No production-shape error handling — `unwrap()` is acceptable in the first slice.

**Acceptance:**

- `cargo build` + `cargo test` pass on the workspace.
- `make -C synthetic_tenants/blog_minimal/ smoke` exits 0 and produces a `trace.jsonl` with 3 records.
- A second IC can run the smoke target and read `trace.jsonl` without external setup beyond `rustup`, Python 3.12, and `make`.

### Hand-off section: DE — Kafka ingress contract (KR1.3)

**Audience:** `teams/platform/data-engineering`.

**Status this loop:** request; the contract itself is DE's deliverable (their O2 KR2.4 in the CEO brief). This section names what the first build slice needs and the constraints we surface so DE's contract is grounded.

**What our first slice needs from DE's contract:**

- **Topic naming.** Per runtime spec §4.5 / §5.3: per-source fact topic, plus a convention for `nanofab.internal.<dag_version>.<edge>` cross-shard topics. We need DE to confirm the dag-version + edge-id encoding (string? semver? content hash?) — our coordinator will mint these names.
- **Partitioning convention.** Hash by the table's primary key (runtime spec §3.2 invariant #2). We need DE to confirm the hash function (e.g., MurmurHash3-32 vs xxHash64) so supervisor and producer agree.
- **Consumer-group naming.** Per supervisor fleet, per DAG version (runtime spec §6.3 blue/green requires fleet B and fleet G to be distinct consumer groups). Suggest `nanofab.<tenant>.<dag_version>.supervisor` — DE to ratify.
- **Manual offset commit + at-least-once.** Confirmed in runtime spec §4.5; DE's contract should restate so it's reviewable by downstream teams.
- **Dead-letter + quarantine topics.** Per runtime spec §7.4 (per-node DLQ) and §7.5 (quarantine for beyond-allowed-lateness). Naming convention is DE's to set.

**Constraints surfaced to DE (so the contract is groundable):**

- **Throughput tier 1 (first slice):** 3 events per smoke run. No real throughput requirement.
- **Throughput tier 2 (post-first-slice):** ~10K events/day per fact stream, matching last loop's `fact_sign_up.parquet` budget. Per-key ordering preferred. Sub-minute end-to-end latency acceptable.
- **Cross-shard fan-out:** rare and explicit per runtime spec §3.2 invariant; DE's contract does not need to optimize for high cross-shard rates.

### Hand-off section: sim-farm — supervisor `--mode=sim` interface (KR1.3)

**Audience:** `teams/application/sim-farm`.

**Status this loop:** runtime-side commitment. The exact surface is locked by runtime spec §8.4; this section restates it so sim-farm's planning has a single authoritative paraphrase.

**Surface we will ship in the first build slice:**

- **CLI flags.** `nanofab-supervisor --mode=sim --workspace=<path> --write-trace=<path>` exactly. `--workspace` points at a training-pipeline-style workspace (training spec §4.2); `--write-trace` is the JSONL output path.
- **Trace format.** One JSONL record per state mutation: `{trace_id, node_id, event_id, key, before, after, event_ts, node_version}`. `before`/`after` are JSON-encoded SCD2 row snapshots. `node_version` is required so sim-farm's shadow-sidecar correlator (sim-farm spec §4.4) can split by version.
- **Reset between jobs.** A `reset_sim()` entry point (sim-farm spec §4.3 warm-pool requirement). First slice: this is a no-op other than truncating the in-memory KV; we will expose it via control-gRPC in a later slice.
- **Determinism.** `--mode=sim` uses a fake clock advanced by `event_ts` (runtime spec §8.2); no wall-clock reads on the hot path. First slice: enforced by review only (the in-memory KV doesn't read the clock).
- **Panic capture.** Per runtime spec §7.4: `panic::catch_unwind` around `process()`; panicked events appear in `trace.jsonl` as a `panic_event` record. First slice: skeleton stub node does not panic; the wrapper is in place.

**Sim-farm asks us to confirm (and we do, here):**

- We will not ship a second engine. Sim-farm always runs the same supervisor binary, just with different startup flags.
- The trace format will not change without an ADR. If we need to evolve it, we will do so behind a `--trace-format=v2` flag with v1 supported for at least one loop.

**What we ask sim-farm to confirm by mid-loop (2026-05-13):** that the surface above matches what their batch-runner (sim-farm spec §4.2) and shadow-sidecar (§4.4) will consume. Any deltas land as a named hand-off section in sim-farm's OKR.

### Hand-off section: DevOps — Helm deployment shape (KR1.3)

**Audience:** `teams/platform/devops`.

**Status this loop:** sketch; DevOps owns the Helm chart artifact (their O3 KR3.3 in the CEO brief). This section names what shape the supervisor needs.

**Shape we need the chart to support:**

- **One Deployment per (tenant, DAG version, supervisor fleet color)** — i.e., blue/green at the chart level. Three Helm releases coexist in steady-state per tenant: blue (live), green (warming or live after cutover), and candidate (during hot-swap windows).
- **Parameters per release** (`values.yaml` shape):
  - `tenant` (string)
  - `dagVersion` (string, e.g., `v17` or content hash)
  - `fleetColor` (enum: `blue` | `green` | `candidate`)
  - `replicas` (int — supervisor count, sized by shard count)
  - `kvEndpoint` (string — TiKV/FDB endpoints; first slice: in-memory, so `"in-memory"`)
  - `kafkaBootstrap` (string; first slice: empty)
  - `coordinatorEndpoint` (string)
  - `nodePluginManifestUri` (URI — points at the content-addressed `.so` store; first slice: a local file path)
- **Sidecar.** Sim-farm's shadow-sidecar (sim-farm spec §4.4) ships in every supervisor pod, dormant by default. The chart needs a sidecar slot.
- **Service / liveness.** One Service per fleet color; readiness on the supervisor's `/healthz`; liveness on the same.
- **Local-dev parity.** First slice runs on minikube via the same chart (ADR-003 stays active). `kvEndpoint=in-memory` and `kafkaBootstrap=""` are acceptable smoke values.

**Constraints we surface to DevOps:**

- Supervisor processes are stateless and fungible (runtime spec §3.2 invariant #1) — rolling updates are safe.
- Plugin distribution is content-addressed (training spec §4.11) — the chart does not bake plugin `.so` files into the image; it pulls by URI at startup. First slice can violate this (local file path), but the chart values should already have the field.
- Coordinator is HA via Raft (runtime spec §4.1) — chart needs to support N>1 for the coordinator; first slice ships N=1.

### Deferred to next loop (KR1.4)

- **`git submodule add`** of `repos/resink-ai/resink-core/` in newbase, with a real remote URL and a `.gitmodules` entry. This loop's repo is in-tree.
- **Implement the trace-path skeleton first build slice** (§ First build slice above). Code, tests, smoke target, the lot.
- **Wire the workspace `members = []`** to include the first crates as they land.
- **Decide the LLM provider abstraction** for the training-pipeline sub-agents (training spec §9). Anthropic + a thin shim is the default; revisit if the first sub-agent surfaces a constraint.
- **First real (non-stub) node generation** from a hand-coded pattern (`scd2_counter_maintainer` from training spec §4.10). Targeted for the loop *after* the first slice ships.
- **Pick the KV vendor for the second slice** — TiKV or FoundationDB. Runtime spec §2 doesn't lock; first slice doesn't need either.
