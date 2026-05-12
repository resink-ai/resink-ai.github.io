---
layout: default
title: "ADR 2026-05-30-001: verify predicted syntax in okrs"
parent: "Decisions (ADRs)"
render_with_liquid: false
date: 2026-05-30
status: draft
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-30
  status: draft
  decision: "Team OKRs that reference predicted-but-unverified syntax (CLI flags, file paths, env vars) must carry an explicit verification marker; the build phase confirms or files an addendum"
-->
# ADR 2026-05-30-001: Verify Predicted Syntax in OKRs at Planning Time

## Context

The `claude --skill` flag was named in three consecutive team OKRs (loops 2026-05-10-2227-002 / 2026-05-16 / 2026-05-23) as the dispatch invocation shape for AE's codegen skill. AE caught at build time that the local CLI doesn't expose it; the canonical form is `claude --bare --plugin-dir <plugin> --print "/<plugin>:<skill> {json}"`. The OKRs were authored against ideal-shape and verified only at build time, which means three loops of downstream consumers planned against a prediction.

ADR-2026-05-16-003 (ratified this loop) requires contracts to include a Verified-against-environment subsection. OKRs are not subject to the same discipline.

## Decision

(Placeholder — to be expanded loop 2026-05-11-2153.) Update `org-os/rituals/team-planning.md` step 3 ("Break key results into tasks with owners") or step 4 ("Flag cross-team asks"): when an OKR mentions a CLI flag, file path, or environment variable that doesn't yet exist on the team's working branch, the OKR carries an explicit "not yet verified — predicted shape" marker (e.g., `<predicted: claude --skill <name>>` in the OKR text). The build phase converts the marker into a confirmed reference (drop the marker) or files an addendum (replace with the actual shape).

Update `org-os/rituals/build.md` step 3 ("Surface blockers inline"): predicted-syntax markers that fail at build time are recorded as "predicted-but-divergent" addenda rather than silent corrections.

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
