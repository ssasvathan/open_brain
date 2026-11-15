# Could txtai serve as the local embedding and search layer?

Research for [#6](https://github.com/ssasvathan/open_brain/issues/6) (map: [#2](https://github.com/ssasvathan/open_brain/issues/2)). Sources checked 2026-11-14 against txtai v9.14.0 (released 2026-10-06, per `gh api repos/neuml/txtai/releases`).

## Answer

Yes, technically. txtai covers every capability the question lists, locally, in one Python process: dense embeddings, BM25 or sparse keyword indexing, hybrid scoring, SQL metadata filtering over JSON fields, an HTTP API, and an MCP endpoint. It can keep everything in its own files (SQLite plus Faiss) or put vectors in pgvector and content in Postgres. The catch is the MCP surface. It is a generic projection of txtai's REST endpoints, served over the deprecated SSE transport. For the pilot, the shortest path is to use txtai as the **library** behind a small hand-written MCP server whose tools fit the domain (the leadership questions), not to expose txtai's built-in `/mcp` as it is.

## Capabilities

| Need | What txtai gives | Source |
|---|---|---|
| Embeddings | `dense: true` uses the default transformers model `all-MiniLM-L6-v2`. `path` takes any local or Hugging Face model. Embedding/LLM use without torch is possible through llama.cpp, litellm or litert. | [General config](https://neuml.github.io/txtai/embeddings/configuration/general/), [Install](https://neuml.github.io/txtai/install/) |
| Keyword / hybrid | `keyword: true` builds a BM25 index. `sparse` uses a learned sparse model. `hybrid: true` adds BM25 alongside dense vectors. In SQL, `similar('query', 0.5)` sets the sparse/dense weight, and the two are weighted equally by default. | [General config](https://neuml.github.io/txtai/embeddings/configuration/general/), [Query guide](https://neuml.github.io/txtai/embeddings/query/) |
| Metadata filtering | With `content` enabled, dict fields become queryable columns: `SELECT text, flag, actiondate FROM txtai WHERE similar('query') AND flag = 1 AND actiondate >= '2022-01-01'`. Nested JSON uses `[parent.child]`. The candidate count defaults to 10x the limit when there are several filters. | [Query guide](https://neuml.github.io/txtai/embeddings/query/) |
| Own storage | `content: true` means SQLite (optional `sqlite.wal`). `duckdb` is also available. ANN defaults to Faiss. Others: hnsw, annoy, ggml, milvus, numpy, torch, sqlite (sqlite-vec), pgvector and more. | [Database config](https://neuml.github.io/txtai/embeddings/configuration/database/), [ANN config](https://neuml.github.io/txtai/embeddings/configuration/ann/) |
| Postgres / pgvector | `backend: pgvector` with `url` (or `ANN_URL`), `schema`, `table` (default `vectors`), `precision`, `efconstruction`, `m`. Content can go to any SQLAlchemy database via `content: client` + `CLIENT_URL`, Postgres included. | [ANN config](https://neuml.github.io/txtai/embeddings/configuration/ann/), [Database config](https://neuml.github.io/txtai/embeddings/configuration/database/) |
| Incremental writes | `upsert` adds or updates without a full rebuild. `delete(ids)`. `reindex` changes the model or backend from stored content without the raw data. | [Indexing](https://neuml.github.io/txtai/embeddings/indexing/) |
| HTTP API | `CONFIG=config.yml uvicorn "txtai.api:app"`. Each instance serves **one** index. It is read-only unless `writable: true`. Routes: `/search`, `/batchsearch`, `/add`, `/index`, `/upsert`, `/delete`, `/reindex`, `/count`, `/explain`, `/transform`. Writes take two steps (`/add`, then `/upsert`). Bearer-token auth via the `TOKEN` env var (SHA-256 hash). | [API config](https://neuml.github.io/txtai/api/configuration/), [Security](https://neuml.github.io/txtai/api/security/), [`routers/embeddings.py`](https://github.com/neuml/txtai/blob/master/src/python/txtai/api/routers/embeddings.py) |
| MCP | `mcp: true` in config adds `/mcp`. "All enabled endpoints set in the API configuration are automatically added as MCP tools." | [MCP docs](https://neuml.github.io/txtai/api/mcp/) |

## The MCP surface, examined

- **Implementation.** `createmcp()` in [`api/application.py`](https://github.com/neuml/txtai/blob/master/src/python/txtai/api/application.py) wraps the FastAPI app in `fastapi_mcp.FastApiMCP` and calls `mcp.mount()`.
- **Transport is SSE.** In [`fastapi_mcp/server.py`](https://github.com/tadata-org/fastapi_mcp/blob/main/fastapi_mcp/server.py), `mount()` is marked "[DEPRECATED] … For backwards compatibility, this method defaults to SSE transport". The recommended path is `mount_http()`. So txtai's MCP works today with clients that still accept SSE, but it sits on a path marked for removal.
- **Tools are generic.** They mirror the REST routes one to one: `search(query)` takes free text or txtai SQL, plus `add`, `upsert`, `delete`, `reindex` and the rest when the index is writable. `mcpargs` passes through to `FastApiMCP`, which supports `include_operations` / `exclude_operations` / `include_tags` / `exclude_tags`, so the tool list can be narrowed (same source file). Nothing in it knows about teams, projects, sources, time windows or supersession. The agent would have to write raw txtai SQL against whatever JSON fields we chose.

## Free versus hand-rolled MCP over pgvector

**What txtai gives for free:**
- An embedding pipeline that downloads and caches the model.
- BM25 plus dense hybrid scoring with a single weight knob. In plain pgvector you would add `tsvector` and write the score fusion yourself.
- SQL-style metadata filtering combined with similarity, with candidate oversampling handled.
- Upsert, delete, and `reindex` to swap the embedding model without re-scraping.
- Storage that is a choice of config value: SQLite+Faiss files for zero setup, or pgvector/Postgres.
- An optional REST API with token auth.

**What you would still build either way:**
- Domain-shaped MCP tools, e.g. "what changed for team X since date Y" or "open risks across projects".
- The record schema (source, channel/board, team, timestamp, author) and the ingest contract the scrapers call.
- Freshness/supersession logic. txtai has upsert by id, but deciding what id a re-scrape maps to is ours.
- Streamable-HTTP or stdio MCP transport. A short FastMCP/`mcp` SDK server importing `txtai.Embeddings` covers this.

**What hand-rolled pgvector gives instead:**
- A single, inspectable Postgres schema and plain SQL.
- No torch/transformers stack, if embeddings come from an external or Ollama endpoint.
- Full control of the MCP transport and tools.

The cost is building hybrid scoring, model management and reindexing yourself.

## Operational cost on a laptop

- **Python 3.11+** ([README](https://github.com/neuml/txtai)).
- **Heavy default dependencies.** `pip install txtai` pulls `torch>=2.4`, `transformers`, and `faiss-cpu` ([`setup.py`](https://github.com/neuml/txtai/blob/master/setup.py)). The docs note that the default torch is the GPU build and give a CPU-only install recipe ([Install](https://neuml.github.io/txtai/install/)). Inference: with GPU torch this means a multi-GB virtualenv, while the CPU wheel is much smaller. The API needs the `api` extra (`fastapi`, `fastapi-mcp`), and pgvector needs the `ann` extra.
- **Lighter option.** `txtai_minimal` has no required dependencies and works with llama.cpp/litellm for vectors ([Install](https://neuml.github.io/txtai/install/)).
- **Default model.** `all-MiniLM-L6-v2` is small and CPU-friendly (inference, not stated by txtai). Leadership-scale volumes (thousands to tens of thousands of chunks) are far below where Faiss or SQLite would strain.
- **Zero-service mode.** SQLite+Faiss needs no daemon, just an index directory. Indexing buffers vectors to temporary storage and builds a memory-mapped array ([Indexing](https://neuml.github.io/txtai/embeddings/indexing/)).
- **Postgres mode.** You run Postgres with the pgvector extension, which is an extra service to install and keep up for each instance (relevant to the second-instance packaging question on #2).
- **One index per API process.** Also, writes through `embeddings.save()` are process-local, so the scrapers and the harness should go through one owning process (API or MCP server), not open the index files concurrently.
- **Project health.** Apache-2.0, about 13.4k stars, monthly releases (v9.12 to v9.14 from July to October 2026) ([GitHub](https://github.com/neuml/txtai)).

## Open points for the decision ticket (#10)

- Is SSE acceptable for the harness client, or is a thin custom MCP server (stdio/streamable HTTP) wanted anyway for domain tools? The second option makes txtai's built-in `/mcp` moot.
- Own storage (SQLite+Faiss, simplest clone-and-run) versus pgvector (one more service, but plain SQL inspectability).
- The torch footprint versus `txtai_minimal` plus a llama.cpp/Ollama embedding model.
