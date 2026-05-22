---
layout: default
title: Loops
has_children: true
nav_order: 2
---
{% raw %}
# Loops

Chronological list of loops. Each loop links to its CEO brief, executive summary, retro, and per-team artifacts.

| Date | Headline |
|---|---|
| [2026-05-15-0001](2026-05-15-0001/) | **Headline.** First loop on 2026-05-15. General-tables **Phase 1b** shipped (re-shaped) — the NodeCtx C-ABI callback contract wired across both sides, bridge proven end-to-end on **real FFI**. AE wired `lib.rs.tmpl` (`nanofab_node_process` accepts `ctx_ptr`; `NanofabNodeCtxVTa... |
| [2026-05-14-0857](2026-05-14-0857/) | **Headline.** Second loop on 2026-05-14. General-tables Phase 1 — and a first-contact probe reshaped it before a line of code was written. The probe found the ADR's "dlopen C-ABI" Phase 1 needs an AE codegen-template change (`ctx_ptr` is unwired in `lib.rs.tmpl`) — so Phase 1... |
| [2026-05-14-0742](2026-05-14-0742/) | **Headline.** First loop on a new calendar date (the prior seven all landed 2026-05-13). Design-first architectural loop: opened a **third** architectural arc with [ADR-2026-05-14-001](../decisions/2026-05-14-001-general-tables-rearchitecture.md), naming the supervisor's hardc... |
| [2026-05-13-1944](2026-05-13-1944/) | **Headline.** Seventh same-day loop. Step 2 of ADR-2026-05-13-001's four-step streaming restoration plan shipped — sized L when CEO pulled the `Supervisor` API extraction forward from step 4. Six interlocked deliverables in one resink-core build session: `InMemoryStream` impl,... |
| [2026-05-13-1844](2026-05-13-1844/) | **Headline.** Sixth same-day loop. Single team active (resink-core); single objective; step 1 of ADR-2026-05-13-001's four-step streaming restoration plan shipped clean in one build session. `EventSource` trait + `ParquetReplay` impl + supervisor wired through trait; `make mvp... |
| [2026-05-13-1422](2026-05-13-1422/) | **Headline.** Fifth loop on the same calendar date. Design-first architectural loop: filed [ADR-2026-05-13-001](../decisions/2026-05-13-001-streaming-event-source-rearchitecture.md) naming the current parquet-eager event-source as a deliberate MVP deviation from the runtime sp... |
| [2026-05-13-1303](2026-05-13-1303/) | **Headline.** Fourth loop on the same calendar date. After three product loops (org-os ratification → observability skill + hot-swap AE-side → resink-core bundle close-out), this loop closes the showcase gap: the work on disk is now substantial enough to show somebody, and the... |
| [2026-05-13-1022](2026-05-13-1022/) | **Headline.** Three-objective resink-core bundle close-out — all shipped clean. **O1 closes ADR-2026-05-16-001's five-loop, three-step dlopen restoration plan.** New in-tree `crates/nanofab-plugin-dim-user-v2/` cdylib + `tests/hot_swap_correctness.rs` integration test exercise... |
| [2026-05-13-0859](2026-05-13-0859/) | **Headline.** Two AE-owned objectives both shipped clean. **O1: First operator-observability surface.** New `observability` marketplace plugin + `cluster-snapshot` skill v1 — a stdlib-Python generator that wraps kubectl and emits a self-contained HTML cluster snapshot (inline... |
| [2026-05-13-0056](2026-05-13-0056/) | **Headline.** Org-os ratification bundle shipped — 2 draft ADRs flipped to active alongside their newly-authored playbooks; 1 ADR's scope extended in-place (V-a-E discipline now admits `runbook` + `convention` + `playbook` alongside `contract` + `rfc`); 1 in-place sibling sect... |
{% endraw %}
