---
title: "Claude Certified Architect — Foundations"
toc: false
---

{{< hextra/hero-badge >}}
  <div class="hx:w-2 hx:h-2 hx:rounded-full hx:bg-primary-400"></div>
  Неофициально · лицензия MIT · открытый исходный код
{{< /hextra/hero-badge >}}

{{< hextra/hero-headline >}}
  Готовьтесь к экзамену CCA-F с Claude в роли проктора.
{{< /hextra/hero-headline >}}

{{< hextra/hero-subtitle >}}
  Откройте репозиторий в Claude Code. Скажите *"start a mock exam."* &nbsp;
  Claude читает по одному учебному разделу за раз и задаёт вам вопросы в том же
  сценарном формате с четырьмя вариантами ответа, что и настоящий экзамен
  Claude Certified Architect — Foundations.
{{< /hextra/hero-subtitle >}}

{{< hextra/hero-button text="Начать с 12 учебных разделов" link="docs" >}}
&nbsp;
{{< hextra/hero-button text="Открыть на GitHub" link="https://github.com/ai-1st/claude-certified-architect" style="background: linear-gradient(90deg, #555 0%, #333 100%);" >}}

<div class="hx:mt-8 hx:flex hx:flex-wrap hx:items-center hx:justify-center hx:gap-2">
  <span class="hx:text-sm hx:font-semibold hx:opacity-70 hx:mr-2">Читать руководство на:</span>
  <a href="/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-neutral-700 hx:text-current hx:no-underline hx:transition hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800">English</a>
  <a href="/es/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-neutral-700 hx:text-current hx:no-underline hx:transition hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800">Español</a>
  <a href="/fr/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-neutral-700 hx:text-current hx:no-underline hx:transition hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800">Français</a>
  <a href="/ru/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:bg-neutral-900 hx:text-white hx:dark:bg-white hx:dark:text-neutral-900 hx:font-semibold hx:no-underline hx:transition hx:hover:opacity-90">Русский</a>
</div>

<div class="hx:mt-6"></div>

## Что это за репозиторий

Учебный помощник для экзамена **Claude Certified Architect — Foundations (CCA-F)** — первой официальной технической сертификации Anthropic, запущенной в марте 2026 года. Экзамен проверяет практическое инженерное суждение при создании продакшен-приложений на Claude: агентные циклы, инструменты MCP, настройку Claude Code, проектирование промптов и структурированного вывода, управление контекстом.

В репозитории есть две основные части:

1. Файл `CLAUDE.md`, который переводит Claude в **режим пробного экзамена** — режим проктора для посекционного опроса: Claude генерирует вопросы в официальном формате, ждёт ваш ответ, а затем объясняет, почему каждый вариант правильный или неправильный.
2. **Двенадцать учебных разделов** (`sections/01..12.md`), покрывающих все домены экзамена. Эти же файлы используются и для пробного экзамена в Claude, и для этого сайта.

## Как работает режим пробного экзамена

{{< cards >}}
  {{< card link="docs/01-exam-overview" title="1. Выберите раздел" subtitle="Claude приветствует вас и спрашивает, какой из 12 разделов или какой из 6 официальных сценариев вы хотите отработать." >}}
  {{< card link="docs/02-agentic-loops" title="2. Один файл за раз" subtitle="Claude читает только выбранный файл раздела. Контекст остаётся компактным, а вопросы остаются привязанными к материалу." >}}
  {{< card link="docs" title="3. Вопросы в формате экзамена" subtitle="Сценарная вводная → формулировка → ровно четыре варианта. Один правильный ответ. Три правдоподобных дистрактора из реальных анти-паттернов." >}}
  {{< card link="docs" title="4. Ответ, затем объяснение" subtitle="Claude ждёт вашу букву, затем объясняет, почему правильный вариант верен и почему каждый дистрактор неверен, ссылаясь на концепт раздела по имени." >}}
{{< /cards >}}

## Экзамен в двух словах

| | |
| --- | --- |
| Формат | 60 вопросов с выбором ответа, один правильный + три дистрактора |
| Время | 120 минут |
| Оценивание | Шкала 100–1000; проходной балл = **720** |
| Сценарии | **4 из 6** выбираются случайно на каждую попытку |
| Домены | D1 Агентная архитектура и оркестрация **27%** · D2 Проектирование инструментов и MCP **18%** · D3 Настройка Claude Code **20%** · D4 Проектирование промптов и структурированный вывод **20%** · D5 Управление контекстом и надёжность **15%** |

## Быстрый старт

```bash
git clone https://github.com/ai-1st/claude-certified-architect.git
cd claude-certified-architect
claude            # or open the folder in Cursor / Claude Code
# then say:  Start a mock exam.
```

Claude автоматически подхватывает `CLAUDE.md` и проводит вас через режим пробного экзамена.

## Дисклеймер

Неофициальный проект. Не связан с Anthropic и не одобрен Anthropic. Экзамен, его домены, сценарии и система оценивания принадлежат Anthropic. Учебные разделы в этом репозитории — независимое изложение публично доступных материалов, организованное для эффективной по контексту практики с Claude.
