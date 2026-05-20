---
title: "Раздел 5 — Tool Interface Design, Distribution и Built-in Tools"
linkTitle: "5. Tool Design & Built-ins"
weight: 5
description: "Domains 2.1, 2.3, 2.5 — различающие описания tools, tool_choice, scoped tool access и built-ins Read/Write/Edit/Bash/Grep/Glob."
---

## Что покрывает этот раздел

Три тесно связанные темы из Domain 2:

- **2.1** — Как писать tool definitions (`name`, `description`, `input_schema`, `input_examples`), чтобы Claude reliably picks the right tool, включая диагностику и исправление description-driven misrouting.
- **2.3** — Как scope tools для каждого agent или subagent, и как использовать параметр `tool_choice` (`auto`, `any`, forced tool, `none`) плюс `disable_parallel_tool_use` для control invocation behavior.
- **2.5** — Как применять built-in tool set Claude Code (`Read`, `Write`, `Edit`, `Bash`, `Grep`, `Glob`, plus `WebFetch`, `WebSearch`, `NotebookEdit`, `Agent`, `Task*`) к реальной работе с codebase, включая canonical pattern Edit-fails-on-non-unique-anchor → Read+Write fallback.

Два scored items напрямую опираются на этот материал: Sample Question 2 (минимальные descriptions `get_customer` / `lookup_order`) и Sample Question 9 (scoping `verify_fact` to the synthesis agent).

## Исходный материал (из официального guide)

### 2.1 Tool interfaces with clear descriptions

Tool descriptions — *primary mechanism*, по которому LLM chooses tools. Minimal descriptions cause unreliable selection between similar tools — `analyze_content` vs `analyze_document`, or `get_customer` vs `lookup_order`. Descriptions must include input formats, example queries, edge cases, and explicit boundary statements ("use this tool when…, do **not** use this tool when…"). System prompt wording is keyword-sensitive and can override an otherwise good description, so system prompt is part of tool-routing surface. Expected fixes: differentiate descriptions on purpose / inputs / outputs / when-to-use, **rename** for overlap (`analyze_content` → `extract_web_results`), **split** generic tools into purpose-specific ones (`analyze_document` → `extract_data_points` + `summarize_content` + `verify_claim_against_source`), and audit system prompts for keyword bleed.

### 2.3 Tool distribution & tool_choice

Giving an agent 18 tools instead of 4–5 measurably degrades selection reliability, and agents with out-of-specialization tools tend to misuse them (synthesis agent attempting web searches). Fix is **scoped tool access** plus small number of **scoped cross-role tools** for high-frequency needs (например, `verify_fact` wired into synthesis agent so it does not round-trip through coordinator). `tool_choice` has four values — `auto`, `any`, `{"type": "tool", "name": "..."}`, and `none` — covered below.

### 2.5 Built-in tools (Read/Write/Edit/Bash/Grep/Glob)

`Grep` searches file contents (regex); `Glob` matches file *paths* (`**/*.test.tsx`). `Read`/`Write` are full-file ops; `Edit` performs unique-string replacement and **fails** if `old_string` is not unique or file has not been read in current session — correct fallback is `Read` then `Write`. Build understanding incrementally: `Grep` for entry points, `Read` to follow imports, then trace usage by enumerating exported names and `Grep`-ing each one.

## Writing a great tool description

Anthropic's "Define tools" guide is unambiguous: **detailed descriptions are by far the most important factor in tool performance**, with target "at least 3–4 sentences per tool description, more if the tool is complex." ([Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools))

### Anatomy of a description (the 5 elements)

1. **What the tool does** — action in one sentence.
2. **When to use it (and when not to)** — boundary statement that disambiguates against neighboring tools.
3. **Inputs** — what each parameter means, accepted formats, examples (`AAPL` for ticker), required vs optional.
4. **Outputs** — what the tool returns and what it deliberately does *not* return.
5. **Caveats / limitations** — rate limits, freshness, regional scope, units.

Plus optional `input_examples` array — schema-validated example inputs that help with complex or nested parameter shapes. Examples cost ~20–50 tokens for simple inputs and ~100–200 for nested objects.

### Before/after examples (bad vs good)

