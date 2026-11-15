# OpenBrain-family references: how they model, ingest and retrieve memories

Research for issue #5 (map: #2). Sources were read on 2026-11-14:

- **OB1**: `NateBJones-Projects/OB1` @ `238df6c` (2026-09-10). This is Nate B. Jones's canonical "Open Brain".
- **srnichols**: `srnichols/OpenBrain` @ `4e4e020` (2026-08-20). A self-hosted fork that "started from Nate B Jones' Open Brain concept but has diverged significantly" (README, "Differences from Nate's Original").
- **MonkeyRun**: the blog post "One brain, every AI: building an MCP server" (Joan, 2026-03-04). It describes a build "based on Nate B. Jones's architecture".

The path citations below are relative to each repo root.

## TL;DR

All three share the same core design. The stored unit is one Postgres `thoughts` row: free-text `content`, one pgvector `embedding` of the *whole* content, and a `metadata` JSONB. Search is cosine similarity over an HNSW index. On capture, a small LLM tags each thought with type, topics, people, action items and dates. That call runs in parallel with the embedding call, and failures are tolerated. The MCP surface is small and thought-shaped: capture, search, list and stats, with update and delete added in later variants.

Every reference assumes **personal, hand-written, atomic "thoughts"**. None chunks long content in its core path. Bulk or automated sources are bolted on through separate importers, deduplication recipes or ranking weights.

Only srnichols runs fully locally out of the box: Docker, Postgres 17 + pgvector, Ollama `nomic-embed-text` and `llama3.2`, MIT licence. OB1 is cloud-first (Supabase plus OpenRouter) and licensed FSL-1.1-MIT. It has a local recipe, but that recipe drops MCP. MonkeyRun is cloud-only, and its linked code repo returns 404.

## Comparison

| | OB1 (canonical) | srnichols/OpenBrain | MonkeyRun post |
|---|---|---|---|
| Stored unit | `thoughts` row | `thoughts` row | `thoughts` row |
| Core columns | `id, content, embedding vector(1536), metadata jsonb, created_at, updated_at`, plus `content_fingerprint` | `id, content, embedding vector(768), metadata, project, created_by, archived, supersedes→thoughts(id), created_at, updated_at`, plus generated `source_file_hash`, `code_hash` | `embedding`, `weight` (other columns not described) |
| Embedding | OpenRouter `openai/text-embedding-3-small` (1536) | Ollama `nomic-embed-text` (768) by default; OpenRouter or Azure OpenAI optional | OpenRouter `text-embedding-3-small` |
| Capture tagging LLM | OpenRouter `openai/gpt-4o-mini`, JSON mode | Ollama `llama3.2` (`format: json`) by default; `gpt-4o-mini` via OpenRouter or Azure | None described; `weight` set by source |
| Tag fields | `people, action_items, dates_mentioned, topics (1-3), type ∈ {observation, task, idea, reference, person_note}` | `type` (13 values), `topics (1-3), people, action_items, dates` | n/a |
| Core MCP tools | `search`, `fetch` (ChatGPT compat), `search_thoughts`, `list_thoughts`, `thought_stats`, `capture_thought` | `search_thoughts`, `list_thoughts`, `capture_thought`, `capture_thoughts` (batch), `thought_stats`, `update_thought`, `delete_thought` | `search_thoughts` |
| Runtime / transport | Deno Supabase Edge Function, Hono + `@hono/mcp` Streamable HTTP | Node 22 TS, Hono REST :8000, MCP over SSE :8080 (plus stdio entry) | Deno Supabase Edge Function |
| Auth | single `MCP_ACCESS_KEY` via `x-brain-key` header or `?key=` | named per-client keys or a bootstrap `MCP_ACCESS_KEY`; `?key=` off by default | static API key in header or query |
| Fully local? | Not canonically; `recipes/local-brain-no-mcp` self-hosts Supabase + Ollama but exposes HTTP, not MCP | Yes (`docker compose up`) | No |
| Licence | FSL-1.1-MIT (source-available; converts to MIT later) | MIT | Not stated; repo 404 |

## OB1 (NateBJones-Projects/OB1)

