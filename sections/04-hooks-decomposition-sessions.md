---
title: "Section 4 — Agent SDK Hooks, Task Decomposition & Session Management"
linkTitle: "4. Hooks, Decomposition & Sessions"
weight: 4
description: "Domains 1.5–1.7 — PostToolUse and call-interception hooks, prompt chaining vs adaptive decomposition, fork_session and named-session resume."
---

## What this section covers

Three closely related architect-level skills that turn a Claude agent from a probabilistic chatbot into a controllable, debuggable system:

1. **Hooks (1.5)** — deterministic Python/TypeScript callbacks (or shell scripts) that intercept the agent loop at well-defined lifecycle points to enforce policy, normalize tool output, and audit every action.
2. **Task decomposition (1.6)** — knowing when to hard-wire a sequential pipeline (prompt chaining) vs. letting the model dynamically generate its own subtasks (orchestrator-workers / adaptive plans).
3. **Session management (1.7)** — the operational discipline of `--continue`, `--resume`, and `--fork-session`, plus the judgment of *when* to throw a session away and start fresh with a structured summary.

Pass criterion: look at a workflow and immediately say "this is a hook problem, not a prompt problem", "this is prompt chaining, not orchestrator-workers", or "this session is stale, summarize and restart."

## Source material (from official guide)

### 1.5 Hooks for interception & normalization

`PostToolUse` intercepts tool *results* and transforms them before the model sees raw bytes. `PreToolUse` intercepts tool *calls* and can block, modify, or redirect — canonical example: "block any refund where `amount > 500` and route to human escalation." Hooks give **deterministic guarantees**; prompts give only **probabilistic compliance**. Skills: normalize heterogeneous timestamps across MCP servers; block policy-violating actions and redirect to alternatives; choose hooks over prompts when compliance must be guaranteed.

### 1.6 Task decomposition strategies

**Prompt chaining** for predictable workflows where steps are known in advance vs. **dynamic adaptive decomposition** for open-ended workflows where subtasks can only be discovered at runtime. Large code reviews: per-file local analysis + a separate cross-file integration pass to avoid attention dilution.

### 1.7 Session state, resumption & forking

`--resume <id-or-name>` continues a specific prior conversation; `--continue`/`-c` resumes the most recent in this `cwd`. `--fork-session` (`fork_session: true` / `forkSession: true` in the SDK) creates an independent branch from a shared baseline. After files change on disk, inform a resumed agent or its cached `Read` results are stale; a fresh session seeded with a hand-crafted summary is sometimes more reliable than resuming with stale tool results.

Source: `guide.txt` lines 229–305.

## Hooks reference

### Hook event types

| Event | When it fires | Typical use |
| --- | --- | --- |
| `SessionStart` | Session begins or resumes (matchers: `startup`, `resume`, `clear`, `compact`) | Init logging, inject project rules |
| `SessionEnd` | Session terminates | Flush logs, clean up |
| `UserPromptSubmit` | User submits a prompt, before model sees it | Inject context, scrub PII, block off-topic prompts |
| `PreToolUse` | Before any tool call executes | **Policy enforcement**, input rewriting, sandbox redirection |
| `PostToolUse` | After a tool call succeeds | **Data normalization**, audit logging, format conversion |
| `PostToolUseFailure` | After a tool call fails | Custom error handling |
| `PostToolBatch` | A parallel batch of tool calls resolves | Inject conventions once per batch |
| `PermissionRequest` / `PermissionDenied` | Permission dialog or auto-mode denial | Custom UX, retry decisions |
| `SubagentStart` / `SubagentStop` | Subagent spawns / finishes | Track parallel work, aggregate results |
| `PreCompact` / `PostCompact` | Conversation compaction lifecycle | Archive transcript before lossy summary |
| `Notification` | Agent emits a notification | Forward to Slack/PagerDuty |
| `Stop` / `StopFailure` | Turn ends normally / via API error | Save state, alert on rate-limit |
| `TaskCreated` / `TaskCompleted` | Task lifecycle | Enforce ticket-ID conventions, gate on tests passing |
| `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `FileChanged`, `WorktreeCreate/Remove`, `Setup` | Misc lifecycle | Audit, reload config, react to external changes |

`SessionStart`/`SessionEnd` are TS-SDK callbacks only; in Python they must be shell hooks in `.claude/settings.json` plus `setting_sources=["project"]`.

### Anatomy of a hook

Two registration paths: **SDK callbacks** (`ClaudeAgentOptions.hooks` / `options.hooks`) or **shell-command hooks** in `.claude/settings.json` — child processes that get event JSON on stdin.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }
        ]
      }
    ]
  }
}
```

