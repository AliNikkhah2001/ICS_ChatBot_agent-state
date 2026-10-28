# Agent File Location Map

Single index for finding agent-related text files across the platform. Start here
when resuming on a new machine.

---

## 1. Agent state submodule (this repo)

| File | Purpose |
|---|---|
| `README.md` | This index — start here |
| `architecture.md` | End-to-end system architecture |
| `insights.md` | Key decisions, lessons learned, open questions |
| `coding-standards.md` | Python/FastAPI/db/model-serving/Persian rules + never-do list |
| `git-workflow.md` | Branch protection, PR workflow, submodule workflow |
| `FILE_MAP.md` | This file — index of all agent files |
| `context/context.md` | Current agent mission |
| `context/todo.md` | Current agent todo list |
| `history/archive/` | Archived agent sessions |
| `skills/` | Agent skill definitions |
| `onboarding.md` | New-machine setup checklist |

---

## 2. Component-level source files

| Concern | Repo | Path |
|---|---|---|
| KB pipeline orchestrator | Work_RAG-KB | `kb-manager/kb_manager/pipeline/orchestrator.py` |
| KB web routes (search) | Work_RAG-KB | `kb-manager/kb_manager/web/routes/search.py` |
| KB pipeline route | Work_RAG-KB | `kb-manager/kb_manager/web/routes/pipeline.py` |
| KB zip browser | Work_RAG-KB | `kb-manager/kb_manager/web/routes/zip_browser.py` |
| KB database models | Work_RAG-KB | `kb-manager/kb_manager/models/database.py` |
| KB config | Work_RAG-KB | `kb-manager/kb_manager/config.py` |
| KB settings | Work_RAG-KB | `kb-manager/kb_manager/settings.py` |
| KB dense index | Work_RAG-KB | `kb-manager/kb_manager/dense.py` |
| KB embedder | Work_RAG-KB | `kb-manager/kb_manager/embedder/sentence_transformer.py` |
| Orchestrator graph | Work_Credit-RAG_Phase1/components/orchestrator | `src/work_rag_orchestrator/graph.py` |
| Orchestrator retrieve node | Work_Credit-RAG_Phase1/components/orchestrator | `src/work_rag_orchestrator/nodes/retrieve.py` |
| Orchestrator build_context | Work_Credit-RAG_Phase1/components/orchestrator | `src/work_rag_orchestrator/nodes/build_context.py` |
| Orchestrator API | Work_Credit-RAG_Phase1/components/orchestrator | `src/work_rag_orchestrator/api.py` |
| Orchestrator state | Work_Credit-RAG_Phase1/components/orchestrator | `src/work_rag_orchestrator/state.py` |
| Orchestrator tracing | Work_Credit-RAG_Phase1/components/orchestrator | `src/work_rag_orchestrator/tracing.py` |
| Orchestrator clients | Work_Credit-RAG_Phase1/components/orchestrator | `src/work_rag_orchestrator/clients/` |
| Guardrails service | Work_Credit-RAG_Phase1/components/guardrails | `src/work_rag_guardrails/service.py` |
| Guardrails API | Work_Credit-RAG_Phase1/components/guardrails | `src/work_rag_guardrails/api.py` |
| Guardrails judge | Work_Credit-RAG_Phase1/components/guardrails | `src/work_rag_guardrails/judge.py` |
| Guardrails dashboard | Work_Credit-RAG_Phase1/components/guardrails | `src/work_rag_guardrails/dashboard.py` |
| Tracing collector | Work_Credit-RAG_Phase1/components/tracing | `app.py` |
| Tracing observe | Work_Credit-RAG_Phase1/components/tracing | `observe.py` |

---

## 3. Platform-level documentation

| Document | Path |
|---|---|
| Consolidated agent manual (umbrella) | `Work_Credit-RAG_Phase1/AGENTS.md` |
| Parent README | `README.md` |
| KB README | `Work_RAG-KB/kb-manager/README.md` |
| Guardrails README | `Work_Credit-RAG_Phase1/components/guardrails/README.md` |
| Orchestrator README | `Work_Credit-RAG_Phase1/components/orchestrator/README.md` |
| LiteLLM guide | `Work_Credit-RAG_Phase1/AGENTS.md` (section 5) |
| H200 hardware runbook | `Work_RAG-Server-Setup/README.runbook.md` |

---

## 4. Evaluation & benchmarks

| Concern | Path |
|---|---|
| Eval artifacts | `Work_Credit-RAG_Phase1/eval/` |
| Benchmarks | `Work_Credit-RAG_Phase1/evaluation/` |
| KB benchmarks | `Work_RAG-KB/kb-manager/versions/` |

---

## 5. New-machine onboarding

```bash
# 1. Clone with all submodules
git clone --recursive https://github.com/AliNikkhah2001/Work_Credit-RAG_Phase1.git
cd Work_Credit-RAG_Phase1

# 2. If already cloned without --recursive
git submodule update --init --recursive

# 3. Read agent onboarding
cat agent-state/onboarding.md
cat agent-state/README.md
cat agent-state/architecture.md
cat agent-state/coding-standards.md
cat agent-state/context/context.md
cat agent-state/context/todo.md

# 4. Check branch
git branch --show-current   # should be 'develop' for integration work
```

---

## 6. Making changes (PR workflow)

1. Create feature branch from `develop`:
   ```bash
   git checkout develop && git pull origin develop
   git checkout -b feature/your-change
   ```
2. Make changes, run tests, compile-check.
3. Commit with conventional prefix (`feat:`, `fix:`, `docs:`, `chore:`).
4. Push feature branch.
5. Open PR: `feature/your-change` → `develop`.
6. Request code review. Address comments.
7. Merge to `develop`.
8. For release: PR `develop` → `main` (production), then deploy.
