---
layout: default
title: platform-agent-engineering OKR — 2026-05-14-0857
date: 2026-05-14
status: active
type: okr
loop: 2026-05-14-0857
owner: teams/platform/agent-engineering
grand_parent: Loops
parent: Loop 2026-05-14-0857
nav_order: 12
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/agent-engineering
  date: 2026-05-14
  status: active
  loop: 2026-05-14-0857
  links: parent: board/okrs/2026-05-14-0857-ceo-brief.md
-->
{% raw %}

# Agent Engineering OKR — 2026-05-14 (loop 2026-05-14-0857)

## Context

Light loop for AE. General-tables Phase 1 was re-shaped to a contract-first split: resink-core leads the NodeCtx C-ABI callback contract, AE co-authors. AE owns `lib.rs.tmpl` (the node side of the C-ABI), so AE's co-authorship verifies the contract is implementable against the template's `nanofab_node_process` shape before resink-core ratifies it. AE's actual template change is **Phase 1b (next loop)** — not this loop.

## Objectives

### O2: Co-author + ack the NodeCtx C-ABI callback contract

source: ceo-brief

Why it matters: A contract resink-core writes alone, that AE later finds un-implementable against `lib.rs.tmpl`'s `nanofab_node_process` shape, fails at Phase 1b. AE's co-authorship this loop is the cheap insurance — and AE names any template-imposed shape constraint before the contract is ratified.

Maps to brief O2 → KR2.1 (review for implementability), KR2.2 (co-author confirmation), KR2.3 (Phase 1b deliverable named), KR2.4 (tenant-isolation; template untouched).

**Key results**

- KR2.1: Review the draft `teams/application/resink-core/contracts/2026-05-14-nodectx-cabi-callback.md` against `lib.rs.tmpl`'s `nanofab_node_process(node_ptr, event_json, event_json_len, ctx_ptr) -> i32` shape. Verify the node can: receive the callback table via `ctx_ptr`, invoke `get_current` / `close_current` / `append_current` during `process`, propagate a `NodeError`.
- KR2.2: Append a one-paragraph co-author confirmation to the contract naming any shape constraint the template imposes — e.g. callback table by-pointer vs. by-value, key/row JSON vs. `repr(C)` buffer, how `NodeError` survives the `extern "C"` boundary.
- KR2.3: AE's exec summary records the Phase 1b template-change as a named, accepted next-loop deliverable — extend `lib.rs.tmpl`'s `nanofab_node_process` to wire `ctx_ptr` per the ratified contract — with a sized estimate.
- KR2.4: Tenant-isolation invariant holds. `lib.rs.tmpl` is NOT changed this loop; AE's edits are limited to the contract file + this OKR + AE's exec summary.

**Tasks**

- [ ] Read the draft contract + cross-check against `lib.rs.tmpl` lines ~446-600 (`nanofab_node_process` + `DefaultProcessCtx`)
- [ ] Verify the three callbacks + `NodeError` propagation are implementable against the template's `process` shape
- [ ] Append the co-author confirmation paragraph to the contract (note any template-imposed constraint)
- [ ] Record the Phase 1b `lib.rs.tmpl` `ctx_ptr` wiring as a named next-loop deliverable in AE's exec summary, with a sized estimate

## What success looks like

The NodeCtx C-ABI callback contract is co-authored by AE and confirmed implementable against `lib.rs.tmpl`. Any template-imposed shape constraint is named in the contract before ratification. AE's Phase 1b template-change deliverable is on the books.

## Out of scope

- Changing `lib.rs.tmpl` — that is the Phase 1b deliverable.
- Any codegen template work beyond reviewing the contract for implementability.
{% endraw %}
