---
layout: default
title: "Spec: 2026-05-08-ai-native-org-os-design"
parent: "Design specs"
render_with_liquid: false
---
# AI-Native Org-OS — Design Spec

**Date:** 2026-05-08
**Status:** Draft, awaiting user review
**Scope:** Phase 1 — repository structure, templates, and rituals as markdown. Automation/runtime wiring is a separate, later spec.

## 1. Problem & goal

The `newbase/` repo today blurs two distinct concerns:

- **A meta concept** — "how an AI-native company operates" (`README.md`).
- **A specific company's vision** — resink.ai's realtime data engine (`ORG.md`).

It is also incomplete: `application/` is empty, the four `platform/<team>/` READMEs are one-line stubs, there is no shared structure for OKRs, decisions, retros, or rituals, and no defined mechanism by which the org evolves itself.

**Goal.** Turn `newbase/` into a working, self-evolving org operating system with engineering-management discipline encoded directly into the file tree, such that:

1. A new AI-native company can adopt the same skeleton by forking `org-os/` and writing a fresh `ORG.md`.
2. resink.ai (the first tenant) has a populated, navigable home for OKRs, decisions, retros, charters, and exec summaries.
3. The "CEO loop" described in `README.md` is formalized as a runnable, document-driven workflow that produces inspectable artifacts at every step.
4. The org can change *itself* — its rituals, roles, team boundaries — through a defined evolution path, not ad-hoc edits.

## 2. Two layers, one repo

```
newbase/
├── README.md            # the concept + how to navigate this repo
├── ORG.md               # tenant vision (resink.ai)
├── org-os/              # PORTABLE — the operating system itself
├── company/             # TENANT — CEO-level state
├── platform/            # TENANT — platform departments
└── application/         # TENANT — application departments (resink-specific)
```

- **`org-os/`** is portable. Nothing in `org-os/` references resink.ai. To bootstrap a new company: copy `org-os/`, write a new `ORG.md`, scaffold a fresh tenant tree using the playbook in `org-os/playbooks/extract-org-os.md`.
- **Tenant directories** (`company/`, `platform/`, `application/`) hold this specific company's living state.

The split is enforced by convention plus a single rule: **no file inside `org-os/` may name a specific tenant**. Tenant-specific examples in `org-os/` use placeholders like `<TENANT>` or `acme.ai`.

## 3. The Executive Loop

The README's six-step loop is formalized as the **Executive Loop**, the top-level rhythm of the company. One iteration produces a fixed, inspectable set of artifacts.

```
[ORG.md, last exec-summaries, open OKRs]
            │
            ▼
   1. CEO Brief                  → company/okrs/YYYY-MM-DD-ceo-brief.md
            │
            ▼
   2. Team Planning (parallel,   → platform/<team>/okrs/YYYY-MM-DD-team-okr.md
      one per team)               application/<team>/okrs/YYYY-MM-DD-team-okr.md
            │
            ▼
   3. Build phase                → tasks executed by agents
                                   (Phase 1: tracked as checklists in OKR docs;
                                    Phase 2: dispatched to coding agents)
            │
            ▼
   4. Team Exec Summary          → platform/<team>/exec-summaries/YYYY-MM-DD.md
                                   application/<team>/exec-summaries/YYYY-MM-DD.md
            │
            ▼
   5. CEO Consolidation          → company/exec-summaries/YYYY-MM-DD.md
      (rolls up all team
       summaries into one view)
            │
            ▼
   6. CEO Retro                  → company/retros/YYYY-MM-DD-ceo-retro.md
            │
            ▼
   7. Evolution Proposals (0+)   → company/decisions/YYYY-MM-DD-<slug>.md
                                   (or RFCs for larger changes)
            │
            ▼
        next loop
```

Each step has a ritual document in `org-os/rituals/` that defines its inputs, outputs, prompt template, and acceptance criteria. The output filenames are stable so any agent or human can find the latest CEO brief, team OKR, etc., without searching.

A loop's cadence is not fixed by the spec. Daily, weekly, or "whenever the CEO calls one" are all valid. The cadence itself is a tenant decision recorded in `company/charter.md`.

## 4. The org-os layer

