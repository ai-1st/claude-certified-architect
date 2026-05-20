---
title: "Раздел 1 — Обзор экзамена, структура и сценарии"
linkTitle: "1. Обзор экзамена"
weight: 1
description: "Зачем существует CCA-F, как он оценивается, шесть сценариев и учебный маршрут, сопоставленный с опубликованными весами доменов."
---

## Что покрывает этот раздел

Базовое понимание того, *зачем* существует Claude Certified Architect – Foundations (CCA-F), *что* он проверяет, *как* он устроен (домены, оценивание, сценарии) и *как* спланировать подготовку, которая аккуратно соответствует опубликованным весам доменов.

## Исходный материал (из официального guide)

**Назначение.** CCA-F подтверждает, что практики умеют принимать обоснованные компромиссные решения при реализации реальных решений с Claude. Экзамен покрывает четыре ключевые технологии: Claude Code, Claude Agent SDK, Claude API и Model Context Protocol (MCP).

**Целевой кандидат.** Solution architect примерно с 6+ месяцами hands-on опыта с Claude, который лично:

- Создавал agentic apps с Claude Agent SDK (orchestration, subagents, tool integration, lifecycle hooks).
- Настраивал Claude Code для команд через `CLAUDE.md`, Agent Skills, MCP servers и plan mode.
- Проектировал интерфейсы MCP tools и resources поверх реальных backend-систем.
- Прорабатывал prompts, которые дают надежный structured output (JSON schemas, few-shot, extraction patterns).
- Управлял context windows в длинных документах и multi-agent handoffs.
- Интегрировал Claude в CI/CD для code review, test generation и PR feedback.
- Принимал решения по escalation и reliability, включая human-in-the-loop и self-evaluation.

**Формат вопросов.** Multiple choice: один правильный ответ плюс три дистрактора. Штрафа за угадывание нет; unanswered questions считаются неправильными.

**Оценивание.** Pass/fail со scaled score от 100 до 1,000. Минимальный проходной балл — **720**. Scaled scoring выравнивает результаты между вариантами экзамена с немного разной сложностью.

**Домены и веса.**

| # | Домен | Вес |
|---|---|---|
| 1 | Agentic Architecture & Orchestration | 27% |
| 2 | Tool Design & MCP Integration | 18% |
| 3 | Claude Code Configuration & Workflows | 20% |
| 4 | Prompt Engineering & Structured Output | 20% |
| 5 | Context Management & Reliability | 15% |

**Сценарии.** Каждый экзамен случайно выбирает **4 из 6** production-сценариев. Полный набор:

