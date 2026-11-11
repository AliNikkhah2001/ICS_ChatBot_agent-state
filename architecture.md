# Project Architecture

End-to-end architecture of the Work Credit RAG Platform. This is the single
source of truth for how data flows through the system. Generated from the
READMEs and actual code in `Work_RAG-KB`, `Work_Credit-RAG_Phase1`, and
`Work_RAG-Server-Setup`.

---

## 📊 High-level data flow (MVP path)

```
Browser (:13000 Open WebUI)
    │
    ▼
Orchestrator (:8100, LangGraph RAG graph)
    │
    ├─ validate_input      → Guardrails :8200 POST /v1/rails/check (input)
    │    ├─ blocked? → format_refusal → back to WebUI (finish_reason=content_filter)
    │    └─ allowed → KB :8000 POST /search/api
    ├─ retrieve            → KB hybrid search (BM25 + MiniLM + RRF + rerank, top_k=5)
    ├─ build_context       → MAX_CHUNKS=5, MAX_CONTEXT_CHARS=6000
    ├─ guarded_generate    → Guardrails :8200 POST /v1/chat/completions
    │    └─ Guardrails → upstream LLM (Gemma :18000 or Manager :9000)
    │    └─ output rails (PII mask / hate / secret) → Orchestrator
    ├─ format_response     → citations + audit → WebUI → Browser
    └─ async trace         → Langfuse :3001 (+ fallback JSONL :3000)
```

---

## 🗺 Service map (H200 host)

| Service | Port | Repo | Role |
|---|---|---|---|
| LLM Inference Manager | 9000 | Work_RAG-Server-Setup | OpenAI-compatible gateway, round-robin over llama.cpp backends |
| llama.cpp backends | 8080-8084, 8090 | Work_RAG-Server-Setup | Per-GPU model servers (Gemma, Qwen, Nemotron, etc.) |
| Embedding servers | 8001, 8002, 8003 | Work_RAG-Server-Setup | multilingual-e5-small (384d), bge-m3 (1024d), paraphrase-multilingual-MiniLM (384d) |
| KB Manager | 8000 | Work_RAG-KB | BM25 + dense hybrid search, entity lists, pgvector HNSW |
| Guardrails | 8200 | Work_RAG-Guardrails | Safety rails, guarded Gemma gateway |
| Orchestrator | 8100 | Work_RAG-Orchestrator | LangGraph RAG agent |
| Tracing collector (fallback) | 3000 | parent | JSONL trace collector, Langfuse-compatible |
| Langfuse v2 UI | 3001 | external | Full trace UI (Postgres + Clickhouse) |
| Open WebUI | 13000 | Docker | Chat frontend → Orchestrator `:8100/v1` |
| LiteLLM (optional) | 4000 | external | Self-hosted OpenRouter-style gateway |

---

## 🔧 Component details

### KB Manager (`Work_RAG-KB/kb-manager/`)

**Ingest pipeline:**
1. Scan `kb-source/` for `.xlsx`, `.docx`, `.pdf`
2. Parse via parser registry
3. Preprocess (clean + Persian normalization)
4. Semantic chunk (qa / article / kv-pair / reason-detail / staff-profile)
5. Quality gates → dedup → embed (MiniLM-L12 384d) → store (SQLite or pgvector HNSW)

**Retrieval pipeline (per query):**
1. Query enhance / expansion (beam 5) / reform / optional HyDE
2. BM25 (tantivy, Rust, RAM) + Dense cosine (GPU, MiniLM-L12 384d)
3. RRF fusion (k=60)
4. Cross-encoder rerank (BAAI/bge-reranker-v2-m3, GPU)
5. Final top-5 + scores (~40 ms total)
6. Redis cache (TTL 600 s)

**Query classification (3-way LLM classifier):**
- `greeting` — no KB lookup
- `specification` — locks subspace (چک/تسهیلات/شرکت/عمومی)
- `question` — passes to ambiguity resolver (Mizan System-1 + Gemma hybrid)

**Entity lists (supplement tool):**
- Built by `kb_manager/entity_lists.py` into structured `content_json` (`_meta.kind`, `subspace_ids`)
- Kinds: `score_table` (person/legal/cheque), `loan_catalog`, `shareholders`, `data_providers`, `report_consumers`
- Stored in `EntityList` with `description_embedding`
- Step-8 inject on `/search/api` (cosine ≥ 0.6) + subspace score-table force-inject (no cross-model bleed)
- Tool path: `POST /search/supplement` (threshold 0.4, subspace/filter_path aware)
- Non-list ضمیمه: glossary → synonym map; welcome → canned; offtopic → rail; limitations → skipped
- Orchestrator: `trim_supplemental` ranks lists by **query relevance** (top-3, ~8 KB/list), `build_context` ranks Entity Data into a ~9000-char budget, `format_response` recovers from Entity Data when the LLM abstains or bleeds a score table

### Orchestrator (`Work_Credit-RAG_Phase1/components/orchestrator/`)

**LangGraph state machine:**
1. `validate_input` — input guardrails check
2. `classify` (+ optional `hybrid_classify`) — greeting / specification / question
3. `retrieve` — KB search + proactive entity tool call + trim
4. `build_context` — assemble system message with context + ranked entity data
5. `guarded_generate` → Guardrails `:8200/v1/chat/completions`
6. `format_response` — citations + audit + Entity Data recovery

**Tracing:** Direct HTTP to Langfuse v2 (`LANGFUSE_HOST`), fallback JSONL collector at `:3000`.

### Guardrails (`Work_Credit-RAG_Phase1/components/guardrails/`)

**Pipeline:**
1. Deterministic signals (injection, jailbreak, HurtLex, profanity, scope)
2. Policy / LLM / Mizan judge (NeMo removed; `enable_thinking:false` on Gemma)
3. PII mask-and-continue vs hard block
4. Fail-closed on timeout/error

**Upstream LLM:** `UPSTREAM_LLM_BASE_URL=http://127.0.0.1:18000/v1` (Gemma) or `:9000` (Manager).

---

## 🧠 Context window management

The Gemma backend has an **8192-token context window**. To stay within it:

- Entity-list payloads trimmed to **top-3 lists by query relevance**, **~8 KB each**, bulky dumps dropped, rows query-token filtered
- RAG context capped at **~9000 chars** (`MAX_CONTEXT`) with `MAX_CHUNKS=20`
- Total prompt (system + context + entity data) target: **~20K chars (~7K tokens)**
- If upstream returns 400 (`max_seq_len` overflow), guardrails returns `finish_reason="length"` gracefully

---

## 🌍 Environment parity

| | Production | Test / Integration |
|---|---|---|
| Branch | `main` / `master` | `develop` |
| Deploy | `bash deploy/vast/start.sh` | Same stack via `deploy/vast/` |
| Model cache | `HF_HOME` + `HF_HUB_CACHE` = same dir | Same |
| `HF_HUB_OFFLINE` | `1` | `1` |

**Startup order:** postgres → Gemma → KB → guardrails → collector → orchestrator → WebUI
(`bash deploy/vast/start.sh`).
