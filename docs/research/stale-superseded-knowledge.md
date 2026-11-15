# How AI memory systems handle stale and superseded knowledge

Research for issue #15. Researched 2026-11-15.

**Question.** A decision recorded last month is revised this month. What should a question asked today get back? How does a system answer "what changed since last week"? This note surveys what production AI-memory and RAG systems actually do. It judges each pattern against this brain's decided design.

**Decided context (not re-opened here).** txtai as a library behind our own MCP server. Ollama runs nomic-embed-text (Nomic AI, US) and gemma4:e4b (Google, US, 4B-class). The stored unit is an **artefact** with immutable **revisions**. Outputs are **syntheses**. Topic pages hold dated, append-only **entries** that get superseded, never rewritten. Writes are idempotent upserts keyed by source. No models from Chinese developers.

## Summary

- Every serious system **keeps** the old fact and marks it no longer current. None quietly overwrites it any more. Zep/Graphiti invalidates graph edges. Mem0's graph variant marks relationships invalid. Mem0 itself has since moved to ADD-only extraction.
- The systems split on **who decides that something is superseded**. Zep, Graphiti and Mem0 ask an LLM at ingest. srnichols/OpenBrain makes the caller supply a `supersedes` UUID. Karpathy's "LLM wiki" uses a periodic lint pass.
- All the LLM-at-ingest designs were built and tested with strong hosted models (gpt-4o-mini or better). Graphiti's README warns that very small models break its extraction.
- Our sources hand us explicit supersession signals for free: Jira status/resolution changes, KIP/FLIP status, Slack edits. Those are cheaper and more reliable than any LLM judgement a 4B model can make.
- "What changed since X" is answered in one of two ways. Either a timestamp filter over an append-only record (Graphiti's `created_at`/`expired_at` filters, Mem0's `history` table), or the source's own change log (Jira `CHANGED`). Nobody computes it by diffing embeddings.

## 1. Per-item timestamps and recency weighting at retrieval

