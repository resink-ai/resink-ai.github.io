---
layout: default
title: "Spec: 2026-05-10-nanofab-product-ux-design"
parent: Design specs
---
{% raw %}
# Nanofab Product UX — Design Spec

**Date:** 2026-05-10
**Status:** Draft, awaiting user review
**Scope:** Sub-project #5 of the resink.ai product family — the customer-facing surfaces. Web dashboard, training chat experience, design-fork rendering, dashboard tabs, access model, audit & source-repo view, CLI, notifications. Implementation choices for the underlying platform (Next.js / FastAPI / etc.) are out of scope here; this spec defines product surface and behaviors.

## 1. Context & motivation

The four prior specs define what nanofab *does*. This spec defines how customers *experience* it. Two design pressures:

1. **Customers don't think in DAGs.** They think in dim tables, dashboards, and metrics. The UX must let them describe what they want in their own language and surface only the technical complexity that requires their input (design forks, sign-offs, billing).
2. **The system runs continuously and AI-driven.** A customer who logs in to find their data warehouse has been quietly retrained needs immediate context: what changed, why, what was validated, what they need to approve. Surface design-time and operate-time states with the same coherence.

## 2. Decisions summary

| Axis | Choice |
|---|---|
| Surface model | Standalone web app (`resink.ai/dashboard`) primary; CLI for power users; Slack/email for time-sensitive events. No embedded chatbots in v1. |
| Chat experience | Hybrid — chat in the center, fixed sidebar with live structured context (schemas, forks, validation state) |
| Design-fork rendering | Inline interactive cards in chat; persistent `/decisions` archive |
| Dashboard scope | Eight tabs: Tables, Lineage, Verdicts, Hot swaps, Sources, Billing, Secrets, Audit |
| Access model | SSO (Google, Okta, GitHub) with three-role RBAC + optional Auditor; SCIM on enterprise SKU |
| Audit / source view | Read-only workspace browser with syntax-highlighted diffs; signed tarball export |

## 3. Surface model

### 3.1 Primary: web app
- URL: `app.resink.ai` (production), `app.staging.resink.ai` (staging).
- Tenants are routed by subdomain or path prefix per Owner preference: `acme.app.resink.ai` or `app.resink.ai/acme/`.
- Built on the existing frontend stack (Next.js 15 App Router per CLAUDE.md). Server-side data fetching via server actions; the typed openapi-fetch client in `src/lib/clients/schema.d.ts` is the only path to backend APIs.

### 3.2 Power-user CLI
- Single binary `resink` (Rust, distributed via Homebrew, apt, scoop, GitHub releases).
- Commands cover: workspace ops (`resink workspace pull`, `workspace diff`), retrain triggers (`resink retrain --add-source ...`, `resink retrain --field-add ...`), manifest export (`resink manifest get vN`), verdict query (`resink verdict latest`, `verdict show <id>`), data peek (`resink table head dim_user --limit 100`), and ops (`resink hot-swap status`, `hot-swap rollback`).
- Auth via OAuth device-code flow that stores a refresh token per machine; same RBAC as the web app.

### 3.3 Notifications
- Slack and email integrations per tenant. Customers configure routing per event type.
- Event categories with default routing:
  - **Critical (page):** training run failed at validator stage 4 awaiting sign-off > 24h, hot swap rolled back, KV unavailable > 5 min, billing hard cap reached.
  - **High (notify):** Sim Farm verdict regressed vs prior run, secret expiring within 14d, soft billing cap reached, new manifest version published.
  - **Info (digest):** training run started, retrain completed cleanly, weekly cost summary.
- No in-product alarm bell that just mirrors emails — drives notification fatigue. Notifications are an *outbound* channel; the dashboard is where state lives.

## 4. Chat experience (training session)

### 4.1 Layout
```
┌──────────────────────────────────────────────────────────┐
│  Header: tenant, current workspace commit, training run #│
├────────────────────────────────────┬─────────────────────┤
│                                    │                     │
│                                    │  Sidebar:           │
│  Chat thread                       │   • Schemas         │
│   (orchestrator + customer)        │     dim_user      ✓ │
│                                    │     dim_post      ✓ │
│   ┌────────────────────────────┐   │     dim_velocity  ⏳│
│   │ Inline fork card           │   │                     │
│   │ Pick: SCD2 vs daily-part.  │   │   • Pending forks   │
│   └────────────────────────────┘   │     2               │
│                                    │                     │
│  [type a message ...]              │   • Validation      │
│                                    │     stage 1: ✓      │
│                                    │     stage 2: ⏳     │
│                                    │     stage 3: —      │
│                                    │     stage 4: —      │
└────────────────────────────────────┴─────────────────────┘
```