1. **Customer Support Resolution Agent** — Agent SDK agent поверх MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`); цель 80%+ first-contact resolution при корректной escalation.
2. **Code Generation with Claude Code** — `CLAUDE.md`, custom slash commands, plan mode vs. direct execution.
3. **Multi-Agent Research System** — Coordinator делегирует web-search, document-analysis, synthesis и report-generation subagents.
4. **Developer Productivity with Claude** — Agent SDK поверх built-in tools (`Read`, `Write`, `Bash`, `Grep`, `Glob`) плюс MCP servers для codebase exploration и boilerplate generation.
5. **Claude Code for Continuous Integration** — интегрированные в CI/CD code review, test generation и PR feedback; минимизация false positives.
6. **Structured Data Extraction** — extraction из unstructured documents с validation через JSON schema и graceful edge-case handling.

**Вне рамок.** Fine-tuning, billing/auth/quotas, vision и computer use, streaming API internals, embedding/vector DBs, RLHF/Constitutional AI, prompt-caching internals, tokenization, конкретные cloud-provider configs и похожие deployment plumbing topics.

## Расширенный контекст

### Экосистема сертификации

Anthropic публично запустила CCA-F **12 марта 2026 года** как свою первую официальную техническую сертификацию. Формат, о котором сообщают несколько вторичных источников: **60 multiple-choice questions, 120 minutes, proctored, closed-book**, результаты в течение примерно 2 рабочих дней с разбивкой по разделам. Стоимость — **$99 за попытку**, waived для первых 5,000 попыток, выделенных Claude Partner Network.

На сегодня доступ **опосредован организацией** через Claude Partner Network, а не полностью self-serve: кандидаты либо работают в partner org, либо согласуются с одной из них, затем запрашивают слот экзамена на `anthropic.skilljar.com`. Поддерживающий curriculum — Anthropic Academy — бесплатен и размещен на Skilljar; он включает "Claude 101," "Building with the Claude API," вводный и advanced курсы MCP, "Claude Code in Action," "Introduction to agent skills," "Introduction to subagents" и дополнительные cloud-deployment courses.

Anthropic дала понять, что **дополнительные tiers для sellers, developers и advanced architects** появятся позже в 2026 году. Рассматривайте CCA-F как entry level многоуровневого credential stack, а не как разовый badge.

### Стратегия по весам доменов

Пять доменов довольно близки по весу (15%–27%), поэтому ни один домен нельзя пропустить, но Domain 1 (Agentic Architecture & Orchestration) — самый большой рычаг. Прагматичный бюджет подготовки по весам:

- **~27% учебного времени на Domain 1** — agent loops, обработка `stop_reason`, subagent delegation, lifecycle hooks. Этот домен также появляется внутри Scenarios 1, 3 и 4, так что его практический эффект выше сырого веса.
- **~20% на каждый из Domains 3 и 4** — configuration Claude Code (`CLAUDE.md` hierarchy, plan mode, slash commands, skills) и prompt/structured-output engineering (JSON schemas, few-shot, retry loops).
- **~18% на Domain 2** — описания MCP tools, различающие похожие tools, structured error responses с флагами `retryable` и resource design.
- **~15% на Domain 5** — context-window management, scratchpads, escalation rules, confidence-based human-in-the-loop routing.

Полезная эвристика: примерно **две трети экзамена** завязаны на agent-loop design, Claude Code configuration и решения prompt/structured-output. Потратьте время там до тонкой настройки long tail.

### Шесть сценариев — чего ожидать

- **Scenario 1 — Customer Support Resolution Agent.** Ждите вопросы о том, *когда* agent escalates (policy gap, явная просьба клиента, невозможность продвинуться) вместо autonomous resolution, и о том, как `process_refund` должен показывать retryable/non-retryable errors, чтобы agent мог восстановиться без loop.
- **Scenario 2 — Code Generation with Claude Code.** Ждите tradeoffs plan-mode-vs-direct-execution, где живут path-specific rules (`.claude/rules/`) и как scoping работает для custom slash commands и skills (`context: fork`, `allowed-tools`).
- **Scenario 3 — Multi-Agent Research System.** Ждите coordinator/subagent boundaries, как передавать cited context между agents без переполнения window, и где enforce evaluator/critic loops для точности citations.
- **Scenario 4 — Developer Productivity with Claude.** Ждите tool-selection questions по built-ins (`Read`, `Write`, `Bash`, `Grep`, `Glob`) плюс MCP add-ons, а также judgments о том, когда держать работу в parent agent, а когда fork subagent.
- **Scenario 5 — Claude Code for CI.** Ждите prompt-engineering questions для *actionable* review comments, explicit review criteria для подавления false positives и multi-pass review patterns для больших diffs.
- **Scenario 6 — Structured Data Extraction.** Ждите schema design с nullable/optional fields, validation-retry loops на `tool_use`, batch processing tradeoffs и graceful degradation, когда документ частично malformed.

### Сопоставимые сертификации

| Сертификация | Vendor | Фокус | Notes |
|---|---|---|---|
| Claude Certified Architect – Foundations | Anthropic | Production Claude architecture: agents, MCP, Claude Code, structured output | Scenario-heavy, judgment-driven; partner-mediated access; $99 |
| AWS Certified AI Practitioner (AIF-C01) | AWS | Foundational AI/ML literacy + Bedrock concepts | 85 questions / 120 min; passing 700/1000; broader and more conceptual, less hands-on |
| Google Cloud Generative AI Leader | Google | Business strategy for GenAI on Vertex AI / Gemini | Leadership-oriented, not implementation-focused |

Самое ясное отличие: AWS AI Practitioner и GCP Generative AI Leader проверяют **широту и literacy** по managed services; CCA-F проверяет **глубину и judgment** внутри agentic stack одного vendor. Multi-cloud architects часто собирают их в stack, а не выбирают один.

## Глоссарий ключевых терминов

- **Agent SDK** — библиотека Anthropic для построения agentic loops с tool calling, subagents и lifecycle hooks.
- **Claude Code** — terminal-native coding agent Anthropic, настраиваемый через `CLAUDE.md`, slash commands, Agent Skills и MCP servers.
- **MCP (Model Context Protocol)** — открытый протокол для предоставления tools, resources и prompts из backend-систем в Claude.
- **Plan mode** — режим Claude Code, который создает plan до любых side-effecting actions.
- **`CLAUDE.md`** — иерархический project/user-level configuration file, автоматически потребляемый Claude Code.
- **Scaled score** — нормализованный score 100–1,000, выравнивающий разные варианты экзамена; cut score — 720.
- **Scenario** — реалистичный production context, который framing batch связанных вопросов; 4 из 6 появляются на экзамене.
- **Distractor** — правдоподобно выглядящий неправильный ответ, ловящий кандидатов с неполным опытом.

## Рекомендованный путь подготовки

1. **Просмотрите 12 учебных разделов по порядку**, уделяя особое внимание task statements в начале каждого раздела — это per-domain learning objectives прямо из официального exam guide.
2. **Пройдите бесплатные курсы Anthropic Academy на Skilljar** в таком порядке: Claude 101 → Building with the Claude API → Introduction to MCP → MCP Advanced Topics → Claude Code in Action → Introduction to agent skills → Introduction to subagents.
3. **Соберите один Agent SDK project с нуля**, который использует полный agentic loop: tool calling, обработка `stop_reason`, subagent delegation, lifecycle hooks и error/retry policy.
4. **Настройте Claude Code для реального repo**: layered `CLAUDE.md`, path-specific rules в `.claude/rules/`, хотя бы один custom slash command, один skill с `context: fork` и `allowed-tools`, и одну MCP server integration.
5. **Спроектируйте и stress-test MCP tools**: напишите tool descriptions, различающие near-duplicates, возвращайте structured errors с флагами `retryable` и проверяйте tool selection на ambiguous prompts.
6. **Ship structured extraction pipeline** с JSON schemas, validation-retry loop, optional/nullable fields и Message Batches API для throughput.
7. **Отработайте шесть сценариев**, написав одностраничный architecture sketch для каждого — primary domains, tool inventory, escalation policy, context strategy и reliability patterns.
8. **Официальный practice exam проходите последним**, разберите каждое объяснение (включая правильные ответы) и бронируйте proctored sitting только когда уверенно набираете выше 720 на simulated material.

## References

- [Anthropic Academy on Skilljar](https://anthropic.skilljar.com/) — бесплатные подготовительные курсы (Claude 101, Building with the Claude API, MCP, Claude Code, agent skills, subagents).
- [Anthropic Learn — Courses index](https://www.anthropic.com/learn/courses) — официальный каталог training courses Anthropic.
- [Claude Certified Architect: How to Get Certified in 2026 (lowcode.agency)](https://www.lowcode.agency/blog/how-to-become-claude-certified-architect) — reported launch date, format, cost и access path.
- [How to become a Claude Certified Architect (datastudios.org)](https://www.datastudios.org/post/how-to-become-a-claude-certified-architect-current-access-path-partner-requirements-preparation) — Partner Network access path, study areas, что официально подтверждено.
- [Claude Certified Architect: who can take the exam (datastudios.org)](https://www.datastudios.org/post/claude-certified-architect-who-can-take-the-exam-and-why-access-is-still-limited) — eligibility и access limitations.
- [Inside Anthropic's Claude Certified Architect Program (dev.to/mcrolly)](https://dev.to/mcrolly/inside-anthropics-claude-certified-architect-program-what-it-tests-and-who-should-pursue-it-1dk6) — breakdown того, что проверяет экзамен, и target audience.
- [How to Pass the CCA Foundations Exam (dev.to/sojs)](https://dev.to/sojs/how-to-pass-the-claude-certified-architect-cca-foundations-exam-3oa9) — candidate-reported 7-week preparation phasing и common pitfalls.
- [Claude Certified Architect Study Guide — 12-week plan (claudecertifications.com)](https://claudecertifications.com/claude-certified-architect/study-guide) — альтернативный более длинный timeline подготовки.
- [Claude Architect vs AWS AI Practitioner (flashgenius.net)](https://flashgenius.net/blog-article/claude-architect-vs-aws-ai-practitioner-2026-which-ai-certification-should-you-choose) — сравнение с AWS AI Practitioner certification.
- [AWS vs Google AI Certifications 2026 (pertamapartners.com)](https://www.pertamapartners.com/insights/aws-google-ai-certifications) — comparison context для GCP Generative AI Leader credential.
