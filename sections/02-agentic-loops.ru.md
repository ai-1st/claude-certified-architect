---
title: "Раздел 2 — Agentic Loops и обработка stop_reason"
linkTitle: "2. Agentic Loops"
weight: 2
description: "Domain 1.1 — control flow agentic loop, значения stop_reason, которые нужно знать, и канонические анти-паттерны."
---

## Что покрывает этот раздел

Как строить центральный control flow Claude agent: отправить request, проверить `stop_reason`, выполнить tools, которые запросил Claude, добавить results в history и повторить. Каждый higher-level pattern (orchestrator-workers, subagents, evaluator-optimizer, Agent SDK) строится поверх этого loop.

## Исходный материал (из официального guide)

### Требуемые знания

- Lifecycle agentic loop: отправить request в Claude, проверить `stop_reason` (`"tool_use"` vs `"end_turn"`), выполнить requested tools, вернуть results для следующей iteration.
- Как tool results добавляются в conversation history, чтобы model могла рассуждать о следующем action.
- Различие между **model-driven decision-making** (Claude рассуждает, какой tool вызвать дальше на основе context) и **pre-configured decision trees** (developer hardcodes tool sequence).

### Требуемые навыки

- Реализовать control flow agentic loop, который продолжается, пока `stop_reason == "tool_use"`, и завершается, когда `stop_reason == "end_turn"`.
- Добавлять tool results в conversation context между iterations, чтобы model могла включать новую информацию в reasoning.
- Избегать анти-паттернов: parsing natural language signals для завершения loop, использование arbitrary iteration caps как основного stop mechanism или проверка assistant text content как completion indicator.

## Agentic loop от начала до конца

Рабочее определение agent у Anthropic — самое простое в отрасли: *"LLMs autonomously using tools in a loop."* Augmented LLM (model + tools + retrieval + memory) — базовый building block; каждый workflow pattern (prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer) составлен из него.

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

Один iteration выглядит так:

1. Отправьте `messages` плюс `tools` schema в `POST /v1/messages`.
2. Claude возвращает `assistant` message. Его `content` — list of blocks: zero or more `text` blocks и zero or more `tool_use` blocks. Top-level `stop_reason` суммирует, почему generation остановилась.
3. Если `stop_reason == "tool_use"`: добавьте assistant turn verbatim, выполните каждый requested tool, добавьте один новый `user` turn, где content — list of `tool_result` blocks (по одному на `tool_use_id`), и снова вызовите API с updated history.
4. Если `stop_reason == "end_turn"`: model решила, что task finished. Return.

Tool results **добавляются в conversation history**, а не summary-away. Каждый новый request несет всю history, поэтому Claude может chain reasoning across many turns. Model — не ваш code — решает, какой tool вызвать дальше, исходя из observed context. Это различие между **model-driven decision-making** (Claude выбирает tool N+1 из running context) и **pre-configured decision trees** (ваш code статически вызывает `tool_a()` → `tool_b()` → `tool_c()`). Decision trees — это workflows; agentic loops — agents. Опубликованная guidance Anthropic рекомендует предпочитать более простой workflow, когда path можно hardcode.

## Значения stop_reason, которые нужно знать

`stop_reason` входит в каждый successful Messages API response. Это единственный signal, по которому следует branch, чтобы решить, продолжать ли loop. Полный набор documented values ниже.

| Значение | Смысл | Что должен делать loop |
| --- | --- | --- |
| `end_turn` | Claude естественно завершил response. | Выйти из loop. Вернуть `response.content` text blocks caller. |
| `tool_use` | Response содержит один или несколько `tool_use` blocks; Claude ожидает, что вы их выполните. | Добавить assistant turn, выполнить каждый `tool_use` block, добавить `user` turn с соответствующими `tool_result` blocks (используйте тот же `tool_use_id`) и снова вызвать API. |
| `max_tokens` | Output достиг параметра `max_tokens`. Response truncated и может содержать **incomplete** `tool_use` block. | Обнаружьте mid-tool-call truncation, проверив, что last content block's `type == "tool_use"`; retry с большим `max_tokens`. Иначе запросите continuation или покажите truncation warning. |
| `stop_sequence` | Output совпал с custom string в `stop_sequences`. Matched sequence находится в `response.stop_sequence`. | Считать successful terminal stop для этого pattern. Продолжить или finalize в зависимости от protocol. |
| `pause_turn` | Server-side sampling loop достиг своего iteration cap при работе с **server tools** (web search, web fetch, code execution и т. д.). Response может содержать `server_tool_use` block без matching `server_tool_result`. | Добавить assistant response **unchanged** и снова вызвать API с теми же tools. Повторять до non-`pause_turn` stop reason. |
| `refusal` | Model отказалась по safety reasons (Sonnet 4.5+ / Opus 4.1+ API safety filter). | Не loop. Показать refusal caller; optionally rephrase, route to a different model (например, Haiku 4.5), or escalate. |
| `model_context_window_exceeded` | Generation остановилась, потому что response достиг полного context window model (не `max_tokens`). Sonnet 4.5+ by default; earlier models need a beta header. | Обрабатывать похоже на `max_tokens` — response valid but capped. Continue, summarize или compact context. |

