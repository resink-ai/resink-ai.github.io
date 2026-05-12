---
layout: default
title: "Spec: 2026-05-09-org-os-bottom-up-flow-design"
date: 2026-05-09
status: active
type: design
owner: board
---

<!-- original-frontmatter:
  title: Org-OS bottom-up flow — team-initiated work and cross-team requests
  type: design
  status: active
  date: 2026-05-09
  owner: board
  links: parent: docs/superpowers/specs/2026-05-08-ai-native-org-os-design.md
-->
# Org-OS bottom-up flow

## Problem

The current org-os is strictly top-down. The executive loop runs CEO brief → team decomposition → build → exec summaries → CEO consolidation → CEO retro. Team OKRs may only carry objectives that trace back to the CEO brief. Two important kinds of work have no home:

1. **Team-initiated stewardship.** A team (e.g. SRE) wants to commit capacity to service stability, observability, or developer ergonomics. These rarely move the company vision directly, but neglecting them grows operational risk and slows every other team. Today there is no path to do this work without the CEO naming it.

2. **Cross-team feedback and requests.** One team (e.g. Application Engineering) needs something from another team (e.g. SRE — better deployment tooling). Today the only place this can surface is as a one-line "ask" in the requesting team's OKR. There is no canonical artifact, no triage protocol, no closure, no roll-up. Asks are easily lost in chat or quietly dropped.

Both gaps mean a healthy chunk of real work runs under the floorboards and only surfaces as friction at retro time, if at all.

## Goals

- Give teams a structured way to propose their own work without inventing a parallel governance track.
- Give cross-team requests a first-class lifecycle owned by the receiver, with explicit accept / decline / escalate semantics.
- Keep the CEO informed of *balance* (capacity by source, request flow health) without forcing the CEO to approve individual stewardship items or small inter-team asks.
- Add as little surface area as possible: one new ritual, two new artifact types, modifications to existing rituals/roles/conventions.

## Non-goals

- Replacing the CEO loop. The brief and consolidation remain the spine.
- Bypassing CEO approval for substantive cross-team work. The four escalation triggers ensure anything large or boundary-crossing routes through the brief.
- Building tooling. This spec defines the file shape and ritual flow only. Lint and automation come later.

## Authority model

Teams **propose**; the CEO **ratifies in the brief**. Bottom-up work is real and originates from the team, but every loop's slate is blessed by the CEO at brief time so capacity allocation across top-down and bottom-up stays coherent. Small cross-team requests stay between two EMs and never reach the CEO; large ones surface through the same intake → brief mechanism without inventing a parallel track.

### Escalation triggers (any one fires → CEO ratifies)

A cross-team request escalates from team-to-team handshake to CEO-ratified when **any** of the following hold:

1. **Effort.** Receiver's sizing exceeds a capacity threshold (e.g. ~1 week of loop capacity, or auto-escalate at `size: L`).
2. **Scope rejection.** Receiver wants to decline or counter-scope, and requester wants to push back.
3. **Interface change.** The request would alter the receiver's charter, owned products, or stable interfaces.
4. **Fan-out.** More than one receiving team is needed to fulfill the request.

## Concepts and lifecycle

### Request

A formal ask from a *requester* team to a *receiver* team. Has its own lifecycle:

```
open → triaged → accepted | declined | escalated → fulfilled | dropped
```

- `open` — requester has filed it; receiver has not yet looked.
- `triaged` — receiver has read and sized it.
- `accepted` — receiver commits to deliver. Spawns a team-initiated objective in the receiver's next OKR.
- `declined` — receiver will not do it. Reason recorded; ends the request.
- `escalated` — at least one of the four triggers fires. Surfaces in the receiver's proposals doc and through to the CEO brief slate.
- `fulfilled` — accepted request shipped; closed at exec-summary time.
- `dropped` — accepted request did not ship in its loop; either re-accepted next loop or withdrawn.

### Team-initiated objective

