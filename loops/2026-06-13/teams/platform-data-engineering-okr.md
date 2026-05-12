---
layout: default
title: platform-data-engineering OKR — 2026-06-13
date: 2026-06-13
status: active
type: okr
loop: 2026-06-13
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/data-engineering
  date: 2026-06-13
  status: active
  loop: 2026-06-13
  links: parent: board/okrs/2026-06-13-ceo-brief.md
-->
# Data Engineering OKR — 2026-06-13

## Context

DE returns to a single-objective build loop after two consecutive paused loops. The headline is canonicalization of the schema-JSON shape (`{key_columns, payload_columns}`) under DE's `conventions/` tree, per the [in-loop request from sim-farm at 2026-06-06](../requests/2026-06-06-001-schema-json-shape-ack.md) — DE accepted in-loop with deadline 2026-06-13, and this is the **first cross-team request lifecycle to fully complete** (open at 06-06; accepted-deferred → fulfilled at 06-13). The new convention is also the **second contract/convention to honor [ADR-2026-05-16-003](../../../../board/decisions/2026-05-16-003-contract-environment-verification.md)'s `Verified-against-environment` discipline** (after sim-farm's 2026-06-06 verdict-contract update), establishing the pattern for DE-owned `conventions/` files going forward. Sister convention to [`conventions/duckdb.md`](../conventions/duckdb.md) in shape and depth.

## Objectives

### O1: Canonicalize the schema-JSON spec under DE's `conventions/` tree

- source: cross-team-request
- links.source: [../requests/2026-06-06-001-schema-json-shape-ack.md](../requests/2026-06-06-001-schema-json-shape-ack.md)

Why it matters: Sim-farm's engine 0.3.0 (loop 2026-06-06) inlined the per-pair schema-JSON shape into its [verdict contract](../../../application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md) under "Schema-aware columns (0.3.0+)" because DE was paused that loop. Canonicalizing the shape under DE makes ownership consistent with DE owning the Kafka ingress contract, the in-memory event-source contract, and the duckdb convention — DE owns the data-shape vocabulary; sim-farm owns the engine that consumes it. The shape itself is intentionally minimal (`key_columns` + `payload_columns` only) so resink-core's orchestrator can adopt it without first negotiating optional fields; follow-up cross-team requests carry any additional fields when a consumer surfaces specific need. Closing this lifecycle (open → accepted-deferred → fulfilled) also produces the first end-to-end demonstration of the request-tracking pattern that ADR-2026-06-06-002 (paused-team request acceptance, ratifying this loop) canonicalizes.

**Key results**

- KR1.1: New file at [`teams/platform/data-engineering/conventions/dim-schema-json.md`](../conventions/dim-schema-json.md) with `type: convention` (canonical, not provisional — the `convention` type was admitted at 2026-05-30; no `type: rfc` workaround needed). Sister to [`conventions/duckdb.md`](../conventions/duckdb.md) in shape and depth. Sections in order: (a) one-paragraph context naming why DE owns the shape (sister to duckdb.md ownership rationale: DE owns the data-shape vocabulary); (b) the JSON shape `{"key_columns": [string], "payload_columns": [string]}` with strict schema (`key_columns` length ≥ 1; `payload_columns` may be empty; additional keys ignored per additivity discipline); (c) two worked examples — `dim_user_schema.json` (key `user_id`/`valid_from`, payload `email`/`country`/`valid_to`/`is_current`) and `dim_account_schema.json` (key `account_id`/`valid_from`, payload `account_type`/`status`/`valid_to`/`is_current`), copied byte-stably from sim-farm verdict contract §"Schema-aware columns (0.3.0+)"; (d) named consumers (sim-farm engine 0.3.0+ via `--schema-ref` flag and `run_diff_multi` 4-tuple, resink-core orchestrator); (e) `Verified-against-environment` subsection per ADR-2026-05-16-003.

- KR1.2: The originating request file [`requests/2026-06-06-001-schema-json-shape-ack.md`](../requests/2026-06-06-001-schema-json-shape-ack.md) flips `status: open → status: fulfilled`; `links.fulfilled_by` field populated with the path of this OKR (`teams/platform/data-engineering/okrs/2026-06-13-team-okr.md`). If ADR-2026-06-06-002 mandated edits from O2 KR2.3 land mid-loop introducing the new optional `deferred_to_loop: <YYYY-MM-DD>` request frontmatter field, populate it with `2026-06-13` (otherwise add post-O2-KR2.3, mid-loop bookkeeping). This produces the first end-to-end fulfilled request lifecycle on the org-os tree.

- KR1.3 (opportunistic, XS): Flip the `<verified-against-rust-impl: pending>` tag in [`contracts/2026-05-10-kafka-ingress.md`](../contracts/2026-05-10-kafka-ingress.md) §2.1 to `verified-2026-06-13-by-resink-core` (or the equivalent dated form). Resink-core confirmed byte-stability for Example A (`user_id="u-001"` → `0x571e3e04781b0ff5`) at 2026-05-23; this single-line edit lands only if §2.1 is touched naturally during the gitbook publishing scope review (board's O1) — does NOT block KR1.1/KR1.2/KR1.4. If §2.1 is not touched this loop, the tag carries forward as before.

- KR1.4: Cross-team coordination handshake to sim-farm. Sim-farm is paused this loop; the back-reference in their verdict contract is mechanical and lands at 2026-06-20. DE files a one-line "convention is ready for back-reference" note in DE's exec summary cross-team section (or, if the rituals admit it, a note appended to sim-farm's `status.md`) so sim-farm picks up the back-reference cleanly when they next plan. Note names the canonical path (`teams/platform/data-engineering/conventions/dim-schema-json.md`) and proposes the one-line replacement for the sim-farm contract's "Schema-JSON shape canonical-ownership note" paragraph.

- KR1.5: Tenant-isolation invariant holds for this objective. All writes are under tenant paths only: `teams/platform/data-engineering/conventions/dim-schema-json.md` (new), `teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md` (frontmatter flip), optionally `teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md` (KR1.3 single-line tag flip). NO writes to `org-os/` from this objective. Per-objective dry-run after the convention lands; cumulative final dry-run before exec summary.

**Tasks**

- [ ] Author `teams/platform/data-engineering/conventions/dim-schema-json.md` per the standing-answer shape (frontmatter `type: convention`, `status: active`, `owner: teams/platform/data-engineering`, `date: 2026-06-13`; five sections per KR1.1; two worked examples copied byte-stably from sim-farm verdict contract) — owner: teams/platform/data-engineering.
- [ ] Populate the `Verified-against-environment` subsection naming: verification date 2026-06-13; the sim-farm engine version exercised (0.3.0); the resink-core orchestrator integration point; runtime deps (`duckdb`, `pyarrow`, `pytz` per the duckdb convention); OS surface (OS-agnostic); failure modes (`ValueError` at schema-ref-load on missing/empty `key_columns`, DuckDB binding error on column-not-in-parquet) — owner: teams/platform/data-engineering.
- [ ] Flip the originating request to `status: fulfilled`; populate `links.fulfilled_by`; add `deferred_to_loop: 2026-06-13` if ADR-2026-06-06-002's request-template edit has landed mid-loop — owner: teams/platform/data-engineering.
- [ ] (Opportunistic) Flip `<verified-against-rust-impl: pending>` tag in Kafka §2.1 to `verified-2026-06-13-by-resink-core` if the file gets touched naturally during board's gitbook scope review — owner: teams/platform/data-engineering.
- [ ] File the cross-team coordination note for sim-farm's next-loop back-reference (in DE's exec summary cross-team section; proposed one-line replacement text included) — owner: teams/platform/data-engineering.
- [ ] Per-objective tenant-isolation dry-run; cumulative final sweep before exec summary — owner: teams/platform/data-engineering.

