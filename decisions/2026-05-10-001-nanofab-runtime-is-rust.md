---
layout: default
title: "ADR 2026-05-10-001: nanofab runtime is rust"
date: 2026-05-10
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-10
  status: active
  decision: The nanofab runtime is a purpose-built distributed Rust application; no JVM, no general SQL execution layer. Spark Structured Streaming (ADR-002) is superseded.
  supersedes: 2026-05-09-002-streaming-engine-choice
-->
{% raw %}

# ADR 2026-05-10-001: Nanofab runtime is Rust

## Context

The five design specs landed on 2026-05-10 under `docs/superpowers/specs/` re-aim the product around a single named technical artifact — **nanofab** — and explicitly redefine the engine question. The runtime spec [`2026-05-10-nanofab-runtime-design.md`](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md) is unambiguous in §1.2 ("Two ideas inherited from the existing material"):

> "**Skip the SQL execution layer.** The processing engine is purpose-built Rust, not a general SQL engine. No JVM, no shuffle framework, no engine-startup tax."

§1.3 ("What this design extends") frames *why* the engine question changes shape under nanofab: the existing `docs/products/sql-to-realtime.md` skill converts "one SQL job into one stream-processor architecture." The new design extends that to convert "a customer's whole portfolio of fact streams and SQL ETLs into one distributed Rust application — Nanofab. Every dim/ADS table in the warehouse is maintained by a node in a single shared DAG, sharing event ingress and state infrastructure rather than each running its own pipeline." Once "one engine, one customer, many DAG nodes hot-swapped independently" is the unit of deployment, the engine candidate set narrows to *runtimes that can host* a single distributed application, not *engines that can run* a SQL job.

§2 ("Decisions summary") commits the seven architectural axes that fall out of this re-framing — all of them assume Rust as the host language and a stateless supervisor + `cdylib` plugin shape:

| Axis | Choice |
|---|---|
| Deployment unit | Single supervisor binary + per-node `cdylib` plugins |
| State storage | External distributed KV (TiKV / FoundationDB), supervisors stateless |
| Sharding | Hash by primary key; small reference tables replicated to every shard |
| Event transport | Kafka/Redpanda for ingress and cross-shard; lock-free in-process channels intra-DAG |
| Hot swap | Per-node shadow + canary; whole-DAG blue/green for breaking changes |
| Correctness | Event-time + at-least-once + idempotent writes |
| Output surface | gRPC point-lookup over KV + CDC sink to Apache Iceberg |

Three of these (cdylib hot swap, lock-free intra-DAG channels, in-process IPC between co-located nodes) are not achievable on a JVM-based engine without giving up the property that motivated them. A Spark executor cannot hot-swap one node of a streaming job without restarting the job; Spark's micro-batch model cannot offer lock-free intra-DAG channels; Spark's shuffle framework is the wrong substrate for "explicit cross-shard hop via internal Kafka topic" (§5.3).

The prior loop's ADR-002 ([2026-05-09-002-streaming-engine-choice.md](2026-05-09-002-streaming-engine-choice.md)) adopted Spark Structured Streaming as the engine for the first `fact_sign_up.parquet` demo, with a recorded re-open trigger: "a future demo with constraints Spark cannot meet." The 2026-05-10 pivot is that trigger — not because a new latency or join requirement surfaced, but because the *shape* of the product changed from "one SQL job per pipeline" to "one distributed application per customer," which is a constraint Spark structurally cannot meet.

The right comparison for the new direction is **not** to Spark or Flink — those answer "which engine runs my SQL?" — but to the prototype playbook in `docs/products/prompts.md`, which records hand-tuned Rust patterns measured at **3.6× faster warm-path than DuckDB** on a representative ETL workload (521 ms warm vs 1,872 ms DuckDB baseline, 300K rows, 5 runs). The patterns that delivered that gap — sweep-line join replacing hash join, FxHash + Struct-of-Arrays layout, parallel mmap IPC reads, per-day delta computation — are exactly the patterns this design productizes at scale. ADR-001 records that this design productizes the prototype playbook, it does not pick a new general engine.

## Decision

The nanofab runtime is a **purpose-built distributed Rust application** as specified in [`docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md`](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md). It is not a general SQL engine, not a JVM-hosted streaming framework, and not a thin wrapper over an existing engine.

Concretely:

1. **Host language is Rust.** Supervisor binary, plugin ABI, state layer, transport layer, and CDC sink are all Rust crates per runtime spec §4. No JVM in the data path.
2. **No general SQL execution layer.** Per §1.2 and §3.3 ("What Nanofab is NOT"): "Not a general stream-processing platform — DAGs come only from the resink.ai training pipeline; there is no user-facing DSL. Not a SQL engine. Not a database — KV is rented from TiKV / FoundationDB." Analytical SQL is delegated to Trino/DuckDB/Spark reading the Iceberg sink.
3. **Single distributed application per customer.** Per §1.3 and §1.4: one nanofab per customer, a versioned DAG of small Rust processors plus a runtime that schedules, hot-swaps, and observes them. Whole-portfolio, not per-job.
4. **The seven §2 decisions are accepted as bound.** Subsequent ADRs may refine specific axes (e.g., TiKV vs FoundationDB, Kafka vs Redpanda) but cannot revisit "Rust supervisor + cdylib plugins" without a new superseding ADR.