An objective that does not trace to a CEO brief objective. Its origin is either the team's own stewardship judgment or an accepted cross-team request. A team OKR thus carries objectives from three sources: `ceo-brief`, `team-initiated`, `cross-team-request`. The `source` field on each objective makes the origin explicit so it can be rolled up cleanly.

### Where bottom-up plugs into the loop

A new `team-intake` ritual runs *first* in each loop, before `ceo-brief`. Each EM produces a per-team proposals doc covering team-initiated objectives, accepted cross-team requests, and escalated requests. The CEO brief then ratifies the slate alongside top-down objectives.

## Artifacts

### NEW artifact: `request`

Lives in the **receiver's** team directory: `teams/<layer>/<team>/requests/YYYY-MM-DD-<slug>.md`. The requester does not own a separate copy — they reference the canonical file via path. Single source of truth, no sync problem.

```yaml
---
type: request
owner: <receiver team path>
date: YYYY-MM-DD
status: open | triaged | accepted | declined | escalated | fulfilled | dropped
requester: <requesting team path>
size: S | M | L
escalation:
  triggers: [effort | scope | interface | fan-out]   # null until/unless escalated
  brief: board/okrs/<date>-ceo-brief.md              # filled when ratified
links:
  fulfilled_by: <team-okr path or null>              # set when accepted; points at the OKR objective
---
```

Body sections:
- **Ask.** What's needed and why.
- **Receiver triage.** Sizing, accept/decline rationale, counter-scope if any.
- **Resolution.** Filled at fulfilled/dropped time; links to the delivered artifact or records the drop reason.

### NEW artifact: `team-proposals`

Lives at `teams/<layer>/<team>/proposals/YYYY-MM-DD-team-proposals.md`. One per team per loop. Input the EM brings into the CEO brief discussion.

```yaml
---
type: team-proposals
owner: <team path>
date: YYYY-MM-DD
status: draft | active | archived
loop: YYYY-MM-DD
---
```

Body: four sections — proposed team-initiated objectives, accepted cross-team requests rolling into this loop, escalated requests for the CEO to arbitrate, and outbound requests this team filed against other teams (visibility only; not for ratification).

### MODIFIED artifact: `okr`

Each objective grows a `source` field with three valid values, mapped to where `links.source` must point:

| `source`              | `links.source` points at                              |
|-----------------------|--------------------------------------------------------|
| `ceo-brief`           | objective in the brief (default; may omit if obvious) |
| `team-initiated`      | line in `<team>/proposals/<date>-team-proposals.md`   |
| `cross-team-request`  | the canonical `<receiver>/requests/<date>-<slug>.md`  |

Existing OKRs default to `source: ceo-brief` and remain valid.

## Directory layout

Every team directory grows two subdirectories:

```
teams/<layer>/<team>/
├── charter.md
├── status.md
├── okrs/
├── retros/
├── exec-summaries/
├── proposals/        # NEW — per-loop team-proposals docs
├── requests/         # NEW — incoming cross-team requests; this team is receiver
└── agents/
```

There is no `requests-out/`. The requester finds outbound requests by grepping `requester: <self>` across all teams' `requests/` dirs, or by following the back-link from its proposals doc.

`board/` is unchanged.

## Rituals

### Updated `executive-loop` step order

```
1. Read state
2. team-intake (per team, parallel)               ← NEW
3. ceo-brief                                       ← MODIFIED
4. team-planning (per team, parallel)              ← MODIFIED
5. build (per team)
6. exec-summary (per team)                         ← MODIFIED
7. ceo-consolidation                               ← MODIFIED
8. retro
```

### NEW ritual: `team-intake`

**When.** First per-team work in each loop, after the previous loop's retro lands. Runs in parallel for every team.

**Inputs.**
- Team's `requests/` inbox.
- `status.md`.
- Last `exec-summaries/<prev>.md`.
- Last team OKR (to see what stewardship debt was deferred).

