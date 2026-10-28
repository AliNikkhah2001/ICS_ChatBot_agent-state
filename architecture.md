# Project Architecture

This document describes the end-to-end architecture of the Work Credit RAG
Platform. It is the single source of truth for how data flows through the
system.

## High-level data flow

```
Browser (:13000 Open WebUI)
    │
    ▼
Orchestrator (:8100, LangGraph RAG graph)
    │
    ├─ validate_input (guardrails input check)
    ├─ retrieve (KB: BM25 + dense + RRF + rerank)
    ├─ build_context (assemble prompt with entity data)
    ├─ guarded_generate (Guardrails :8200 → Gemma :18000 or Manager :9000)
    ├─ format_response (citations + audit)
    └─ response → Open WebUI → Browser
```

## Service map (H200 host)

| Service | Port | Repo | Role |
|---|---|---|---|
| LLM Inference Manager | 9000 | Work_RAG-Server-Setup | OpenAI-compatible gateway, round-robin over backends |
| llama.cpp backends | 8080-8084, 8090 | Work_RAG-Server-Setup | Per-GPU model servers (Gemma, Qwen, etc.) |
| Embedding servers | 8001-8003 | Work_RAG-Server-Setup | MiniLM, bge-m3, paraphrase-MiniLM |
| KB Manager | 8000 | Work_RAG-KB | BM25 + dense hybrid search, entity lists |
| Guardrails | 8200 | Work_RAG-Guardrails | Safety rails, guarded LLM gateway |
| Orchestrator | 8100 | Work_RAG-Orchestrator | LangGraph RAG agent |
| Tracing collector | 3000 | Work_Credit-RAG_Phase1 | Fallback Langfuse-compatible collector |
| Langfuse v2 | 3001 | External | Full trace UI |
| Open WebUI | 13000 | Docker | Chat frontend |
| LiteLLM (optional) | 4000 | External | Self-hosted OpenRouter-style gateway |

## Key components

### KB Manager (`components/knowledgebase/kb-manager/`)
- Hybrid search: BM25 (tantivy) + dense (MiniLM-L12 384d) + RRF fusion (k=60)
- Cross-encoder rerank (BAAI/bge-reranker-v2-m3)
- Entity lists with rich Persian descriptions, exposed via `POST /search/supplement`
- pgvector HNSW option, Redis cache (TTL 600s)

### Orchestrator (`components/orchestrator/`)
- LangGraph state machine with 5 nodes
- Proactive entity-list tool calls triggered by keyword matching
- Tracing via direct HTTP to Langfuse v2 (with JSONL fallback)

### Guardrails (`components/guardrails/`)
- Deterministic signals (injection, jailbreak, HurtLex, profanity, scope)
- LLM judge (NeMo LLMRails on local Gemma)
- PII mask-and-continue vs hard block
- Fail-closed on timeout/error

## Context window management

The Triton Gemma backend has an 8192-token context window. To stay within it:
- Entity-list payloads are trimmed to top-3 lists, 4KB each, query-token filtered
- RAG context capped at ~6000 chars
- Total prompt (system + context + entity data) target: ~20K chars (~7K tokens)

## Environment parity

- Production: `main` / `master` branch, deployed via `deploy/vast/start.sh`
- Test: `develop` branch, same stack via `deploy/vast/`
- All services require `HF_HOME` + `HF_HUB_CACHE` pointing to the same offline model dir