## Cross-team asks

- **From `teams/application/sim-farm` (next loop, mechanical):** sim-farm's `2026-05-16-mvp-loop-verdict.md` contract gets a one-line back-reference replacing the existing "Schema-JSON shape canonical-ownership note" paragraph at the end of §"Schema-aware columns (0.3.0+)". The replacement points at `teams/platform/data-engineering/conventions/dim-schema-json.md` as the canonical owner; the JSON-shape table and worked examples can stay (sim-farm contract retains them as engine-side documentation) or be replaced by a link. Sim-farm picks the depth. **Sim-farm is paused this loop; this is a 2026-06-20 ask, not a 2026-06-13 ask** — no in-loop coordination needed. DE's KR1.4 files the note that signals readiness.

- **From `teams/application/resink-core` (informational, already satisfied):** confirmation that the minimal `{key_columns, payload_columns}` shape suffices for the orchestrator's usage of the per-pair schema-ref. Resink-core consumed engine 0.3.0 against this shape during their 2026-06-06 build phase (`make mvp-loop` GREEN with both `dim_user_schema.json` and `dim_account_schema.json`); the shape proved sufficient there. If resink-core surfaces a need for additional fields (`is_scd2`, `compaction_strategy`, `column_types`, `tolerances`) once the convention is canonicalized, the additivity discipline named in KR1.1 (additional keys ignored by current parsers) admits them in a future minor revision via a follow-up cross-team request.

