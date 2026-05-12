---
layout: default
title: platform-agent-engineering OKR — 2026-05-16
nav_exclude: true
render_with_liquid: false
date: 2026-05-16
status: active
type: okr
loop: 2026-05-11-0958
owner: teams/platform/agent-engineering
---

<!-- original-frontmatter:
  type: okr
  owner: teams/platform/agent-engineering
  date: 2026-05-16
  status: active
  loop: 2026-05-16
  links: parent: board/okrs/2026-05-11-0958-ceo-brief.md
-->
# Agent Engineering OKR — 2026-05-16

## Context

This loop is the company's MVP-focus loop. The CEO brief ([`board/okrs/2026-05-11-0958-ceo-brief.md`](../../../../board/okrs/2026-05-11-0958-ceo-brief.md)) makes the binding decision that AE's previously-floored Bundle B (the seven `org-os/rituals/*` edits carried from 2026-05-10) is **deferred to loop 2026-05-11-1113** so AE's full bandwidth lands on O3: a Claude Code skill plus subagent-dispatch contract that the resink-core training-pipeline orchestrator (brief O1 KR1.3) can call to generate Rust source for one SCD2-maintainer node bound to the synthetic `dim_user` fixture.

Two consecutive loops of Bundle B deferral (2026-05-10's stretch slip → 2026-05-16's CEO-driven pivot) is now a pattern. AE flags it explicitly here for retro consideration this loop, without complaint: the deferrals had different causes (mid-loop ADR-gate slip then; explicit MVP-focus call now), but the cumulative carry is real. The retro should decide whether Bundle B becomes a 2026-05-23 hard floor or whether re-slicing is warranted.

