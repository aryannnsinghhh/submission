# Part 4: Technical Communication

I chose PR #1172 because I've personally built something very similar, a local RAG engine with a vector database, document embedding pipeline, and search powered by Ollama. Having worked with embeddings and vector search, I understood what this PR was doing (giving embeddings their own config instead of borrowing it from the LLM). While I had not previously experienced this issue directly, it was close enough to my area of expertise.

Having already worked with FastAPI and Pydantic models, I was able to easily relate to this PR as it is formatted similarly to how I format my config classes (using Pydantic to validate). I also noticed the hard-coded 1536 dimenison value very quickly because I ran into that exact same problem when I switched embedding providers in my RAG project. Therefore, the process of reviewing the PR was fairly simple for me.

The new indexes in PR work perfectly well. All dimensions of the embedding will come from the provider that was used at the time of creating the Index. However, if you have created and saved the FAISS Indexes prior to even the creation of the new Index during the migration process, you may experience issues with the old FAISS Indexes. For example, if you created your FAISS Index in OpenAI where the dimensions are 1536 and then request to load that FAISS Index in Gemini where the dimensions are only 768, it will silently fail with Gemini returning error. Bugs like these are  difficult to pinpoint as everything appears to be normal on the surface-level.

I'd make sure to create a new, small metadata file each time I create a new FAISS index. Each of these metadata files will have information about what type of embedding provider was used, the model used and what dimension size is being used. Whenever you load an index, the system will read the metadata file and check if the current configuration matches the one recorded in the metadata file. If they do not match, an error will be generated immediately. By doing this, we prevent any silent failures from occurring and provide additional safety when switching embedding providers.

## Integrity Declaration

I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.
