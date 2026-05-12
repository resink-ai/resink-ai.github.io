---
layout: default
title: platform-data-engineering Exec Summary — 2026-05-11-1113
date: 2026-05-11
status: active
type: exec-summary
loop: 2026-05-11-1113
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/data-engineering
  date: 2026-05-11
  status: active
  loop: 2026-05-11-1113
  links: parent: teams/platform/data-engineering/okrs/2026-05-11-1113-team-okr.md
-->
# Data Engineering Exec Summary — 2026-05-23

## Headline

Three deliverables shipped — `fact_account_open` schema fragment (Kafka §9), `xxHash64(seed=0)` ratification with byte-stable worked examples (Kafka §2.1), and the `pytz` DuckDB conventions doc establishing the team's first `conventions/` tree. Board migrated both DE contracts from `type: rfc → type: contract` cleanly, closing three loops (2026-05-10, 2026-05-16, 2026-05-23) of the `type: rfc` workaround.

## What we shipped

### O1 (source: ceo-brief) — DE's consolidation tail

- **`fact_account_open` schema fragment (KR1.1) — additive §9 in the Kafka ingress contract.** Six sub-sections (§9.1 topic naming `nanofab.ingress.<tenant>.fact_account_open`; §9.2 single PK `account_id` + back-reference to §2.1; §9.3 full `FactAccountOpenEvent` Rust shape + JSON worked example for `account_id=42`; §9.4 cross-dim FK note `account.user_id → dim_user.user_id`; §9.5 production parity restatement against §2 + §4 + §7; §9.6 cross-link to in-memory contract). Paired §5.1 cross-link added to the in-memory event-source contract pointing at Kafka §9 as the single source of truth (parquet path `fixtures/fact_account_open.parquet`, `op="insert"`-only callout, no schema duplication). Addendum-to-both-contracts path chosen per KR1.1, not a third standalone file. `verified-at: master @ contracts/2026-05-10-kafka-ingress.md §9 + contracts/2026-05-16-in-memory-event-source.md §5.1`.
- **`xxHash64(seed=0)` ratification (KR1.2) — Kafka §2.1 `Ratification (2026-05-23)`.** Two worked examples computed against Python `xxhash.xxh64(bytes, seed=0).intdigest()`: (A) `user_id="u-001"` → bytes `0x75 0x2d 0x30 0x30 0x31` → `xxh64=0x571e3e04781b0ff5` (decimal `6277523119516880885`), `mod 4 = 1`; (B) `account_id=42` stringified → bytes `0x34 0x32` → `xxh64=0x6de6f5d076d742b9` (decimal `7919287270473417401`), `mod 4 = 1`. Decimal-string UTF-8 encoding rule pinned explicitly (not the `i64` binary form). `shard_count` callouts: `4` (widened fixture), `1` (MVP), `36` (production default). Reference impl `twox-hash::xxh64::xxh64`. **Verified byte-stable on first attempt by resink-core's supervisor swap against `twox-hash::xxh64::xxh64`** — Example A hex matched exactly (see Surprises). `verified-at: master @ contracts/2026-05-10-kafka-ingress.md §2.1`. In-memory contract §4 gained the one-paragraph back-reference pinning the same encoding rule.
- **`conventions/duckdb.md` pytz convention (KR1.3) — new file.** First DE-owned `conventions/` tree. One normative paragraph stating DuckDB's Python driver lazy-imports `pytz` at `TIMESTAMPTZ` row-fetch time, so any consumer reading `TIMESTAMPTZ` from DuckDB MUST declare `pytz` as a hard dep in `pyproject.toml` (under `[project].dependencies`, not optional). Canonical example link: sim-farm's `pyproject.toml` (already carries `pytz>=2024.1` per the 2026-05-16 retro). `verified-at: master @ conventions/duckdb.md`.
- **Contract-type migration cooperation (KR1.4) — board orchestrated cleanly.** Board's migration commit landed; DE confirmed `type: rfc → type: contract` on both `2026-05-10-kafka-ingress.md` and `2026-05-16-in-memory-event-source.md`. Three loops of the `type: rfc` workaround are now closed. `verified-at: master @ frontmatter of both contract files`.