Bad — canonical Anthropic example:

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

Good — canonical Anthropic counter-example:

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

Good description tells Claude *what*, *when*, *what it returns*, and *what it does not return* — exactly the four gaps that cause `get_customer` to be picked over `lookup_order` in Sample Question 2.

## Tool naming & decomposition

### Splitting an over-broad tool

Single `analyze_document` tool with vague description forces Claude to guess. Decompose into purpose-specific tools whose names *are* the disambiguator:

| Original | Replacement |
| --- | --- |
| `analyze_document` | `extract_data_points`, `summarize_content`, `verify_claim_against_source` |
| `fetch_url` (generic) | `load_document` (validates the URL is a document) |
| `analyze_content` (overlaps web + doc) | `extract_web_results` (web only) |

Anthropic's [writing-tools-for-agents guide](https://www.anthropic.com/engineering/writing-tools-for-agents) makes the inverse point: avoid one-tool-per-endpoint sprawl (`list_users`, `list_events`, `create_event`) when consolidated `schedule_event` better matches actual workflow. Rule is *one tool per natural subdivision of a task*, not one tool per API call.

### Renaming for clarity

Use **service prefixes** (`github_list_prs`, `slack_send_message`, `asana_search`) when same agent has tools across multiple systems. Anthropic explicitly notes that prefix vs suffix namespacing has measurable effects on evaluation accuracy. Rename tools that overlap in meaning until each name is unambiguous in isolation — `analyze_content` and `analyze_document` are textbook collision; `extract_web_results` and `summarize_pdf` are not.

## Distributing tools across agents

### Why fewer tools = better selection

Anthropic's MCP evaluations on Opus 4 measured tool selection accuracy collapsing as tool count grew: 50+ tools loaded upfront landed at 49% accuracy, recovering to 74% only when [Tool Search](https://code.claude.com/docs/en/agent-sdk/tool-search) deferred load was enabled. Guide's "18 vs 4–5" framing matches this curve — every additional tool increases chance of misrouting and burns context (50 tool definitions can cost 10–20K tokens).

Architectural takeaway: give each subagent the smallest tool set that lets it finish its role, and let coordinator hold larger surface. When tool set must remain large, enable Tool Search so only 3–5 relevant definitions materialize per turn.

### Scoped cross-role tools

Synthesis-agent pattern from Sample Question 9 is canonical exam case. Synthesis legitimately needs *some* fact lookup (85% of its verifications are simple), but full web-search toolset over-provisions it. Fix is a single, **constrained** `verify_fact` tool — narrow inputs, narrow outputs, no general fetch — that handles common case, while 15% complex investigations still route through coordinator to dedicated web search agent. This is principle of least privilege applied to tools.

### tool_choice configuration table

| `tool_choice` | Behavior | Use when |
| --- | --- | --- |
| `{"type": "auto"}` | Default with tools present. Claude decides whether to call any tool, possibly multiple in parallel. | General agent loops where model should reason about whether tools are needed. |
| `{"type": "any"}` | Claude must call some tool (any of provided ones). No conversational text emitted. | Structured-output agents where free-text answers are an error. Pair with `strict: true` for schema-guaranteed inputs. |
| `{"type": "tool", "name": "extract_metadata"}` | Forces Claude to call a specific tool. Assistant message is prefilled to a `tool_use` block. | "Run X first" pipelines (e.g., `extract_metadata` before enrichment). Process subsequent steps in follow-up turns. |
| `{"type": "none"}` | Default when no tools are provided. Claude cannot call tools. | Pure chat turns inside a tool-equipped session. |
| `disable_parallel_tool_use: true` | Modifier on any of the above. With `auto` Claude calls *at most one* tool; with `any`/`tool` exactly one. | When downstream tools have ordering dependencies or rate limits. |

Three constraints worth memorizing:

- `tool_choice: any` and forced-tool mode are **not compatible with extended thinking** — only `auto` and `none` work in that mode.
- Changing `tool_choice` invalidates cached message blocks under prompt caching (tools and system prompt remain cached).
- `disable_parallel_tool_use` must be set on the request that returns the `tool_use` block; setting it on follow-up has no retroactive effect.

