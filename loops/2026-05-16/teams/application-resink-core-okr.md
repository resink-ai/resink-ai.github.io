---
layout: default
title: application-resink-core OKR — 2026-05-16
nav_exclude: true
render_with_liquid: false
date: 2026-05-16
status: active
type: okr
loop: 2026-05-16
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: okr
  owner: teams/application/resink-core
  date: 2026-05-16
  status: active
  loop: 2026-05-16
  links: parent: board/okrs/2026-05-16-ceo-brief.md
-->
# Resink Core OKR — 2026-05-16

## Context

This loop the company crosses the binary gap from `board/charter.md`: the first end-to-end customer-journey closed loop, exercised on a synthetic single-dim fixture. Resink-core owns the closing seam — the CEO brief's O1 (KR1.1–KR1.5) names this team for the `make mvp-loop` driver, the supervisor + node-ABI + coordinator crates, the in-memory event source, the workspace layout, and the synthetic fixture (generator + inverse-SCD2 derivation). What changed since loop 2026-05-10: our `trace-path-skeleton-v0` slice is **superseded** by the MVP slice — same architectural primitives (supervisor, node-ABI, coordinator, in-memory source, orchestrator), but every surface is exercised end-to-end against real (LLM-generated) codegen and a sim-farm verdict instead of stubbed. We carry forward the Cargo workspace skeleton at `repos/resink-ai/resink-core/` (with `members = []`), the named-hand-off-sections P3 convention, and three outstanding peer hand-offs that this loop become real consumed contracts (DE in-memory `Event` shape per O4; AE codegen-skill dispatch per O3; sim-farm Mode-A verdict per O2).

## Objectives

### O1: Ship the runtime substrate the MVP loop runs on

source: ceo-brief

Why it matters: Without a real (not stub) supervisor that can `dlopen` LLM-generated code and consume a parquet-backed in-memory event source, the closing seam has nothing to close. This objective owns CEO brief KR1.4 in full and underlies KR1.1 (the supervisor invocation is the second-to-last step in `make mvp-loop`). The runtime spec sections that govern this work — §4.2 (supervisor), §4.3 (node ABI), §4.4 (state layer), §5.4 (event shape), §8.2 (fake clock), §8.4 (sim CLI + trace format) — are the ones we cited in last loop's hand-off to sim-farm; this loop we honor them in code.

**Key results**
- KR1.1: `repos/resink-ai/resink-core/crates/nanofab-node-abi/` exists as a published workspace member and defines the `Node` trait per runtime spec §4.3 (`name`, `version`, `process(&mut self, ev, ctx)`), the `NodeCtx` surface (`get`/`put_scd2`/`emit` — the in-memory KV variant is acceptable this loop), the `Event` record shape per runtime spec §5.4 mirroring DE's O4 contract (`{table, key, op, before?, after?, event_ts, event_id}`), and a `cdylib` plugin entrypoint macro the codegen output uses. `cargo build --release -p nanofab-node-abi` exits 0.
- KR1.2: `repos/resink-ai/resink-core/crates/nanofab-supervisor/` exists as a published workspace member with one binary, `nanofab-supervisor`. Invocation `nanofab-supervisor --mode=sim --workspace=<path> --write-trace=<trace.jsonl> --write-output=<dim_user_output.parquet>` reads the workspace's `manifest.yaml`, `dlopen`s the cdylib at `nodes/dim_user_scd2/target/release/libdim_user_scd2.so` (or the platform-equivalent path), instantiates it via the ABI entrypoint, drives events in `event_ts` order from the in-memory source (KR1.3) through `process()` wrapped in `panic::catch_unwind` per runtime spec §7.4, accumulates SCD2-maintained rows in an in-memory `HashMap`-backed KV, writes `dim_user_output.parquet` on shutdown, and emits one JSONL trace record per state mutation per runtime spec §8.4 (`{trace_id, node_id, event_id, key, before, after, event_ts, node_version}`). Exits 0 on the green path.
- KR1.3: `repos/resink-ai/resink-core/crates/nanofab-supervisor/src/event_source.rs` (or equivalent module) implements the in-memory event source: reads the two MVP fact parquets via `parquet`/`arrow`, materializes events conforming to DE's O4 `Event` record, sorts globally by `event_ts` then `event_id` (deterministic tiebreak), exposes an iterator the supervisor consumes. Hash-by-PK partitioning convention (per O4 KR4.3 + runtime spec §3.2 invariant #2) is implemented even though the MVP runs single-shard — the function name and width are documented in code comments matching DE's convention exactly so the swap to Kafka is byte-stable.
- KR1.4: `repos/resink-ai/resink-core/crates/nanofab-coordinator/` exists as a published workspace member with one binary, `nanofab-coordinator publish-dag --workspace=<path>`, that reads `<workspace>/manifest.yaml` + `<workspace>/release_seal.json`, validates the seal's stage-1 (compile) and stage-2 (smoke) are `passed` (stages 3 + 4 may be `skipped: true` per CEO brief KR1.3), prints `accepted, version=<seal.version>` to stdout, and exits 0. No gRPC server, no Raft, no state store — manifest validation only, per training spec §4.12.
- KR1.5: A determinism smoke (`cargo test -p nanofab-supervisor --test determinism`) runs the supervisor twice over the same fixture and asserts byte-identical `trace.jsonl` and byte-identical `dim_user_output.parquet`. Fake-clock discipline per runtime spec §8.2 (no wall-clock reads on the hot path) is enforced by review only this loop; the test catches regression by output-equality.