**Stored unit and schema.** The setup guide creates `thoughts(id uuid, content text not null, embedding vector(1536), metadata jsonb default '{}', created_at, updated_at)`. It adds an HNSW `vector_cosine_ops` index and a GIN index on `metadata` (`docs/01-getting-started.md` §2). A later step adds `content_fingerprint TEXT` with a partial unique index and an `upsert_thought(p_content, p_payload)` RPC. When the same content is captured again, that RPC *merges metadata* instead of inserting a new row (same file, ~L191-226). The guide describes it as storing "your thoughts as raw text, vector embeddings, and structured metadata".

**Capture and tagging.** `capture_thought` takes a single `content` string, described as "a clear, standalone statement that will make sense when retrieved later". It runs `getEmbedding` and `extractMetadata` in parallel via `Promise.all`, then calls `upsert_thought` with `metadata: {...metadata, source: "mcp"}` (`server/index.ts` ~L439-497). Embeddings use `openai/text-embedding-3-small`. Metadata comes from `openai/gpt-4o-mini` with `response_format: json_object`, both through OpenRouter (`server/index.ts` L14, L54, L74). The prompt asks for `people`, `action_items`, `dates_mentioned`, `topics` ("1-3 short topic tags (always at least one)") and `type` (one of `observation, task, idea, reference, person_note`). It also says "Only extract what's explicitly there." If parsing fails, the code falls back to `{topics:["uncategorized"], type:"observation"}` (L76-96). The guide calls this metadata "best-effort … The embedding is what powers semantic search … The metadata is a bonus layer" (`docs/01-getting-started.md` ~L900-913).

**Retrieval.** `match_thoughts(query_embedding, match_threshold, match_count, filter jsonb)` returns rows where `1 - (embedding <=> q) > threshold` and `metadata @> filter`, ordered by distance (`docs/01-getting-started.md` §2.3). There is no keyword or hybrid search in the core. The optional schemas add more:
- `schemas/enhanced-thoughts`: top-level `type, sensitivity_tier, importance, quality_score, source_type, enriched`, plus a tsvector-then-ILIKE `search_thoughts_text`.
- `schemas/text-search-trgm`: a trigram index for that ILIKE fallback.
- `schemas/recency-boosted-match-thoughts`: `match_thoughts_recency`, which blends cosine similarity with exponential recency decay via `recency_weight` and `half_life_days`.
- `schemas/entity-extraction`: an entity and relationship queue.
- `schemas/provenance-chains`: links derived artifacts such as digests back to source thoughts.

**MCP tools (core server).** `search`, `fetch` (read-only, for ChatGPT compatibility), `search_thoughts`, `list_thoughts` (type, topic, person and days filters), `thought_stats` and `capture_thought` (`server/index.ts` L109-440). Core has no update or delete. Those come as separate integrations: `integrations/update-thought-mcp`, `integrations/delete-thought-mcp`, and `integrations/enhanced-mcp`, which adds `get_thought, update_thought, search_thoughts_text, count_thoughts, related_thoughts, graph_search, entity_detail, ops_*`.

**Stack and hosting.** Supabase (Postgres + pgvector + Edge Functions) plus OpenRouter. The server is a Deno Edge Function using Hono and `StreamableHTTPTransport` from `@hono/mcp`. Auth is one shared key, compared against `x-brain-key` or `?key=` (`server/index.ts` L4-12, L596-606). The repo's `CLAUDE.md` makes this a rule: "MCP servers must be remote (Supabase Edge Functions), not local. Never use … `StdioServerTransport`, or local Node.js servers."

**Can it run locally?** Not in canonical form. Two recipes come close:
- `recipes/local-brain-no-mcp` self-hosts the full Supabase docker stack with an Ollama sidecar. It uses 768-dim embeddings by default and needs about 8 GB RAM. It exposes `capture`/`search`/`list` as plain HTTPS Edge Functions called with `curl` through `skills/ob1-local-http`. It explicitly removes MCP and calls itself a "deliberate exception to canonical OB1".
- `recipes/local-ollama-embeddings` replaces only the embedding step and still writes to Supabase.

**Licence.** `LICENSE.md` is FSL-1.1-MIT. Any "Permitted Purpose" is allowed, except a Competing Use: a commercial product that substitutes for or replicates the software. "Internal use and access" is explicitly permitted.

