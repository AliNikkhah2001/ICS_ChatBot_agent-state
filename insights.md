# Key Insights & Decisions

This file records architectural decisions, lessons learned, and open questions
so that agents resuming on another machine have full context.

## Decisions

| # | Decision | Rationale | Date |
|---|---|---|---|
| 1 | `dense.model` → `dense._ensure_model()` | `DenseSemanticIndex` has no public `.model` attr; all `hasattr` guards were False, silently disabling embedding in step-8 injection and `/search/supplement` | 2026-10-27 |
| 2 | Entity lists embedded via `embed_texts(title + desc)` | All 10 reason-code variants share identical generic descriptions; the title is what distinguishes them | 2026-10-27 |
| 3 | `trim_supplemental()` + 4KB total budget in build_context | Triton Gemma has 8192-token context; 11 full lists = 663KB caused HTTP 500 context overflow | 2026-10-27 |
| 4 | Guardrails `finish_reason="length"` not `"error"` | Pydantic `ChatCompletionChoice.finish_reason` is a Literal of stop/length/content_filter; `"error"` crashed validation and masked the real upstream 400 | 2026-10-27 |
| 5 | `should_clarify` short-circuits on entity evidence | When the supplement tool matches a dictionary, clarify is counterproductive; model can answer from entity rows | 2026-10-27 |
| 6 | Code-token fallback in supplement search | Cosine for "MTX1"/"SBD1" is diluted by stopwords (~0.35 < 0.4 threshold); mixed alphanumeric tokens in content guarantee a match | 2026-10-27 |

## Lessons learned

- **Never assume `hasattr(dense, "model")` works.** Always use `_ensure_model()` or `_encode()` on `DenseSemanticIndex`.
- **Entity lists need explicit embeddings.** The pipeline route didn't pass an embedder, so all 22 lists had `description_embedding=None`.
- **Context window is the bottleneck.** The 8192-token Triton limit caused silent 500s; trimming fixed it.
- **Guardrails swallows upstream errors.** A bare 500 hid the real Triton 400 until we reproduced in-process.

## Open questions

- Should `develop` be protected with required PR reviews on GitHub?
- Is `vast-gemma4-migration` the right long-term branch, or should changes merge to `master`/`main`?
- Should LiteLLM be adopted as the unified gateway, replacing per-app provider config?

## Recently archived

- `archive/todo_deepseek_v4_paused_20260824_1211.md` — paused DeepSeek-V4 mission, pending user decision
