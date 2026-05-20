---
title: "Claude Certified Architect — Foundations"
toc: false
---

{{< hextra/hero-badge >}}
  <div class="hx:w-2 hx:h-2 hx:rounded-full hx:bg-primary-400"></div>
  Неофициально · лицензия MIT · open source
{{< /hextra/hero-badge >}}

{{< hextra/hero-headline >}}
  Готовьтесь к экзамену CCA-F с Claude в роли проктора.
{{< /hextra/hero-headline >}}

{{< hextra/hero-subtitle >}}
  Откройте репозиторий в Claude Code. Скажите *"start a mock exam."* &nbsp;
  Claude читает по одному учебному разделу за раз и задает вопросы в том же
  сценарном формате с четырьмя вариантами ответа, что и настоящий экзамен
  Claude Certified Architect — Foundations.
{{< /hextra/hero-subtitle >}}

{{< hextra/hero-button text="Начать с 12 учебных разделов" link="docs" >}}
&nbsp;
{{< hextra/hero-button text="Открыть на GitHub" link="https://github.com/ai-1st/claude-certified-architect" style="background: linear-gradient(90deg, #555 0%, #333 100%);" >}}

<div class="hx:mt-6"></div>

## Что это за репозиторий

Учебный помощник для экзамена **Claude Certified Architect — Foundations (CCA-F)** — первой официальной технической сертификации Anthropic, запущенной в марте 2026 года. Экзамен проверяет практическое инженерное суждение при создании production-grade приложений на Claude: agent loops, MCP tools, настройку Claude Code, prompt engineering и structured-output engineering, управление контекстом.

В репозитории есть две основные части:

1. Файл `CLAUDE.md`, который переводит Claude в **Mock Exam Mode** — режим проктора для посекционного квиза: Claude генерирует вопросы в официальном формате, ждет ваш ответ, а затем объясняет, почему каждый вариант правильный или неправильный.
2. **Двенадцать учебных разделов** (`sections/01..12.md`), покрывающих все домены экзамена. Эти же файлы используются и для mock exam в Claude, и для этого сайта.

## Как работает Mock Exam Mode

{{< cards >}}
  {{< card link="docs/01-exam-overview" title="1. Выберите раздел" subtitle="Claude приветствует вас и спрашивает, какой из 12 разделов или какой из 6 официальных сценариев вы хотите отработать." >}}
  {{< card link="docs/02-agentic-loops" title="2. Один файл за раз" subtitle="Claude читает только выбранный файл раздела. Контекст остается компактным, а вопросы остаются привязанными к материалу." >}}
  {{< card link="docs" title="3. Вопросы в формате экзамена" subtitle="Сценарная вводная → формулировка → ровно четыре варианта. Один правильный ответ. Три правдоподобных дистрактора из реальных анти-паттернов." >}}
  {{< card link="docs" title="4. Ответ, затем объяснение" subtitle="Claude ждет вашу букву, затем объясняет, почему правильный вариант верен и почему каждый дистрактор неверен, ссылаясь на концепт раздела по имени." >}}
{{< /cards >}}

## Экзамен в двух словах

| | |
| --- | --- |
| Формат | 60 вопросов multiple-choice, один правильный + три дистрактора |
| Время | 120 минут |
| Оценивание | Шкала 100–1000; проходной балл = **720** |
| Сценарии | **4 из 6** выбираются случайно на попытку |
| Домены | D1 Agentic Architecture & Orchestration **27%** · D2 Tool Design & MCP **18%** · D3 Claude Code Config **20%** · D4 Prompt Engineering & Structured Output **20%** · D5 Context Management & Reliability **15%** |

## Быстрый старт

```bash
git clone https://github.com/ai-1st/claude-certified-architect.git
cd claude-certified-architect
claude            # or open the folder in Cursor / Claude Code
# then say:  Start a mock exam.
```

Claude автоматически подхватывает `CLAUDE.md` и проводит вас через Mock Exam Mode.

## Дисклеймер

Неофициальный проект. Не связан с Anthropic и не одобрен Anthropic. Экзамен, его домены, сценарии и система оценивания принадлежат Anthropic. Учебные разделы в этом репозитории — независимое изложение публично доступных материалов, организованное для эффективной по контексту практики с Claude.