## What we didn't ship and why

- Nothing material. Light loop, four KRs, four delivered. The conventions/duckdb.md file remains at `type: rfc` (not migrated this loop) because the board's migration was scoped to the two existing contract files; the conventions doc is a new artifact filed under the same workaround and will migrate when the next `conventions` type lands (or simply stays at `rfc` — defensible per the existing workaround comment). This is intentional, not a miss.

## Surprises

- **xxHash64 worked example verified byte-stable on first attempt.** Resink-core's supervisor `partition()` impl, calling `twox-hash::xxh64::xxh64`, produced **`0x571e3e04781b0ff5`** for `user_id="u-001"` — identical to DE's Python-computed value. The risk row in the team OKR (worked-example hex divergence) closed without a correction needed. The `<verified-against-rust-impl: pending>` tag in §2.1 can be flipped to `verified` next loop.
- **`type: rfc → type: contract` migration finally landed.** Three loops of the workaround (Kafka 2026-05-10, in-memory 2026-05-16, this loop's `fact_account_open` schema fragment) are closed. Board orchestrated the migration centrally — DE just confirmed, no body edits to the contracts beyond frontmatter. Cleaner than the team OKR's risk row anticipated.
- **First DE-owned `conventions/` tree established.** The `teams/platform/data-engineering/conventions/` directory did not exist before this loop. With `duckdb.md` landed, DE now has a dedicated home for cross-package conventions (DuckDB Python driver behaviors, future Arrow/Parquet conventions, future tz/timestamp conventions). Discoverability is the open question — see Asks.
- **Schema fragment did not need a same-loop addendum.** Resink-core's `dim_account_fixture` column set held to the CEO brief O1 KR1.1 verbatim spec `(account_id, user_id, account_type, status, valid_from, valid_to, is_current)`; no divergence surfaced during build, so the cross-team reconciliation risk row closed without action.

## Asks

- **Board (low priority):** cross-link the DE `conventions/` tree from `org-os/conventions.md` (if appropriate) so future ICs can discover it without spelunking the team tree. Not blocking; just a discoverability hint.
- **Reaffirm posture (informational):** no new event-source pattern contracts (webhook source, S3-poll source, CDC-from-snapshot source) needed until consumer pressure arises. DE continues to carry these as deferred per the 2026-05-16 status and this loop's Out-of-scope list.

## Metrics

- **KR1.1** (`fact_account_open` schema fragment lands as additive §9 in Kafka contract + parallel cross-link in in-memory contract; not a new file): **met** — Kafka §9.1–§9.6 + in-memory §5.1. Single PK `account_id`, `op="insert"` only, SCD2-shaped `after` row matching `dim_account_fixture`, cross-dim FK note `account.user_id → dim_user.user_id`, topic `nanofab.ingress.<tenant>.fact_account_open`, partitioning via §2.1 worked example B.
- **KR1.2** (xxHash64 ratification entry inside Kafka §2 with worked example, UTF-8 decimal-string encoding rule, `shard_count` callouts, `twox-hash::xxh64::xxh64` reference impl): **met** — Kafka §2.1 `Ratification (2026-05-23)`. Two worked examples (Example A `u-001`, Example B `42`), decimal-string rule pinned, `shard_count` 4/1/36 all called out, reference impl named. Byte-stability verified on first attempt by resink-core's supervisor (see Surprises).
- **KR1.3** (`pytz` DuckDB conventions doc at `teams/platform/data-engineering/conventions/duckdb.md`, one-paragraph normative entry, sim-farm `pyproject.toml` as canonical example): **met** — file landed; first DE `conventions/` tree established; normative paragraph + canonical example link in place. Filed `type: rfc` per the workaround (board's migration scoped to the two prior contracts; no same-loop migration of this file).
- **KR1.4** (contract-type migration cooperation — confirm board's `type: rfc → type: contract` migration on three files; remove type-note HTML comments): **met for the two contracts** — `2026-05-10-kafka-ingress.md` and `2026-05-16-in-memory-event-source.md` both at `type: contract` on master, frontmatter migration confirmed. The conventions doc stays at `type: rfc` (not part of this loop's migration scope; intentional, see What we didn't ship).
