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

Prior to this PR, there was no separate configuration route for embedding in `MetaGPT`'s `RAG` pipeline beyond the overall `llm` configuration block, which also controlled chat and completions. As a result, the embedding provider and the completion provider had to be the same, or have a matching API type, i.e. `OpenAI` or `Azure`. If you were running `Ollama` locally or using `Gemini`, you'd have to manually provide an embed model object at runtime, and there was no way to clearly state "use this particular provider for embeddings" without creating confusion in relation to your other `LLM` configuration settings.

This PR introduces a separate embedding section in the configuration. A new `EmbeddingConfig` class will hold all fields used to configure an embedding: API type, key, base URL, version, model name, and batch size. This configuration will be located next to the current `LLM` configuration in `config2.py`; they will not be integrated into one. The embedding factory, which is responsible for building the embedding object, will read from the embedding configuration first.

There are four `LlamaIndex` embedding providers supported directly by `LlamaIndex`: `OpenAI`, `Azure`, `Gemini`, and `Ollama`. `LlamaIndex` will allow you to use either provider's implementation of its corresponding embedding class. `LlamaIndex` maintains backward compatibility by defaulting to the old behaviour if no embedding config has been specified and the main `LLM` type is either `OpenAI` or `Azure`, therefore not disrupting existing setups. If neither of these two conditions is met, then `LlamaIndex` will return a detailed error message so that you know why the system cannot proceed.

Also, a minor change was made to the way `FAISS` handles vector dimensions. The prior hardcoded value of `1536` was only correct for the `OpenAI` `text-embedding-ada-002` embedding model but incorrect for both `Gemini` (`768`) and `Ollama` (`4096`). This makes it so that if you use any `FAISS` embedding backend but do not provide a dimension value to the index create call, the system will default to using `0` as the dimension, and will automatically set the correct dimension based on the embedding provider specified in the configuration, as opposed to causing an index failure at runtime due to the dimensions used being inconsistent with what is expected by the `FAISS` library.

Finally, the token counter has been updated to include the `gpt-4-turbo` embedding model so that cost estimation and enforcement of the context window will work properly for users of that model.

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

**Edge Case 4 — `embed_batch_size` set beyond the provider's limits**
`EmbeddingConfig` includes an `embed_batch_size` field. Setting a very high batch size (say, 2048) with a provider that has strict rate limits or per-request size caps — Gemini in particular has input limits — will trigger API errors mid-pipeline. The embedding factory currently has no guard for this. At minimum, there should be sensible per-provider defaults for batch size, along with documentation on the limits for each provider.

---

## 3.1.5 Initial Prompt

```
You are working on the MetaGPT open-source repository — a multi-agent LLM framework that
simulates a software development team. Your task is to implement the changes described in
PR #1172: making the RAG embedding configuration independent from the main LLM config,
and adding gpt-4-turbo support to the token counter.

--- CONTEXT ---

MetaGPT has a RAG module (under metagpt/rag/) that allows agents to retrieve relevant chunks
from documents before generating answers. The retrieval step requires converting text into
vector embeddings. Currently, the embedding provider is inferred from the main `llm` config
block in config2.py — the same block that controls which model handles chat completions.
This makes it impossible to cleanly use a different provider for embeddings versus completions
(e.g., Ollama for embeddings + GPT-4 for answers), which is a common production setup.

--- WHAT YOU NEED TO IMPLEMENT ---

1. CREATE metagpt/configs/embedding_config.py
   Define two new objects:
   - `EmbeddingType`: an enum with values for OPENAI, AZURE, GEMINI, OLLAMA
   - `EmbeddingConfig`: a Pydantic model with fields: api_type (EmbeddingType, optional),
     api_key (str, optional), base_url (str, optional), api_version (str, optional),
     model (str, optional), embed_batch_size (int, optional with a sensible default)

2. MODIFY metagpt/config2.py
   Add `embedding: EmbeddingConfig = EmbeddingConfig()` as a top-level field so users can
   declare their embedding provider separately from `llm`.

3. REFACTOR metagpt/rag/factories/embedding.py
   The embedding factory must now read from `config.embedding` first.
   - If `embedding.api_type` is OPENAI → return LlamaIndex OpenAIEmbedding initialised
     with fields from EmbeddingConfig
   - If AZURE → return AzureOpenAIEmbedding with api_key, base_url, api_version from
     EmbeddingConfig
   - If GEMINI → return GeminiEmbedding
   - If OLLAMA → return OllamaEmbedding using base_url from EmbeddingConfig
   - If no embedding config is set AND llm config is OpenAI or Azure → fall back to
     existing LLM-config-based behavior (backward compatibility)
   - If no embedding config is set AND llm type is neither OpenAI nor Azure → raise a
     descriptive ValueError telling the user to configure the `embedding` block in config2.yaml

4. MODIFY metagpt/rag/schema.py
   Change the FAISS `dimensions` default from 1536 to 0. Then auto-populate it based on
   the configured embedding type:
   - GEMINI → 768
   - OLLAMA → 4096
   - All others (including OpenAI/Azure fallback) → 1536
   This logic should trigger when dimensions is 0 at index build time.

5. MODIFY setup.py
   Add the following to the `rag` extras_require list:
   - llama-index-embeddings-gemini
   - llama-index-embeddings-ollama

6. MODIFY metagpt/utils/token_counter.py
   Add gpt-4-turbo to the model pricing and token limit table. Use the correct values:
   context window = 128,000 tokens, pricing consistent with OpenAI's published rates
   for gpt-4-turbo at time of this PR.

7. UPDATE config/config2.example.yaml
   Add a commented-out `embedding:` section showing all configurable fields, with a note
   explaining that if this section is omitted, the system falls back to OpenAI/Azure from
   the main llm config.

8. UPDATE tests/metagpt/rag/factories/test_embedding.py
   Add test cases for:
   - EmbeddingConfig with each of the four api_types producing the correct LlamaIndex class
   - Fallback path when embedding config is empty and llm is OpenAI
   - Fallback path when embedding config is empty and llm is Azure
   - Error path when embedding config is empty and llm is a non-OpenAI/Azure type

--- ACCEPTANCE CRITERIA TO SATISFY ---

- Setting api_type: gemini in the embedding config must use GeminiEmbedding and set FAISS
  dimensions to 768 automatically
- Setting api_type: ollama must use OllamaEmbedding and set FAISS dimensions to 4096
- Omitting the embedding config with an OpenAI LLM must not break existing behavior
- Omitting the embedding config with an unsupported LLM type must raise a clear ValueError
- gpt-4-turbo in the token counter must not raise a KeyError and must return correct limits
- All existing RAG tests must continue to pass

--- EDGE CASES TO HANDLE ---

- If EmbeddingConfig has api_type set but api_key is missing, raise a config validation
  error at startup — do not let it fail silently at query time
- If FAISS index is loaded from disk and the dimension of the loaded index does not match
  the auto-filled dimension from the current embedding config, surface a clear error
  message explaining the provider mismatch rather than an internal FAISS assertion
- If api_type is OLLAMA and the connection to base_url fails, catch the connection error
  and re-raise with a message pointing to the embedding config and suggesting the user
  verify their Ollama service is running

--- TESTING REQUIREMENTS ---

Write unit tests for the embedding factory covering all four provider branches, both
fallback paths, and the error path. Tests should mock the LlamaIndex embedding classes
so they do not require live API credentials. Confirm that FAISSRetrieverConfig correctly
auto-fills dimensions for Gemini and Ollama without user input.
```

## Integrity Declaration

I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.