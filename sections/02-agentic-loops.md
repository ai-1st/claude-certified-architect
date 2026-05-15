---
title: "Section 2 — Agentic Loops & stop_reason Handling"
linkTitle: "2. Agentic Loops"
weight: 2
description: "Domain 1.1 — agentic loop control flow, stop_reason values you must know, and the canonical anti-patterns."
---

## What this section covers

How to build the central control flow of a Claude agent: send a request, inspect `stop_reason`, run any tools Claude asked for, append results to history, iterate. Every higher-level pattern (orchestrator-workers, subagents, evaluator-optimizer, Agent SDK) is built on top of this loop.

## Source material (from official guide)

### Knowledge required

- The agentic loop lifecycle: send request to Claude, inspect `stop_reason` (`"tool_use"` vs `"end_turn"`), execute the requested tools, return results for the next iteration.
- How tool results are appended to conversation history so the model can reason about the next action.
- The distinction between **model-driven decision-making** (Claude reasons about which tool to call next based on context) and **pre-configured decision trees** (the developer hardcodes the tool sequence).

### Skills required

- Implement agentic loop control flow that continues while `stop_reason == "tool_use"` and terminates when `stop_reason == "end_turn"`.
- Append tool results to the conversation context between iterations so the model can incorporate new information into its reasoning.
- Avoid anti-patterns: parsing natural language signals to terminate the loop, using arbitrary iteration caps as the primary stop mechanism, or checking assistant text content as a completion indicator.

## The agentic loop, end-to-end

Anthropic's working definition of an agent is the simplest one in the field: *"LLMs autonomously using tools in a loop."* The augmented LLM (model + tools + retrieval + memory) is the foundational building block — every workflow pattern (prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer) is composed of it.

```text
  ┌──────────────────────────────────────────────────────────────┐
  │  user prompt + tool definitions  ─────────► messages array   │
  └──────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  POST /v1/messages  (Claude reasons about the next action)   │
  └──────────────────────────────────────────────────────────────┘
                              │
                              ▼
            ┌──────  inspect response.stop_reason  ──────┐
            │                                            │
   "tool_use"                                       "end_turn"
            │                                            │
            ▼                                            ▼
  ┌───────────────────────────┐               ┌─────────────────┐
  │ 1. append assistant turn  │               │ return final    │
  │    (incl. tool_use blocks)│               │ text to caller  │
  │ 2. execute each tool      │               └─────────────────┘
  │ 3. append a user turn     │
  │    with tool_result blocks│
  │ 4. loop back to /messages │
  └───────────────────────────┘
```

Walkthrough of one iteration:

1. Send `messages` plus the `tools` schema to `POST /v1/messages`.
2. Claude returns an `assistant` message. Its `content` is a list of blocks: zero or more `text` blocks and zero or more `tool_use` blocks. The top-level `stop_reason` summarizes why generation stopped.
3. If `stop_reason == "tool_use"`: append the assistant turn verbatim, execute each requested tool, append a single new `user` turn whose content is a list of `tool_result` blocks (one per `tool_use_id`), and call the API again with the updated history.
4. If `stop_reason == "end_turn"`: the model has decided the task is finished. Return.

Tool results are **appended to conversation history**, not summarized away. Each new request carries the entire history, so Claude can chain reasoning across many turns. The model — not your code — decides which tool to call next based on what it observed. This is the difference between **model-driven decision-making** (Claude picks tool N+1 from the running context) and **pre-configured decision trees** (your code statically calls `tool_a()` → `tool_b()` → `tool_c()`). Decision trees are workflows; agentic loops are agents. Anthropic's published guidance is to prefer the simpler workflow whenever the path can be hardcoded.

## stop_reason values you must know

`stop_reason` is part of every successful Messages API response. It is the only signal you should branch on to decide whether to keep looping. The full set of documented values is below.