```
org-os/
├── README.md            # what the org-os is, how to use it, how to extract it
├── conventions.md       # naming, frontmatter, status fields, directory contracts
├── roles/               # role contracts
│   ├── CEO.md
│   ├── EM.md            # engineering manager / team lead
│   ├── IC.md            # individual contributor (the "agent doing the build")
│   └── Agent.md         # how a coding agent is configured & invoked
├── rituals/             # runnable workflows
│   ├── executive-loop.md
│   ├── ceo-brief.md
│   ├── team-planning.md
│   ├── build.md
│   ├── exec-summary.md
│   ├── ceo-consolidation.md
│   ├── retro.md
│   └── decision-review.md
├── templates/           # blank documents the rituals fill in
│   ├── charter.md
│   ├── okr.md
│   ├── exec-summary.md
│   ├── retro.md
│   ├── adr.md
│   ├── rfc.md
│   ├── status.md
│   └── agent-spec.md
└── playbooks/           # one-off procedures
    ├── onboard-team.md
    ├── spawn-agent.md
    ├── extract-org-os.md
    └── merge-evolution-proposal.md
```

### 4.1 Roles

A role document describes:

- **Purpose** — what this role exists to do.
- **Inputs** — files, signals, decisions consumed.
- **Outputs** — artifacts produced and where they go.
- **Decision rights** — what this role can approve unilaterally vs. must escalate.
- **Boundaries** — what this role does *not* do.

Roles are not job titles for humans. They are **contracts** that any actor (human, Claude Code session, scheduled agent) can fulfill. The same person or agent can hold multiple roles.

### 4.2 Rituals

A ritual document is a runnable prompt with structured outputs. Format:

```markdown
# Ritual: <name>

## When
<trigger>

## Inputs
<list of files / state to read>

## Steps
<numbered prompt the actor follows>

## Outputs
<exact file paths and templates to fill>

## Acceptance
<how to know this ritual has been completed correctly>
```

In Phase 1, an actor (human or Claude session) executes a ritual by reading the document and following it. In Phase 2, rituals become slash commands and `/loop` schedules.

### 4.3 Templates

Every artifact in the company tree is created from a template in `org-os/templates/`. Templates carry YAML frontmatter so artifacts are machine-readable:

```yaml
---
type: okr            # one of: charter | okr | exec-summary | retro | adr | rfc | status | agent-spec
owner: platform/ae   # team path or "company"
date: 2026-05-08
status: draft        # draft | active | archived
loop: 2026-05-09-1715     # which executive loop this artifact belongs to (if any)
links:
  parent: company/okrs/2026-05-08-ceo-brief.md
---
```

Frontmatter is the contract that makes rolling up team summaries into a CEO retro mechanical rather than artisanal.

### 4.4 Playbooks

Playbooks are procedures that don't run on the executive-loop cadence: spinning up a new team, defining a new agent persona, extracting `org-os/` for a new company, merging an evolution proposal.

## 5. The tenant layer

### 5.1 `company/` — the CEO desk

```
company/
├── charter.md           # tenant identity; cadence; pointer to ORG.md
├── okrs/                # YYYY-MM-DD-ceo-brief.md per loop
├── decisions/           # ADRs and merged RFCs (cross-cutting decisions)
├── retros/              # YYYY-MM-DD-ceo-retro.md per loop
└── exec-summaries/      # rolled-up summaries per loop
```

### 5.2 `platform/<team>/` and `application/<team>/`

Every team directory has the same shape:

```
<team>/
├── charter.md           # mission, owned products, interfaces, success metrics
├── okrs/                # team OKRs per loop
├── retros/              # team retros (optional, when surfaced from CEO retro)
├── exec-summaries/      # team exec summaries per loop
├── status.md            # the latest at-a-glance state (single file, overwritten)
└── agents/              # agent-spec.md files for this team's agents
```

This uniformity is the contract that lets the executive-loop ritual roll up summaries without team-specific code paths.

### 5.3 Tenant seed for resink.ai (Phase 1 deliverables)

- `ORG.md` — light cleanup (typos, headings) preserving the existing vision text.
- `company/charter.md` — points to `ORG.md`, declares loop cadence (start with "weekly"), names CEO.
- `company/okrs/2026-05-08-ceo-brief.md` — first CEO OKR derived from `ORG.md`: the three top objectives are (a) shippable training experience, (b) shippable serving experience, (c) Sim Farm v0.
- Four platform charters upgraded from one-line stubs:
  - `platform/ae/charter.md` — Agent Engineering. Owned products: plugin marketplace, internal agent library.
  - `platform/de/charter.md` — Data Engineering. Owned products: pipeline runtime, dbt/Spark/Flink components.
  - `platform/devops/charter.md` — DevOps. Owned products: IaC, CI/CD, deployment automation.
  - `platform/sre/charter.md` — SRE. Owned products: monitoring, alerting, runbooks, incident response.
