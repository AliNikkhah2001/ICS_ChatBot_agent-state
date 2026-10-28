# Agent Skills

Definitions of what agents can do in each component of the Work Credit RAG
Platform. Use these to scope agent work and match tasks to the right component
repo.

---

## 📋 KB Manager skills (`Work_RAG-KB/kb-manager/`)

| Skill | What it does | Key files |
|---|---|---|
| **Ingest pipeline** | Parse, chunk, embed, store KB documents | `kb_manager/pipeline/orchestrator.py` |
| **Hybrid search** | BM25 + dense + RRF + rerank retrieval | `kb_manager/web/routes/search.py` |
| **Entity lists** | Store/query supplementary dictionaries (reason codes, shareholders) | `kb_manager/models/database.py` (`EntityList`) |
| **Supplement tool** | `POST /search/supplement` — embedding search over entity lists | `kb_manager/web/routes/search.py` |
| **Pipeline routes** | Trigger full/incremental rebuilds via HTTP | `kb_manager/web/routes/pipeline.py` |
| **Zip browser** | Ingest from uploaded ZIP archives | `kb_manager/web/routes/zip_browser.py` |
| **Quality gates** | Validate chunks before storage | `kb_manager/pipeline/quality.py` |
| **Versioning** | Track document versions | `kb_manager/pipeline/versioning.py` |

## 🔄 Orchestrator skills (`Work_Credit-RAG_Phase1/components/orchestrator/`)

| Skill | What it does | Key files |
|---|---|---|
| **LangGraph RAG graph** | 5-node state machine (validate → retrieve → build_context → guarded_generate → format_response) | `src/work_rag_orchestrator/graph.py` |
| **Retrieve node** | KB search + proactive entity tool calls | `src/work_rag_orchestrator/nodes/retrieve.py` |
| **Build context** | Assemble system message with RAG context + entity data | `src/work_rag_orchestrator/nodes/build_context.py` |
| **Guarded generate** | Call Guardrails chat/completions | `src/work_rag_orchestrator/nodes/guarded_generate.py` |
| **Format response** | Citations + audit trail | `src/work_rag_orchestrator/nodes/format_response.py` |
| **Tracing** | Direct HTTP to Langfuse v2, JSONL fallback | `src/work_rag_orchestrator/tracing.py` |
| **API** | FastAPI `/v1/chat/completions`, `/v1/models`, `/health` | `src/work_rag_orchestrator/api.py` |

## 🛡 Guardrails skills (`Work_Credit-RAG_Phase1/components/guardrails/`)

| Skill | What it does | Key files |
|---|---|---|
| **Input rails** | Deterministic signals + LLM judge on input | `src/work_rag_guardrails/service.py` |
| **Output rails** | PII mask / hate / secret detection on output | `src/work_rag_guardrails/service.py` |
| **LLM judge** | NeMo LLMRails on local Gemma | `src/work_rag_guardrails/judge.py` |
| **Dashboard** | Decision counts, block rate, SVG plots | `src/work_rag_guardrails/dashboard.py` |
| **API** | FastAPI `/v1/rails/check`, `/v1/chat/completions` | `src/work_rag_guardrails/api.py` |

## 🔍 Tracing skills (`Work_Credit-RAG_Phase1/components/tracing/`)

| Skill | What it does | Key files |
|---|---|---|
| **Fallback collector** | Langfuse-compatible ingestion, JSONL storage | `app.py` |
| **Timeline builder** | Request → spans → response timelines | `observe.py` |

---

## 🎯 How to match a task to a skill

| Task type | Recommended skill | Repo |
|---|---|---|
| Add a new retrieval feature | Hybrid search | KB Manager |
| Add a new entity list type | Entity lists + Supplement tool | KB Manager |
| Change RAG graph flow | LangGraph RAG graph | Orchestrator |
| Add a new guardrail category | Input rails / LLM judge | Guardrails |
| Fix context window overflow | trim_supplemental + build_context budget | Orchestrator |
| Fix embedding issues | `_dense_encode()` helper | KB Manager |
| Add a new dashboard metric | Dashboard | Guardrails |
| Change tracing backend | Tracing collector | Tracing |