| Value | Meaning | What your loop should do |
| --- | --- | --- |
| `end_turn` | Claude finished its response naturally. | Exit the loop. Return `response.content` text blocks to the caller. |
| `tool_use` | Response contains one or more `tool_use` blocks; Claude expects you to execute them. | Append the assistant turn, run every `tool_use` block, append a `user` turn with matching `tool_result` blocks (use the same `tool_use_id`), and call the API again. |
| `max_tokens` | Output hit the `max_tokens` parameter. The response is truncated and may contain an **incomplete** `tool_use` block. | Detect mid-tool-call truncation by checking the last content block's `type == "tool_use"`; retry with a higher `max_tokens`. Otherwise, prompt for continuation or surface a truncation warning. |
| `stop_sequence` | Output matched a custom string in `stop_sequences`. The matched sequence is in `response.stop_sequence`. | Treat as a successful terminal stop for that pattern. Continue or finalize depending on your protocol. |
| `pause_turn` | The server-side sampling loop hit its iteration cap while running **server tools** (web search, web fetch, code execution, etc.). The response may contain a `server_tool_use` block with no matching `server_tool_result`. | Append the assistant response **unchanged** and call the API again with the same tools. Repeat until you get a non-`pause_turn` stop reason. |
| `refusal` | The model declined for safety reasons (Sonnet 4.5+ / Opus 4.1+ API safety filter). | Do not loop. Surface a refusal to the caller; optionally rephrase, route to a different model (e.g. Haiku 4.5), or escalate. |
| `model_context_window_exceeded` | Generation stopped because the response reached the model's full context window (not `max_tokens`). Sonnet 4.5+ by default; earlier models need a beta header. | Treat similarly to `max_tokens` — the response is valid but capped. Continue, summarize, or compact context. |

Branching on `stop_reason` is the **only** correct termination test. Do not parse text like "I'm done" or "Final answer:" — that is the canonical anti-pattern below.

## Reference implementations

### Python — raw Messages API loop