The codegen scope this loop is deliberately narrow: **one pattern (`scd2_maintainer`), one dim (`dim_user`), one fixture**. Per training spec [§4.10](../../../../docs/superpowers/specs/2026-05-10-nanofab-training-pipeline-design.md#410-pattern-library-nanofab-node-patterns) the pattern is `scd2_counter_maintainer`-shaped; per [§4.1](../../../../docs/superpowers/specs/2026-05-10-nanofab-training-pipeline-design.md#41-orchestrator-nanofab-trainer-orchestrator) the orchestrator is the dispatch caller; per runtime spec [§5.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-runtime-design.md#54-watermark-advancement) the event shape is fixed. AE therefore over-invests in the **dispatch contract** (the cross-team artifact resink-core consumes), and treats the codegen prompt itself as something to iterate until `cargo build --release` exits 0.

## Decisions made this loop (recorded here so resink-core can plan against them)

- **Skill location and name.** `repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/SKILL.md`. The `nanofab` plugin is new (sibling to existing `agent-forge` and `org-os` plugins); AE creates it this loop with this single skill inside. Choosing a fresh plugin keeps nanofab-specific skills from polluting `agent-forge` (skill-authoring) or `org-os` (rituals); future codegen patterns (`scd1_first_event`, `window_stats_with_decrement`, etc.) drop in as sibling skill directories under the same plugin.
- **Dispatch mechanism.** Local `claude` CLI invocation, wrapped by a thin Markdown contract spec (no Python framework). Justification: resink-core's orchestrator is itself a long-running agent process per training spec §4.1; spawning a child `claude` invocation with the skill name and the JSON inputs is the smallest dispatch surface that preserves the training-spec invariant "sub-agents are stateless and replayable" (§3.2). A Python wrapper would be premature abstraction for one skill, one consumer; CLI invocation is also what every other skill in this marketplace already uses.
- **Codegen output type.** **Source-only.** The skill writes `lib.rs` + `Cargo.toml` to `output_dir`; resink-core (or the MVP `make mvp-loop` driver) is responsible for `cargo build --release`. Justification: the brief's KR3.3 acceptance is "the produced crate compiles," which AE verifies in O1 KR1.4 by running `cargo build --release` itself during smoke-test, but the runtime artifact (`.so`) is a resink-core concern downstream. Source-only keeps the skill's outputs deterministic-by-content (same prompt → same `lib.rs` bytes → same crate hash) and aligns with training spec §4.7 where `node-codegen` always emits the crate, not the compiled artifact.
- **Codegen approach.** **Template-with-LLM-fill is the primary approach this loop**, not pure-LLM codegen. The brief's risk section explicitly permits template-with-fill as a non-stretch pivot; AE adopts it from the start. Reasoning: (a) the SCD2-maintainer pattern is well-defined enough that a hand-authored Rust skeleton with named slots is cheap to write, (b) the LLM only needs to fill in column names, types, and the SCD2 column triple per the input schema, (c) `cargo build` reliability goes from "iterate the prompt" to "iterate the template once, then trust the slot-fill," and (d) future patterns reuse the same scaffolding shape. Pure-LLM remains a fallback if the template proves too rigid for `dim_user`'s shape (it should not — the fixture is single-tenant, single-shard, no cross-key dependencies).

## Objectives

### O1: Ship the `nanofab-codegen-scd2` skill and dispatch contract that closes resink-core's O1 KR1.3

source: ceo-brief

Why it matters: Brief O1 KR1.3 explicitly names AE's deliverable as the gate on the training-pipeline half of the MVP loop. Without a working skill the orchestrator can call, resink-core cannot emit `manifest.yaml` + `lib.rs` for `dim_user_scd2`, and the closed-loop test (`make mvp-loop`) cannot run. This objective is the entire AE contribution to MVP closure. Maps to brief O3 → KR3.1, KR3.2, KR3.3, KR3.4.

**Key results**
- KR1.1: `repos/resink-ai/resink-marketplace/plugins/nanofab/` exists as a self-contained plugin per the marketplace conventions in `repos/resink-ai/resink-marketplace/CLAUDE.md` ("Adding a new plugin"). Specifically: `.claude-plugin/plugin.json`, `skills/codegen-scd2-node/SKILL.md`, registration in the marketplace root's `.claude-plugin/marketplace.json`, and a row in `.version-bump.json`. `bash tests/run-all.sh` exits 0 against the new plugin's manifest validation.
- KR1.2: `plugins/nanofab/skills/codegen-scd2-node/SKILL.md` exists and authoritatively documents the skill contract per the **Skill contract** section below. The skill takes `(schema_json, pattern_name, output_dir)` as inputs, produces `lib.rs` + `Cargo.toml` under `<output_dir>/`, returns a status (`ok` | `error: <message>`).
- KR1.3: A Rust template with named slots lives under `plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/` containing at minimum `lib.rs.tmpl` and `Cargo.toml.tmpl`. The template implements the SCD2-maintainer pattern over events of the runtime spec §5.4 shape: on each `insert` event, append a row with `valid_from = event_ts`, `valid_to = NULL`, `is_current = true`, and previous-current row's `valid_to = event_ts`, `is_current = false`; on each `update` event, same. The template is hand-authored Rust (no LLM), with `{{column_name}}`-style slots filled by the skill's slot-fill stage.
- KR1.4: Smoke-test on the MVP fixture's `dim_user` schema passes. Acceptance: AE invokes the skill with the `dim_user` schema (from a stub `schema_json` AE writes locally — does not require resink-core's fixture to land first) and runs `cargo build --release` in the produced crate directory; exit code is 0. Recorded in this OKR's exec summary with the exact command and exit-code.
- KR1.5: A dispatch-contract Markdown spec lives at `plugins/nanofab/skills/codegen-scd2-node/DISPATCH.md` and authoritatively documents how resink-core's orchestrator invokes the skill: the exact `claude` CLI invocation shape, the input JSON schema, the output filesystem layout, the status reporting convention, the error semantics. This is the single artifact resink-core reads to integrate; it is referenced verbatim by AE's cross-team ask to resink-core (below).
- KR1.6: Tenant-isolation invariant holds. Acceptance: `grep -rEi "resink|nanofab|acme\.ai" /Users/shijinglu/Workspace/resink.ai/newbase/org-os/` returns no real tenant/product names introduced by this loop's edits. (No `org-os/` edits are in scope this loop, but the discipline is reaffirmed and the dry-run is recorded.)

**Tasks**
- [x] Scaffold the `nanofab` plugin per `repos/resink-ai/resink-marketplace/CLAUDE.md` "Adding a new plugin" — owner: agent-engineering/ic-marketplace-1
  - Created `plugins/nanofab/` with all four per-platform manifests: `.claude-plugin/plugin.json`, `.codex-plugin/plugin.json` (with `interface{}`), `gemini-extension.json` + `GEMINI.md`, `package.json` + `.opencode/plugins/nanofab.js`. Modeled on the `org-os` plugin's shape; same author/license/version (`0.1.0`).
