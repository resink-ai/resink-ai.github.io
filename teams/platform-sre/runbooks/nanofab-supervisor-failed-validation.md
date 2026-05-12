---
layout: default
title: "platform-sre runbook: nanofab-supervisor-failed-validation"
parent: "Team: platform-sre"
grand_parent: "Teams"
render_with_liquid: false
date: 2026-05-23
status: active
type: runbook
owner: teams/platform/sre
---

<!-- original-frontmatter:
  type: runbook
  owner: teams/platform/sre
  date: 2026-05-23
  status: active
  severity_tiers: [S1, S2, S3]
-->
# nanofab-supervisor — Failed Validation Runbook

> **Frontmatter-type note.** As of 2026-05-30, `type: runbook` is canonical in
> [`org-os/conventions.md`](../../../../org-os/conventions.md) per
> [ADR-2026-05-23-001](../../../../board/decisions/2026-05-23-001-conventions-enum-extension.md);
> the `severity_tiers` required field is satisfied by the S1/S2/S3 commitment
> in [§ Severity tiers](#severity-tiers). The SRE-local extension noted at
> first authoring is no longer needed.
>
> Historical note: this runbook was first filed (loop 2026-05-11-1113) under
> `type: runbook` as a SRE-local extension because the enum did not yet
> include it. ADR-2026-05-23-001 (ratified loop 2026-05-11-1302) made the type
> canonical; this frontmatter is unchanged in structure, only canonical in
> meaning.

## Purpose

This runbook is the single triage entry-point for any non-success terminal
state of the **nanofab-supervisor** binary
(`repos/resink-ai/resink-core/crates/nanofab-supervisor`) in
`--mode=sim`, when the supervisor is driving an MVP closed-loop run against
the **Mode-A** sim-farm verdict layer
([`teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`](../../../application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md)).
It catalogs the failure modes an operator can hit today (MVP scope) and stubs
the failure modes that arrive when Modes B/C ship.

The framing throughout: **the loop never silently passes**. Every failure
mode below either surfaces cleanly as a non-zero exit / refusal-to-write or
is flagged as a hot-path observability gap with a forward-pointer. If a run
looks "green" but the operator suspects otherwise, the General triage flow
below is the place to start.

## Scope

**MVP (covered in this revision):**

- Sim-farm Mode-A verdict mismatch types: `missing`, `extra`, `diverged`
  (the three values of `Mismatch.type`).
- Sim-farm engine-failure (exit code `2`) per
  [verdict contract § Exit-code semantics](../../../application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md).
- Supervisor `panic::catch_unwind` path (codegen `process()` panic) per
  [`main.rs` lines 167-197](../../../../repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs).
- Manifest-validation failure (coordinator/supervisor rejects malformed
  seal — `main.rs` lines 93-125: read/parse, DAG count ≠ 1, node count ≠ 1,
  unknown `op` strings).
- Cargo-build failure (stage-1 gate — codegen output that doesn't compile;
  static linking per the Wave-2 Option-A ABI decision).

**Post-Mode-B / post-Mode-C (stubs only):**

- Mode-B sidecar pair-orphan handling (`orphan_count > 1%` containment)
  [§6.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md).
- Sim-worker pod crash / `INFRA_FAILURE` [§6.1](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md).
- Supervisor write-trace socket overrun (`trace_dropped`) [§6.5](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md).
- Mode-A coordinator crash / restart [§6.6](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md).
- DuckDB diff engine failures beyond the MVP exit-code-2 case (OOM retries,
  schema-mismatch as structural diff) [§6.3](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md).
- The runtime coordinator's `BLOCKED` hot-swap state-machine surface (does
  not exist in MVP scope; see Cross-cutting note 2 below).

## Severity criteria

The runbook commits to three severity tiers. **Every** failure-mode
subsection tags itself with one of these.

- **S1 — supervisor stuck or wrong-data risk to downstream.** Any case
  where a non-zero exit slips past the calling driver, where the supervisor
  hangs (no progress for > 5× expected fixture duration), or where a panic
  is silently swallowed. The four MVP mismatch types
  (`missing`/`extra`/`diverged`/`engine_failure`) are S1 when they reach a
  hot-swap or training-stage-3 gate; S2 in pure MVP dev-loop context
  because `make mvp-loop` gates on the engine's exit code correctly.
- **S2 — failure surfaces cleanly and stops the loop, but operator must
  investigate the root cause.** Panic with non-zero exit, manifest
  rejection, cargo-build failure. The supervisor refused to proceed; no
  bad data went downstream; the operator's job is triage of the offending
  DAG, codegen output, or fixture, not unwinding bad state.
- **S3 — cosmetic, observability, or contract-version gap.** A
  `verdict.json` field renamed-but-runbook-not-updated; a trace-log line
  missing a tenant label; verdict-layer or engine-version drift that
  surfaces as a parsing nit rather than an operational fault.