**Personal-thought assumption.** It is pervasive. The README calls it "persistent memory of you". The tool description says "a clear, standalone statement". The type vocabulary is personal (`person_note`, `idea`). The built-in extensions are household, meal planning, family calendar, CRM and job hunt. Slack appears only as *quick capture*: "Type a thought in a Slack channel" (`integrations/slack-capture/README.md`). It is not channel scraping.

Bulk sources come through community importers in `recipes/*-import`. The OB1 parts most relevant to scraped digests:
- `docs/ingestion-metadata-contract.md` requires importers to write `source_type, source_label, imported_at, importer_name/version, input_hash, content_fingerprint, sensitivity_tier, provenance`, and optionally `source_id, source_url, original_created_at`, so re-runs are safe and sources can be audited or deleted.
- `integrations/smart-ingest` uses an LLM to split raw text into atomic thoughts. It then reconciles each one against existing thoughts by fingerprint or semantic similarity: `skip` at ≥0.92, `append_evidence` or `create_revision` at ≥0.85, otherwise `add`.

## srnichols/OpenBrain

**Stored unit and schema.** `db/init.sql` defines `thoughts(id uuid, content text not null, embedding VECTOR(768), metadata jsonb, project text, created_by text, archived boolean default false, supersedes uuid references thoughts(id), created_at, updated_at)`. It has HNSW cosine and GIN metadata indexes, plus B-tree indexes on `created_at`, `project`, `created_by` and `supersedes`, and a partial index on non-archived rows. Migration 003 adds generated columns `source_file_hash`/`code_hash` from `metadata->'provenance'` and a `match_thoughts_by_source` RPC (`db/migrations/003-add-provenance-helpers.sql`).

**Capture and tagging.** `capture_thought` and the batch `capture_thoughts` run `embedder.generateEmbedding` and `embedder.extractMetadata` in parallel for each item. They merge the result with caller metadata and record `source` (default `'mcp'`) and the transport (`src/mcp/server.ts`, the capture cases). `update_thought` re-embeds and re-extracts. The embedder can be Ollama, OpenRouter or Azure OpenAI (`src/embedder/*`). Defaults are Ollama `nomic-embed-text` for embeddings and `llama3.2` for tagging (`config/settings.yaml`, `.env.example`). The extraction prompt is `METADATA_PROMPT` in `src/embedder/types.ts`. It returns `type` (13 values: the original 7 personal types plus `architecture, pattern, postmortem, requirement, bug, convention`), `topics` (1-3, lowercase hyphenated), `people`, `action_items` and `dates`. Failures fall back to `DEFAULT_METADATA` (`type: observation`). Note: `config/settings.yaml` still carries an older 7-type prompt, but the code uses the 13-type one.

**No chunking.** If content is longer than the embedder's safe byte budget, the full text is still stored and only the first ~N bytes are embedded. The row is flagged `embedding_truncated`, and a warning tells the caller to "Split this into smaller captures (e.g. one per H3 section)" (`src/api/validation.ts` ~L220-240).

**Retrieval.** `match_thoughts(query_embedding, match_threshold=0.5, match_count=10, filter jsonb, project_filter, include_archived=false, user_filter)` filters by cosine threshold, metadata containment, project, archived flag and creator (`db/init.sql`). There is no full-text or hybrid search. `supersedes` links decisions and `archived` soft-hides rows, but superseded rows are *not* excluded from search automatically. Only `archived` is.

**MCP tools.** `search_thoughts` (query, limit, threshold, project, type, topic, include_archived, created_by), `list_thoughts` (type, topic, person, days, project, …), `capture_thought` (content, project, source, supersedes, created_by), `capture_thoughts` (batch), `thought_stats`, `update_thought` and `delete_thought` (`src/mcp/server.ts` L53-253). A REST API (Hono, :8000) mirrors these.

