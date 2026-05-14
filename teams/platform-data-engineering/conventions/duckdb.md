---
layout: default
title: "platform-data-engineering convention: duckdb"
date: 2026-05-23
status: active
type: convention
owner: teams/platform/data-engineering
grand_parent: Teams
parent: "Team: platform-data-engineering"
---

<!-- original-frontmatter:
  type: convention
  owner: teams/platform/data-engineering
  date: 2026-05-23
  status: active
-->
{% raw %}

<!--
Type note: conventions.md (as of 2026-05-23) does not include `contract`
or `convention` in the type enum; the closest fit is `rfc` (per the same
workaround used by the 2026-05-10 Kafka ingress contract and the 2026-05-16
in-memory event-source contract). ADR-2026-05-10-004 (board-owned, ratifying
this loop per CEO brief 2026-05-23 O2 KR2.1) introduces a `contract` type;
once it lands, this file migrates `type: rfc → type: contract` alongside the
two existing contracts per CEO brief O2 KR2.4 / DE team OKR KR1.4. Until then,
this file is filed as an RFC under the existing workaround.
-->

# DuckDB pytz Convention — Python-driver runtime dep

**Scope:** every package in the resink.ai monorepo that depends on `duckdb` via
the Python driver and reads any `TIMESTAMPTZ` column.

**Convention.** DuckDB's Python driver requires the `pytz` package at runtime
to materialize `TIMESTAMPTZ` rows. The DuckDB wheel does **not** declare `pytz`
as a transitive dependency — it lazy-imports `pytz` only when a `TIMESTAMPTZ`
column is fetched, then raises `ModuleNotFoundError` if the import fails.
Therefore, any package whose code path can read a `TIMESTAMPTZ` column from
DuckDB MUST declare `pytz` as an explicit hard dependency in its
`pyproject.toml` (under `[project].dependencies`, not under
`[project.optional-dependencies]`). The canonical example is
[`repos/resink-ai/resink-core/sim-farm/pyproject.toml`](../../../../repos/resink-ai/resink-core/sim-farm/pyproject.toml),
whose SCD2 diff engine fetches tz-aware UTC `valid_from` / `valid_to` columns
and which carries `pytz>=2024.1` as a load-bearing dep alongside `duckdb` and
`pyarrow`. The dep is non-optional because the DuckDB driver fails at row-fetch
time, not at import time, so a missing `pytz` becomes a runtime regression that
unit tests against in-memory fixtures may not catch unless those fixtures also
include `TIMESTAMPTZ` columns.

## Links

- Canonical example: [sim-farm pyproject.toml](../../../../repos/resink-ai/resink-core/sim-farm/pyproject.toml)
- Triggering retro item: [2026-05-16 CEO retro](../../../../board/retros/2026-05-11-0958-ceo-retro.md)
- Driving CEO brief: [2026-05-23 CEO brief — O6 KR6.2](../../../../board/okrs/2026-05-11-1113-ceo-brief.md)
- DE team OKR for this loop: [2026-05-23 team OKR — KR1.3](../okrs/2026-05-11-1113-team-okr.md)
- Adopting ADR (pending this loop): [ADR-2026-05-10-004 — `contract` artifact type](../../../../board/decisions/2026-05-10-004-contract-artifact-type.md)
{% endraw %}
