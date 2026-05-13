---
layout: default
title: "Team: platform-agent-engineering"
---
{% raw %}
# Team: platform-agent-engineering

## Mission

Build and maintain the AI agent infrastructure for the org. Produces the agent runtime, plugin marketplace, and internal libraries that let the company itself run autonomously.

## Owned products

- Plugin marketplace — shared agentic plugins compatible with Claude Code, Codex CLI, Gemini CLI, OpenCode.
- Internal agent library — common tools, prompts, and orchestration primitives used across teams.
- Agent runtime conventions — the contract between `org-os/templates/agent-spec.md` and runnable agents.
- Nanofab codegen plugin (`repos/resink-ai/resink-marketplace/plugins/nanofab/`) — pattern-library skills the training-pipeline orchestrator dispatches; pattern set tracked against training spec §4.10.

## Interfaces

**Consumes:** nothing from other teams in Phase 1; in later phases, observability from SRE.
**Produces:** agent runtime + libraries used by every other team.

## Success metrics

- Every other team can spawn a Phase 2 agent using only `org-os/playbooks/spawn-agent.md` and AE-provided libraries.
- Plugin marketplace has at least one shared plugin used by 2+ teams.

## Decision rights

- Decides unilaterally on: plugin marketplace acceptance criteria; agent runtime API.
- Must escalate: changes to `org-os/templates/agent-spec.md` (org-os change requires CEO + ADR).

## Out of scope

- Product code (application layer's job).
- Cloud infra (DevOps).

## Executive summaries

- [2026-05-12-1826](../../loops/2026-05-12-1826/teams/platform-agent-engineering-exec-summary.html)
- [2026-05-12-1254](../../loops/2026-05-12-1254/teams/platform-agent-engineering-exec-summary.html)
- [2026-05-12-0645](../../loops/2026-05-12-0645/teams/platform-agent-engineering-exec-summary.html)
- [2026-05-11-2153](../../loops/2026-05-11-2153/teams/platform-agent-engineering-exec-summary.html)
- [2026-05-11-1631](../../loops/2026-05-11-1631/teams/platform-agent-engineering-exec-summary.html)
- [2026-05-11-1302](../../loops/2026-05-11-1302/teams/platform-agent-engineering-exec-summary.html)
- [2026-05-11-1113](../../loops/2026-05-11-1113/teams/platform-agent-engineering-exec-summary.html)
- [2026-05-11-0958](../../loops/2026-05-11-0958/teams/platform-agent-engineering-exec-summary.html)
- [2026-05-10-2227-002](../../loops/2026-05-10-2227-002/teams/platform-agent-engineering-exec-summary.html)
- [2026-05-10-2227-001](../../loops/2026-05-10-2227-001/teams/platform-agent-engineering-exec-summary.html)
- [2026-05-09-1715](../../loops/2026-05-09-1715/teams/platform-agent-engineering-exec-summary.html)

## OKRs

- [2026-05-11-1631](../../loops/2026-05-11-1631/teams/platform-agent-engineering-okr.html)
- [2026-05-11-1302](../../loops/2026-05-11-1302/teams/platform-agent-engineering-okr.html)
- [2026-05-11-1113](../../loops/2026-05-11-1113/teams/platform-agent-engineering-okr.html)
- [2026-05-11-0958](../../loops/2026-05-11-0958/teams/platform-agent-engineering-okr.html)
- [2026-05-10-2227-002](../../loops/2026-05-10-2227-002/teams/platform-agent-engineering-okr.html)
- [2026-05-10-2227-001](../../loops/2026-05-10-2227-001/teams/platform-agent-engineering-okr.html)
- [2026-05-09-1715](../../loops/2026-05-09-1715/teams/platform-agent-engineering-okr.html)
{% endraw %}
