---
layout: default
title: platform-data-engineering Exec Summary — 2026-05-10-2227-002
date: 2026-05-10
status: active
type: exec-summary
loop: 2026-05-10-2227-002
owner: teams/platform/data-engineering
grand_parent: Loops
parent: Loop 2026-05-10-2227-002
nav_order: 17
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/data-engineering
  date: 2026-05-10
  status: active
  loop: 2026-05-10-2227-002
  links: parent: teams/platform/data-engineering/okrs/2026-05-10-2227-002-team-okr.md
-->
{% raw %}

# Data Engineering Exec Summary — 2026-05-10

## What we shipped

### O1: Ratify the runtime pivot via an ADR that supersedes ADR-002
- New ADR landed at [`../../../../board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md`](../../../../board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md) with `status: active` and `supersedes: 2026-05-09-002-streaming-engine-choice` in frontmatter; grounded in runtime spec §1.2, §1.3, and §2; alternatives enumerate Spark, Flink, general Rust streaming engines, and wrap-prototype-in-Spark with rejection rationale for each. (KR1.1)
- ADR-002 archived in place at [`../../../../board/decisions/2026-05-09-002-streaming-engine-choice.md`](../../../../board/decisions/2026-05-09-002-streaming-engine-choice.md): frontmatter flipped `active` → `archived` with `superseded_by: 2026-05-10-001-nanofab-runtime-is-rust`; body preserved verbatim for history. (KR1.2)

### O2: Kafka ingress contract (replaces the deferred primitive contract)
- New stable hand-off directory created at [`../contracts/`](../contracts/).
- Kafka ingress contract landed at [`../contracts/2026-05-10-kafka-ingress.md`](../contracts/2026-05-10-kafka-ingress.md) with `status: active`, written in named-hand-off-sections format (P3 from the 2026-05-09 retro). Eight sections cover topic naming, partitioning (xxHash64 over primary key, same hash as KV state layer), consumer-group naming, DLQ/quarantine topics, per-tenant SASL/SCRAM-SHA-512, broker bootstrap, manual offset commits gated on terminal-node KV success, and tenant-isolation invariant. Grounded in runtime spec §3, §4.5, §5.3, §5.7, §6.3, §7, §8. (KR2.1)
- ADR-001 and the Kafka ingress contract cross-reference each other: ADR-001 links the contract from its downstream-artifact links entry; the contract links back from its Links section. (KR2.2)

## What we didn't ship and why

Nothing was blocked or deferred. All six tasks in the OKR (O1 T1–T3, O2 T1–T3) shipped this loop. The prior-loop carryover ("shared streaming primitive contract for the first demo") was intentionally replaced by the Kafka ingress contract per the runtime pivot, not deferred again.

## Surprises

- **`contract` is not in the `org-os/conventions.md` type enum.** The Kafka ingress contract was filed with `type: rfc` (proposer = DE EM) because the conventions enum permits only `charter | okr | exec-summary | retro | adr | rfc | status | agent-spec`. An HTML-comment note in the contract documents the choice. Not blocking — `rfc` is a defensible fit — but conventions need an evolution proposal.
- **Named-hand-off-sections format settled cleanly on first use.** P3 from the 2026-05-09 retro was an untested pattern; DE expected to discover edge cases. Labeling each section with its consumer up front made the structure obvious, and no section required splitting.
- **ADR-002 archival was a single-line frontmatter edit.** We had budgeted risk for a body-rewrite to keep history coherent under the supersession; preserving the body verbatim turned out to be the correct call and trivial to land.
- **Cross-shard internal topics had a natural home in the contract.** Originally scoped as a possible follow-up section, the `nanofab.internal.<dag_version>.<edge>` naming convention dropped in cleanly alongside ingress, avoiding a second hand-off document.

## Asks

- **From the board:** approval of [`../../../../board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md`](../../../../board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md). It landed `status: active` for the consolidation step to surface; CEO sign-off is the unblocker for resink-core's re-aimed OKR (CEO brief O2).
- **From the board + DE (next-loop retro):** propose an `org-os` evolution to add a `contract` type (or a single unified hand-off-document type) to `org-os/conventions.md`. New ask surfaced during build; not blocking this loop.
- **From `teams/application/resink-core`:** consumer-side constraints on the Kafka ingress surface beyond what runtime spec §3 captures (minimum partition counts, retention windows for replay, quarantine-topic handling). Same surface-constraints-early pattern that made ADR-002 defensible.
- **From `teams/application/sim-farm`:** confirmation that the supervisor `--mode=sim` ingress shape (spec §8.4 in-memory event source) matches the contract's offset-commit and idempotency expectations. Flagging divergence now prevents a re-draft next loop.

## Metrics

- **KR1.1** (new ADR exists, `status: active`, supersedes ADR-002 in frontmatter, grounded in runtime spec §1.2 + §2): **met** — [`../../../../board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md`](../../../../board/decisions/2026-05-10-001-nanofab-runtime-is-rust.md).
- **KR1.2** (ADR-002 flipped to `archived` with `superseded_by`): **met** — [`../../../../board/decisions/2026-05-09-002-streaming-engine-choice.md`](../../../../board/decisions/2026-05-09-002-streaming-engine-choice.md).
- **KR2.1** (Kafka ingress contract exists, `status: active`, all seven named hand-off sections present with consumer labels): **met** — eight sections landed (seven required + tenant-isolation closing invariant). [`../contracts/2026-05-10-kafka-ingress.md`](../contracts/2026-05-10-kafka-ingress.md).
- **KR2.2** (new ADR and Kafka ingress contract cross-reference each other; contract grounded in runtime spec §3 with explicit citations): **met** — bidirectional links in place; contract cites §3, §4.5, §5.3, §5.7, §6.3, §7, §8.
{% endraw %}