**Matchers**: `*`/`""`/omitted matches all; letters+digits+`|` is exact / pipe-list (`Edit|Write`); anything else is a JS regex (`^mcp__memory__`). Tool hooks match on tool *name* only — filter `file_path` inside the handler.

**Input** — every hook receives `session_id`, `cwd`, `hook_event_name` plus event-specific fields (`tool_name`, `tool_input`, `tool_response`, …). Subagent context adds `agent_id`, `agent_type`.

**Output** — JSON in two layers: top-level (`systemMessage`, `continue` / `continue_`, `additionalContext`) and `hookSpecificOutput` (event-dependent). For `PreToolUse`: `permissionDecision` ∈ `{"allow", "deny", "ask", "defer"}`, `permissionDecisionReason`, `updatedInput`. For `PostToolUse`: `additionalContext` (append) or `updatedToolOutput` (replace).

**Shell-hook exit codes**: `0` = success, parse stdout as JSON; `2` = blocking error, stderr fed to model (for `PreToolUse` blocks the call); any other non-zero = non-blocking error.

When multiple hooks fire on the same event: **deny > defer > ask > allow** — a single `deny` blocks.

### Concrete examples

**1. PostToolUse normalizing heterogeneous date formats** — three MCP servers return Unix epoch seconds, ISO 8601 strings, and numeric millisecond integers. Force one ISO 8601 representation before the model has to reason about it.

```python
from datetime import datetime, timezone

async def normalize_timestamps(input_data, tool_use_id, context):
    if input_data["hook_event_name"] != "PostToolUse":
        return {}
    response = input_data.get("tool_response", {})
    raw_ts = response.get("timestamp")
    if raw_ts is None:
        return {}

    if isinstance(raw_ts, (int, float)):
        ts = raw_ts / 1000 if raw_ts > 1e12 else raw_ts
        iso = datetime.fromtimestamp(ts, tz=timezone.utc).isoformat()
    else:
        iso = datetime.fromisoformat(str(raw_ts).replace("Z", "+00:00")).isoformat()

    response["timestamp"] = iso
    return {
        "hookSpecificOutput": {
            "hookEventName": "PostToolUse",
            "updatedToolOutput": response,
        }
    }

options = ClaudeAgentOptions(
    hooks={"PostToolUse": [HookMatcher(matcher="^mcp__", hooks=[normalize_timestamps])]}
)
```

The model only ever sees ISO 8601, so cross-tool date arithmetic just works.

**2. PreToolUse blocking high-value refunds** — guarantee that `process_refund` can never fire above $500.

```typescript
import { HookCallback, PreToolUseHookInput } from "@anthropic-ai/claude-agent-sdk";

const refundGuard: HookCallback = async (input) => {
  if (input.hook_event_name !== "PreToolUse") return {};
  const pre = input as PreToolUseHookInput;
  if (pre.tool_name !== "mcp__billing__process_refund") return {};

  const amount = (pre.tool_input as { amount?: number }).amount ?? 0;
  if (amount > 500) {
    return {
      systemMessage: `Refund of $${amount} exceeds policy cap; escalating.`,
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny",
        permissionDecisionReason:
          "Refunds above $500 require human approval. Use mcp__support__create_escalation instead."
      }
    };
  }
  return {};
};
```

`permissionDecisionReason` is the magic ingredient — it tells the model *why* the call was blocked and what alternative tool to use, so the agent self-corrects on the next turn instead of looping on the same denied call.

### When NOT to use a hook (use a prompt instead)

| Situation | Hook | Prompt |
| --- | --- | --- |
| Hard regulatory rule ("never log SSNs") | Yes | No |
| Data shape contract ("always ISO 8601") | Yes | No |
| Audit log of every tool call | Yes | No |
| Per-call quota / cost cap | Yes (PreToolUse) | No |
| Soft style preference ("prefer 2-space indent") | No | Yes |
| Persona / tone ("be concise, no emojis") | No | Yes |

Rule of thumb: if violating it is a P0 incident, it goes in a hook.

## Decomposition patterns

### Prompt chaining (sequential, predictable)

