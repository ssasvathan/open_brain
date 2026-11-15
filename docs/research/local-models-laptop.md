# Local embedding and LLM models for a laptop pilot

Research for [#7](https://github.com/ssasvathan/open_brain/issues/7) (map: [#2](https://github.com/ssasvathan/open_brain/issues/2)). Sources checked 2026-11-14. All benchmark numbers are as reported by the model owners on their own cards, not independently re-run.

## Answer

- **Embeddings, default:** `qwen3-embedding:0.6b` via Ollama, stored as `vector(1024)` in pgvector with an HNSW cosine index. Apache-2.0, 32K context, best MTEB English score of any sub-1B model checked (70.70).
- **Embeddings, lighter fallback:** `nomic-embed-text` (v1.5, 137M params, 768d, Apache-2.0, 8K context). It is Ollama's most-pulled embedder and fine on a low-RAM machine, but scores clearly lower (MTEB 62.28, older benchmark version).
- **Tagging and extraction LLM:** `qwen3.5:4b` (3.3 to 4.0 GB, Apache-2.0, tools, 256K context) with Ollama structured outputs (JSON schema in `format`). Step up to `qwen3.5:9b` (6.6 to 7.6 GB) or `gemma4:e4b` (Apache-2.0) if 4B extraction quality falls short.
- **pgvector:** any of these fits. `vector` columns can be indexed with HNSW up to 2,000 dimensions. Keep embeddings at 2,000d or below (or use `halfvec` up to 4,000d), use `vector_cosine_ops`, and keep the default HNSW parameters at pilot scale.
- **Cloud for comparison:** OpenAI `text-embedding-3-small` (1536d) and `-large` (3072d), and Google `gemini-embedding-2` (3072d, truncatable to 128 to 3072). Both 3072d defaults go over the 2,000d HNSW limit on `vector`, so request ≤2,000 dimensions (for example 1536) or use `halfvec`.

One lock-in to note for an ADR: the embedding model and its dimension are fixed per table. Switching models means re-embedding everything, because the docs say to "use the same embedding model for both indexing and querying" ([Ollama embeddings](https://docs.ollama.com/capabilities/embeddings)).

## Embedding shortlist

| Model (Ollama tag) | Params | Dims (MRL options) | Context | Licence | Reported quality | Notes |
|---|---|---|---|---|---|---|
| `qwen3-embedding:0.6b` | 0.6B | 1024 (32 to 1024) | 32K | Apache-2.0 | MTEB Eng v2 70.70; Multilingual 64.33 | Queries take the prefix `Instruct: {task}\nQuery:{query}`; documents need no prefix. The card says instructions add 1 to 5%. [card](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B) |
| `qwen3-embedding:4b` / `:8b` | 4B / 8B | 2560 / 4096 | 32K | (not stated on the 0.6B card) | Eng v2 74.60 / 75.22 | Too heavy to keep loaded next to an LLM on a laptop, and the native dims exceed the 2,000d HNSW limit on `vector`. [card](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B) |
| `embeddinggemma` | 300M | 768 (512/256/128) | 2048 | Gemma terms (gated) | MTEB Eng v2 69.67; Multilingual 61.15 | Prompts: `task: search result \| query: …` and `title: none \| text: …`. Does not support fp16 activations. [card](https://huggingface.co/google/embeddinggemma-300m) |
| `embeddinggemma-2` | 270M to 740M | 768 (512/256/128) | 8K | not stated (card gated) | MTEB Multilingual v2 61.36 at 768d | New, multimodal, built for laptops and mobile. Text-only 270m is about 0.3 GB. [Ollama](https://ollama.com/library/embeddinggemma-2) |
| `nomic-embed-text` (v1.5) | 137M | 768 (512/256/128/64) | 8192 | Apache-2.0 | MTEB 62.28 (older MTEB version; not comparable to v2 numbers) | Requires the `search_query:` / `search_document:` prefixes. 97.8M pulls. [card](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5) |
| `bge-m3` | 567M | 1024 | 8192 | MIT | (not checked) | No query instruction needed. Also produces sparse and ColBERT vectors, though Ollama returns dense only. [card](https://huggingface.co/BAAI/bge-m3) |
| `mxbai-embed-large`, `snowflake-arctic-embed`, `all-minilm`, `granite-embedding` | 22M to 568M | various | — | — | not evaluated here | Listed in the [Ollama embedding catalog](https://ollama.com/search?c=embedding). |

Ollama `/api/embed` takes a string or an array (batching), returns L2-normalised vectors, accepts a `dimensions` integer for MRL truncation, and truncates over-long input by default (`truncate: true`) ([API ref](https://docs.ollama.com/api/embed), [guide](https://docs.ollama.com/capabilities/embeddings)). Set `truncate: false` if silently cut Slack threads would be a problem.

**Caveat:** MTEB is a general benchmark. None of these numbers measure terse Slack/Jira business text. Treat the shortlist as a starting point, and test the top two against real acceptance questions from #2.

## Small LLM shortlist (tagging and entity extraction)

| Model (Ollama tag) | Size on disk | Context | Licence | Capabilities | Notes |
|---|---|---|---|---|---|
| `qwen3.5:4b` | 3.3 to 4.0 GB | 256K | Apache-2.0 | tools, thinking, vision | Released Feb 2026. **Thinking is on by default** and the `/nothink` switch is gone, so disable thinking for bulk extraction. [card](https://huggingface.co/Qwen/Qwen3.5-4B), [Ollama](https://ollama.com/library/qwen3.5) |
| `qwen3.5:9b` | 6.6 to 7.6 GB | 256K | Apache-2.0 | tools, thinking, vision | Step up if a 16 GB+ laptop has headroom. [Ollama](https://ollama.com/library/qwen3.5) |
| `qwen3.5:2b` / `:0.8b` | 2.7 to 3.1 / 1.2 to 1.3 GB | 256K | Apache-2.0 | same | For very constrained machines. [Ollama](https://ollama.com/library/qwen3.5) |
| `gemma4:e4b` | 6.6 to 9.5 GB | 128K | Apache-2.0 | tools, thinking, vision, audio | 4.5B effective (8B with embeddings). Native function calling. Tech report published Jul 2026. [card](https://huggingface.co/google/gemma-4-E4B-it), [Ollama](https://ollama.com/library/gemma4) |
| `gemma4:e2b` | 4.6 to 7.5 GB | 128K | Apache-2.0 | same | About 2.3B effective. [Ollama](https://ollama.com/library/gemma4) |
| `qwen3:4b` / `:8b` | 2.5 / 5.2 GB | 256K / 40K | not stated on Ollama page | tools, thinking | Previous generation, last updated about 1 year ago. Superseded by qwen3.5. [Ollama](https://ollama.com/library/qwen3) |

For extraction, use Ollama **structured outputs**: pass a JSON Schema in `format` (`response_format` on the OpenAI-compatible endpoint), repeat the schema in the prompt, and set `temperature: 0` ([docs](https://docs.ollama.com/capabilities/structured-outputs)). Ollama's cloud does not support structured outputs, so this only works locally. Gemma 4's recommended sampling (temp 1.0, top_p 0.95, top_k 64) is for chat. For extraction, the structured-output guidance on temperature 0 is the more relevant one.

**Memory budget (inference, not measured):** a 16 GB laptop can keep the 0.6B embedder (about 0.6 GB) and a 4B LLM (about 3 to 4 GB) loaded together alongside Postgres. A 9B LLM or `gemma4:e4b` gets tight on 16 GB and is comfortable on 32 GB. No throughput figures were found in primary sources. Benchmark on the target laptop if ingest latency matters.

## pgvector pairing

From the [pgvector README](https://github.com/pgvector/pgvector) (v0.8.7):

- Storage: `vector` and `halfvec` up to 16,000 dims. Indexing: HNSW and IVFFlat index `vector` **up to 2,000 dims**, `halfvec` up to 4,000, `bit` up to 64,000, and HNSW indexes `sparsevec` up to 1,000 non-zero elements.
- Operators: `<=>` cosine, `<#>` negative inner product, `<->` L2. Ollama, OpenAI and Gemini at 3072d all return unit-length vectors, so cosine and inner product rank identically, and `vector_cosine_ops` is the safe default.
- **HNSW** (recommended for the pilot): defaults `m=16`, `ef_construction=64`, `hnsw.ef_search=40`. Can be built on an empty table.
- **IVFFlat:** "Create the index *after* the table has some data". Lists start at `rows/1000`, probes at `sqrt(lists)`. Little benefit at single-user pilot scale.
- At pilot scale (thousands to low hundreds of thousands of chunks), an exact scan without an index is also viable. HNSW is the standard upgrade path.

Fit per model: qwen3-embedding-0.6B (1024), the nomic and gemma embedders (768) and bge-m3 (1024) index directly as `vector(n)`. qwen3-embedding-4B/8B, `text-embedding-3-large` and `gemini-embedding-*` at their 3072d default must be truncated to ≤2,000 (via the `dimensions` / `output_dimensionality` parameters) or stored as `halfvec`.

## Cloud equivalents

| Provider model | Default dims | Shorten to | Max input | Reported MTEB | Source |
|---|---|---|---|---|---|
| OpenAI `text-embedding-3-small` | 1536 | `dimensions` param | 8192 | 62.3% | [OpenAI guide](https://developers.openai.com/api/docs/guides/embeddings) |
| OpenAI `text-embedding-3-large` | 3072 | `dimensions` (for example 1024 or 256) | 8192 | 64.6% | same |
| Google `gemini-embedding-2` | 3072 | 128 to 3072 (768/1536/3072 recommended), auto-normalised | 8192 | — | [Gemini docs](https://aistudio.google.com/docs/embeddings) |
| Google `gemini-embedding-001` | 3072 | 128 to 3072, normalise manually | 2048 | — | same |

OpenAI's MTEB figures use the older MTEB version, so they compare with nomic's 62.28, not with Qwen3/EmbeddingGemma "v2" scores. On cloud LLMs for extraction: any hosted frontier model will out-extract a local 4B. The trade-off is data governance (scraped org content leaving the laptop), which is a separate ticket on the map.

## Open points

- EmbeddingGemma 2's licence could not be read (gated HF card). Confirm it before choosing it.
- Licences for qwen3-embedding 4B/8B were not confirmed on the card checked.
- No primary-source laptop throughput numbers. Measure ingest speed on the target hardware.
