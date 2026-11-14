# Production Integrity Plan — last-known-good, health-gated deploys, clean promotion

Status: **active** (2026-11-14). Written after the Nov 12–14 silent production
outage and a read-only audit of Jenkins, git flow, submodules and the compose
overlays.

## 1. Goal

The GPU production host always runs, always has a recorded last-known-good
artifact set, and can always be returned to it. Dev changes reach production
only through `develop` → PR → `main`. `main` is always deployable. Jenkins can
never report a success for a deploy that did nothing.

## 2. What actually broke (evidence, not hypothesis)

| # | Finding | Evidence |
|---|---|---|
| 1 | `compose.base.yml` publishes **zero** ports. Every port lives in an overlay. | `grep -n ports deploy/compose/compose.base.yml` → no match; `compose.local.yml:16-106`, `compose.stage.yml:7-29`, `compose.gpu.yml:21-67` |
| 2 | A restart that used base alone recreated containers healthy but unreachable. | `gpu_full_recover_report.sh:583-592` comment documents the exact regression; previously ran `-f compose.base.yml up -d` alone |
| 3 | Prod overlay did not publish webui/tracing/pg/redis at all. | `compose.gpu.yml` before this change: only kb, orchestrator, guardrails |
| 4 | Jenkins marks a build red when the branch cannot even be fetched. | `dana-rag/develop #54` log: `UnknownHostException: api.github.com` |
| 5 | Jenkins marks a build red for a lint tool that is not installed. | `#52`: `error: externally-managed-environment` + `ruff: not found` (both swallowed with `|| true`, but a later step exits 1 — the visible red is `dana-stage-kb-1 is unhealthy`) |
| 6 | A `SUCCESS` build can have done nothing. | Laptop controller has `DANA_ROLE=stage`; `Deploy prod` requires `CI_MAY_DEPLOY_PROD=1`, so `main` builds go green with every deploy stage skipped |
| 7 | `main` and `develop` were pushed directly, bypassing PRs. | `git log --first-parent origin/develop`: `4a4a42e` is a direct push; `origin/main` has `a371d7a` and `4bfd2fa` direct |
| 8 | **No branch protection anywhere.** | `gh api .../branches/main/protection` → 404 `Branch not protected` on all 6 repos; `rulesets` → `[]` |
| 9 | Last-known-good was SHA-only, never an image. | `scripts/ci_record_last_good.sh` writes a sha; no image tag/digest, no config hash |
| 10 | Auto-heal pushed reports into `components/server-setup`, i.e. a **submodule of the repo Jenkins polls**. | `gpu_auto_heal.sh:34-35` (`REPORT_REPO=$ROOT/components/server-setup`) — pushing there dirties a pinned submodule and can trigger prod builds |
| 11 | Health checks are advisory, never gating. | `Jenkinsfile` deploy stage: `curl ... \|\| echo "port $p warming"` then `ci_record_last_good.sh` unconditionally |
| 12 | Rollback re-*builds* from a SHA instead of restoring the exact images. | `ci_rollback.sh:75-81` uses `up -d --build` |
| 13 | Jenkins suffixed workspaces leaked into git status. | `.ci-workspace@2/`, `@3/`, `@tmp` untracked; `.gitignore` only had `.ci-workspace/` |

## 3. Design

### 3.1 Artifact of record

A deploy is identified by a **deploy manifest**, not a commit:

```
deploy/compose/.last_good.json
{
  "sha":        "<parent commit>",        # provenance
  "images":     { "kb": "rag-kb:<TAG>", ... },  # what is actually running
  "digests":    { "rag-kb:<TAG>": "sha256:..." },
  "config_hash":"<sha256 of layered compose config + env keys>",
  "health":     { "8000": "ok", ... },
  "recorded_at":"<ISO-8601Z>",
  "host":       "<hostname>"
}
```

`.last_good_sha` stays as a thin compatibility file (existing scripts read it).

### 3.2 Gate order (deploy is only "good" if all pass)

1. `compose config` renders without error, **and** the rendered config
   publishes the expected host ports (`13000`, `8100`, `8000`, `8200`).
2. Containers are healthy (`docker compose ps --format json`, health=healthy
   when a healthcheck exists).
3. Every endpoint answers: KB `/health`, guardrails `/health`, orchestrator
   `/health` + `/ready`, WebUI `/` (any 2xx/3xx).
4. Only then write `.last_good.json` + `.last_good_sha`; otherwise roll back
   to the previous manifest immediately (same run, no human needed).

### 3.3 Rollback = restore images, not rebuild

`ci_rollback.sh` reads `.last_good.json`, re-tags the recorded digests back to
the recorded tags, `compose up -d --no-build` with the recorded config hash
validated, then re-runs the same health gate.

### 3.4 Reporting must not be able to trigger builds

All automated health/report pushes go to `Work_RAG_HealthCheck` only. The
server-setup path is removed from `gpu_auto_heal.sh`.

### 3.5 Branch protection (the only real fix for "commits straight to main")

Rulesets on `main` + `develop` for all six repos: require PR, 1 approval,
dismiss stale approvals, no force-push, no deletion, include administrators.
Enforcement requires a paid plan for private repos — see §6.

## 4. Implementation

- [x] `scripts/ci_record_last_good.sh` → writes manifest (sha + image tags +
      digests + config hash + health), keeps `.last_good_sha`, keeps `.prev`.
- [x] `scripts/ci_deploy_gate.sh` (new) → port/publish + health verification,
      exits non-zero with a precise reason.
- [x] `scripts/ci_rollback.sh` → restore-from-manifest (re-tag digests,
      `--no-build`), gate after.