**What people do.** The canonical recipe comes from the Generative Agents paper (Park et al., Stanford/Google). Retrieval score = recency + importance + relevance. Recency is an exponential decay (factor 0.995 per sandbox hour since the memory was last retrieved). Importance is an LLM-assigned 1–10 integer. Relevance is cosine similarity. Each is min-max scaled to [0,1], and the weights are all set to 1 ([arXiv 2304.03442, §4.1](https://arxiv.org/html/2304.03442)).

**What txtai gives us.** txtai's SQLite content store writes an `entry DATETIME` column on every document ([`statement.py`](https://github.com/neuml/txtai/blob/973512fc0d30250613f7571a7b6446101aa9b92e/src/python/txtai/database/schema/statement.py)). It sets `entry` to `datetime.now(utc)` at insert time ([`rdbms.py`](https://github.com/neuml/txtai/blob/973512fc0d30250613f7571a7b6446101aa9b92e/src/python/txtai/database/rdbms.py)). Documents are written with `INSERT OR REPLACE`. So `entry` is the time of the *last write*, not when the event happened. You can filter it in SQL alongside `similar()`, e.g. `where similar('...') and entry >= date('now','-1 day')`, or filter on your own metadata date columns ([txtai query guide](https://neuml.github.io/txtai/embeddings/query/)).

**Trade-offs for this brain.**
- Cheap and model-free. It works with a 4B model or none at all.
- Recency is a *proxy* for currency. It cannot tell "revised decision" from "fresh chatter that mentions the old decision". A re-scraped Slack thread that only gained an emoji would look new if we used ingest time.
- So we must store our own source-event timestamps (Jira `updated`, Slack `ts`/`latest_reply`, page last-modified) as metadata columns. txtai's `entry` must not serve as the recency signal.
- Decay weighting is a ranking tweak. It does not *remove* superseded content from answers. Use it as a tiebreaker, not as the supersession mechanism.

## 2. Bi-temporal / valid-time models (Zep, Graphiti)

**What Zep/Graphiti do.** Zep "implements a bi-temporal model". Timeline T orders events. Timeline T′ records ingestion. Each edge carries `t_valid`/`t_invalid` (when the fact held true) and `t′_created`/`t′_expired` (when the system learned or retired it) ([Zep paper, arXiv 2501.13956, §2.1, §2.2.3](https://arxiv.org/html/2501.13956v1)). On ingest, "the system employs an LLM to compare new edges against semantically related existing edges". If they contradict, "it invalidates the affected edges by setting their t_invalid to the t_valid of the invalidating edge" (§2.2.3). Relative dates ("two weeks ago") are resolved against a per-episode reference timestamp (§2.1). The README summary: "When information changes, old facts are invalidated — not deleted", and you can "query what's true now, or what was true at any point in time" ([Graphiti README](https://github.com/getzep/graphiti)). Search exposes filters on `valid_at`, `invalid_at`, `created_at` and `expired_at` ([`search_filters.py`](https://github.com/getzep/graphiti/blob/3a8a78c997a07cceeb3411ede533b0f342d71c64/graphiti_core/search/search_filters.py)). Those filters answer both "as of date D" and "learned since D".

**Lessons from the source.** Graphiti's current contradiction prompt has grown defensive. It says: "Do NOT infer contradiction from recency alone, episode dates, meeting dates, weekly snapshots, section headings such as 'updates' or 'status changes'". It also says not to treat progress or status facts as contradicting ownership facts ([`dedupe_edges.py`](https://github.com/getzep/graphiti/blob/3a8a78c997a07cceeb3411ede533b0f342d71c64/graphiti_core/prompts/dedupe_edges.py)). Those are exactly the false positives repeated Slack/Jira scrapes would cause. The date arithmetic that decides which edge expires is deterministic code, not LLM output ([`edge_operations.py`, `_apply_temporal_invalidation`](https://github.com/getzep/graphiti/blob/3a8a78c997a07cceeb3411ede533b0f342d71c64/graphiti_core/utils/maintenance/edge_operations.py)).

**Model dependence and provenance.**
- "Graphiti works best with LLM services that support Structured Output (such as OpenAI, Anthropic, and Gemini). Using other services may result in incorrect output schemas and ingestion failures." Also: "Very small models frequently emit JSON that doesn't match the requested schema" ([README](https://github.com/getzep/graphiti)).
- **Flag:** the README's Ollama example uses `deepseek-r1:7b` (DeepSeek, China) as the LLM. That is excluded for us. Its embedder there is `nomic-embed-text`, which is fine.
- **Flag:** the Zep paper's evaluation used "BGE-m3 models from BAAI" (Beijing Academy of AI, China) for embedding and reranking, and gpt-4o-mini (OpenAI, US) for graph construction (§4.1). Its published accuracy numbers therefore depend on an excluded model family and a far stronger LLM than gemma4:e4b.

**Trade-offs for this brain.**
- The *data model* is the valuable part, and we can adopt it without Graphiti. Put valid time on syntheses/entries ("decision in force from D1"). Use our revision timestamps as transaction time.
- *Fact-level* extraction plus LLM contradiction checks is the fragile part. It needs triple extraction, entity resolution and contradiction judgement on every ingest. Each is a strong-LLM task. A 4B model will both miss contradictions and invent them.
- Our granularity is coarser, and that helps. An artefact revision is already a transaction-time record. A topic-page entry is already a dated claim. The question shrinks from "which atomic facts conflict" to "does entry B supersede entry A on this topic page". That decision is smaller and easier to review.

## 3. Explicit supersedes links and versioning

**srnichols/OpenBrain.** The `capture_thought` call accepts `supersedes`, a "UUID of a prior thought this replaces". The schema has `supersedes UUID REFERENCES thoughts(id)` with a B-tree index for "thought chain traversal". The stated motive: "Decisions get superseded; stale guidance misleads AI agents" ([README](https://github.com/srnichols/OpenBrain)). The caller sets the link by hand; the README says no LLM detects it. Its separate update path edits content *in place* and re-embeds, with only `created_at`/`updated_at` kept. So there is no revision history. The README does not say whether search hides or down-ranks superseded thoughts. Only `archived` is filtered by default. Defaults: `nomic-embed-text` and `llama3.2` (Meta, US) via Ollama.

**Letta.** Letta's current memory (MemFS) is a git repository of markdown files. The agent edits files and commits, so versioning is git history ([Letta MemFS docs](https://docs.letta.com/concepts/memfs)). There is no structured supersedes field. Supersession is an in-place edit, and the old text survives only in git history.

**FLIP/KIP documents supersede themselves only partly.** Apache Flink says "A FLIP records the design at the time it was accepted. The FLIP page does not need to be updated for later changes" ([Flink Improvement Proposals](https://cwiki.apache.org/confluence/display/FLINK/Flink+Improvement+Proposals)). An Accepted FLIP can therefore be stale about implementation details while its *status* stays accurate.

**Trade-offs for this brain.**
- An explicit `supersedes` link on an entry fits our append-only topic pages exactly. It is deterministic, auditable and cheap at query time ("exclude entries that have a successor").
- The open question is who sets it. Hand-set links (srnichols) don't scale to scrape volume. Source signals (§6) cover many cases. An LLM *proposal* that the brain owner confirms covers the rest.
- Don't copy srnichols' in-place update. Our immutable revisions already give the history it lacks.

## 4. LLM-driven memory updates: add/update/delete/merge (Mem0, Letta/MemGPT)

**Mem0 as published.** For each candidate fact, Mem0 retrieves the top *s* similar memories (*s* = 10). An LLM, via a tool call, picks ADD, UPDATE, DELETE ("removal of memories contradicted by new information") or NOOP. The graph variant Mem0g uses "an LLM-based update resolver" that marks conflicting relationships "as invalid rather than physically removing them to enable temporal reasoning". All of it runs on GPT-4o-mini (OpenAI, US) ([Mem0 paper, arXiv 2504.19413, §2.1–2.2](https://arxiv.org/html/2504.19413v1)).

**Mem0 today: it walked back destructive updates.** The migration guide says: "The previous algorithm could UPDATE or DELETE existing memories during extraction. The new algorithm only adds new facts". When a fact changes, "both facts are preserved with temporal context", and "your memory count will grow rather than plateau" ([Mem0 migration guide](https://docs.mem0.ai/migration/)). The open-source code matches. The `ADDITIVE_EXTRACTION_PROMPT` says "Your sole operation is ADD". It asks the model to put related existing memory IDs in `linked_memory_ids` for "updated/shifted preference" and to "capture the transition — what the new state is AND what it replaces". It grounds relative dates in an "Observation Date", not today's date ([`prompts.py`](https://github.com/mem0ai/mem0/blob/f1791daa86d469249c9b002f62644d58a1eb96a0/mem0/configs/prompts.py)). Explicit `update`/`delete` API calls still write old→new rows to a SQLite `history` table (`memory_id, old_memory, new_memory, event, created_at, updated_at, is_deleted`) ([`storage.py`](https://github.com/mem0ai/mem0/blob/f1791daa86d469249c9b002f62644d58a1eb96a0/mem0/memory/storage.py), [`main.py`](https://github.com/mem0ai/mem0/blob/f1791daa86d469249c9b002f62644d58a1eb96a0/mem0/memory/main.py)). Memories can carry an `expiration_date` that hides them from search after that day (`main.py`). The OSS defaults are OpenAI `gpt-5-mini` and `text-embedding-3-small`.

**Letta/MemGPT.** MemGPT's idea is "virtual context management" across memory tiers ([arXiv 2310.08560](https://arxiv.org/abs/2310.08560)). Core memory blocks are "editable by agents via memory tools" ([Letta memory guide](https://docs.letta.com/guides/agents/memory)). Archival memory is append-mostly: "Agents cannot easily modify or delete archival memories (though developers can via SDK)" ([Letta archival memory](https://docs.letta.com/guides/agents/archival-memory)). Background "dreaming" subagents "review recent conversations, consolidate useful lessons, and update memory". An optional second pass reviews the proposed updates ([Letta sleep-time docs](https://docs.letta.com/guides/agents/sleep-time-agents)).

**Trade-offs for this brain.**
- An LLM-decided DELETE/UPDATE is lossy and hard to audit. The vendor that made it popular has backed away to ADD-only plus links. That matches our "entries are never rewritten" rule.
- Mem0's ADD-only design pushes the "which is current" question to retrieval time. Mem0's own evaluation page says "Knowledge update (93.6) remains the hardest category for an additive, ADD-only architecture" ([Mem0 memory evaluation](https://docs.mem0.ai/core-concepts/memory-evaluation)). We should not rely on retrieval alone. We should also store an explicit supersession link.
- Every variant assumes a GPT-4o-mini-class model makes the call. gemma4:e4b can do the *mechanical* parts under schema-constrained decoding: tag entities, extract status, ground relative dates. Ollama enforces a JSON schema via `format` and recommends temperature 0 ([Ollama structured outputs](https://docs.ollama.com/capabilities/structured-outputs)). A schema guarantees shape, not judgement. "Does this supersede that?" should be a *proposal*, not an automatic write.

## 5. Idempotent upserts keyed by source vs append-only logs

**What txtai does.** `upsert`: "If the index exists, new data is appended to the index, existing data is updated" ([txtai methods](https://neuml.github.io/txtai/embeddings/methods/)). Under the hood that is `INSERT OR REPLACE` on `id` ([`statement.py`](https://github.com/neuml/txtai/blob/973512fc0d30250613f7571a7b6446101aa9b92e/src/python/txtai/database/schema/statement.py)). So a txtai index keyed by artefact ID holds **only the latest content** and resets `entry` on each write. It is a "current state" view with no history.

**Pattern in the surveyed systems.** All of them pair a mutable current view with an append-only record. Graphiti keeps expired edges plus episodes. Mem0 keeps the vector store plus its `history` table. Letta keeps blocks plus git. srnichols is the exception, and it loses history on update.

**Trade-offs for this brain.**
- Upserts keyed by source make re-scrapes idempotent. That matters for repeated Slack/Jira pulls. But history must be kept somewhere else.
- Recommended split: (a) an append-only `revisions` table (artefact_id, revision_no, content_hash, source_updated_at, captured_at, collector), written only when the hash changes; (b) the txtai index keyed by `artefact_id` for the current revision, plus optionally `artefact_id@rev` documents if old revisions must stay searchable. Skipping writes on an unchanged hash also stops txtai's `entry` from churning.
- Pure append-only logs (one document per scrape) would flood search with near-duplicates of the same thread and need dedup at query time. Avoid them.

## 6. How supersession gets detected

| Detector | Who uses it | Needs strong LLM? | Fit here |
|---|---|---|---|
| LLM contradiction check at ingest | Zep/Graphiti, Mem0 (old), Mem0g | Yes (all built and tested on gpt-4o-mini or better) | Poor as an auto-write. OK as a proposal |
| LLM-written links, ADD-only | Mem0 (current) | Moderate | Fair, as a suggestion |
| Periodic lint pass | Karpathy's LLM-wiki gist: lint for "contradictions between pages, stale claims that newer sources have superseded" ([gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)) | Moderate. Batch, so it can retry or use a bigger model offline | Good for topic pages. Output = proposed supersession for review |
| Explicit caller link | srnichols/OpenBrain `supersedes` | No | Good for owner-written entries |
| Source signals | Jira, KIP/FLIP, Slack (below) | No | **Best first line** |

**Source signals we can read deterministically.**
- **Jira.** "A work item resolution is usually set when the status is changed". A filled **Resolution** marks the item resolved ([Atlassian: statuses, priorities, resolutions](https://support.atlassian.com/jira-cloud-administration/docs/what-are-issue-statuses-priorities-and-resolutions/)). JQL `CHANGED` finds issues where a field changed, with `AFTER/BEFORE/DURING/ON/BY/FROM/TO` predicates. It works only on Assignee, Fix Version, Priority, Reporter, Resolution and Status ([JQL operators reference](https://confluence.atlassian.com/jirasoftwareserver/advanced-searching-operators-reference-939938745.html)). The issue changelog ([REST v3 Issues](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/)) gives per-field from/to history for diffing.
- **KIP/FLIP.** After a vote the author must "update the FLIP wiki page to reflect the current stage … This acts as the permanent record indicating the result". The index has sections Under discussion / Accepted / Released / Discarded ([Flink FLIP process](https://cwiki.apache.org/confluence/display/FLINK/Flink+Improvement+Proposals)). The Kafka index uses Adopted / Under discussion / Discarded sections and title prefixes like "[DISCARDED]" ([Kafka KIPs](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals)). A status move to Discarded supersedes the proposal itself. A move to Accepted supersedes earlier "under discussion" entries.
- **Slack.** Edited messages fire `message_changed`, carrying the updated `message` with an `edited` {user, ts} object ([Slack `message_changed`](https://docs.slack.dev/reference/events/message/message_changed)). `conversations.replies` accepts `oldest`/`latest` Unix-timestamp bounds ([Slack conversations.replies](https://docs.slack.dev/reference/methods/conversations.replies)). Slack has *no* structured "decision revised" signal. Revisions inside a thread need either the owner or an LLM proposal.

## 7. Answering "what changed since <date>"

- **Valid/transaction-time filters.** Graphiti filters edges on `created_at`/`expired_at`/`valid_at`/`invalid_at` (§2 above). For us: `WHERE captured_at >= :d` on revisions, plus entries whose `valid_from >= :d`, plus entries whose `superseded_at >= :d`.
- **Change log / history table.** Mem0 reads its `history` table (old_memory → new_memory, event, timestamp). For us, the revisions table *is* the change log. Jira's changelog and JQL `status CHANGED AFTER -7d` give a source-native version for Jira.
- **Diff over revisions.** No surveyed system computes semantic diffs. But with immutable revisions we can diff text for revision n−1 vs n, then have gemma4:e4b *summarise* the diff. Summarising a given diff is a much easier task for a small model than spotting contradictions across the corpus.
- **Today's question.** Default retrieval should exclude entries with a successor and rank by source event time. It should cite the superseded entry only when asked for history. That is Graphiti's "what's true now" default, done with a SQL filter in txtai.

## Recommendation for this brain

Keep the decided artefact/revision/entry model. Add three small things: a source-event timestamp column, an append-only revisions table, and an explicit `superseded_by` link plus `valid_from` on entries. Detect supersession first from source signals (Jira status/resolution changes, KIP/FLIP status, Slack edits). For the rest, have a periodic lint pass with gemma4:e4b *propose* supersession links for the brain owner to confirm, never auto-apply them. Answer "what changed since D" with SQL over revisions and supersession timestamps, plus Jira's changelog, with the 4B model only summarising concrete diffs.

## Caveats

- **Model provenance.** Graphiti's README Ollama example uses `deepseek-r1:7b` (DeepSeek, China): excluded. The Zep paper's results used BGE-m3 (BAAI, China): excluded, and its numbers don't transfer. Mem0 and srnichols default to US models (OpenAI; Meta `llama3.2`; Nomic `nomic-embed-text`).
- **Small-model failure modes.** Graphiti warns that small models emit off-schema JSON. Ollama's schema-constrained `format` fixes shape but not judgement. Expect a 4B model to over-flag contradictions on status updates and weekly snapshots. Graphiti's prompt now explicitly forbids those cases, which shows even strong models got them wrong.
- **Mem0 evidence.** The ADD-only switch is documented on the Mem0 Platform migration page and visible in OSS `main` at the commit cited. The paper describes the older ADD/UPDATE/DELETE/NOOP algorithm.
- **txtai `entry`.** Behaviour comes from reading source at the cited commit. The docs don't define it.
- **Not verified.** The exact Jira changelog response shape (the REST page would not render for fetching). The precise meaning of Kafka's "Adopted" vs "Accepted" labels.