- **From `board` (opportunistic):** if board's O1 (gitbook publishing) touches `contracts/2026-05-10-kafka-ingress.md` §2.1 during the in-scope-filter review (e.g., normalizing relative links for the Jekyll tree), DE's KR1.3 piggybacks the `<verified-against-rust-impl>` tag flip into the same touch. Strictly opportunistic — if board doesn't touch §2.1 this loop, the tag carries to a future loop where §2.1 is naturally touched.

## Risks

- **The schema-JSON shape might prove insufficient for resink-core's orchestrator usage** once the convention is canonicalized — e.g., resink-core might want `is_scd2: bool`, `compaction_strategy: enum`, or per-column type hints (`column_types: {col: type}`). **The minimal shape is intentional this loop:** the brief (Risk #5) names this risk explicitly with the intended mitigation — the shape is `{key_columns, payload_columns}` only because that's what engine 0.3.0 consumed cleanly at 2026-06-06; additional fields go through the follow-up cross-team request mechanism (per ADR-2026-06-06-002 paused-team request acceptance + ADR-2026-06-06-001 provisional-and-migrate playbook for any field that needs to land provisional first). The convention's KR1.1 additivity discipline ("additional keys ignored") admits future fields without breaking the parser. If resink-core surfaces pressure mid-loop, DE files a v2 same-loop only if the additive field is trivially derivable; otherwise the v2 lands at 2026-06-20 via the standard request mechanism.

## Out of scope this loop

- **Next-slice event-source contracts** (webhook source, S3-poll source, CDC-from-snapshot source, multi-tenant + multi-shard runtime addenda) — deferred-not-dropped per the long-standing posture; DE authors only when a consumer surfaces specific pressure. None has surfaced this loop.
- **Authoring schema-JSON shape v2 with optional fields** (`is_scd2`, `compaction_strategy`, `column_types`, `tolerances`) — out of scope this loop; minimal shape is intentional. Future cross-team request from resink-core (or any consumer) triggers v2.
- **Sim-farm's verdict-contract back-reference** — sim-farm's responsibility next loop (2026-06-20); sim-farm is paused this loop. DE's KR1.4 only files the readiness note; sim-farm makes the actual one-line edit.
- **Migrating `conventions/duckdb.md` from `type: rfc` to `type: convention`** — not in this loop's scope per status.md ("`conventions/duckdb.md` remains at `type: rfc` (intentional — not in this loop's migration scope)"); the `convention` type was admitted at 2026-05-30 but the duckdb migration carries until a loop with explicit migration scope. The new `dim-schema-json.md` lands directly as `type: convention` (no workaround needed).
- **`<verified-against-rust-impl>` tag flip on Kafka §2.1 as a forced edit** — opportunistic only (KR1.3); does not block KR1.1/KR1.2/KR1.4. If §2.1 is not touched naturally this loop, the tag carries.
- **Authoring DE's own exec summary content** — happens in the exec-summary ritual after build phase, not in this OKR.
