# Which public corpus can stand in for scraped org artifacts, and what do leaders ask of it?

Research for [#3](https://github.com/ssasvathan/open_brain/issues/3) (map: [#2](https://github.com/ssasvathan/open_brain/issues/2)). Sources checked 2026-11-14. The volume figures below come from live, anonymous API calls made that day (queries shown), not from secondary write-ups.

## Answer

Use **Apache Kafka and Apache Flink, January–June 2025**, pulled from the Apache Software Foundation's own public systems:

- **Jira** at `issues.apache.org/jira` stands in for Jira tickets.
- **Confluence** at `cwiki.apache.org` stands in for Confluence pages. Kafka's KIPs and Flink's FLIPs are design proposals that carry ADR-style status, and each project also has release-plan pages.
- **The `dev@` mailing lists** stand in for team chat. Their `[DISCUSS]` and `[VOTE]` threads are where architecture gets argued and decided.
- **ASF board reports** are quarterly project-status write-ups aimed at the foundation's leadership.

Both projects are the real products (Jira and Confluence) behind the artifact kinds named in #3. Everything is readable anonymously over REST. The artifacts are timestamped and heavily cross-linked: a KIP page links its discussion thread, its vote thread and its Jira ticket. In that window, Kafka 4.0 shipped on 2025-03-18 with ZooKeeper removed, Flink 2.0 shipped the same day, and the Kafka "Diskless Topics" proposal was debated and accepted. That gives realistic "what changed" and "what was decided" questions.

The weak spot is chat. Mailing-list threads are longer and more formal than Slack. The ASF has no public Slack archive for Kafka. Flink's Slack is archived publicly, but its archive could not be fetched (see below).

## Candidates compared

| Candidate | Artifact kinds | How to fetch | Size | Licence / terms | Resemblance to enterprise artifacts |
|---|---|---|---|---|---|
| **ASF projects (Kafka, Flink; Spark/Cassandra as extras)** | Jira, Confluence (KIPs/FLIPs, release plans, meeting notes), dev@ mail, board reports | REST APIs, no auth (below) | Kafka+Flink H1-2025: 2,120 issues, ~6,900 mails, ~136 new wiki pages | Mail archives are "deemed to be published by the sender … made public without conditions" ([ASF public archives](https://www.apache.org/foundation/public-archives.html)). Under Apache License 2.0, "Contribution" covers communication on "mailing lists, source code control systems, and issue tracking systems" for discussing the work ([LICENSE-2.0](https://www.apache.org/licenses/LICENSE-2.0.txt)). The cwiki pages show no licence footer (checked) | **Highest.** Same Jira and Confluence products and fields. Decision threads read like design-review discussion. Board reports read like status reports to leadership |
| [The Public Jira Dataset](https://doi.org/10.5281/zenodo.5882881) (Montgomery et al., MSR 2022, [arXiv:2201.08368](https://arxiv.org/abs/2201.08368)) | Jira only: 16 public Jiras, 1,822 projects, ~2.7M issues, ~9M comments, ~1M links, ~32M changes | Zenodo download, `ThePublicJiraDataset.zip`, 5.81 GB (v7, 2025-06-23) | Large | CC-BY-4.0, open access (Zenodo API). Older versions are restricted because they held personal data | Good for Jira. No chat, pages or ADRs, and too big for a pilot. Live ASF Jira gives the same shape on demand |
| [Software-related Slack Chats with Disentangled Conversations](https://doi.org/10.5281/zenodo.3627124) (Chatterjee et al., MSR 2020) | Slack only: python, clojure and elm communities, Jul 2017–Jun 2019, 38,955 conversations / 437,893 utterances, anonymised, XML | Zenodo zip, ~30 MB | Small | Zenodo licence field: `other-open` | Real Slack shape (short, threaded). But these are Q&A help channels, not team decision-making, and unrelated to the other artifacts |
| GitHub repos with ADRs, e.g. [backstage/backstage `docs/architecture-decisions`](https://github.com/backstage/backstage/tree/master/docs/architecture-decisions) (17 files, Apache-2.0), [alphagov/govuk-aws `docs/architecture/decisions`](https://github.com/alphagov/govuk-aws/tree/master/docs/architecture/decisions) (42, MIT), [alphagov/govuk-infrastructure](https://github.com/alphagov/govuk-infrastructure/tree/main/docs/architecture/decisions) (26, MIT) | Classic Nygard-style ADRs | `gh api repos/<o>/<r>/contents/<dir>` or `git clone` | Tens of files | Repo licence (checked via `gh api`) | Exact ADR format. But each set stands alone, with no Jira or chat for the same team. [Buchgeher et al. 2023](https://research.jku.at/en/publications/using-architecture-decision-records-in-open-source-projects-an-ms/) found 921 GitHub repos with ADRs, about half holding only 1–5 |
| Flink Slack public archive | Chat | linen.dev | ? | Flink says "All messages from public channels in our Slack are permanently stored and published" ([Flink community page](https://flink.apache.org/what-is-flink/community/)) | Closest to Slack. **Unverified:** `linen.dev/s/apache-flink` returned HTTP 429 on 2026-11-14. Flink also says important decisions "must be reflected back to the mailing lists … If it didn't happen on a mailing list, it didn't happen" (same page), so the mailing list is where decisions are recorded |

Projects were also checked for being on ASF Jira (live JQL). Kafka, Flink, Spark and Cassandra are active, each with issues created on 2026-11-14. Airflow and Iceberg are not on ASF Jira; they use GitHub Issues. Hudi's newest issue is from 2025-09 and Pulsar's from 2023.

## How to fetch (verified endpoints)

- **Jira.** `GET https://issues.apache.org/jira/rest/api/2/search?jql=project=KAFKA AND created>=2025-01-01 AND created<2025-07-01&fields=...&startAt=N&maxResults=100` works anonymously. `GET .../rest/api/2/issue/KAFKA-18669?expand=changelog` returns the full status and field history with timestamps, which supports over-time questions. `issuelinks` carries the cross-links (e.g. KAFKA-17057 → KAFKA-17441).
- **Confluence (cwiki).** `GET https://cwiki.apache.org/confluence/rest/api/content/search?cql=space=KAFKA and type=page and created>="2025-01-01"&expand=version,history,body.storage`. Earlier page versions come from `GET .../rest/api/content/{id}?status=historical&version=N&expand=body.storage`. Checked on KIP-1150: version 2 is dated 2025-04-16, the current version is 20, and the page reads "Current state: Accepted", followed by links to its discussion thread, vote thread and `KAFKA-19161`.
- **Mailing lists (Pony Mail).** `GET https://lists.apache.org/api/stats.lua?list=dev&domain=kafka.apache.org&d=2025-03` returns that month's messages as JSON: subject, from, epoch, message-id, in-reply-to and thread structure. Full mbox comes from `GET https://lists.apache.org/api/mbox.lua?list=dev@flink.apache.org&date=2025-03` (6.4 MB for that month). Drop messages whose `from` contains "(Jira)". They are notification copies of Jira events, about half of Kafka dev@.
- **Board reports.** Plain text at `https://www.apache.org/foundation/records/minutes/2025/board_minutes_2025_MM_DD.txt`. Kafka and Spark reports are in the 2025-02-19 and 2025-05-21 minutes. Flink's are in 2025-03-19 and 2025-06-18.
- **Releases** (for timeline anchors). `archive.apache.org/dist/kafka/` lists 4.0.0 on 2025-03-18 and 3.9.1 on 2025-05-19. `archive.apache.org/dist/flink/` lists 2.0.0 on 2025-03-18 and 1.20.1 on 2025-02-11.

Fetch into a local, git-ignored data directory, never into this repo. Throttle requests: ASF systems are shared infrastructure, and Jira's rate limits were not looked up.

## Recommended corpus

| | Apache Kafka | Apache Flink |
|---|---|---|
| Jira issues created 2025-01-01 to 2025-06-30 | 1,073 (969 resolved in the window) | 1,047 |
| Busiest components ("sub-teams") | streams 91, clients 89, consumer 56, kraft 36 (717 have no component) | Flink CDC 176, Table SQL/Planner 116, Table SQL/API 54, Kubernetes Operator 41, **Connectors/Kafka 29** |
| dev@ messages Jan–Jun 2025 | 4,060 (1,972 Jira copies; 868 with `[DISCUSS]` in the subject) | 2,817 (1,049 Jira copies; 605 `[DISCUSS]`) |
| cwiki pages created in window | 72 (e.g. KIP-1150 Diskless Topics, KIP-1134 Virtual Clusters, Release Plan 4.0.1) | 64 (e.g. FLIP-529 Connections in Flink SQL, FLIP-504 Blue/Green Deployments, "2.1 Release") |
| Board reports in window | 2025-02, 2025-05 | 2025-03, 2025-06 |
| Milestone in window | 4.0.0 (ZooKeeper removed) on 2025-03-18 | 2.0.0 on 2025-03-18 |

After dropping Jira copies, this comes to about 2,100 tickets, 3,800 human-written emails, 136 or more wiki pages and 4 board reports. That is a few thousand documents, which fits a laptop and the embedder chosen in #7. Treat the two projects as "teams" and their Jira components as sub-teams. The cross-team signal comes from Flink's Kafka connector work and Flink CDC. It is real but thin: 44 FLINK issues in the window mention "kafka". No KAFKA issue mentions "flink", and no FLINK issue mentions "KIP".

Optional extras:

- **Spark** as a third team, for more cross-project overlap. It is much larger: 1,910 issues in the window, 11 of which mention Kafka.
- **One GitHub ADR set** (Backstage, Apache-2.0) to exercise the strict ADR format, since KIPs and FLIPs are proposal-style rather than Nygard ADRs.
- **A small slice of the Chatterjee Slack set**, to test short chat-shaped messages, if mailing-list prose proves too unlike Slack.

## Draft leadership questions (for the human to edit)

> DRAFT. These are proposed acceptance cases written against the corpus above. Edit, replace or drop them freely.

1. *What architectural decisions did Kafka and Flink accept between January and June 2025, and who drove each one?* (KIP/FLIP status + vote threads)
2. *Where does the Diskless Topics proposal (KIP-1150) stand, how did it change between its first draft and acceptance, and what were the main objections?* (page versions + DISCUSS thread + KAFKA-19161)
3. *What is Flink doing that depends on, or is affected by, changes in Kafka?* (Connectors/Kafka and CDC tickets; cross-team dependency)
4. *What risks or blockers threatened the Kafka 4.0 and Flink 2.0 releases, and were they resolved before 18 March 2025?* (release-plan pages, blocker tickets, changelog)
5. *What changed in Kafka between the 4.0 release and the end of April 2025?* (status changes over time)
6. *Which design proposals have been under discussion for more than six weeks without reaching a vote?* (stalled decisions)
7. *Which areas have the most open critical or blocker bugs, and is that rising or falling over the half-year?* (trend by component)
8. *What did each project tell the ASF board about its health and concerns this half, and does that match what the tickets and threads show?* (board report vs ground truth)
9. *Are the two teams solving the same problem in different ways?* (e.g. Kubernetes deployment, state/storage, Java version upgrades)

## Caveats

- **Personal data.** Mail and Jira carry real names and addresses. Using them locally fits the ASF's "public without conditions" stance, but per #3 nothing confidential, and no re-published bulk content, should be committed here. Commit only fetch scripts and question lists.
- **Chat realism.** `dev@` threads are asynchronous and essay-like. If the pilot's real input is mostly terse Slack, results on this corpus may flatter long-text retrieval. That ties into the embedder bake-off noted under "Not yet specified" on #2.
- **What was not verified.** The list of the 16 Jiras in the Public Jira Dataset (the arXiv abstract does not name them). The terms of the Chatterjee `other-open` licence beyond the Zenodo field. Flink's linen.dev archive (HTTP 429). ASF Jira rate limits.