**Stack and hosting.** Node 22 + TypeScript, Postgres 17 + pgvector (`pgvector/pgvector:pg17`) and Ollama. MCP runs over SSE on a raw Node HTTP server at :8080 (`src/index.ts`), and a stdio entry exists too (`src/mcp/server.ts` ~L328). Auth uses named per-client keys (`npm run key -- mint`) or a bootstrap `MCP_ACCESS_KEY`. Query-string keys are off by default (`.env.example`). Deployment guides cover Docker Desktop, Kubernetes, Azure (Bicep) and cheap hosting (`docs/07`-`12`).

**Can it run locally?** Yes, fully: `docker compose up` starts Postgres and the app, and Ollama runs on the host. The README contrasts it with Nate's version: "Ollama (`nomic-embed-text`, 768-dim) — free, local, private".

**Licence.** MIT (`LICENSE`, © 2026 Scott Nichols).

**Personal-thought assumption.** Weaker than OB1's, but still present. The target is "solo or as a small team (1-3 devs)" capturing *developer* decisions, patterns and postmortems. The added fields (`project`, `created_by`, `supersedes`, `archived`, `source`, provenance hashes) are the most useful in the family for re-scraped sources: they give scoping, supersession and provenance. Ingest is still one thought per call or per batch item, with no splitting of long documents.

## MonkeyRun post

**Stored unit and schema.** A `thoughts` table in Supabase with pgvector. The post names only the embedding and a `weight` column: "We had to add a `weight` column to our schema."

**Capture and tagging.** There are two inputs. "Push (Discord): Short, highly contextual, manually written thoughts" and "Pull (Gmail): Long, noisy, automated chunks of email threads." The post describes no LLM tagging. Weight depends on the source: Discord gets `1.5`, Gmail gets `1.0`. Embeddings use OpenRouter `text-embedding-3-small`. The post does not say how Gmail is chunked.

**Retrieval.** `search_thoughts` embeds the query and calls `match_thoughts`. The team "updated our Supabase `match_thoughts` RPC function to multiply the cosine similarity score by the weight". The reason: "the vector search would often return the Gmail chunks because they were longer".

**MCP tools.** Only `search_thoughts` is named.

**Stack and hosting.** Supabase plus a Deno Edge Function as the MCP server, with Cursor, Claude Code and Claude Desktop as clients. Auth is a static long-lived API key passed in a header or query parameter. This replaced Supabase JWTs, which "broke Claude Desktop connections" when they expired.

**Can it run locally?** No local path is described.

**Licence.** Not stated. The linked repo `github.com/matthallett/monkeyrun-open-brain` returned 404 through the GitHub API on 2026-11-14.

**Lessons relevant to us:**
1. An IVFFlat index built on an empty table and filled incrementally "fails silently". HNSW "handles incremental additions perfectly".
2. Long automated chunks crowd out short manual notes in similarity ranking. Mixing bulk and hand-written content needs explicit source weighting or separation.

**Personal-thought assumption.** This is the only reference whose *core* deals with mixing personal notes and bulk automated content. Its fix is to down-rank the bulk content, not to model it differently.

## Where each assumes personal thoughts rather than bulk scraped digests

- **Unit size:** all three embed the whole `content` as a single vector, with no chunking. srnichols truncates the embedding and warns. OB1's tool description demands "a standalone statement".
- **Tag vocabulary:** the types are personal (`person_note`, `idea`, `task`) or developer-oriented (srnichols). None has team, project-of-org, channel, ticket or status fields. srnichols's `project` is the closest.
- **Write path:** one LLM call per thought on capture. That is cheap at "20 thoughts/day" (`integrations/slack-capture/README.md` cost note), but a bulk scrape multiplies it. OB1's `smart-ingest` caps cost with hard ceilings for this reason.
- **Freshness:** OB1 core has only fingerprint-merge for exact duplicates. Semantic dedup and revisions come through `smart-ingest`, and time-aware ranking through `match_thoughts_recency`. srnichols has explicit `supersedes` and `archived` but no automatic exclusion of superseded rows. MonkeyRun has neither.
- **Provenance:** OB1's ingestion-metadata contract (source ids/urls, `original_created_at`, `input_hash`) and srnichols's provenance hashes are the reusable parts for scraped Slack and Jira output.

## Open questions not answered by these sources

- No reference measures retrieval quality on long digest documents against atomic thoughts.
- MonkeyRun's code is unavailable, so its Gmail chunking strategy is unknown.
