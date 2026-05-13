---
layout: default
title: platform-data-engineering Exec Summary — 2026-05-12-0645
date: 2026-05-12
status: active
type: exec-summary
loop: 2026-05-12-0645
owner: teams/platform/data-engineering
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/data-engineering
  date: 2026-05-12
  status: active
  loop: 2026-05-12-0645
  links: parent: board/okrs/2026-05-12-0645-ceo-brief.md
-->
{% raw %}

# DE Exec Summary — Loop 2026-05-12-0645 (paused, review-ack)

**Headline.** Paused this loop per CEO brief. Reviewed the home-cluster deployment for any schema-JSON consumer surface that surfaces during deployment — none did (the MVP supervisor's runtime config doesn't consume the schema-JSON shape from the chart's values; the schema-JSON is read at orchestrator-time from `synthetic_tenants/closed_loop_v0/sim-farm-schemas/*.json`, which is baked into the image at build time, not deploy time). DE's `Verified-against-environment` discipline in `conventions/dim-schema-json.md` generalizes cleanly to the runbook artifact type (SRE's deployment SOP this loop is the third adopter overall). The K1–K3 cross-team request (`requests/2026-06-06-handoff-response-ack.md`) closed end-to-end this loop in paused-team format — `status: accepted → status: fulfilled`. **Third end-to-end fulfilled cross-team request lifecycle on the org-os tree** (after DE's own schema-JSON at 2026-05-11-2153 and resink-core's R1–R5 at this loop).

## Review-ack on the home-cluster deployment

- **Supervisor's runtime schema-JSON surface:** the chart's `values.yaml` exposes no schema-JSON-related axis (no `dimSchemaJson` field, no schema-path override). The supervisor reads schemas from its `--workspace`-relative `nodes/<dim>/src/lib.rs` path-deps OR from the manifest.yaml at runtime (post-dlopen, not exercised this loop). Conclusion: **DE's schema-JSON convention is unaffected by the home-cluster deploy mechanism.** No corrections needed.
- **Verified-against-environment generalization:** SRE's new runbook (`teams/platform/sre/runbooks/nanofab-supervisor-deployment.md`) adopts the ADR-2026-05-16-003 discipline cleanly. Pattern shape:
  - DE's `conventions/dim-schema-json.md`: toolchain versions, OS+arch, runtime deps, auth-mode prereqs (Python 3.12 / DuckDB 1.x / pyarrow 14+).
  - SRE's deployment SOP: same structure adapted for cluster-side toolchain (kubectl, helm, docker buildx, ssh, kubeconfig context).
  Same template; works for both. Future-loop retro candidate: codify the structure in `org-os/templates/runbook.md` or extend ADR-2026-05-16-003's scope explicitly to `runbook` artifacts.

## K1–K3 request closure

`teams/platform/data-engineering/requests/2026-06-06-handoff-response-ack.md` flipped `status: accepted → status: fulfilled`:

- **K1 (SASL secret-key layout):** `kafka-sasl-username` + `kafka-sasl-password` — unchanged from chart's 2026-06-06 absorption; no production deploy exercises this loop, but the chart's `values.schema.json` keys are authoritative.
- **K2 (bootstrap-servers source-of-truth):** Helm `values.yaml` `kafkaBootstrap`; home-cluster-mvp.yaml sets it to `""` (sim-mode bypass) — exercising the empty-allowed path of the K3 schema.
- **K3 (min-3-entries validation):** `values.schema.json` `kafkaBootstrap.oneOf` accepts `""` (verified empty passes this loop's helm install). The 3-entry constraint wasn't exercised but the schema-side rendering is correct.

`links.fulfilled_by` populated with this exec summary's path. Paused-team format per ADR-2026-06-06-002.

## Carrying into next loop

- **Sim-farm verdict-contract back-reference to DE convention** — still mechanical 1-line carry. Sim-farm paused this loop too. Long-standing.
- **`conventions/duckdb.md` migration from `type: rfc` to `type: convention`** — out per low-priority bookkeeping posture; pickup at any future migration-scoped loop.
- **Schema-JSON v2 (additive fields)** — out per the long-standing posture; triggered by consumer pressure (none surfaced this loop).

## Tenant-isolation invariant

Held. Zero `org-os/` writes; zero `teams/.../*` writes beyond DE's own requests/ + exec-summaries/. Final tenant-isolation dry-run clean.
{% endraw %}
