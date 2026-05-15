---
title: "Section 5 — Tool Interface Design, Distribution & Built-in Tools"
linkTitle: "5. Tool Design & Built-ins"
weight: 5
description: "Domains 2.1, 2.3, 2.5 — writing distinguishing tool descriptions, tool_choice, scoped tool access, and the Read/Write/Edit/Bash/Grep/Glob built-ins."
---

## What this section covers

Three tightly related topics from Domain 2:

- **2.1** — How to write tool definitions (`name`, `description`, `input_schema`, `input_examples`) so Claude reliably picks the right tool, including how to recognize and fix description-driven misrouting.
- **2.3** — How to scope which tools each agent or subagent sees, and how to use the `tool_choice` parameter (`auto`, `any`, forced tool, `none`) plus `disable_parallel_tool_use` to control invocation behavior.
- **2.5** — How to apply the Claude Code built-in tool set (`Read`, `Write`, `Edit`, `Bash`, `Grep`, `Glob`, plus `WebFetch`, `WebSearch`, `NotebookEdit`, `Agent`, `Task*`) to real codebase work, including the canonical Edit-fails-on-non-unique-anchor → Read+Write fallback pattern.

Two scored items lean directly on this material: Sample Question 2 (minimal `get_customer` / `lookup_order` descriptions) and Sample Question 9 (scoping `verify_fact` to the synthesis agent).

## Source material (from official guide)

### 2.1 Tool interfaces with clear descriptions

Tool descriptions are *the* primary mechanism the LLM uses to choose tools. Minimal descriptions cause unreliable selection between similar tools — `analyze_content` vs `analyze_document`, or `get_customer` vs `lookup_order`. Descriptions must include input formats, example queries, edge cases, and explicit boundary statements ("use this tool when…, do **not** use this tool when…"). System prompt wording is keyword-sensitive and can override an otherwise good description, so the system prompt is part of the tool-routing surface. Expected fixes: differentiate descriptions on purpose / inputs / outputs / when-to-use, **rename** for overlap (`analyze_content` → `extract_web_results`), **split** generic tools into purpose-specific ones (`analyze_document` → `extract_data_points` + `summarize_content` + `verify_claim_against_source`), and audit system prompts for keyword bleed.

### 2.3 Tool distribution & tool_choice

Giving an agent 18 tools instead of 4–5 measurably degrades selection reliability, and agents with out-of-specialization tools tend to misuse them (a synthesis agent attempting web searches). The fix is **scoped tool access** plus a small number of **scoped cross-role tools** for high-frequency needs (e.g., `verify_fact` wired into the synthesis agent so it does not round-trip through the coordinator). `tool_choice` has four values — `auto`, `any`, `{"type": "tool", "name": "..."}`, and `none` — covered in detail below.

### 2.5 Built-in tools (Read/Write/Edit/Bash/Grep/Glob)

`Grep` searches file contents (regex); `Glob` matches file *paths* (`**/*.test.tsx`). `Read`/`Write` are full-file ops; `Edit` performs a unique-string replacement and **fails** if `old_string` is not unique or the file has not been read in the current session — at which point the correct fallback is `Read` then `Write`. Build understanding incrementally: `Grep` for entry points, `Read` to follow imports, then trace usage by enumerating exported names and `Grep`-ing each one.

## Writing a great tool description

Anthropic's "Define tools" guide is unambiguous: **detailed descriptions are by far the most important factor in tool performance**, with a target of "at least 3–4 sentences per tool description, more if the tool is complex." ([Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools))

### Anatomy of a description (the 5 elements)

1. **What the tool does** — the action, in one sentence.
2. **When to use it (and when not to)** — the boundary statement that disambiguates against neighboring tools.
3. **Inputs** — what each parameter means, accepted formats, examples (`AAPL` for ticker), required vs optional.
4. **Outputs** — what the tool returns and what it deliberately does *not* return.
5. **Caveats / limitations** — rate limits, freshness, regional scope, units.

Plus the optional `input_examples` array — schema-validated example inputs that help with complex or nested parameter shapes. Examples cost ~20–50 tokens for simple inputs and ~100–200 for nested objects.

### Before/after examples (bad vs good)

Bad — the canonical Anthropic example:

```json
{
  "name": "get_stock_price",
  "description": "Gets the stock price for a ticker.",
  "input_schema": {
    "type": "object",
    "properties": { "ticker": { "type": "string" } },
    "required": ["ticker"]
  }
}
```

Good — the canonical Anthropic counter-example:

