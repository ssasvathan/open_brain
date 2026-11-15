# Vector store or LLM-maintained wiki as the brain's core?

Research for issue #16 (parent map: #2). Researched 2026-11-14.

## Answer

Neither approach on its own. Keep the **indexed raw store as the core**: raw source items, each embedded and keyword-indexed in txtai, with metadata in SQL. Add **wiki-style topic pages as derived items** on top. Examples are a risk register per project and a decisions log per project. Each page is written as dated, append-only entries, and each entry cites the IDs of the raw items it came from.

The wiki pattern answers "what are the outstanding risks?" better than top-k similarity search does. But the people who proposed it still treat raw sources as immutable truth. Karpathy concedes you need search once the wiki grows past roughly a hundred sources. The ingest fan-out it needs (10 to 15 page edits per source) is the expensive part on a laptop running a 4B-class local model. A hybrid keeps the raw store cheap and auditable, and spends model time only on the few topic pages leadership actually asks about.

## What the proponents actually proposed

### Andrej Karpathy

**Tweet, "LLM Knowledge Bases" (2026-04-02)** ([x.com/karpathy/status/2039805659525644595](https://x.com/karpathy/status/2039805659525644595), read via the fxtwitter mirror):

- Source documents go into a `raw/` directory. An LLM "compiles" them into a wiki of Markdown files with summaries, backlinks and concept articles. Obsidian is the viewer, and he rarely edits the wiki by hand.
- At about **100 articles / ~400K words** the LLM answers complex questions without "fancy RAG", using auto-maintained index files and summaries.
- Outputs (Markdown, Marp slides, plots) get filed back into the wiki. LLM "health checks" find inconsistencies and fill gaps.
- He also built **a small search engine over the wiki** (web UI and CLI) for the LLM to use as a tool.

**Gist, "LLM Wiki" idea file (2026-04-04)** ([gist.github.com/karpathy/442a6bf555914893e9891c11519de94f](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)):

- **Three layers.** (1) Raw sources, which are *immutable* and the source of truth. The LLM reads them and never edits them. (2) The wiki, which the LLM owns. (3) A schema file (CLAUDE.md / AGENTS.md) that turns the agent into a "disciplined wiki maintainer".
- **Ingest.** Write a summary page, update `index.md`, update related entity and concept pages, and append to `log.md`. "A single source might touch 10-15 wiki pages."
- **Query.** Read `index.md` first, then the relevant pages, then synthesise an answer with citations. Good answers can be filed back in as pages.
- **Lint.** A periodic pass for contradictions, "stale claims that newer sources have superseded", orphans, and missing cross-references.
- **Staleness.** The only mechanism is to note contradictions at ingest and catch superseded claims at lint time. There is no explicit temporal model.
- **Scale.** The index-file approach "works surprisingly well at moderate scale (~100 sources, ~hundreds of pages)". Beyond that the gist recommends a search tool, and names **qmd** (local hybrid BM25 + vector + LLM re-ranking, with CLI and MCP server).
- **Provenance.** Answers cite wiki pages. The gist defines no formal scheme for tracing claims back to specific raw items.

The core claim is "compiled once and then kept current, not re-derived on every query." Embeddings are not rejected. They are optional search infrastructure once the wiki outgrows its index.

### Matt Pocock

I found no Pocock post, video or gist that proposes a wiki as a *general knowledge base*. His primary-source proposal is the **`chief-of-staff` skill** in his skills repo ([mattpocock/skills, `skills/in-progress/chief-of-staff/SKILL.md`](https://github.com/mattpocock/skills/blob/5863806dd4e15c63174bee4e7515a4256792ecfa/skills/in-progress/chief-of-staff/SKILL.md), last changed in commit `5ae48169`, 2026-10-31, "shared wiki first"). The skill is still marked in-progress. It tells a long-running coordinating agent to:

- "Keep your memory in a **wiki**: a folder of interlinked markdown pages that outlives this session", with `index.md` listing every page with a one-line summary. Each page covers one thing: a decision, a recurring problem, a subsystem, or a preference.
- "Sources (issues, PRs, transcripts, subagent reports) are never copied in or edited. Pages cite them."
- Use the same Ingest / Query / Lint operations as Karpathy. Lint looks for "claims newer sources have overtaken."
- **"Write history, not the present."** "Date every claim." Cite the commit a code claim was true at. Never write "currently" without a date. "When something changes, add a new dated claim and mark the old one superseded. Don't rewrite the old one."

Pocock's version adds the piece Karpathy leaves open: an explicit **supersession discipline** (dated, append-only claims with a superseded marker) and a strict rule that pages *cite* sources and never absorb them. The "timestamped entries" in the ticket's wording come from him, not from Karpathy. Note that his wiki is the agent's working memory at the scale of one goal, not a corpus index. Search does not come up in it.

## Other prior art

| Prior art | What it is | Relevance |
|---|---|---|
| **GraphRAG**, Microsoft Research ([arXiv 2404.16130](https://arxiv.org/abs/2404.16130), 2024) | An LLM builds an entity graph, then pre-generates "community summaries" for groups of related entities. Answers are map-reduced over those summaries. | Shows the core claim directly: "RAG fails on global questions directed at an entire text corpus". Evaluated on corpora of about 1M tokens. Pre-computed summaries are the wiki idea, done mechanically. |
| **RAPTOR**, Stanford ([arXiv 2401.18059](https://arxiv.org/abs/2401.18059), 2024) | Recursively embeds, clusters and summarises chunks into a tree, and retrieves at any level. | A hybrid: summaries are derived nodes inside the vector index. It addresses the fact that chunk retrieval only returns "short contiguous chunks". |
| **Generative Agents**, Stanford / Google ([arXiv 2304.03442](https://arxiv.org/abs/2304.03442), 2023) | A complete natural-language memory stream plus periodically synthesised higher-level "reflections", retrieved dynamically. | The original pattern of raw stream plus derived synthesis. Reflections are derived items over a retrievable raw log. |
| **Graphiti / Zep**, Zep AI, US ([repo](https://github.com/getzep/graphiti), [arXiv 2501.13956](https://arxiv.org/abs/2501.13956)) | A temporal knowledge graph. Each fact has a validity window. Old facts are "invalidated — not deleted". "Every derived fact traces back" to raw **episodes**. | The most rigorous model of staleness and provenance. But its README warns it "works best with LLM services that support Structured Output" and that schema failures are "particularly problematic when using smaller models". That is a risk for gemma4:e4b. |
| **qmd**, Tobi Lütke ([tobi/qmd](https://github.com/tobi/qmd)) | The local Markdown search engine Karpathy recommends: BM25 + vector + RRF + LLM re-ranking, with CLI and MCP. | Shows that wiki users end up adding hybrid search. **Its default models cannot be used here**: the reranker is `qwen3-reranker-0.6b` (Qwen, Alibaba, China), and the query-expansion model is a fine-tune listed beside it. Its default embedder `embeddinggemma-300M` is from Google (US). The ideas carry over (typed lex/vec queries, context trees). The defaults do not. |

## Comparison

| Axis | Vector / hybrid store over raw items | LLM-maintained wiki | Hybrid (recommended) |
|---|---|---|---|
| **Cross-team synthesis** ("outstanding risks", "finalised decisions") | Weak when used alone. Top-k chunks can't enumerate every risk across Kafka and Flink, which is GraphRAG's "global question" failure. Metadata filters help (txtai SQL: `WHERE similar(...) AND project='KAFKA' AND date >= ...`), but the synthesis still happens at read time over many items. | Strong. A `risks.md` page already holds the list, so the query is "read one page". This is the pattern's main advantage. | Topic pages (risk register, decisions log) answer the enumerating questions. Raw search fills in detail and covers topics with no page yet. |
| **Staleness / supersession** | No built-in notion. Old and new items rank equally unless queries filter or weight by date. Freshness has to be a read-time rule. | Karpathy: lint for superseded claims, with no mechanism. Pocock: dated, append-only claims, old ones marked superseded, never rewritten. That is good, but it depends on the agent noticing. | Use Pocock's discipline on derived pages (dated entries, `superseded_by`). Raw items stay immutable with source timestamps. A Graphiti-style validity window is the formal version. |
| **Ingest cost on a laptop** | Cheap and predictable. Per item: one nomic-embed-text call (~2K-token context, so long threads need chunking) plus one gemma4:e4b tagging call. Linear in item count. | Expensive. Karpathy's own estimate is 10–15 page edits per source. Each edit is a long-context read-and-rewrite by the local model. A Kafka+Flink half-year (thousands of Jira issues and dev@ messages, my estimate) multiplies that. Rewrite quality from a 4B-class model is unverified. | Raw ingest stays cheap. Derived-page updates are limited to the handful of pages a tagged item touches (for example, only items tagged `risk` or `decision` update the register), batched and run off-peak. |
| **Scale limits** | Scales to large corpora. This is what the index is for. | Index-file navigation works at "~100 sources, ~hundreds of pages" (gist) or ~400K words (tweet). Past that, both Karpathy and qmd add search. Large pages also overflow a local model's context. | The page count stays small (one page per topic per project), and search handles the long tail. |
| **Auditability to raw item** | Direct. Every hit *is* a raw item with its source URL and ID. | Indirect. Answers cite wiki pages, and the gist defines no trace from claim to raw item. Pocock requires pages to cite sources, which is necessary but not enforced. LLM rewrites can drift from what the sources say. | Every derived entry must carry raw-item IDs (Graphiti's "episodes" idea). The answer cites the entry *and* the raw item, so a derived claim can be checked mechanically. |
| **Failure mode** | Incomplete answers (missed items). | Confident, stale or drifted answers (a wrong page is read as truth). | A wrong derived entry can be caught against the raw item it cites. Pages can be regenerated from the raw store. |

## Hybrids, concretely

The cited prior art converges on the same shape: an **immutable raw layer as truth** (Karpathy's `raw/`, Pocock's uncopied sources, Graphiti's episodes, Generative Agents' memory stream) plus a **derived synthesis layer** (wiki pages, community summaries, RAPTOR nodes, reflections). They differ only in how the derived layer is found and kept fresh. For Open Brain that means:

1. **Unit of storage stays the raw source item.** Each item is indexed in txtai (vector + BM25 + SQL metadata). This is the map's existing "raw items are truth, optional linked derived items".
2. **A wiki page is a derived item type.** It gets a stable ID per topic (for example `kafka/risk-register`), it is indexed like any other item so search finds it, and its body is dated entries, each citing raw-item IDs and optionally marked superseded.
3. **Write-time vs read-time.** Start with read-time synthesis plus *on-demand* page materialisation: when an executive-report query runs, save the result as a dated derived page. Move to write-time page maintenance only for topics that get asked repeatedly, and only if a 4B-class local model's page edits prove accurate. This is Karpathy's "file good answers back" used as the default, with ingest-time curation as an optimisation.

## Caveats

- Karpathy's sources are a tweet and an idea file. They describe a personal research workflow at ~100 sources, not a multi-team operational corpus.
- Pocock's skill is marked `in-progress` and is aimed at agent working memory, not a searchable corpus.
- The Kafka+Flink volume and the per-item edit cost for gemma4:e4b are estimates. A small benchmark (ingest 200 items both ways and time them) would settle the laptop-cost question.
- No local-model figures were measured for this note.