- `application/` seeded with two department stubs derived from `ORG.md`: `realtime-pipeline/` (training+serving) and `sim-farm/`. Each gets a charter only; OKRs come in the first loop.

## 6. Self-evolution

A retro is the only place new evolution proposals originate. The retro ritual classifies each proposal into one of three change classes, with distinct review paths:

| Class | Targets | Review path | Lands as |
|---|---|---|---|
| **Product** | code in application/* or platform/* repos | normal team OKR in next loop | task in team OKR |
| **Tenant** | charter, OKR scope, team boundaries | team lead approves; CEO approves cross-team | edit + ADR in `company/decisions/` |
| **Org-OS** | files inside `org-os/` | CEO approves; ADR mandatory | edit + ADR in `company/decisions/` |

Every Org-OS change requires an ADR. This is the load-bearing rule that keeps "self-evolving" from drifting into "self-mutating in untraceable ways".

The playbook `org-os/playbooks/merge-evolution-proposal.md` walks an actor through:

1. Identify class.
2. Open ADR (template in `org-os/templates/adr.md`).
3. Apply the change.
4. Record the ADR link in the next CEO brief so it propagates.

## 7. Conventions

`org-os/conventions.md` codifies:

- **Naming** — `YYYY-MM-DD-<slug>.md` for dated artifacts; `charter.md`, `status.md` for stable singletons.
- **Frontmatter** — required fields per `type` (see §4.3).
- **Status field** — `draft | active | archived`. Anything older than two loops without `active` is auto-archived (Phase 2; manual in Phase 1).
- **Linking** — relative paths from repo root, no absolute paths.
- **No tenant names in `org-os/`** — placeholder is `<TENANT>`.

## 8. README.md update

The current `README.md` is a prose vision. It will be rewritten to:

- Open with a one-paragraph "what this repo is" framing.
- Show the two-layer structure with a labeled tree.
- Point to `ORG.md` for the tenant vision.
- Point to `org-os/README.md` for the operating system.
- Briefly describe the Executive Loop with a link to `org-os/rituals/executive-loop.md`.
- Keep it under 100 lines.

The original prose explanation of the loop moves into `org-os/rituals/executive-loop.md` as the canonical source.

## 9. Out of scope (Phase 1)

- Slash commands, `/loop` automation, agent runtime, CI checks for frontmatter, dashboards, status auto-archival, retroactive ADR backfill, multi-tenant isolation tooling, repo-level GitHub Actions.
- Anything beyond markdown documents and directory structure.
- The actual product code for resink.ai (lives in the existing `resink.ai/` parent repo, not here).

These are candidates for Phase 2 specs once Phase 1 is in regular use.

## 10. Acceptance criteria

Phase 1 is complete when:

1. The repo layout in §2 exists.
2. Every file listed in §4 (org-os) and §5 (tenant) exists with non-stub content.
3. The first executive loop has actually been run end-to-end: every artifact listed in §3 exists for date `2026-05-08` (CEO brief, team OKRs, team exec summaries, company exec summary, CEO retro, and zero or more decisions), all generated from the templates in `org-os/templates/`.
4. The four `platform/<team>/charter.md` files describe owned products, interfaces, and success metrics — no one-line stubs.
5. `org-os/playbooks/extract-org-os.md` produces, when followed, a clean fork-able skeleton with no resink.ai references.
6. A reader new to the repo can answer "where do I look for X?" for X in {vision, current OKRs, latest decisions, team status, how a ritual works, how to start a new team} by reading `README.md` alone.

## 11. Risks & mitigations

- **Risk: ritual sprawl.** Too many rituals, not enough adoption.
  *Mitigation:* Phase 1 ships exactly the 8 rituals listed in §4. New rituals require a retro proposal and an ADR.
- **Risk: tenant leakage into `org-os/`.** Examples drift to mention resink.ai.
  *Mitigation:* the "no tenant names in `org-os/`" rule, plus an `extract-org-os.md` dry-run in the acceptance criteria.
- **Risk: docs that nobody reads.** Templates fill up but the loop never runs.
  *Mitigation:* `README.md` foregrounds the Executive Loop; the first real loop is run as part of acceptance.
- **Risk: evolution path used to bypass review.** "Org-OS changes" smuggled in as "tenant changes".
  *Mitigation:* the classification rule in §6 is in `merge-evolution-proposal.md`; every `org-os/` edit is checked at retro time.
