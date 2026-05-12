# Part 3: Prompt Preparation Document

**Selected PR:** #1172 — Make RAG Embedding Configurable and Add GPT-4-Turbo in Token Counter
**PR Link:** https://github.com/FoundationAgents/MetaGPT/pull/1172

---

## 3.1.1 Repository Context

`MetaGPT` is an agent-based system that can take a task described in plain English, pass that task description through the agents, and create an extensive list of documents that describe how to construct that task. The main concept of this system is that the outcome of collaborating with an agent whose input is only one part of the task will be more productive than just providing a single prompt to one agent.

`MetaGPT` has two modules: the first module is the collaborative module, and the second module is a `RAG` (`Retrieval-Augmented Generation`) module, which uses significant documentation to base responses. Each agent will convert all documentation into `vectors` — numeric representations based on capturing the meaning of the document — enabling the system to find the best matching document, which will then be used as input to the model to produce a response.

Most of the people building with `MetaGPT` are developers and researchers constructing agent-based applications or exploring what multi-agent `Large Language Models` can actually do. Many of them run mixed-provider setups: a cheap local model via `Ollama` for embeddings paired with `GPT-4` for the reasoning-heavy steps.

It addresses the problem faced in software development automation. Normally, turning an idea into a working project involves many steps: understanding requirements, planning the architecture, writing code, and testing the result. `MetaGPT` tries to organize that process by giving different `AI agents` different responsibilities. The `RAG` module supports this by helping those agents use external documents when the answer cannot come only from the model’s built-in knowledge.

---

## 3.1.2 Pull Request Description

Prior to this PR, `MetaGPT`'s RAG pipeline had no separate configuration route for embeddings. Everything ran through the same `llm` config block that also handled chat and completions. This meant the embedding provider and the completion provider had to match, both had to be `OpenAI` or `Azure`. If you were using `Ollama` locally or `Gemini`, you had to manually pass an embed model object at runtime. There was no clean way to say "use this provider for embeddings" without conflicting with your existing LLM settings.

This PR introduces a dedicated `EmbeddingConfig` class that holds all embedding-related fields: API type, key, base URL, version, model name, and batch size. This config sits alongside the existing `LLM` config in `config2.py` — completely separate, not merged in. The embedding factory now reads from `EmbeddingConfig` first. Four providers are supported out of the box:  OpenAI`, `Azure`, `Gemini`, and `Ollama`. Backward compatibility is maintained, if no embedding config is set and the main LLM is `OpenAI` or `Azure`, the old behaviour kicks in  automatically. Otherwise, a clear error is thrown explaining why the system cannot proceed.

A fix was also made to how `FAISS` handles vector dimensions. The hardcoded value of `1536` only worked for OpenAI's `text-embedding-ada-002`. It was wrong for `Gemini` (`768`) and `Ollama` (`4096`). Now, if no dimension is provided, the system defaults to `0` and auto-fills the correct value based on the active embedding provider, preventing silent index failures at runtime.

Lastly, `gpt-4-turbo` was added to the token counter so cost estimation and context window enforcement work correctly for users of that model.

---

## 3.1.3 Acceptance Criteria

- When an `embedding` section is present in `config2.yaml` with `api_type: openai`, the RAG engine must use `OpenAIEmbedding` from LlamaIndex — not fall back to the `llm` config.

- When `api_type: gemini` is set in the embedding config, the system must use `GeminiEmbedding` and create the FAISS index with `dimensions = 768` without the user having to set it manually.

- When `api_type: ollama` is set in the embedding config, the system must use `OllamaEmbedding` and default the FAISS index to `dimensions = 4096`.

- When no `embedding` config is provided and the main `llm` config is OpenAI or Azure, the existing behavior must be preserved — graceful fallback, no error raised.

- When no `embedding` config is provided and the main `llm` config is neither OpenAI nor Azure (e.g. a local model), the system must raise a clear, descriptive error that tells the user to set the `embedding` block in their config — not crash with an internal exception.

- When `api_type: azure` is set in the embedding config, the system must use `AzureOpenAIEmbedding` and must correctly read `api_key`, `base_url`, and `api_version` from the embedding config — not from the LLM config.

- `gpt-4-turbo` must appear in the token counter's pricing and context window table, and calling the token counter with that model string must return the correct token limit and cost per token without raising a `KeyError` or falling through to a wrong default.

- All existing RAG tests must continue to pass — no regressions in any retriever, ranker, or storage functionality as a side effect of the embedding config refactoring.

- The updated tests in `tests/metagpt/rag/factories/test_embedding.py` must cover at minimum: OpenAI embedding from dedicated config, Azure embedding from dedicated config, Gemini embedding from dedicated config, Ollama embedding from dedicated config, OpenAI fallback from `llm` config, and the error case for unsupported LLM type with no embedding config.

---

## 3.1.4 Edge Cases

**Edge Case 1 — Partial embedding config with missing fields**
A user sets `api_type: openai` in the embedding config but forgets to include `api_key`. The system shouldn't silently fall back to the LLM config's API key without saying anything, and it shouldn't crash with a cryptic authentication error from OpenAI. The right behaviour is either to raise a clear config validation error at startup (before any network call goes out), or to fall back deliberately and log a warning. Silently inheriting credentials from the `llm` config could mask a real misconfiguration in production.

**Edge Case 2 — FAISS index loaded from disk was built with a different embedding provider**
Suppose a user created and persisted a FAISS index while using OpenAI embeddings (dimension 1536), then later switches their embedding config to Gemini (dimension 768) and tries to load the old index. The dimension mismatch will cause FAISS to throw an internal assertion error. The auto-fill logic in this PR helps catch mismatches at build time, but it does nothing for this load-from-disk scenario. The fix needs either a metadata file stored alongside the index that records which embedding config was used, or at minimum a clear error message explaining what went wrong.

**Edge Case 3 — Ollama server not running when embedding is requested**
Unlike OpenAI or Gemini, Ollama runs locally. If a user sets `api_type: ollama` but the Ollama server isn't up at the configured `base_url`, the failure will happen at query time, not at config load time — likely surfacing as a connection refused exception deep inside the LlamaIndex Ollama class. This should be caught and re-raised with a message that points back to the embedding config and suggests checking whether the Ollama service is actually running, rather than propagating a raw socket error.

---

## 3.1.5 Initial Prompt

```
You are working on MetaGPT, a multi-agent LLM framework that simulates a software development team. Your task is to implement PR #1172: making the RAG embedding configuration independent from the main LLM config, and adding `gpt-4-turbo` to the token counter.

