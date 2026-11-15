# Local embedding and LLM models for a laptop pilot

Research for [#7](https://github.com/ssasvathan/open_brain/issues/7) (map: [#2](https://github.com/ssasvathan/open_brain/issues/2)). Sources checked 2026-11-14. Benchmark numbers are the ones model owners report on their own cards. Nobody has re-run them independently.

**Revision 2.** #7 was reopened with a standing constraint: no models from Chinese developers, and no fine-tunes derived from them. The first revision recommended `qwen3-embedding` and `qwen3.5` (Alibaba), so that recommendation is withdrawn. This revision:

- names the developer, country and licence for every model;
- checks each model card for a base model of Chinese origin;
- assesses the pairing the user already tested hands-on, `nomic-embed-text` plus `gemma4:e4b` via Ollama (srnichols/OpenBrain `.env`), on the target hardware:
  - the user's 2023 M3 MacBook with **32 GB** of unified memory (the target);
  - the manager's second instance, probably a **16 GB** Apple Silicon machine (the floor);
  - a home PC with a GTX 1070 Ti (8 GB VRAM).

  8 GB machines are unsupported.

## Answer

Keep the pairing that has already been tested. Nothing that passes the provenance rule is clearly better for terse engineering text.

- **Embedder: `nomic-embed-text` (v1.5).** From Nomic AI (US), Apache-2.0, 137M parameters, 274 MB in Ollama.
  - Store it as **`vector(768)`** with an **HNSW index using `vector_cosine_ops`** and the default parameters (m=16, ef_construction=64).
  - **Chunk to 2K tokens or fewer**, because Ollama serves it with a 2K context (see [Context limits](#context-limits-and-chunking)).
  - **Use its required task prefixes**: `search_document: ` on stored text and `search_query: ` on queries.
- **Tagging and extraction LLM: `gemma4:e4b`.** From Google DeepMind (US), Apache-2.0. Ollama download is 6.6 GB, context 128K.
  - Call it through **Ollama structured outputs** (a JSON Schema in `format`) with `temperature: 0` and thinking off.
- **Challenger for the embedder bake-off: `embeddinggemma-2:270m`** (Google DeepMind, US, Apache-2.0, text-only, 304 MB). It has an 8K context and the same 768 dimensions, so the bake-off needs no schema change, only a re-embed. Neither the challenger nor anything else has evidence that it beats nomic on Slack/Jira-style text, so it does not displace the incumbent without the bake-off already planned on #2.
- **Hardware fit:**
  - **16 GB Apple Silicon (floor):** runs the pair comfortably.
  - **32 GB (the user's target):** runs the pair with plenty of headroom, and has room for `gemma4:12b` if extraction quality ever falls short.
  - **8 GB:** unsupported.
  - **GTX 1070 Ti (8 GB VRAM):** holds the pair only with a modest context (keep `num_ctx` around 8K or lower). Confirm with `ollama ps` that the model is 100% on the GPU.
- **LLM fallbacks:**
  - Smaller: `granite4:micro` (IBM, US, Apache-2.0, 2.1 GB), `gemma4:e2b` (4.6 GB) or `phi4-mini` (Microsoft, US, MIT, 2.5 GB).
  - Larger, on the 32 GB Mac: `gemma4:12b` (7.7 GB) or `ministral-3:8b` (Mistral AI, France, Apache-2.0, 6.0 GB).
- **Embedder fallbacks:** `embeddinggemma:300m` (Google, Gemma terms, 2K context) and `granite-embedding:278m` (IBM, Apache-2.0, multilingual).

Lock-in to record in an ADR: the embedding model and its dimension are fixed per table. Ollama's docs say to "use the same embedding model for both indexing and querying" ([Ollama embeddings](https://docs.ollama.com/capabilities/embeddings)), so changing models means re-embedding everything. Choosing 768 dimensions keeps nomic, EmbeddingGemma 1 and 2, and Granite on the same column type.

## Provenance check

Developer, country and licence come from each model card or Ollama page. The base-model check reads the `base_model` field in the card metadata (Hugging Face API) and searches the card text for "based on", "builds on" and "initialised from".

### Allowed

| Model | Developer (country) | Licence | Base model / Chinese-origin check | Source |
|---|---|---|---|---|
| nomic-embed-text v1.5 | Nomic AI (US) | Apache-2.0 | Trained from Nomic's own `nomic-bert-2048`. No Chinese base. | [card](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5) |
| EmbeddingGemma 2 | Google DeepMind (US/UK) | Apache-2.0 (Gemma 4 licence page) | "Builds upon … Gemma 4". No Chinese base. | [card](https://huggingface.co/google/embeddinggemma-2), [Ollama](https://ollama.com/library/embeddinggemma-2) |
| EmbeddingGemma 300m | Google DeepMind (US/UK) | Gemma terms (gated, own use policy) | Gemma 3 with T5Gemma initialisation. No Chinese base. | [card](https://huggingface.co/google/embeddinggemma-300m) |
| granite-embedding 30m / 278m, english-r2 | IBM (US) | Apache-2.0 | Encoders "trained internally at IBM Research" (RoBERTa/XLM-R-like; r2 is ModernBERT on IBM's GneissWeb). No Chinese base. | [30m](https://huggingface.co/ibm-granite/granite-embedding-30m-english), [278m](https://huggingface.co/ibm-granite/granite-embedding-278m-multilingual), [r2](https://huggingface.co/ibm-granite/granite-embedding-english-r2) |
| snowflake-arctic-embed (v1: xs/s/m/l, m-long) | Snowflake (US) | Apache-2.0 | Bases: xs on all-MiniLM-L6-v2, s/m/l on Microsoft `intfloat/e5-*-unsupervised`, m-long on Nomic. No Chinese base. | [card](https://huggingface.co/Snowflake/snowflake-arctic-embed-l) |
| Gemma 4 (e2b, e4b, 12b) | Google DeepMind (US/UK) | Apache-2.0 | Base `google/gemma-4-E4B` etc. | [card](https://huggingface.co/google/gemma-4-E4B-it) |
| Granite 4.0 (micro 3B, micro-h, tiny-h 7B-A1B) | IBM (US) | Apache-2.0 | Fine-tuned from IBM's own Granite-4.0 base. | [card](https://huggingface.co/ibm-granite/granite-4.0-micro), [Ollama](https://ollama.com/library/granite4) |
| Phi-4-mini-instruct (3.8B) | Microsoft (US) | MIT | Microsoft's own Phi-4 series. | [card](https://huggingface.co/microsoft/Phi-4-mini-instruct) |
| Ministral 3 (3B/8B/14B) | Mistral AI (France) | Apache-2.0 | Base `mistralai/Ministral-3-*-Base-2512`. | [card](https://huggingface.co/mistralai/Ministral-3-3B-Instruct-2512) |
| Llama 3.2 (1B/3B) | Meta (US) | Llama 3.2 Community Licence (gated, acceptable-use policy) | Meta's own base. Card gated, so not read in full. | [Ollama](https://ollama.com/library/llama3.2) |

### Excluded

| Model | Why |
|---|---|
| `snowflake-arctic-embed2` (= arctic-embed-l-v2.0) | US developer, but the card says it "builds on [BAAI/bge-m3-retromae]". BAAI is Chinese, so this is a derived model. [card](https://huggingface.co/Snowflake/snowflake-arctic-embed-l-v2.0) |
| snowflake-arctic-embed-m-v2.0 | The card says it "builds on [GTE-multilingual-base]" (Alibaba). Derived model. [card](https://huggingface.co/Snowflake/snowflake-arctic-embed-m-v2.0) |
| `mxbai-embed-large` (Mixedbread) | The card names no base model and its metadata has none, so provenance can't be confirmed from the primary source. Excluded until Mixedbread confirms. [card](https://huggingface.co/mixedbread-ai/mxbai-embed-large-v1) |
| `bge-m3`, `qwen3-embedding`, `qwen3.5`, `qwen3`, and any GTE, DeepSeek, GLM, Kimi, Yi, MiniMax or InternLM model | Chinese developer. |

`nomic-embed-text-v2-moe` (Nomic, Apache-2.0) is allowed: it is built on Nomic's `nomic-xlm-2048`, which traces to Meta's XLM-R. It was dropped anyway because its card states a 512-token maximum input ([card](https://huggingface.co/nomic-ai/nomic-embed-text-v2-moe)).

## Embedding shortlist (provenance-clean)

| Ollama tag | Params / download | Dims (MRL) | Context | Reported quality | Notes |
|---|---|---|---|---|---|
| **`nomic-embed-text`** (incumbent) | 137M / 274 MB | 768 (512/256/128/64) | 8192 on the card, **2K as served by Ollama** | MTEB 62.28 (v1 benchmark) | Prefixes are required: the card says the prompt "*must* include a task instruction prefix". The Ollama page's examples omit them, so check that the existing OpenBrain setup adds them. [card](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5), [Ollama](https://ollama.com/library/nomic-embed-text) |
| **`embeddinggemma-2:270m`** (challenger) | 270M text / 304 MB (q8_0) | 768 (512/256/128) | 8,192 on the card. Ollama's tag list shows 256K, which conflicts with the card, so trust the card. | MTEB eng v2 68.46, multilingual v2 61.36 | Prefixes `task: search result \| query: …` and `title: {title or none} \| text: …`. **Never run in float16** (it returns NaN or silently degraded vectors). Use bf16 or fp32, or the q8 tag. [card](https://huggingface.co/google/embeddinggemma-2) |
| `embeddinggemma:300m` | 300M / 622 MB (QAT q8 338 MB) | 768 (512/256/128) | 2048 | MTEB eng v2 69.67, multilingual v2 61.15 | Gemma terms rather than Apache-2.0. Same float16 caveat. [card](https://huggingface.co/google/embeddinggemma-300m) |
| `granite-embedding:278m` / `:30m` | 278M / 563 MB; 30M / 63 MB | 768 / 384 | 512 (r1) | 30m: BEIR 49.1 | Trained without MS-MARCO (licence-clean data). The 8K ModernBERT r2 (149M, MTEB-v2 62.8) is not in the Ollama library. [Ollama](https://ollama.com/library/granite-embedding), [r2 card](https://huggingface.co/ibm-granite/granite-embedding-english-r2) |
| `snowflake-arctic-embed:m-long` | 137M / 274 MB | 768 | 2K in Ollama | — | Nomic-based v1 model. No advantage over nomic itself. [Ollama](https://ollama.com/library/snowflake-arctic-embed) |

**Is anything clearly better than nomic for terse engineering text?** Not on the evidence available.

- EmbeddingGemma 1 and 2 report higher MTEB English **v2** scores (68 to 70). nomic's 62.28 is on the older MTEB **v1** suite, and the two suites cannot be compared directly.
- No card measures short, jargon-heavy Slack/Jira text.
- The user has seen good results with nomic on their own data, and that is direct evidence.

So nomic stays the default. `embeddinggemma-2:270m` is the one challenger worth the bake-off. It is the only Apache-2.0 option with a better published score and a longer usable context in Ollama. The bake-off is already on the map (#2, "Evaluating the acceptance cases").

### Context limits and chunking

- **nomic-embed-text in Ollama is limited to 2K tokens.** The Ollama page lists "2K context window" for every tag, even though the HF card supports 8192. `/api/embed` defaults to `truncate: true`, so anything longer is **cut silently** ([API ref](https://docs.ollama.com/api/embed)).
- A long Confluence page or a long Jira comment thread must therefore be split before embedding. Chunks of about 500 to 1,000 tokens fit well inside the limit and also retrieve better than one vector per long page.
- Send `truncate: false` so an over-long chunk fails loudly instead of being cut.
- **Storage implication:** this matters for the unit-of-storage decision. The OpenBrain references store one unchunked row per thought (#5). Here, a raw source item needs either child chunk rows (item id, chunk index, text, `vector(768)`) or an item-level summary embedding. Short Jira tickets and chat messages fit in one chunk.
- **Effect of the challenger:** `embeddinggemma-2` (8K) would allow larger chunks or whole tickets in one vector. It does not remove the need to chunk Confluence pages.

## Small LLM shortlist (tagging and entity extraction, provenance-clean)

| Ollama tag | Developer (country), licence | Download | Context | Notes |
|---|---|---|---|---|
| **`gemma4:e4b`** (incumbent) | Google DeepMind (US/UK), Apache-2.0 | 6.6 GB (QAT tag `e4b-it-qat` 6.1 GB) | 128K | About 4.5B effective parameters (8B including embeddings). The download includes vision and audio encoders. Native function calling. Thinking turns on only when `<\|think\|>` is in the system prompt, and with thinking off the E2B/E4B models emit no thought tags. [Ollama](https://ollama.com/library/gemma4) |
| `gemma4:e2b` | same | 4.6 GB (QAT 4.3 GB) | 128K | About 2.3B effective. The low-RAM option within the same family. |
| `gemma4:12b` | same | 7.7 GB (QAT 7.2 GB) | 256K | Step up on the 32 GB Mac. |
| `granite4:micro` (3B dense) / `micro-h` | IBM (US), Apache-2.0 | 2.1 GB / 1.9 GB | 128K | Ollama lists "extraction", "classification" and function calling among intended uses. Smallest credible fallback. [Ollama](https://ollama.com/library/granite4) |
| `phi4-mini` (3.8B) | Microsoft (US), MIT | 2.5 GB | 128K | Tools. Text only. [Ollama](https://ollama.com/library/phi4-mini) |
| `ministral-3:3b` / `:8b` | Mistral AI (France), Apache-2.0 | 3.0 / 6.0 GB | 256K | "Native function calling and JSON outputting". Vision. [Ollama](https://ollama.com/library/ministral-3) |
| `llama3.2:3b` | Meta (US), Llama 3.2 Community Licence | 2.0 GB | 128K | Older (2024). Custom licence. Listed only for completeness. |

For extraction, use Ollama **structured outputs**:

- Pass a JSON Schema in `format`, or `response_format` on the OpenAI-compatible endpoint.
- Also put the schema in the prompt.
- Set `temperature: 0`.

Ollama's cloud "currently does not support structured outputs", so this works only locally ([docs](https://docs.ollama.com/capabilities/structured-outputs)). Gemma 4's recommended sampling (temperature 1.0, top_p 0.95, top_k 64) is meant for chat. For extraction, the structured-output advice of temperature 0 takes precedence.

## Hardware fit and expected speed

Hardware facts:

- **M3 MacBook Pro (14-inch, Nov 2023):** the base M3 chip has 100 GB/s memory bandwidth and ships with 8 GB or 16 GB, configurable to 24 GB ([Apple](https://support.apple.com/117735)). The base M3 has no 32 GB option, so the user's 32 GB machine is probably an M3 Pro or M3 Max. Those have more bandwidth and would be faster than the estimates below, so the figures here are conservative.
- **GTX 1070 Ti:** 8 GB GDDR5 on a 256-bit bus, 256 GB/s ([TechPowerUp review](https://www.techpowerup.com/review/msi-gtx-1070-ti-gaming/2.html)). Ollama supports it as compute capability 6.1, but cards in the 5.0 to 6.2 range need **NVIDIA driver 570 or newer** ([Ollama GPU docs](https://docs.ollama.com/gpu)).
- Ollama uses Metal on Apple Silicon.

Speeds below are **estimates, not measurements**. No primary source publishes tokens per second for these models on this hardware. Token generation is limited by memory bandwidth, so the ceiling is roughly bandwidth divided by the bytes of weights read per token. For `gemma4:e4b` at 4-bit that is about 2.5 to 3 GB per token, and real throughput usually lands at about half the ceiling.

| Machine | `nomic-embed-text` + `gemma4:e4b`? | Expected LLM generation | Recommendation |
|---|---|---|---|
| Apple Silicon, 16 GB (floor: the manager's instance) | **Yes.** About 7 GB of models, plus Postgres (well under 1 GB at pilot scale), plus macOS and apps. Comfortable if heavy apps are closed during bulk ingest. | Ceiling about 35 to 40 tok/s at 100 GB/s, so roughly 15 to 25 tok/s in practice | Incumbent pair. If memory pressure appears, drop to `gemma4:e4b-it-qat` (6.1 GB) or `gemma4:e2b`. |
| M3-family, 32 GB (target: the user's machine) | **Yes, with plenty of room.** | at least the figure above, and higher if the chip is an M3 Pro or Max | Incumbent pair. `gemma4:12b` (7.7 GB, about half the speed) is the step up if extraction quality falls short. |
| 8 GB (unsupported) | No. | — | If a machine like this ever has to be used, try `gemma4:e2b` or `granite4:micro`. |
| GTX 1070 Ti, 8 GB VRAM | **Tight.** 6.6 GB of weights plus the KV cache plus nomic (about 0.3 GB) is close to 8 GB. Ollama moves layers that don't fit to the CPU, which is much slower. | Ceiling about 85 to 100 tok/s, so roughly 40 to 60 tok/s if fully on the GPU | Use the QAT tag (`gemma4:e4b-it-qat`, 6.1 GB). Keep `num_ctx` around 8K or lower. Check `ollama ps` shows "100% GPU". Otherwise use `gemma4:e2b` or `granite4:micro`. The embedder can stay on the CPU without much cost. |

**Per-item cost (estimate).** Tagging one item means reading about 1 to 2K tokens of prompt and writing about 100 to 200 tokens of JSON. That works out to a few seconds to around 15 seconds per item on an M3, and less on the 1070 Ti when fully on the GPU. Embedding with a 137M model is fast by comparison.

A one-off backfill of thousands of items is an overnight job. Incremental re-scrapes are not a bottleneck.

**Measure on the real machine before relying on any of these numbers.** `ollama run --verbose` prints prompt and generation rates.

## pgvector pairing

From the [pgvector README](https://github.com/pgvector/pgvector) (v0.8.7):

- Storage: `vector` up to 16,000 dims, at `4 * dims + 8` bytes each (768d is about 3 KB per row). Indexing: HNSW and IVFFlat index `vector` **up to 2,000 dims**, and `halfvec` up to 4,000.
- Every model shortlisted here outputs 768 dimensions or fewer, so a plain **`vector(768)`** column and **HNSW with `vector_cosine_ops`** at the defaults (m=16, ef_construction=64, `hnsw.ef_search`=40) is enough. HNSW can be built on an empty table. IVFFlat should only be built after the table has data, and it gains little at pilot scale.
- Ollama returns L2-normalised vectors ([API ref](https://docs.ollama.com/api/embed)), so cosine (`<=>`) and inner product rank results the same way.
- Ollama's `dimensions` parameter allows MRL truncation (for example 512) if storage ever matters. The truncated dimension then becomes the locked one.

## Cloud equivalents (unchanged; for comparison only)

OpenAI `text-embedding-3-small` (1536d) and `-large` (3072d), from OpenAI (US). Google `gemini-embedding-2` (3072d, truncatable from 128 up), from Google (US). Both 3072d defaults exceed the 2,000-dim HNSW limit on `vector`, so request 2,000 dimensions or fewer, or use `halfvec` ([OpenAI](https://developers.openai.com/api/docs/guides/embeddings), [Gemini](https://aistudio.google.com/docs/embeddings)). Whether data may leave the laptop at all is a data-governance question tracked elsewhere on the map.

## Open points

- **Bake-off:** compare `nomic-embed-text` and `embeddinggemma-2:270m` on the real acceptance questions from #2 and #3, with the same chunking for both.
- **Prefixes:** confirm that the existing OpenBrain setup adds nomic's `search_document:` and `search_query:` prefixes. If it doesn't, adding them is a free quality gain, but every stored vector must then be re-embedded.
- **Office Mac chip:** the base M3 has no 32 GB option, so confirm whether the machine is an M3 Pro or M3 Max. This only affects the speed estimates, not the recommendation.
- **Measured speeds:** `gemma4:e4b` ingest rate on the M3 and the 1070 Ti is not yet measured.
- **Ollama context for `embeddinggemma-2`:** the tag list says 256K but the card says 8K. Verify with `ollama show embeddinggemma-2:270m`.
- **`mxbai-embed-large` provenance:** unconfirmed. Ask Mixedbread if it is ever wanted.
