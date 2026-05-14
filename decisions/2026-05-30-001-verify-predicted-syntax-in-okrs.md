---
layout: default
title: "ADR 2026-05-30-001: verify predicted syntax in okrs"
date: 2026-05-30
status: active
type: adr
owner: board
parent: Decisions (ADRs)
nav_order: 5
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-30
  status: active
  decision: "Team OKRs that reference predicted-but-unverified syntax (CLI flags, file paths, env vars) must carry an explicit verification marker; the build phase confirms or files an addendum"
-->
{% raw %}

# ADR 2026-05-30-001: Verify Predicted Syntax in OKRs at Planning Time

## Context

The `claude --skill` flag was named in three consecutive team OKRs (loops 2026-05-10-2227-002 / 2026-05-16 / 2026-05-23) as the dispatch invocation shape for AE's codegen skill. AE caught at build time that the local CLI doesn't expose it; the canonical form is `claude --bare --plugin-dir <plugin> --print "/<plugin>:<skill> {json}"`. The OKRs were authored against ideal-shape and verified only at build time, which means three loops of downstream consumers planned against a prediction.

ADR-2026-05-16-003 (ratified this loop) requires contracts to include a Verified-against-environment subsection. OKRs are not subject to the same discipline.

## Decision

Team-planning ritual gains an explicit predicted-syntax discipline. When an OKR (CEO brief or team OKR) mentions a CLI flag, a file path, or an environment variable that doesn't yet exist on the team's working branch — i.e., the author is reasoning about an *intended* shape rather than an *observed* one — the reference MUST be wrapped in a `<predicted: ...>` marker. The marker is a literal text inclusion in the OKR body; example shapes:

- `<predicted: claude --skill <name>>`
- `<predicted: scripts/publish-to-gitbook.py --incremental>`
- `<predicted: env var RESINK_TENANT_ID>`

The marker travels with the OKR through team-planning, into the build phase, and is the planning author's promise that the syntax has not been ground-truthed against an actual environment.

Two ritual edits land alongside this ratification:

1. **`org-os/rituals/team-planning.md`** gains a new step (between current steps 3 "Break key results into tasks with owners" and 4 "Flag cross-team asks") instructing planners to scan their drafted KRs for any reference to a CLI flag, file path, or env var that is not yet present on the team's working branch, and wrap each such reference in a `<predicted: ...>` marker. Acceptance gains a corresponding line: "Every reference to a CLI flag, file path, or env var that does not exist on the working branch at planning time is wrapped in a `<predicted: ...>` marker."

2. **`org-os/rituals/build.md`** step 3 ("Surface blockers inline") gains an explicit clause: when a build IC encounters a `<predicted: ...>` marker in the OKR, the IC verifies the syntax against the actual environment. On match, the IC drops the marker (the reference is now confirmed). On mismatch, the IC files an addendum (one bullet under the task) tagged "predicted-but-divergent" naming the actual shape; the addendum is recorded inline rather than silently corrected, so consolidation can count divergences across loops.

**Grandfathering.** OKRs filed before this ADR's `status: active` date (i.e., the 2026-06-13 ratification loop) do not need retroactive marker insertion. Future OKRs comply.

**Marker shape rationale.** The angle-bracket `<predicted: ...>` form is chosen because (a) it is unmistakable at human reading time, (b) it parses cleanly under any future frontmatter-lint script (DevOps's deferred work), and (c) it does not interfere with markdown rendering of normal text or backticked code spans inside the marker.

## Alternatives considered

- **A: Status quo (verify at build).** Rejected because the pattern recurred three loops with the same syntax error before being caught publicly.
- **B: Pre-verify every reference at planning time (CI-style validation).** Rejected as too heavy for current org-os scale; markers are lighter than full validation.
- **C: Extend ADR-2026-05-16-003 to cover OKRs as well as contracts.** Rejected because contracts and OKRs have different audiences (contracts: cross-team consumers; OKRs: same-team build phase); the verification surfaces should be similar but distinct.

## Consequences

- **Positive:** Predicted-syntax patterns surface at planning-review time rather than build time. Downstream teams (consuming the OKR's cross-team asks) see "predicted" markers and don't plan against unverified syntax. Three loops of `claude --skill` confusion don't recur.
- **Negative / costs:** OKR authoring grows a small discipline (mark predicted references). Build phase grows a small step (resolve markers or file addenda).
- **Follow-ups required:** Update `org-os/rituals/team-planning.md` + `org-os/rituals/build.md`. Sister to ADR-2026-05-16-003 (contracts); together they govern the "verify against actual environment" discipline.

## Links

- Triggering retro: [board/retros/2026-05-11-1302-ceo-retro.md](../retros/2026-05-11-1302-ceo-retro.md) — P2.
- Sister ADR: [2026-05-16-003-contract-environment-verification](2026-05-16-003-contract-environment-verification.md) — same discipline, different artifact type.
- Worked-example failure: three loops of `claude --skill` prediction culminated in AE's DISPATCH.md addendum at loop 2026-05-11-1302.
{% endraw %}
