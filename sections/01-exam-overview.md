---
title: "Section 1 — Exam Overview, Structure & Scenarios"
linkTitle: "1. Exam Overview"
weight: 1
description: "Why CCA-F exists, how it scores, the six scenarios, and a study path mapped to the published domain weights."
---

## What this section covers

A grounding in *why* the Claude Certified Architect – Foundations (CCA-F) exists, *what* it tests, *how* it is structured (domains, scoring, scenarios), and *how* to plan a study path that maps cleanly to the published domain weights.

## Source material (from official guide)

**Purpose.** The CCA-F validates that practitioners can make informed tradeoff decisions when implementing real-world solutions with Claude. The exam covers four core technologies: Claude Code, the Claude Agent SDK, the Claude API, and the Model Context Protocol (MCP).

**Target candidate.** A solution architect with roughly 6+ months of hands-on Claude experience who has personally:

- Built agentic apps with the Claude Agent SDK (orchestration, subagents, tool integration, lifecycle hooks).
- Configured Claude Code for teams via `CLAUDE.md`, Agent Skills, MCP servers, and plan mode.
- Designed MCP tool and resource interfaces against real backends.
- Engineered prompts that produce reliable structured output (JSON schemas, few-shot, extraction patterns).
- Managed context windows across long documents and multi-agent handoffs.
- Integrated Claude into CI/CD for code review, test generation, and PR feedback.
- Made escalation and reliability decisions including human-in-the-loop and self-evaluation.

**Question format.** Multiple choice, one correct answer plus three distractors. There is no penalty for guessing; unanswered questions are scored as incorrect.

**Scoring.** Pass/fail with a scaled score of 100–1,000. The minimum passing score is **720**. Scaled scoring equates results across exam forms with slightly different difficulty.

**Domains and weightings.**

| # | Domain | Weight |
|---|---|---|
| 1 | Agentic Architecture & Orchestration | 27% |
| 2 | Tool Design & MCP Integration | 18% |
| 3 | Claude Code Configuration & Workflows | 20% |
| 4 | Prompt Engineering & Structured Output | 20% |
| 5 | Context Management & Reliability | 15% |

**Scenarios.** Each exam draws **4 of 6** production scenarios at random. The full set:

1. **Customer Support Resolution Agent** — Agent SDK agent over MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`); target 80%+ first-contact resolution with sound escalation.
2. **Code Generation with Claude Code** — `CLAUDE.md`, custom slash commands, plan mode vs. direct execution.
3. **Multi-Agent Research System** — Coordinator delegating to web-search, document-analysis, synthesis, and report-generation subagents.
4. **Developer Productivity with Claude** — Agent SDK over built-in tools (`Read`, `Write`, `Bash`, `Grep`, `Glob`) plus MCP servers for codebase exploration and boilerplate generation.
5. **Claude Code for Continuous Integration** — CI/CD-integrated code review, test generation, and PR feedback; minimize false positives.
6. **Structured Data Extraction** — JSON-schema-validated extraction from unstructured documents with graceful edge-case handling.

**Out of scope.** Fine-tuning, billing/auth/quotas, vision and computer use, streaming API internals, embedding/vector DBs, RLHF/Constitutional AI, prompt-caching internals, tokenization, specific cloud-provider configs, and similar deployment plumbing topics.

## Enriched context

### Certification ecosystem

Anthropic publicly launched the CCA-F on **March 12, 2026** as its first official technical certification. The format that has been reported by multiple secondary sources is **60 multiple-choice questions, 120 minutes, proctored, closed-book**, with results delivered within ~2 business days including section breakdowns. Cost is **$99 per attempt**, waived for the first 5,000 attempts allocated to the Claude Partner Network.

Access today is **organization-mediated** through the Claude Partner Network rather than fully self-serve: candidates either work at a partner org or align with one, then request an exam slot on `anthropic.skilljar.com`. The supporting curriculum — Anthropic Academy — is free and lives on Skilljar; it includes "Claude 101," "Building with the Claude API," MCP introductory and advanced courses, "Claude Code in Action," "Introduction to agent skills," "Introduction to subagents," and additional cloud-deployment courses.

Anthropic has signaled that **additional tiers for sellers, developers, and advanced architects** are coming later in 2026. Treat the CCA-F as the entry level of a multi-tier credential stack rather than a one-off badge.

### Domain weighting strategy

The five domains are tightly clustered (15%–27%), so no domain can be skipped, but Domain 1 (Agentic Architecture & Orchestration) is the single largest lever. A pragmatic study budget based on weights:

- **~27% of study time on Domain 1** — agent loops, `stop_reason` handling, subagent delegation, lifecycle hooks. This domain also shows up inside Scenarios 1, 3, and 4, so the leverage is higher than the raw weight suggests.
- **~20% each on Domains 3 and 4** — Claude Code configuration (`CLAUDE.md` hierarchy, plan mode, slash commands, skills) and prompt/structured-output engineering (JSON schemas, few-shot, retry loops).
- **~18% on Domain 2** — MCP tool descriptions that disambiguate similar tools, structured error responses with `retryable` flags, and resource design.
- **~15% on Domain 5** — context-window management, scratchpads, escalation rules, confidence-based human-in-the-loop routing.

A useful heuristic: roughly **two-thirds of the exam** turns on agent-loop design, Claude Code configuration, and prompt/structured-output decisions. Spend study time there before tuning the long tail.

### The six scenarios — what to expect

- **Scenario 1 — Customer Support Resolution Agent.** Expect questions about *when* the agent escalates (policy gap, explicit customer request, inability to make progress) versus resolves autonomously, and how `process_refund` should expose retryable/non-retryable errors so the agent can recover without looping.
- **Scenario 2 — Code Generation with Claude Code.** Expect plan-mode-vs-direct-execution tradeoffs, where path-specific rules live (`.claude/rules/`), and how custom slash commands and skills are scoped (`context: fork`, `allowed-tools`).
- **Scenario 3 — Multi-Agent Research System.** Expect coordinator/subagent boundaries, how to pass cited context between agents without blowing the window, and where to enforce evaluator/critic loops to keep citations accurate.
- **Scenario 4 — Developer Productivity with Claude.** Expect tool-selection questions across built-ins (`Read`, `Write`, `Bash`, `Grep`, `Glob`) plus MCP add-ons, and judgments about when to keep work in the parent agent vs. fork a subagent.
- **Scenario 5 — Claude Code for CI.** Expect prompt-engineering questions for *actionable* review comments, explicit review criteria to suppress false positives, and multi-pass review patterns for large diffs.
- **Scenario 6 — Structured Data Extraction.** Expect schema design with nullable/optional fields, validation-retry loops on `tool_use`, batch processing tradeoffs, and graceful degradation when a document is partially malformed.

### Comparable certifications

| Certification | Vendor | Focus | Notes |
|---|---|---|---|
| Claude Certified Architect – Foundations | Anthropic | Production Claude architecture: agents, MCP, Claude Code, structured output | Scenario-heavy, judgment-driven; partner-mediated access; $99 |
| AWS Certified AI Practitioner (AIF-C01) | AWS | Foundational AI/ML literacy + Bedrock concepts | 85 questions / 120 min; passing 700/1000; broader and more conceptual, less hands-on |
| Google Cloud Generative AI Leader | Google | Business strategy for GenAI on Vertex AI / Gemini | Leadership-oriented, not implementation-focused |

The clearest contrast: AWS AI Practitioner and GCP Generative AI Leader test **breadth and literacy** across managed services; CCA-F tests **depth and judgment** inside a single vendor's agentic stack. Multi-cloud architects often stack them rather than choose.

## Key terms glossary

- **Agent SDK** — Anthropic's library for building agentic loops with tool calling, subagents, and lifecycle hooks.
- **Claude Code** — Anthropic's terminal-native coding agent, configured via `CLAUDE.md`, slash commands, Agent Skills, and MCP servers.
- **MCP (Model Context Protocol)** — Open protocol for exposing tools, resources, and prompts from backend systems to Claude.
- **Plan mode** — Claude Code mode that produces a plan before any side-effecting action runs.
- **`CLAUDE.md`** — Hierarchical project/user-level configuration file consumed automatically by Claude Code.
- **Scaled score** — A 100–1,000 normalized score that equates across exam forms; the cut score is 720.
- **Scenario** — A realistic production context that frames a batch of related questions; 4 of 6 appear per exam.
- **Distractor** — A plausible-looking wrong answer that catches candidates with incomplete experience.

## Recommended preparation path

1. **Skim the 12 study sections in order**, paying extra attention to the task statements quoted at the top of each section — they are the per-domain learning objectives drawn straight from the official exam guide.
2. **Complete the free Anthropic Academy courses on Skilljar** in this order: Claude 101 → Building with the Claude API → Introduction to MCP → MCP Advanced Topics → Claude Code in Action → Introduction to agent skills → Introduction to subagents.
3. **Build one Agent SDK project from scratch** that exercises a full agentic loop: tool calling, `stop_reason` handling, subagent delegation, lifecycle hooks, and error/retry policy.
4. **Configure Claude Code for a real repo**: layered `CLAUDE.md`, `.claude/rules/` path-specific rules, at least one custom slash command, one skill with `context: fork` and `allowed-tools`, and one MCP server integration.
5. **Design and stress-test MCP tools**: write tool descriptions that disambiguate near-duplicates, return structured errors with `retryable` flags, and verify tool selection on ambiguous prompts.
6. **Ship a structured extraction pipeline** with JSON schemas, a validation-retry loop, optional/nullable fields, and the Message Batches API for throughput.
7. **Drill on the six scenarios** by writing a one-page architecture sketch for each — primary domains, tool inventory, escalation policy, context strategy, and reliability patterns.
8. **Take the official practice exam last**, review every explanation (including the ones you got right), and book the proctored sitting only once you are scoring comfortably above 720 on simulated material.

## References

- [Claude Certified Architect — Foundations Exam Guide (claudecertifiedarchitect.net)](https://claudecertifiedarchitect.net/) — Community-maintained summary of the official exam guide, scoring, and access path.
- [Anthropic Academy on Skilljar](https://anthropic.skilljar.com/) — Free preparation courses (Claude 101, Building with the Claude API, MCP, Claude Code, agent skills, subagents).
- [Anthropic Learn — Courses index](https://www.anthropic.com/learn/courses) — Anthropic's official catalog of training courses.
- [Claude Certified Architect: How to Get Certified in 2026 (lowcode.agency)](https://www.lowcode.agency/blog/how-to-become-claude-certified-architect) — Reported launch date, format, cost, and access path.
- [How to become a Claude Certified Architect (datastudios.org)](https://www.datastudios.org/post/how-to-become-a-claude-certified-architect-current-access-path-partner-requirements-preparation) — Partner Network access path, study areas, what is officially confirmed.
- [Claude Certified Architect: who can take the exam (datastudios.org)](https://www.datastudios.org/post/claude-certified-architect-who-can-take-the-exam-and-why-access-is-still-limited) — Eligibility and access limitations.
- [Inside Anthropic's Claude Certified Architect Program (dev.to/mcrolly)](https://dev.to/mcrolly/inside-anthropics-claude-certified-architect-program-what-it-tests-and-who-should-pursue-it-1dk6) — Breakdown of what the exam tests and target audience.
- [How to Pass the CCA Foundations Exam (dev.to/sojs)](https://dev.to/sojs/how-to-pass-the-claude-certified-architect-cca-foundations-exam-3oa9) — Candidate-reported 7-week preparation phasing and common pitfalls.
- [Claude Certified Architect Study Guide — 12-week plan (claudecertifications.com)](https://claudecertifications.com/claude-certified-architect/study-guide) — Alternative longer preparation timeline.
- [Claude Architect vs AWS AI Practitioner (flashgenius.net)](https://flashgenius.net/blog-article/claude-architect-vs-aws-ai-practitioner-2026-which-ai-certification-should-you-choose) — Comparison with the AWS AI Practitioner certification.
- [AWS vs Google AI Certifications 2026 (pertamapartners.com)](https://www.pertamapartners.com/insights/aws-google-ai-certifications) — Comparison context for the GCP Generative AI Leader credential.
