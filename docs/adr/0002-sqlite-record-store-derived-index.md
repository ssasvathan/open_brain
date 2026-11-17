# SQLite record store as the truth, a rebuildable txtai index beside it, no Postgres

Each brain keeps its artefacts, revisions, syntheses, citations, config and the worker's outbox in one SQLite database, the only store worth backing up. Search runs over a separate txtai index (Faiss vectors, BM25 keyword index, txtai's own SQLite for chunk text and filter fields) that is derived from the record store and can always be rebuilt from it. Both live in one container with one data volume. We chose this over Postgres + pgvector, which every reference project uses, because every person in the organisation runs their own single-user brain. A second database container in each install would make the package harder to hand over and give a single-user pilot nothing in return. Keeping the index derived means swapping the search layer (another embedded vector store, pgvector, OpenSearch) costs a rebuild, not a data migration.

## Consequences

- A capture writes the revision and its outbox row in one SQLite transaction (transactional outbox). The in-process worker indexes at least once, upserting into txtai by revision ID.
- Moving to Postgres later, for a shared or central brain, migrates only the record store's tables. The index is rebuilt rather than moved.