### 4.2 Chat semantics
- Orchestrator messages are typed: plain prose, design-fork card, schema preview, validation report, gate-stage update.
- Customer messages are: free-form text (routed to orchestrator triage per training spec §6.5), inline fork resolution, or commands (e.g., `/rewind to v1` reverts the workspace).
- Long-running operations (codegen, validator runs) are represented as collapsed status cards that expand to show progress; chat is not blocked while they run.
- The chat pane and the sidebar share a session — clicks in the sidebar (e.g., "edit schema") open a modal that the next chat message acknowledges.

### 4.3 Sidebar
- **Schemas:** live list of dim/ADS tables. Each is `(name, status: ✓ approved | ⏳ proposed | ✏ edited | ⚠ stale)` with click-to-view full schema. Status `✏ edited` means the customer modified the YAML directly; orchestrator picks up on its next stage.
- **Pending forks:** count badge; click to jump to the fork card in chat.
- **Validation:** the four gate stages with status (`—` not yet attempted, `⏳` running, `✓` passed, `✗` failed). Click a stage to see its report.
- **Workspace state:** current commit SHA + dirty/clean indicator + "view workspace" link to the audit tab.

## 5. Design-fork rendering

### 5.1 Inline card structure
```
┌──────────────────────────────────────────────────┐
│  Fork: dim_user_engagement_window_stats          │
│  Where? sub_agent: dim-table-designer            │
│                                                  │
│  ◉ Option A — SCD2 (recommended)                │
│      One row per (user, window) updated each     │
│      event. Best for sub-second lookups.         │
│  ○ Option B — Daily-partitioned                 │
│      Snapshot per day, simpler model. Adds 24h   │
│      to "current" stats freshness.               │
│  ○ Option C — Hybrid (per-day partitions for    │
│      historical + SCD2 for current 7d window)    │
│                                                  │
│  [ Pick Option A ]  [ Edit YAML ]  [ Discuss ]  │
└──────────────────────────────────────────────────┘
```

- **Pick** resolves the fork and progresses the orchestrator.
- **Edit YAML** opens a modal with the proposed schema YAML; on save, the modified YAML becomes the answer (orchestrator treats this as Option D bespoke).
- **Discuss** lets the customer ask a clarifying question; the chat continues with the orchestrator's response, and the card stays pinned at the bottom of the chat until resolved.

### 5.2 `/decisions` archive page
- Time-ordered list of every resolved fork. Each entry: timestamp, fork question, options shown, option chosen, rationale (auto-extracted from the chat exchange around the resolution), link to the resulting workspace commit.
- Filterable by sub-agent, by training run, by date.
- Used during audit and during retraining (orchestrator references prior decisions when proposing similar choices).

## 6. Dashboard tabs

### 6.1 Tables
- List of all dim/ADS tables in the live DAG. For each: schema, sample rows (configurable row count, refresh-on-demand), source-stream lineage, last update time, current row count.
- Sample-row fetcher uses the Query Gateway (`GetTableSample(name, limit)`); count derives from KV stats.
- Per-table "history" sub-view shows SCD2 versions for a given key over time (debug-friendly).

### 6.2 Lineage
- Interactive DAG visualization. Nodes grouped visually by shard. Edges colored by intra-shard (in-process) vs cross-shard (Kafka).
- Click a node: detail panel with the node's pattern, current stats (events/sec, p99 latency, cache hit rate, dedup hit rate), source code link (audit tab), most recent test pass status.
- Hover an edge: event-rate moving average, schema of the messages flowing on it.
- Visualizes "what changed" between two manifest versions when the customer toggles a comparison mode.

### 6.3 Verdicts
- Sim Farm verdict history. Filterable by mode (A/B/C), by tenant action (training, hot swap), by status.
- Per-verdict drill-down shows the three layers, sampled divergent rows, throughput metrics, link to full forensic trace in object storage.
- "Compare to previous" button highlights regression deltas — the workhorse view during a hot-swap rollback investigation.

### 6.4 Hot swaps
- In-flight: any active per-node shadow or DAG blue/green session, with progress bars and live diff rate.
- Recent: last 30 days of completed swaps. Per-swap timeline (training pushed → CI built → simfarm shadow → cutover or rollback) with click-to-jump on each event.
- Rollback button on completed swaps (gated by Owner role; double-confirm dialog).

### 6.5 Sources
- Configured ingest streams (Kafka topics, parquet sources). Per-source: schema, current event rate, ingest lag (Kafka consumer lag), error rate, last successful ingestion time.
- Add-a-source wizard launches a training retrain.

### 6.6 Billing
- Current month: per-line-item usage with cap progress bars (compute, KV ops, KV storage, Kafka, Iceberg, LLM tokens, Sim Farm runs).
- Historical: monthly costs back to tenant inception.
- Anomaly alerts: any +2σ days highlighted with explanation.
- Soft and hard cap configuration. Owner only.