- [x] `scripts/gpu_keep_latest.sh` → use the manifest; health-gate before
      recording; never record on failure.
- [x] `scripts/gpu_auto_heal.sh` → report to `Work_RAG_HealthCheck` only;
      don't restart when already healthy; respect last-known-good on failure.
- [x] `Jenkinsfile` → submodule sync already retried; make lint non-fatal for
      real, gate deploy on `ci_deploy_gate.sh`, record only on success.
- [x] `install_daemon*.sh` → unattended daily + 3h retry, push to health repo.
- [x] `.gitignore` → `.ci-workspace@*`, `reports/`, `.last_good.json(.prev)`,
      `deploy/gpu/state/`, `deploy/gpu/sim/.state-*`.
- [x] `deploy/jenkins/docker-compose.yml` → pin `dns: [1.1.1.1, 8.8.8.8]`.
      Build #54 died on `UnknownHostException: api.github.com` inside the
      container before any stage ran; the Docker embedded resolver inherits host
      forwarders and intermittently returned NXDOMAIN.
- [x] **`deploy/gpu/deploy.sh`** — single supported deploy entry point. Fixes the
      layering by construction (profile → files + project + env), runs a port
      PREFLIGHT before starting anything, orders the rollout with readiness
      gates, runs `health.sh` from the host, promotes only on green, and
      auto-rolls-back on red.
- [x] **`deploy/gpu/reconcile.sh`** — auto-heal entry point. Heals *towards the
      last-known-good*, never towards "whatever the working tree says". Never
      loops forever; cadence lives in state (`next_retry_at`), timer fires hourly
      and the script decides if it is due (daily + retry every 3h while red).
- [x] **`deploy/gpu/rollback.sh`** — restores the recorded image IDs as stable
      local tags (`dana-lastgood/<svc>:<stamp>-<sha>`), adds a generated pin
      override LAST in the layering, brings the stack up tier by tier with
      `--no-build`, then re-verifies published ports via `health.sh`.
- [x] **`deploy/gpu/health.sh`** — verdict per service (`OK`, `NO_CONTAINER`,
      `NO_PUBLISHED_PORT`, `EXPECTED_PORT_MISSING`, `UNHEALTHY`, `HOST_PROBE_FAIL`)
      plus a resolved-compose port contract check, so "healthy but unpublished"
      is a named, testable failure instead of a mystery.
- [x] **`deploy/gpu/lib/{common.sh,state.py}`** — profile→layering table, port
      specs, docker-inspect health, and a JSON state/manifest tool (`validate`,
      `get`, `promote`, `order`, history append).
- [x] **`deploy/gpu/sim/simulate.sh`** — GPU-free proof of the guarantees.
      Recreates the Nov 2026 outage shape (base compose only, no `ports:`) and
      asserts: health.sh fails loudly, deploy.sh refuses, a good deploy
      promotes, and a broken stack is healed by rollback with the recorded
      image restored.
- [ ] GitHub branch protection/rulesets — needs plan/permissions (see §6).

## 4a. Defects found BY the simulation (and fixed)

The simulator is not a demo; it caught three real bugs that would have made
recovery fail on the GPU host:

| Bug | Symptom | Fix |
|---|---|---|
| `rollback.sh` pin override had `services:` written once per service | compose rejected the file (`mapping key "services" already defined`), so rollback "failed" and nothing was restored | Write the header exactly once, before the loop |
| Two heal paths (`deploy.sh` auto-rollback + `reconcile.sh`) collide in the same second → same `RUN_ID` → same pin file | Both appended to one file, interleaving YAML keys | Pin file name now includes `$$` and `$RANDOM` |
| Readiness gate read `compose ps {{.Health}}` | Empty for a container that is still registering AND for one with no healthcheck, so the gate passed instantly and health.sh failed later on `health=starting` | New `dana_service_health()` asks `docker inspect` directly and distinguishes `missing` / `starting` / `none` / `healthy` |

Before these fixes the suite was `13 passed / 3 failed`. After: **16 passed / 0
failed**.

## 5. Verification

- `bash -n` on every script touched (all pass).
- `deploy/gpu/sim/simulate.sh` → **16/16 pass** (a: portless layering detected
  and refused; b: healthy deploy promoted with a valid manifest; c: broken stack
  healed by rollback to the recorded image).
- `ci_deploy_gate.sh --ports 8000,8100,8200,13000 --no-compose` against the live
  laptop stack (`DANA_GATE_PROJECT=dana-local`) → PASS.
- `ci_deploy_gate.sh --ports 8000,9999` → FAIL (exit 1), proving the gate is not
  a no-op.
- `ci_record_last_good.sh` writes a manifest with real image IDs; `ci_rollback.sh
  --dry-run` reads it back.
- `gpu_auto_heal.sh --check` → all five ports up, exit 0 (probe-only mode added
  so the daemon can be tested without mutating anything).
- `gpu_keep_latest.sh --dry-run` → correctly identifies remote tip vs
  last-known-good and states the exact action it would take.

## 6. Still blocked on the user

1. **Branch protection needs a GitHub Pro/Team plan** for private repos. The
   API returns 403 on rulesets for private repos on free plans. Until then the
   only enforcement is process + the release gate (§7 of `release-tagging.md`).
2. **GPU host access.** Nothing in this plan can be verified on production
   without it. Every script here is host-agnostic and dry-run safe.
3. **Jenkins DNS.** `#54` failed on `UnknownHostException: api.github.com`
   inside `dana-jenkins`. That is a Docker DNS/proxy problem, not a pipeline
   problem. Fix: `dns: [1.1.1.1, 8.8.8.8]` on the jenkins service, or set
   `HTTP_PROXY`/`NO_PROXY` on that container.