A fixed N-step pipeline where step *k* feeds step *k+1*. Each LLM call does an easier task than a single mega-prompt would. From [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents): *"ideal for situations where the task can be easily and cleanly decomposed into fixed subtasks. The main goal is to trade off latency for higher accuracy, by making each LLM call an easier task."*

Add programmatic *gates* between steps to fail fast: outline → check rubric (gate) → write document; per-file lint analysis → cross-file integration review.

### Dynamic adaptive decomposition

A.k.a. **orchestrator-workers**. The orchestrator LLM looks at the input, decides what subtasks are needed (it could not have known in advance), spawns workers, and synthesizes results. *"Well-suited for complex tasks where you can't predict the subtasks needed (in coding, the number of files that need to be changed and the nature of the change in each file likely depend on the task)."*

In the Agent SDK this maps to the `Task` / subagent pattern, optionally tracked via `SubagentStart`/`SubagentStop` hooks.

### Decision matrix

| Workflow shape | Pattern |
| --- | --- |
| Steps known in advance, identical for every input | Prompt chaining |
| Independent subtasks of *known* shape | Parallelization (sectioning) |
| Same task, want N votes for confidence | Parallelization (voting) |
| Distinct categories, each with a specialist | Routing |
| Subtask count/shape depends on input | Orchestrator-workers (adaptive) |
| Output benefits from critique loop | Evaluator-optimizer |
| Open-ended, multi-turn, unknown horizon | Full agent loop |

### Worked example — per-file + cross-file code review

A 40-file PR jammed into one prompt suffers attention dilution: the model skims and misses bugs. Decompose:

**Phase 1 — per-file local analysis (chained, parallelizable)**: each file gets its own context, forked from a shared baseline session that has already loaded `CLAUDE.md`, the PR description, and the diff stat. Forking avoids re-paying tokens to re-establish context per file.

```python
async def review_file(path: str, baseline_sid: str) -> dict:
    async for msg in query(
        prompt=f"Review {path}. Find correctness bugs, missing error handling, "
               f"and security issues. Output JSON {{'file','findings'}}.",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Grep"],
            resume=baseline_sid,
            fork_session=True,
            max_turns=8,
        ),
    ):
        if isinstance(msg, ResultMessage) and msg.subtype == "success":
            return json.loads(msg.result)
```

**Phase 2 — cross-file integration pass (single call):**

```python
findings = await asyncio.gather(*(review_file(p, baseline_sid) for p in changed_files))
async for msg in query(
    prompt=f"Per-file findings: {json.dumps(findings)}. "
           f"Identify cross-cutting issues: API contract drift, type mismatches, "
           f"missing call-site updates, security holes spanning files.",
    options=ClaudeAgentOptions(resume=baseline_sid, allowed_tools=["Read", "Grep"]),
):
    ...
```

Prompt chaining (Phase 1 → Phase 2) layered on top of parallelization within Phase 1.

**Open-ended variant — "add tests to a legacy codebase":** *adaptive*, not chained. Map structure → identify high-impact untested modules → build a prioritized backlog → take the top item, write tests, discover a hidden dependency, push it back onto the backlog, repeat. Step 4..N is only knowable at runtime — orchestrator-workers, not chaining.

## Session management

### `--resume` vs `--continue` vs new session

| Need | Use | How it finds the session |
| --- | --- | --- |
| Most recent session in this directory | `claude -c` / `continue: true` | Newest in `~/.claude/projects/<cwd-slug>/` |
| Specific named or ID'd session | `claude -r "auth-refactor"` / `resume: "<id>"` | Exact ID or `--name` lookup |
| Brand-new conversation | `claude` | Fresh session ID |
| One-shot, no disk persistence (TS only) | `persistSession: false` | In-memory only |

Sessions live at `~/.claude/projects/<slugified-cwd>/<session-id>.jsonl`, where the slug is the absolute working directory with every non-alphanumeric char replaced by `-`. **A different `cwd` means `resume` cannot find the file** — the #1 cause of "why is resume returning a fresh session."

Capture the session ID from the `ResultMessage` (Python) / `SDKResultMessage` (TS) on every run if you intend to resume programmatically. In TS it's also on the init `SystemMessage`.

### `fork_session`: when and how

Forking copies the existing transcript into a *new* session ID and lets it diverge. Original is untouched.

```python
forked_id = None
async for message in query(
    prompt="Try OAuth2 instead of JWT for the auth module",
    options=ClaudeAgentOptions(resume=session_id, fork_session=True),
):
    if isinstance(message, ResultMessage):
        forked_id = message.session_id
```

