---
layout: default
title: "Spec: 2026-05-10-nanofab-serving-deployment-design"
---
# Nanofab Serving & Deployment — Design Spec

**Date:** 2026-05-10
**Status:** Draft, awaiting user review
**Scope:** Sub-project #4 of the resink.ai product family — the operational shape of running nanofab in production for many tenants. Cloud topology, multi-tenancy, region/availability, CI/CD for AI-generated code, secrets, cost & billing. Product UX is sub-project #5.

## 1. Context & motivation

This spec defines *how* the artifacts produced by sub-projects #1-3 (runtime, training, Sim Farm) are deployed and operated. The functional design of those systems is out of scope; this spec deals with the deployment topology, isolation, CI/CD, secrets, and cost accounting that wraps them.

Two design pressures shape the choices:
1. **Time to market.** v1 ships a multi-tenant SaaS to early customers; full enterprise isolation comes later as a paid SKU.
2. **The product is "AI-generated production code shipped continuously."** That puts unusual weight on CI/CD safety — every retrain produces new Rust artifacts that flow into production via the runtime's hot-swap path. The serving layer must enforce the gate (Sim Farm seal + manifest signature) without exception.

## 2. Decisions summary

| Axis | v1 choice | v2 / paid SKU |
|---|---|---|
| Cloud topology | Resink-managed AWS, single account per environment | BYOC (customer's cloud) as paid enterprise SKU |
| Multi-tenancy | Logical isolation in shared infrastructure (KV prefix, Kafka ACLs, IAM roles) | Dedicated supervisor pods + KV cluster + Kafka cluster as paid SKU |
| Region & availability | Single region, multi-AZ HA | Multi-region active-passive with tenant-level failover |
| CI/CD for AI codegen | Shared GitHub Actions, per-tenant content-addressed artifact store, gated by Sim Farm seal + manifest signature | Same |
| Secrets | AWS Secrets Manager per tenant, IAM role assumption per workload, 90d rotation | Same |
| Billing | Usage-based per tenant (compute + KV + Kafka + Iceberg + LLM tokens), base monthly minimum | Annual enterprise contract with committed-use discounts |

## 3. Cloud topology

### 3.1 v1 — Resink-managed AWS
- One AWS account per environment (`resink-prod`, `resink-staging`, `resink-dev`).
- All workloads in `us-east-2` (3 AZs). Region choice driven by lowest blast radius for AWS regional outages combined with broad service coverage; revisit before any tenant requires data residency elsewhere.
- IaC: Terraform repo (`websites/resinkit_deployer/terraform/`-style, per CLAUDE.md). Every resource tagged with `tenant=shared` or `tenant=<id>` for cost allocation.

### 3.2 v2 — BYOC paid SKU
- Customer creates an AWS account (or eventually GCP/Azure) and grants resink an IAM role with scoped permissions (EKS admin within a dedicated VPC, S3 R/W in named buckets, Secrets Manager R within named namespaces).
- Resink runs the same Terraform modules against the customer's account. Same nanofab software; different ownership of the underlying compute and storage.
- Sim Farm and the coordinator stay in the resink account (verdicts cross account boundaries via cross-account IAM); supervisor fleet, KV, Kafka, Iceberg sit in the customer's account.
- Pricing: annual contract; customer pays AWS bills directly + a resink platform fee.

## 4. Multi-tenancy isolation

### 4.1 v1 — Logical isolation in shared infrastructure

| Resource | Isolation mechanism |
|---|---|
| KV state | Key prefix `t/<tenant>/...`, enforced by the State Layer (runtime spec §4.4). The State Layer constructor takes a `tenant_id` and prepends it on every read/write — no API surface lets a request escape its tenant prefix. |
| Kafka ingress topics | Naming `<tenant>.fact_*` plus per-tenant SASL credentials with topic-level ACLs (read on customer's source topics, write on internal topics scoped to that tenant). |
| Kafka internal topics (cross-shard, watermark, simfarm forensics) | Same `<tenant>.internal.*` and `<tenant>.simfarm.*` naming. |
| Iceberg tables | One namespace per tenant in the catalog (`tenant_<id>.dim_*`), customer-readable via per-tenant Glue/REST credentials. |
| Workspace repos | One git repo per tenant on resink-controlled gitea/gitlab; resink-write, customer-read via per-tenant deploy keys. |
| Query Gateway API | Per-tenant API keys; gateway extracts tenant from key and prefixes all KV reads. |
| Compute | Shared supervisor pods, scheduled by k8s with per-tenant CPU/memory quotas via `LimitRange` + per-tenant `ResourceQuota`. Tenant fairness via cgroup CPU shares. |
| Sim Farm | Shared coordinator + worker pool; jobs tagged with `tenant_id` for accounting and forensic isolation. |
| Logs / metrics | Tenant_id is a required label on every metric and structured log line. Per-tenant Grafana datasources project the same Prometheus instance through a tenant filter. |

The runtime supervisor is the single most security-sensitive component (it holds in-memory state for many tenants). To bound risk: every node plugin runs inside `panic::catch_unwind` (already required for crash safety), tenant-prefix enforcement is duplicated in the State Layer AND in the supervisor's request router (defense in depth), and a chaos-test in CI fires cross-tenant access attempts to detect regressions.

### 4.2 v2 — Dedicated SKU
For tenants on the dedicated tier:
- Their own supervisor pods (k8s `nodeSelector` + taints), no co-residence with other tenants.
- Their own KV cluster (TiKV/FoundationDB instance).
- Their own Kafka cluster (or dedicated Redpanda nodes).
- Sim Farm coordinator + workers stay shared (verdict isolation already enforced by tenant-tagged jobs); enterprise customers can opt to dedicate Sim Farm too at additional cost.

## 5. Region & availability

### 5.1 v1 — Single region, multi-AZ HA
- 3 AZs in `us-east-2`. Every component runs ≥ 2 replicas spread across AZs.
- Coordinator (runtime, training, Sim Farm) runs HA via Raft with one quorum member per AZ.
- KV cluster (TiKV) replicates with 3-AZ placement rules.
- Kafka brokers spread across AZs with `min.insync.replicas=2`.
- DR: continuous Iceberg snapshot replication to a DR S3 bucket in `us-west-2`. RPO target < 5 min for the Iceberg snapshots; RTO measured in hours (manual rebuild from snapshots, since a passive region with warm state is v2 work).

### 5.2 v2 — Multi-region active-passive with tenant-level failover
- Passive replica region (`us-west-2`) running idle supervisor + KV + Kafka clusters.
- Iceberg snapshots streamed to passive region; Kafka MirrorMaker2 replicates ingress topics.
- Per-tenant failover: a tenant can be moved between regions individually. Their KV namespace is rebuilt from the passive region's continuously-applied Iceberg snapshots + Kafka tail; cutover time bounded by Kafka tail catch-up (typically minutes).
- Active-active cross-region is explicitly out of scope (would require distributed transactions across KV regions, conflicting with the runtime's "no distributed txns in hot path" stance, runtime spec §7.7).

## 6. CI/CD for AI-generated code

### 6.1 Pipeline shape
Training pushes a release tag (`release/v<n>`) to the per-tenant workspace repo. A GitHub Actions (or Argo Workflows) pipeline triggers:

```
1. Checkout workspace repo at release tag.
2. cargo build --release --workspace
   - per-node crate cached by content-hash; unchanged crates skip rebuild
   - failure → CI fails; orchestrator notified; no artifact emitted
3. For each built .so:
     hash = sha256(file)
     upload to s3://nanofab-artifacts/<tenant>/<node_id>/<hash>.so
4. Generate signed manifest:
     {dag_version, node_id → s3 URI mapping, build commit, ci_run_id}
   Sign with a CI-only KMS key (`nanofab-ci-signer`).
5. Verify Sim Farm release seal exists for this commit (look up
   simfarm.verdicts Iceberg table by workspace commit_sha).
6. POST signed manifest + seal reference to runtime coordinator's
   PublishDagVersion endpoint.
7. Coordinator validates BOTH the manifest signature AND the seal,
   then accepts the DAG version.
```

Step 5+7 are non-negotiable: without a Sim Farm seal, no manifest reaches the runtime. CI cannot bypass; the coordinator's signature-validator rejects any unsigned manifest.

### 6.2 Artifact store
- S3 bucket with versioning enabled and tenant-scoped IAM policies.
- Content-addressed objects are immutable; same `(tenant, node_id, content_hash)` always returns the same `.so`. This is what enables training's incremental retrain (training spec §3.2 step 4) — unchanged nodes' URIs don't change, so the runtime coordinator's diff classifier (runtime spec §6.1) treats them as no-ops.
- Lifecycle policy: keep current + previous 4 manifest versions' artifacts (5 total), tier older versions to S3 Glacier, expire after 1 year (or whatever the tenant's retention contract specifies).

### 6.3 Plugin distribution to supervisors
- Supervisor's startup sequence (runtime spec §4.2) fetches the DAG manifest from the coordinator, then resolves each node's `.so` URI and downloads it to local disk before `dlopen`.
- Per-supervisor disk cache (default 10 GB, LRU). Hot reload is therefore a one-line URI swap + cached download check + `dlopen`.
- For per-node hot swap (runtime spec §6.2), the new `.so` URI flows through the same path; supervisors download in parallel during shadow phase, eliminating cold-fetch latency at cutover.

## 7. Secrets & credentials

### 7.1 Storage
- AWS Secrets Manager per tenant. Hierarchical naming: `/tenants/<id>/<category>/<name>`.
- Categories:
  - `ingest/kafka_sasl` — credentials for the customer's source Kafka (when ingesting from a customer-owned cluster instead of having them push into ours)
  - `ingest/parquet_source` — IAM keys for customer S3 buckets they want us to ingest from
  - `vendors/<vendor>` — third-party API keys (IP geo, fraud scoring) the customer brings
  - `egress/iceberg_role` — IAM role ARN for the customer's S3 if Iceberg egress is to a customer-owned bucket
  - `gateway/api_keys` — the customer-facing API keys for the Query Gateway (auto-rotated, revealed once on creation)

### 7.2 Access
- Workloads do not have static credentials. Each pod runs under a tenant-scoped IAM role (e.g., `arn:aws:iam::<acct>:role/nanofab-supervisor-<tenant>`) granted via IRSA (IAM Roles for Service Accounts) on EKS.
- The role's policy permits `secretsmanager:GetSecretValue` only on `/tenants/<tenant>/*`. No supervisor pod can read another tenant's secrets even if compromised.
- Cross-tenant supervisor pods don't exist in v1 — when a supervisor is shared (the v1 default), its IAM role allows `secretsmanager:GetSecretValue` only on the secrets needed for the shared workload (Kafka cluster admin, KV cluster admin), never per-tenant secrets. Per-tenant secrets are accessed by *worker subprocesses* spawned per tenant ingress (tighter blast radius).

### 7.3 Rotation
- Default 90-day rotation, automated via Secrets Manager rotation Lambdas.
- Customer-supplied secrets (vendor API keys): customer rotates; resink surfaces an "expiring soon" banner in the product UX 14 days before contract metadata says they expire.
- Resink-generated secrets (Gateway API keys, ingest credentials we manage): rotated automatically with overlap window; customer notified of new key, old key valid for 7 days.

## 8. Billing & cost

### 8.1 Per-tenant accounting
Every chargeable operation tags its metric with `tenant_id`:

| Metric | Source | Unit |
|---|---|---|
| Compute | k8s metrics server, container CPU·s and memory·s | vCPU·hours, GB·hours |
| KV ops | State Layer counters (per `(tenant, op)`) | ops |
| KV storage | TiKV per-prefix size | GB·month |
| Kafka in/out | Broker JMX, per-topic | bytes |
| Kafka storage | Broker JMX, per-topic | GB·month |
| Iceberg storage | S3 inventory, per-namespace | GB·month |
| Iceberg scans | Trino/DuckDB query stats (when proxied through resink) or unmetered (direct customer queries) | TB scanned |
| LLM tokens (training) | Per-sub-agent provider invoices | tokens |
| Sim Farm runs | Coordinator job log | run-seconds, by mode |

A daily Spark-or-DuckDB aggregator writes per-tenant rows to a `billing.usage` Iceberg table. Schema: `(tenant_id, day, metric, value, unit)`.

### 8.2 Pricing model (v1 starting point)

- **Base tier:** $X/month per tenant. Includes a small compute envelope, a small KV envelope, up to N nodes in the DAG, up to M training runs/month. Designed to be profitable on the smallest customers.
- **Usage tier:** anything above base is charged per-unit. Compute, KV ops, KV storage, Kafka, Iceberg storage, training LLM tokens — each line item.
- **Sim Farm:** included in base for reasonable validation cadence; metered for excessive shadow sessions or expensive blue/green warmups.
- **Enterprise SKU:** annual contract, committed-use discount, dedicated infrastructure, BYOC option, support SLA.

Specific dollar values are intentionally not in this spec — those depend on COGS analysis once we have early customer data.

### 8.3 Cost guardrails
- Per-tenant *soft* spending caps (notify at 80% of monthly budget set by the customer in product UX) and *hard* caps (suspend training runs, throttle ingestion above N events/sec; production reads always preserved unless customer's account is delinquent).
- Anomaly detection: per-tenant daily-spend Z-score vs trailing 30-day baseline; alerts at +2σ, suspends discretionary work (training, retrain) at +3σ until human review.

## 9. Open questions

- **Data residency requirements** (GDPR, China, etc.): deferred until a customer asks; will gate v2 multi-region work and may force EU/Asia regional deployments earlier than planned.
- **SOC 2 timeline**: not in this spec; a sales prerequisite that drives auditing requirements onto every other component.
- **Encryption-at-rest default**: TiKV, Kafka, S3 all support it; assume on by default and revisit only if performance forces otherwise.

## 10. Out of scope

- The functional design of runtime, training, Sim Farm — covered by their own specs.
- Product UX (billing dashboard, secret rotation UI, infrastructure status page) — sub-project #5.
- Specific dollar pricing — depends on COGS data.
- v2 BYOC or multi-region implementation details — separate spec when those are the next priority.

## 11. Glossary

- **BYOC (Bring Your Own Cloud):** v2 paid SKU where customer provides the AWS/GCP/Azure account and resink operates the software within it.
- **Logical isolation:** tenants share infrastructure but are separated by namespacing (key prefixes, topic names, IAM scopes). v1 default.
- **Physical isolation:** tenants get dedicated infrastructure (pods, KV, Kafka). v2 paid SKU.
- **Release seal:** the `release_seal.json` Sim Farm produces (training spec §4.9) — required for the runtime coordinator to accept a DAG version.
- **Manifest signature:** signature on the CI-built manifest using `nanofab-ci-signer` KMS key — also required at the runtime coordinator gate.
- **IRSA (IAM Roles for Service Accounts):** the EKS mechanism that lets a pod assume a specific IAM role without static credentials.
