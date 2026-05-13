---
layout: default
title: "ADR 2026-05-08-002: tenant tree restructure"
date: 2026-05-08
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-08
  status: active
  decision: Tenant tree restructured — board/ for executive desk, teams/{platform,application}/ for workforce, agent-engineering/data-engineering full-name slugs, application/realtime-pipeline → teams/application/resink-core
-->
{% raw %}

# ADR 002: Tenant tree restructure

## Context

After Phase 1 landed, the user performed a user-driven directory rename on disk. Four changes were made simultaneously:

1. **`company/` → `board/`** — the executive-desk directory was renamed to better reflect its semantic role as an executive board dashboard rather than a generic "company" bucket.
2. **`platform/` and `application/` → `teams/platform/` and `teams/application/`** — all team directories were moved under a new top-level `teams/` parent, grouping the entire workforce under one directory and distinguishing team artifacts clearly from the board and org-os layers.
3. **Full-name slugs for AE and DE** — `platform/ae/` became `teams/platform/agent-engineering/` and `platform/de/` became `teams/platform/data-engineering/`. The other slugs (`devops`, `sre`) were already descriptive and kept unchanged.
4. **`application/realtime-pipeline/` → `teams/application/resink-core/`** — the product was renamed from its internal descriptor ("realtime-pipeline") to its brand-aligned product name ("Resink Core"), matching the product vision in `ORG.md`.

These moves were already on disk. However, `org-os/conventions.md`, all org-os rituals, roles, playbooks, templates, and every tenant artifact (charters, OKRs, exec summaries, retros, decisions) still referenced the old paths and the old owner format. A propagation pass was required to bring every file into alignment with the new structure.

The `org-os/conventions.md` owner field format was also updated: it previously accepted a short enum (`company | platform/<team> | application/<team>`); it now states that the owner is the **directory path from the repo root** to the unit's home (e.g., `board`, `teams/platform/agent-engineering`, `teams/application/resink-core`). This is a meaningful semantic change — the owner field now self-describes the artifact's location rather than using an abbreviated taxonomy.

Per spec §6 and the merge-evolution-proposal playbook, any change to `org-os/` requires an ADR before the change is merged. This ADR is the formal record after the fact, covering both the directory rename and the org-os propagation pass.

## Decision

Adopt the new directory shape as the canonical tenant layout for resink.ai:

- `board/` holds all executive-board artifacts (charter, OKRs, decisions, retros, exec summaries).
- `teams/platform/<team>/` holds platform team artifacts. Current teams: `agent-engineering`, `data-engineering`, `devops`, `sre`.
- `teams/application/<team>/` holds application team artifacts. Current teams: `resink-core`, `sim-farm`.

The `owner` field in YAML frontmatter is now defined as: **the directory path from the repo root to the unit's home**. This definition is codified in `org-os/conventions.md`. All tenant artifacts and org-os docs have been updated to match. The new tree shape and owner enum are the source of truth going forward; any further structural change requires a new ADR.

## Alternatives considered

- **A: Revert the directory renames** — rejected. The new naming is more semantically meaningful: `board` better describes executive-level artifacts than the generic `company`; full-name slugs (`agent-engineering`, `data-engineering`) are self-documenting and eliminate the cognitive load of resolving `ae` and `de`; `resink-core` aligns the internal artifact name with the product brand. Reverting would discard a genuine improvement in clarity.
- **B: Keep org-os referencing the old paths and update only tenant artifacts** — rejected. Doing so would break the `extract-org-os.md` playbook for any new tenant adopting the framework, because the playbook's `mkdir` commands and directory contract trees would describe a structure that does not match the tenant layout the tenant would actually create. It would also contradict the directory contracts in `org-os/conventions.md`, undermining the self-consistency guarantee that makes the org-OS portable.

## Consequences

- **Positive:** cleaner semantic naming throughout; all teams clearly grouped under a `teams/` parent (easy to see workforce vs. board vs. org-OS at a glance); product name (`resink-core`) is aligned with the brand stated in `ORG.md`; owner field is now a self-describing path rather than an opaque shorthand.
- **Negative / costs:** a one-time edit ripple across approximately 30 files was required; old commit history still references the old paths in commit messages (acceptable — git history is immutable and the ADR serves as the dated record of the transition); any external tooling or scripts that hard-coded the old paths (`company/`, `platform/ae/`, etc.) will need updating when Phase 2 automation lands.
- **Follow-ups required:** Phase 2 automation (slash commands, CI lints, scheduled agents) must use the new paths from day one. The frontmatter linter stub (from ADR-001 / DevOps next OKR) should validate that `owner:` values start with `board`, `teams/platform/`, or `teams/application/`.

## Links

- Triggering change: user-driven directory rename on disk (no triggering retro this time; this ADR is the formal record after the fact, per the spec §6 rule that any org-os change requires an ADR).
- Related: [ADR-001](2026-05-08-001-frontmatter-validation.md).
- Affected playbook: [`extract-org-os.md`](../../org-os/playbooks/extract-org-os.md).
{% endraw %}