```typescript
for await (const message of query({
  prompt: "Try OAuth2 instead of JWT for the auth module",
  options: { resume: sessionId, forkSession: true }
})) {
  if (message.type === "system" && message.subtype === "init") {
    forkedId = message.session_id;
  }
}
```

Use cases: A/B compare two refactoring approaches from a shared baseline; testing-strategy bake-off; risky exploration with a guaranteed fall-back to the parent.

Caveat: forking branches the *conversation*, not the *filesystem*. If both forks edit files in the same repo, those edits collide. Combine with [file checkpointing](https://code.claude.com/docs/en/agent-sdk/file-checkpointing) or git worktrees (`claude -w <name>`) for true isolation.

### Stale-context decision tree

After a code change, before resuming an investigation:

```
Did the agent's last tool calls touch files that have since been edited?
├── No  → Resume normally.
└── Yes →
    Is the affected surface small AND were the edits surgical?
    ├── Yes → Resume + tell the agent: "Files X, Y were edited since
    │         you last saw them. Re-Read them before continuing."
    └── No  → Throw the session away. Start fresh with a structured
              summary: (a) goal, (b) decisions made, (c) current
              state of the code, (d) open questions.
```

A clean session with a curated summary often beats a resume because the resumed transcript still contains stale `Read` outputs the model trusts — it may "remember" function signatures that no longer exist and hallucinate calls. Pattern: keep a long-lived **named session** (`claude -n design-review`) for stable architectural context and **fork** it per investigation. Throw away the forks.

## Exam-style focus points

- **Hooks are deterministic; prompts are probabilistic.** Any rule that must hold 100% of the time goes in `PreToolUse` (outgoing) or `PostToolUse` (incoming).
- Memorize `permissionDecision` values: `allow`, `deny`, `ask`, `defer`. Memorize priority: **deny > defer > ask > allow**.
- `PostToolUse.updatedToolOutput` *replaces* what the model sees; `additionalContext` *appends*. `PreToolUse.updatedInput` rewrites the tool input — but only if you also return `permissionDecision: "allow"`.
- Shell-hook exit codes: `0` = parse JSON; `2` = block (PreToolUse) / feed stderr to model; anything else = non-blocking error.
- Decomposition picker: known-shape → prompt chaining; unknown-shape → orchestrator-workers. Big code review = per-file pass + cross-file integration pass.
- `--continue` needs no ID but only finds the most recent session in the *current* `cwd`. `--resume` needs an ID or `--name`. `--fork-session` requires `--resume`/`--continue` and yields a new ID.
- Session storage: `~/.claude/projects/<slugified-cwd>/<session-id>.jsonl`. A wrong `cwd` is the #1 reason resume silently returns a fresh session.
- Stale tool results in a resumed session can hurt more than they help — sometimes a fresh session with a curated summary outperforms a resume.
- Fork to compare; resume to continue; restart-with-summary when context has rotted.

## References

- [Intercept and control agent behavior with hooks](https://code.claude.com/docs/en/agent-sdk/hooks) — full Agent SDK hook reference, event table, callback shape, examples.
- [Hooks reference](https://code.claude.com/docs/en/hooks) — every event, matcher patterns, JSON output, exit-code semantics, shell/HTTP/MCP-tool hook variants.
- [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide) — quickstart with worked examples.
- [Configure hooks (Anthropic blog)](https://claude.com/blog/how-to-configure-hooks) — power-user `settings.json` walkthrough.
- [Work with sessions](https://code.claude.com/docs/en/agent-sdk/sessions) — `continue`, `resume`, `fork_session`, capturing IDs, cross-host caveats.
- [CLI reference](https://code.claude.com/docs/en/cli-reference) — `--continue`, `--resume`, `--fork-session`, `--name`, `--session-id`, `--from-pr`.
- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — Anthropic's canonical taxonomy: prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer.
- [Anthropic cookbook — agents patterns](https://github.com/anthropics/claude-cookbooks/tree/main/patterns/agents) — runnable notebooks for each pattern.
- [How the agent loop works](https://code.claude.com/docs/en/agent-sdk/agent-loop) — conceptual model of turns, tool calls, where hooks fit in.
- [Permissions](https://code.claude.com/docs/en/agent-sdk/permissions) — companion mechanism to `PreToolUse` for tool-level access control.