### 6.7 Secrets
- Per-secret card: name, category (ingest / vendor / egress / gateway), expiration (or "auto-rotated"), last-rotated time, status (✓ healthy, ⚠ expiring < 14d, ✗ expired or invalid).
- Customer-managed secrets get a "rotate now" form; resink-managed secrets get "rotate now" button (with overlap window explanation).
- Owner role required for view; Auditor can see metadata only (no values).

### 6.8 Audit
- Read-only browser of the per-tenant Cargo workspace repo. Files with syntax highlighting (Rust, YAML, SQL, Python).
- Commit history with diffs. Each commit linked to the training run and sub-agent that produced it.
- Signed manifest history: every published DAG version, its CI build artifact links, the Sim Farm verdict that gated it, the coordinator's acceptance log entry.
- "Export workspace" button: generates a signed tarball (`resink-tenant-<id>-<date>.tar.gz`) with the workspace + decision log + verdicts. Owner role; rate-limited to once per day.
- Coordinator decisions log: immutable append-only feed of every coordinator action (DAG version published, hot swap initiated, rollback decided, etc.) with timestamp, actor (system or human), and rationale.

## 7. Access model

### 7.1 Authentication
- SSO providers in v1: Google OAuth, Okta OIDC, GitHub OAuth. One provider per tenant (Owner sets it during onboarding).
- Session: JWT-backed, 12h sliding window for active users, 24h hard cap.
- Service accounts (for CI integrations, automation): per-tenant, scoped tokens with explicit allowed actions.

### 7.2 RBAC roles
- **Owner:** everything. Billing edits, secret rotation, tenant deletion, member management. At least 1 per tenant; cannot be removed below 1.
- **Editor:** training chat, schema edits, customer sign-off (validator stage 4), source configuration, hot-swap rollback (with confirmation). Cannot edit billing or secrets.
- **Viewer:** read-only dashboard. No chat, no edits, no rollbacks.
- **Auditor:** read-only including the Audit tab and the export action; cannot see secret values; cannot see chat. Designed for compliance reviewers and external auditors.

### 7.3 SCIM (enterprise SKU)
- v2 paid feature: tenants can wire their SCIM provider (Okta, Azure AD) for user provisioning. Roles map from SCIM groups.

## 8. CLI surface (load-bearing commands)

```
resink login                     # OAuth device flow
resink workspace pull            # clone the tenant workspace to a local path
resink workspace diff            # diff local edits vs orchestrator's view

resink retrain --request "add fraud signals: > 5 posts in first hour"
resink retrain --add-source kafka://broker/topic --schema schema.json
resink retrain --field-add fact_post_publish.is_draft:bool

resink manifest get vN > manifest.yaml
resink manifest current

resink verdict latest
resink verdict show <run_id>

resink table head dim_user --limit 100
resink table schema dim_user

resink hot-swap status
resink hot-swap rollback <swap_id>   # interactive confirm

resink decisions list
resink decisions show <decision_id>
```

CLI commands hit the same APIs as the web app; permissions enforced server-side via the user's RBAC role.

## 9. Open questions

- **Visual companion / mockups for design forks**: do customers benefit from inline diagram previews (e.g., a small DAG-fragment visualization for "before vs after this fork")? Defer to post-launch UX research.
- **In-product chat-with-resink-engineer**: support escalation from training chat to a human resink engineer. Useful for novel-pattern reviews; out of scope for v1, defer to a v2 spec when support team scales.
- **Marketplace / pattern sharing**: customers contributing back patterns to the shared library. Interesting future direction; not v1.

## 10. Out of scope

- Specific visual design (colors, typography, component library) — covered by design system work, not this spec.
- Marketing site (`resink.ai` root) and pricing page — separate.
- Mobile app — not v1.
- Customer-built dashboards on top of the Iceberg tables (BI integration) — customers use their own BI tools (Tableau, Looker, etc.) against the Iceberg surface; we don't reinvent.

## 11. Glossary

- **Tenant:** the customer organization; the unit of isolation, billing, and workspace ownership.
- **Owner / Editor / Viewer / Auditor:** RBAC roles within a tenant.
- **Workspace:** the per-tenant Cargo + manifest git repo (defined in training spec §4.2).
- **Fork:** a non-trivial design choice surfaced by a sub-agent; rendered as an inline card in chat with options.
- **Decision:** a resolved fork; archived in `/decisions`.
- **Verdict:** Sim Farm's typed output (defined in Sim Farm spec §4.7); displayed in the Verdicts tab.
- **Hot swap:** any DAG version transition (per-node or DAG-level) defined in runtime spec §6; surfaced in the Hot swaps tab.
{% endraw %}
