# Release tagging policy — Work Credit RAG umbrella

Status: **adopted as policy** (2026-11-14). Written by the branch/tag hygiene worker.
This file is the single source of truth for how releases are tagged across the parent
repo and its submodules. It does not change branch protection or CI configuration.

---

## 1. Scope

Versioning is **umbrella-first**. One release tag on the parent repo names an exact,
reproducible application state, because the parent pins every submodule by gitlink SHA.
Submodule tags are informative; the parent tag is authoritative.

| Level | Tag lives in | Authoritative? | Purpose |
|---|---|---|---|
| Parent (umbrella) | `Work_Credit-RAG_Phase1` | **Yes** | Names the deployable app state (pins + deploy/ + contracts/) |
| Submodule | `Work_RAG-KB`, `-Orchestrator`, `-Guardrails`, `-Server-Setup`, `-WebUI`, `-Evaluation_Suite` | No | Marks a component milestone other consumers may pin |
| Marker | parent | No | Mutable pointer to the newest promoted release |

A parent tag is only meaningful if every gitlink it records resolves on that
submodule's remote. That is the primary reason red builds happen after "pin advance"
commits — see §5.

---

## 2. Version scheme

```
v<MAJOR>.<MINOR>.<PATCH>[-<pre-release>]
```

- **MAJOR** — breaking contract change (`contracts/`, request/response shape), or a
  change that requires a coordinated redeploy of all services.
- **MINOR** — new capability (retrieval feature, subspace, guardrail policy, WebUI
  surface) that is backward compatible.
- **PATCH** — fixes, pin-only advances, CI/deploy changes, docs.
- Pre-release suffix `-rc.N` for a build that is built and healthy on **stage** but
  has not completed a **production (GPU `main`)** deploy.

Rules:

1. **Annotated tags only** for releases. `git tag -a` — never lightweight
   (`v1.1.4` and `v1.2.0` in the parent are lightweight today and are grandfathered,
   not imitated).
2. Tags are **immutable**. Never move or re-point a release tag; issue the next
   PATCH instead. Force-pushing a tag is prohibited.
3. Tags are created on the **merge commit that is already on `main`** — never on a
   feature branch, never on an unmerged commit.
4. One release tag per promoted state. Do not tag both `main` and `develop` with the
   same release number; if their trees are identical, tag `main` only and let
   `develop` point at the same content (see §6 for the `develop` case).
5. `stable` / `last-good` are **markers**, not releases: they are force-updated to the
   newest fully-qualified release. They must never be the only record of a version.

### Tag message template (required)

```
<project> v<X.Y.Z> — <one-line summary>

Parent:   <40-char sha>
Branch:   main
UTC:      <ISO-8601>
Stage:    jenkins dana-rag/develop #<N> — <SUCCESS|...> (<sha>)
Prod:     jenkins dana-rag/main #<N> — <SUCCESS|...> (<sha>)
Health:   health.sh <pass|n/a> (<ports>) | smoke <pass|n/a>
Pins:
  ag    agent-state              <sha> (<branch>)
  eval  components/evaluation    <sha> (<branch>)
  grd   components/guardrails    <sha> (<branch>)
  kb    components/knowledgebase <sha> (<branch>)
  orch  components/orchestrator  <sha> (<branch>)
  srv   components/server-setup  <sha> (<branch>)
  web   components/webui         <sha> (<branch>)
Evidence: <URL to Jenkins build / health report>
```

The `Pins:` block is mandatory. It makes "which component was in production at
vX.Y.Z" answerable from `git show vX.Y.Z` without checking out the tree.

---

## 3. Gate: what qualifies a commit for a release tag

All of the following must be true. Any failure means **no tag** — fix forward.

| # | Gate | How to verify |
|---|---|---|
| G1 | Commit is on `main` (parent) via merged PR, not a direct push | `git log --first-parent --no-merges origin/main` must contain no commit lacking an associated PR |
| G2 | Every gitlink resolves on its submodule remote **and** is reachable from that submodule's production branch (`main`, or `master` for KB) | `git submodule status --recursive` clean + `git -C <sub> merge-base --is-ancestor <pin> origin/<prod>` |
| G3 | Jenkins `dana-rag/main` build for that exact SHA is `SUCCESS` **with the deploy stages actually executed** (not `skipped due to when conditional`) | build log must show `Deploy prod` ran and `E2E stage` ran |
| G4 | `bash deploy/gpu/health.sh` green on the production host for that SHA | health output captured in the ops report |
| G5 | E2E smoke passes: greeting, one credit QA, KB `/search/api` returns hits | `curl` matrix captured |
| G6 | No secrets / artifacts in the tree | `git ls-tree -r --name-only <sha> \| grep -iE '\.env$\|secret\|credential\|\.pem$\|\.npz$\|\.log$'` returns only `*.env.example` / redacted docs |
| G7 | Working tree clean enough that the tag cannot be mistaken for uncommitted work | `git status --porcelain` empty at tag time |

