# Open Brain

A central place for the work knowledge scattered across an organisation's systems and people's heads, so keeping track of it no longer depends on memory, it can be found quickly, and reports can be drawn from it on demand.

## Stored units

**Artefact**:
One natural unit of work, copied faithfully (never summarised) from a source system, or written by the brain owner: a Jira issue with its comments and history, a whole chat thread, a wiki or decision-record page, a status report, the brain owner's own account of a meeting. The source of truth; never an output. Anything that holds knowledge no other artefact contains, such as the brain owner's own judgement, is an artefact even when it links to the artefacts that prompted it.
_Avoid_: Thought, item, source item, record, document, note

**Revision**:
One recorded state of an artefact. An artefact's revisions form its history, ranked by source time. Immutable; a new one exists only when the content or source fields changed.
_Avoid_: Version, update

**Current revision**:
The top-ranked revision: the latest state of the artefact the brain knows of.
_Avoid_: Latest version, head

**Source time**:
When, by the source system's account, an artefact was in the state a revision records. What "as of" means in an answer.
_Avoid_: Updated-at, event time

**Capture time**:
When the brain received a capture: when the brain learned something, never when it happened.
_Avoid_: Captured-at, ingest time

**Synthesis**:
Anything produced from artefacts rather than captured: a digest, an extracted risk or decision, a topic-page entry, a saved harness answer. It cites the artefacts it came from, and can be regenerated from them. A synthesis re-produced for the same purpose, such as a re-run of last week's digest, supersedes the earlier one rather than replacing it. The test is regeneration: what could be re-derived from the cited artefacts is a synthesis; what could not is an artefact.
_Avoid_: Derived item, insight, finding, note

**Synthesis key**:
The named job a synthesis was made to do, chosen by the collector or harness that captures it, such as one channel's digest for one week or one risk on a risk register. A later synthesis with the same synthesis key supersedes the earlier one.
_Avoid_: Key (alone), slot, purpose, supersedes link

**Citation**:
A synthesis's reference to one artefact, recording which revision was current when the synthesis was stored. A citation is *unresolved* while the artefact is not yet in the brain, and *outdated* once the artefact has a revision with a later source time than the one cited.
_Avoid_: Reference, source, link, stale citation

**Superseded**:
Said of a synthesis or entry that a later one, made for the same purpose, has replaced. It is kept but no longer current. Artefacts are never superseded; they gain revisions instead, and a synthesis never supersedes an artefact.
_Avoid_: Replaced, deleted, obsolete, stale

**Digest**:
A synthesis summarising a time window, such as a week of one Slack channel.
_Avoid_: Summary, report

**Entry**:
A synthesis that makes one dated claim on a topic page, dated by the latest source time among the artefacts it cites. Entries are never rewritten; a later entry supersedes an earlier one.
_Avoid_: Item, record

**Capture**:
One hand-off of artefacts or syntheses to the brain, by a single collector or the harness. Each item in it is accepted or rejected on its own, and each artefact in it becomes at most one new revision.
_Avoid_: Push, upload, ingest, sync, batch

**Topic page**:
A running list on one subject that the brain owner wants kept, such as outstanding risks or decisions made, built from the entries saved to it. Reading a topic page means reading its current entries; it has no text of its own. A topic page is about a subject, not a platform team, so one entry on it can concern several teams.
_Avoid_: Wiki page, report, document

**Source field**:
A named value the source system shows as a labelled field beside an artefact's content, such as a Jira issue's status, priority or resolution, or a decision page's status. Never read out of the content itself. Which source fields exist varies by source system and by what the collector sends. Captured alongside the artefact's content, and part of its revision.
_Avoid_: Metadata, property, attribute

**Tag**:
A label saying what an artefact or synthesis is about or what it contains, such as a topic, a risk or decision flag, or a person mentioned. The brain tags everything the same way; a collector may add its own tags, which are kept and marked as the collector's.
_Avoid_: Label, category, classification

## Brain

**Brain**:
One running instance of Open Brain, with its own store and config, belonging to exactly one brain owner. It keeps the ledger: it records, ranks and retrieves by the same rules for every source system.
_Avoid_: Instance, database, knowledge base

**Brain owner**:
The one person whose brain it is: their collectors write to it and they set its config. Anyone else who reads its answers is outside the model.
_Avoid_: User, owner (alone), operator, admin

**Collector**:
One of the brain owner's routines that captures artefacts or syntheses into a brain, such as a scheduled Slack scrape or a manual capture of a colleague's digest. Every revision records its collector.
_Avoid_: Workflow, scraper, connector, producer, feed, ingestor

**Harness**:
The AI client the brain owner asks questions through, such as Claude Code or Claude Desktop. A synthesis saved from an answer records the harness where a capture would otherwise record its collector. It does the reasoning: interpreting, comparing and summarising what the brain returns.
_Avoid_: Client, assistant, agent, chatbot

## Organisation

**Platform team**:
An organisational unit whose work a brain follows, such as the Kafka or Flink team. A platform team exists in a brain once a collector credits an artefact to it; the brain keeps no list of its own. Stakeholders, consumers and other people appear only as people in artefacts, never as platform teams.
_Avoid_: Team, project, squad

**Credited platform team**:
The platform team a collector credits an artefact to, if any. An artefact has at most one; a cross-functional artefact, such as a thread in a shared leads channel, has none. The latest capture's credit stands, and changing it never creates a revision. A synthesis has no credited platform team of its own: its teams are those of the artefacts it cites.
_Avoid_: Owner, owning team, assigned platform team

**Source location**:
The container an artefact lives in inside its source system, such as a Jira project, a Slack channel, or a wiki space.
_Avoid_: Project, channel (as a generic term)
