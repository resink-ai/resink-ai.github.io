---
layout: default
title: Retro — 2026-05-13-1022
date: 2026-05-13
status: active
type: retro
loop: 2026-05-13-1022
owner: board
---

<!-- original-frontmatter:
  type: retro
  owner: board
  date: 2026-05-13
  status: active
  loop: 2026-05-13-1022
  links: parent: board/exec-summaries/2026-05-13-1022.md
-->
{% raw %}

# Resink.ai CEO Retro — 2026-05-13-1022

## What worked

- **ADR-2026-05-16-001 multi-loop arc closes cleanly.** Five loops, three steps (with one mid-arc split), one named deviation. Step 1 (AE template extension, 2026-06-06) + Step 2 (resink-core supervisor swap, 2026-05-11-2153) + Step 3 (split: AE-side 2026-05-13-0859 + resink-core-side this loop). The runtime spec §4.3 hot-swap property is now verified at the production code path. **Reproducibility:** the named-deviation + multi-loop-restoration pattern works at full closure. Pre-conditions: (a) the deviation is time-boxed at filing, (b) each step has a named owner + target loop, (c) when a step slips a third time the planning re-shapes (split vs single-loop dedication vs indefinite slip — the prior retro's option-(a)/(b)/(c) framework). The arc absorbed one third-consecutive slip via the split and still closed within the original time budget.

- **Bundling 3 small carryover items into one loop worked exactly as predicted.** Prior retro recommended "when 2-3 carryover items are all in the same submodule's product surface and none of them is large alone, bundling reduces per-loop coordination cost." This loop applied the pattern: hot-swap test (M) + CI bootstrap (S) + Makefile target (S) ≈ M-L total, fit comfortably in one loop with no scope creep. The per-loop cost (one PR set, one cargo cache, one workstation context, one CI bootstrap run) was paid once instead of three times. **Reproducibility:** the bundling heuristic is now well-tested across two loops (this one + 2026-05-13-0056's org-os ratification bundle). Both used the same shape — multiple small same-team items combined into a focused loop. The pattern is canonical at this point.

- **Honest scope re-shaping at mid-loop discovery.** O1's brief framing said "close ADR-2026-05-16-001 step 3 fully this loop" — but mid-loop investigation surfaced that DEVIATION.md's "full read-back" contract requires a supervisor-side NodeCtx bridge (substantially separate work). The hot-swap test was re-shaped: load → swap → load cycle through `PluginNode` (achievable); source-level deviation pin (achievable); cross-swap read-back of `valid_to = event_ts - 1` (gated on NodeCtx bridge — deferred). The ADR closing-section is explicit about the distinction (step 3 closed; step 3.5 follow-up named). **Reproducibility:** when a contract doc names its preconditions explicitly (DEVIATION.md did this — "this assumes the supervisor reads from `ctx_ptr` instead of `DefaultProcessCtx`"), the implementing loop can ship what's achievable and flag what's deferred without re-litigating the contract. Worth a small pattern entry; see P3 for proposal.

- **`make bootstrap` shape generalizes.** Idempotent recovery target that automates a mechanical recipe. Three lines of Makefile + a doc-comment block. Verified by simulating the actual incident (delete venv → run bootstrap → verdict pass). **Reproducibility:** any post-disruption recovery scenario where the steps are known but unwritten is a candidate for this pattern. The pattern is "automation over documentation when the recovery is mechanical, idempotent, and non-destructive." Three constraints — all three matter. Documentation is the right shape when recovery has decision points (operator chooses between options); automation is the right shape when it's deterministic.

- **CI bootstrap surfaced one latent contract gap immediately.** `helm lint --strict` required `-f values/home-cluster-mvp.yaml` to satisfy `values.schema.json`'s `tenant: minLength: 1` constraint. The chart's default `values.yaml` ships `tenant: ""` as a placeholder; the schema enforces non-empty. Without the workflow, the constraint was never tested in any automated context. **Reproducibility:** CI bootstrap is a forcing function that exposes latent contract gaps — gaps that would have surfaced only when an operator ran a stricter validation locally. Pattern: when bootstrapping CI on a previously-uninstrumented surface, expect 1-2 latent issues to surface; budget for them rather than treat them as scope-out.

## What didn't

- **CI scope-back: cargo job + branch protection both deferred mid-loop.** Two mid-loop discoveries forced honest scope reduction on O2:
  - **Cargo CI requires cross-repo marketplace access.** Supervisor's `Cargo.toml` path-deps point at `synthetic_tenants/closed_loop_v0/workspace/nodes/` crates which are materialised by `make orchestrate` — and the orchestrator reads templates from `../resink-marketplace/` (private sibling repo). CI checkout of resink-core alone can't `git clone` the marketplace without a deploy-key or PAT secret on the resink-core remote.
  - **Branch protection requires GitHub Pro for private repos.** Both `/branches/{branch}/protection` and `/rulesets` return `403 — Upgrade to GitHub Pro or make this repository public` on the free plan. Free plan only allows protection on public repos.
  
  Both deferrals are honest scope rather than slippage — neither was knowable from the brief without doing the work. The helm job ships green in CI; the operator manually verifies green helm CI in the PR UI before merging until protection lands. **Moderate:** the CI gate is partial. Carries forward as P5/P6 below.

- **`helm install --dry-run` requires a reachable k8s API server.** Default `--dry-run` is server-side; GitHub Actions runners have no cluster. Dropped from the workflow. Operators run it locally against the home cluster before deploying. **Trivial:** discovery + fix in one mid-loop iteration; documented in workflow comments + user-guide.md. Not a slip, just a contract surprise from helm v3.16's default mode.

- **`uv sync` package reinstallation on idempotent re-run.** `make bootstrap` re-run on a "clean" tree still produced install output (`+ duckdb==1.5.2`, etc.) — because `uv sync` reconciles the venv with the lockfile, not "no-op if directory exists." End state is correct; operator might be surprised. Documented in the Makefile comment block. **Trivial:** behavior, not bug; documentation cost was zero.

- **`cargo test --workspace --release` is slow on first run.** ~30 seconds because the hot-swap test's `ensure_plugin_built` helper compiles v2 from scratch on the cold path. Subsequent runs are 0.4s (cargo cache). Same shape as the existing `dlopen_integration.rs` test (which has the same `ensure_plugin_built` pattern). **Trivial:** cargo caches; not a regression.

- **The hot-swap test under `static-plugins` (default) is a `#[cfg(...)]`-gated no-op file.** Under the default feature set, the test file compiles to zero tests — the dlopen-plugins variant is what closes step 3. A reader looking at the test list under default features would see "no hot_swap tests" and might draw the wrong conclusion. **Mild:** documentation issue. The test file's header comment is explicit about this; could surface in `docs/user-guide.md` § "Continuous integration" too for redundancy.

## Evolution proposals

### P1: Multi-loop-blocker-arc report-type ADR (class: **org-os**)

- **Problem it solves:** Carryover from 2026-05-12-1254 retro § P3, deferred until an arc closed. **An arc just closed** (ADR-2026-05-16-001's five-loop dlopen restoration). The pattern of named deviation + multi-loop plan + time-boxed steps + post-arc retrospective is now concrete enough to standardize.
- **Proposed change:** New ADR drafted as `board/decisions/<date>-NNN-multi-loop-blocker-arc-report-type.md`. Body proposes a new artifact type (`arc-report`) emitted when a multi-loop ADR closes. Contents: timeline (loop dates per step), slip record (which steps slipped, why), what-shipped per step, what-deferred + carryover, what-the-arc-validated-or-broke (i.e., what we learned about the deviation that motivated the original ADR).
- **Recommendation:** **Draft this loop OR defer to next non-resink-core loop.** The ADR-2026-05-16-001 arc is the worked example; authoring while the arc is fresh is cheaper than re-deriving later. Sized S (one ADR draft + one worked-example arc-report against ADR-2026-05-16-001). **Recommend: draft + ratify next loop alongside the link-existence smoke (prior retro P2) since both are non-resink-core; bundle naturally.**
- **Review path:** ADR-class. Draft next loop; ratify in the same loop (the worked example exists in tree already so the codification is observation-not-decision).
- **Owner:** board.
- **Timing:** **Next non-resink-core loop.**

### P2: Supervisor-side NodeCtx bridge (step 3.5 follow-up to ADR-2026-05-16-001) (class: **resink-core**)

- **Problem it solves:** Makes the DEVIATION.md § "Consumer-side acceptance contract" full cross-swap read-back assertion achievable. Currently both plugin templates ignore `ctx_ptr` and use their own in-process `DefaultProcessCtx`; row state dies with each library handle, so the swap is mechanically correct but the deviation isn't observable across it.
- **Proposed change:** Two parts. (a) Template-side: AE extends `templates/scd2_maintainer{,_v2}/lib.rs.tmpl` to use `ctx_ptr` when non-null (fall back to `DefaultProcessCtx` only when null, preserving back-compat). (b) Supervisor-side: resink-core adds a `SupervisorCtx` Rust type implementing the trait surface, marshals it across the FFI boundary as `*mut c_void`, and verifies the cross-swap read-back in an extended `hot_swap_correctness.rs` test.
- **Recommendation:** **Demand-driven, not scheduled.** Triggers: (i) a second hot-swap deviation needs runtime observation; (ii) multi-tenant supervisor state-layer integration arrives. Until then, the source-level pin (DEVIATION.md + per-plugin smoke + the hot-swap test's `v2_source_contains_saturating_sub_v1_does_not` assertion) is sufficient.
- **Review path:** ADR-class if it lands (cross-cuts AE + resink-core templates + supervisor; not a tenant-internal change). Draft when scheduled.
- **Owner:** Joint AE + resink-core.
- **Timing:** **Indefinite; demand-driven.**

### P3: "Contract preconditions" body-shape convention for cross-team contract docs (class: **org-os**)

- **Problem it solves:** "What worked" #3 named the pattern. DEVIATION.md's explicit "this assumes the supervisor reads from `ctx_ptr` instead of `DefaultProcessCtx`" precondition let this loop's implementing test re-shape cleanly (close what's achievable + name what's deferred). Pattern is worth codifying so future contract docs reliably include preconditions, not just acceptance.
- **Proposed change:** Extend `org-os/conventions.md § "Body-shape rules"` with a "Preconditions" sub-section for cross-team contract docs (`type: contract` or similar). When a contract names what the consumer must do to satisfy the contract, it should also name what's-assumed-of-the-consumer-environment for the contract to be testable. The DEVIATION.md worked example serves as the canonical shape.
- **Recommendation:** **Deferred candidate — codification-without-motivation.** The pattern surfaced cleanly this loop, but it's only one worked example. Wait for a second instance before codifying. If a future cross-team contract is authored without preconditions and that absence causes friction, the friction motivates the codification.
- **Review path:** Eventually ADR-class (conventions.md edit); not this loop, not next loop.
- **Owner:** board.
- **Timing:** **Deferred; revisit when motivated.**

### P5: Cargo CI gate — unblock cross-repo marketplace access (class: **tenant**)

- **Problem it solves:** "What didn't" #1 first sub-bullet. The cargo CI job is deferred until CI can clone the resink-marketplace sibling. Both unblocking paths preserve **private** for both repos (repo visibility is a user-only decision and is out of scope for retro proposals; see memory note `feedback-no-public-repo.md`):
- **Options:**
  - **(a) Configure a deploy-key (read-only) on resink-marketplace; store the private key as a `MARKETPLACE_DEPLOY_KEY` secret on resink-core; use it in the CI workflow's clone step.** Per-repo scoped; ~5 minutes of GitHub UI setup.
  - **(b) Configure a fine-scoped PAT (Personal Access Token) with cross-repo read access; store as a secret on resink-core; use in CI.** Similar to (a) but org-wide rather than per-repo. Slightly broader blast-radius than a deploy key.
  - **(c) Bypass cross-repo entirely.** Commit a minimal test fixture of the codegen-output crates into resink-core itself (e.g., under `crates/test-fixtures/dim_user_scd2/` + `dim_account_scd2/`), used as the CI-only path-dep targets; production runtime path continues to use the orchestrator-generated `workspace/nodes/` outputs. Largest change but removes the cross-repo dependency from CI entirely.
- **Recommendation:** **(a) deploy key.** Smallest blast-radius (read-only, scoped to the one repo); the secret config is the user's manual GitHub UI work, well-bounded.
- **Review path:** Tenant decision (no ADR — secret configuration is an operational choice).
- **Owner:** board (decides + sets up the secret); resink-core (uncomments the deferred cargo job in `.github/workflows/ci.yml` once the secret is in place).
- **Timing:** **Next loop OR opportunistic** when the user is in GitHub admin UI anyway.

### P6: Branch protection on resink-core master — paid-feature precondition (class: **tenant**)

- **Problem it solves:** "What didn't" #1 second sub-bullet. Free GitHub plan blocks branch protection and rulesets on private repos. The repo stays private (per memory note `feedback-no-public-repo.md`), so the only path is the paid-plan upgrade:
- **Options:**
  - **(a) Upgrade the org to GitHub Pro / Team / Enterprise.** Paid; standard for orgs with private CI requirements.
  - **(b) Defer indefinitely.** Operator manually verifies green helm CI in the PR UI before merging until/unless the org plan changes. Acceptable for single-contributor reality.
- **Recommendation:** **(b) defer indefinitely** while the contributor model is single-operator; revisit when a second contributor lands and unverified-master becomes a real risk surface.
- **Review path:** Tenant decision.
- **Owner:** board.
- **Timing:** **Indefinite; reevaluate when contributor count > 1.**

### P4: Make `post-publish-CI-verification` standing practice formal (class: **deferred**)

- **Problem it solves:** Last loop's retro § P1 codified this as memory'd standing practice (not ADR, not playbook). This loop didn't trigger it (resink-core changes didn't touch gitbook). When a future loop's publish step lands, the practice runs; if it surfaces as load-bearing across multiple loops, then it earns a playbook entry.
- **Proposed change:** Keep as-is (memory'd). Revisit if a sixth publish-CI failure surfaces undetected (i.e., if the memory'd discipline fails).
- **Recommendation:** **No action this loop.**
- **Review path:** None unless invoked.
- **Owner:** memory'd; agent on the keyboard during loop close-out.
- **Timing:** **Indefinite.**

## Decisions to record

**New ADR placeholders this loop: None.**

- P1 (multi-loop-blocker-arc report-type) drafts next loop, not this one.
- P2 (NodeCtx bridge) is demand-driven.
- P3 (contract preconditions convention) is deferred.
- P5 (cargo CI cross-repo access) is tenant-class (repo visibility / secrets / Pro decision); no ADR.
- P6 (branch protection on private repo) is tenant-class; bundled with P5's decision.
- P4 (post-publish-CI-verification) is standing practice; no ADR.

**Net new ADR drafts this retro: 0.** The trend (zero new ADRs since 2026-05-13-0056's ratification bundle) continues. Org-os process surface is stable; product work is closing planned arcs without spawning new authoring debt.

## Carryover ADRs / open work still on the books

- **ADR-2026-05-16-001 arc:** **CLOSED.** Closing-section added this loop; the multi-loop plan ends. The retro's P1 (arc-report-type ADR draft) is the natural retrospective surface.
- **Supervisor-side NodeCtx bridge (P2 this retro):** demand-driven; no scheduling.
- **Link-existence smoke for `publish-to-gitbook.py`** (2026-05-13-0859 retro § P2): next non-resink-core loop. **Bundles with P1 this loop** if next loop authors the arc-report-type ADR + the link-smoke at the same time (both are non-resink-core, both are S-sized, both touch board/scripts surface).
- **Branch protection enable + first-post-merge-CI-verification on resink-core remote:** in this loop's publish phase.
- **Multi-table hot-swap** (dim_account, etc.): future-loop slice if a second SCD2 deviation needs hot-swap verification.
- **Observability v2 slices** (logs, metrics, events-tail): demand-driven; nothing scheduled.
- **`teams/application/resink-core/resink-core/` symlink cleanup:** non-load-bearing carry from 2026-05-23.
- **`<verified-against-rust-impl: pending>` tag in DE Kafka contract §2.1:** owner = DE; carries.

## Tenant-isolation dry-run

Held trivially this loop. Zero `org-os/` edits. Final sweep: `grep -nrE "(resink|nanofab|home-cluster|acme\.ai)" org-os/` returns one line at `org-os/conventions.md:121` — the canonical `acme.ai` placeholder. Pass.

All this-loop content lives under `repos/resink-ai/resink-core/` (v2 crate + hot-swap test + CI workflow + Makefile bootstrap + docs CI section), `teams/application/resink-core/` (status/OKR/exec), `board/decisions/2026-05-16-001-abi-option-a-mvp-deviation.md` (closing-section amendment), and `board/{okrs,exec-summaries,retros}/`. None of these are under `org-os/`.

## Arc retrospective: ADR-2026-05-16-001

(This subsection is the candidate worked example for P1's `arc-report` artifact type.)

**Timeline.** Filed 2026-05-16 as a named deviation from runtime spec §4.3 (the hot-swap property the supervisor's static-linking-only MVP path couldn't satisfy). Three-step time-boxed restoration plan: step 1 = AE template extension; step 2 = resink-core supervisor swap; step 3 = joint hot-swap correctness test.

**Per-step status:**

| Step | Original target | Actual close | Slip | Note |
|---|---|---|---|---|
| 1 (AE template extension) | 2026-05-11-1302 | 2026-06-06 | +1 loop | Bandwidth went to ritual edits (Bundle B). |
| 2 (supervisor swap) | 2026-05-11-1631 | 2026-05-11-2153 | +1 loop | Partial scaffold landed at 2026-06-06; full execution at 2026-05-11-2153. |
| 3 (hot-swap test) | "loop after step 2" (≈2026-06-20) | 2026-05-13-1022 | +3 loops | Slipped three times under single-focus loops; resolved via prior retro § P1 option-(b) split. |

**What the arc validated.** The named-deviation pattern absorbed real schedule pressure without losing the time-box. Step 3's third-consecutive slip didn't break the arc — it forced a planning re-shape (split into AE-only + resink-core-only halves) that landed both halves in adjacent loops. The arc's total budget held; the closed test (load + swap cycle through production `PluginNode`) validates the load-bearing operational invariant (hot-swap property).

**What the arc didn't validate.** Cross-swap read-back of the deviation (DEVIATION.md's full contract) requires the supervisor-side NodeCtx bridge — explicitly named as step 3.5 follow-up, not in scope for the arc's closure. The arc closed at "supervisor can mechanically swap libraries"; cross-swap observability is the next-arc starting point.

**Patterns extracted.** (1) The split-vs-dedicate-vs-defer triage works for third-consecutive slips (option (b) split this time, option (a) dedicate in prior loops, option (c) indefinite for indefinite carryovers). (2) Contract docs with named preconditions (DEVIATION.md) let implementing loops close what's achievable + flag what's deferred without re-litigating. (3) Bundling small co-located items reduces per-loop cost (this loop's 3-item bundle). (4) CI bootstrap exposes latent contract gaps (the helm-lint-strict + values.schema.json discovery).
{% endraw %}