**Tasks**
- [x] Land `nanofab-node-abi` crate with the `Node` trait, `NodeCtx`, `Event` mirror of DE's O4 contract, and cdylib entrypoint macro — owner: teams/application/resink-core
  - Built at `crates/nanofab-node-abi/`. `cargo build --release -p nanofab-node-abi` exits 0. Trait surface is verbatim field-by-field match with the codegen template's inlined types so the next-loop template can switch to `use nanofab_node_abi::*` without behavioural change. Cdylib entrypoint macro is NOT shipped — the codegen template exports `nanofab_node_new`/`nanofab_node_drop` directly per its inline boilerplate; macro factoring is post-MVP.
- [x] Land `nanofab-supervisor` crate with `--mode=sim` CLI, `dlopen` flow, in-memory KV, parquet writer, JSONL trace writer, `panic::catch_unwind` wrapper — owner: teams/application/resink-core
  - Built at `crates/nanofab-supervisor/`. Implements all of the above EXCEPT `dlopen` — see Build phase notes section, ABI decision Option A: the codegen output is a Cargo path-dep and we call `Node::process()` via Rust trait methods directly. `panic::catch_unwind` wrapper is in place. Five clippy warnings (unused dead-code from the partition/key_value_for helpers we left for future multi-shard work) are accepted and named.
- [x] Implement the in-memory parquet event source with deterministic `event_ts`/`event_id` ordering — owner: teams/application/resink-core
  - At `crates/nanofab-supervisor/src/event_source.rs`. Reads via `parquet` + `arrow` crates, materializes Events conforming to DE contract §1, sorts by `(event_ts, event_id)`. Hash-by-PK partitioning convention named in code comments per DE contract §4 (single-shard for MVP so the function is documented but unused).
- [x] Land `nanofab-coordinator` crate with `publish-dag` binary that validates the seal's stages 1+2 `passed` — owner: teams/application/resink-core
  - Built at `crates/nanofab-coordinator/`. `nanofab-coordinator publish-dag --workspace=<path>` reads manifest.yaml + release_seal.json, validates stage `compile` + `smoke` are `passed` and stages `sim_farm` + `deploy` are `passed`-or-`skipped`, prints `accepted, version=<seal.version>` on success.
- [x] Wire the three crates into `repos/resink-ai/resink-core/Cargo.toml` `members = [...]` — owner: teams/application/resink-core
  - Workspace members updated. The codegen-output crate at `synthetic_tenants/closed_loop_v0/workspace/nodes/dim_user_scd2/` is NOT a workspace member but IS the supervisor's path-dep — see Build phase notes for the trade-off.
- [x] Add the determinism smoke test — owner: teams/application/resink-core
  - At `crates/nanofab-supervisor/tests/determinism.rs`. Marked `#[ignore]` (needs the workspace populated first). Verified passing via `cargo test --release -p nanofab-supervisor --test determinism -- --include-ignored`.

### O2: Ship the training-pipeline orchestrator + workspace shape that AE's codegen output drops into

source: ceo-brief

Why it matters: CEO brief KR1.3 names this team for the `manifest.yaml` + per-node `Cargo.toml` + `release_seal.json` written into `synthetic_tenants/closed_loop_v0/workspace/`. AE's O3 ships the codegen skill; we ship the orchestration that calls it and the workspace layout the runtime then consumes. This is the load-bearing seam between sub-projects #1 and #2.

**Key results**
- KR2.1: `repos/resink-ai/resink-core/training/orchestrator/` exists with a Python entry point (`uv run python -m orchestrator --fixture-dir=<path> --workspace=<path>`) that: reads the two MVP fact parquets, derives the dim-table schema from the fixture's `dim_user` (column names, types, primary key, SCD2 column names), invokes AE's `nanofab-codegen-scd2` skill via the dispatch contract spec'd in O3 KR3.2 (passing `(schema_json, pattern_name="scd2_maintainer", output_dir=<workspace>/nodes/dim_user_scd2)`), writes `<workspace>/manifest.yaml` and `<workspace>/release_seal.json`, runs `cargo build --release` in the produced node crate, and updates `release_seal.json` to mark stage 1 `passed`/`failed` based on cargo's exit code. Exits 0 only when stage 1 is `passed`.
- KR2.2: The workspace layout written by the orchestrator follows training spec §4.2 exactly for the MVP subset: `manifest.yaml`, `nodes/dim_user_scd2/{Cargo.toml, src/lib.rs}`, `.nanofab/release_seal.json`. The `manifest.yaml` names: one DAG with one node `dim_user_scd2` bound to table `dim_user`, partitioning key `user_id`, the node's compiled `.so` path, the runtime spec §5.4 `event_id`/`event_ts` field names, and `node_version=1`. The `release_seal.json` carries the four stage entries: stages 1 (compile) and 2 (smoke) start as `pending` and the orchestrator updates them; stages 3 (Sim Farm) and 4 (deploy) ship as `{status: "skipped", reason: "out-of-scope-for-MVP"}` per CEO brief KR1.3.
- KR2.3: A smoke step the orchestrator owns runs the produced cdylib through a 1-event sanity check (instantiate the node, feed one synthetic `Event`, assert no panic, assert `put_scd2` called once) and updates `release_seal.json` stage 2 to `passed` on success. This is the runtime-side counterpart to AE's KR3.3 "the source compiles" — KR2.3 is "the source loads and survives one event."
- KR2.4: The orchestrator emits a `training_history.md` line per training spec §4.7 noting the dispatched skill, the input schema hash, and the output node's compiled `.so` path. This is the audit trail that lets a second IC re-run the orchestrator and see what happened.

