---
layout: default
title: "ADR 2026-05-23-001: conventions enum extension"
date: 2026-05-23
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-23
  status: active
  decision: "Extend the type enum in org-os/conventions.md to admit five additional owned-artifact types in one ratification batch: `runbook`, `convention`, `playbook`, `report`, and `role`; add a row to the additional-fields-per-type table for each; migrate existing files filed under workaround types to their new canonical types"
-->
# ADR 2026-05-23-001: Extend Conventions Type Enum (Five-Type Batch)

## Context

The `type` enum in `org-os/conventions.md` has been growing one ADR per type as owned-artifact surfaces have emerged. ADR-2026-05-10-004 added `contract` (loop 2026-05-11-1113 ratification). The pattern of adding owned-artifact types one ADR at a time produces a workaround backlog: each team that needs a new type either files under `type: rfc` as a temporary workaround or adopts a local extension and flags it for later canonicalization. By the time loop 2026-05-11-1302 opens, five separate owned-artifact surfaces are pending or have just landed:

- **SRE filed** `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` with `type: runbook` as an SRE-local extension (loop 2026-05-11-1113). Flagged in SRE's exec summary for canonicalization.
- **DE filed** `teams/platform/data-engineering/conventions/duckdb.md` as `type: rfc` because `convention` is not in the enum.
- **The new `org-os/playbooks/out-of-retro-org-os-change.md`** (per ADR-2026-05-09-006, loop 2026-05-11-1113) lives at a playbooks path but has no frontmatter — the enum doesn't admit `playbook`.
- **This loop (2026-05-30)** introduces `board/reports/` as a new artifact tree to host the HTML capabilities report; the path is conventions-illegal without a `report` type.
- **AE's Bundle C this loop** is creating `org-os/roles/{EM,CEO,IC}.md` — three role definitions filed under a directory the conventions table already references ("What does role Z do? → `org-os/roles/Z.md`") but with no enum entry for `role`.

Each team-owned artifact type that doesn't fit the existing enum (`charter | okr | exec-summary | retro | adr | rfc | contract | status | agent-spec | request | team-proposals`) triggers either an inline workaround or a one-ADR-per-type ratification — neither scales as the company adds owned-artifact surfaces. Batching is overdue.

## Decision

Add FIVE types to the `org-os/conventions.md` `type` enum in one ratification:

- **`runbook`** — for SRE's runbook tree under `teams/platform/sre/runbooks/`.
- **`convention`** — for DE's conventions tree under `teams/platform/data-engineering/conventions/` and analogous trees other teams may create.
- **`playbook`** — for `org-os/playbooks/` (existing entries + new ones).
- **`report`** — new this loop, for `board/reports/` (the HTML capabilities report and future reports).
- **`role`** — new this loop, for `org-os/roles/` (the role definitions AE is creating in Bundle C).

For each added type, add a row to the "Additional fields per type" table with the type-specific required keys:

| type         | additional required keys                                                |
|--------------|-------------------------------------------------------------------------|
| `runbook`    | `severity_tiers: [<list>]` (e.g., `[S1, S2, S3]`)                       |
| `convention` | none beyond the base required fields                                    |
| `playbook`   | `invocation_trigger: <one-line>` (when an actor should invoke it)       |
| `report`     | `audience: <one-line>` (who reads this — engineer, investor, hire, etc.)|
| `role`       | `responsibilities: [<list>]` (one-line entries enumerating the duties)  |

**Migration of existing files filed under workaround types.** After this ADR flips active:

1. `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md` — already filed with `type: runbook` (SRE's local extension); the local extension becomes canonical. Verify the frontmatter declares the required `severity_tiers` field (SRE's runbook committed S1/S2/S3 tiers; this is canonical going forward).
2. `teams/platform/data-engineering/conventions/duckdb.md` — migrate `type: rfc` → `type: convention`. No additional required fields; existing frontmatter is otherwise valid.
3. `org-os/playbooks/out-of-retro-org-os-change.md` — currently has no frontmatter (or carries a workaround type); add canonical frontmatter with `type: playbook` and the required `invocation_trigger` field.
4. `board/reports/2026-05-30-resink-core-capabilities.md` and `.html` — file as `type: report` from creation (both produced this loop).
5. `org-os/roles/{EM,CEO,IC}.md` — AE owns authorship; AE files with `type: role` from creation (per the conventions-legal path established by this ADR).

