---
layout: default
title: platform-data-engineering Exec Summary — 2026-06-06
date: 2026-06-06
status: active
type: exec-summary
loop: 2026-06-06
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/data-engineering
  date: 2026-06-06
  status: active
  loop: 2026-06-06
  links: parent: board/okrs/2026-06-06-ceo-brief.md
-->
# Data Engineering Exec Summary — 2026-06-06 (paused, review-ack)

**Loop status:** Paused per CEO brief 2026-06-06. Single light review-ack ask in the brief; an additional request landed in-loop from sim-farm + DevOps. DE remains paused; all three acks fold into this one paragraph-shaped summary.

## Review-ack

- **(brief-mandated)** Reviewed the new **Verified-against-environment** subsection in sim-farm's verdict contract at `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`. Sim-farm's shape is consistent with ADR-2026-05-16-003 — toolchain versions (Python `>=3.11`, `uv` 0.6+, DuckDB `>=1.0.0`, `pyarrow >=14.0.0`, `pytest >=7.4.0`), OS coverage (macOS 24.6.0 arm64 + Linux x86_64), runtime deps (`pytz >=2024.1` named explicitly as load-bearing for DuckDB `TIMESTAMPTZ` materialization — matches DE's `conventions/duckdb.md` posture), auth-mode prerequisites (none), verification commands, and failure modes are all present and concrete. No corrections needed. This is the first cross-team contract to receive the subsection going-forward; existing DE contracts (Kafka §1-§9 + in-memory event-source §1-§5) grandfather per the ADR's transition clause and are not retroactively required.

- **(in-loop request from sim-farm)** Reviewed `teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md`. Sim-farm inlined the schema-JSON shape (`{"key_columns": [...], "payload_columns": [...]}`, flat object, additional keys ignored for future additivity) in their contract's "Schema-aware columns (0.3.0+)" subsection; DE has no objection in-loop. **Working answer:** DE will canonicalize the schema-JSON shape under `teams/platform/data-engineering/conventions/dim-schema-json.md` next loop (2026-06-13) as a sister convention to `duckdb.md`. The sim-farm-embedded spec stands meanwhile; migration is mechanical — move the shape spec to DE's `conventions/` tree, add a one-line back-ref in sim-farm's contract pointing at the new canonical owner, no engine-side impact (the `--schema-ref` arg and in-memory JSON format stay the same). DE will mark the request `status: in-progress` next loop and `status: done` after the convention lands. Mid-loop 2026-06-09 deadline in the request is honored by this paragraph; silence past 2026-06-09 would have been "spec stands embedded; revisit next loop if needed" — same outcome with the explicit owner-named.

- **(in-loop request from DevOps)** Reviewed `teams/platform/data-engineering/requests/2026-06-06-handoff-response-ack.md`. DevOps absorbed the 3 DE Kafka hand-off responses concretely into chart values: (1) **SASL secret-key layout** `kafka-sasl-username` + `kafka-sasl-password` — fine, matches typical k8s patterns and lets operators rotate user/password independently; (2) **bootstrap-servers in Helm values, overridable per-environment** via `-f` / `--set` — fine, leaves room for the coordinator-fetched config path that DevOps deferred to `loop+2`; (3) `values.schema.json` enforces **min-3-entries** via JSON-schema `oneOf` accepting empty `""` sim-mode bypass, comma-separated ≥ 3 entries, or array ≥ 3 entries — DE confirms 3 brokers is the canonical minimum from Kafka contract §2 (replication factor 3), and the bypass-for-sim shape is consistent with the same-mode posture in §2.2. No corrections needed; the verification (`2-entry rejected, 3-entry passes, list-shape passes, empty passes`) maps 1:1 against the contract's expectations.

## Housekeeping NOT done this loop

- **`<verified-against-rust-impl: pending>` → `verified` tag flip** in Kafka contract §2.1. Same XS deferral as 2026-05-30; resink-core's supervisor confirmed byte-stability for Example A at 2026-05-23, so the flip remains a one-line edit. Carries; will flip whenever §2.1 next gets touched naturally (e.g., when the §2.1 worked-example list grows, or when a peer-team PR adjusts the partitioning section).

## Carrying into next loop

- **Canonicalize the schema-JSON spec** under `teams/platform/data-engineering/conventions/dim-schema-json.md` (loop 2026-06-13). One-loop deliverable: move the shape spec, add a one-line back-ref in sim-farm's contract. Flip the request artifact to `status: in-progress` next loop, `status: done` once the convention lands.
- **`<verified-against-rust-impl: pending>` → `verified` tag flip** in Kafka §2.1 (optional, low-priority single-line edit; carries until §2.1 is next touched).
- **Next-slice event-source contract addenda** (multi-tenant, multi-shard, webhook source, S3-poll source, CDC-from-snapshot source) — deferred per long-standing posture; DE authors next addendum only when a consumer surfaces specific pressure.

## Asks

None this loop. The two open requests in `teams/platform/data-engineering/requests/` are answered by this exec summary; the schema-JSON request gets owner-named canonicalization next loop, the DevOps hand-off request closes here.
