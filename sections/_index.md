---
title: "Study Sections"
linkTitle: "Study sections"
weight: 1
cascade:
  type: docs
sidebar:
  open: true
---

The CCA-F study material is broken into **12 sections** so each one comfortably fits in a Claude Code conversation. When you start Mock Exam Mode, Claude reads exactly one of these per round — that's how the questions stay specific, the distractors stay plausible, and the context budget stays sane.

Pick a section to skim, or jump into [Section 1 — Exam Overview]({{< ref "01-exam-overview" >}}) for orientation. To practice with Claude as your proctor, see the [home page](/) for the kickoff prompt.

{{< cards >}}
  {{< card link="01-exam-overview" title="1. Exam Overview" subtitle="Why CCA-F exists, scoring, the 6 scenarios, and a study path mapped to domain weights." >}}
  {{< card link="02-agentic-loops" title="2. Agentic Loops" subtitle="Domain 1.1 — agentic loop control flow, stop_reason values, anti-patterns." >}}
  {{< card link="03-multi-agent-orchestration" title="3. Multi-Agent Orchestration" subtitle="Domains 1.2–1.4 — coordinator/subagent boundaries, Task tool, context passing." >}}
  {{< card link="04-hooks-decomposition-sessions" title="4. Hooks, Decomposition & Sessions" subtitle="Domains 1.5–1.7 — PostToolUse hooks, prompt chaining vs adaptive decomposition, fork_session." >}}
  {{< card link="05-tool-design" title="5. Tool Design & Built-ins" subtitle="Domains 2.1, 2.3, 2.5 — tool descriptions, tool_choice, Read/Write/Edit/Bash/Grep/Glob." >}}
  {{< card link="06-mcp-integration" title="6. MCP Integration" subtitle="Domains 2.2, 2.4 — structured error envelopes, .mcp.json scopes, resources vs tools." >}}
  {{< card link="07-claude-md-and-rules" title="7. CLAUDE.md & Rules" subtitle="Domains 3.1, 3.3 — CLAUDE.md hierarchy, @imports, .claude/rules/ glob patterns." >}}
  {{< card link="08-commands-skills-plan-mode" title="8. Commands, Skills & Plan Mode" subtitle="Domains 3.2, 3.4, 3.5 — slash commands, skills with context: fork, plan mode." >}}
  {{< card link="09-cicd-integration" title="9. CI/CD Integration" subtitle="Domain 3.6 — claude -p, --output-format json, prompt design for CI." >}}
  {{< card link="10-prompt-engineering" title="10. Prompt Engineering" subtitle="Domains 4.1, 4.2, 4.4 — explicit criteria, few-shot, multi-pass review." >}}
  {{< card link="11-structured-output-and-batch" title="11. Structured Output & Batch" subtitle="Domains 4.3, 4.5, 4.6 — JSON Schema, validation-retry loops, Message Batches API." >}}
  {{< card link="12-context-and-reliability" title="12. Context & Reliability" subtitle="Domain 5 — case-facts, escalation triggers, structured errors, claim–source mapping." >}}
{{< /cards >}}

## How to use these on the exam

1. **Skim section 1 first** — it sets up the exam structure, the 6 scenarios, and the domain weights. Everything else hangs off it.
2. **Drill the heavy domains first.** D1 (Agentic Architecture) is 27% of the exam and shows up across three scenarios — sections 2, 3, 4 are your highest-leverage hours.
3. **Then D3 + D4** (20% each) — sections 7, 8, 9 (Claude Code) and sections 10, 11 (prompt engineering / structured output).
4. **Finish with D2 and D5** — sections 5, 6 (tools / MCP) and 12 (context, escalation, provenance — small domain but cross-cutting).
5. **Use Mock Exam Mode in Claude Code** to drill each section with real-format questions until you can predict the distractor pattern before reading the options.