```json
{
  "name": "get_stock_price",
  "description": "Retrieves the current stock price for a given ticker symbol. The ticker symbol must be a valid symbol for a publicly traded company on a major US stock exchange like NYSE or NASDAQ. The tool will return the latest trade price in USD. It should be used when the user asks about the current or most recent price of a specific stock. It will not provide any other information about the stock or company.",
  "input_schema": {
    "type": "object",
    "properties": {
      "ticker": {
        "type": "string",
        "description": "The stock ticker symbol, e.g. AAPL for Apple Inc."
      }
    },
    "required": ["ticker"]
  },
  "input_examples": [
    { "ticker": "AAPL" },
    { "ticker": "MSFT" }
  ]
}
```

The good description tells Claude *what*, *when*, *what it returns*, and *what it does not return* — exactly the four gaps that cause `get_customer` to be picked over `lookup_order` in Sample Question 2.

## Tool naming & decomposition

### Splitting an over-broad tool

A single `analyze_document` tool with a vague description forces Claude to guess. Decompose into purpose-specific tools whose names *are* the disambiguator:

| Original | Replacement |
| --- | --- |
| `analyze_document` | `extract_data_points`, `summarize_content`, `verify_claim_against_source` |
| `fetch_url` (generic) | `load_document` (validates the URL is a document) |
| `analyze_content` (overlaps web + doc) | `extract_web_results` (web only) |

Anthropic's [writing-tools-for-agents guide](https://www.anthropic.com/engineering/writing-tools-for-agents) makes the inverse point: avoid one-tool-per-endpoint sprawl (`list_users`, `list_events`, `create_event`) when a consolidated `schedule_event` better matches the actual workflow. The rule is *one tool per natural subdivision of a task*, not one tool per API call.

### Renaming for clarity

Use **service prefixes** (`github_list_prs`, `slack_send_message`, `asana_search`) when the same agent has tools across multiple systems. Anthropic explicitly notes that prefix vs suffix namespacing has measurable effects on evaluation accuracy. Rename tools that overlap in meaning until each name is unambiguous in isolation — `analyze_content` and `analyze_document` are a textbook collision; `extract_web_results` and `summarize_pdf` are not.

## Distributing tools across agents

### Why fewer tools = better selection

Anthropic's MCP evaluations on Opus 4 measured tool selection accuracy collapsing as the tool count grew: 50+ tools loaded upfront landed at 49% accuracy, recovering to 74% only when [Tool Search](https://code.claude.com/docs/en/agent-sdk/tool-search) deferred load was enabled. The guide's "18 vs 4–5" framing matches this curve — every additional tool increases the chance of misrouting and burns context (50 tool definitions can cost 10–20K tokens).

The architectural takeaway: give each subagent the smallest tool set that lets it finish its role, and let a coordinator hold the larger surface. When a tool set must remain large, enable Tool Search so only 3–5 relevant definitions are materialized per turn.

### Scoped cross-role tools

The synthesis-agent pattern from Sample Question 9 is the canonical exam case. Synthesis legitimately needs *some* fact lookup (85% of its verifications are simple), but giving it the full web-search toolset over-provisions it. The fix is a single, **constrained** `verify_fact` tool — narrow inputs, narrow outputs, no general fetch — that handles the common case, while the 15% of complex investigations still route through the coordinator to the dedicated web search agent. This is the principle of least privilege applied to tools.

### tool_choice configuration table

| `tool_choice` | Behavior | Use when |
| --- | --- | --- |
| `{"type": "auto"}` | Default with tools present. Claude decides whether to call any tool, possibly multiple in parallel. | General agent loops where the model should reason about whether tools are needed. |
| `{"type": "any"}` | Claude must call some tool (any of the provided ones). No conversational text is emitted. | Structured-output agents where free-text answers are an error. Pair with `strict: true` for schema-guaranteed inputs. |
| `{"type": "tool", "name": "extract_metadata"}` | Forces Claude to call a specific tool. The assistant message is prefilled to a `tool_use` block. | "Run X first" pipelines (e.g., `extract_metadata` before enrichment). Process subsequent steps in follow-up turns. |
| `{"type": "none"}` | Default when no tools are provided. Claude cannot call tools. | Pure chat turns inside a tool-equipped session. |
| `disable_parallel_tool_use: true` | Modifier on any of the above. With `auto` Claude calls *at most one* tool; with `any`/`tool` exactly one. | When downstream tools have ordering dependencies or rate limits. |

Three constraints worth memorizing:

- `tool_choice: any` and forced-tool mode are **not compatible with extended thinking** — only `auto` and `none` work in that mode.
- Changing `tool_choice` invalidates cached message blocks under prompt caching (tools and system prompt remain cached).
- `disable_parallel_tool_use` must be set on the request that returns the `tool_use` block; setting it on a follow-up has no retroactive effect.

