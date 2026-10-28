# Agent Context — Current Mission

**Platform:** Work Credit RAG Platform (Persian-first RAG with LangGraph + Guardrails)
**Branch:** `develop` (integration/test) → `main`/`master` (production)
**Date:** 2026-10-28

## Current state

All repos have been cleaned to two branches:
- `main` / `master` — production
- `develop` — integration/test staging

The `vast-gemma4-migration` branch has been deleted from all repos.
Branch protection rules should be configured on GitHub (PR + code review required).

## Active component repos

| Repo | Default branch | Owns |
|---|---|---|
| Work_RAG-KB | `master` | KB ingestion, retrieval, reranking |
| Work_Credit-RAG_Phase1/components/knowledgebase | `master` | KB submodule (pinned to Work_RAG-KB) |
| Work_Credit-RAG_Phase1/components/orchestrator | `main` | LangGraph RAG agent |
| Work_Credit-RAG_Phase1/components/guardrails | `main` | Safety + guarded Gemma |
| Work_Credit-RAG_Phase1 (parent) | `main` | Pins, integrated startup, E2E |

## Key architectural facts

- KB: BM25 + dense MiniLM-L12 (384d) + RRF k=60 + cross-encoder rerank, pgvector HNSW
- Orchestrator: LangGraph (validate → retrieve → build_context → guarded_generate → format_response)
- Guardrails: signals → LLM judge → PII mask/block → upstream LLM → output rails
- LLM gateway: Work_RAG-Server-Setup port 9000, round-robin llama.cpp backends 8080-8084
- Triton Gemma context window: 8192 tokens — entity payloads trimmed to 4KB total

## Recent changes (supplement entity-list tool path)

- `_dense_encode()` helper replaces dead `dense.model.encode()` calls (5 sites in search.py)
- Pipeline route passes `SentenceTransformerEmbedder()` so entity descriptions get embeddings
- `trim_supplemental()` caps entity data at top-3 lists × 4KB each
- `build_context` total budget: 4000 chars
- Guardrails `finish_reason="length"` instead of `"error"` (pydantic Literal fix)
- `should_clarify` + retrieve gates short-circuit when entity evidence exists
- Code-token fallback: mixed alphanumeric tokens (MTX1, SBD1) match at threshold regardless of cosine

## Open tasks

- Configure GitHub branch protection rules (PR + 1 review required on main/master and develop)
- Create GitHub repo for agent-state submodule and push local agent-state to it
- Run draft tests against live stack (KB :8000, orchestrator :8100, guardrails :8200)
