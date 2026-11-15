# How harnesses connect to a local MCP server

Research for [#8](https://github.com/ssasvathan/open_brain/issues/8) (map: [#2](https://github.com/ssasvathan/open_brain/issues/2)). Sources read 2026-11-14; primary sources only (MCP specification, Claude Code docs, MCP-hosted Claude Desktop guide, Claude Help Center).

## Short answer

- **Use stdio as the default transport.** Claude Code and Claude Desktop both launch a local stdio server as a subprocess, using a config entry made of a command, args and env. The MCP spec says clients SHOULD support stdio, and it says stdio servers get credentials from the environment rather than from OAuth. On a single-user laptop this means **no auth layer** is needed.
- **Streamable HTTP on localhost is an option for Claude Code** (`--transport http`). It is **not reachable from Claude Desktop's or claude.ai's "custom connectors"**, because Anthropic's cloud makes those connections and they need a public URL on port 443. If HTTP is used, the server must validate `Origin`, bind to 127.0.0.1, and should authenticate (a static bearer header is enough for Claude Code).
- **Size limits are set by the harness.** Claude Code warns above 10k tokens per tool result and caps results at 25k tokens by default. Text results over 50,000 characters go to a file. Tool descriptions are cut at 4,096 chars (16,384 if loaded via tool search). HTTP responses over 16 MiB fail.
- **A scraping workflow calls a write tool like any other tool**: the model in the harness calls `mcp__<server>__<tool>` with JSON arguments. In unattended `claude -p` runs that tool must be pre-allowed (`--allowedTools` or `permissions.allow`), otherwise the call is denied or blocked.
- **Design constraints for Open Brain:** keep write tools coarse-grained and idempotent, with small, flat, schema-valid JSON inputs. Return short acknowledgements, not echoes. Paginate and summarise reads to fit well under 25k tokens. Return validation failures as `isError` results. Annotate tools (`readOnlyHint`, `idempotentHint`). Keep the tool count and descriptions small.

## 1. Transports

### What the spec defines

The current MCP revision is **2026-07-28**. It defines two standard transports, stdio and Streamable HTTP ([spec transports overview](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports)). In the 2025-06-18 revision the spec said "Clients **SHOULD** support stdio whenever possible" ([2025-06-18 transports](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports)).

**stdio** ([2025-06-18 transports](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports)):
- "The client launches the MCP server as a subprocess." The server's lifetime is tied to the harness session.
- Messages are newline-delimited JSON-RPC and "**MUST NOT** contain embedded newlines".
- "The server **MUST NOT** write anything to its `stdout` that is not a valid MCP message". Logging goes to `stderr`.
- The 2026-07-28 revision says any reliable byte-stream custom transport (Unix socket, TCP) SHOULD reuse stdio framing ([transports overview](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports)).

**Streamable HTTP** ([2026-07-28 Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)):
- The server is an independent process with a single POST endpoint. Each request is its own POST, and the server answers with JSON or a request-scoped SSE stream.
- 2026-07-28 **removed protocol-level sessions (`Mcp-Session-Id`) and the GET stream**. Earlier revisions (2025-03-26 through 2025-11-25) had both. Servers that want to support older clients have to handle both eras.
- Security (verbatim): "Servers **MUST** validate the `Origin` header on all incoming connections to prevent DNS rebinding attacks … When running locally, servers **SHOULD** bind only to localhost (127.0.0.1) rather than all network interfaces (0.0.0.0) … Servers **SHOULD** implement proper authentication for all connections. Without these protections, attackers could use DNS rebinding to interact with local MCP servers from remote websites."
- The old HTTP+SSE transport (2024-11-05) is deprecated: "New implementations **SHOULD NOT** adopt it."

### Claude Code

From [Claude Code MCP docs](https://code.claude.com/docs/en/mcp):
- Transports: `http` (recommended for remote), `sse` (deprecated), `stdio` ("Local process"), and WebSocket (config only).
- stdio: `claude mcp add [--env KEY=VAL] <name> -- <command> [args...]`. HTTP: `claude mcp add --transport http <name> <url> [--header "Authorization: Bearer …"]`. In JSON, `type: "streamable-http"` is an alias for `http`.
- Claude Code sets `CLAUDE_PROJECT_DIR` in a stdio server's environment.
- **Scopes:** `local` (default, `~/.claude.json`, this project only), `project` (`.mcp.json` in the repo root, shared through git), and `user` (all projects). Precedence: local > project > user > plugin > claude.ai connectors.
- Interactive sessions ask for approval of project `.mcp.json` servers. `claude -p` and SDK sessions load them without asking.
- `${VAR}` and `${VAR:-default}` expand in `command`, `args`, `env`, `url` and `headers`.
- `claude mcp add-from-claude-desktop` imports Desktop's config.
- Timeouts: `MCP_TIMEOUT` for startup. `MCP_TOOL_TIMEOUT` or a per-server `timeout` for tool calls (the default is about 28 h). The idle timeout is 30 min for stdio and 5 min for HTTP. Calls still running after 2 min are moved to a background task in the main conversation.

### Claude Desktop

From the [MCP "Connect to local MCP servers" guide](https://modelcontextprotocol.io/docs/develop/connect-local-servers), which uses Claude Desktop as its example:
- The config file is `claude_desktop_config.json`, opened from Settings → Developer → Edit Config. Its location is macOS `~/Library/Application Support/Claude/`, Windows `%APPDATA%\Claude\`, or Linux `~/.config/Claude/`.
- Entries are `mcpServers.<name>` with `command`, `args` and `env`, i.e. stdio launch. Paths must be absolute. Desktop has to be fully restarted to pick up changes.
- Logs: `mcp.log`, plus `mcp-server-<NAME>.log` holding the server's stderr.
- "All actions require your explicit approval before execution."

From [Claude Help Center: custom connectors via remote MCP](https://support.claude.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp):
- "Claude connects to your remote MCP server from Anthropic's cloud infrastructure, rather than from your local device". The server "must be reachable over the public internet from Anthropic's IP ranges", on port 443.
- Locally configured servers "are not available in claude.ai or in Cowork sessions that run in the cloud."

**Implication:** a `localhost` HTTP URL cannot be added as a Desktop or claude.ai custom connector. A local Open Brain reaches Desktop only through the stdio entry in `claude_desktop_config.json`. A stdio server reaches both Claude Code and Desktop with the same command line.

## 2. Auth expectations

- MCP spec: "Authorization is **OPTIONAL** for MCP implementations." Implementations "using an STDIO transport **SHOULD NOT** follow this specification, and instead retrieve credentials from the environment." The OAuth 2.1 flow (PRM / RFC 9728, PKCE, audience-bound tokens) applies only to HTTP transports ([2025-06-18 authorization](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)).
- Claude Code supports static headers (`--header` / `headers`), a `headersHelper` command (10 s limit, re-run after 401/403) and OAuth (`/mcp` or `claude mcp login`). OAuth cannot run inside `claude -p`, so the user has to sign in interactively first ([Claude Code MCP docs](https://code.claude.com/docs/en/mcp)).
- **For the pilot:** stdio needs no auth, because the OS process boundary and the user's account are the trust boundary. With localhost HTTP, a static bearer token from an env var plus `Origin` checking is enough. Full OAuth is unnecessary for a single local user.

## 3. Tool and result size limits (Claude Code)

All from [Claude Code MCP docs](https://code.claude.com/docs/en/mcp):

| Limit | Value |
|---|---|
| Warning on tool output | > 10,000 tokens |
| Default max tool output | 25,000 tokens (`MAX_MCP_OUTPUT_TOKENS` overrides) |
| Text result persisted to file instead of shown inline | > 50,000 chars (a per-tool `_meta["anthropic/maxResultSizeChars"]` raises it, ceiling 500,000) |
| Over the token limit | result is saved to a file under `~/.claude/projects/.../tool-results`; Claude gets the path |
| `isError` text | over ~11,000 chars, only the first 5,000 and last 5,000 chars are kept |
| HTTP/SSE response body | > 16 MiB fails the request ("return less data per response … paginating") |
| Server instructions / upfront tool description | cut at 4,096 chars |
| Tool description loaded via tool search | cut at 16,384 chars |
| Full tool name `mcp__<server>__<tool>` | > 128 chars means the tool is dropped |
| Input schema | top-level property names must be 1–64 chars, `[A-Za-z0-9_.-]`, and the schema must be valid JSON Schema 2020-12, otherwise the tool is excluded; root-level `anyOf/oneOf/allOf` gets flattened |

- Tool search is on by default. Only tool names and server instructions load at start, so the **server `instructions` text is what tells Claude when to look for the brain's tools**.
- MCP spec tool names SHOULD be 1–128 chars from `[A-Za-z0-9_.-]` ([2025-11-25 tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)).
- These docs give no comparable published numbers for Claude Desktop. Design for the Claude Code numbers.

## 4. How a scraping workflow calls a write tool

- MCP tools are "model-controlled". The harness's model chooses to call `tools/call` with `name` and `arguments` that match the tool's `inputSchema` ([2025-11-25 tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)). In Claude Code the tool appears as `mcp__<server>__<tool>`, and that full name is used in permission rules, skill `allowed-tools`, subagent `tools` and hooks ([Claude Code MCP docs](https://code.claude.com/docs/en/mcp)).
- So an existing scraper skill or prompt keeps scraping, and at the point where it now writes Markdown it calls e.g. `mcp__openbrain__<write_tool>` with the content. Nothing else in the scraper changes. The scraper must be running inside a harness session where the server is configured.
- **Permissions** ([Claude Code permissions](https://code.claude.com/docs/en/permissions)):
  - `mcp__openbrain` or `mcp__openbrain__*` allows every tool on the server, and `mcp__openbrain__<tool>` allows one tool.
  - Allow globs need a literal `mcp__<server>__` prefix. A bare `mcp__*` allow is ignored.
  - In `dontAsk` mode, or with `permissions.denyInsteadOfPrompting`, calls that would prompt are denied. Unattended `claude -p` scrapes therefore need the write tool in `--allowedTools` or `permissions.allow`.
  - Never set `_meta["anthropic/requiresUserInteraction"]` on the write tool. With it set, the tool prompts every time and is denied in non-interactive mode.
- **Retries:** if a tool is annotated `idempotentHint: true` (or `readOnlyHint: true`), Claude Code may repeat a call that was pending when an HTTP connection dropped. `readOnlyHint` tools can also run in parallel ([Claude Code MCP docs](https://code.claude.com/docs/en/mcp)).
- **Errors:** tool execution errors go in the result with `isError: true` and "contain actionable feedback that language models can use to self-correct and retry". Protocol errors are for malformed requests ([2025-11-25 tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)).
- **Human-in-the-loop:** the spec says hosts SHOULD let users deny tool calls ([2025-11-25 tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)). Claude Desktop asks for approval per action. Claude Code follows its permission rules.

## 5. Constraints this puts on Open Brain's tool design

These are inferences from the facts above, not quoted requirements.

1. **Ship as a stdio server first.** A single command (`openbrain serve` or similar) launched by the harness serves both Claude Code (`claude mcp add openbrain -- …` or a checked-in `.mcp.json`) and Claude Desktop (`claude_desktop_config.json`). Nothing goes to stdout except protocol messages, and logs go to stderr. "Clone and run your own" then means one config entry. Localhost HTTP is an optional extra for Claude Code only.
2. **Multiple processes, one store.** With stdio, each harness session spawns its own server process, so Desktop plus two Claude Code sessions means three processes. The storage layer must tolerate concurrent writers (e.g. SQLite WAL / file locking), or the design must choose a single long-running HTTP daemon instead. **This is the main architectural fork.**
3. **No auth in the pilot.** Config (data dir, team list) comes from env/args, matching the spec's "credentials from the environment" guidance for stdio.
4. **Write tools take content, not files.** Arguments are JSON produced by the model, so one call should carry one scrape unit (one channel/board digest), well under the 16 MiB transport cap and small enough for the model to emit reliably. Use flat schemas, top-level property names `[A-Za-z0-9_.-]` up to 64 chars, and no root `anyOf`.
5. **Make writes idempotent and annotate them.** Accept a caller-supplied source key (e.g. source + channel + period) so that re-scrapes and harness retries upsert instead of duplicating. Set `idempotentHint: true`. This also helps the "freshness and supersession" open question on the map.
6. **Return small acknowledgements** (id, created/updated, short summary). Do not echo stored content.
7. **Bound read results.** Default to compact, paginated or summarised results that fit far below 25k tokens or 50k chars. Otherwise Claude Code moves the result to a file and the answer quality on "ask the harness" questions drops. Use `_meta["anthropic/maxResultSizeChars"]` only for deliberate bulk reads.
8. **Validation errors as `isError` results** with short, actionable text (under ~10k chars) so the scraping model can fix and retry.
9. **Few tools, short names, strong server `instructions`.** With tool search on, the server `instructions` (≤4,096 chars) decide whether Claude finds the brain at all. Keep `mcp__openbrain__<tool>` names short and put the key words at the start of descriptions.
10. **Unattended scrapes need a pre-allow rule.** Document the `permissions.allow: ["mcp__openbrain__*"]` (or per-tool) entry as part of setup.

## Sources

- MCP spec 2026-07-28: [overview](https://modelcontextprotocol.io/specification/latest), [transports](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports), [Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- MCP spec 2025-06-18: [transports](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports), [authorization](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)
- MCP spec 2025-11-25: [tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
- [MCP docs: Connect to local MCP servers (Claude Desktop)](https://modelcontextprotocol.io/docs/develop/connect-local-servers)
- [Claude Code: MCP](https://code.claude.com/docs/en/mcp)
- [Claude Code: Permissions](https://code.claude.com/docs/en/permissions)
- [Claude Help Center: custom connectors using remote MCP](https://support.claude.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp)

## Gaps

- The 2026-07-28 tools page was not re-read. Tool-level facts come from 2025-11-25 and may have small revisions.
- These sources do not document whether Claude Desktop's local config accepts `url`/HTTP entries, nor Desktop's result-size limits. Desktop Extensions (`.mcpb`) packaging was not investigated.
- Which MCP protocol revision Claude Code and Desktop negotiate today was not verified. It matters only if Open Brain uses HTTP, because sessions were removed in 2026-07-28.
