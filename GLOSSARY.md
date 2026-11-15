# Open Brain

A central place for the work knowledge scattered across an organisation's systems and people's heads, so keeping track of it no longer depends on memory, it can be found quickly, and reports can be drawn from it on demand.

## Stored units

**Artefact**:
One natural unit of work, copied from a source system or written by the brain owner: a Jira issue with its comments and history, a whole chat thread, a wiki or decision-record page, a status report, the brain owner's own account of a meeting. The source of truth; never an output.
_Avoid_: Thought, item, source item, record, document, note

**Revision**:
An immutable snapshot of an artefact's content as captured by one push from a collector. A new revision exists only when the content changed.
_Avoid_: Version, update

**Synthesis**:
Anything produced from artefacts rather than captured: a digest, an extracted risk or decision, a topic-page entry, a saved harness answer. It cites the artefacts it came from, and can be regenerated from them.
_Avoid_: Derived item, insight, finding, note

**Digest**:
A synthesis summarising a time window, such as a week of one Slack channel.
_Avoid_: Summary, report

**Entry**:
A synthesis that makes one dated claim on a topic page. Entries are never rewritten; a later entry supersedes an earlier one.
_Avoid_: Item, record

**Topic page**:
A named grouping of entries on one subject, such as a platform team's risk register or decisions log. It holds no content of its own.
_Avoid_: Wiki page, report, document

## Brain

**Brain**:
One running instance of Open Brain, with its own store and config, belonging to exactly one brain owner.
_Avoid_: Instance, database, knowledge base

**Brain owner**:
The one person whose brain it is: their collectors write to it and they set its config. Anyone else who reads its answers is outside the model.
_Avoid_: User, owner (alone), operator, admin

**Collector**:
One of the brain owner's routines that pushes artefacts or syntheses into a brain, such as a scheduled Slack scrape or a manual push of a colleague's digest. Every revision records its collector.
_Avoid_: Workflow, scraper, connector, producer, feed, ingestor

**Harness**:
The AI client the brain owner asks questions through, such as Claude Code or Claude Desktop. A synthesis saved from an answer records the harness as its producer.
_Avoid_: Client, assistant, agent, chatbot

## Organisation

**Platform team**:
An organisational unit whose work a brain follows, such as the Kafka or Flink team. Stakeholders, consumers and other people appear only as people in artefacts, never as platform teams.
_Avoid_: Team, project, squad

**Assigned platform team**:
The platform team a source location is assigned to, if any. A source location has at most one; a cross-functional location has none.
_Avoid_: Owner, owning team

**Source location**:
The container an artefact lives in inside its source system, such as a Jira project, a Slack channel, or a wiki space.
_Avoid_: Project, channel (as a generic term)