**A `SUCCESS` build in which every deploy and E2E stage was skipped does not satisfy
G3.** This is the most common false green in this repo: the laptop controller runs
with `CI_MAY_DEPLOY_PROD=0`, so `main` builds go green without deploying anything.

---

## 4. Procedure

```bash
# 0. Preconditions — read-only, must all pass
git fetch --all --tags --prune
git status --porcelain                 # must be empty for a release
git diff --quiet origin/main develop    # note: identical tree ⇒ same content

# 1. Resolve and prove pins (G2)
git submodule status --recursive
for s in agent-state components/evaluation components/guardrails \
         components/knowledgebase components/orchestrator \
         components/server-setup components/webui; do
  pin=$(git rev-parse "origin/main:$s")
  prod=$([ "$s" = components/knowledgebase ] && echo master || echo main)
  git -C "$s" merge-base --is-ancestor "$pin" "origin/$prod" \
    && echo "OK   $s $pin" || echo "FAIL $s $pin not on $prod"
done

# 2. Prove CI (G3) — must show Deploy prod + E2E executed, not skipped
gh run list --limit 5
# Jenkins UI: dana-rag/main #<N> → check stage list for "Deploy prod"

# 3. Prove health (G4/G5) on the production host
bash deploy/gpu/health.sh
curl -s http://127.0.0.1:8100/ready
curl -s http://127.0.0.1:8100/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"gemma-4-31b","messages":[{"role":"user","content":"سلام"}]}' | head -c 300

# 4. Tag (annotated, on the main merge commit)
SHA=$(git rev-parse origin/main)
git tag -a "v${VER}" "$SHA" -F - <<MSG
work-credit-rag v${VER} — <summary>

Parent:   $SHA
Branch:   main
UTC:      $(date -u +%Y-%m-%dT%H:%M:%SZ)
Stage:    jenkins dana-rag/develop #<N> — SUCCESS ($(git rev-parse origin/develop))
Prod:     jenkins dana-rag/main #<N> — SUCCESS
Health:   health.sh pass | smoke pass
Pins:
  ag    agent-state              $(git rev-parse origin/main:agent-state)
  eval  components/evaluation    $(git rev-parse origin/main:components/evaluation)
  grd   components/guardrails    $(git rev-parse origin/main:components/guardrails)
  kb    components/knowledgebase $(git rev-parse origin/main:components/knowledgebase)
  orch  components/orchestrator  $(git rev-parse origin/main:components/orchestrator)
  srv   components/server-setup  $(git rev-parse origin/main:components/server-setup)
  web   components/webui         $(git rev-parse origin/main:components/webui)
Evidence: <jenkins build url>
MSG

# 5. Push ONLY the tag(s) — never branches
git push origin "refs/tags/v${VER}"

# 6. Move the marker (force-update of a marker tag is allowed; of a release tag is not)
git tag -f -a stable "$SHA" -m "stable → v${VER} (${SHA:0:7})"
git push --force origin refs/tags/stable
```

Submodule release tags (optional, only when a component ships standalone value):

```bash
cd components/orchestrator
git tag -a "v${VER}" "$(git rev-parse origin/main)" -m "orchestrator v${VER} — <summary>"
git push origin "refs/tags/v${VER}"
```

---

## 5. Known failure modes this policy prevents

| Symptom | Cause | Gate that catches it |
|---|---|---|
| Red build right after a "advance pins" commit | Parent gitlink points at a submodule commit that only exists on a local/feature branch and was never pushed | G2 |
| Green build, broken production | Laptop controller (`CI_MAY_DEPLOY_PROD=0`) skips `Deploy prod`/`E2E stage`; `SUCCESS` means "did nothing" | G3 |
| Two different SHAs both called "latest" | `develop` tip is one no-op commit ahead of `main`; tagging both invites divergence | §2 rule 4 + `git diff origin/main develop` |
| Cannot answer "what was in prod at vX" | Lightweight tags without pin manifest | §2 annotated + `Pins:` block |
| Production pin silently behind component `main`/`master` | Pin advance not done after a component PR merged | G2 (must be an ancestor of prod branch, not just "exists") |
| Divergent local `main` in a submodule | Local `main` merged locally but never pushed | §4 step 0 `git status` / fetch check |

---

## 6. Branch semantics for releases

- `main` — always taggable; every release tag lives on a commit reachable from `main`.
- `develop` — gets **no** release tag. If `git diff origin/main origin/develop` is
  empty (trees identical), `develop` already carries the released content; a
  `-rc.N` tag is unnecessary and misleading.
- `feature/*` — never tagged.
- `ops/*`, `chore/*`, `release/*` — never tagged; they are promotion vehicles.

## 7. Naming collisions to avoid

Existing tags in this repo that must **not** be reused or extended:
`final-bug-free`, `v1.0.0-final-bugfree`, `phase1-complete-2026-10-28`,
`legacy/*`, `archive/*`. They are historical markers of a different scheme. The next
release continues the `v1.2.0` line as `v1.3.0` (or `v1.2.1` for a fix-only promote).
