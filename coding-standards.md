# Coding Standards & Rules

Generated from the actual codebase patterns in `Work_RAG-KB`, `Work_Credit-RAG_Phase1`,
and `Work_RAG-Server-Setup`, plus general Python/FastAPI best practices.

---

## 1. Python style

| Rule | Detail |
|---|---|
| **Type hints** | All public functions must have type hints. Use `typing` module (`List`, `Dict`, `Optional`, `Any`, `Annotated`). |
| **Docstrings** | Every public function/class needs a docstring. Follow the pattern in `kb_manager/pipeline/orchestrator.py` (Args/Returns sections). |
| **Async** | Use `async def` for I/O-bound operations (DB, HTTP). Do NOT mix `asyncio.to_thread()` with asyncpg — thread-hop breaks the pool ("attached to a different loop"). |
| **Error handling** | Wrap external calls (HTTP, DB, model) in `try/except`. Log with `log.warning()`/`log.error()`. Fail closed on guardrails. |
| **Logging** | Use module-level `logger = logging.getLogger(__name__)`. Do not `print()`. |
| **No comments on obvious code** | The codebase convention: comments only for *why*, not *what*. |
| **Imports** | Standard library → third-party → local. No wildcard imports. |

---

## 2. FastAPI patterns

| Rule | Detail |
|---|---|
| **Pydantic models** | Use `BaseModel` for request/response bodies. Validate all inputs. |
| **Factory pattern** | Use `create_app --factory` (see `work_rag_guardrails.api`, `work_rag_orchestrator.api`). |
| **Endpoints** | RESTful nouns. `/v1/chat/completions`, `/v1/models`, `/health`, `/ready`. |
| **Timeouts** | `httpx.AsyncClient(timeout=120, trust_env=False)` — bypass corporate proxy for localhost. |
| **Status codes** | Return proper HTTP status. Guardrails catches upstream errors and returns `finish_reason="length"` (valid Literal) instead of crashing with `"error"`. |

---

## 3. Database patterns

| Rule | Detail |
|---|---|
| **Async SQLAlchemy** | Use `async_sessionmaker`, `AsyncSession`. Always `async with db.session() as session:`. |
| **pgvector** | HNSW index for chunk embeddings. `1 - (embedding <=> CAST(:q AS vector))` for cosine. |
| **Entity lists** | `EntityList` table: `list_name`, `description`, `description_embedding` (JSON), `content_json`. Always populate `description_embedding` via `SentenceTransformerEmbedder.embed_texts()`. |
| **Migrations** | None automated — schema changes via manual SQL + code updates. |

---

## 4. Model serving patterns

| Rule | Detail |
|---|---|
| **Gemma** | Must use `chat_template_kwargs:{"enable_thinking":false}` + `--no-mmproj --jinja` or `<unused*>` tokens leak. |
| **Context window** | Triton Gemma: 8192 tokens. Keep prompts under ~20K chars (~7K tokens) + 1024 max_tokens. |
| **VRAM** | 22.7/24 GB at ctx 8192. OOM under load → restart llama-server with `--ctx-size 4096`. |
| **Proxy bypass** | `trust_env=False` on httpx clients for localhost calls. Without it, Squid proxy (`192.168.203.2:3128`) causes hangs. |

---

## 5. Persian language handling

| Rule | Detail |
|---|---|
| **Normalization** | Use `preprocessor/persian.py`, `regex_persian.py` for Persian text normalization. |
| **Synonyms** | 93-entry map in `query_expansion.py` (colloquial → formal). |
| **Tokenizer** | tantivy BM25 with Persian word + 3-gram stream. |
| **Embedding model** | `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` (384d). |

---

## 6. Git rules (from AGENTS.md)

| Rule | Detail |
|---|---|
| **Submodule pins** | Parent pins submodules to exact commits. Check: `git submodule status --recursive`. |
| **Component fix** → commit on component's branch → advance parent gitlink |
| **Deployment-only** (`deploy/`, `docs/`, `eval/`) → parent `main` |
| **Never force-push** |
| **Never commit secrets** (`*.env`, tokens, passwords) |
| **Never commit generated artifacts** (`*.npz`, `__pycache__`, `*.log`, venvs) |

---

## 7. Vast.ai / deployment gotchas

| Gotcha | Fix |
|---|---|
| No Docker (unprivileged container) | Host venvs only |
| NAT ports fixed at creation | SSH tunnels for public access |
| Phantom overlay dentries | `/workspace/.hf_home` looks populated but is empty; real cache is `/tmp/hf_clean` |
| HF cache | Set BOTH `HF_HOME` and `HF_HUB_CACHE` to same dir |
| asyncpg loop bug | `search_api` must `await` directly, NOT via `asyncio.to_thread()` |
| Langfuse v2 | Top-level `id` + ISO `timestamp`; updates via same-id `trace-create` (no `trace-update`) |
| Gemma thinking leak | `--no-mmproj --jinja` + `enable_thinking:false` |

---

## 8. Never do

1. Never `pkill python` — kills everything. Use `bash deploy/vast/stop.sh`.
2. Never launch gemma-manager or download another GGUF (Gemma is external `:18000`).
3. Never expose `:18000`, postgres, or redis publicly.
4. Never commit `.env` / secrets / `*.npz` / logs / venvs.
5. Never force-push any repo.
6. Never run Docker on this unprivileged host.
7. Never set only one of `HF_HOME`/`HF_HUB_CACHE`.
8. Never omit `request_id` in Studio run inputs.