**Steps.**
1. **Drain the inbox.** For each request with `status: open`: size it (S/M/L), apply the four escalation rules, set `status` to one of `accepted | declined | escalated`, fill the "Receiver triage" section. L-sized requests auto-escalate; smaller ones may also escalate if any of the four triggers fire.
2. **Surface team-initiated proposals.** EM lists stewardship/observability/devex objectives the team wants to commit to, each with rough cost and "what gets dropped if this lands."
3. **Write the proposals doc.** `<team>/proposals/<date>-team-proposals.md` with three sections: team-initiated objectives, accepted cross-team requests, escalated requests for CEO.

**Outputs.**
- `<team>/proposals/<date>-team-proposals.md`.
- Updated frontmatter on every triaged request file.

**Acceptance.**
- Inbox contains zero `open` requests.
- Proposals doc has `status: active`.
- Every accepted request points at this proposals doc by `<team>/proposals/...` reference.

### MODIFIED ritual: `ceo-brief`

**New inputs.** Every team's `<date>-team-proposals.md`.

**New step.** Insert between drafting top-down objectives and assigning to teams:
- Read every team-proposals doc.
- For each team: ratify the team-initiated slate (approve, trim, or defer with reason), ratify accepted cross-team requests as belonging in this loop (or defer), sequence escalated requests across teams (which receiver picks them up, which loop, which trade-offs).

**Modified output.** Each team's section of the brief lists objectives from all three sources. The brief explicitly states the capacity split it expects per team (e.g., "platform/sre: ~60% top-down, ~30% stewardship, ~10% cross-team support").

### MODIFIED ritual: `team-planning`

**New input.** Team's own ratified `<date>-team-proposals.md`.

**Modified steps.** When decomposing CEO brief objectives into team OKR objectives:
- Carry through every ratified team-initiated objective with `source: team-initiated` and `links.source: ../proposals/<date>-team-proposals.md`.
- Carry through every accepted cross-team request as an objective with `source: cross-team-request` and `links.source: ../requests/<date>-<slug>.md`. Set `links.fulfilled_by` on the request file to point back at this OKR.
- Top-down objectives carry `source: ceo-brief` (default).

**New acceptance.**
- Every objective in the OKR has a `source` field.
- Every non-`ceo-brief` objective has a `links.source`.
- Every cross-team-request objective has a matching back-link from the request file.

### MODIFIED ritual: `exec-summary`

**New step.** Insert before "list asks for the CEO": for every cross-team-request objective in this loop's OKR, update the request file:
- Shipped → `status: fulfilled`, fill "Resolution" section with link to delivered artifact.
- Not shipped → `status: dropped`, record reason. Requester decides whether to re-file.

**Modified roll-up.** "Shipped" and "non-shipped" sections get sub-grouped by `source` so the receiver can see at a glance how much went to top-down vs stewardship vs cross-team support.

**New acceptance.** Every request linked from this loop's OKR has been moved out of `accepted` into `fulfilled` or `dropped`.

### MODIFIED ritual: `ceo-consolidation`

**New step.** Compute and write two roll-ups in the company exec summary:
- **Capacity by source, per team and overall.** "platform/sre delivered 12 objectives: 7 ceo-brief, 3 team-initiated, 2 cross-team-request."
- **Request flow.** Total filed, triaged, accepted, declined, escalated, fulfilled, dropped — overall and per requester→receiver pair. This is the early-warning signal for cross-team friction.

**Modified acceptance.** Company exec summary contains both roll-ups; each non-zero entry links to its source artifacts.

### `retro` — no structural change

Existing `product | tenant | org-os` classification still covers everything. New sources of friction (request triage drag, escalation overuse, stewardship being squeezed out) all show up as `org-os` proposals. Add one line to retro guidance: review the consolidation request-flow roll-up as input to "what didn't work."

## Roles

### `roles/EM.md` — expanded

**Inputs (additions).**
- Team's `requests/` inbox.
- Last loop's request flow for this team.

**Outputs (additions).**
- `<team>/proposals/<date>-team-proposals.md`.
- Triage state on every incoming request (status, sizing, decision rationale, counter-scope when offered).
- Closed-out request files at exec-summary time (`fulfilled` or `dropped` with resolution).