Minimal, runnable shape using the `anthropic` Python SDK (the same loop pattern Anthropic shows in [their docs](https://docs.anthropic.com/en/docs/build-with-claude/handling-stop-reasons#handling-tool-use-workflows)).

```python
from anthropic import Anthropic

client = Anthropic()
MODEL = "claude-opus-4-7"

tools = [{
    "name": "get_weather",
    "description": "Get current weather for a city.",
    "input_schema": {
        "type": "object",
        "properties": {"location": {"type": "string"}},
        "required": ["location"],
    },
}]

def run_tool(name: str, tool_input: dict) -> str:
    if name == "get_weather":
        return f"Weather in {tool_input['location']}: 72F, clear"
    raise ValueError(f"unknown tool: {name}")

def agent_loop(user_prompt: str) -> str:
    messages = [{"role": "user", "content": user_prompt}]
    while True:
        resp = client.messages.create(
            model=MODEL, max_tokens=4096, tools=tools, messages=messages,
        )
        if resp.stop_reason == "end_turn":
            return "".join(b.text for b in resp.content if b.type == "text")
        if resp.stop_reason == "pause_turn":
            messages.append({"role": "assistant", "content": resp.content})
            continue
        if resp.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": resp.content})
            tool_results = [
                {"type": "tool_result", "tool_use_id": b.id,
                 "content": run_tool(b.name, b.input)}
                for b in resp.content if b.type == "tool_use"
            ]
            messages.append({"role": "user", "content": tool_results})
            continue
        raise RuntimeError(f"unhandled stop_reason: {resp.stop_reason}")
```

Notes: the assistant turn is appended **verbatim** (the `tool_use` blocks must survive into history). Tool results are returned in a single `user` message whose `content` is a **list** of `tool_result` blocks, one per `tool_use_id`. `pause_turn` requires re-sending the assistant content unchanged; do not synthesize a tool result.

### TypeScript — Claude Agent SDK

For Anthropic's higher-level [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/agent-loop) (`@anthropic-ai/claude-agent-sdk`), the loop is already implemented for you. You consume an async stream of typed messages and check the terminal `ResultMessage`.

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const stream = query({
  prompt: "Find the failing tests in auth.ts and fix them.",
  options: {
    model: "claude-opus-4-7",
    maxTurns: 20,
    maxBudgetUsd: 1.0,
    permissionMode: "acceptEdits",
    allowedTools: ["Read", "Edit", "Bash", "Grep", "Glob"],
  },
});

for await (const message of stream) {
  if (message.type === "assistant") {
    console.log(`turn: ${message.message.content.length} blocks`);
  }
  if (message.type === "result") {
    if (message.subtype === "success") {
      console.log("done:", message.result);
    } else {
      console.error("stopped early:", message.subtype);
    }
  }
}
```

The SDK runs the same `stop_reason`-driven loop internally: Claude evaluates, requests tools, the SDK executes them, results feed back automatically, and one full Claude turn + tool execution is what the SDK calls a *turn*. The loop ends when Claude produces an assistant message with no `tool_use` blocks. `maxTurns` and `maxBudgetUsd` are **guardrails**, not the primary stop mechanism — they produce a `ResultMessage` with subtype `error_max_turns` or `error_max_budget_usd` when tripped.

## Anti-patterns to avoid

- **Parsing natural-language signals to terminate the loop.** Looking for "Final answer:" or "DONE" in `response.content` is brittle — the model can phrase completion in infinite ways and may still want to call another tool. **Correct:** branch only on `response.stop_reason`.
- **Using an iteration cap as the primary stopping mechanism.** Hardcoding `for _ in range(10):` and exiting on the cap means you'll terminate mid-task on hard problems and waste tokens on easy ones. **Correct:** let `stop_reason == "end_turn"` end the loop; keep iteration caps and `max_budget_usd` as safety guardrails only.
- **Checking assistant text content to decide completion.** A turn can contain *both* `text` and `tool_use` blocks at once (Claude can narrate while requesting a tool). Treating "has text" as "done" drops tool calls. **Correct:** inspect `stop_reason`; iterate over content blocks by `type`.
- **Dropping the assistant turn when you append tool results.** Sending tool results without first appending the assistant `tool_use` turn produces an invalid `messages` array and an API error. **Correct:** append the assistant turn verbatim, then append a single user turn of `tool_result` blocks.
- **Adding extra text after `tool_result` blocks.** Trailing `text` blocks in the same user turn teach Claude to expect user text after every tool call, causing empty `end_turn` responses. **Correct:** the user turn after a `tool_use` should contain only `tool_result` blocks.
- **Ignoring `pause_turn`.** With server tools the server hits its own 10-iteration cap and returns `pause_turn` with no `tool_result` for you to produce. Treating this like `end_turn` truncates the agent. **Correct:** append the assistant response unchanged and call again.
- **Ignoring `max_tokens` truncation inside a `tool_use` block.** If `stop_reason == "max_tokens"` and the last block is `tool_use`, the JSON input is incomplete and retrying with the same limit fails again. **Correct:** detect the case and retry with a higher `max_tokens`.
- **Hardcoding the tool sequence.** Calling `read_file → search → write_file` from your own code with no model-in-the-loop reasoning is a **workflow**, not an agent. Fine when the path is known — but don't expect it to recover from novel inputs.

## Exam-style focus points

- Given a `stop_reason` value, identify the correct loop action (continue with tool results, append-and-resend for `pause_turn`, exit on `end_turn`, retry-with-larger-budget on `max_tokens` mid-tool, surface refusal).
- Identify which `messages` mutations are required between iterations: append assistant turn verbatim (including `tool_use` blocks), then append a user turn of `tool_result` blocks keyed by `tool_use_id`.
- Distinguish model-driven decision-making from pre-configured decision trees, and pick the right pattern for a described task (open-ended task = agent; well-defined fixed path = workflow).
- Spot the anti-patterns in a code sample: text-parsing for completion, iteration-cap-as-stop, missing assistant turn, extra `text` blocks after `tool_result`, ignoring `pause_turn`.
- Know that `max_turns` / `max_budget_usd` in the Claude Agent SDK are guardrails — the primary loop terminator is still "no `tool_use` blocks in the assistant response."

## References

- [Handling stop reasons — Claude API docs](https://docs.anthropic.com/en/docs/build-with-claude/handling-stop-reasons) — authoritative list of every `stop_reason` value with handling examples in Python, TypeScript, Go, Java, C#, PHP, and Ruby.
- [Messages API reference — Create a Message](https://docs.anthropic.com/en/api/messages) — request/response schema including the `text`, `tool_use`, and `tool_result` content block types.
- [Tool use overview](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview) — defining tools and the `tool_use` / `tool_result` exchange.
- [Building effective agents — Anthropic Engineering (Dec 19, 2024)](https://www.anthropic.com/engineering/building-effective-agents) — canonical post defining workflows vs. agents and the augmented-LLM building block. Required reading for the exam.
- [Effective context engineering for AI agents — Anthropic Engineering (Sep 29, 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — restates the working definition of an agent as "LLMs autonomously using tools in a loop"; covers compaction and subagent patterns for long-horizon loops.
- [How the agent loop works — Claude Agent SDK docs](https://code.claude.com/docs/en/agent-sdk/agent-loop) — official description of the SDK turn/message lifecycle, `max_turns`, `max_budget_usd`, and `ResultMessage` subtypes.
- [Agent SDK reference — Python](https://code.claude.com/docs/en/sdk/sdk-python) and [TypeScript](https://code.claude.com/docs/en/sdk/sdk-typescript) — `query()` vs `ClaudeSDKClient`, message types, and the typed event stream.
- [anthropics/claude-cookbooks — tool_use](https://github.com/anthropics/claude-cookbooks/tree/main/tool_use) — runnable examples of the loop, `tool_choice`, and programmatic tool calling.
- [Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python) and [TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript) — current client shapes used in the loops above.