References: [Define tools — Forcing tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools#forcing-tool-use) and [Parallel tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use).

## Built-in tools cheat-sheet

Current Claude Code built-in surface ([Tools reference](https://docs.claude.com/en/docs/claude-code/tools-reference)):

| Tool | Use when | Avoid when | Fallback / note |
| --- | --- | --- | --- |
| `Read` | Loading a known file path; viewing images, PDFs, notebooks. Required before any `Edit` or `Write` to existing file. | Listing a directory (use `Bash ls` or `Glob`). | Errors on oversize files — retry with `offset`/`limit`. |
| `Write` | Creating new files; overwriting after a `Read`. | Targeted edits inside a large file (use `Edit`). | Fails on existing files unread in session. |
| `Edit` | Targeted replacement of a unique string in previously-read file. | Anchor string non-unique, or file changed on disk after read. | Read again, then `Write` full updated content; or set `replace_all: true` if you genuinely want every occurrence. |
| `Bash` | Running scripts, package managers, git, formatters. Long-running processes via `run_in_background: true`. | Reading or writing files (use `Read`/`Write`/`Edit` so permissions and edit-eligibility checks apply). | Default 2-min timeout, raisable to 10 min; output capped at 30K chars (file written for rest). |
| `Grep` | Content search across codebase (function names, error strings, imports). Built on ripgrep — escape regex metacharacters. | Locating files purely by name pattern (use `Glob`). | Respects `.gitignore`; pass explicit path to bypass. Use `multiline: true` to cross line boundaries. |
| `Glob` | File-name pattern matching: `**/*.test.tsx`, `src/**/*.ts`. | Searching inside file contents (use `Grep`). | Capped at 100 results, sorted by mtime; does *not* respect `.gitignore` by default. |
| `WebFetch` | Pulling a known URL and extracting answers via small extractor model. | Needing raw HTML or specific selectors (use `Bash curl`). | Lossy by design — re-fetch with more specific extraction prompt if needed. Cached 15 min per URL. |
| `WebSearch` | Discovering URLs by query; up to 8 backend searches per call; supports `allowed_domains` *or* `blocked_domains` (not both). | Reading the result pages (chain `WebFetch` after). | Permission rule has no specifier — allow/deny whole tool. |
| `NotebookEdit` | Modifying a Jupyter cell by `cell_id` (`replace`, `insert`, `delete`). | Cross-cell text replacement. | Permission rules use `Edit(...)` path format. |
| `Agent` | Spawning a scoped subagent. | Routine operations parent can do directly. | Subagent `tools` / `disallowedTools` frontmatter scopes its surface. |
| `Task*` (`TaskCreate` / `TaskList` / `TaskUpdate`) | Multi-step work tracking in interactive sessions. | One-shot trivial tasks. | `TodoWrite` is headless-mode equivalent. |

Exam-relevant pattern is **Edit failure recovery loop**: `Edit` requires anchor to appear exactly once in a file already read this session. When anchor repeats — common in config files, repeated import lines, or boilerplate — correct response is `Read` (full file) → `Write` (full updated file), not increasingly creative regex tricks.

## Exam-style focus points

- "Both tools have minimal descriptions" → first fix is **expand the descriptions** (inputs, examples, edge cases, boundaries). Few-shot prompts, routing layers, and tool consolidation are wrong-first-step distractors. (Sample Q2.)
- "Synthesis agent needs frequent simple verifications" → give it a **scoped `verify_fact` tool**; do not hand it full web search toolset. Batching and speculative caching are wrong. (Sample Q9.)
- `tool_choice: any` ≠ "any tool I want" — it means *Claude must call some tool*, not free text.
- Forced tool use (`{"type": "tool", "name": "..."}`) is not compatible with extended thinking.
- `Glob` finds *files*, `Grep` finds *content*. `**/*.test.tsx` is a Glob pattern.
- `Edit` failures from non-unique anchors → `Read` + `Write`, not repeated `Edit` retries.
- Tool-set size matters: 18 tools is too many for one agent; 4–5 is target. Beyond 30–50 across session, enable Tool Search.
- `disable_parallel_tool_use: true` belongs on request that emits `tool_use` block.

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