**Decision rights (additions).**
- **Triages incoming requests.** Sets `accepted | declined | escalated` per the four escalation rules. Declines must carry a written reason; escalations must name the trigger.
- **Proposes team-initiated objectives.** Surfaces them in the proposals doc; CEO ratifies in the brief.
- **Files outbound requests.** Writes a `request` file in the receiving team's `requests/` directory on behalf of the team.
- **Counter-scopes a request.** May propose a smaller version; if requester accepts the counter, status moves to `accepted`; if requester pushes back, EM moves it to `escalated`.

**Boundaries (additions).**
- Does not unilaterally accept a request that fires any of the four escalation triggers — must escalate.
- Does not fulfill an accepted request without a corresponding objective in the team OKR (no shadow work).
- Does not file requests against another team without a written ask in the canonical `requests/<date>-<slug>.md` location (no chat-only asks).

**Rituals owned (addition).** [team-intake](../rituals/team-intake.md).

### `roles/CEO.md` — expanded

**Inputs (additions).** Every team's `<date>-team-proposals.md` (read at brief time).

**Decision rights (additions).**
- **Ratifies team-initiated slates.** Approves, trims, or defers each team's stewardship proposals during the brief.
- **Sequences escalated cross-team requests.** Decides which receiver picks them up, in which loop, and which top-down objectives they displace.
- **Sets capacity expectations by source.** May write target splits per team in the brief to keep stewardship from being squeezed out over consecutive loops.

**Boundaries (additions).**
- Does not file or triage individual cross-team requests — that is the EM's job.
- Does not skip the team-intake outputs when writing the brief; team proposals docs are required inputs, not optional context.
- Does not approve a team-initiated objective that wasn't surfaced in the proposals doc — bottom-up flow must be visible, not whispered.

### `roles/IC.md` — small addition

ICs may **draft** outbound requests and surface team-initiated proposals to the EM, but only the EM files them and only the EM triages incoming ones. Keeps the request graph one-EM-thick and prevents IC-to-IC asks from becoming invisible side channels.

### `roles/Agent.md` — no structural change

Agents inherit whichever role they fulfill. An Agent acting as EM gains the new triage and intake duties automatically.

## Conventions

### `type` enum (additions)

```yaml
type: <one of: charter | okr | exec-summary | retro | adr | rfc | status | agent-spec
              | request | team-proposals>
```

### Required frontmatter per new type

| type             | additional required keys                                                          |
|------------------|-----------------------------------------------------------------------------------|
| `request`        | `requester: <team path>`, `size: S \| M \| L`, `links.fulfilled_by` (nullable)   |
| `team-proposals` | `loop: YYYY-MM-DD`                                                                |

### `status` enum (extended for `request` only)

Existing `draft | active | archived` stays for every other type. `request` uses a richer state machine because its lifecycle is the point of the artifact:

- `charter | okr | exec-summary | retro | adr | rfc | agent-spec | team-proposals`: `draft | active | archived`
- `status` (singleton file): `live | archived`
- `request`: `open | triaged | accepted | declined | escalated | fulfilled | dropped`

A note in `conventions.md` makes the asymmetry explicit so tooling can validate it: "`request` is the only type whose status field is its lifecycle, not its publication state."

### OKR objective `source` field

Each objective gains a one-line `source:` and optional `links.source:` per objective in the body, per the table in the Artifacts section above.

### Mutation rule for requests

Added under "Mutation rules":
- **`request` files mutate in place.** Unlike OKRs and briefs, a request's status field moves through its lifecycle on the same file across loops. The file is created in the loop it was filed and stays under the receiver's `requests/` directory throughout. A request reaching `fulfilled` or `dropped` is terminal; after one full loop has passed it becomes eligible to be moved to `requests/archive/` for housekeeping (the status field stays `fulfilled` or `dropped` — there is no separate `archived` status for `request`).

### Validation hints (advisory section)

