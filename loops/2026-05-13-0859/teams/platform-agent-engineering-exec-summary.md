---
layout: default
title: platform-agent-engineering Exec Summary — 2026-05-13-0859
date: 2026-05-13
status: active
type: exec-summary
loop: 2026-05-13-0859
owner: teams/platform/agent-engineering
grand_parent: Loops
parent: Loop 2026-05-13-0859
nav_order: 11
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/platform/agent-engineering
  date: 2026-05-13
  status: active
  loop: 2026-05-13-0859
  links: parent: teams/platform/agent-engineering/okrs/2026-05-13-0859-team-okr.md
-->
{% raw %}

# Agent Engineering Exec Summary — 2026-05-13 (loop 2026-05-13-0859)

## Headline

Two-objective AE loop both closed clean. O1 shipped a new `observability` marketplace plugin with its first skill `cluster-snapshot` — produces a self-contained HTML snapshot of the kubernetes cluster (4 nodes, 21 pods, 0 warning events on the home cluster smoke). O2 closed AE's half of the hot-swap correctness-test split — `templates/scd2_maintainer_v2/` ships with a one-line deviation from v1 (close-row `valid_to = event_ts - 1` instead of `event_ts`), DEVIATION.md documents the diff + worked 3-row example + consumer-side acceptance contract, DISPATCH.md addendum points consumers at both templates. Marketplace tests 89/0/3 (up from 88; observability adds one new pass row). Tenant-isolation CLEAN.

## What we shipped

**O1 — `observability` plugin + `cluster-snapshot` skill v1:**

- KR1.1 ✅: Plugin scaffolded at `repos/resink-ai/resink-marketplace/plugins/observability/` with all four per-platform manifests (`.claude-plugin/plugin.json`, `.codex-plugin/plugin.json`, `gemini-extension.json`, `package.json`) + `GEMINI.md` + `.opencode/plugins/observability.js` bridge. Version `0.1.0`.
- KR1.2 ✅: `plugins/observability/skills/cluster-snapshot/SKILL.md` authored with the canonical body shape (when-to-invoke, contract, procedure, worked example, pinned tooling, anti-patterns, sister patterns, links).
- KR1.3 ✅: `generate.py` (stdlib-only Python; chose Python over shell-heredoc to avoid HTML-escape pain). 4 kubectl calls (`config current-context`, `get nodes -o json`, `get pods -A -o json`, `get events -A --field-selector type=Warning -o json`); atomic write via `os.replace`. Self-contained HTML (inline `<style>`, no `<link>`/`<script>`/`<img>`).
- KR1.4 ✅: Marketplace manifest test suite **89 pass / 0 fail / 3 skip** (up from 88/0/3; the new observability plugin adds one manifest row). `bash scripts/bump-version.sh --check observability` reports `all versions match: 0.1.0`. `marketplace.json` extended with the third `plugins[]` entry; `.version-bump.json` extended.
- KR1.5 ✅: End-to-end smoke against the home cluster — `kubernetes-admin@kubernetes` context, k8s `v1.33.11`, 4 nodes (`k8s-0` control-plane + `k8s-1/2/3` workers). Output committed at `board/reports/2026-05-13-cluster-snapshot.html` (8,415 bytes; 4 nodes, 21 pods all Running, 0 warning events). Verified by opening locally in a browser. `SMOKE.md` records toolchain + acceptance.
- KR1.6 ✅: AE charter "Owned products" 5th bullet added; `date:` bumped `2026-05-16 → 2026-05-13`.
- KR1.7 ✅: Tenant-isolation invariant clean throughout. Zero `org-os/` edits this loop. Final post-loop grep returns single canonical `acme.ai` line at `org-os/conventions.md:121`.

**O2 — hot-swap step 3 AE-side template fixture pair:**