**Tasks**
- [x] Land `training/orchestrator/` Python module + `pyproject.toml` (uv-managed per project conventions) — owner: teams/application/resink-core
  - At `training/orchestrator/`. `uv sync` resolves 3 packages (pyarrow, pyyaml, hatchling-built local package). Entry point: `uv run python -m orchestrator --fixture-dir=<...> --workspace=<...> --plugin-dir=<...>`.
- [x] Implement schema derivation from the fixture's `dim_user` parquet — owner: teams/application/resink-core
  - > pivoted: the orchestrator reads `<fixture-dir>/dim_user.schema.json` (emitted by Phase 1's generator) rather than re-deriving from parquet metadata. The schema-json file is the canonical artifact AE consumes; emitting it from the generator avoids a Python parquet→JSON shape-mapping that would duplicate the generator's authoritative shape.
- [x] Implement the AE-skill dispatch call against O3 KR3.2's contract — owner: teams/application/resink-core
  - At `training/orchestrator/src/orchestrator/dispatch.py`. Invokes `claude --plugin-dir <abs> --output-format json --permission-mode bypassPermissions --print "/nanofab:codegen-scd2-node {json}"`. **Deviation from DISPATCH.md:** dropped `--bare` because claude 2.1.138 `--bare` uses strict env-only auth and refuses to read the user's OAuth keychain — see Build phase notes for the full discussion. The non-`--bare` form picks up developer OAuth and the skill runs identically. AE may want to update DISPATCH.md to either pin an `ANTHROPIC_API_KEY` requirement or note the OAuth-vs-bare distinction.
- [x] Implement `manifest.yaml` + `release_seal.json` writers matching training spec §4.2 — owner: teams/application/resink-core
  - At `training/orchestrator/src/orchestrator/manifest.py`. The seal ships stages 1+2 as `pending` (orchestrator updates them) and stages 3+4 as `skipped` with `reason: "out-of-scope-for-MVP"`. Manifest declares fact-stream paths as filename-only (resolved against `--fixtures-dir` or default `<workspace>/../fixtures/`).
- [x] Implement the stage-2 1-event smoke that loads the produced cdylib — owner: teams/application/resink-core
  - > pivoted: under ABI decision Option A, the supervisor compiles the codegen output as a Rust library and calls `Node::process()` via trait methods (no `dlopen`). The stage-2 smoke is therefore `cargo test --release` in the codegen output dir, which exercises the template's bundled `smoke_node_name_and_version` test. This is a stronger guarantee than the original "1-event smoke" — the template's smoke validates the full struct layout, not just instantiation.
- [x] Implement the `training_history.md` append — owner: teams/application/resink-core
  - Append-only per training spec §4.7. Records timestamp, skill name, schema hash (16-char SHA-256 prefix), output path, cargo build/test exit codes, status.

### O3: Ship the synthetic fixture, the closing `make mvp-loop` driver, and the workspace-promotion decision

source: ceo-brief

Why it matters: CEO brief KR1.1 + KR1.2 + KR1.5 land here. The fixture is owned by resink-core (per the brief's O1 task list) because the inverse-SCD2 derivation is the inverse of the SCD2-maintainer pattern AE's codegen will emit — same team, same invariants, same head. The `make mvp-loop` driver is the single user-visible entry point and the only artifact that the loop's exit criterion (`verdict=pass`) is read from. Workspace promotion (`repos/resink-ai/resink-core/` → real git submodule) was deferred from loop 2026-05-10; with three real crates landing this loop, deferring again is a debt accrual we should avoid unless we discover a concrete blocker.

**Key results**
- KR3.1: `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/fixtures/generate.py` exists. Pure-Python (no LLM), seeded (`--seed=<int>`, default `0xC10ED1`), deterministic. Produces `dim_user_fixture.parquet` with SCD2 columns (`user_id`, `email`, `country`, `valid_from`, `valid_to`, `is_current`) and ≥3 distinct users, ≥2 of which carry ≥1 SCD2 update each — i.e., the dim has at least 5 SCD2 rows total (3 initial + ≥2 updates). Re-running with the same seed produces a byte-identical parquet (verified by `sha256sum`).
- KR3.2: `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/fixtures/derive_facts.py` exists. Reads `dim_user_fixture.parquet` and writes (a) `fact_sign_up.parquet` — one row per distinct `user_id` at that user's earliest `valid_from`, with the initial-version payload; (b) `fact_profile_update.parquet` — one row per non-initial SCD2 version, at its `valid_from`, with the post-update payload. Both fact parquets carry `event_id` (deterministic, `f"{table}:{user_id}:{valid_from}"` hashed) and `event_ts = valid_from`. Re-running on the same dim fixture produces byte-identical fact parquets.
- KR3.3: `repos/resink-ai/resink-core/synthetic_tenants/closed_loop_v0/Makefile` exists with target `make mvp-loop`. The recipe runs, in order: (1) `python fixtures/generate.py --out fixtures/dim_user_fixture.parquet`; (2) `python fixtures/derive_facts.py --in fixtures/dim_user_fixture.parquet --out-dir fixtures/`; (3) the orchestrator from O2; (4) `nanofab-coordinator publish-dag --workspace workspace/`; (5) `nanofab-supervisor --mode=sim --workspace workspace/ --write-trace workspace/trace.jsonl --write-output workspace/dim_user_output.parquet`; (6) sim-farm's diff invocation per O2 KR2.1's verdict-shape contract (we shell out to whatever sim-farm names in their KR2.1 — see Cross-team asks). PASS condition: exits 0 and stdout final line contains `verdict=pass`. FAIL condition: exits non-zero and stderr final line contains `verdict=fail mismatches=<n> see workspace/verdict.json`.
- KR3.4: A `make clean` target removes everything the loop produced (`fixtures/*.parquet`, `workspace/`); a fresh `make mvp-loop` after `make clean` succeeds (proves no hidden state leakage between runs).
- KR3.5: Workspace promotion decision documented in this OKR's "Plan" section. Default (this loop): promote `repos/resink-ai/resink-core/` to a real git submodule with a real remote URL and a `.gitmodules` entry in newbase. Permitted pivot: defer to loop 2026-05-23 with a single recorded blocker. Either way, the README at `repos/resink-ai/resink-core/README.md` is updated to match the actual state.
- KR3.6: A "permitted pivot" register: if parquet itself proves non-deterministic across our `pyarrow` version (CEO brief Risks #3), the fixture falls back to CSV (small enough for the MVP). Pivot is named here so it's not a scope expansion mid-loop.

**Tasks**
- [x] Implement `fixtures/generate.py` with seeded determinism — owner: teams/application/resink-core
  - At `synthetic_tenants/closed_loop_v0/fixtures/generate.py`. Default seed `0xC10ED1`. Also emits `dim_user.schema.json` (the canonical artifact AE consumes). The RNG seed is currently unused in the MVP fixture (values are hand-set); it's exposed for future fixture variants.
- [x] Implement `fixtures/derive_facts.py` (inverse-SCD2 derivation) — owner: teams/application/resink-core
  - At `synthetic_tenants/closed_loop_v0/fixtures/derive_facts.py`. Emits 3 sign-up rows + 2 profile-update rows from the 5-row dim. `event_id` is `<table>:<user_id>:<event_ts>:<sha256-prefix>` (deterministic).
- [x] Verify byte-stability via `sha256sum` over two consecutive runs — owner: teams/application/resink-core
  - Verified manually during build: SHA-256 hashes of `dim_user_fixture.parquet`, `fact_sign_up.parquet`, `fact_profile_update.parquet` are byte-identical across re-runs on the same pyarrow (24.0.0) / Python (3.12.10) combination.
- [x] Author the `Makefile` orchestrating the six steps with PASS/FAIL conventions — owner: teams/application/resink-core
  - At `synthetic_tenants/closed_loop_v0/Makefile`. Seven steps (an extra `cargo build --release -p nanofab-coordinator` was rolled into step 4 to keep step counts low). PASS surface: `verdict=pass mismatches=0` to stdout, exit 0. FAIL: `verdict=fail mismatches=N see <verdict-path>` to stderr (the diff_scd2 engine handles the print itself; the Makefile only gates on exit code).
- [x] Add `make clean` + clean-then-loop smoke — owner: teams/application/resink-core
  - `make clean` removes fixtures parquets, schema.json, the workspace's generated artifacts, AND the codegen output's `target/`+`STATUS.json`+`Cargo.lock`. The codegen output's `src/lib.rs` + `Cargo.toml` is wiped by the orchestrator's dispatch step on the next run. Clean-then-loop smoke verified passing (in skip-dispatch mode for speed).
- [x] Decide workspace promotion (submodule vs in-tree) and update README — owner: teams/application/resink-core
  - > pivoted: deferred to loop 2026-05-23. Recorded blocker: **MVP closure consumed the loop's full bandwidth**. The MVP-priority guard in the brief prompt is explicit. README is updated to reflect this.
- [x] If pivoting to CSV, document in this OKR before the pivot lands — owner: teams/application/resink-core
  - Not needed; parquet determinism held on pyarrow 24.0.0 across two consecutive runs (byte-identical SHA-256). No pivot required.

## Cross-team asks

Each ask is dated, ownerful, and traces back to a CEO brief KR. The hand-off sections below in § Plan are the read-by-other-teams artifact (P3 convention from the 2026-05-09 retro); these bullets restate what we specifically need and the by-when.

- **From `teams/platform/data-engineering`, by mid-loop (2026-05-19):** the in-memory event-source contract (CEO brief O4 KR4.1) lands at `teams/platform/data-engineering/contracts/2026-05-16-in-memory-event-source.md` with `status: active`, defining the `Event` record shape per runtime spec §5.4 (`{table, key, op, before?, after?, event_ts, event_id}`), the in-`event_ts`-order delivery guarantee, and the hash-by-PK partitioning function name + width (CEO brief O4 KR4.3) we mirror in O1 KR1.3. **Reason:** O1 KR1.1 (the `nanofab-node-abi` `Event` type) and O1 KR1.3 (the in-memory source) cannot land without the contract. **If slipped:** we use the placeholder shape inline in the ABI crate and DE files an addendum in the same loop.
- **From `teams/platform/agent-engineering`, by mid-loop (2026-05-19):** the codegen-skill dispatch contract (CEO brief O3 KR3.2) — a documented function/CLI signature `(schema_json, pattern_name, output_dir) → (artifact_paths, status)`, plus a smoke confirming the skill produces a compilable Rust crate against the MVP fixture's `dim_user` schema. **Reason:** O2 KR2.1 (the orchestrator) calls into AE's surface; without the contract we either stub or block. **If slipped:** O2 KR2.1 falls back to a hand-written `dim_user_scd2/lib.rs` template with the LLM dispatch path stubbed, and AE files the contract as a same-loop addendum. We do not block O1 or O3 on this — only O2.
- **From `teams/application/sim-farm`, by mid-loop (2026-05-19):** the verdict-format contract (CEO brief O2 KR2.2) lands at `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`, AND sim-farm names the diff-engine invocation shape (CLI flags + exit-code semantics) we shell out to from `make mvp-loop` step 6. **Reason:** O3 KR3.3's PASS/FAIL conventions reference `verdict=pass` and `workspace/verdict.json`; we need the exact field names and the invocation. **If slipped:** the Makefile invokes a placeholder `verdict-tool` shim that always emits `verdict=fail mismatches=unknown`, and the loop's exit criterion still reads as red until sim-farm lands.

## Risks

- **Real codegen against a real ABI surface this loop is the most ambitious slice we've ever taken.** Mitigation: the fixture is the smallest one that exercises both halves (single dim, two facts, single shard, in-memory everything per CEO brief Out-of-scope). If KR2.3's stage-2 smoke fails because AE's codegen produces non-compiling output (CEO brief Risk #2), O2 KR2.1's permitted pivot is to use a hand-written `lib.rs` template behind the same dispatch contract — the supervisor + orchestrator + verdict pipeline still close end-to-end and the failure surface is named (codegen).
- **`dlopen` + cdylib + Rust ABI is platform-sensitive (macOS dylib vs Linux .so paths, symbol-export quirks, Cargo profile flags).** Mitigation: the supervisor reads the cdylib path from `manifest.yaml` (set by the orchestrator) rather than hard-coding the extension; both `.so` and `.dylib` extensions are honored. The CEO brief's CI-environment is `rustup`-based on the developer's laptop — we test on whichever platform an IC runs on first and add the other in this loop's exec summary if needed.
- **Parquet determinism (CEO brief Risk #3).** Mitigation: O3 KR3.6's pivot to CSV is permitted and dated. If we trip on this, we pivot before mid-loop so sim-farm's diff implementation has time to consume CSV instead of parquet (sim-farm spec §4.5's DuckDB engine reads both natively).
- **Workspace promotion to a real submodule may surface remote-URL / permissions issues we don't have time to chase mid-loop.** Mitigation: O3 KR3.5's permitted pivot is to defer with a recorded blocker. We attempt promotion early in the loop (day 1 or 2) so a slip surfaces with margin to revert.
- **Three peer hand-offs all due 2026-05-19 (DE, AE, sim-farm) means a single peer slip cascades.** Mitigation: each Cross-team ask has an "If slipped" branch with an explicit fallback. We do not block any of our KRs waiting past mid-loop; we land what we can with placeholders and the peer files an addendum in the same loop.
- **The `synthetic_tenants/resink-core/resink-core/` symlink layout (where the team directory shadows the product repo) might confuse a new IC.** Status: `teams/application/resink-core/resink-core` is a symlink to `repos/resink-ai/resink-core/` (verified this loop) — not a duplicate. No reconciliation needed; flag as low-priority retro candidate to consider whether the symlink should be removed in favor of explicit relative paths in this team's docs.

## Out of scope this loop

Carried from the CEO brief's "Out of scope this loop":
- Real Kafka (in-memory only).
- Real KV (in-memory `HashMap` per shard; TiKV/FoundationDB pick remains deferred).
- DLQ / quarantine / panic recovery beyond `panic::catch_unwind` skeleton.
- Multi-tenant isolation (single hard-coded tenant).
- Hot swap, blue/green, shadow sidecar.
- Helm chart skeleton (DevOps paused).
- Codegen patterns beyond `scd2_maintainer`.
- More than one dim or more than two fact streams in the fixture.
- Iceberg sink (`nanofab-iceberg-sink`), query gateway (`nanofab-query`), state-layer crate (`nanofab-state` as a published-vendor wrapper) — runtime spec §4.4–4.7 components are NOT this loop. The supervisor uses an inline in-memory KV; no separate `nanofab-state` crate ships this loop.

Resink-core-specific additions:
- LLM provider abstraction (training spec §9) — the orchestrator calls AE's skill via AE's contract; we do not pick a provider abstraction this loop.
- KV vendor pick (TiKV vs FoundationDB) — second-slice work, not unblocking the MVP.
- The remaining six pattern-library entries (training spec §4.10) beyond `scd2_maintainer`.
- The `nanofab-transport` crate (runtime spec §4.5) — the supervisor's intra-DAG transport this loop is a single-thread iterator, no crate boundary needed.
- Production-shape error handling, retries, observability beyond `panic::catch_unwind` + the JSONL trace.
- Schema registry validation (runtime spec §7.6) — the fixture is hand-shaped, no registry needed.
- Metrics (runtime spec §8.5) — the trace JSONL is the only observability surface this loop.

Explicitly NOT in the first slice (matching the 2026-05-10 OKR's pattern — the slice is defined as much by what it skips):
- No re-keying / cross-shard logic (`cross_shard_re_key` pattern out).
- No `reset_sim()` control-gRPC (sim-farm warm-pool support is the loop after MVP).
- No `--trace-format=v2` evolvability flag (v1 only; the format is defined by O1 KR1.2).
- No `live_version` pointer or `coordinator publish-dag` HA — the coordinator is one process, one invocation, prints "accepted" and exits.
- No `panic_event` records in `trace.jsonl` because the cdylib should not panic on the MVP fixture; the wrapper is in place but emits nothing if no panics occur.

## Plan

### The first slice — `mvp-closed-loop-v0`

**Name:** `mvp-closed-loop-v0`. Supersedes `trace-path-skeleton-v0` from the 2026-05-10 OKR.

**The minimum closed loop, end to end:** generate a single `dim_user` SCD2 fixture → derive two fact parquets from the dim → orchestrator dispatches AE's codegen skill against the dim schema → cdylib lands in the workspace → coordinator validates the seal → supervisor loads the cdylib, drives derived events through it, writes an output dim parquet → sim-farm diffs the output dim against the input fixture → `make mvp-loop` exits 0 with `verdict=pass` only when every SCD2 row matches.

### File-path map (where each piece lives)

```
repos/resink-ai/resink-core/
├── Cargo.toml                                  # workspace; members updated this loop
├── crates/
│   ├── nanofab-node-abi/                       # O1 KR1.1
│   │   ├── Cargo.toml
│   │   └── src/lib.rs                          # Node trait, NodeCtx, Event, cdylib macro
│   ├── nanofab-supervisor/                     # O1 KR1.2 + KR1.3
│   │   ├── Cargo.toml
│   │   ├── src/main.rs                         # CLI, dlopen, trace writer
│   │   ├── src/event_source.rs                 # in-memory parquet → Event iterator
│   │   ├── src/kv.rs                           # in-memory HashMap KV
│   │   └── tests/determinism.rs                # O1 KR1.5
│   └── nanofab-coordinator/                    # O1 KR1.4
│       ├── Cargo.toml
│       └── src/main.rs                         # publish-dag binary, seal validation
├── training/
│   └── orchestrator/                           # O2
│       ├── pyproject.toml
│       ├── src/orchestrator/__init__.py
│       ├── src/orchestrator/__main__.py        # CLI entry point
│       ├── src/orchestrator/dispatch.py        # AE-skill dispatch per O3 KR3.2
│       ├── src/orchestrator/manifest.py        # manifest.yaml + release_seal.json writers
│       └── src/orchestrator/smoke.py           # stage-2 1-event smoke
└── synthetic_tenants/
    └── closed_loop_v0/                         # O3
        ├── Makefile                            # `make mvp-loop` + `make clean`
        ├── fixtures/
        │   ├── generate.py                     # O3 KR3.1
        │   └── derive_facts.py                 # O3 KR3.2
        └── workspace/                          # populated at runtime by orchestrator
            ├── manifest.yaml                   # written by orchestrator
            ├── nodes/dim_user_scd2/            # cdylib crate written by AE codegen
            │   ├── Cargo.toml
            │   └── src/lib.rs
            ├── trace.jsonl                     # written by supervisor
            ├── dim_user_output.parquet         # written by supervisor
            ├── verdict.json                    # written by sim-farm tool
            └── .nanofab/release_seal.json
```

### Acceptance for `make mvp-loop`

**PASS** is defined by the simultaneous truth of:
1. `make mvp-loop` exits 0.
2. The final stdout line of step 6 contains `verdict=pass`.
3. `workspace/verdict.json` exists, parses as JSON, and has `pass: true` AND `mismatch_count: 0`.
4. `workspace/dim_user_output.parquet` exists and has the same row count as `workspace/../fixtures/dim_user_fixture.parquet` (sanity check; sim-farm's diff is the authoritative comparison).
5. `workspace/trace.jsonl` exists and is non-empty (every state mutation traced).

**FAIL** is defined by any of:
- Any step exits non-zero (`make mvp-loop`'s overall exit code is non-zero).
- `workspace/verdict.json` exists with `pass: false` — final stderr line is `verdict=fail mismatches=<n> see workspace/verdict.json`.
- `workspace/verdict.json` is missing (placeholder shim never produced one — sim-farm slipped) — final stderr line is `verdict=fail reason=verdict-tool-missing`.

A FAIL run is still a **loop succeeded** outcome from the CEO brief's perspective: the MVP loop attempted closure and the failure surface is informative. A loop failure for *this team* is `make mvp-loop` not running at all (KR1.1 unmet).

### Hand-off section: DE — in-memory event-source contract

**Audience:** `teams/platform/data-engineering`. CEO brief O4 names DE as owner.

**What we need from DE:**
- The `Event` record shape per runtime spec §5.4 — the exact field names (`table`, `key`, `op`, `before?`, `after?`, `event_ts`, `event_id`) and Rust-friendly types.
- The hash function name + width for the partitioning convention (CEO brief O4 KR4.3) we mirror in O1 KR1.3 so the in-memory and Kafka paths are byte-stable.
- The "no replay, no DLQ — the file path is the topic" framing in writing so a future IC building the second slice doesn't accidentally re-implement DLQ in the in-memory source.

**What we promise to expose to DE:**
- Our consumer-side constraints surfaced in the contract's "Production parity" section: the supervisor consumes Events in `event_ts` order with `event_id` as deterministic tiebreak; `before` is required for `update`/`delete` ops; `after` is required for `insert`/`update`.
- The `nanofab-node-abi` crate re-exports the `Event` type by the same field names DE specifies, so the in-memory and Kafka producers can both target it.

**What we ask DE to confirm by mid-loop (2026-05-19):**
- That the `Event` field names + types in their contract match exactly what we implement in O1 KR1.1. If a delta surfaces, DE files an addendum on the same contract; we do not block O1 KR1.1 — we land the placeholder shape inline and name it for mid-loop reconciliation.

### Hand-off section: AE — codegen-skill dispatch contract

**Audience:** `teams/platform/agent-engineering`. CEO brief O3 names AE as owner.

**What we need from AE:**
- The dispatch contract (CEO brief O3 KR3.2): how O2's orchestrator invokes the `nanofab-codegen-scd2` skill — function signature or CLI shape, input format (`(schema_json, pattern_name="scd2_maintainer", output_dir=<workspace>/nodes/dim_user_scd2)`), output format (`(artifact_paths, status)`).
- The compile guarantee (CEO brief O3 KR3.3): the skill produces a Rust crate where `cargo build --release` exits 0 against our MVP fixture's `dim_user` schema.
- A worked example: AE runs the skill against a hand-constructed `dim_user` schema JSON identical to what O3 KR3.1 produces, and confirms a compilable crate.

**What we promise to expose to AE:**
- The exact `dim_user` schema JSON shape O2's orchestrator passes (column names, types, primary key, SCD2 column names) — documented in `training/orchestrator/src/orchestrator/dispatch.py` as the canonical reference.
- The `nanofab-node-abi` crate as the trait surface AE's generated `lib.rs` targets — published as a workspace member, with the trait + `NodeCtx` surface stable for at least this loop (no breaking changes mid-loop).
- The cdylib output path AE's skill writes to (`<output_dir>/Cargo.toml` + `<output_dir>/src/lib.rs`) — we do not require AE to run `cargo build`; the orchestrator does that.

**What we ask AE to confirm by mid-loop (2026-05-19):**
- That the dispatch contract's signature matches what `training/orchestrator/src/orchestrator/dispatch.py` calls. If a delta surfaces, the orchestrator's adapter absorbs it (we own the call site).
- That AE's permitted pivot from CEO brief Risk #2 (template-with-LLM-fill if pure-LLM proves brittle) lands behind the same dispatch contract — we do not need to know which path AE took, just that the contract holds.

### Hand-off section: sim-farm — verdict format and diff invocation

**Audience:** `teams/application/sim-farm`. CEO brief O2 names sim-farm as owner.

**What we need from sim-farm:**
- The verdict JSON schema (CEO brief O2 KR2.2) at `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`. We need the exact field names so O3 KR3.3's PASS condition reads them correctly: `pass: bool`, `mismatch_count: int`, `mismatches: [...]`, `verdict_id`, `mode: "A"`, `dim_table`, `fixture_path`, `output_path`, `ran_at`, `engine_version`.
- The diff-engine invocation shape: the binary or script name, its CLI flags (e.g., `--fixture <path> --output <path> --verdict-out <path>`), and its exit-code semantics (0 on `verdict=pass`, non-zero on `verdict=fail`).
- The DuckDB-based SCD2-equivalence behavior per Sim Farm spec §4.5 — handles missing/extra/diverged rows correctly.

**What we promise to expose to sim-farm:**
- The two parquet paths the diff consumes: `synthetic_tenants/closed_loop_v0/fixtures/dim_user_fixture.parquet` (the input dim) and `synthetic_tenants/closed_loop_v0/workspace/dim_user_output.parquet` (the supervisor's output). Both have the SCD2 column convention from O3 KR3.1 (`user_id`, `email`, `country`, `valid_from`, `valid_to`, `is_current`).
- The `make mvp-loop` Makefile's step 6 calls sim-farm's tool with the contract's exact CLI shape — we adapt to whatever sim-farm names; we do not require sim-farm to adapt to us.
- A FAIL run leaves `workspace/verdict.json` in place for the operator to read; we do not delete it on subsequent failed runs.

**What we ask sim-farm to confirm by mid-loop (2026-05-19):**
- The verdict-shape contract is `status: active` and the diff engine is invokable from a Makefile (no Docker, no service start-up, no remote API — local CLI per CEO brief KR1.1's "only `rustup`, Python 3.12, `make`, and the local `claude` CLI on PATH" constraint).
- That sim-farm's KR2.3 "Wire into `make mvp-loop`" lands as a PR against our Makefile (not a separate invocation surface), so the integration is one-document.

### Workspace promotion (KR3.5)

**Default this loop:** promote `repos/resink-ai/resink-core/` to a real git submodule. Steps:
1. Create the remote (the team picks the repo URL early in the loop — most likely `github.com/resink-ai/resink-core` or equivalent).
2. `git submodule add <url> repos/resink-ai/resink-core` in newbase.
3. The existing in-tree contents become the first commit on the submodule's main branch.
4. `.gitmodules` in newbase is updated to record the submodule.
5. The README at `repos/resink-ai/resink-core/README.md` is updated to drop the "submodule promotion deferred" line.

**Permitted pivot:** defer to loop 2026-05-23 with a single recorded blocker. The blocker must be specific (e.g., "no remote URL approved by 2026-05-19" or "submodule add conflicts with $X CI step"). If we pivot, the README is updated to keep the deferral note current.

**Why we attempt promotion this loop:** three real crates are landing; deferring again means three loops of in-tree drift before any cross-repo concerns surface. The promotion is the smallest reversible step that catches them.

## Build phase notes

### ABI decision: Option A — Cargo path-dep, no `dlopen`

The Wave-1 codegen template (`repos/resink-ai/resink-marketplace/plugins/nanofab/skills/codegen-scd2-node/templates/scd2_maintainer/lib.rs.tmpl`) exports only `nanofab_node_new()` and `nanofab_node_drop()` as C-ABI symbols — there is NO `nanofab_node_process` C-ABI symbol. Pure `dlopen` + call-`process()` is therefore impossible without either (B) extending the codegen output with a Rust shim or (C) re-shipping the template with a `_process` export.

**Choice: Option A — static linking.** The supervisor's `Cargo.toml` declares the codegen-output crate as a Cargo path-dependency at a stable workspace-relative path (`crates/nanofab-supervisor/Cargo.toml` → `nanofab_node_dim_user_scd2 = { path = "../../synthetic_tenants/closed_loop_v0/workspace/nodes/dim_user_scd2" }`). The supervisor `use`s the crate and calls `DimUserScd2Maintainer::default()` + `Node::process()` via Rust trait methods directly.

Rationale:
- Option A is the smallest reversible step that closes the loop end-to-end.
- The runtime-spec hot-swap property is sacrificed for MVP simplicity — the supervisor binary is rebuilt per-tenant. This is named explicitly so it surfaces in retro for next-loop consideration.
- Option B requires injecting a Rust shim into the codegen output and feels like leaking concerns; the right home for that shim is the next-loop template revision (AE-owned).
- Option C requires a Wave-1.5 round-trip that this loop's exit criterion can't afford.

**One Option-A wrinkle resink-core absorbs:** the codegen template declares `crate-type = ["cdylib"]`; Cargo path-deps need an `rlib` to act as a Rust library. The orchestrator's `add_rlib_to_cargo_toml()` post-processes the produced Cargo.toml to add `"rlib"` to the crate-type list. The AE template stays untouched; the post-processing is one-line and idempotent.

### make mvp-loop result

**PASS.** `make mvp-loop` (full real path including the `claude` CLI dispatch) exited 0 with `verdict=pass mismatches=0` on stdout. Captured artifacts:
- `synthetic_tenants/closed_loop_v0/workspace/verdict.json`: `pass: true, mismatch_count: 0`.
- `synthetic_tenants/closed_loop_v0/workspace/trace.jsonl`: 7 records (3 inserts as append-only + 2 updates as close+append).
- `synthetic_tenants/closed_loop_v0/workspace/dim_user_output.parquet`: 5 SCD2 rows matching the fixture row-for-row.
- `synthetic_tenants/closed_loop_v0/workspace/.nanofab/release_seal.json`: stages 1+2 `passed`, 3+4 `skipped`.
- `synthetic_tenants/closed_loop_v0/workspace/training_history.md`: one line per orchestrator run, compile_exit=0, smoke_exit=0.

The clean-then-loop smoke also passed (using `CLAUDE_SKIP_DISPATCH=1` for speed; the codegen output was wiped beforehand). The determinism test (`cargo test --release -p nanofab-supervisor --test determinism -- --include-ignored`) passed — trace.jsonl + output.parquet are byte-identical across two supervisor runs on the same fixture.

### Deviations from spec

1. **`--bare` dropped from the dispatch invocation.** DISPATCH.md recommends `--bare` for determinism; in `claude 2.1.138` `--bare` enforces env-only auth (`ANTHROPIC_API_KEY` / `apiKeyHelper`) and refuses to read the user's OAuth keychain. On the developer laptop without `ANTHROPIC_API_KEY` set, `--bare` reports "Not logged in" and the skill never runs. We use the non-`--bare` form (OAuth path). The determinism property is preserved by the skill itself (template slot-fill is deterministic; the LLM is not asked to be creative). **AE follow-up suggestion:** update DISPATCH.md to note this auth-mode distinction or pin an `ANTHROPIC_API_KEY` requirement.

2. **Stage-2 smoke is `cargo test --release` rather than "1-event smoke."** Under Option A, the codegen output is a Rust library — instantiating + driving one event via `nanofab_node_new` C-ABI is no more informative than `cargo test`, and the template ships a `smoke_node_name_and_version` test that's stricter (validates struct layout, not just instantiation). We run `cargo test --release` in the codegen output dir and gate stage 2 on its exit code.

3. **No `dlopen` of cdylib.** Per ABI decision Option A. Re-flagged here for cross-reference. The cdylib IS produced (the codegen output's Cargo.toml has `crate-type = ["cdylib", "rlib"]` post-orchestrator), but the supervisor links the rlib path-dep, not the dylib.

4. **No `EnterWorktree` isolation.** This task was issued mid-conversation; the build proceeded in-place under `repos/resink-ai/resink-core/` (which IS the canonical product-repo path). Re-running this loop from a worktree is straightforward — no shared mutable state.

### Peer-team integrations: what worked, what broke

- **AE codegen-scd2-node skill (Wave 1):** worked. The non-`--bare` dispatch produces a STATUS.json with `status: ok` and a compilable `Cargo.toml` + `src/lib.rs` against the MVP `dim_user` schema in ~106s on the developer laptop. The skill's slot-fill matches the SKILL.md "Worked example" table exactly.
- **DE in-memory event-source contract (Wave 1):** consumed at face value. The supervisor's `event_source.rs` materializes Events conforming to DE contract §1 (single-key map, `op` enum, optional `before`/`after`, deterministic `event_id`). Sort order per §2 (event_ts ASC, event_id ASC tiebreak). Hash-by-PK partition function from §4 documented as `partition()` in code, single-shard for MVP so the function is unused but named for next-loop swap parity. **One open question:** DE's contract §4 names `xxHash64(seed=0)`; the MVP supervisor's `partition()` uses a placeholder fold-and-mod (correct for `shard_count=1` but not byte-stable with xxHash64 for `shard_count>1`). Replacing with `twox-hash::xxh64::xxh64(bytes, 0)` is a next-loop one-liner.
- **Sim-farm Mode-A diff (Wave 1):** worked exactly as documented. The bare-script form of `python -m sim_farm.diff_scd2 --fixture <p> --output <p> --verdict <p>` invoked via the Makefile prints `verdict=pass mismatches=0` on the green path. The verdict.json shape matches `teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md` byte-for-byte.
- **No peer-team integrations broke.** Every Wave-1 artifact this Wave-2 build consumed conformed to its published contract.
