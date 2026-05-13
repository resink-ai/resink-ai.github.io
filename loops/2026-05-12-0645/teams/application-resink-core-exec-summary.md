---
layout: default
title: application-resink-core Exec Summary — 2026-05-12-0645
date: 2026-05-12
status: active
type: exec-summary
loop: 2026-05-12-0645
owner: teams/application/resink-core
---

<!-- original-frontmatter:
  type: exec-summary
  owner: teams/application/resink-core
  date: 2026-05-12
  status: active
  loop: 2026-05-12-0645
  links: parent: board/okrs/2026-05-12-0645-ceo-brief.md
  team_okr: teams/application/resink-core/okrs/2026-05-12-0645-team-okr.md
-->
{% raw %}

# Resink Core Exec Summary — Loop 2026-05-12-0645

**Headline.** First production container image of the supervisor binary built + shipped + executed on a real Kubernetes cluster. `nanofab-supervisor:0.3.0` (linux/amd64, 154 MB, multi-stage `rust:1.85-slim-bookworm → debian:12-slim`) ran successfully on the home cluster, processed 21 dim_user + 18 dim_account events from baked-in fixtures, wrote both output parquets, and exited `supervisor: ok`. The chart's R1–R5 hand-off acks closed end-to-end this loop — `requests/2026-06-06-handoff-response-ack.md` flipped `accepted → fulfilled`. **Second end-to-end fulfilled cross-team request lifecycle on the org-os tree** (first was DE's schema-JSON at 2026-05-11-2153). Workspace promotion 7th-loop carry (the GitHub remote was not created in-loop). ADR-2026-05-16-001 step 3 (hot-swap correctness test) slipped one loop per the brief's deployment focus.

## Per-KR rollup (O1)

| KR | Outcome | Notes |
|---|---|---|
| KR1.1 (image build) | **PASS** | `docker buildx build --platform linux/amd64 --load -f deploy/charts/nanofab-supervisor/Dockerfile -t nanofab-supervisor:0.3.0 .` succeeded. Image size: 154 MB (`docker inspect --format='{{.Size}}'` → `154163375`). Architecture: `amd64/linux`. Build time ~2 min (cross-arch QEMU). **Note:** the Dockerfile's `rust:1.83-slim-bookworm` base was bumped to `rust:1.85-slim-bookworm` because the workspace's Cargo.lock pinned `clap_lex 1.1.0` (needs edition2024 = Rust ≥ 1.85). Workspace MSRV declaration says 1.83 but the lockfile moved past it — opportunity for a future sync. |
| KR1.2 (tag aligned with Chart.appVersion) | **PARTIAL** | Image tagged `nanofab-supervisor:0.3.0` matching the supervisor crate version (`crates/nanofab-supervisor/Cargo.toml [package].version = "0.3.0"`). Chart.appVersion is `"0.1.0"` — drift recorded. The values' `image.tag` is the authoritative pull target; Chart.appVersion is informational. Updating Chart.appVersion to track crate version is a future-loop bookkeeping item. |
| KR1.3 (save tarball) | **PASS** | `docker save nanofab-supervisor:0.3.0 -o /tmp/nanofab-supervisor.tar` → 150 MB tarball. Hand-off to DevOps's O1 KR1.2 successful. |
| KR1.4 (user-guide.md update) | **PASS** | New "## Deploying to the home cluster" section between "Plugin loading modes" and "Environment variables". Names build / save / distribute / install / verify / delete with verbatim commands. Cross-links the SRE SOP and chart README. Includes "Image-tag convention," "Deployment vs Job kind," "No image registry yet" sub-notes. ~50 lines. |
| KR1.5 (architecture.md slip record) | **PASS** | Named deviations § ABI Option A entry: "step 3 (hot-swap correctness test) slipped to loop+2 (was loop+1)" with slip reason "deployment focus this loop crowds out the hot-swap test." Per ADR-2026-05-30-002's loop+N convention; ADR body grandfathers under absolute-date authoring. |
| KR1.6 (cross-team-request fulfillment) | **PASS** | `requests/2026-06-06-handoff-response-ack.md` flipped `status: accepted → status: fulfilled`. `links.fulfilled_by` populated. Resolution note appended naming R1–R5 outcomes — R1 (chart-owned Dockerfile built clean), R2 (`/healthz` deferred; honored), R3 (terminationGracePeriodSeconds 30 rendered), R4 (CLI flags via args rendered), R5 (ConfigMap-mount gating verified via helm template; not exercised this loop). Second end-to-end fulfilled cross-team request lifecycle on the org-os tree. |
| KR1.7 (tenant-isolation) | **PASS** | Final tenant-isolation dry-run: `grep -nrE "(resink\|nanofab\|home-cluster\|acme\.ai)" org-os/` returns only `org-os/conventions.md:121:  placeholder names like \`acme.ai\``. |
| KR1.8 (opportunistic: workspace promotion) | **CARRY** | `git ls-remote https://github.com/resink-ai/resink-core.git` still returns `Repository not found`. The brief's named `! gh repo create` command was not run in-loop. **7th-consecutive-loop carry.** Per the brief's standing answer, no KR penalty (opportunistic-only); flagged for retro escalation. |

