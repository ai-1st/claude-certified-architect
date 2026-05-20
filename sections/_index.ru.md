---
title: "Учебные разделы"
linkTitle: "Учебные разделы"
weight: 1
cascade:
  type: docs
sidebar:
  open: true
---

Учебный материал CCA-F разбит на **12 разделов**, чтобы каждый из них комфортно помещался в разговор Claude Code. Когда вы запускаете Mock Exam Mode, Claude читает ровно один раздел за раунд — так вопросы остаются конкретными, дистракторы правдоподобными, а бюджет контекста управляемым.

Выберите раздел для быстрого просмотра или перейдите к [Разделу 1 — Обзор экзамена]({{< ref "01-exam-overview" >}}), чтобы сориентироваться. Чтобы практиковаться с Claude как проктором, см. стартовую фразу на [главной странице](/).

{{< cards >}}
  {{< card link="01-exam-overview" title="1. Обзор экзамена" subtitle="Зачем существует CCA-F, оценивание, 6 сценариев и учебный маршрут, сопоставленный с весами доменов." >}}
  {{< card link="02-agentic-loops" title="2. Agentic Loops" subtitle="Domain 1.1 — управление agentic loop, значения stop_reason, анти-паттерны." >}}
  {{< card link="03-multi-agent-orchestration" title="3. Multi-Agent Orchestration" subtitle="Domains 1.2–1.4 — границы coordinator/subagent, Task tool, передача контекста." >}}
  {{< card link="04-hooks-decomposition-sessions" title="4. Hooks, Decomposition & Sessions" subtitle="Domains 1.5–1.7 — PostToolUse hooks, prompt chaining vs adaptive decomposition, fork_session." >}}
  {{< card link="05-tool-design" title="5. Tool Design & Built-ins" subtitle="Domains 2.1, 2.3, 2.5 — описания tools, tool_choice, Read/Write/Edit/Bash/Grep/Glob." >}}
  {{< card link="06-mcp-integration" title="6. MCP Integration" subtitle="Domains 2.2, 2.4 — structured error envelopes, scopes в .mcp.json, resources vs tools." >}}
  {{< card link="07-claude-md-and-rules" title="7. CLAUDE.md & Rules" subtitle="Domains 3.1, 3.3 — иерархия CLAUDE.md, @imports, glob patterns в .claude/rules/." >}}
  {{< card link="08-commands-skills-plan-mode" title="8. Commands, Skills & Plan Mode" subtitle="Domains 3.2, 3.4, 3.5 — slash commands, skills с context: fork, plan mode." >}}
  {{< card link="09-cicd-integration" title="9. CI/CD Integration" subtitle="Domain 3.6 — claude -p, --output-format json, prompt design для CI." >}}
  {{< card link="10-prompt-engineering" title="10. Prompt Engineering" subtitle="Domains 4.1, 4.2, 4.4 — explicit criteria, few-shot, multi-pass review." >}}
  {{< card link="11-structured-output-and-batch" title="11. Structured Output & Batch" subtitle="Domains 4.3, 4.5, 4.6 — JSON Schema, validation-retry loops, Message Batches API." >}}
  {{< card link="12-context-and-reliability" title="12. Context & Reliability" subtitle="Domain 5 — case-facts, escalation triggers, structured errors, claim–source mapping." >}}
{{< /cards >}}

## Как использовать разделы при подготовке

1. **Сначала просмотрите раздел 1** — он задает структуру экзамена, 6 сценариев и веса доменов. Все остальное опирается на него.
2. **Сначала тренируйте самые тяжелые домены.** D1 (Agentic Architecture) — 27% экзамена и встречается в трех сценариях; разделы 2, 3, 4 дают самый высокий эффект за час подготовки.
3. **Затем D3 + D4** (по 20%) — разделы 7, 8, 9 (Claude Code) и разделы 10, 11 (prompt engineering / structured output).
4. **Завершите D2 и D5** — разделы 5, 6 (tools / MCP) и 12 (context, escalation, provenance — небольшой домен, но сквозной).
5. **Используйте Mock Exam Mode в Claude Code**, чтобы отрабатывать каждый раздел вопросами в формате реального экзамена, пока вы не начнете угадывать паттерн дистракторов еще до чтения вариантов.
