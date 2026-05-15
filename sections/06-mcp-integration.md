---
title: "Section 6 — MCP Integration: Errors, Servers & Resources"
linkTitle: "6. MCP Integration"
weight: 6
description: "Domains 2.2, 2.4 — structured error envelopes (isError, isRetryable), .mcp.json scopes and env expansion, resources vs tools."
---

## What this section covers

Two operationally critical MCP domains: how tools should *report failure* so the agent can recover intelligently (2.2), and how servers and resources should be wired into Claude Code so a team gets the right capabilities at the right scope (2.4). MCP is a contract between a server and an agent — the quality of that contract (errors, descriptions, resources) determines whether the agent makes smart decisions or thrashes.

## Source material (from official guide)

### 2.2 Structured error responses

Knowledge: the four error classes (transient, validation, business, permission); why uniform `"Operation failed"` responses break recovery; retryable vs non-retryable.

Skills: return `errorCategory`, `isRetryable`, and a human-readable description; mark business-rule violations with `retriable: false` plus a customer-friendly explanation; have subagents do local recovery for transient errors and escalate only what can't be locally resolved (with partial results and what was attempted); distinguish *access failures* from *valid empty results*.

### 2.4 Server integration

Knowledge: project (`.mcp.json`) vs user (`~/.claude.json`) scope; `${VAR}` expansion for secrets; tools from all servers are discovered at connection time and available simultaneously; resources expose content catalogs to reduce exploratory tool calls.

Skills: configure shared servers in `.mcp.json` with env-var expansion; configure personal servers in user scope; write rich tool descriptions so the agent doesn't fall back to built-ins like `Grep`; pick community servers (Jira, GitHub, Postgres) over custom; expose content catalogs as resources.

## MCP at 30,000 feet