--- CONTEXT ---

MetaGPT's RAG module (`metagpt/rag/`) lets agents retrieve relevant document chunks before generating answers. Currently, the embedding provider is pulled from the same `llm` block in `config2.py` that controls chat completions. This means you cannot cleanly use different providers for embeddings and completions, a common real-world need.

--- WHAT YOU NEED TO IMPLEMENT ---

1. CREATE `metagpt/configs/embedding_config.py`
   Define `EmbeddingType` (enum: OPENAI, AZURE, GEMINI, OLLAMA) and `EmbeddingConfig` (Pydantic model with optional fields: `api_type`, `api_key`, `base_url`, `api_version`, `model`, `embed_batch_size`)

2. MODIFY `metagpt/config2.py`
   Add `embedding: EmbeddingConfig = EmbeddingConfig()` as a top-level field, separate from the existing `llm` block.

3. REFACTOR `metagpt/rag/factories/embedding.py`
   Read from `config.embedding` first. Route each `api_type` to its correct LlamaIndex class. If no embedding config is set and `llm` is OpenAI or Azure, fall back to existing behavior. If `llm` is any other type, raise a descriptive `ValueError`.

4. MODIFY `metagpt/rag/schema.py`
   Change FAISS `dimensions` default from `1536` to `0`. Auto-fill at index build time: GEMINI → 768, OLLAMA → 4096, all others → 1536.

5. MODIFY `setup.py`
   Add `llama-index-embeddings-gemini` and `llama-index-embeddings-ollama` to the `rag` extras list.

6. MODIFY `metagpt/utils/token_counter.py`
   Add `gpt-4-turbo` with a 128,000 token context window and correct pricing.

7. UPDATE `config/config2.example.yaml`
   Add a commented-out `embedding:` section with all configurable fields and a note that omitting it falls back to the main `llm` config.

8. UPDATE `tests/metagpt/rag/factories/test_embedding.py`
   Add tests for all four provider branches, both fallback paths, and the error path when `llm` is an unsupported type.

--- ACCEPTANCE CRITERIA ---
- `api_type: openai` explicitly set must use `OpenAIEmbedding` — not fall back to the `llm` config
- `api_type: gemini` must use `GeminiEmbedding` and auto-set FAISS dims to 768
- `api_type: ollama` must use `OllamaEmbedding` and auto-set FAISS dims to 4096
- `api_type: azure` must use `AzureOpenAIEmbedding` and read `api_key`, `base_url`, `api_version` from embedding config — not from `llm` config
- No embedding config + OpenAI/Azure LLM must not break existing behaviour
- No embedding config + unsupported LLM must raise a clear `ValueError`
- `gpt-4-turbo` must return correct token limits without a `KeyError`
- All existing RAG tests must continue to pass
- `test_embedding.py` must cover all four provider branches, both fallback paths, and the error case for unsupported LLM type

--- EDGE CASES ---
- `api_type` set but `api_key` missing → raise a validation error at startup, not silently at query time
- FAISS index loaded from disk with mismatched dimensions → raise a clear error explaining the provider mismatch before any query runs
- OLLAMA `base_url` unreachable → catch and re-raise with a message pointing to the embedding config and asking the user to verify Ollama is running
- `embed_batch_size` set beyond provider limits → do not crash mid-pipeline; use sensible per-provider defaults and document the limits clearly

--- TESTING REQUIREMENTS ---

Mock all LlamaIndex classes — no live API credentials required. Cover all four provider branches, both fallback paths, and the error path. Confirm `FAISSRetrieverConfig` auto-fills dimensions for Gemini and Ollama correctly.
```

## Integrity Declaration

I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.