- [x] Run `bash scripts/scaffold-skill.sh --plugin nanofab codegen-scd2-node` (or hand-author equivalent) — owner: agent-engineering/ic-marketplace-1
  - Hand-authored the skill directory at `plugins/nanofab/skills/codegen-scd2-node/` with full `SKILL.md`, `DISPATCH.md`, `SMOKE.md`, and `templates/scd2_maintainer/{Cargo.toml,lib.rs}.tmpl`. Skipped the scaffold script because the placeholder `SKILL.md` it emits would have been overwritten anyway.
- [x] Hand-author `templates/scd2_maintainer/lib.rs.tmpl` against runtime spec §5.4 event shape — owner: agent-engineering/ic-codegen-1
  - Wrote ~500-line template implementing the SCD2-maintainer pattern over `Event { table, op: Insert|Update|Delete, event_ts, event_id, before, after }`. Inlines `Node` trait + `NodeCtx` + `NodeError` + `Op` + `FieldValue` since the upstream `nanofab-node-abi` crate is not yet stable per risk #5. Exports `nanofab_node_new` / `nanofab_node_drop` C-ABI symbols for `dlopen` per runtime spec §4.3. Slot vocabulary documented in SKILL.md "Slot-fill table" section (`crate_name`, `table`, `key_struct_name`, `row_struct_name`, `node_struct_name`, `node_version`, `pk_field_decls`, `non_pk_field_decls`, `key_field_extractions`, `row_field_extractions`, `scd2_valid_from`, `scd2_valid_to`, `scd2_is_current`).