The [Model Context Protocol](https://modelcontextprotocol.io/specification/latest) is an open, JSON-RPC 2.0 protocol that connects LLM hosts to capability providers (servers) over a stateful, capability-negotiated session. Servers expose three primitives:

| Primitive | Controlled by | Purpose | Example |
| --- | --- | --- | --- |
| **Tool** | Model | Executable action with side effects | `create_issue`, `run_query`, `send_email` |
| **Resource** | Application / user | Read-only content identified by URI | `jira://issues/PROJ-123`, `postgres://schema/orders` |
| **Prompt** | User | Pre-built templates invoked by user choice (slash commands, menu picks) | `/code-review`, `/summarize-incident` |

Transports defined in the spec: **stdio** (local subprocess), **Streamable HTTP** (called `streamable-http` in the spec, aliased to `http` in Claude Code config), and the legacy **SSE** transport. See the [schema reference](https://modelcontextprotocol.io/specification/2025-11-25/schema) for the wire format.

## Structured error responses

### Error category taxonomy

| Category | What it means | Typical `isRetryable` | What the agent should do |
| --- | --- | --- | --- |
| `transient` | Timeout, 5xx, rate limit, connection reset | `true` | Back off and retry locally inside the subagent |
| `validation` | Schema mismatch, bad enum, missing field | `false` | Reformulate input; do not retry the same call |
| `business` | Policy/limit/state violation ("refund > $500", "ticket already closed") | `false` | Surface the customer-facing explanation; stop |
| `permission` | 401/403, scope missing, RBAC denied | `false` | Escalate to coordinator or human |
| `not_found` | Valid query, zero matches | `false` (and `isError` is debatable — see below) | Treat as a legitimate empty result, not a failure |

### `isError` flag — schema and example

The official [`CallToolResult`](https://modelcontextprotocol.io/specification/2025-11-25/schema) shape:

```ts
interface CallToolResult {
  content: ContentBlock[];
  structuredContent?: { [key: string]: unknown };
  isError?: boolean;  // default false
  _meta?: { [key: string]: unknown };
}
```

The spec is explicit: *errors originating from the tool itself* must be reported inside the result with `isError: true`, **not** as a JSON-RPC protocol error — protocol errors are invisible to the model, and the LLM needs to see the error to self-correct. Protocol errors are reserved for "tool not found" or "server doesn't support tool calls."

A well-structured tool failure looks like this:

```json
{
  "isError": true,
  "content": [
    { "type": "text", "text": "Refund of $750 exceeds the $500 single-transaction policy. Ask the customer to split the refund or open a manager-approval ticket." }
  ],
  "structuredContent": {
    "errorCategory": "business",
    "isRetryable": false,
    "code": "REFUND_LIMIT_EXCEEDED",
    "limit": 500,
    "requested": 750,
    "customerMessage": "We can only process refunds up to $500 in one transaction."
  }
}
```

Compare that to the anti-pattern — `{ "isError": true, "content": [{ "type": "text", "text": "Operation failed" }] }` — which forces the agent to either retry blindly or give up.

### Retryable vs non-retryable decision tree

```
isError === true ?
├── No  → success path (or empty-result path if content is intentionally empty)
└── Yes
    ├── errorCategory === "transient" && isRetryable === true
    │     → backoff + retry locally (subagent), cap attempts (e.g. 3)
    ├── errorCategory === "validation"
    │     → re-plan with corrected input; do NOT replay the same call
    ├── errorCategory === "permission"
    │     → escalate to coordinator with the missing scope/role
    └── errorCategory === "business"
          → surface customerMessage; stop; do NOT retry
```

The structured metadata is what makes this tree executable. Without `errorCategory` and `isRetryable`, the agent has to *guess* whether a failure is worth retrying, which is the single largest cause of "tool storm" runaway loops.

### Subagent local recovery vs coordinator escalation

A subagent owns local recovery for transient failures — retry with backoff, fall back to a secondary endpoint, serve a stale cached read — and only escalates upward when the error class is non-retryable *or* the local retry budget is exhausted. When it does escalate, it returns a structured envelope including the final `errorCategory`/`isRetryable`, a description for the coordinator's logs, **partial results** gathered before the failure, and **what was attempted** (which endpoints, how many retries) so the coordinator doesn't repeat the work.

Critically, a *valid empty result* (e.g. "no orders match this filter") is **not** an error. Conflating "I couldn't run the query" with "the query returned zero rows" causes the coordinator to retry needlessly or misreport state to the user.

## MCP server configuration in Claude Code

### Scopes: local / project / user

[Claude Code's MCP docs](https://docs.claude.com/en/docs/claude-code/mcp) define three scopes:

| Scope | Storage | Shared via git? | Use for |
| --- | --- | --- | --- |
| `local` (default) | `~/.claude.json`, keyed to the current project path | No | Personal/experimental servers in one project, secrets you don't want shared |
| `project` | `.mcp.json` at the repo root | **Yes** | Team-shared tooling (Jira, internal APIs, the team's deploy bot) |
| `user` | `~/.claude.json`, cross-project | No | Personal utilities you want everywhere (your scratch DB, your notes server) |

Add with the CLI:

```bash
claude mcp add --scope project --transport http jira https://mcp.atlassian.com/v1/sse
claude mcp add --scope user    --transport stdio notes -- npx -y @me/notes-mcp
claude mcp add --scope local   --transport stdio scratch -- python ./scratch_server.py
```

When the same server name exists at multiple scopes, **local wins, then project, then user**, then plugin-provided servers, then `claude.ai` connectors. That precedence lets a developer override a team-shared config locally without editing `.mcp.json`.

### `.mcp.json` schema (with example)

A project-scoped `.mcp.json` lives at the repo root and is checked in. Minimal schema:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp",
      "headers": {
        "Authorization": "Bearer ${GITHUB_PERSONAL_ACCESS_TOKEN}"
      }
    },
    "postgres-readonly": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL_RO}"],
      "env": {
        "PGSSLMODE": "require"
      }
    },
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```

Each entry's `type` is one of `stdio`, `http` (alias `streamable-http`), or `sse`. For `stdio` you supply `command` + optional `args` and `env`. For `http`/`sse` you supply `url` and optional `headers`. Project-scoped servers prompt for user approval the first time a session loads them (reset with `claude mcp reset-project-choices`).

### Environment variable expansion

Claude Code expands two forms inside `.mcp.json`:

- `${VAR}` — value of `VAR`; **fails to parse** if unset
- `${VAR:-default}` — value of `VAR` if set, else `default`

Expansion works in `command`, `args`, `env`, `url`, and `headers`. The recommended pattern: commit `.mcp.json` with `${GITHUB_TOKEN}` placeholders and keep real tokens in each developer's shell, CI runner, or 1Password CLI. Gotcha: `${CLAUDE_PROJECT_DIR}` is set in the *server's* environment, not Claude Code's — referencing it in a top-level `command` or `args` typically needs `${CLAUDE_PROJECT_DIR:-.}` as a default.

### Verifying connection (`/mcp` command)

Inside Claude Code, `/mcp` lists every connected server, the count of tools/resources/prompts it advertises, its auth state, and any reconnection backoff in progress. HTTP/SSE servers auto-reconnect up to five times with exponential backoff; stdio servers, being local subprocesses, do not. `/mcp` is also where you run the OAuth flow for remote servers that return `401`/`403` with a `WWW-Authenticate` header.

## MCP tools vs MCP resources

### When to model something as a tool

Tools are **model-controlled** and may have side effects. Use a tool when the agent must *decide* to do something:

- `create_jira_issue`, `update_pr_status`, `run_sql(query)`, `send_slack_message`
- Anything that mutates state, calls an external API, or requires arguments the model must reason about

### When to model it as a resource

Resources are **application-controlled**, read-only, identified by URIs, and ideally cacheable. Use a resource when the agent benefits from *passive context* without having to ask for it:

- A directory of every open Jira ticket in the current sprint → `jira://sprints/current/issues`
- The Postgres schema of the orders database → `postgres://schema/public/orders`
- The contents of a runbook or design doc → `confluence://pages/12345`

Because resources are URI-addressable and don't move state, they cost less in retries and are friendlier to caching than wrapping the same data as a tool call.

### Examples (issue catalog, schema introspection)

The guide's "content catalog" pattern: instead of forcing the agent to call `list_issues` then `get_issue` for each match (an exploratory N+1), the server exposes an `issues://summary` resource the host can attach automatically. The agent sees the catalog in context and jumps directly to the relevant `get_issue` call. The same idea applies to schema introspection — exposing `db://schema` as a resource turns "what columns does `orders` have?" into a zero-tool-call answer.

## Picking community vs custom servers

### When to build your own

Build a custom MCP server only when the integration is **team-specific** and no maintained community option exists: an internal microservice, your deploy pipeline, a homegrown CRM. The cost of a custom server is not the code — it's the ongoing maintenance: schema drift, auth refresh, transport upgrades, security review.

### Popular servers worth knowing

The [`modelcontextprotocol/servers`](https://github.com/modelcontextprotocol/servers) repo holds the reference implementations and links to the official [MCP Registry](https://registry.modelcontextprotocol.io/). Names worth remembering: **Filesystem** (sandboxed file ops), **Fetch** (URL → LLM-friendly text), **Git**, **Memory** (knowledge-graph persistence), **Sequential Thinking**, plus vendor-hosted **GitHub** (`api.githubcopilot.com/mcp`), **Postgres**, **Slack**, **Sentry**, **Notion**, **Atlassian/Jira**, **Stripe**, **PayPal**, **HubSpot**, **Asana**, and **Playwright**. Community directories ([mcpforge.org](https://www.mcpforge.org/directory), [mcpindex.net](https://mcpindex.net/), [mcpdir.dev](https://mcpdir.dev/)) catalog thousands more, but the official registry is the source of truth for provenance.

A final skill the guide calls out: **enhance MCP tool descriptions**. If your Jira MCP tool says "search Jira" while built-in `Grep` is described in vivid detail, the model will reach for `Grep` against a local cache instead. Tool descriptions are the model's only signal for tool choice — write them like the docstring of a function the model has never seen.

## Exam-style focus points

- `isError: true` lives inside `CallToolResult`, **not** as a JSON-RPC error — because the model must *see* the failure to recover.
- Always include `errorCategory` and `isRetryable` in `structuredContent`; "Operation failed" is the canonical wrong answer.
- Distinguish access failures from empty results; `isError: false` with empty `content` is a valid, common case.
- `.mcp.json` is project-scoped and committed; `~/.claude.json` is local/user-scoped and private.
- `${VAR}` and `${VAR:-default}` expansion is for keeping secrets out of git, not for runtime config logic.
- Tools = model-controlled, side-effectful; Resources = application-controlled, read-only, URI-addressable.
- Prefer community servers; custom servers exist for team-specific workflows only.
- Subagents recover transient errors locally; only non-recoverable errors (with partial results and attempt log) propagate to the coordinator.

## References

- MCP Specification (latest): https://modelcontextprotocol.io/specification/latest
- MCP Schema (2025-11-25): https://modelcontextprotocol.io/specification/2025-11-25/schema
- MCP Server Primitives Overview: https://modelcontextprotocol.io/specification/2025-11-25/server
- Claude Code — Connect to tools via MCP: https://docs.claude.com/en/docs/claude-code/mcp
- Claude Code — MCP installation scopes: https://docs.claude.com/en/docs/claude-code/mcp#mcp-installation-scopes
- Claude Code — Environment variable expansion in `.mcp.json`: https://docs.claude.com/en/docs/claude-code/mcp#environment-variable-expansion-in-mcp-json
- Reference servers: https://github.com/modelcontextprotocol/servers
- MCP Registry: https://registry.modelcontextprotocol.io/
