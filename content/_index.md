---
title: "Claude Certified Architect — Foundations Prep"
toc: false
---

{{< hextra/hero-badge >}}
  <div class="hx:w-2 hx:h-2 hx:rounded-full hx:bg-primary-400"></div>
  Unofficial · MIT licensed · open source
{{< /hextra/hero-badge >}}

{{< hextra/hero-headline >}}
  Practice for the CCA-F exam, with Claude as your proctor.
{{< /hextra/hero-headline >}}

{{< hextra/hero-subtitle >}}
  Open the repo in Claude Code. Say *"start a mock exam."* &nbsp;
  Claude reads one study section at a time and quizzes you in the same
  scenario-based, four-option format used on the real Claude Certified
  Architect — Foundations exam.
{{< /hextra/hero-subtitle >}}

{{< hextra/hero-button text="Start with the 12 study sections" link="docs" >}}
&nbsp;
{{< hextra/hero-button text="View on GitHub" link="https://github.com/ai-1st/claude-certified-architect" style="background: linear-gradient(90deg, #555 0%, #333 100%);" >}}

<div class="hx:mt-6"></div>

## What this repo is

A study companion for the **Claude Certified Architect — Foundations (CCA-F)** exam — Anthropic's first official technical certification, launched in March 2026. The exam tests practical judgement on building production-grade Claude applications: agent loops, MCP tools, Claude Code configuration, prompt and structured-output engineering, context management.

The repo contains two things:

1. A `CLAUDE.md` file that puts Claude into **Mock Exam Mode** — a section-by-section quiz proctor that generates questions in the official format, waits for your answer, then explains why each option is right or wrong.
2. **Twelve study sections** (`sections/01..12.md`) covering every domain on the exam. The same files power both Claude's mock exam and this website.

## How Mock Exam Mode works

{{< cards >}}
  {{< card link="docs/01-exam-overview" title="1. Pick a section" subtitle="Claude greets you and asks which of the 12 sections (or which of the 6 official scenarios) you want to drill." >}}
  {{< card link="docs/02-agentic-loops" title="2. One file at a time" subtitle="Claude reads only the chosen section file. Context stays lean; questions stay grounded." >}}
  {{< card link="docs" title="3. Real-format questions" subtitle="Scenario lead-in → stem → exactly four options. One correct answer. Three plausible distractors drawn from real anti-patterns." >}}
  {{< card link="docs" title="4. Answer, then explanation" subtitle="Claude waits for your letter, then explains why the correct option is correct and why each distractor is wrong, citing the section's concept by name." >}}
{{< /cards >}}

## The exam at a glance

| | |
| --- | --- |
| Format | 60 multiple-choice questions, one correct + three distractors |
| Time | 120 minutes |
| Scoring | Scaled 100–1000; pass = **720** |
| Scenarios | **4 of 6** drawn at random per sitting |
| Domains | D1 Agentic Architecture & Orchestration **27%** · D2 Tool Design & MCP **18%** · D3 Claude Code Config **20%** · D4 Prompt Engineering & Structured Output **20%** · D5 Context Management & Reliability **15%** |

## Quick start

```bash
git clone https://github.com/ai-1st/claude-certified-architect.git
cd claude-certified-architect
claude            # or open the folder in Cursor / Claude Code
# then say:  Start a mock exam.
```

Claude picks up `CLAUDE.md` automatically and walks you through Mock Exam Mode.

## Disclaimer

Unofficial. Not affiliated with or endorsed by Anthropic. The exam, its content domains, scenarios, and scoring are Anthropic's. The study sections in this repo are an independent re-statement of publicly-available material, organized for context-efficient practice with Claude.
