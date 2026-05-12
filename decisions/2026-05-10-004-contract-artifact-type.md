---
layout: default
title: "ADR 2026-05-10-004: contract artifact type"
parent: "Decisions (ADRs)"
render_with_liquid: false
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
  decision: Add `contract` to the `type` enum in `org-os/conventions.md` with required `consumers:` and `producers:` fields; migrate existing rfc-typed contracts to `type: contract` same loop.
-->
# ADR 2026-05-10-004: Contract (hand-off-document) artifact type

## Context

Surfaced by DE during build of loop 2026-05-10-2227-002. The Kafka ingress contract at `teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md` is a hand-off document with named consumer sections (resink-core supervisor, sim-farm, SRE, serving/deployment). The current `type` enum in `org-os/conventions.md` does not include `contract` (or any hand-off-document type), so the document was filed as `type: rfc` with `proposer: teams/platform/data-engineering (EM)` as a workaround. Across loops 2026-05-10-2227-002 and 2026-05-16 the workaround compounded: the DE in-memory-event-source contract (2026-05-16) and the sim-farm MVP-loop-verdict contract (2026-05-16) both filed as `type: rfc` with in-body notes explaining the workaround. Three loops of contracts-filed-as-RFC ends here. Hand-off documents have a recognizable shape — named consumers, named producers, normative sections that downstream teams rely on — that deserves its own type rather than borrowing `rfc` (which is semantically a proposal, not a decision).

## Decision

Two changes to `org-os/conventions.md`:

1. **Add `contract` to the `type` enum.** The required-for-every-artifact frontmatter shape gains `contract` alongside `charter | okr | exec-summary | retro | adr | rfc | status | agent-spec | request | team-proposals`.
2. **Add a row to the "Additional fields per type" table:**

   | `contract` | `consumers: [<list of team paths>]`, `producers: [<team path>]` |

   `consumers` is the list of team directory paths that consume the contract (e.g., `teams/application/resink-core`, `teams/application/sim-farm`). `producers` is the single producing team's directory path. Optional `links.parent` may point to a team OKR that originated the contract.

The `status` field on a `contract` follows the standard publication-state enum (`draft | active | archived`) — a contract is `active` when its producing team commits to honoring it; consumers may begin to depend on it at that point.

Same loop the ADR ratifies (2026-05-23), the three existing `rfc`-typed contract files migrate to `type: contract`:

- `teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md`
- `teams/platform/data-engineering/contracts/2026-05-16-in-memory-event-source.md`
- `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`

Each migration drops the `proposer:` field, adds `consumers:` and `producers:`, and removes the in-body workaround note explaining the type choice.

## Alternatives considered

- **A: Keep filing contracts as `rfc`.** Rejected — RFCs are proposals; contracts are decisions. The semantic mismatch surfaced across three loops and the workaround note in every file is itself a discipline failure indicator.
- **B: Introduce a broader `handoff` type covering contracts, interfaces, protocols.** Rejected for now — the named hand-off-sections convention (P3 from 2026-05-09 retro) is the *structure* for any hand-off document; the type field only needs to name the artifact category, and `contract` is the precise word. If interfaces and protocols later diverge from contracts in shape, a separate type can be added.
- **C: Extend `rfc` semantics to cover hand-off documents.** Rejected because RFCs in this org-os have a specific lifecycle (proposer → review → decision); contracts do not have that lifecycle (they are produced by a team and consumed by named others).
- **D: Use `agent-spec` as a parallel pattern.** Considered; agent-spec is for agents specifically, not for cross-team data/interface contracts. Different audience, different required fields.

## Consequences

- **Positive:** Contracts gain a first-class home in the artifact taxonomy. The `consumers` field makes cross-team dependencies mechanically discoverable (a grep over `consumers:` reveals every team that depends on a given producer). The workaround note in three files drops. Future contracts (Helm chart shape, supervisor `--mode=sim` interface, sim-farm verdict format, runbook scope) file directly under the new type.
- **Negative / costs:** A one-time migration on three files. Future tooling/lint (Phase 2) needs to learn the new type. Conventions documentation grows by one row.
- **Follow-ups required:** Migrate the three existing contract files this loop (Step 3 of the ratification work). Future contracts file under `teams/<layer>/<team>/contracts/`. Pair with the named-hand-off-sections convention (P3 from 2026-05-09 retro) — the structure of a contract is the named-sections pattern; the type is what makes it a contract.

## Links

- Triggering retro: [board/retros/2026-05-10-2227-002-ceo-retro.md](../retros/2026-05-10-2227-002-ceo-retro.md) (P2)
- Surfacing artifact: [teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md](../../teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md)
- Conventions updated: [org-os/conventions.md](../../org-os/conventions.md) (`type` enum + additional-fields-per-type row)
- Migrated contracts (this loop): [2026-05-10-kafka-ingress](../../teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md), [2026-05-16-in-memory-event-source](../../teams/platform/data-engineering/contracts/2026-05-16-in-memory-event-source.md), [2026-05-16-mvp-loop-verdict](../../teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md)
- Predecessor pattern: P3 from [board/retros/2026-05-10-2227-001-ceo-retro.md](../retros/2026-05-10-2227-001-ceo-retro.md)
- Related ADRs: [2026-05-09-005-carryover-load-in-brief](2026-05-09-005-carryover-load-in-brief.md), [2026-05-09-006-org-os-change-routing](2026-05-09-006-org-os-change-routing.md)
