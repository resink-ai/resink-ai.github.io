---
layout: default
title: "ADR 2026-05-16-003: contract environment verification"
date: 2026-05-16
status: active
type: adr
owner: board
---

<!-- original-frontmatter:
  type: adr
  owner: board
  date: 2026-05-16
  status: active
  decision: "Every cross-team contract or RFC artifact must include a 'Verified-against-environment' subsection naming toolchain versions, OS, runtime dependencies, and auth-mode prerequisites it was exercised against; new contracts comply going forward, existing contracts grandfather"
-->
# ADR 2026-05-16-003: Contract-Against-Environment Verification

## Context

During the 2026-05-16 MVP-focus loop, three cross-team contracts shipped with idealized-environment claims that did not survive consumer-side reality:

1. **AE's DISPATCH.md** prescribed `claude --bare` for determinism; the local `claude 2.1.138 --bare` enforces env-only auth and refuses to read the user's OAuth keychain. Resink-core's orchestrator dropped `--bare` mid-build.
2. **Sim-farm's verdict contract** implicitly promised "no extra installs" but the DuckDB Python driver requires `pytz` at runtime for TIMESTAMPTZ. Sim-farm caught it mid-build and added the dep to its `pyproject.toml`.
3. **Three planning artifacts** (AE OKR, resink-core OKR, CEO brief) referenced `claude --skill <name>` as the dispatch shape — a flag that does not exist in the local CLI. AE caught it during build authoring.

Pattern: contracts were specified against the spec, not the developer-laptop environment they were exercised in. Consumer teams discovered every gap mid-build. The contract authors had no per-loop discipline forcing them to verify against the runtime environment before declaring the contract `status: active`. The cost lands on consumer-team bandwidth at the worst possible moment — after the contract has been published, after the consumer has begun integration.

## Decision

Update `org-os/conventions.md` to require every cross-team contract or RFC artifact (any file with `type: contract` or `type: rfc` whose body documents a producer/consumer interface) to include a top-level "Verified-against-environment" subsection naming:

- **Toolchain versions** the contract was exercised against (`cargo --version`, `rustc --version`, `python --version`, `claude --version`, `helm version`, `duckdb --version`, etc., as applicable).
- **OS + architecture** (e.g., `macOS 24.6.0 arm64`, `Ubuntu 22.04 x86_64`).
- **Runtime dependencies** the contract assumes are available — including transitive ones that may not be obvious (system libraries, Python sub-deps like `pytz`, environment-specific binaries).
- **Auth-mode prerequisites** the consumer must satisfy (env vars, OAuth, service accounts, etc.) and any failure modes if those prerequisites are not met.

Consumer teams read this section first; if their environment differs, they surface the delta in their OKR's cross-team asks BEFORE consuming the contract. The build ritual gains an implicit step: a consumer team validates that its environment matches the contract's Verified-against-environment section, and if not, surfaces the delta mid-loop rather than discovering it through a build failure.

**Scope and grandfathering.** This requirement applies to:
- All NEW cross-team contracts or RFCs filed on or after this ADR's `status: active` date (2026-05-30).
- All cross-team contracts or RFCs whose body is materially revised after that date (a Verified-against-environment subsection lands alongside the revision).

Existing contracts filed before 2026-05-30 grandfather: they are not retroactively required to add the subsection this loop. Owners MAY add it opportunistically as part of unrelated edits; they are NOT obligated to open a same-loop addendum solely to satisfy this ADR.

The placement: `org-os/conventions.md` (rather than `org-os/rituals/build.md`) because the requirement is a frontmatter-and-body shape constraint on a specific `type`, not a step in the build ritual. The conventions file already hosts the "Additional fields per type" table and the validation hints section; the Verified-against-environment requirement extends that pattern.

## Alternatives considered

- **A: Lint contracts against a canonical environment in CI.** Rejected as too heavy for current org-os scale; the developer-laptop environment is the canonical environment until production CI exists. A future frontmatter-lint script (DevOps's deferred work) could check for the subsection's presence without prescribing its contents.
- **B: Status quo (consumer discovers in build).** Rejected because the cost — consumer-team bandwidth burned mid-build on environment deltas that the contract author already knew about — recurred across three contracts in one loop.
- **C: Require executable acceptance tests in every contract.** Rejected as too prescriptive for early-loop contracts; the Verified-against-environment subsection is the minimal viable discipline. Executable tests remain a per-contract author choice.
- **D: Land in `org-os/rituals/build.md` as a consumer-side checklist.** Rejected because the obligation lies with the contract author (producer), not the consumer. A consumer-side checklist would push the cost back onto the wrong actor and would not prevent gaps from being introduced in the first place.

## Consequences

- **Positive:** Cross-team contracts gain a verifiable environment-baseline. Consumer teams have a one-paragraph read-first surface that pre-flights compatibility. Patterns like the `--bare` deviation and the `pytz` dependency become discoverable at contract-read time, not build time. The conventions enum gains a body-shape rule for contract-class artifacts, paving the way for future frontmatter-lint coverage.
- **Negative / costs:** Contract authors do one extra section of work per contract — typically one paragraph or one short bullet list. The Verified-against-environment section can drift if the author's environment changes without a contract addendum; owners are responsible for keeping it accurate on material revision.
- **Follow-ups required:**
  - Update `org-os/conventions.md` with the new body-shape rule (lands alongside this ADR's ratification).
  - When the DevOps frontmatter-lint script ships (deferred per ADR-2026-05-08-001), extend it to assert presence of the subsection on `type: contract` artifacts produced on or after 2026-05-30.
  - Pair with [ADR-2026-05-10-003](2026-05-10-003-verify-state-claims-at-ritual-transitions.md) (verify-state-claims) — both target the same conventions/ritual surface; both ratified.

## Links

- Triggering retro: [board/retros/2026-05-11-0958-ceo-retro.md](../retros/2026-05-11-0958-ceo-retro.md) — P3.
- Sister ADR: [2026-05-10-003-verify-state-claims-at-ritual-transitions](2026-05-10-003-verify-state-claims-at-ritual-transitions.md) (same family, different scope).
- Companion ADR ratified same loop: [2026-05-16-004-focus-loop-pattern](2026-05-16-004-focus-loop-pattern.md).
- Conventions edit landed alongside this ratification: [org-os/conventions.md](../../org-os/conventions.md).
- Worked-example contracts (grandfathered; not retroactively edited): AE DISPATCH.md, sim-farm verdict contract, DE in-memory event-source contract — all filed 2026-05-16.
