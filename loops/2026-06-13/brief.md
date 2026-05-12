---
layout: default
title: Brief
nav_order: 1
parent: "Loop 2026-06-13"
grand_parent: "Loops"
render_with_liquid: false
date: 2026-06-13
status: active
type: okr
loop: 2026-06-13
owner: board
---

<!-- original-frontmatter:
  type: okr
  owner: board
  date: 2026-06-13
  status: active
  loop: 2026-06-13
  links: parent: ""
-->
# Resink.ai CEO Brief — 2026-06-13

## Context

The company has shipped six loops of structured org-os work. Every artifact lives in this monorepo's filesystem; every loop produces a CEO brief, an exec summary, a retro, team OKRs, team exec summaries, ADRs, contracts, runbooks, conventions, playbooks, and (since 2026-05-30) an HTML capabilities report. **The work is observable on disk and unobservable to anyone outside the immediate clone.** ORG.md's "AI-native realtime data processing engine" framing presumes an outside reader can see the company's progress; today they cannot. The HTML capabilities report at 2026-05-30 is a single artifact a reader can be handed; the rest of the org-os output requires a clone + a competent grep.

**The gap.** Distance from vision is now external observability. Six loops of work, two MVP demos GREEN, three multi-loop deviations closed, four ratified ADRs, two open board-action tickets, six teams' worth of exec summaries — none of it discoverable beyond `ls`. The user's framing for this loop names the deliverable directly: **publish the company's structured artifacts as a navigable site at `resink-ai/resink-ai.github.io` (CNAME `blog.resink.ai`) so AI progress is visible online — exec summaries, OKRs, specs, reports — served as a GitBook-style doc tree.**

**The shape of this loop.** A **build-and-publish loop** — a hybrid: code-bearing on the publishing pipeline + four ADR ratifications + the dlopen restoration plan's step 2 (resink-core supervisor swap full execution per ADR-2026-05-16-001). Five teams active; one paused with a single light ack ask. The publishing pipeline is the headline; everything else is concurrent track work that doesn't depend on it.

**Probe results entering the loop:**

- `git ls-remote git@github.com:resink-ai/resink-ai.github.io.git` → exists; `main:960c7e3`. Jekyll minimal-theme repo with CNAME `blog.resink.ai`. **Publishing target ready** — no gating board action needed.
- `git ls-remote git@github.com:resink-ai/resink-core.git` → still `Repository not found`. Workspace promotion 5th-loop carry. Board-action `2026-06-06-001`'s forcing function fires this loop (see Out-of-scope below).
- `which minikube` → `not found`. DevOps minikube smoke 4th-loop carry. Board-action `2026-06-06-002`'s forcing function fires this loop.

**Re P5 from the 2026-06-06 retro (toolchain installs as board-action tickets — meta-question):** **Working answer — board-action tickets are right for blockers on shared infrastructure (GitHub remotes, IAM roles, secrets); developer-machine toolchain installs are NOT board-action tickets — they're per-team Prerequisite documentation that the team running the smoke installs as part of their own build phase.** Implication: board-action `2026-06-06-002` (minikube install) is the LAST toolchain board-action; future toolchain blockers route through `<chart>/README.md § Prerequisites` per the team's own discipline. Action: this loop, board-action 2026-06-06-002 closes either as `done` (if the install lands) or as `superseded-by-team-prerequisite-pattern` (if not — DevOps's chart README already names `brew install minikube` in its Prerequisites; the next attempt picks it up via the team's own surface).

**Re ADR-2026-05-16-001 (dlopen restoration plan, loop+N convention applied):** Step 1 (AE template extension) closed at `loop-1` (= 2026-06-06). Step 2 (resink-core supervisor swap full execution) lands this loop = `loop+0`. Step 3 (hot-swap correctness test) targets `loop+1` (= 2026-06-20).

**CEO decisions for this loop:**

