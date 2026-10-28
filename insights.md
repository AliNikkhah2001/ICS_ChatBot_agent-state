# Key Insights & Decisions

Generated from the actual codebase, READMEs, and debugging sessions. These are the
lessons an agent needs to know before making changes.

---

## 🔑 Key decisions

| # | Decision | Rationale | Date |
|---|---|---|---|
| 1 | `dense._ensure_model()` instead of `dense.model` | `DenseSemanticIndex` has no public `.model` attribute. All `hasattr(dense, "model")` guards were always False, silently disabling embedding in step-8 entity injection and `/search/supplement`. | 2026-10-27 |
| 2 | Entity lists embedded via `embed_texts(title + desc)` | All 10 reason-code variants share identical generic descriptions. The `list_name` (title) is what distinguishes them. | 2026-10-27 |
| 3 | `trim_supplemental()` + 4KB total budget in `build_context` | Triton Gemma has 8192-token context. 11 full entity lists = 663KB caused HTTP 500 context overflow (Triton 400: `default_max_tokens` negative). | 2026-10-27 |
| 4 | Guardrails `finish_reason="length"` not `"error"` | Pydantic `ChatCompletionChoice.finish_reason` is a Literal of `stop`/`length`/`content_filter`. `"error"` crashed validation and masked the real upstream 400 context-overflow. | 2026-10-27 |
| 5 | `should_clarify` short-circuits on entity evidence | When the supplement tool matches a dictionary (reason code, shareholder, etc.), clarify is counterproductive — the model can answer from entity rows directly. | 2026-10-27 |
| 6 | Code-token fallback in supplement search | Cosine for queries like "MTX1" or "SBD1" is diluted by stopwords (~0.35 < 0.4 threshold). Mixed alphanumeric tokens appearing in list content guarantee a match regardless of cosine. | 2026-10-27 |

---

## 📚 Lessons learned

- **Never assume `hasattr(dense, "model")` works.** Always use `dense._ensure_model()` or `dense._encode()` on `DenseSemanticIndex`. Checked in `kb_manager/web/routes/search.py` at 5 sites.
- **Entity lists need explicit embeddings.** The pipeline route constructed `PipelineOrchestrator(database=db, chunker=chunker)` without an embedder, so all 22 entity lists had `description_embedding=None`. Fixed by passing `SentenceTransformerEmbedder()`.
- **Context window is the bottleneck.** The 8192-token Triton limit caused silent 500s. Fixed by trimming entity payloads and capping `build_context` budget at 4000 chars.
- **Guardrails swallows upstream errors.** A bare HTTP 500 hid the real Triton 400 until we reproduced in-process with the exact `prompt_messages`.
- **Persian query path matters.** The KB uses `preprocessor/persian.py` + `regex_persian.py` for normalization, plus a 93-entry synonym map in `query_expansion.py`. Ignoring this breaks retrieval for Persian queries.

---

## ❓ Open questions

- Should `develop` be protected with required PR reviews on GitHub? (Recommended: yes)
- Is `vast-gemma4-migration` still needed, or should changes merge directly to `master`/`main`? (Branch deleted per user decision)
- Should LiteLLM be adopted as the unified gateway, replacing per-app provider config? (See AGENTS.md section 5)

---

## 📋 Recently archived

- `history/archive/todo_deepseek_v4_paused_20260824_1211.md` — paused DeepSeek-V4 mission, pending user decision
- `history/archive/audit-fix-2026-09-27.md` — audit fix session