**Why batch five.** Three of the five (`runbook`, `convention`, `playbook`) are observed-and-pending from loop 2026-05-11-1113; two (`report`, `role`) are observed-and-pending from loop 2026-05-11-1302. Filing five separate ADRs over the next two loops would burn six ratification cycles to land what is mechanically one enum extension. The batch is bounded — the conventions table has finite type-space, and after this ratification the workaround pool is essentially drained.

## Alternatives considered

- **A: One ADR per type.** Rejected because five ratifications would consume five loop slots; the per-type required fields are short and discoverable; the surface area of the change is small.
- **B: Drop the enum entirely, accept arbitrary types.** Rejected because the enum is the discoverability + validation surface for future tooling (frontmatter-lint script in DevOps's deferred list relies on it).
- **C: Defer until more types surface.** Rejected because the workaround pool already spans five types this loop alone; the rate is high enough to act now.
- **D: Ratify the original three-type scope (`runbook`, `convention`, `playbook`) and file separate ADRs for `report` and `role` next loop.** Rejected because (a) both `report` and `role` are needed this loop to make the loop's two largest deliverables (HTML report, AE Bundle C roles) conventions-legal at write time, and (b) the additional authoring cost of two more rows in one ADR is trivial compared to two more ADR rounds.

## Consequences

- **Positive:** Conventions enum stops bottlenecking owned-artifact-type ratification for the foreseeable future. SRE/DE/board/AE can file runbooks/conventions/playbooks/reports/roles as first-class artifacts. The `type: rfc` workaround pool drains. Two of the loop's largest deliverables (HTML capabilities report, AE roles bundle) ship to conventions-legal paths at write time rather than awaiting retroactive migration.
- **Negative / costs:** Five more rows in the enum + five more rows in the additional-fields table. Risk of admitting types that don't yet have stable per-type required-field shape (mitigation: ratify with minimal required fields; extend per-type-fields later as patterns emerge — the `runbook` `severity_tiers` field, for example, may grow additional structure as SRE accumulates runbooks).
- **Follow-ups required:**
  - Update `org-os/conventions.md` (lands alongside this ratification): add five entries to the type enum, add five rows to the additional-fields table.
  - Migrate the three pre-existing files (SRE runbook frontmatter verification, DE conventions/duckdb.md, org-os/playbooks/out-of-retro-org-os-change.md) — board owns this loop.
  - Future frontmatter-lint script (DevOps's deferred work) gains type-specific required-key checks for the five new types.

## Links

- Triggering retro: [board/retros/2026-05-11-1113-ceo-retro.md](../retros/2026-05-11-1113-ceo-retro.md) — P4.
- Triggering CEO brief (further extension to five types): [board/okrs/2026-05-11-1302-ceo-brief.md](../okrs/2026-05-30-ceo-brief.md).
- Sister ADR (same family — `contract` type added): [2026-05-10-004-contract-artifact-type](2026-05-10-004-contract-artifact-type.md).
- Companion ADRs ratified same loop: [2026-05-16-003-contract-environment-verification](2026-05-16-003-contract-environment-verification.md), [2026-05-16-004-focus-loop-pattern](2026-05-16-004-focus-loop-pattern.md).
- Conventions edit landed alongside this ratification: [org-os/conventions.md](../../org-os/conventions.md).
- Worked-example artifacts (migrated to canonical types this loop): `teams/platform/sre/runbooks/nanofab-supervisor-failed-validation.md`, `teams/platform/data-engineering/conventions/duckdb.md`, `org-os/playbooks/out-of-retro-org-os-change.md`.