ADR-002 is superseded effective this loop. The archival edit lands in parallel with this ADR ([2026-05-09-002-streaming-engine-choice.md](2026-05-09-002-streaming-engine-choice.md) status flips to `archived`, `superseded_by` field points at this ADR).

## Alternatives considered

- **A: Keep Spark Structured Streaming (extend ADR-002).** Rejected. Spark's strengths — broad team familiarity, ecosystem integration with Delta/Iceberg, well-trodden k8s deployment path (the explicit "positive consequences" in ADR-002) — cannot justify retention once the system must be a **single distributed Rust application** with hot-swap per node, lock-free intra-DAG channels, and `cdylib` plugins. Per-node shadow+canary (§6.2) and lock-free in-process IPC (§4.5) are not Spark-shaped. The cost of retro-fitting Spark to host arbitrary Rust `cdylib`s and per-node hot-swap is strictly larger than building the supervisor directly.
- **B: Apache Flink.** Rejected for the same structural reason as A, plus Flink has a higher operational footprint than Spark without compensating in any axis the new design needs (Flink's exactly-once is moot under the §7.1 idempotent-write contract; Flink's sub-second latency is already exceeded by lock-free in-process channels).
- **C: A general Rust streaming engine (Arroyo, Materialize, RisingWave).** Rejected. These engines answer "run my SQL with low latency"; nanofab answers "run *the* customer's whole DAG as one distributed application." The decision criterion is not latency — these engines are fast — it is *plugin ABI, hot-swap, and DAG-as-data*. Adopting any of them would mean shipping their DSL surface to customers, which §3.3 explicitly forbids ("DAGs come only from the resink.ai training pipeline; there is no user-facing DSL"). All three would also require either forking the engine or shipping our own scheduling layer on top, which is what building a purpose-built supervisor accomplishes directly.
- **D: Wrap the prototype playbook patterns as a library, embed in Spark.** Rejected. The 3.6× gap in `docs/products/prompts.md` came from controlling memory layout (SoA), IPC (mmap), and the join algorithm end-to-end — exactly the controls a JVM executor takes away. The patterns are not portable; the architecture that uses them is.

## Consequences

- Positive:
  - The seven decisions in runtime spec §2 are board-ratified; team OKRs reference a single source of truth instead of negotiating engine choice in every retro.
  - resink-core can plan its first build slice against a stable target (the supervisor binary + node ABI), not against a streaming engine whose API the team would re-litigate.
  - The performance ceiling is set by the prototype playbook (3.6× warm vs DuckDB), not by the JVM. Future optimization work is on the productized side of patterns already measured.
  - Hot-swap modes (per-node shadow/canary; per-DAG blue/green) are specified in §6 — the SRE runbook target reshapes from "Spark job restart procedure" to "the supervisor's seven failure modes" (§7).
- Negative / costs:
  - resink-core's first build slice as planned in loop 2026-05-10-2227-001 (a Spark job producing `dim_user_signup`) is cancelled. Replacement build slice is owned by resink-core's 2026-05-10 OKR.
  - DE's deferred "shared streaming primitive contract" (KR2.1 in 2026-05-09) is also cancelled in its Spark-flavored form; it re-shapes into a **Kafka ingress contract** for nanofab, owned by DE this loop (see DE's 2026-05-10 OKR, O2).
  - Team familiarity drops: no one on the team has shipped a distributed Rust application of this scope before. Mitigation: the prototype playbook (`docs/products/prompts.md` + `docs/products/sql-to-realtime.md`) is the on-ramp; Sim Farm is the validation gate (§8) so risky work is checked before it reaches production.
  - No off-the-shelf observability dashboards. Mitigation: §8.5 specifies Prometheus + OpenTelemetry surfaces from day one.
- Follow-ups required:
  - DE lands the Kafka ingress contract (see DE 2026-05-10 OKR O2) referenced from this ADR and grounded in runtime spec §3 + §7.
  - resink-core lands a build-slice ADR for the first runtime component (likely the supervisor skeleton + node ABI crate) under its 2026-05-10 OKR.
  - SRE's runbook target moves to the supervisor's seven failure modes documented in §7.
  - Sub-project ownership (Runtime, Training Pipeline, Sim Farm, Serving & Deployment, Product UX) is recorded separately in [ADR 2026-05-10-002](2026-05-10-002-nanofab-sub-project-decomposition.md).

## Links

- Runtime spec (grounds the decision): [docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md](../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md)
- Prototype playbook (performance grounding): [docs/products/prompts.md](../../docs/products/prompts.md), [docs/products/sql-to-realtime.md](../../docs/products/sql-to-realtime.md), [docs/products/overview.md](../../docs/products/overview.md)
- Superseded ADR: [2026-05-09-002-streaming-engine-choice](2026-05-09-002-streaming-engine-choice.md)
- Companion ADR (sub-project ownership): [2026-05-10-002-nanofab-sub-project-decomposition](2026-05-10-002-nanofab-sub-project-decomposition.md)
- Kafka ingress contract (downstream artifact this loop): [teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md](../../teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md)
- Parent CEO brief: [board/okrs/2026-05-10-2227-002-ceo-brief.md](../okrs/2026-05-10-ceo-brief.md)
- Triggering retro (carryover): [board/retros/2026-05-10-2227-001-ceo-retro.md](../retros/2026-05-10-2227-001-ceo-retro.md)
{% endraw %}
