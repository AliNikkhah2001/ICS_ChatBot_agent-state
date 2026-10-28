# Agent Onboarding — New Machine Setup

Use this checklist when starting work on a new machine.

## Prerequisites

- Git installed
- Python 3.11+ (KB) or 3.12+ (orchestrator/guardrails)
- `uvicorn`, `fastapi`, `httpx`, `sqlalchemy[asyncio]`, `asyncpg`
- Access to the GitHub repos (credentials configured)

## Step 1: Clone the umbrella repo with all submodules

```bash
git clone --recursive https://github.com/AliNikkhah2001/Work_Credit-RAG_Phase1.git
cd Work_Credit-RAG_Phase1
```

If already cloned without `--recursive`:
```bash
git submodule update --init --recursive
```

## Step 2: Verify submodule status

```bash
git submodule status --recursive
```

Expected output shows all submodules pinned to commits with `-` (not `+`):
```
 670c8df... components/knowledgebase (heads/master)
 d43c071... components/guardrails (v1.1.0-1-g...)
 86537a8... components/orchestrator (v1.1.1-1-g...)
 2d20462... components/server-setup (final-bug-free-5-g...)
 d0bfbcb... agent-state (heads/main)
```

A `+` prefix means the submodule is out of sync — run `git submodule update --recursive`.

## Step 3: Read agent context

```bash
cat agent-state/context/context.md
cat agent-state/context/todo.md
cat agent-state/architecture.md
cat agent-state/coding-standards.md
```

## Step 4: Check current branch

```bash
git branch --show-current
```

Should be `develop` for integration work. Create feature branches from here:
```bash
git checkout -b feature/your-change
```

## Step 5: Verify services are running (if testing locally)

```bash
for p in 18000 8000 8200 8100; do printf "%s: " "$p"; curl -s --max-time 8 http://127.0.0.1:$p/health | head -c 120; echo; done
```

## Step 6: Make changes

Follow the PR workflow in `agent-state/git-workflow.md`:
1. Feature branch from `develop`
2. Commit with conventional prefix
3. Push feature branch
4. Open PR → `develop`
5. Code review → merge

## Step 7: Release (production)

```bash
git checkout main
git pull origin main
git merge --no-ff develop
git push origin main
bash deploy/vast/stop.sh
bash deploy/vast/start.sh
bash deploy/vast/health.sh
```