References: [Define tools — Forcing tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools#forcing-tool-use) and [Parallel tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use).

## Built-in tools cheat-sheet

The current Claude Code built-in surface ([Tools reference](https://docs.claude.com/en/docs/claude-code/tools-reference)):

| Tool | Use when | Avoid when | Fallback / note |
| --- | --- | --- | --- |
| `Read` | Loading a known file path; viewing images, PDFs, notebooks. Required before any `Edit` or `Write` to an existing file. | Listing a directory (use `Bash ls` or `Glob`). | Errors on oversize files — retry with `offset`/`limit`. |
| `Write` | Creating new files; overwriting after a `Read`. | Targeted edits inside a large file (use `Edit`). | Fails on existing files unread in the session. |
| `Edit` | Targeted replacement of a unique string in a previously-read file. | The anchor string is non-unique, or the file changed on disk after the read. | Read again, then `Write` the full updated content; or set `replace_all: true` if you genuinely want every occurrence. |
| `Bash` | Running scripts, package managers, git, formatters. Long-running processes via `run_in_background: true`. | Reading or writing files (use `Read`/`Write`/`Edit` so permissions and edit-eligibility checks apply). | Default 2-min timeout, raisable to 10 min; output capped at 30K chars (file written for the rest). |
| `Grep` | Content search across the codebase (function names, error strings, imports). Built on ripgrep — escape regex metacharacters. | Locating files purely by name pattern (use `Glob`). | Respects `.gitignore`; pass an explicit path to bypass. Use `multiline: true` to cross line boundaries. |
| `Glob` | File-name pattern matching: `**/*.test.tsx`, `src/**/*.ts`. | Searching inside file contents (use `Grep`). | Capped at 100 results, sorted by mtime; does *not* respect `.gitignore` by default. |
| `WebFetch` | Pulling a known URL and extracting answers via a small extractor model. | Needing the raw HTML or specific selectors (use `Bash curl`). | Lossy by design — re-fetch with a more specific extraction prompt if needed. Cached 15 min per URL. |
| `WebSearch` | Discovering URLs by query; up to 8 backend searches per call; supports `allowed_domains` *or* `blocked_domains` (not both). | Reading the result pages (chain `WebFetch` after). | Permission rule has no specifier — allow/deny the whole tool. |
| `NotebookEdit` | Modifying a Jupyter cell by `cell_id` (`replace`, `insert`, `delete`). | Cross-cell text replacement. | Permission rules use the `Edit(...)` path format. |
| `Agent` | Spawning a scoped subagent. | Routine operations the parent can do directly. | Subagent `tools` / `disallowedTools` frontmatter scopes its surface. |
| `Task*` (`TaskCreate` / `TaskList` / `TaskUpdate`) | Multi-step work tracking in interactive sessions. | One-shot trivial tasks. | `TodoWrite` is the headless-mode equivalent. |

The exam-relevant pattern is the **Edit failure recovery loop**: `Edit` requires the anchor to appear exactly once in a file already read this session. When the anchor repeats — common in config files, repeated import lines, or boilerplate — the correct response is `Read` (full file) → `Write` (full updated file), not increasingly creative regex tricks.

## Exam-style focus points

- "Both tools have minimal descriptions" → first fix is **expand the descriptions** (inputs, examples, edge cases, boundaries). Few-shot prompts, routing layers, and tool consolidation are all wrong-first-step distractors. (Sample Q2.)
- "Synthesis agent needs frequent simple verifications" → give it a **scoped `verify_fact` tool**; do not hand it the full web search toolset. Batching and speculative caching are wrong. (Sample Q9.)
- `tool_choice: any` ≠ "any tool I want" — it means *Claude must call some tool*, not free text.
- Forced tool use (`{"type": "tool", "name": "..."}`) is not compatible with extended thinking.
- `Glob` finds *files*, `Grep` finds *content*. `**/*.test.tsx` is a Glob pattern.
- `Edit` failures from non-unique anchors → `Read` + `Write`, not repeated `Edit` retries.
- Tool-set size matters: 18 tools is too many for one agent; 4–5 is the target. Beyond 30–50 across the session, enable Tool Search.
- `disable_parallel_tool_use: true` belongs on the request that emits the `tool_use` block.

## References

- [Define tools — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- [Tool reference — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference)
- [Parallel tool use — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use)
- [Strict tool use — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use)
- [Writing effective tools for AI agents — Anthropic Engineering](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Tools reference — Claude Code Docs](https://docs.claude.com/en/docs/claude-code/tools-reference)
- [Scale to many tools with tool search — Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/tool-search)
- [Configure permissions — Claude Code Docs](https://code.claude.com/docs/en/permissions)
- [anthropics/skills — claude-api/shared/tool-use-concepts.md](https://github.com/anthropics/skills/blob/main/skills/claude-api/shared/tool-use-concepts.md)
