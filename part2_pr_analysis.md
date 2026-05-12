# Part 2: Pull Request Analysis

## Task 2.1: PR Selection and Comprehension

Repository selected: [FoundationAgents/MetaGPT](https://github.com/FoundationAgents/MetaGPT)

I reviewed all 10 PRs and selected the following 2 PRs that i could best comprehend:

- [PR #1172: Make RAG embedding configurable and add gpt-4-turbo in token_counter](https://github.com/FoundationAgents/MetaGPT/pull/1172)
- [PR #1457: Integrated Milvus with MetaGPT](https://github.com/FoundationAgents/MetaGPT/pull/1457)

### PR #1172: Make RAG Embedding Configurable and Add GPT-4-Turbo in Token Counter

#### PR Summary

This PR makes MetaGPT’s RAG embedding setup more flexible. Previously, the embedding provider was mostly tied to the main `llm` configuration (mainly OpenAI or Azure via `llm_config`), which made it awkward if you wanted to generate embeddings using a different provider than the one used for chat/completions. 

Now there is a separate section for configuring RAG embeddings, which allows you to use whatever embedding provider you wish (e.g. OpenAI, Azure, Gemini or Ollama) without having to fit it into an already existing LLM model configuration. Additionally, updating the token count to handle `gpt-4-turbo` costs and token counts correctly will be included.

Overall separating the models used in RAG will enable greater flexibility and scalability on RAG pipelines.

---

#### Technical Changes

- Added a new `embedding` section in **`config/config2.example.yaml`** (with a note about backward compatibility if it isn’t set).
- Introduced `EmbeddingConfig` and `EmbeddingType` in **`metagpt/configs/embedding_config.py`**.
- Added `embedding: EmbeddingConfig` as a top-level field in **`metagpt/config2.py`**.
- Refactored **`metagpt/rag/factories/embedding.py`** so embeddings are created using the new embedding config:
  - Supports OpenAI, Azure, Gemini, and Ollama embeddings via LlamaIndex embedding classes.
  - Keeps backward compatibility: if embedding config isn’t provided and the LLM API type is OpenAI/Azure, it falls back to the existing LLM config.
  - Raises a clear error if embeddings aren’t configured and the LLM type isn’t one of those compatible fallback types.
- Updated FAISS dimension handling in **`metagpt/rag/schema.py`**:
  - Changed FAISS `dimensions` default from `1536` to `0`, then auto-fills based on the configured embedding type:
    - Gemini → `768`
    - Ollama → `4096`
    - Otherwise defaults to `1536`
- Added required dependencies in **`setup.py`**:
  - `llama-index-embeddings-gemini`
  - `llama-index-embeddings-ollama`
  - (Also added `docx2txt==0.8`)
- Updated `gpt-4-turbo` pricing + token limit support in **`metagpt/utils/token_counter.py`**.
- Updated and expanded tests in **`tests/metagpt/rag/factories/test_embedding.py`** to cover the new embedding config behavior.
- Minor RAG-related updates:
  - **`examples/rag_pipeline.py`**: simplified the example to use FAISS only (removed BM25 in that example).
  - **`metagpt/rag/factories/llm.py`** and **`metagpt/utils/async_helper.py`**: adjusted how async execution is handled using a `NestAsyncio.apply_once()` helper.

---

#### Implementation Approach

They added a unique path to configure "how do we create embeddings". Currently, it is being inferred from the existing model configurations from the LLM (Large Language Models). 

In addition to adding an `EmbeddingConfig` Model (with `api_type`, `api_key`, `base_url`, `api_version`, `model`, and `embed_batch_size`), the embedding factory will utilize the new embedding config to create the right LlamaIndex embedding implementation. When `embedding.api_type` is not specified, it attempts to preserve current behaviour by defaulting to the OpenAI/Azure configurations specified in the `llm`, allowing older configs to be kept working.

In addition, FAISS indexing becomes safer across different embedding providers by automatically choosing the correct vector dimension when it’s not explicitly set.

---

#### Potential Impact

The main impact of this update will be on MetaGPT's RAG pipeline and those using embedding-based retrieval methods. The update makes it easy to use multiple providers, such as Ollama or Gemini embeddings with an OpenAI Chat model, without needing to modify the main `llm` configuration file.

Existing users using the default OpenAI or Azure configuration will largely remain unaffected because the fallback behavior is unchanged. However, if you are using a non-OpenAI or Azure LLM and have not setup the embeddings manually, the system now has a new option in `config2.yaml` that will prompt you to specify the `embedding`.

Due to `gpt-4-turbo` being recognized directly by the token counter, Token and Cost tracking has also improved slightly.

### PR #1457: Integrated Milvus with MetaGPT

#### PR Summary

This PR add **Milvus** as an extra vector database option in RAG for MetaGPT; currently RAG supports vector stores such as FAISS/Chroma/Elasticsearch and now with this update allows customers who have a preference on milvus (mainly for larger and production retrievals) to integrate milvus into the same factory/config based RAG workflow as the other vector stores. All Milvus specific configuration types have been created, wired into the existing index/retriever factories, and have a lightweight wrapper for the milvus document store added. The handling of dependencies is intentionally conservative - Milvus specific dependencies are kept optional/lazy-loaded so as to avoid the potential problems of version conflicts identified in the discussion of this PR.

#### Technical Changes

- Added **`metagpt/document_store/milvus_store.py`** (new file), introducing:
  - `MilvusConnection`
  - `MilvusStore` (a small wrapper around `pymilvus`, imported lazily)
- Updated **`metagpt/rag/factories/index.py`** to support building Milvus-backed vector indexes via LlamaIndex’s `MilvusVectorStore`.
- Updated **`metagpt/rag/factories/retriever.py`** to build and return Milvus retrievers as part of the normal retriever factory flow.
- Added **`metagpt/rag/retrievers/milvus_retriever.py`** (new file), implementing a Milvus retriever class on top of LlamaIndex’s `VectorIndexRetriever`.
- Extended the RAG schema in **`metagpt/rag/schema.py`** by adding:
  - `MilvusRetrieverConfig`
  - `MilvusIndexConfig`
  - dimension handling logic so vector size can be derived from embedding settings when not explicitly set
- Updated dependency notes in:
  - **`requirements.txt`** (Milvus dependency is present but commented)
  - **`setup.py`** (Milvus/LlamaIndex vector store dependency is also present but commented)
- Added Milvus-related tests in:
  - `tests/metagpt/document_store/test_milvus_store.py` (skipped by default because Milvus deps aren’t installed)
  - `tests/metagpt/rag/factories/test_index.py`
  - `tests/metagpt/rag/factories/test_retriever.py`

#### Implementation Approach

The PR follows the existing design of MetaGPT’s RAG module rather than creating a separate Milvus-only pathway.

1. **Milvus document store wrapper (optional)**
   A `MilvusStore` helper is added as a wrapper around `pymilvus`. Importing `pymilvus` is delayed until runtime so the project doesn’t require Milvus dependencies unless someone actually uses that functionality.

2. **Schema + configuration support**
   New config models (`MilvusRetrieverConfig` and `MilvusIndexConfig`) are added to the RAG schema. These hold key parameters like `uri`, `collection_name`, `token`, optional metadata, and vector dimensions. Dimensions are handled carefully because different embedding providers can produce different vector sizes.

3. **Factory integration**
   - The index factory is extended so it can create a Milvus-backed index using **LlamaIndex’s `MilvusVectorStore`**.
   - The retriever factory is extended so it can build a Milvus-backed index and return a `MilvusRetriever` in the same way it already does for other backends.

4. **Retriever behavior**
   `MilvusRetriever` extends LlamaIndex’s `VectorIndexRetriever`, adds support for inserting nodes, and treats persistence as automatic (since Milvus manages storage itself).

#### Potential Impact

The **RAG stack** in the MetaGPT system has been updated by adding Milvus as an additional backend option for supporting scalable semantic search and retrieval workflows.

The primary practical risk of this change is related to managing your **dependencies**. The PR intentionally leaves the Milvus-related dependencies as optional because those may conflict with any currently installed version (pinned) of those dependencies. In addition, because the path to Milvus also includes integration of LlamaIndex's Milvus Vector Store, users may also need to install the LlamaIndex version of the Milvus package in addition to `pymilvus`, depending on how they intend to use that functionality.

## Integrity Declaration

I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.