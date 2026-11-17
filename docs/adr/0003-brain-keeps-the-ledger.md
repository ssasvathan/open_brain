# The brain keeps the ledger; the harness reasons

A brain records, ranks and retrieves by rules that are the same for every source system. Interpreting what a source field means, deciding that one item is outdated or replaced by another, and summarising or comparing content all happen in the harness at read time, or in a collector at capture time. We chose this because:

- every source-specific rule in the brain makes it less general and has to be maintained for each new source system;
- the brain's local model is too small to be trusted with judgement;
- every brain owner installs the same package, so judgement living in the harness can be tuned per person without changing the brain.

## Considered options

- **Status lifecycle mapping** (e.g. KIP "Accepted" = final): rejected. The harness reads raw values through `filter_options` and filters on them.
- **A lint pass that proposes supersession:** rejected for the brain. If it is ever built, it is a collector that captures syntheses.
- **Read-time summaries or a diff option on `fetch`:** rejected. The harness fetches the revisions and compares them itself.
- **Rejecting captures older than the current revision:** rejected. They take their place in the artefact's history by source time.
- **Recency boost in search ranking:** rejected. Newer is not more current; date filters keep time explicit.
- **The brain's worker updating topic pages at capture time** (gemma4:e4b judging whether a new artefact changes an entry): rejected. Entries come from the harness, with the brain owner's consent, or from a collector that captures entries.

## Consequences

- The brain's tools offer no operation that interprets, judges or summarises.
- The server `instructions` tell the harness that this reasoning is its job.
- The acceptance cases must test whether the harness can chain `list` and `fetch` to reach such conclusions on its own.
