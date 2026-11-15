# Does the pilot need a local reranker, and which non-Chinese one?

Research for [#17](https://github.com/ssasvathan/open_brain/issues/17). Sources checked 2026-11-15. Model facts come from Hugging Face model cards and `config.json` files, read through the Hugging Face API on that date. txtai facts come from the `neuml/txtai` source on `master`. Ollama facts come from Ollama release notes (latest release v0.40.16).

## Answer

**Not on day one. Build a rerank seam, then decide in the bake-off.**

- The acceptance questions are list and aggregation questions. Their quality depends on **recall**: did the right Jira issues, threads and KIP/FLIP pages make it into the candidate pool at all? A reranker only **reorders** the pool that hybrid search returns. It cannot bring back an item that BM25 and dense search both missed.
- A reranker earns its place in one case: the pool is bigger than what we can hand to gemma4:e4b (for example, 150 candidates but room for only 20 to 30 in the prompt), and the right items rank too low to make the cut. That is a precision@N problem. Only the bake-off can show whether we have it.
- If we do, the first candidate is **IBM `granite-embedding-reranker-english-r2`**. It is from IBM (US), Apache-2.0, 149M parameters, 8,192-token context, and was trained without MS MARCO. It runs through txtai's `Similarity(crossencode=True)` pipeline. The second choice is **gemma4:e4b as a pointwise yes/no reranker** through Ollama logprobs. It needs no new model, but it is slower: 11 to 16 s per 50 candidates on this machine's 8 GB GPU.

**What to measure in the bake-off:**

1. Recall@K of hybrid search alone, for K = 20, 50 and 100, on the acceptance questions.
2. The gain in recall@N (N = what fits in the prompt) when a reranker reorders the top K.
3. Latency per query on the target laptop, CPU only and with GPU.

Add the reranker only if (2) is material and (3) is acceptable.

## Why recall decides it for this pilot

- In BEIR, every BM25+CE result reranks **the BM25 top 100**. The paper describes the model as "BM25 + CE reranks the top-100 retrieved hits from a first-stage BM25 (Anserini) model" ([Thakur et al., BEIR, NeurIPS 2021 Datasets track, §4](https://arxiv.org/abs/2104.08663)). Whatever falls outside that pool stays out.
- qmd works the same way. It fuses BM25 and vector lists with RRF, takes the top 30 to 40 candidates (`candidateLimit`, default 40), and then reranks. It also blends the reranker score with the retrieval rank so that strong lexical hits survive: 75% retrieval weight for ranks 1 to 3, 60% for ranks 4 to 10, and 40% after that ([qmd README](https://github.com/tobi/qmd)).
- More candidates is not always better. Jacob et al. found that the best rerankers improve at first as they score more documents, but past a point quality *drops* ([Drowning in Documents, arXiv:2411.11767](https://arxiv.org/abs/2411.11767)). For list questions, the safer lever is often a bigger, better-filtered pool from SQL metadata (project, date range, status, issue type, "vote passed" and similar), not reranking deeper.
- What follows for us: put effort first into recall (hybrid weights, metadata filters, chunking, query expansion as in qmd's typed `lex`/`vec` sub-queries). Treat the reranker as a precision step for the final cut into the prompt.

## Evidence that rerank helps hybrid retrieval

| Source | Finding | Trust and caveats |
|---|---|---|
| [BEIR](https://arxiv.org/abs/2104.08663), Table 2 | BM25+CE (MiniLM-L6 cross-encoder over the BM25 top 100) is the best zero-shot system, at +11% nDCG@10 over BM25 on average. It beats BM25 on 16 of 18 datasets, and loses on ArguAna and Touché-2020. | Peer-reviewed. First stage is BM25 only, not hybrid. |
| BEIR, short-text datasets | **Signal-1M** (tweets, about 14 words per document): BM25 0.330, BM25+CE 0.338, so almost no gain. **CQADupStack** (StackExchange questions): 0.299 to 0.370. **Quora** (short duplicate questions): 0.789 to 0.825, but dense ANCE (0.852) and ColBERT (0.854) beat the cross-encoder. | Closest public proxy for Slack and Jira text. Mixed: for very short text, rerank gains are small. |
| BEIR, Table 3 | BM25+CE on 1M DBPedia documents: 450 ms on a V100 GPU and **6,100 ms on an 8-core Xeon CPU** per query (top 100). | 2021 software. Gives a rough CPU anchor. |
| [Azure AI Search team (Microsoft)](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/azure-ai-search-outperforming-vector-search-with-hybrid-retrieval-and-reranking/3929167) | Hybrid then semantic reranker: customer datasets NDCG@3 48.4 to 60.1; BEIR 48.4 to 50.0. For **keyword queries** it is worse than keyword alone (66.9 vs 79.2). For **short queries**: hybrid 53.0, with reranker 63.9. | Vendor benchmark, proprietary reranker. On public BEIR the gain is small (+1.6). Large gains are on their own data. |
| [Anthropic, Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) | Top-20 retrieval failure: contextual embeddings + contextual BM25 2.9%, plus reranking 1.9%. That is top 150 reranked to top 20 with Cohere's reranker. | Vendor, hosted reranker. Same pattern we would use: big pool, rerank to what fits. |
| [Déjean et al., arXiv:2403.10407](https://arxiv.org/abs/2403.10407) | Reranking SPLADE: "traditional cross-encoders remain very competitive" against GPT-4-based listwise rerankers, and both the model and the number of documents reranked matter. | Peer-reviewed workshop/preprint. |

**For Slack messages and Jira issues:** the evidence is weakest exactly where our text is shortest (Signal-1M tweets). The pilot's unit of storage is the *artefact*: a whole Slack thread, a Jira issue with its comments, a KIP page. Those are much longer than a single tweet, so they sit closer to CQADupStack, where rerank gains were large. No public benchmark matches Jira/Slack/dev@ artefacts, so the bake-off has to measure this.

## Candidates

The developer is the organisation that published the weights. "Base" is the model it was fine-tuned from, checked in the model card, `base_model` metadata and `config.json`.

### Usable (not Chinese-developed, base not Chinese)

| Model | Developer (country) | Base | License | Params | Context | Plugs into | Notes |
|---|---|---|---|---|---|---|---|
| [`ibm-granite/granite-embedding-reranker-english-r2`](https://huggingface.co/ibm-granite/granite-embedding-reranker-english-r2) | IBM (US) | IBM `granite-embedding-english-r2` (ModernBERT architecture) | Apache-2.0 | 149M | 8,192 | txtai `Similarity(crossencode=True)`; sentence-transformers `CrossEncoder`; TEI `/rerank` | Released 2025-09-08. The card says IBM does "not use the popular MS-MARCO retrieval dataset". BEIR avg 55.0 to 55.8 when paired with Granite retrievers (metric not named in the card). The 8K context covers whole Jira issues and threads without truncation. **Best fit.** |
| [`mixedbread-ai/mxbai-rerank-base-v1`](https://huggingface.co/mixedbread-ai/mxbai-rerank-base-v1) (also `xsmall` 71M, `large` 435M) | Mixedbread (Berlin, Germany, per third-party profiles; not confirmed on a first-party page) | DeBERTa-v3 (Microsoft, US): `config.json` is `DebertaV2ForSequenceClassification` with the 128,100-token DeBERTa-v3 vocabulary. The card does not name the base. | Apache-2.0 | 184M | 512 | txtai crossencode; sentence-transformers | BEIR nDCG@10 over the lexical top 100: xsmall 43.9, base 46.9, large 48.8 ([Mixedbread blog](https://www.mixedbread.com/blog/mxbai-rerank-v1)). **Training data and teacher are not disclosed**, so provenance is partly unclear. The 512-token limit means long artefacts must be reranked per chunk. |
| [`cross-encoder/ms-marco-MiniLM-L6-v2`](https://huggingface.co/cross-encoder/ms-marco-MiniLM-L6-v2) (also L12, TinyBERT-L2) | sentence-transformers project (originally Nils Reimers, UKP Lab TU Darmstadt, Germany; now hosted by Hugging Face, US) | `microsoft/MiniLM-L12-H384-uncased` (Microsoft, US), via L12 | Apache-2.0 weights | 22.7M | 512 | txtai crossencode; sentence-transformers | TREC DL19 nDCG@10 74.30; 1,800 docs/s on a V100. This is the BEIR BM25+CE model family. **Trained on MS MARCO, whose terms say "intended for non-commercial research purposes only"** ([MS MARCO](https://microsoft.github.io/msmarco/)). Fine for a pilot on public data; a licensing question for production. Fastest option. |
| [`colbert-ir/colbertv2.0`](https://huggingface.co/colbert-ir/colbertv2.0) | Stanford (US) | BERT-base (Google, US) | MIT weights | 110M | 512 | txtai `Similarity(lateencode=True)` (the txtai Reranker docs example) | Late interaction. Same MS MARCO caveat. BEIR avg 50.02 per the Answer.AI comparison below. |
| [`nvidia/llama-nemotron-rerank-1b-v2`](https://huggingface.co/nvidia/llama-nemotron-rerank-1b-v2) | NVIDIA (US) | `meta-llama/Llama-3.2-1B` (Meta, US) | OpenMDW-1.1 plus Llama 3.2 Community License ("Built with Llama") | 1.24B | 8,192 | transformers with `trust_remote_code=True` (custom `LlamaBidirectionalForSequenceClassification`); txtai compatibility not checked | Tested on Ampere/Hopper/Lovelace GPUs only; no CPU guidance. Recall@5 with NVIDIA's embedder on BEIR+TechQA: 68.60% to 73.64%. **Too heavy for a CPU laptop**, and its license is more complex. Listed only for completeness. |
| `gemma4:e4b` as an LLM reranker | Google DeepMind (US/UK) | (already chosen) | Gemma terms | 4.5B effective (8B with embeddings) | 128K ([Ollama](https://ollama.com/library/gemma4)) | Ollama `/api/generate` with `logprobs` (since [v0.12.11](https://github.com/ollama/ollama/releases/tag/v0.12.11)) | See the LLM-as-reranker section. |

### Usable only if Jina provenance is accepted

| Model | Developer | License | Params | Context | Note |
|---|---|---|---|---|---|
| [`jinaai/jina-reranker-v1-turbo-en`](https://huggingface.co/jinaai/jina-reranker-v1-turbo-en) | Jina AI | Apache-2.0 | 38M | 8,192 (JinaBERT, `trust_remote_code`) | Small, long context, permissive license. |
| [`jinaai/jina-reranker-v2-base-multilingual`](https://huggingface.co/jinaai/jina-reranker-v2-base-multilingual) | Jina AI | **CC-BY-NC-4.0** (non-commercial) | 278M | 1,026 positions (XLM-RoBERTa) | The license rules it out for bank use regardless of provenance. |

**Jina AI's provenance:** Jina AI GmbH has its headquarters in Berlin. Its press kit says it represents "Jina AI Limited (HK), Jina AI (Beijing) Limited, Beijing Jina Technology Limited, Jina AI GmbH" and gives a Beijing office address ([jina-ai/press-kit](https://github.com/jina-ai/press-kit)). Elastic (NYSE: ESTC) completed its acquisition of Jina AI on 2025-10-09 ([jina.ai/about-us](https://jina.ai/about-us); [Business Wire](https://www.businesswire.com/news/home/20251009619654/en/Elastic-Completes-Acquisition-of-Jina-AI-a-Leader-in-Frontier-Models-for-Multimodal-and-Multilingual-Search)). The pre-2025 models were built by a group with Chinese subsidiaries. **Treat this as a provenance flag for the bank's model-risk team, not a clear pass.** Newer Jina rerankers are excluded anyway (see below).

### Excluded (Chinese developer or Chinese base model)

| Model | Reason (source) |
|---|---|
| Qwen3-Reranker (qmd's default) | Alibaba (China). Listed as `qwen3-reranker` in the [qmd README](https://github.com/tobi/qmd). |
| `mixedbread-ai/mxbai-rerank-base-v2` / `large-v2` | `config.json`: `Qwen2ForCausalLM`, 151,936-token Qwen vocabulary. Fine-tune of Qwen2.5. |
| `jinaai/jina-reranker-v3` | `base_model: Qwen/Qwen3-0.6B`, and CC-BY-NC-4.0. |
| `answerdotai/answerai-colbert-small-v1` (Answer.AI, US) | Its initial weights are a weight average of `gte-small` (Alibaba) and `bge-small-en-v1.5` (BAAI), distilled from BGE-M3 and BGE Gemma2 rerankers ([Answer.AI blog](https://www.answer.ai/posts/2024-08-13-small-but-mighty-colbert.html)). It looks US-made but derives from Chinese models. |
| `lightonai/GTE-ModernColBERT-v1` (LightOn, France) | `base_model: Alibaba-NLP/gte-modernbert-base`. |
| BAAI `bge-reranker-*`, Alibaba `gte-reranker-*` | Chinese developers. |

## txtai integration

- **`Similarity(path, crossencode=True)`** wraps any Hugging Face `*ForSequenceClassification` reranker. It sends `{"text": query, "text_pair": doc}` pairs to a transformers `text-classification` pipeline and returns `(index, score)` sorted by score ([`pipeline/text/crossencoder.py`](https://github.com/neuml/txtai/blob/master/src/python/txtai/pipeline/text/crossencoder.py); [Similarity docs](https://neuml.github.io/txtai/pipeline/text/similarity/)). `lateencode=True` does the same for ColBERT models.
- **`Reranker(embeddings, similarity)`** runs `embeddings.batchsearch(query, limit * factor)` (factor defaults to 10), rescores `row["text"]` with the Similarity pipeline, and returns the top `limit`. It needs `content: true` ([`pipeline/text/reranker.py`](https://github.com/neuml/txtai/blob/master/src/python/txtai/pipeline/text/reranker.py); [Reranker docs](https://neuml.github.io/txtai/pipeline/text/reranker/)).
- **Gotcha:** `Reranker` passes the *same* query string to search and to the cross-encoder. Our MCP server will issue SQL (`... WHERE similar(:q) AND project = 'KAFKA' AND created >= ...`), and a cross-encoder should not score a SQL string. So in our MCP server, run the filtered SQL search ourselves, then call `Similarity(crossencode=True)(natural_language_query, texts)`. That is about 10 lines of code. This is the "seam" to build now.
- **Ollama:** Ollama has **no rerank endpoint**. The feature request [ollama#3368](https://github.com/ollama/ollama/issues/3368) (opened 2024-03) and PR [#14172](https://github.com/ollama/ollama/pull/14172) are both still open. Dedicated cross-encoders therefore run in-process with transformers (torch), not in Ollama. That adds torch (CPU wheel) to the stack. Only an LLM reranker would go through Ollama.

## LLM-as-reranker with gemma4:e4b

**Evidence:** RankGPT showed that "properly instructed LLMs can deliver competitive, even superior results to state-of-the-art supervised methods", but with ChatGPT/GPT-4-class models ([Sun et al., EMNLP 2023](https://arxiv.org/abs/2304.09542)). Déjean et al. found traditional cross-encoders "remain very competitive" against them ([arXiv:2403.10407](https://arxiv.org/abs/2403.10407)). I found no published reranking benchmark for a 4B-class Gemma model. Its quality relative to a dedicated cross-encoder is **unknown** and must be measured.

**Timing measured on this machine** (2026-11-15). AMD Ryzen 7 1800X (8 cores), 16 GB RAM, GTX 1070 Ti 8 GB, Ollama 0.40.15, gemma4:e4b fully on GPU (`ollama ps`: 100% GPU, 4,096 context). 50 synthetic Kafka/Flink-vocabulary candidates, model warm:

| Mode | Candidate length | Time for 50 |
|---|---|---|
| Pointwise: one call per candidate, `num_predict: 1`, read the `yes` logprob | ~40 words | 11.3 s |
| Pointwise | ~150 words | 15.9 s |
| Listwise: one prompt, output ranked ids | ~40 words (2,622 prompt tokens) | 7.4 s |
| Listwise | ~150 words | 8.7 s, **but invalid**: the prompt was cut to 2,051 tokens by Ollama's default 4,096 context |

- Pointwise works. The `logprobs`/`top_logprobs` fields came back as documented, so a calibrated score is `P(yes)`.
- Listwise needs `num_ctx` raised to about 16K for 50 artefact-length candidates. It also needs id-parsing and windowing as in RankGPT, and it gives no per-item score to blend with RRF.
- Cold load of the model took about 15 s.
- **CPU-only runs were not measured.** Expect them to be several times slower, because each pointwise call prefills about 100 to 300 tokens through a 4.5B-effective model.

**Verdict:** gemma4:e4b is a viable *fallback* reranker for small pools (20 to 30) when a GPU is present. It is not a good default for 50 to 100 candidates on a CPU-only 16 GB laptop. It also competes for the same Ollama instance that generates the answer.

## CPU latency for about 50 candidates (dedicated cross-encoders)

**Not measured.** A local benchmark run that would download and execute these models was blocked by the harness permission policy. Published anchors:

- **MiniLM-L6:** BEIR measured 6,100 ms on an 8-core Xeon 8168 for 100 short DBPedia candidates ([BEIR Table 3](https://arxiv.org/abs/2104.08663)). That is about 3 s per 50 on 2021 software. On a V100 GPU it scores 1,800 docs/s, about 30 ms per 50 ([model card](https://huggingface.co/cross-encoder/ms-marco-MiniLM-L6-v2)).
- **Sentence-transformers:** recommends ONNX-O3 (non-Intel) or OpenVINO / OpenVINO-qint8 (Intel) on CPU. It says to never use fp16 or bf16 on CPU ("roughly 0.17x") ([efficiency docs](https://sbert.net/docs/cross_encoder/usage/efficiency.html)).
- **Rough estimate,** scaling MiniLM-L6 by non-embedding parameters (about 11M):
  - base-class models (mxbai-base, granite-r2, about 85M to 110M) cost about 8 to 10 times as much: **roughly 5 to 25 s per 50 on CPU**, depending on candidate length (512 vs 8K tokens) and backend.
  - large models (about 300M) and 1B models: tens of seconds to minutes on CPU, so GPU only.
  - Treat these as order-of-magnitude figures only. Truncating candidates (for example to the best chunk, or to 512 tokens) cuts the cost roughly in proportion.

## Caveats

- **Provenance:** Mixedbread v1's training data and teacher are undisclosed, and Mixedbread's Berlin location comes from third-party profiles, not a first-party page. Jina has Chinese subsidiaries (now part of Elastic). Answer.AI's small ColBERT looks US-made but is built from GTE and BGE weights.
- **Licensing:** the ms-marco-* and ColBERTv2 weights are Apache/MIT, but MS MARCO's terms say "non-commercial research purposes only". jina-reranker-v2 and v3 are CC-BY-NC-4.0. Nemotron uses OpenMDW-1.1 plus the Llama 3.2 license. IBM Granite is the cleanest: Apache-2.0, and MS MARCO was deliberately avoided.
- **Evidence gaps:** there is no public benchmark on Jira, Slack or dev@ artefacts. Vendor results (Azure, Anthropic) use proprietary rerankers. The short-text evidence (Signal-1M) shows little gain.
