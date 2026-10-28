# Agent File Location Map

This file is the single index for finding agent-related text files across the
platform. If you are an agent starting on a new machine, start here.

## 1. Agent state submodule (this repo)

| File | Purpose |
|---|---|
| `agent-state/README.md` | This index — start here |
| `agent-state/architecture.md` | End-to-end system architecture |
| `agent-state/insights.md` | Key decisions, lessons, open questions |
| `agent-state/context/context.md` | Current agent mission and context |
| `agent-state/context/todo.md` | Current agent todo list |
| `agent-state/history/archive/` | Archived agent sessions |
| `agent-state/skills/` | Agent skill definitions |

## 2. Component-level agent files

| Concern | Repo | Path |
|---|---|---|
| KB pipeline | `Work_RAG-KB/kb-manager/` | `kb_manager/pipeline/orchestrator.py` |
| KB web routes | `Work_RAG-KB/kb-manager/` | `kb_manager/web/routes/search.py` |
| KB pipeline route | `Work_RAG-KB/kb-manager/` | `kb_manager/web/routes/pipeline.py` |
| KB zip browser | `Work_RAG-KB/kb-manager/` | `kb_manager/web/routes/zip_browser.py` |
| Orchestrator graph | `Work_Credit-RAG_Phase1/components/orchestrator/` | `src/work_rag_orchestrator/graph.py` |
| Orchestrator retrieve | `Work_Credit-RAG_Phase1/components/orchestrator/` | `src/work_rag_orchestrator/nodes/retrieve.py` |
| Orchestrator build_context | `Work_Credit-RAG_Phase1/components/orchestrator/` | `src/work_rag_orchestrator/nodes/build_context.py` |
| Orchestrator API | `Work_Credit-RAG_Phase1/components/orchestrator/` | `src/work_rag_orchestrator/api.py` |
| Guardrails service | `Work_Credit-RAG_Phase1/components/guardrails/` | `src/work_rag_guardrails/service.py` |
| Guardrails API | `Work_Credit-RAG_Phase1/components/guardrails/` | `src/work_rag_guardrails/api.py` |
| Guardrails judge | `Work_Credit-RAG_Phase1/components/guardrails/` | `src/work_rag_guardrails/judge.py` |
| Tracing collector | `Work_Credit-RAG_Phase1/components/tracing/` | `app.py` |
| Tracing observe | `Work_Credit-RAG_Phase1/components/tracing/` | `observe.py` |

## 3. Platform-level docs

| Document | Path |
|---|---|
| Consolidated agent manual | `Work_Credit-RAG_Phase1/AGENTS.md` |
| Parent README | `README.md` |
| KB README | `Work_RAG-KB/kb-manager/README.md` |
| Guardrails README | `Work_Credit-RAG_Phase1/components/guardrails/README.md` |
| Orchestrator README | `Work_Credit-RAG_Phase1/components/orchestrator/README.md` |
| LiteLLM guide | `Work_Credit-RAG_Phase1/AGENTS.md` (section 5) |

## 4. Evaluation and benchmarks

| Concern | Path |
|---|---|
| Eval artifacts | `Work_Credit-RAG_Phase1/eval/` |
| Benchmarks | `Work_Credit-RAG_Phase1/evaluation/` |
| KB benchmarks | `Work_RAG-KB/kb-manager/versions/` |

## 5. How to resume on a new machine

```bash
# 1. Clone the umbrella repo with all submodules
git clone --recursive https://github.com/AliNikkhah2001/Work_Credit-RAG_Phase1.git
cd Work_Credit-RAG_Phase1

# 2. (If already cloned without --recursive)
git submodule update --init --recursive

# 3. Read the agent onboarding files
cat agent-state/README.md
cat agent-state/architecture.md
cat agent-state/insights.md
cat agent-state/context/context.md
cat agent-state/context/todo.md

# 4. Check current branch
git branch --show-current   # should be 'develop' for integration work
```

## 5. Making changes (PR workflow)

1. Create a feature branch from `develop`:
   ```bash
   git checkout develop
   git checkout -b feature/your-change
   ```
2. Make changes, commit, push.
3. Open a pull request from `feature/your-change` → `develop`.
4. After code review and approval, merge into `develop`.
5. When ready for release, PR `develop` → `main` (production).