Branching по `stop_reason` — **единственный** корректный termination test. Не parse text вроде "I'm done" или "Final answer:" — это канонический анти-паттерн ниже.

## Reference implementations

### Python — raw Messages API loop

Минимальная runnable shape на `anthropic` Python SDK (тот же loop pattern, который Anthropic показывает в [docs](https://docs.anthropic.com/en/docs/build-with-claude/handling-stop-reasons#handling-tool-use-workflows)).

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

Notes: assistant turn добавляется **verbatim** (`tool_use` blocks должны сохраниться в history). Tool results возвращаются в одном `user` message, где `content` — **list** of `tool_result` blocks, по одному на `tool_use_id`. `pause_turn` требует re-sending assistant content unchanged; не synthesize tool result.

### TypeScript — Claude Agent SDK

Для higher-level [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/agent-loop) Anthropic (`@anthropic-ai/claude-agent-sdk`) loop уже реализован. Вы потребляете async stream typed messages и проверяете terminal `ResultMessage`.

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

SDK internally запускает тот же `stop_reason`-driven loop: Claude evaluates, requests tools, SDK executes them, results feed back automatically, и один полный Claude turn + tool execution — это то, что SDK называет *turn*. Loop заканчивается, когда Claude produces assistant message без `tool_use` blocks. `maxTurns` и `maxBudgetUsd` — **guardrails**, а не primary stop mechanism; при срабатывании они дают `ResultMessage` с subtype `error_max_turns` или `error_max_budget_usd`.

## Анти-паттерны, которых нужно избегать

- **Parsing natural-language signals to terminate the loop.** Поиск "Final answer:" или "DONE" в `response.content` brittle — model может сформулировать completion бесконечным числом способов и все еще хотеть вызвать another tool. **Правильно:** branch only on `response.stop_reason`.
- **Using an iteration cap as the primary stopping mechanism.** Hardcoding `for _ in range(10):` и выход по cap означает, что вы terminate mid-task на сложных задачах и тратите tokens на легких. **Правильно:** пусть `stop_reason == "end_turn"` завершает loop; держите iteration caps и `max_budget_usd` только как safety guardrails.
- **Checking assistant text content to decide completion.** Turn может содержать *both* `text` and `tool_use` blocks at once (Claude может narrate while requesting a tool). Treating "has text" as "done" drops tool calls. **Правильно:** inspect `stop_reason`; iterate over content blocks by `type`.
- **Dropping the assistant turn when you append tool results.** Sending tool results without first appending assistant `tool_use` turn produces invalid `messages` array and API error. **Правильно:** append assistant turn verbatim, then append a single user turn of `tool_result` blocks.
- **Adding extra text after `tool_result` blocks.** Trailing `text` blocks in the same user turn teach Claude to expect user text after every tool call, causing empty `end_turn` responses. **Правильно:** user turn после `tool_use` должен содержать только `tool_result` blocks.
- **Ignoring `pause_turn`.** С server tools server hits its own 10-iteration cap and returns `pause_turn` with no `tool_result` for you to produce. Treating this like `end_turn` truncates agent. **Правильно:** append assistant response unchanged and call again.
- **Ignoring `max_tokens` truncation inside a `tool_use` block.** If `stop_reason == "max_tokens"` and the last block is `tool_use`, JSON input incomplete and retrying with the same limit fails again. **Правильно:** detect the case and retry with a higher `max_tokens`.
- **Hardcoding the tool sequence.** Calling `read_file → search → write_file` from your own code with no model-in-the-loop reasoning is a **workflow**, not an agent. Fine when the path is known — but don't expect it to recover from novel inputs.

## Exam-style focus points

- По заданному `stop_reason` определить correct loop action (continue with tool results, append-and-resend for `pause_turn`, exit on `end_turn`, retry-with-larger-budget on `max_tokens` mid-tool, surface refusal).
- Определить, какие mutations `messages` нужны между iterations: append assistant turn verbatim (including `tool_use` blocks), then append user turn of `tool_result` blocks keyed by `tool_use_id`.
- Отличать model-driven decision-making от pre-configured decision trees и выбирать правильный pattern для описанной task (open-ended task = agent; well-defined fixed path = workflow).
- Находить anti-patterns в code sample: text-parsing for completion, iteration-cap-as-stop, missing assistant turn, extra `text` blocks after `tool_result`, ignoring `pause_turn`.
- Знать, что `max_turns` / `max_budget_usd` в Claude Agent SDK — guardrails; primary loop terminator все равно "no `tool_use` blocks in the assistant response."

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
