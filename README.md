# Agent State Submodule

This submodule holds **agent working files**: context, todo lists, session
history, skills, and project architecture insights. It is versioned separately
from the source code so that an agent can continue coding on any machine by
cloning this submodule alongside the main repo.

> **Why a submodule?** Agent state changes independently of source code and
> should not pollute `git status` in the main repos. A submodule keeps these
> files versioned, cloneable, and out of the way.

---

## Where to find everything

| What you need | Path (inside this submodule) | Source of truth |
|---|---|---|
| **Agent context / mission** | `context/context.md` | Synced from `.opencode/context.md` |
| **Agent todo list** | `context/todo.md` | Synced from `.opencode/todo.md` |
| **Archived sessions** | `history/archive/` | Synced from `.opencode/archive/` |
| **Agent skills** | `skills/` | Copied from component `*/skills/` dirs |
| **Project architecture** | `architecture.md` | See below |
| **Key insights / decisions** | `insights.md` | See below |
| **File location map** | `FILE_MAP.md` | See below |

---

## Quick reference: file locations across the whole platform

| Concern | Repo / path | Notes |
|---|---|---|
| **Umbrella repo** | `Work_Credit-RAG_Phase1/` | Pins submodules, integrated startup (`deploy/vast/`) |
| **KB ingestion/retrieval** | `components/knowledgebase/kb-manager/` | `Work_RAG-KB` repo |
| **Orchestrator (LangGraph)** | `components/orchestrator/` | `Work_RAG-Orchestrator` repo |
| **Guardrails** | `components/guardrails/` | `Work_RAG-Guardrails` repo |
| **Tracing collector** | `components/tracing/` | Parent repo (not a submodule) |
| **Hardware/infra setup** | `Work_RAG-Server-Setup/` | Separate repo (H200 hardware) |
| **Agent state (this submodule)** | `agent-state/` | This repo |

---

## Branching model

| Branch | Purpose | Protection |
|---|---|---|
| `main` / `master` | **Production** — deployed code, PR-only | Branch protection: PR + review required |
| `develop` | **Integration / test** — feature branches merge here | PR required |
| `feature/*` | Individual dev work | PR from feature → develop |

Never push directly to `main`/`master` or `develop`. Open a pull request and
require at least one code review before merging.

---

## How to use on a new machine

1. Clone the umbrella repo:
   ```bash
   git clone --recursive https://github.com/AliNikkhah2001/Work_Credit-RAG_Phase1.git
   ```
   The `--recursive` flag clones all submodules including `agent-state/`.

2. If you already cloned without `--recursive`:
   ```bash
   git submodule update --init --recursive
   ```

3. The agent state files are now available under `agent-state/`. Read
   `agent-state/architecture.md` for the project architecture and
   `agent-state/insights.md` for key decisions.

---

## Architecture summary (`architecture.md`)

- **KB Manager**: BM25 + dense MiniLM + RRF + cross-encoder rerank, pgvector HNSW
- **Orchestrator**: LangGraph RAG graph (`validate → retrieve → build_context → guarded_generate → format_response`)
- **Guardrails**: Signal → LLM judge → PII mask/block → upstream LLM → output rails
- **LLM Gateway**: Work_RAG-Server-Setup serves Gemma/Qwen/etc. via round-robin llama.cpp backends on port 9000
- **LiteLLM**: Self-hosted OpenRouter-style gateway (optional, port 4000)

---

## How to update agent state

When an agent session produces new context, todos, or insights:

1. Edit files inside `agent-state/` (e.g. `context/context.md`, `history/archive/`).
2. Commit and push the `agent-state` submodule repo:
   ```bash
   cd agent-state
   git add .
   git commit -m "chore(agent): update context and todo"
   git push
   ```
3. In the umbrella repo, the submodule gitlink will show as modified. Stage and commit:
   ```bash
   cd ..   # back in Work_Credit-RAG_Phase1
   git add agent-state
   git commit -m "chore: advance agent-state submodule"
   git push
   ```

---

## Notes

- `node_modules/`, `__pycache__/`, `*.pyc`, `.env`, `*.log` are **not** stored here.
- This submodule does **not** contain secrets. Never commit API keys, tokens, or passwords.
- Large binary files (models, docker images) live in `offline-prep/` (gitignored).