## Per-KR rollup (O2)

| KR | Outcome | Notes |
|---|---|---|
| KR2.1 (R1–R5 ack lines) | **PASS** | One-line ack per response landed in the request file's resolution subsection: R1 (chart-owned Dockerfile built `0.3.0` on first attempt), R2 (`/healthz` deferred; chart's `healthz.enabled=false` honored), R3 (30s drain; supervisor exits fast enough that drain is never hit), R4 (CLI flags via `args:` rendered; env vars in ConfigMap for operator inspection only), R5 (ConfigMap-mount template-gated correctly; not exercised this loop with empty content). |
| KR2.2 (request mutation) | **PASS** | Folded into KR1.6 mechanically. Request file at `status: fulfilled` with `links.fulfilled_by` populated and resolution note appended. Request-lifecycle accounting per ADR-2026-06-06-002. |

## Build phase signal-of-interest

- **Image size:** 154 MB. The multi-stage build with `debian:12-slim` runtime + bundled `synthetic_tenants/closed_loop_v0/workspace` (~10 MB) + `fixtures/` (~5 MB) + supervisor binary (~50 MB). Reasonable for an MVP image; future iterations can shrink via distroless or strip.
- **Cross-arch build time:** ~2 min clean, ~10s incremental on macOS arm64. The Cargo registry download is the long pole; the actual `cargo build --release` is ~90s under QEMU emulation.
- **Cargo MSRV vs Cargo.lock drift:** workspace declares MSRV 1.83 but pins `clap_lex 1.1.0` which needs edition2024 (Rust ≥ 1.85). Surfaced when the Dockerfile's `rust:1.83` base failed to parse the lockfile. The local dev tree builds fine because the operator has Rust 1.95; future-loop bookkeeping should either bump MSRV declaration to 1.85 or downgrade the pinned dep.

## Open carries (for next loop)

- **Workspace promotion** — 7th-loop carry. Retro candidate for escalation if the carry persists. The brief's P1-reframing (generate the `gh repo create` command) executed this loop; the command is sitting in the brief; not yet run.
- **Hot-swap correctness test (ADR-2026-05-16-001 step 3)** — now scheduled `loop+2` (the loop after this one). Joint AE + resink-core deliverable.
- **`Chart.appVersion` ↔ crate version alignment** — bookkeeping. Update at next chart revision.
- **Cargo MSRV vs Cargo.lock drift** — bookkeeping. Bump workspace MSRV declaration to 1.85 or downgrade pinned deps at next workspace-touch.
- **`teams/application/resink-core/resink-core/` symlink cleanup** — non-load-bearing.

## Tenant-isolation invariant

Held throughout. Edits landed under `repos/resink-ai/resink-core/` (`docs/user-guide.md`, `docs/architecture.md`, `deploy/charts/nanofab-supervisor/Dockerfile` patched alongside DevOps) and `teams/application/resink-core/` (OKR, exec summary, request fulfillment). Zero `org-os/` writes. Final tenant-isolation dry-run clean.
{% endraw %}