S1 calls justify paging on-call (when on-call rotation exists; deferred per
[§ Out of scope](#out-of-scope-this-revision)). S2 stops the loop and is
filed as a ticket to the responsible team. S3 is recorded as a follow-up
and rolled up at retro.

## General triage flow

The supervisor produces, on every run, up to four artifacts. Inspect them
in this order:

1. **`workspace/verdict.json`** — if it exists, the sim-farm engine ran to
   some terminal state. Parse it as JSON; read `pass` (single-dim shape) or
   `overall_pass` (multi-dim shape). If either is `true` and yet the
   operator reached this runbook, jump to §6 (Out-of-scope failure modes).
   If `false`, branch by `engine_error` field presence:
   - `engine_error` present → engine-internal failure (§3.4).
   - `engine_error` absent and `mismatches[]` (or
     `verdicts[i].mismatches[]`) non-empty → §3.1 / §3.2 / §3.3 per
     `Mismatch.type` enum value.
2. **`<write-trace>` JSONL** — the supervisor's per-mutation trace per
   runtime spec §8.4. If the file is empty or truncated mid-line, the
   supervisor exited before flushing — branch to §4.1 (panic) or §4.2
   (manifest validation).
3. **Supervisor stderr** — captured by the calling driver (or the k8s
   container log; see Cross-cutting note 1). The supervisor's only error
   surface is one line of the form `supervisor: error: <msg>` followed by
   `std::process::exit(1)`. Match `<msg>` against the catalog in §4.1 / §4.2.
4. **Cargo build log** — only present if the codegen output failed to
   compile and the supervisor binary therefore never booted. Look for
   `error[E\d+]:` lines from rustc. Branch to §4.3.

If none of the four artifacts exists, the supervisor never started. Verify
the driver actually invoked the binary; this is **out-of-scope-of-runbook**
infrastructure: container scheduling, image pull, IRSA — escalate to
DevOps.

### Cross-cutting note 1 — Filter logs by tenant

DevOps's
[`nanofab-supervisor` Helm chart](../../../../repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/templates/_helpers.tpl)
emits a tenant label set on every chart object. The labels the SRE-owned
log/metric pipelines should filter on:

```
tenant: "<tenant-id>"                    # the SRE-canonical filter
app.kubernetes.io/name: "nanofab-supervisor"
app.kubernetes.io/instance: "<tenant-id>"
app.kubernetes.io/version: "<binary-tag>"
nanofab.resink.ai/dag-version: "<dag-version>"
nanofab.resink.ai/fleet-color: "blue" | "green" | "candidate"
```

The canonical filter is `tenant=<id>` (not `tenant_id=<id>`; the chart's
`_helpers.tpl` defines the bare key `tenant`). The label is sourced from
the chart's required `.Values.tenant` parameter — the template `fail`s if
unset, so an unlabeled supervisor pod is a chart-config bug, not a missing
label.

### Cross-cutting note 2 — `BLOCKED` observable is a known gap

> **`TODO: verify against impl` — runtime coordinator `BLOCKED` state.**
>
> The SRE OKR's KR5.3 (b) commits to documenting the supervisor's "blocked
> because sim verdict failed" observable. Read of the current MVP
> supervisor source ([`main.rs`](../../../../repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs))
> shows there is **no distinct `BLOCKED` state-machine surface**: `run()`
> returns `Result<(), String>`, `main()` either prints
> `supervisor: ok` and exits `0`, or prints
> `supervisor: error: <msg>` to stderr and calls
> `std::process::exit(1)`. Every non-success terminal state is collapsed
> into exit code `1` plus one stderr line.
>
> The runtime coordinator that would expose a richer `BLOCKED` state to
> the hot-swap state machine (per runtime spec §6) is **not in MVP
> scope**. It is expected to land after `dlopen`-based dynamic node
> loading restoration — earliest 2026-06-06, more likely later. Until
> then, the runbook treats the pair `(exit code 1, stderr line)` as the
> only available observable.
>
> Forward-pointer: when the coordinator surfaces a `BLOCKED` state with
> structured fields (verdict id, mismatch count, mismatch types,
> tenant), this runbook's General triage flow and §3 subsections should
> be revised to consume the structured surface instead of stderr
> grepping.

## 3. Sim-farm MVP verdict failures (Mode A)

The sim-farm diff engine
[(verdict contract)](../../../application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md)
runs after the supervisor writes its dim output parquet(s). The engine
itself produces `workspace/verdict.json` and one of three exit codes:
`0` (overall pass), `1` (at least one mismatch in at least one dim), or
`2` (engine-internal failure).

This section covers exit codes `1` and `2`. Exit code `0` is the green
path and is not an alert surface.

**Version reference.** The contract referenced throughout this section is
`type: contract`, `engine_version: 0.2.0` (loop 2026-05-11-1113). The single-dim
shape is preserved byte-identically from `engine_version: 0.1.0`
(loop 2026-05-11-0958) except for the `engine_version` field itself. The
multi-dim shape (top-level `overall_pass` + `verdicts[]`) is new in
`0.2.0`. Subsections below use the multi-dim shape's `verdicts[i].mismatches[]`
path; for single-dim runs, drop the `verdicts[i].` prefix.

### 3.1 `missing` rows (Mode-A verdict)

**Symptom.** Sim-farm engine exits `1`. Final stderr line:
`verdict=fail mismatches=<N> see <verdict-path>`. The
`workspace/verdict.json` contains at least one
`verdicts[i].mismatches[]` entry with `"type": "missing"`. Operationally:
**the fixture has rows the supervisor's output is missing** — the
supervisor either skipped an event, processed an event with a code path
that didn't `Op::Insert`, or dropped a row that should have been retained.

**Diagnosis.**

```bash
# 1. List all missing keys.
jq -r '.verdicts[]? // . | .mismatches[]? | select(.type=="missing")
       | .key' workspace/verdict.json
# Single-dim: drop the .verdicts[] step:
#   jq -r '.mismatches[] | select(.type=="missing") | .key' workspace/verdict.json

# 2. For each missing key, grep the trace JSONL for the event_id that
#    *should* have produced an Insert mutation. The trace records every
#    state mutation per runtime spec §8.4.
jq -c 'select(.key.user_id=="u-007")' workspace/<write-trace>.jsonl

# 3. If trace shows NO record for the key: the supervisor never processed
#    the relevant event. Check the fact stream parquets for that user_id;
#    suspect upstream codegen / fixture mismatch.
# 4. If trace shows a record but with op != Insert (or a later Delete):
#    the codegen's process() produced wrong mutations. File against
#    teams/application/resink-core for the codegen path.
```

The trace records carry `node_name` (`DimUserScd2Maintainer`), `node_version`,
`event_id`, and the mutation kind. Cross-reference `event_id` against the
two fact-stream parquets (`fact_sign_up`, `fact_profile_update`) named in
the manifest's `node_spec.fact_streams[]`.

**Mitigation.**

1. **Pause downstream consumers.** This is the most important action: if
   any downstream stage (training stage-3, hot-swap) is consuming the
   supervisor's `<write-output>` parquet, halt it now. A `missing` row
   means a customer query for that key will return a stale or missing
   answer.
2. **Do not re-dispatch the supervisor.** Re-dispatching the same
   manifest + same fact streams reproduces the same defect. The fix is
   either upstream (fixture/fact-stream content) or codegen-side
   (`process()` logic). Filing it as a Mode-A `missing` reproducer ticket
   on `teams/application/resink-core` is the next step.
3. **If the codegen path is suspect, request a re-emit of the codegen
   output crate** from the orchestrator (the
   `<workspace>/nodes/<table>_scd2/` crate). Cargo-rebuild the supervisor
   binary against the new crate; re-run.
4. **For training-stage-3 gates,** mark the verdict's `verdict_id` as
   FAIL in the training validator's blocklist.

**Severity.** **S1** when the verdict feeds a hot-swap or
training-stage-3 gate (downstream wrong-data risk). **S2** in pure MVP
dev-loop context (`make mvp-loop`) because the Makefile gates on
non-zero exit and refuses to proceed.

### 3.2 `extra` rows (Mode-A verdict)

**Symptom.** Sim-farm engine exits `1`. `verdicts[i].mismatches[]`
contains at least one entry with `"type": "extra"`. Operationally:
**the supervisor's output has rows the fixture does not** — the codegen
produced an `Op::Insert` for a key that should not exist in the
SCD2 truth, or duplicated an SCD2 row with a `valid_from` not in the
fixture.

**Diagnosis.**

```bash
# 1. List all extra keys.
jq -r '.verdicts[]? // . | .mismatches[]? | select(.type=="extra")
       | .key' workspace/verdict.json

# 2. For each extra key, fetch the row from the supervisor's output:
jq -r '.verdicts[]? // . | .mismatches[]? | select(.type=="extra")
       | .output_row' workspace/verdict.json

# 3. Grep the trace for the event_id that produced the extra Insert:
jq -c '.event_id as $e | select(.event_id==$e and .mutation_kind=="insert")' \
   workspace/<write-trace>.jsonl

# 4. Inspect the fact streams: was there a fact event for this key that
#    the codegen incorrectly mapped to an Insert when the fixture (truth)
#    treats it as a no-op?
```

**Mitigation.**

1. **Pause downstream consumers** (same as 3.1: extras cause customer
   queries to return rows that shouldn't exist).
2. **Do not re-dispatch the supervisor** unchanged — same defect will
   recur.
3. **Suspect the codegen `process()` first**, then the fact streams. If
   `process()` is correctly mapping fact events to mutations, the
   fixture itself may be wrong — coordinate with
   `teams/application/sim-farm` on whether the fixture or the codegen
   reflects truth.
4. **For Mode-A specifically**, an extra row often indicates an SCD2 row
   close-out that didn't fire (i.e., the codegen Inserted a new version
   but didn't Update the previous version's `valid_to` to close it).
   Cross-check the previous row in the output via trace records.

**Severity.** **S1** when reaching a hot-swap or training-stage-3 gate.
**S2** in dev-loop context.

### 3.3 `diverged` rows (Mode-A verdict)

**Symptom.** Sim-farm engine exits `1`. `verdicts[i].mismatches[]`
contains at least one entry with `"type": "diverged"`. Operationally:
**both the fixture and the output have a row at the SCD2 key
`(user_id, valid_from)`, but at least one payload column differs**
under `IS NOT DISTINCT FROM` (NULL-safe equality on `email`, `country`,
`valid_to`, `is_current`).

**Diagnosis.**

```bash
# 1. List diverged keys with both rows side-by-side:
jq -r '.verdicts[]? // .
       | .mismatches[]? | select(.type=="diverged")
       | {key, fixture: .fixture_row, output: .output_row}' \
   workspace/verdict.json

# 2. For each diverged row, identify the payload columns that disagree.
#    The Mismatch object carries both .fixture_row and .output_row in
#    full, so column-diffing is local.

# 3. Grep the trace for every mutation tied to the diverged key. SCD2
#    rows are produced by potentially several events (insert + later
#    update + close-out); a divergence on the *current* row may stem
#    from a mis-ordered or dropped earlier event.
jq -c 'select(.key.user_id=="u-003")' workspace/<write-trace>.jsonl
```

**Mitigation.**

1. **Pause downstream consumers** (diverged rows produce wrong customer
   answers in the most insidious way: the row exists with the right
   key, but the data is wrong).
2. **Bucket the divergence by column.** A single column diverging
   across many rows points at one upstream defect (e.g., country code
   normalization). Many columns diverging on one row points at a
   row-level codegen bug.
3. **Do not re-dispatch the supervisor** unchanged. File against
   `teams/application/resink-core` (codegen) or
   `teams/application/sim-farm` (fixture truth) depending on which
   side the operator's analysis implicates.
4. **Specifically for SCD2,** divergence on `valid_to` or `is_current`
   usually indicates the codegen failed to close a previous version
   when a new one arrived — see also 3.2 (`extra`).

**Severity.** **S1** when reaching a hot-swap or training-stage-3 gate.
**S2** in dev-loop context.

### 3.4 `engine_failure` (engine exit code 2)

**Symptom.** Sim-farm engine exits `2`. Final stderr line:
`verdict=engine_failure error=<ExceptionClass>: <message>` followed by a
Python traceback. The `workspace/verdict.json` exists (best-effort
write per
[contract § Exit-code semantics](../../../application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md))
and includes an `engine_error` field carrying `<ExceptionClass>: <message>`.
In multi-dim mode `overall_pass: false` and `verdicts: []`. Operationally:
**the diff engine itself failed** — bad parquet path, DuckDB query
exception, schema mismatch the engine cannot classify as `extra` /
`diverged`.

**Diagnosis.**

```bash
# 1. Read the engine_error field.
jq -r '.engine_error' workspace/verdict.json
# Examples:
#   "FileNotFoundError: [Errno 2] No such file or directory: '/.../output.parquet'"
#   "duckdb.duckdb.BinderException: Column 'is_current' is missing in fixture"
#   "ValueError: --fixture count (2) != --output count (1)"

# 2. Distinguish the three engine-failure subclasses:
#    a) Bad path: the supervisor didn't write the output parquet at all.
#       Branch to §4.x (supervisor failure) instead — the engine never
#       had data to diff.
#    b) DuckDB query exception (OOM, syntax): the engine ran but blew up
#       on a query. The contract's first-retry-on-smaller-sample logic
#       per spec §6.3 is post-MVP; in MVP the engine just exits 2.
#    c) Schema mismatch the engine refused to classify: structural
#       diffs (column added on output side that fixture doesn't know
#       about) SHOULD surface as extra/diverged rows, NOT exit 2; if
#       this happens, it's a contract violation by the engine. File on
#       teams/application/sim-farm.
```

**Mitigation.**

1. **Branch by subclass.** If the engine's `<ExceptionClass>` is
   `FileNotFoundError`, the supervisor didn't write its output —
   triage as a supervisor failure (§4.1, §4.2, or §4.3) first; the
   engine-failure is downstream noise.
2. **DuckDB-side failures (OOM, query):** re-run with the same inputs
   to confirm determinism; if reproducible, file against
   `teams/application/sim-farm` with the verdict path and tenant label
   attached. Do not retry from the same invocation: the contract
   forbids silent retries (
   [verdict contract § 6.7](../../../application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md);
   ditto sim-farm spec §6.7).
3. **Contract-violation case** (schema mismatch as engine failure):
   S3 escalation to sim-farm; the engine should have classified the
   rows. Block the run until classified.
4. **`make mvp-loop` Makefile** must distinguish exit `1` and exit `2`
   in its `verdict=…` print — see verdict contract § Invocation
   contract.

**Severity.** **S2** by default — the engine refused to deliver a
verdict; no bad data went downstream because nothing downstream
consumes an `engine_error` verdict. **S3** for the contract-violation
case (the bug is on the engine, no operational stoppage required
beyond filing the ticket). **S1** only if `verdict.json` is missing
entirely AND the calling driver did not gate on the non-zero exit
code (i.e., a regression in `make mvp-loop` / hot-swap caller).

## 4. Supervisor-side failures

These are the failure surfaces of the supervisor binary itself. They
fire either before any sim-farm engine invocation (the supervisor never
wrote `<write-output>`, so the engine had nothing to diff) or as a
build-time precondition (the supervisor binary doesn't exist because
its codegen-output dependency didn't compile).

### 4.1 Supervisor `panic::catch_unwind` path (codegen `process()` panic)

**Symptom.** Supervisor stderr ends with:
`supervisor: error: panic processing event <event_id>`. Exit code `1`.
The `<write-output>` parquet is **not** written (the supervisor returns
`Err` before reaching
[`output::write_dim_user_parquet`](../../../../repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs)).
The `<write-trace>` JSONL contains zero or more mutation records up to
but not including the panicking event. The downstream sim-farm engine
then exits `2` with `FileNotFoundError` on the absent output (see §3.4
subclass (a)).

**Diagnosis.**

```bash
# 1. Read the event_id from the supervisor stderr line.
grep -E '^supervisor: error: panic processing event' <stderr-log>

# 2. Look up the event in the fact-stream parquets — what fact_streams
#    name and what event payload made the codegen's process() panic?
#    The supervisor logs a startup line one above the error:
#      [supervisor] loaded <N> events from <M> streams
#    so the event_id is one of N events the supervisor saw.

# 3. The supervisor wraps process() in panic::catch_unwind per
#    main.rs lines 172-176. The catch_unwind ABSORBS the panic and the
#    main thread converts it into a String error; there is no Rust
#    backtrace surfaced through stderr in the MVP.
#    To get a backtrace, re-run with RUST_BACKTRACE=1 ON A LOCAL FIXTURE
#    that reproduces the same event (production pods do not capture
#    backtraces today — see KR5.3(b) note above).

# 4. Inspect the codegen-output crate's process() impl at
#    <workspace>/nodes/<table>_scd2/src/lib.rs for unsafe unwraps,
#    array OOB, or Option::unwrap() on input fields.
```

**Mitigation.**

1. **No re-dispatch.** Re-dispatching the same manifest reproduces the
   panic on the same event_id. The fix is upstream in
   `teams/application/resink-core` codegen.
2. **Capture & pin the panicking event.** Extract the panicking event's
   row from its fact-stream parquet and attach to the resink-core
   ticket; that row is the minimal reproducer.
3. **Pause downstream consumers** as in §3.1-3.3 — the supervisor's
   `<write-output>` is absent or partial.
4. **Trace integrity check.** The MVP supervisor flushes the trace
   writer only on the green path
   ([`main.rs` line 200](../../../../repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs));
   on the panic path the trace may have buffered records that never
   reached disk. Treat the trace JSONL as best-effort for panic-failed
   runs.

**Severity.** **S2** — the supervisor refused to proceed, exited
non-zero, and the downstream engine fails cleanly on the absent output.
The panic itself is a codegen defect requiring root-cause triage, not a
silent-failure incident. **S1** only if a downstream consumer ignores
exit `1` and reads a partial `<write-output>` (today this can't happen
because the supervisor never writes on the panic path; if a future
revision changes this, re-classify).

### 4.2 Manifest-validation failure (coordinator/supervisor rejects malformed seal)

**Symptom.** Supervisor stderr ends with one of:

- `supervisor: error: only --mode=sim is supported in the MVP; got <other>`
  (mode flag wrong;
  [`main.rs` line 109](../../../../repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs)).
- `supervisor: error: read manifest: <io error>` (manifest.yaml missing
  or unreadable; line 94).
- `supervisor: error: parse manifest: <serde-yaml error>` (manifest is
  malformed YAML or schema mismatch; line 95).
- `supervisor: error: expected exactly 1 DAG in manifest, got <N>`
  (DAG count violation; lines 114-115).
- `supervisor: error: expected exactly 1 node in DAG <name>, got <N>`
  (node count violation; lines 118-123).
- `supervisor: error: unknown op <op>` (fact-stream `op` not in
  `{insert, update, delete}`;
  [`main.rs` line 103](../../../../repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs)).

Exit code `1`. No `<write-output>` written; no `<write-trace>` opened
(failures fire before trace-writer construction at line 164).

**Diagnosis.**

```bash
# 1. Read the supervisor stderr; the message names the specific clause.

# 2. For "expected exactly 1 DAG / 1 node" rejections:
#    - The MVP supervisor enforces single-DAG, single-node manifests.
#    - If the orchestrator was upgraded to emit multi-node manifests
#      (cf. CEO brief 2026-05-23 O1 KR1.2 — multi-dim widening) and
#      this supervisor binary was NOT rebuilt against the matching
#      resink-core version, the supervisor will reject the new manifest.
#    - Check the binary's build provenance: which version of the
#      orchestrator and which manifest version? Mismatch = bump the
#      supervisor binary; not a manifest defect.
cat <workspace>/manifest.yaml | yq '.dags | length'
cat <workspace>/manifest.yaml | yq '.dags[0].nodes | length'

# 3. For "unknown op": inspect the manifest fact_streams entries.
yq '.dags[].nodes[].fact_streams[].op' <workspace>/manifest.yaml
# Expected: every value in {insert, update, delete}.

# 4. For "parse manifest" / "read manifest": validate the manifest
#    against the orchestrator's emitter contract (per resink-core's
#    Manifest emitter spec; today: hand-built YAML for the MVP fixture).
```

**Mitigation.**

1. **Do not re-run the same manifest.** A manifest-validation failure
   is deterministic; the rejection condition does not depend on
   external state.
2. **Determine the failure side.** If the orchestrator emitted a
   manifest that the supervisor cannot consume, file against
   `teams/application/resink-core` (or DevOps if a CI-gate seal
   validator should have caught the malformed manifest earlier — per
   the boundary in [SRE charter §
   Interfaces](../charter.md)). If the manifest is correct and the
   supervisor's binary is stale, redeploy the supervisor.
3. **Multi-dim widening note.** O1 of the
   [2026-05-23 CEO brief](../../../../board/okrs/2026-05-11-1113-ceo-brief.md)
   commits the orchestrator to emit multi-node manifests this loop. If
   the supervisor's `main.rs` is still at the single-node check
   (`nodes.len() != 1`), the rejection is expected for multi-node
   inputs. The runbook treats this as a **transitional S3** — a
   contract-version gap that resolves when the supervisor catches up
   to multi-node. If the supervisor *should* be multi-node but is
   rejecting, re-classify as **S2** and file as a supervisor regression.
4. **CI gate is the correct prevention surface.** Per
   [SRE charter § Boundary with DevOps](../charter.md), DevOps owns
   the manifest-signature / Sim Farm seal validator at the
   coordinator. A manifest rejection at the supervisor that the
   coordinator should have caught earlier is a defect in the CI gate;
   file against DevOps for the gate, against resink-core for the
   manifest.

**Severity.** **S2** by default — supervisor refused to proceed; no
bad data downstream; operator triage needed for which side authored
the malformed seal. **S3** for the transitional case (single-node vs
multi-node supervisor lag in a known-in-flight migration) described
in mitigation step 3.

### 4.3 Cargo-build failure (codegen output doesn't compile — orchestrator stage-1 gate)

**Symptom.** The supervisor binary at
`crates/nanofab-supervisor/target/release/nanofab-supervisor` does
not exist or is stale. `cargo build -p nanofab-supervisor` fails with
one or more `error[E\d+]: ...` lines. The failure is typically inside
the path-dep `nanofab-node-dim-user-scd2` (the codegen-output crate at
`<workspace>/nodes/<table>_scd2/`), not in the supervisor crate's own
sources. The supervisor never boots; no `<write-trace>` or
`<write-output>` is ever opened; no `workspace/verdict.json` is ever
written. The operator sees the failure at the orchestrator's
**stage-1 gate** (compile-the-tenant-binary), before any runtime
artifacts exist.

**Diagnosis.**

```bash
# 1. Re-run cargo with full diagnostics:
cargo build -p nanofab-supervisor --release 2>&1 | tee /tmp/build.log

# 2. Filter for compile errors:
grep -nE '^error\[E[0-9]+\]:' /tmp/build.log

# 3. Common classes:
#    a) Trait-impl mismatch — codegen emitted a process() with the
#       wrong signature (e.g., wrong &mut Ctx type). Means the codegen
#       template is out of sync with the supervisor's nanofab-node API.
#    b) Missing crate / unresolved import — the codegen output crate
#       lost its Cargo.toml or its Cargo.toml's path-dep was renamed.
#    c) Unsafe / unsoundness lints upgraded to errors after a rustc
#       version bump — pin the toolchain in the workspace's
#       rust-toolchain.toml.

# 4. The Wave-2 Option-A ABI decision (per OKR context) means the
#    supervisor is built per-tenant with a hard-coded path-dep on
#    <workspace>/nodes/<table>_scd2. Inspect that crate's Cargo.toml
#    and src/lib.rs.
cat <workspace>/nodes/<table>_scd2/Cargo.toml
head -50 <workspace>/nodes/<table>_scd2/src/lib.rs
```

**Mitigation.**

1. **Do not skip the stage-1 gate.** Per the OKR, this gate's purpose
   is to surface codegen defects before any binary boots. A workaround
   that hand-edits the codegen crate to "make it compile" defeats the
   gate; file against resink-core instead.
2. **Pin the failing codegen output.** Attach the failing
   `<workspace>/nodes/<table>_scd2/` directory tarball to the
   resink-core ticket; the codegen emitter that produced it is the
   defect surface.
3. **Toolchain drift,** if implicated, is fixed by the
   `rust-toolchain.toml` in the resink-core repo; coordinate with
   resink-core for the pin.
4. **No downstream impact** — the supervisor never ran, so no traces,
   no outputs, no verdicts, no customer-visible behavior change.
5. **CI gate placement.** Per
   [SRE charter § Boundary with DevOps](../charter.md), DevOps owns
   the CI/CD gate. A cargo-build failure at the stage-1 gate is the
   gate working as designed. File the ticket on resink-core (the code
   produced the defect); copy DevOps for awareness.

**Severity.** **S2** — supervisor never booted; gate fired correctly;
no wrong data downstream; clear root-cause triage owner
(resink-core codegen). **S3** for transient toolchain-drift cases that
resolve with a clean re-checkout.

## 5. Out of scope (this revision)

The runbook deliberately does not cover the following classes; these
land as separate runbooks in future loops, per the 2026-05-23 SRE OKR's
[§ Out of scope](../okrs/2026-05-11-1113-team-okr.md):

- **KV cluster failures** (the in-memory `MemKv` is MVP; production
  uses a persistent KV — separate runbook when that lands).
- **Kafka broker loss** (no Kafka in MVP scope; supervisor reads
  parquets, not Kafka).
- **Iceberg snapshot replication lag** (no Iceberg in MVP scope; the
  verdict is a JSON file on disk).
- **Region failover** (single-AZ MVP; multi-region post-MVP per SRE
  charter §5).
- **SLO-per-supervisor-pod operational response** (deferred; depends
  on the on-call rotation policy that has not yet been authored).
- **Alerting / paging wiring** (no production sub-project #4
  deployment yet; DevOps's first slice is Helm-chart-shaped per
  CEO brief O4).
- **The runtime coordinator hot-swap state machine** (post-Mode-B; the
  `BLOCKED` observable is a known gap, see Cross-cutting note 2).
- **Sub-project #5 (Product UX) operational surfaces** (ownership
  deferred per CEO brief).

## 6. Post-MVP stubs

The following failure modes will be covered in this runbook when the
corresponding sim-farm mode ships. Each is a one-paragraph stub with a
forward-pointer to the spec section, so a future-loop operator can
expand it in place.

### 6.1 Sim-worker pod crash / `INFRA_FAILURE` (post-Mode-B / post-Mode-C)

Per sim-farm spec
[§6.1](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md):
the coordinator detects a sim-worker pod crash via gRPC stream EOF + pod
status, marks the job FAILED, and surfaces
`Verdict{overall: INFRA_FAILURE, layers: empty}` to the caller. The
runbook needs to cover: how SRE distinguishes an `INFRA_FAILURE`
verdict from a Mode-A engine-failure exit code 2, the retry policy
(training retries 2×; runtime treats it per
`allow_simfarm_unavailable_for_blue_green`), and the operator path when
a pod-crash signal is genuinely a node-level (host-down) outage rather
than an application bug. **Full coverage when Mode B or Mode C ships
in CI** — currently expected post-2026-06-06.

### 6.2 Mode-B sidecar pair-orphan handling (`orphan_count > 1%`)

Per sim-farm spec
[§6.4](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md)
and the runtime mid-shadow containment story: when a sidecar restarts
mid-session, in-flight LRU pairs are lost and the `sidecar_restart_lost_pairs`
counter increments; coordinator marks the session UNRELIABLE if
sidecar restarts exceed 3× in 5 min. The runbook needs to cover:
operator path for an UNRELIABLE session, the `orphan_count > 1%`
threshold (Mode-B specific containment), the SHADOW_DEGRADED escalation
to the runtime coordinator, and the "never silently keep reporting
partial data as complete" framing. **Full coverage when Mode B ships.**

### 6.3 Supervisor write-trace socket overrun (`trace_dropped` counter)

Per sim-farm spec
[§6.5](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md):
the supervisor's `--write-trace` socket is non-blocking with a bounded
`SO_SNDBUF` (8 MiB default); if a sidecar can't drain fast enough,
supervisor drops trace events and increments `trace_dropped`. The
sidecar's verdict heartbeat then carries `dropped_events_observed:
true`, coordinator marks the session DEGRADED. The runbook needs to
cover: how SRE detects `trace_dropped > 0` (metrics endpoint), the
operator path for a DEGRADED session, the "slow differ never produces
a falsely-clean verdict" invariant. **Full coverage when Mode B ships
with the sidecar.**

### 6.4 Mode-A coordinator crash / restart

Per sim-farm spec
[§6.6](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md):
coordinator state lives in a metadata store (Postgres/etcd); restart
re-attaches via job-id lookup; both-coordinator-and-sidecars-down
(regional outage) triggers a runtime-coordinator session-timeout abort
and a revert-to-live-fleet fallback. The runbook needs to cover: SRE's
detection path for coordinator unreachability, the conservative
revert-to-live-fleet semantics, the operator escalation when a
regional outage genuinely happened (vs a transient coordinator pod
restart). **Full coverage when the coordinator lands operationally** —
likely post-2026-06-06.

### 6.5 DuckDB diff failures beyond MVP exit-code-2

Per sim-farm spec
[§6.3](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md):
DuckDB OOM / syntax errors retry once with a smaller row sample before
emitting `Verdict{overall: DIFF_ENGINE_FAILURE}`; schema-mismatch
between candidate and reference is treated as structural diff
(extra/diverged rows), not engine failure. The MVP today
treats every DuckDB error as exit-code-2 with no retry. The runbook
needs to cover: how SRE distinguishes a "retried-and-still-failed"
DIFF_ENGINE_FAILURE from a single-shot MVP exit-2, the operator path
for OOM tuning, and the contract-version gate that promotes the
retry-once behavior. **Full coverage when the smaller-sample retry
behavior ships in the engine** — sim-farm-owned, no committed loop yet.

### 6.6 Supervisor `panic_event` trace-record (post-Mode-B per spec §6.2)

Per sim-farm spec
[§6.2](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md):
a panicked event is recorded in `trace.jsonl` as a `panic_event` and
treated as a stage-1 verdict failure
(`node_coverage=FAIL` with the panic node + event surfaced). The MVP
today does **not** write a `panic_event` trace record; the supervisor
just returns `Err` (see §4.1 above) and exits 1. The runbook needs to
cover: parsing `panic_event` trace records, the relationship between
a `panic_event` and the `node_coverage` layer, the operator path for a
hot-path panic vs an MVP cold-loop panic. **Full coverage when the
trace-writer is extended to emit `panic_event` records** —
resink-core-owned; depends on the broader trace v2 schema, no committed
loop yet.

## 7. Cross-references

- **Sim-farm verdict contract (`type: contract`,
  `engine_version: 0.2.0`):**
  [`teams/application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md`](../../../application/sim-farm/contracts/2026-05-16-mvp-loop-verdict.md).
  The `0.1.0` shape is preserved byte-identically in single-dim mode;
  the `0.2.0` shape adds `overall_pass` + `verdicts[]` for multi-dim.
- **Sim-farm design spec §6 (failure handling):**
  [`docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md`](../../../../docs/superpowers/specs/2026-05-10-nanofab-sim-farm-design.md).
- **Supervisor MVP source:**
  [`repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs`](../../../../repos/resink-ai/resink-core/crates/nanofab-supervisor/src/main.rs).
  Specific lines referenced above (subject to commit drift; pin at
  refresh time): mode check ~109, manifest read 93-95, DAG/node counts
  114-124, op-from-string 98-104, panic::catch_unwind 167-197,
  output write 200-208, main / exit 213-224.
- **DevOps Helm chart (tenant label injection):**
  [`repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/templates/_helpers.tpl`](../../../../repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/templates/_helpers.tpl).
  Canonical label is `tenant: <id>`; additional labels listed in
  Cross-cutting note 1.
- **SRE charter:**
  [`teams/platform/sre/charter.md`](../charter.md). Establishes the
  runbook storage convention (`teams/platform/sre/runbooks/<slug>.md`)
  and the SRE/DevOps boundary at the metric-emission seam.
- **2026-05-23 SRE OKR:**
  [`teams/platform/sre/okrs/2026-05-11-1113-team-okr.md`](../okrs/2026-05-11-1113-team-okr.md).
- **2026-05-23 CEO brief (O5 is the parent objective):**
  [`board/okrs/2026-05-11-1113-ceo-brief.md`](../../../../board/okrs/2026-05-11-1113-ceo-brief.md).