- **Activate four teams + board.** Resink-core + DE + DevOps + board active. Sim-farm + AE paused; one-paragraph paused-team ack each (sim-farm reviews resink-core's dlopen swap + DE's new schema-JSON conventions doc; AE reviews resink-core's consumption of the template extension to confirm the ABI contract held).

- **GitBook publishing pipeline (O1 below) is the headline this loop.** Add `resink-ai/resink-ai.github.io` as a submodule at `repos/resink-ai/resink-ai.github.io`. Author a publishing script that produces a Jekyll-organized site from the parent repo's structured artifacts. First publication: the cumulative loop history (2026-05-08 through 2026-06-13). Subsequent loops: incremental publish.

- **Ratify the 2026-06-13 ADR batch (4 ADRs).** All four flip `draft → active`:
  1. [`ADR-2026-05-30-001`](../decisions/2026-05-30-001-verify-predicted-syntax-in-okrs.md) — verify predicted syntax in OKRs at planning time. Full body authoring.
  2. [`ADR-2026-06-06-001`](../decisions/2026-06-06-001-provisional-and-migrate-playbook.md) — provisional-and-migrate coordination playbook.
  3. [`ADR-2026-06-06-002`](../decisions/2026-06-06-002-paused-team-request-acceptance.md) — paused-team request acceptance canonicalization.
  4. **NEW this loop:** ADR `2026-06-13-001` — admit `action` to the conventions enum (closes the provisional-and-migrate cycle for `board/actions/<date>-<slug>.md` artifacts; sister to ADR-2026-05-23-001's enum extension).

- **Resink-core executes ADR-2026-05-16-001 step 2.** Supervisor swap full execution against AE's template extension (loop-1's deliverable). `cargo build --features dlopen-plugins` → real plugin load against `target/release/lib<crate>.{so,dylib}` → `make mvp-loop` GREEN under `dlopen-plugins`-on. Step 3 (hot-swap test) reschedules to `loop+1` per the existing multi-loop plan.

- **DE canonicalizes the schema-JSON spec.** Per the in-loop request from sim-farm at 2026-06-06 (`teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md`). Author `teams/platform/data-engineering/conventions/dim-schema-json.md` as a sister convention to `duckdb.md`. Sim-farm's verdict contract gains a back-reference next loop (sim-farm paused this loop; mechanical migration there).

- **DevOps minikube smoke remains carry IF minikube is not installed in-loop.** Per the working answer to P5: future toolchain installs route through team Prerequisite docs, not board-action tickets. `2026-06-06-002` closes either `done` (if installed in-loop by the human) or `superseded-by-team-prerequisite-pattern` (if not). The chart's `README.md § Prerequisites` already names `brew install minikube`; DevOps re-attempts the smoke at the next loop in which the toolchain is present.

- **Workspace promotion (resink-core KR2.1) — 5th-loop carry.** Per the forcing function in `board/actions/2026-06-06-001`: this brief's Out-of-scope section names the implication explicitly with the count of downstream blockers (see § Out of scope). The board-action ticket stays `status: open`; if the human creates the remote in-loop, resink-core picks up the promotion attempt opportunistically (no KR; opportunistic only).

**Standing CEO answers (closing carryover loops):**

- **GitBook publishing scope (what's in / what's out):**
  - **In scope (canonical published artifacts):** `board/okrs/<date>-ceo-brief.md`, `board/exec-summaries/<date>.md`, `board/retros/<date>-ceo-retro.md`, `board/decisions/<date>-NNN-<slug>.md` (all ADRs, both `draft` and `active`), `board/actions/<date>-NNN-<slug>.md` (all action tickets, all states), `board/reports/*` (markdown + HTML), `teams/<layer>/<team>/exec-summaries/<date>.md`, `teams/<layer>/<team>/okrs/<date>-team-okr.md`, `teams/<layer>/<team>/charter.md`, `teams/<layer>/<team>/contracts/*.md`, `teams/<layer>/<team>/conventions/*.md`, `teams/<layer>/<team>/runbooks/*.md`, `ORG.md` (root), and the design specs at `docs/superpowers/specs/*.md`.
  - **Out of scope (NOT published):** `org-os/` (engine internals; tenant-agnostic doc set; if published later, lives under a separate namespace like `/engine/`), `teams/<layer>/<team>/status.md` (high-frequency internal state; refreshed each loop), `teams/<layer>/<team>/proposals/*.md` (often pending or accepted-and-folded-in), `teams/<layer>/<team>/requests/*.md` (cross-team coordination; should the requests appear in linked OKRs but not as standalone pages — defer that decision; this loop they're out), product-repo source code under `repos/resink-ai/resink-core/{crates,training,sim-farm,deploy,synthetic_tenants,target}/` (the `docs/` tree there IS in scope as part of resink-core's docs).
  - **Site organization (Jekyll):** Landing page links to "Latest loop", "Decisions", "Reports", "Teams". Loop pages live under `/loops/<date>/` with a small index per loop (brief + exec summary + retro + per-team summaries). Decisions index sorts by date with status filter. Reports index lists HTML reports. Teams index has one page per team with charter + chronological exec-summary list.
  - **Publishing trigger:** Manually invoked `python3 scripts/publish-to-gitbook.py` produces the Jekyll tree under the submodule + commits + pushes. CI integration is a follow-up (sized at next loop).

- **Resink-core dlopen swap acceptance:** `cargo build --no-default-features --features dlopen-plugins` exit 0; `cargo test --features dlopen-plugins` exit 0 (with the no-op shim path retained for unit-test sanity); a new integration test that builds a generated plugin's `.so`/`.dylib` from AE's template, loads it via `libloading::Library::open`, invokes `nanofab_node_process` on a known event, asserts return code `NANOFAB_NODE_OK = 0` and output node state matches the static-linking-path output byte-for-byte. `make mvp-loop` GREEN under both `static-plugins` (existing) AND `dlopen-plugins` (new); verdict.json shape unchanged.

- **DE schema-JSON conventions doc shape:** New file at `teams/platform/data-engineering/conventions/dim-schema-json.md`, `type: convention` (canonical, not provisional — the `convention` type was admitted at 2026-05-30). One section names the JSON shape (`{"key_columns": [string], "payload_columns": [string]}`), one section gives the worked examples (`dim_user`, `dim_account`), one section names the consumers (sim-farm engine 0.3+, resink-core orchestrator), and one section names the `Verified-against-environment` per ADR-2026-05-16-003 (the second contract/convention to honor the discipline). Sim-farm's `2026-05-16-mvp-loop-verdict.md` contract gains a back-reference next loop.

- **GitBook publishing's first-loop content:** Cumulative loop history from 2026-05-08 through 2026-06-13 inclusive, plus the existing HTML capabilities report and ORG.md. The publishing script processes the full repo at run time; "first loop" is just whatever exists when it runs. No backfill — the structured artifacts already span seven loops.

**Carryover load by team** (per ADR-2026-05-09-005):

| Team | Carried items | Sized | Re-shape note |
|---|---|---|---|
| board | GitBook publishing pipeline (new); 4-ADR ratification batch; multi-loop plan absorption for ADR-2026-05-16-001 step 2 closure record; board-action ticket forcing-function decisions (close 06-06-002 either way; 06-06-001 stays open as 5th-loop carry) | L | Headline-heavy; gitbook + 4 ADRs + 1 ADR-001 step-record |
| resink-core | Dlopen restoration step 2 (full execution against AE's template); workspace promotion (opportunistic only, no KR; gated on board action ticket); orchestrator updates if the dispatch.py auto-detect needs revision | M | Single bounded code deliverable + opportunistic carry |
| DE | Schema-JSON conventions doc canonicalization (closes 2026-06-06-001 sim-farm request); optional `<verified-against-rust-impl>` tag flip (XS, opportunistic); supports board's GitBook scope decisions on contracts/conventions | M | First DE-owned `conventions/` artifact under canonical type |
| DevOps | Minikube smoke (carries unless minikube installed in-loop — per P5 working answer, no this-loop KR); supervisor-side hand-off responses already absorbed at 2026-06-06; supports board's GitBook scope decisions on chart README + values.schema.json publishing | XS–S | Minikube carry; gitbook participation is responding to scope queries only |
| sim-farm | Schema-aware engine 0.3 GREEN on master (no carry); paused this loop; reviews DE's new conventions doc + resink-core's dlopen swap; verdict-contract back-reference to DE conventions next loop | XS | Paused; review-ack only |
| AE | No carry from 2026-06-06 (Bundle C closed, dlopen template step 1 closed); paused this loop; reviews resink-core's consumption of the template extension to confirm the ABI contract held under real dlopen | XS | Paused; review-ack only |

## Objectives

### O1: Board ships the GitBook publishing pipeline and first-publication

source: ceo-brief

Why it matters: The company's structured org-os output (six loops of briefs, exec summaries, retros, ADRs, contracts, runbooks, conventions, action tickets, reports) is observable on disk only. The "AI-native organization" claim from `ORG.md` presumes outside readers can see the company's progress; today they require a clone + a grep. Publishing the artifacts as a navigable Jekyll site at `blog.resink.ai` (CNAME of `resink-ai/resink-ai.github.io`) closes that gap — exec summaries, OKRs, specs, ADRs, action tickets, reports all become navigable from a URL. This is the headline external-facing observability artifact for the company. Scope decisions (what's in, what's out, how it's organized) are documented above; this objective ships the implementation.

**Key results**
- KR1.1: `resink-ai/resink-ai.github.io` added as a submodule at `repos/resink-ai/resink-ai.github.io`. Submodule URL: `git@github.com:resink-ai/resink-ai.github.io.git`. The submodule pointer at parent-master tracks the published tip.
- KR1.2: A publishing script at `scripts/publish-to-gitbook.py` (Python 3.12+) reads the in-scope artifacts (per the standing-answer scope above), produces a Jekyll-organized tree under the submodule, and commits + pushes. Script is idempotent: re-running with no parent-repo changes produces an empty diff in the submodule. Documented in a `scripts/README.md` (or appended to existing) with usage + scope description.
- KR1.3: Site organization implemented per the standing-answer Jekyll layout. At minimum: landing page (`index.md`); chronological loop index page (`/loops/index.md`); per-loop pages under `/loops/<date>/` containing the brief, exec summary, retro, and per-team OKR + exec-summary content (or links to standalone team pages — author decides during build); decisions index page (`/decisions/index.md`) listing all ADRs sorted by date with `status` filter; actions index page (`/actions/index.md`) listing all `board/actions/` tickets; reports index page (`/reports/index.md`) linking the HTML capabilities report and the markdown source; teams index page (`/teams/index.md`) with one page per team containing charter + chronological exec-summary links + contract/convention/runbook links.
- KR1.4: First publication runs cleanly. Manually invoke the script; commit the submodule with a `Loop 2026-06-13: first publication` commit message; push to `main` on the GitHub Pages repo. Verify the site builds via GitHub Pages (the repo's existing `_config.yml` uses `jekyll-theme-minimal`; either keep that theme or upgrade to a more navigable theme like `just-the-docs` if the publishing tree benefits — author decides). After push, the site at `https://blog.resink.ai` (or `https://resink-ai.github.io` if CNAME isn't yet propagated) renders the new content.
- KR1.5: Tenant-isolation invariant: this objective writes ONLY to `repos/resink-ai/resink-ai.github.io/` (a tenant-only path) and `scripts/publish-to-gitbook.py` (tenant-only path). It does NOT write to `org-os/`. Publishing-scope filter explicitly excludes `org-os/` per the standing answer; the script enforces this at filter time (refuses to copy any file under `org-os/`).
- KR1.6: Publishing-scope decisions documented in `scripts/publish-to-gitbook.py`'s module docstring + a `scripts/PUBLISHING-SCOPE.md` (or appended block in `scripts/README.md`) so future scope changes have a single source-of-truth. Cross-link from this brief.

**Tasks**
- [ ] Add `resink-ai/resink-ai.github.io` as a submodule at `repos/resink-ai/resink-ai.github.io` — owner: board.
- [ ] Author `scripts/publish-to-gitbook.py` with the in-scope filter + Jekyll layout generator + commit/push step — owner: board.
- [ ] Implement the site organization (landing, loops index, per-loop pages, decisions, actions, reports, teams) — owner: board.
- [ ] Run the script for first publication; verify clean submodule diff; commit + push the submodule; commit the parent submodule pointer bump — owner: board.
- [ ] Verify the rendered site at `blog.resink.ai` (or fallback `resink-ai.github.io`) — owner: board.
- [ ] Document scope at `scripts/PUBLISHING-SCOPE.md` — owner: board.
- [ ] Tenant-isolation dry-run (the script writes to repos/resink-ai/resink-ai.github.io/ and scripts/; no org-os/ writes) — owner: board.

### O2: Board ratifies the 2026-06-13 ADR batch (4 ADRs)

source: ceo-brief

Why it matters: Three ADRs have been in `draft` across the last two retros (one from 2026-05-30 retro: P2 verify predicted syntax in OKRs; two from 2026-06-06 retro: P2 provisional-and-migrate playbook + P3 paused-team request acceptance). A fourth, sister to ADR-2026-05-23-001's enum extension, admits `action` to the conventions enum so the two open `board/actions/` tickets and the future-pattern artifact tree become canonical (no longer provisional). Ratifying all four in one batch is consistent with the 2026-05-30 + 2026-05-23 batching practice (3+ ADRs ratified together rather than serialized).

**Key results**
- KR2.1: ADR `2026-05-30-001` (verify predicted syntax in OKRs) flips `draft → active`. Body authored per `org-os/templates/adr.md` shape. Mandated edits: extend `org-os/rituals/team-planning.md` step 3 (or step 4) to require predicted-syntax markers (e.g., `<predicted: claude --skill <name>>`) when an OKR mentions a CLI flag, file path, or env var that doesn't yet exist on the team's working branch; extend `org-os/rituals/build.md` step 3 to convert markers into confirmed references or addenda. Existing OKRs grandfather; future OKRs comply.
- KR2.2: ADR `2026-06-06-001` (provisional-and-migrate playbook) flips `draft → active`. Body authored. Mandated edit: author new `org-os/playbooks/provisional-and-migrate.md` with `type: playbook` codifying the pattern (when to use, shape, three worked examples cited from prior loops, anti-pattern). Cross-link from `org-os/conventions.md` § "frontmatter."
- KR2.3: ADR `2026-06-06-002` (paused-team request acceptance) flips `draft → active`. Body authored. Mandated edits: `org-os/rituals/team-intake.md` specifies request file as canonical source-of-truth + new optional `deferred_to_loop: <YYYY-MM-DD>` frontmatter field on requests; `org-os/templates/request.md` documents the optional field; `org-os/rituals/ceo-brief.md` step 1 mentions surfacing any request hitting its deferred-loop this loop.
- KR2.4: NEW ADR `2026-06-13-001` (admit `action` to conventions enum) flips `draft → active` in the SAME loop it's drafted (sister to ADR-2026-05-23-001's enum-extension precedent). Body authored. Mandated edits: `org-os/conventions.md` admits `type: action` to the enum; "Additional fields per type" gains a row for `action` (required: `due:`, `status: open|done|superseded`); the two existing `board/actions/2026-06-06-001*.md` and `board/actions/2026-06-06-002*.md` migrate from provisional `type: action` to canonical (the in-body provisional notes get removed; same provisional-and-migrate pattern AE used at 2026-05-30 for `rfc → role`).
- KR2.5: Tenant-isolation invariant holds across all 4 ADR mandated edits to `org-os/`. Per-ADR dry-run + cumulative final dry-run.

**Tasks**
- [ ] Author full body for ADR-2026-05-30-001; apply mandated edits to `org-os/rituals/team-planning.md` + `org-os/rituals/build.md` — owner: board.
- [ ] Author full body for ADR-2026-06-06-001; author new `org-os/playbooks/provisional-and-migrate.md` — owner: board.
- [ ] Author full body for ADR-2026-06-06-002; apply mandated edits to `org-os/rituals/team-intake.md` + `org-os/templates/request.md` + `org-os/rituals/ceo-brief.md` — owner: board.
- [ ] Draft + ratify NEW ADR-2026-06-13-001 (admit `action` to enum) in the same loop; apply mandated edits to `org-os/conventions.md`; migrate the two existing `board/actions/` tickets from provisional to canonical — owner: board.
- [ ] Per-ADR tenant-isolation dry-run + cumulative final sweep — owner: board.

### O3: Resink-core executes ADR-2026-05-16-001 step 2 — supervisor swap full execution

source: ceo-brief

Why it matters: Step 2 of the dlopen restoration plan (loop+0 of the loop+N convention; previously named 2026-06-06 absolute and slipped one loop). With AE's template extension landed at 2026-06-06 (`nanofab_node_process` + 4 status code constants exported), resink-core can now actually exercise the `dlopen-plugins` Cargo feature flag against a built `.so`/`.dylib`. Closing step 2 unblocks step 3 (hot-swap correctness test, loop+1 = 2026-06-20) and removes the named ABI-Option-A deviation from the multi-loop plan's "in force" list.

**Key results**
- KR3.1: A built plugin artifact exists in the resink-core working tree under `target/release/lib<crate>.{so,dylib}` (or `target/debug/...` if release fails first; doc the build command in resink-core's docs). The plugin is built from a generated crate that was produced by AE's `codegen-scd2-node` skill against the `dim_user` table fixture (use the existing `synthetic_tenants/closed_loop_v0/` fixture's schema).
- KR3.2: A new integration test in `crates/nanofab-supervisor/tests/dlopen_integration.rs` (or equivalent) that builds the plugin via `cargo build` (or pre-built path), loads the `.so`/`.dylib` via `libloading::Library::open(...)`, looks up the `nanofab_node_process` symbol, invokes it on a known event JSON, asserts return code `NANOFAB_NODE_OK = 0`, and verifies output node state matches the static-linking-path output byte-for-byte. Test passes on first attempt.
- KR3.3: `cargo build --no-default-features --features dlopen-plugins -p nanofab-supervisor` exits 0. `cargo test --features dlopen-plugins -p nanofab-supervisor` exits 0 (existing tests + the new integration test).
- KR3.4: `make mvp-loop` from `synthetic_tenants/closed_loop_v0/` runs GREEN under BOTH feature configurations: (a) default `static-plugins` (existing baseline), (b) `dlopen-plugins`-on against the built plugin artifact. Verdict.json shape unchanged in both runs (`overall_pass: true`, both per-dim `pass: true`, `mismatch_count: 0`, `engine_version: 0.3.0`). Document the dlopen-plugins-on invocation pattern in `repos/resink-ai/resink-core/docs/user-guide.md` (small revision; one paragraph in the "Common commands" section).
- KR3.5: ADR-2026-05-16-001 multi-loop plan's slippage tally updated in this loop's exec summary: step 2 → done at loop+0 (= 2026-06-13); step 3 → loop+1 (= 2026-06-20) per the existing schedule. ADR body grandfathers under absolute-date authoring; the slippage record lives in the brief/exec summary per ADR-2026-05-30-002's loop+N convention.
- KR3.6: Resink-core's `docs/architecture.md` § "Named deviations" updated: ABI Option A entry moves from "still in place" to "step 2 closed; hot-swap test pending loop+1"; the entry surfaces the dlopen-plugins feature flag invocation as the new canonical path under `--features dlopen-plugins`.
- KR3.7: Tenant-isolation invariant holds; no `org-os/` edits this objective.

**Tasks**
- [ ] Generate a plugin crate from AE's template against the `dim_user` schema; build the `.so`/`.dylib` artifact — owner: teams/application/resink-core.
- [ ] Author the `dlopen_integration.rs` test exercising the full FFI roundtrip against the built artifact — owner: teams/application/resink-core.
- [ ] Verify `cargo build` + `cargo test` green under `dlopen-plugins` — owner: teams/application/resink-core.
- [ ] Run `make mvp-loop` under both feature configurations; capture verdicts — owner: teams/application/resink-core.
- [ ] Update `docs/user-guide.md` and `docs/architecture.md` with the dlopen-plugins-on path — owner: teams/application/resink-core.
- [ ] Tenant-isolation dry-run — owner: teams/application/resink-core.

### O4: DE canonicalizes the schema-JSON spec under the `conventions/` tree

source: cross-team-request
links.source: teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md

Why it matters: Sim-farm filed a request at 2026-06-06 for DE to canonicalize the schema-JSON shape (currently inlined in sim-farm's verdict contract) under DE's `conventions/` tree. DE accepted in-loop with deadline 2026-06-13. Canonicalizing here makes the shape DE-owned (consistent with DE owning Kafka contracts, the in-memory event-source contract, and the duckdb convention). Sim-farm's verdict contract gains a back-reference next loop (sim-farm paused this loop; mechanical migration there). This is also the first cross-team request lifecycle to fully complete (open at 06-06; accepted-deferred → fulfilled at 06-13).

**Key results**
- KR4.1: New file at `teams/platform/data-engineering/conventions/dim-schema-json.md` with `type: convention` (canonical, not provisional). Sister to `conventions/duckdb.md` in shape and depth. Sections: (a) one-paragraph context (why DE owns the shape); (b) the JSON shape `{"key_columns": [string], "payload_columns": [string]}` with strict schema; (c) two worked examples (`dim_user`, `dim_account`); (d) named consumers (sim-farm engine 0.3+, resink-core orchestrator); (e) `Verified-against-environment` subsection per ADR-2026-05-16-003 (the second contract/convention to honor the discipline).
- KR4.2: The originating request (`teams/platform/data-engineering/requests/2026-06-06-001-schema-json-shape-ack.md`) flips `status: open → status: fulfilled`; `links.fulfilled_by` field populated with the path of this loop's DE OKR; new `deferred_to_loop: 2026-06-13` field if the team-intake ritual update from O2 KR2.3 has landed mid-loop (otherwise add post-O2-KR2.3, mid-loop bookkeeping).
- KR4.3: Optional housekeeping: flip the `<verified-against-rust-impl: pending>` tag in `teams/platform/data-engineering/contracts/2026-05-10-kafka-ingress.md` §2.1 to `verified-2026-06-13-by-resink-core` (or similar). XS edit; opportunistic; lands if §2.1 gets touched naturally during the gitbook-publishing scope review.
- KR4.4: Sim-farm's `2026-05-16-mvp-loop-verdict.md` contract gets a one-line addition referencing the new DE convention as the canonical schema-JSON shape — but this is sim-farm's responsibility next loop (sim-farm paused this loop). DE's KR4.4 is just the cross-team coordination handshake: file a one-line note at sim-farm's exec summary or status.md (or in DE's exec summary cross-team section) signaling the convention is ready for back-reference.
- KR4.5: Tenant-isolation invariant holds; no `org-os/` edits this objective.

**Tasks**
- [ ] Author `teams/platform/data-engineering/conventions/dim-schema-json.md` per the standing-answer shape — owner: teams/platform/data-engineering.
- [ ] Flip the originating request to `status: fulfilled`; populate `links.fulfilled_by` — owner: teams/platform/data-engineering.
- [ ] (Opportunistic) Flip `<verified-against-rust-impl>` tag in Kafka §2.1 if the file gets touched — owner: teams/platform/data-engineering.
- [ ] Cross-team coordination note for sim-farm's next-loop back-reference — owner: teams/platform/data-engineering.
- [ ] Tenant-isolation dry-run — owner: teams/platform/data-engineering.

### O5: DevOps closes board-action 2026-06-06-002 either way and absorbs the GitBook publishing scope query for chart contracts

source: ceo-brief

Why it matters: Board-action 2026-06-06-002 (install minikube on dev machine) hit its 2026-06-13 forcing function; per the P5 working answer (toolchain installs are per-team Prerequisite docs, not board-action tickets), the ticket closes either `done` (if the human installed minikube in-loop) or `superseded-by-team-prerequisite-pattern`. DevOps surfaces the right close-reason in this loop's exec summary; if `superseded`, the next minikube smoke attempt picks up the toolchain via the chart README's existing Prerequisites section (no new board-action ticket needed). DevOps also responds to board's GitBook publishing scope queries on the chart README and `values.schema.json` (both should be in scope).

**Key results**
- KR5.1: `board/actions/2026-06-06-002-install-minikube-on-dev-machine.md` updated this loop. If minikube is installed (`which minikube` exits 0) at brief-authoring time of the next loop, flip `status: open → status: done` with a one-line "Resolution" subsection naming the install date + version. Otherwise, flip `status: open → status: superseded` with a "Resolution" subsection naming the supersession reason ("per CEO brief 2026-06-13's working answer to retro P5: toolchain installs route through team Prerequisite docs, not board-action tickets") and pointing at the chart README's existing `brew install minikube` prerequisite. **Default action this loop:** if the brief's probe (`which minikube`) returns "not found" — which it did at brief authoring, see § Context probe results — DevOps closes the ticket as `superseded`.
- KR5.2: Chart README's `Prerequisites` section gets a small refresh: add a one-line note at the top stating "Toolchain installs are documented here, not in board-action tickets, per CEO brief 2026-06-13" with a forward-link to ADR-2026-06-06-001 (provisional-and-migrate playbook — pattern reuse).
- KR5.3: Minikube smoke (KR carry from 2026-06-06): re-attempt only if `which minikube` exits 0 in DevOps's build phase. If not, the smoke remains carry per the chart README's Prerequisites; no new board-action ticket; pickup happens whenever the toolchain lands.
- KR5.4: Board's GitBook publishing scope query response: chart README is IN scope (lives at `repos/resink-ai/resink-core/deploy/charts/nanofab-supervisor/README.md` — but it's product-repo content, not a `teams/` artifact; DevOps recommends including it under a `/repos/resink-core/charts/` namespace in the gitbook IF the publishing script sees product-repo `docs/` and `deploy/charts/*/README.md` as in-scope; otherwise it's covered by the resink-core docs tree's existing module-catalog entry). `values.schema.json` is JSON not markdown; out of gitbook scope by default (it's referenced from the chart README which IS in scope). DevOps confirms or adjusts board's scope decisions as part of O1's planning.
- KR5.5: Tenant-isolation invariant holds; no `org-os/` edits this objective.

**Tasks**
- [ ] Update board-action 2026-06-06-002 with `done` or `superseded` per the probe — owner: teams/platform/devops (with board sign-off via this brief's standing answer).
- [ ] Refresh chart README Prerequisites section with the policy note + ADR-2026-06-06-001 forward-link — owner: teams/platform/devops.
- [ ] Re-attempt minikube smoke if toolchain is present; otherwise carry per chart Prerequisites — owner: teams/platform/devops.
- [ ] Respond to board's GitBook publishing scope queries on chart README + values.schema.json — owner: teams/platform/devops.
- [ ] Tenant-isolation dry-run — owner: teams/platform/devops.

## Risks

- **The publishing pipeline (O1) could fail at GitHub Pages build time** even after a successful local Jekyll render (different Jekyll version, plugin allowlist differences, theme compatibility). Mitigation: KR1.4 verifies the rendered site at `blog.resink.ai` (or fallback `resink-ai.github.io`); if GitHub Pages build fails, the script's first publication commit can be reverted (`git revert` in the submodule) and the build error inspected. Falling back to the default GitHub-Pages-supported theme set (`jekyll-theme-minimal`, `jekyll-theme-cayman`, etc.) is a one-line `_config.yml` change. Don't introduce custom theme dependencies or non-allowlisted plugins this loop.
- **Site-organization decisions (O1 KR1.3) could prove wrong on second iteration** — the per-loop pages might be too long, the per-team pages too sparse, the decisions index too flat. Mitigation: this loop is the FIRST publication; iteration is expected. Document the publishing-scope decisions explicitly (KR1.6); next loop's retro will likely propose layout improvements based on actual reading experience. The first publication is "correctness" not "polish."
- **The 4-ADR ratification batch (O2) is the heaviest org-os write of any single loop since 2026-05-30 (3 ADRs + extensions).** Mitigation: AE is paused this loop, so there's no parallel `org-os/` writer creating merge friction. Board can serialize the four within its own work stream without coordination overhead. Per-ADR dry-run keeps the tenant-isolation invariant observable at every step.
- **Resink-core dlopen swap (O3) could surface FFI design issues at integration time** that AE's template smoke missed — e.g., the `ctx_ptr: *mut c_void` semantics, the panic boundary's interaction with the supervisor's existing panic handlers, library-load symbol versioning. Per the 2026-06-06 retro's mild "what didn't" #4: the integration smoke is the load-bearing test, and only resink-core's loop+0 (this loop) exercises it. Mitigation: if surprises surface, file an in-loop ADR addendum to ADR-2026-05-16-001 naming the actual signature/semantics rather than re-litigating the original; resink-core proceeds against the actual built artifact's exported symbols. The "verify-against-environment" discipline (ADR-2026-05-16-003) applies — confirm the actual exported symbols match the template's declared exports before declaring KR3.2.
- **DE's schema-JSON conventions doc (O4) could prove insufficient for resink-core's needs** if resink-core wants additional fields (e.g., `is_scd2` flag, `compaction_strategy`, etc.) once the convention is canonicalized. Mitigation: the JSON shape is intentionally minimal this loop (key_columns + payload_columns only); resink-core can file a follow-up cross-team request for additional fields when it surfaces a need. The minimal shape is sufficient for the engine 0.3 contract.
- **Workspace promotion 5th-loop carry surfaces escalation pressure on retro.** The board-action ticket's forcing function is naming the count of downstream blockers; this loop names them again. If 2026-06-20 still shows the carry, the retro should propose a different escalation mechanism (e.g., the brief generates the `gh repo create` command and asks the human to run it via the `! prefix` mechanism in the conversation, rather than relying on out-of-loop human action). Mitigation: this loop continues the existing forcing function; escalation is for the 2026-06-20 retro if needed.
- **Submodule sprawl:** Adding `resink-ai/resink-ai.github.io` makes 3 active submodules (marketplace, home-cluster, gitbook). Submodule pointer bumps are now coordination work in every loop that touches publishing or marketplace plugin work. Mitigation: documented in `scripts/PUBLISHING-SCOPE.md` + the existing submodule-bump pattern (commit submodule first, push, open PR; bump pointer in parent) is well-established by now (used twice across the prior 4 loops).

## Out of scope this loop

- **Workspace promotion (`resink-ai/resink-core` GitHub remote creation)** — 5th-consecutive-loop defer. **Per board-action 2026-06-06-001's forcing function: this section names the count of downstream blockers explicitly, as required.** Downstream blockers known at 2026-06-13:
  1. Workspace promotion itself (resink-core KR2.1, original carry).
  2. ADR-2026-05-16-001 step 2 commit graph: lands this loop in-tree only; future bisect/blame across the runtime separation gains friction.
  3. ADR-2026-05-16-001 step 3 (hot-swap correctness test, loop+1 = 2026-06-20): same friction multiplied; the test fixture's commit history will be in-tree until promotion.
  4. The HTML capabilities report at `board/reports/2026-05-30-resink-core-capabilities.html` cites in-tree paths for evidence; once promoted, paths become submodule-relative and need a one-paragraph update.
  5. The GitBook publishing pipeline's resink-core docs tree integration (O1 KR1.3 + KR1.4): lives in the in-tree path `repos/resink-ai/resink-core/docs/` this loop; once promoted, the publishing script needs a one-line submodule-walk update to follow the resink-core submodule pointer.
  Total: **5 named downstream blockers**, increasing by 1 each loop (the gitbook publishing path is new this loop). The board-action ticket stays `status: open`. The 2026-06-20 retro should propose a different escalation mechanism if the carry persists (e.g., the brief generates the `gh repo create` command and asks the human to run it via `! prefix`).
- **DevOps minikube smoke full execution** — depends on toolchain install per the standing-answer-to-P5 working answer. Carry through chart Prerequisites; pickup on next loop where minikube is present.
- **Hot-swap correctness test** — ADR-2026-05-16-001 step 3, target loop+1 (= 2026-06-20). Joint AE + resink-core deliverable. Out of scope this loop.
- **Sim-farm Modes B + C** — out of scope per long-standing posture; trigger is post-multi-dim, post-dlopen.
- **Sim-farm own product repo decision** — trigger condition not met (first non-Python sim-farm component).
- **Three-layer verdict** (Sim Farm spec §4.7) — out of scope until training-pipeline gate stage 3 adopts coverage-spec-driven verdicts.
- **Future codegen patterns** (`scd1_first_event`, `window_stats_with_decrement`, etc.) — out of scope per the consolidation framing; AE paused this loop.
- **GitBook publishing CI integration** — first publication is manual this loop; CI integration (e.g., a GitHub Actions workflow that auto-runs the publishing script on parent-repo merge to master) is sized at next loop or whenever the manual cadence proves friction-bearing.
- **GitBook publishing of `org-os/`** — out of scope this loop. If the engine doc set is published later, lives under a separate namespace (e.g., `/engine/`) at a separate URL or subpath. The 2026-06-13 publishing scope is tenant-only.
- **GitBook publishing of `teams/.../status.md` files** — out of scope (high-frequency internal state).
- **GitBook publishing of `teams/.../{proposals,requests}/` directories** — out of scope this loop; the requests appear linked from OKRs but not as standalone pages. Defer the standalone-page decision to a future retro if reading experience surfaces a need.
- **Resink-core stretch capacity decision** (sub-project #5 Product UX team standup vs internal split) — still deferred; revisit at next MVP-class deliverable surfacing.
- **DE next-slice event-source contracts** — deferred-not-dropped; DE authors only when a consumer surfaces specific pressure.
- **SRE BLOCKED observable verification** — multi-loop wait; this loop closes ADR-2026-05-16-001 step 2 but not step 3.
- **SRE Modes B/C runbook expansion** — wait on sim-farm Modes B/C.
- **Frontmatter-lint CI script** — fifth consecutive defer per loop 2026-05-08 ADR-001 framing.
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing housekeeping.

## Ratifications this loop

- **ADR-2026-05-30-001** (verify predicted syntax in OKRs at planning time) flips `draft → active`. Mandated edits to `org-os/rituals/team-planning.md` + `org-os/rituals/build.md`.
- **ADR-2026-06-06-001** (provisional-and-migrate coordination playbook) flips `draft → active`. Mandated edit: new `org-os/playbooks/provisional-and-migrate.md` with `type: playbook`.
- **ADR-2026-06-06-002** (paused-team request acceptance canonicalization) flips `draft → active`. Mandated edits to `org-os/rituals/team-intake.md` + `org-os/templates/request.md` + `org-os/rituals/ceo-brief.md` step 1; new optional `deferred_to_loop` request frontmatter field.
- **ADR-2026-06-13-001** (admit `action` to conventions enum) drafted AND flipped to `active` in the same loop (sister to ADR-2026-05-23-001's enum-extension precedent). Mandated edits to `org-os/conventions.md`. Two existing `board/actions/2026-06-06-001*.md` and `board/actions/2026-06-06-002*.md` migrate from provisional `type: action` to canonical (in-body provisional notes removed).
- **No contract migrations this loop** (the engine 0.3 verdict contract migration completed at 2026-06-06; the schema-JSON spec canonicalization at O4 is a NEW conventions doc, not a migration).
- **Multi-loop plan slippage absorption (not a new ratification):** ADR-2026-05-16-001 step 2 = closed loop+0 (this loop); step 3 = loop+1 (= 2026-06-20). Recorded here per ADR-2026-05-30-002's loop+N convention; the ADR body grandfathers under absolute-date authoring.
