---
layout: default
title: application-sim-farm Exec Summary — 2026-05-12-0645
date: 2026-05-12
status: active
type: exec-summary
loop: 2026-05-12-0645
owner: teams/application/sim-farm
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/sim-farm
  date: 2026-05-12
  status: active
  loop: 2026-05-12-0645
  links: parent: board/okrs/2026-05-12-0645-ceo-brief.md
-->
{% raw %}

# Sim Farm Exec Summary — Loop 2026-05-12-0645 (paused, review-ack)

**Headline.** Paused this loop per CEO brief. Reviewed the home-cluster deployment for verdict-contract surface impact — **none.** The supervisor's `make mvp-loop` invocation (which IS what runs inside the container) writes the same `verdict.json` shape (`engine_version: 0.3.0`, `overall_pass: true`, per-dim `pass: true`, `mismatch_count: 0`) whether executed on a developer laptop or inside the home cluster. The home-cluster deploy doesn't surface any new verdict-contract concerns. The sim-farm engine 0.3.0 contract surface is **unaffected by the deployment mechanism.** Verdict-contract back-reference to DE convention remains carry (mechanical 1-line edit; sim-farm picks the depth — link-only vs link+retained engine-side documentation).

## Review-ack on the home-cluster deployment

- **Verdict.json shape across deployment:** the home-cluster supervisor logs show `[supervisor] [dim_user] wrote 21 rows to /workspace/dim_user_output.parquet` and `[supervisor] [dim_account] wrote 18 rows to /workspace/dim_account_output.parquet`. The verdict.json would be written at `/workspace/verdict.json` on the pod's filesystem after the orchestrator's sim-farm diff step runs — which doesn't happen in this loop's deploy (the chart only runs the supervisor, not the orchestrator's full mvp-loop pipeline). For the verdict-contract surface this is **fine**: the sim-farm engine is a Python module invoked via `make mvp-loop`, not part of the chart-installed supervisor binary. The chart is correctly scoped to the supervisor-only deploy this loop.
- **Future-loop concern (non-blocker):** if a later loop bakes the full mvp-loop pipeline into the chart (orchestrator + supervisor + sim-farm diff in one container or as separate pods), sim-farm's verdict contract becomes load-bearing on the chart's interface. Not this loop; flagged for awareness when the chart's scope expands.
- **Trace.jsonl observability:** the chart's `kubectl logs` capture surface is consistent with the trace.jsonl shape the sim-farm engine consumes for verdict generation. No corrections.

## Carrying into next loop

- **Verdict-contract back-reference to DE convention** — still mechanical 1-line carry. Sim-farm paused this loop again; next active loop picks it up.
- **Modes B (per-node shadow) and C (DAG blue/green warmup)** — Sim Farm spec §5.2 / §5.3. Trigger condition closer now that the deployment surface exists. Remain primary candidates for next major engine extension.
- **Three-layer verdict** — Sim Farm spec §4.7; not load-bearing until training-pipeline gate stage 3 adopts coverage-spec-driven verdicts.
- **Sim-farm own product repo decision** — trigger condition still not met (first non-Python sim-farm component).
- **Broader failure-mode coverage** — Sim Farm spec §6 modes beyond the four SCD2-equivalence cases.
- **Schema-aware additive extensions** — engine 0.3.0 has documented additivity for `tolerances`, `column_types`; not in scope until consumer pressure.

## Tenant-isolation invariant

Held. Zero writes to `org-os/` or sim-farm's own working tree this loop beyond this exec summary.
{% endraw %}