- Every `team-initiated` objective must have a `links.source` pointing inside the same team's `proposals/`.
- Every `cross-team-request` objective must have a `links.source` pointing inside some other team's `requests/`, and that request file's `links.fulfilled_by` must point back at this OKR.
- Every `accepted` request whose `loop` matches the current loop must appear in exactly one team OKR as a `cross-team-request` objective.
- An `open` request older than two loops is a triage-debt warning.

## Worked flows

### Flow 1 — Team suggests its own project (e.g. SRE proposes a stability initiative)

1. During `team-intake`, the SRE EM drafts an entry in `teams/platform/sre/proposals/<date>-team-proposals.md` under "team-initiated objectives": one-liner, why it matters, rough cost, what gets dropped if it lands.
2. CEO brief ratifies the slate. Approved team-initiated objectives are listed in the brief under SRE's section, marked `source: team-initiated`.
3. `team-planning` writes the team OKR. SRE's OKR carries an objective with `source: team-initiated` and `links.source: ../proposals/<date>-team-proposals.md`. From here on it's a normal OKR objective.
4. `build` and `exec-summary` treat it like any other objective.
5. `ceo-consolidation` rolls up by source: "30% of platform capacity went to team-initiated stewardship this loop" becomes a visible line.

### Flow 2 — Team sends a request to another team (e.g. App Eng asks SRE for better deployment tooling)

1. App Eng EM files the request at `teams/platform/sre/requests/<date>-deploy-ergonomics.md` (canonical location is the receiver's dir). Frontmatter: `type: request`, `owner: teams/platform/sre`, `requester: teams/application/app-eng`, `status: open`, `size: M`.
2. App Eng's own next `team-intake` lists this in their proposals doc under "outbound requests filed" — for visibility, not for ratification (App Eng isn't doing the work).
3. The request sits in SRE's `requests/` inbox until SRE's next intake.

### Flow 3 — How the receiving team consumes feedback

1. At SRE's next `team-intake`, the EM reads every file in `requests/` with `status: open`. For each: size it, decide accept/decline/escalate per the four rules, record the decision in the "Receiver triage" section, update `status`.
2. Accepted requests are listed in SRE's proposals doc under "accepted requests this loop." Each will spawn a team-initiated objective in SRE's OKR with `source: cross-team-request`, `links.source: ../requests/<date>-deploy-ergonomics.md`. The request file's `links.fulfilled_by` is set to point back at the OKR objective.
3. Declined requests stay in `requests/` with `status: declined` and a written reason. Requester sees the close and may re-file with revised scope, or drop it.
4. Escalated requests are listed in SRE's proposals doc under "escalations for CEO." They surface in the CEO brief discussion; CEO sequences them across teams and ratifies.
5. CEO brief ratifies SRE's slate, which now includes both team-initiated stewardship and accepted cross-team requests.
6. `exec-summary` closes the loop on each request: when the spawning OKR objective ships, the EM updates `status: fulfilled`. If it didn't ship, `status: dropped` plus reason.
7. `ceo-consolidation` rolls up cross-team request flow as a health metric: requests filed, accepted, declined, escalated, fulfilled, dropped — per team and per requester→receiver pair.

## What this gives the org

- Every team-initiated project has a paper trail from idea (proposals doc) → approval (CEO brief) → execution (OKR objective) → outcome (exec summary).
- Every cross-team request has a single canonical file owned by the receiver, with a lifecycle that closes explicitly. No asks lost in chat.
- Small asks stay between two EMs; large ones surface through the same intake → brief mechanism without inventing a parallel track.
- Roll-up by `source` lets the CEO watch the *balance* between top-down work, stewardship, and cross-team support — a leading indicator of org health — without micromanaging individual items.

## Open questions

- Whether to formalize a default capacity cap for team-initiated work in `conventions.md` (e.g., "no team should propose more than ~30% of loop capacity for team-initiated objectives without CEO conversation"), or leave it as a brief-time judgment call. Lean: leave it judgmental in Phase 2; revisit if data shows squeeze.
- Whether to add a `priority: high | normal` field to `request` so receivers can sequence the inbox without reading every file. Lean: not yet; sizing covers most of it and adding priority invites politicking.