- [x] Hand-author `templates/scd2_maintainer/Cargo.toml.tmpl` (cdylib, runtime ABI deps per resink-core's published node-ABI crate name — confirm via cross-team ask below) — owner: agent-engineering/ic-codegen-1
  - Wrote cdylib manifest with `crate-type = ["cdylib"]` and **zero external dependencies** for the MVP — the node-ABI surface is inlined in `lib.rs` (per risk #5 fallback). Cargo.toml.tmpl carries an explicit comment naming the future migration path (`nanofab-node-abi = "0.1"` once resink-core publishes it).
- [x] Author SKILL.md body (instructions to the LLM for the slot-fill stage; what schema fields map to which slots; failure modes) — owner: agent-engineering/ic-codegen-1
  - SKILL.md authoritatively documents: contract summary, full input/output shapes, the LLM workflow (validate → compute slots → slot-fill → verify no `{{` remains → write `STATUS.json`), the slot-fill table, the column-type → Rust-type mapping, a fully-worked `dim_user` example, and a failure-mode catalog. Frontmatter uses `name:` + `description:` per existing skill convention.
- [x] Author DISPATCH.md (the cross-team contract) — owner: agent-engineering/ic-codegen-2
  - DISPATCH.md is the authoritative cross-team artifact for resink-core's orchestrator. Pinned against `claude 2.1.138` / `cargo 1.95.0`. Documents the canonical invocation form, input JSON shape, output filesystem layout, `STATUS.json` schema, determinism guarantee (for content-hash dedup), acceptance contract (STATUS=ok AND `cargo build --release` exits 0), full error catalog, and versioning policy (plugin version is the contract version).
- [x] Smoke-test: invoke skill on stub `dim_user` schema, run `cargo build --release`, record exit code — owner: agent-engineering/ic-codegen-2
  - Hand-filled the template into `/tmp/nanofab-codegen-smoke-2026-05-16/` against the MVP `dim_user` schema (PK `user_id: string`, non-PK `email: string`, `country: Option<String>`). `cargo build --release` exit 0 (compile time 1.81s, zero warnings, zero external deps). `cargo test --release` also exit 0 (1 test passed). Toolchain: `cargo 1.95.0 (Homebrew)`, `rustc 1.95.0 (Homebrew)`. Full result recorded in `plugins/nanofab/skills/codegen-scd2-node/SMOKE.md`. Smoke output dir not committed per OKR instruction.
- [x] Update marketplace root `.claude-plugin/marketplace.json` with the new plugin entry — owner: agent-engineering/ic-marketplace-1
  - Appended `nanofab` as the third entry in `plugins[]` with `source: "./plugins/nanofab"`, `version: "0.1.0"`, matching author block.
- [x] Update `.version-bump.json` with the nanofab plugin row — owner: agent-engineering/ic-marketplace-1
  - Added `"nanofab"` entry with `marketplace_field: "plugins.2.version"` and the four per-platform file paths. `bash scripts/bump-version.sh --check nanofab` reports `nanofab: all versions match: 0.1.0`.
- [x] Run `bash tests/run-all.sh`; resolve any manifest-validation failures — owner: agent-engineering/ic-marketplace-1
  - `bash tests/run-all.sh` exit 0. Manifest validation: 70 pass / 0 fail / 2 skip (skips are nanofab having no hooks/hooks.json — expected, and the codex `plugin list` host-state-dependent skip). Claude-code skill-trigger and command-runs suites also passed. The pre-existing org-os "10 rit-* skills" hardcoded count is still satisfied (nanofab adds a separate `find` scope).
- [x] Run tenant-isolation dry-run; paste output into exec summary — owner: AE EM
  - `grep -rEi "resink|nanofab|acme\.ai" /Users/shijinglu/Workspace/resink.ai/newbase/org-os/` returned exactly one match: `org-os/conventions.md: placeholder names like 'acme.ai'.` — pre-existing, not introduced this loop. Tenant-isolation invariant holds; AE made zero edits under `org-os/` this loop.
- [ ] Confirm in mid-loop check-in (2026-05-19) that resink-core's orchestrator integration matches DISPATCH.md's contract; addendum if not — owner: AE EM
  - Deferred to mid-loop (2026-05-19) per task description; cannot be ticked in this build phase. DISPATCH.md is in place and ready for resink-core to read; check-in will follow on the named date.

### O2: Charter touch-up — record the nanofab-plugin ownership

source: ceo-brief

Why it matters: The charter currently lists only the marketplace and the agent runtime as owned products. The new `nanofab` plugin is a third owned surface; recording it in the charter prevents future drift over who maintains nanofab codegen patterns. This is a one-line edit, sized S, kept as a separate small objective so it is not silently dropped under O1's cadence.

Maps to brief O3 → implicit (charter discipline; the brief does not require this but the convention is to record new owned surfaces in the charter when they land).

**Key results**
- KR2.1: `teams/platform/agent-engineering/charter.md` "Owned products" section gains a third bullet: "Nanofab codegen plugin (`repos/resink-ai/resink-marketplace/plugins/nanofab/`) — pattern-library skills the training-pipeline orchestrator dispatches; pattern set tracked against training spec §4.10."

**Tasks**
- [x] Edit charter.md "Owned products" section; bump charter `date:` to 2026-05-16 — owner: AE EM
  - Added a fourth bullet under "Owned products" naming the nanofab codegen plugin and pointing at the marketplace path; bumped `date: 2026-05-08` → `date: 2026-05-16`. (Charter already had three bullets, not two — the OKR's "third bullet" framing predated the existing list; the nanofab line lands as the fourth bullet, which preserves the documented intent of recording the new owned surface.)

## Cross-team asks

- **From `teams/application/resink-core`, by mid-loop 2026-05-19:** publish the `dim_user` schema JSON from the MVP fixture (resink-core's brief O1 KR1.2 deliverable) at a stable path AE can pull (e.g., `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/fixtures/dim_user.schema.json`). AE's smoke-test (O1 KR1.4) blocks on this schema — AE will use a self-authored stub if it slips, but a real schema is required before resink-core's `make mvp-loop` end-to-end run.
- **From `teams/application/resink-core`, by mid-loop 2026-05-19:** confirm the runtime node-ABI crate name and version that the generated `Cargo.toml` depends on (the crate that exports the `Node` trait the supervisor `dlopen`s per runtime spec §4.3). AE needs the exact crate name + version pin to bake into `Cargo.toml.tmpl`. If the ABI crate is not yet on a stable name, resink-core publishes a placeholder crate (even an empty one with the trait declaration) by 2026-05-19 so the template can compile.
- **From `teams/application/resink-core`, by 2026-05-20 at the latest:** read AE's `DISPATCH.md` (under `plugins/nanofab/skills/codegen-scd2-node/DISPATCH.md`) and confirm in writing (a comment in resink-core's OKR mid-loop status update is sufficient) that the orchestrator's invocation shape matches. If the orchestrator wants a different shape, AE files an addendum to DISPATCH.md rather than letting the contracts diverge.
- **From `teams/platform/data-engineering`, no this-loop ask.** AE does not consume DE's in-memory event-source contract directly this loop; the skill's template hardcodes the runtime spec §5.4 event shape. If DE's addendum (brief O4 KR4.1) renames any field, AE updates the template after the addendum lands; the change is mechanical.
- **From `teams/application/sim-farm`, no this-loop ask.** Sim-farm's verdict format is not consumed by AE this loop; the codegen output is upstream of the diff.
- **From board / CEO, end-of-loop (informational):** acknowledge in CEO consolidation that the `nanofab` plugin landed and is the new home for nanofab-specific Claude Code skills, so future loops route nanofab pattern-library expansion to AE without ambiguity.

## Skill contract

This subsection is the authoritative dispatch surface resink-core's orchestrator consumes. The full canonical version lives in `plugins/nanofab/skills/codegen-scd2-node/DISPATCH.md` once the build phase ships it; this is the contract AE commits to.

### Invocation

The orchestrator dispatches the skill via a child `claude` CLI invocation:

```
claude --skill nanofab:codegen-scd2-node \
       --input '{"schema_json": <inline JSON or @file>, "pattern_name": "scd2_maintainer", "output_dir": "<absolute path>"}'
```

(Exact CLI flag names may shift — the canonical form is fixed in DISPATCH.md once the build phase confirms against the local `claude` CLI version. AE pins to the version on the developer-laptop critical path.)

### Inputs

- `schema_json` — JSON object with shape:
  ```json
  {
    "table": "dim_user",
    "primary_key": ["user_id"],
    "scd2_columns": {"valid_from": "valid_from", "valid_to": "valid_to", "is_current": "is_current"},
    "columns": [
      {"name": "user_id", "type": "string", "nullable": false},
      {"name": "email", "type": "string", "nullable": false},
      {"name": "country", "type": "string", "nullable": true}
    ]
  }
  ```
  AE accepts the column-type vocabulary as a small enum: `string`, `int64`, `float64`, `bool`, `timestamp_ms`. The MVP `dim_user` only needs `string`; AE implements the rest as template stubs that error cleanly at slot-fill time if hit.
- `pattern_name` — string. For the MVP: `"scd2_maintainer"`. The skill rejects other values with `error: unknown pattern <name>` and exits non-zero.
- `output_dir` — absolute filesystem path. The skill creates the directory if it does not exist. The skill is the sole writer of files under `output_dir` for the duration of the invocation.

### Outputs

Files written to `<output_dir>/`:
- `Cargo.toml` — cdylib crate manifest, `crate-type = ["cdylib"]`, dependencies include the runtime node-ABI crate (name + version per the cross-team ask above).
- `src/lib.rs` — Rust source implementing the SCD2-maintainer pattern over events of the runtime spec §5.4 shape, exporting the `Node` symbol the supervisor `dlopen`s per runtime spec §4.3.
- `STATUS.json` — `{"status": "ok"|"error", "message": "<optional>", "pattern": "scd2_maintainer", "table": "<table>"}`.

The skill exit code is 0 on `status: ok`, non-zero on `status: error`. Stdout is captured by the orchestrator and may be logged but is not part of the contract.

### Determinism

Given identical `(schema_json, pattern_name)`, the skill produces byte-identical `Cargo.toml` and `lib.rs` output across invocations. (The template approach gives this for free; it is called out so resink-core's content-hash dedup per training spec §3.2 invariant 4 works correctly.)

### Acceptance contract (between AE and resink-core)

The skill is considered correct iff: (a) `STATUS.json.status == "ok"`, AND (b) `cargo build --release` in `<output_dir>` exits 0. Functional correctness of the generated code is verified end-to-end by resink-core's `make mvp-loop` (brief O1 KR1.5 verdict). AE owns (a) and (b); resink-core owns the end-to-end functional check.

## Risks

- **LLM-codegen quality (template-with-fill mitigation already chosen).** Pure-LLM codegen could produce non-compiling Rust. **Mitigation: template-with-LLM-fill is the primary approach this loop, not a fallback** — see "Decisions made this loop" above. The LLM's job is reduced to filling column names, types, and the SCD2 column triple from `schema_json` into a hand-authored Rust skeleton; cargo-build success is gated on the template being correct, which AE verifies once. If the template itself proves too rigid for `dim_user` (it should not), AE pivots to pure-LLM codegen with cargo-build retry-loop in the same loop; the brief explicitly permits this pivot.
- **Local `claude` CLI version drift.** The skill is invoked via the local `claude` CLI per the dispatch decision; if the CLI's flag names or skill-invocation surface change between AE's authoring and resink-core's integration, the contract breaks silently. Mitigation: AE pins the CLI version in DISPATCH.md and notes the exact `claude --version` output the smoke-test was run against. Resink-core's `make mvp-loop` checks `claude --version` matches before dispatch.
- **Schema delivery slip from resink-core.** AE's smoke-test (KR1.4) needs the `dim_user` schema; if resink-core's fixture lands late, AE smoke-tests against a self-authored stub that mirrors the brief's KR1.2 column list (`user_id`, `email`, `country` + SCD2 columns). Mitigation: stub is in AE's own tree, not committed to the resink-core fixture path; once the real schema lands, AE re-runs smoke-test against it.
- **Plugin scaffolding tooling friction.** AE has not yet created a third plugin in this marketplace; the marketplace docs assume `agent-forge` and `org-os` exist as references but the bump-version + manifest-validation tooling may surface new edge cases for a fresh plugin. Mitigation: `bash tests/run-all.sh` is run early in the loop, before SKILL.md content is finalized; failures surface mechanical fixes (missing fields in `plugin.json`, missing entries in `marketplace.json`).
- **Runtime node-ABI not yet stable.** The brief assumes resink-core's `Node` trait is `dlopen`-shaped per runtime spec §4.3, but the trait crate may not be published on a stable name yet. Mitigation: AE's cross-team ask requests a placeholder crate by 2026-05-19; if even the placeholder slips, AE's `Cargo.toml.tmpl` declares a path-dependency on a sibling AE-owned `nanofab-node-abi-stub` crate so `cargo build` exits 0 and resink-core swaps the dep declaration when their crate is real.
- **Two consecutive Bundle B deferrals (pattern-flag, not a this-loop risk).** Bundle B has now been deferred from 2026-05-10 (mid-loop ADR-gate slip) and 2026-05-16 (CEO MVP-focus call). Not a risk to this loop's deliverables, but flagged here so the retro can decide whether 2026-05-23 becomes a hard floor or whether AE re-slices Bundle B into smaller chunks that could be absorbed alongside MVP-shaped work in future loops.
- **AE bandwidth.** Three AE ICs plus EM, ~5 working days. The plugin scaffolding plus template authoring plus smoke-test plus DISPATCH.md is comfortable but not slack; if the cargo-build smoke fails on the first attempt and template iteration is needed, KR2 (charter touch-up) gets the cut, not any KR1 line item.

## Out of scope this loop

- **Bundle B (7 rituals tasks: T6–T12)** — explicitly deferred to **loop 2026-05-11-1113** per CEO brief. Forward pointer: AE will plan Bundle B as the loop-2026-05-23 floor in that loop's team OKR. Status of T6–T12 stays `carried`; nothing in `org-os/rituals/` is touched this loop.
- **Bundle C (5 tasks: T13–T17)** — remains carried per the 2026-05-10 OKR's "out of scope" framing. Earliest target is loop 2026-05-11-1302 (after Bundle B).
- **ADR-003 (verify-state-claims at ritual transitions)** — explicitly deferred to **loop 2026-05-11-1113** per CEO brief. AE does not draft, edit, or merge it this loop.
- **All `org-os/` edits.** Zero edits this loop. Tenant-isolation dry-run (KR1.6) is run as discipline-reaffirmation only.
- **Pattern-library expansion beyond `scd2_maintainer`.** No `scd1_first_event`, `window_stats_with_decrement`, `sweep_line_pair_count`, etc. this loop — they sit under the same `nanofab` plugin in future loops as sibling skill directories.
- **Compiled `.so` artifact production by the skill.** The skill outputs source-only; resink-core (or `make mvp-loop`) runs `cargo build`. AE's smoke-test runs `cargo build` only to verify the source compiles, not to produce a deliverable artifact.
- **Skill testing infrastructure beyond the smoke-test.** No fixture-based regression suite, no per-pattern golden-output tests this loop. The MVP gate is `cargo build` passes plus resink-core's end-to-end run; broader test infra is post-MVP.
- **Sub-agent dispatch via anything other than the local `claude` CLI.** No Python wrapper, no background job queue, no gRPC. The training spec §4.1 invocation surface is in AE's view the right long-term shape, but for one-skill / one-consumer this loop, CLI is the smallest surface that works.
- **Multi-tenant or multi-pattern dispatch.** The skill takes one `(schema, pattern, output_dir)` per invocation; the orchestrator is responsible for parallelizing across nodes per training spec §4.7.
- **Marketplace plugin documentation beyond the per-skill SKILL.md and DISPATCH.md.** No top-level `plugins/nanofab/README.md` this loop; the plugin's purpose is documented in the marketplace root's plugin entry and in the skill's SKILL.md frontmatter.