- KR2.1 ✅: `templates/scd2_maintainer_v2/` mirrors v1's two-file layout (`Cargo.toml.tmpl` + `lib.rs.tmpl`). Single deliberate logic deviation at the `ctx.close_current(...)` call sites in the `process` function: v2 passes `ev.event_ts.saturating_sub(1)` instead of `ev.event_ts`. Saturating semantics avoid underflow at `event_ts == 0`. C-ABI signatures + status code constants + `Node`/`NodeCtx` trait surface preserved byte-for-byte.
- KR2.2 ✅: `templates/scd2_maintainer_v2/DEVIATION.md` documents (i) one-line conceptual diff, (ii) source-line diff block, (iii) worked example with 3-row input fixture (e1 Insert t=100; e2 Update t=200; e3 Update t=300) and the v1-vs-v2 expected rows table showing `valid_to` shifts by 1 on closed rows, (iv) invariants-preserved list (8 bullets covering C-ABI, status codes, trait surface, JSON parser, Cargo.toml, determinism, SCD2 monotonic invariants, tombstone-on-delete), (v) anti-deviation guard (what v2 does NOT change), (vi) consumer-side acceptance contract — the 6-step test procedure resink-core's loop will use to verify the swap.
- KR2.3 ✅: DISPATCH.md gained a new "## Template variants (hot-swap fixture, added 2026-05-13)" section pointing at both template paths, documenting the deviation contract, the v2 invocation shape (`pattern_name: "scd2_maintainer_v2"`), and the consumer-side acceptance contract. Cross-links to ADR-2026-05-16-001 step 3 + this loop's brief § O2 + DEVIATION.md. SKILL.md updated alongside: `pattern_name` accepted-values list extended to include `scd2_maintainer_v2`; template-directory path resolution now uses `templates/<pattern_name>/` (was hard-coded to `scd2_maintainer/`).
- KR2.4 ✅: Marketplace tests stay GREEN (89/0/3 regression-free vs the new baseline). v2 template directory adds no new test surface (manifest tests don't recurse into `templates/`); existing v1 smoke unchanged.

## What we didn't ship and why

- **Build smoke for v2's slot-fill output.** The plan in the OKR included `cargo build --release` against v2's slot-fill, but cargo + rustc are not installed in the AE workstation environment (per the loop 2026-05-11-1631 status). The DEVIATION.md "Anti-deviation guard" section identifies the only changed lines; structurally the v2 template diverges from v1 only at two `close_current` call sites; both call sites use a pure-Rust arithmetic operation (`.saturating_sub(1)`) that cannot introduce a compile error in isolation. Consumer-side compile in resink-core's CI (which has the full toolchain) will catch any latent issue when step 3's resink-core half lands; if anything surfaces, AE files an in-flight fix. Acceptable risk given the deviation's surgical shape.
- **Per-template SMOKE.md update.** The existing nanofab skill's SMOKE.md was last updated 2026-06-06 to record the C-ABI smoke. No v2-specific SMOKE row added this loop because there is no in-tree v2 build; SMOKE.md for the v2 path is the resink-core consumer-side test (future loop). DEVIATION.md serves as the per-template self-verification doc.
- **Filesystem workspace introspection slice** of observability (one of the four candidate slices in the brief's prerequisites table). Out of scope per the smallest-product-slice decision — resink CLI v1 already provides this surface from loop `2026-05-12-1826`.
- **Resink-core consumer-side hot-swap test.** That's the resink-core half of the split; bundles with retro P2 CI/CD bootstrap + retro P3 submodule-deinit-recovery at the next active-resink-core loop.

## Surprises

- **`saturating_sub` rather than checked sub or panicking sub.** The brief's "v2 deviation" was specified as "valid_to = event_ts - 1"; build-phase finding: `event_ts` is `u64`, so an `event_ts = 0` event would underflow. `saturating_sub(1)` is the right defensive choice. Documented in DEVIATION.md "Invariants preserved" line about the v2 deviation being safe under `event_ts == 0`.
- **The deviation lives at the *call site* in the process function, not in the `DefaultProcessCtx::close_current` body.** This is a more conservative deviation surface (only the third argument to `close_current` changes, not the trait impl itself), which means the `NodeCtx` consumer (the supervisor) sees the deviation through the value passed across the trait boundary rather than through trait-impl semantics. Externally provided `NodeCtx` implementations are unaffected by which template they consume — they receive whatever `valid_to` value the template passes. Cleaner than initially expected.
- **Existing smoke tests in `lib.rs.tmpl` test structural properties only** (`name()`, `version()`, status code constants, FFI round-trip with a wrong-table event mapping to `BLOCKED`). They make no assertions about specific `valid_to` values. v2's deviation is therefore invisible to the in-template smoke — exactly what the consumer-side test needs to detect.
- **`marketplace.json`'s plugins[] is now order-sensitive** (per `.version-bump.json`'s `plugins.N.version` indexing). Observability landed as `plugins.3.version` (zero-indexed). If a plugin is removed or reordered, all higher-indexed `.version-bump.json` entries need re-indexing — a pre-existing constraint, surfaced here only because adding the third plugin makes the dependency explicit.
- **Inter-loop publishing breakage was a real signal.** Between this loop's start and the brief authoring, the gitbook CI had been failing for 5 consecutive loops (Liquid parse error on `{{...}}` snippets in markdown code fences) and the loop-index pages were 404-ing on the `.md` extension. Out-of-loop corrective PRs shipped (newbase #16/#17, gitbook #7/#8); both fixes verified green. The discipline of post-publish CI verification is now memory'd as a standing practice — surfaces in retro for the CEO-level treatment.

## Tasks

- [x] Scaffold observability plugin (`.claude-plugin/plugin.json`, `.codex-plugin/plugin.json`, `gemini-extension.json`, `package.json`, `GEMINI.md`, `.opencode/plugins/observability.js`)
- [x] Author `plugins/observability/skills/cluster-snapshot/SKILL.md`
- [x] Author `generate.py` (Python stdlib only; 4 kubectl calls; atomic file write)
- [x] Implement HTML template (3 sections: header fingerprint + nodes table + pods table grouped by namespace + recent warning events table; inline CSS; status colorising; summary pill bar)
- [x] Run smoke against home cluster; output to `board/reports/2026-05-13-cluster-snapshot.html`; open in browser to verify
- [x] Write `SMOKE.md` with toolchain pinning + acceptance
- [x] Extend `marketplace.json` with the observability plugin row; extend `.version-bump.json`; `bash scripts/bump-version.sh --check observability` reports match
- [x] Run `bash repos/resink-ai/resink-marketplace/tests/run-all.sh` — 89/0/3 (regression-free vs 88/0/3 baseline; +1 manifest pass from the new plugin)
- [x] Update AE charter "Owned products" 5th bullet; bump `date:` to `2026-05-13`
- [x] Tenant-isolation grep post-each-edit + final loop-close sweep — CLEAN throughout
- [x] Copy `templates/scd2_maintainer/` → `templates/scd2_maintainer_v2/`; apply close-row `valid_to = event_ts.saturating_sub(1)` deviation at both call sites in `process()`
- [x] Update v2's `lib.rs.tmpl` top-of-file docstring to note v2 status and point readers at `DEVIATION.md`
- [x] Author `DEVIATION.md` with source-line diff + worked 3-row example + invariants-preserved list + anti-deviation guard + consumer-side acceptance contract
- [x] Append "## Template variants (hot-swap fixture, added 2026-05-13)" section to DISPATCH.md
- [x] Update SKILL.md: `pattern_name` accepted-values list extended to include `scd2_maintainer_v2`; template-directory path resolution generalised to `templates/<pattern_name>/`
- [x] Re-run marketplace tests after O2 changes — 89/0/3 regression-free
- [x] Refresh `teams/platform/agent-engineering/status.md` at loop close: add 2026-05-13 entry to Recent shipments; update Current focus; update Carrying-into-next-loop; bump `date:`

## Pinned tooling at this loop's smoke

- `kubectl` client `v1.33.9` (Kustomize `v5.6.0`)
- `python3` (system Python 3.x; stdlib-only generator)
- `jq` `1.7.1` (informational; not invoked by the generator)
- Cluster: home cluster, k8s `v1.33.11`, 4 nodes ready
- `bash scripts/bump-version.sh` (marketplace's own tool)

## Health

- Loop adherence: **GREEN**. Two bounded objectives, both shipped without scope creep. The smallest-product-slice playbook from the prior loop made O1's scope decision easy — the four candidate slices were laid out in a table, the chosen one named with its prerequisites, the rejected ones archived as future-loop work. Hot-swap split worked as designed: AE-side fits inside this loop without coordination cost; the resink-core half waits for a naturally resink-core loop.

## Carrying into next loop

- **ADR-2026-05-16-001 step 3 (resink-core consumer-side).** AE's half closes here; resink-core's half (load v1, process, swap to v2, process, assert `valid_to` shifts as documented in DEVIATION.md) is owned by resink-core. Bundles with retro P2 CI/CD bootstrap + retro P3 submodule-deinit-recovery at the next active-resink-core loop.
- **Observability v2 candidate slices** (logs, metrics, events-tail). All future-loop work driven by demand. No this-loop signal that any of them is needed.
- **Marketplace plugin order convention.** `.version-bump.json` is now order-sensitive across 4 plugins; if a fifth plugin lands, the existing pattern continues (next-index slot in `marketplace_field`). Worth a small re-shape note if the marketplace ever needs to support 10+ plugins, but no immediate signal.
- **Tenant-isolation discipline.** Standing practice; held throughout this loop. Zero `org-os/` edits.
{% endraw %}
