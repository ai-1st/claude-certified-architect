---
title: "Раздел 9 — Claude Code в пайплайнах CI/CD"
linkTitle: "9. Интеграция с CI/CD"
weight: 9
description: "Домен 3.6 — claude -p и --output-format json, --json-schema, дизайн промптов для действенного ревью с низкой долей ложных срабатываний."
---

## Что покрывает этот раздел

Как запускать Claude Code без присмотра внутри сборочной системы: флаги, предотвращающие интерактивные зависания; формы вывода, которые могут разбирать инструменты ниже по конвейеру; контекст уровня проекта, удерживающий модель в рамках политик; и архитектурные паттерны (независимый ревьюер, многопроходное ревью, батч против синхрона), которые экзамен проверяет вопросами 10, 11 и 12.

## Исходный материал (из официального руководства)

Task Statement 3.6 (строки 600–629 руководства) покрывает: `-p` / `--print` для неинтерактивного режима; `--output-format json` плюс `--json-schema` для машиночитаемых замечаний; `CLAUDE.md` как механизм проектного контекста для Claude Code, вызываемого из CI (стандарты тестирования, соглашения по фикстурам, критерии ревью); а также изоляцию контекста сессии — сессия, которая *написала* код, — неподходящая сессия для того, чтобы его *ревьюить*. Навыки: предотвращать интерактивные зависания через `-p`; выдавать структурированные замечания для встроенных комментариев к PR; передавать предыдущие замечания обратно, чтобы подавить дубликаты комментариев; передавать существующие тестовые файлы при генерации тестов, чтобы не покрывать заново уже покрытые сценарии; документировать стандарты тестирования и фикстуры в `CLAUDE.md`.

Sample Question 10 (`-p` — правильный флаг для headless; `CLAUDE_HEADLESS`, `--batch` и `< /dev/null` — отвлекающие варианты), Question 11 (Message Batches API для ночного отчёта о техдолге, *а не* для блокирующей проверки до merge) и Question 12 (разбить PR из 14 файлов на проходы по файлу плюс проход интеграции) — все опираются на этот раздел.

## Headless / неинтерактивный Claude Code

### Флаг `-p`

`claude -p "<prompt>"` (синоним `--print`) запускает Claude Code как одноразовый SDK-запрос: он принимает промпт, стримит или печатает ответ в stdout и завершается с кодом возврата. Никакой проверки TTY, никакого цикла подтверждения разрешений, никакого ожидания на stdin. Это единственный документированный механизм для CI. Ловушка из вопроса 10: переменной окружения `CLAUDE_HEADLESS` нет, флага `--batch` нет, а перенаправление stdin из `/dev/null` не работает, потому что Claude Code всё равно ожидает рендерить интерактивный UI.

```bash
claude -p "Review the diff for SQL-injection and SSRF risks. Be terse." \
  --model claude-sonnet-4-6 \
  --max-turns 6
```

Добавляйте `--bare` (Claude Code 2.1.x), когда вам не нужны хуки, плагины, MCP-серверы или автоматически обнаруженный `CLAUDE.md`; это сокращает задержку запуска и устраняет класс отказов «почему мой CI подхватил MCP-конфиг с моего ноутбука».

### Форматы вывода (text, json, stream-json)

`--output-format` управляет тем, как сериализуется ответ:

- `text` (по умолчанию) — простая проза; хорошо для людей, плохо для парсеров.
- `json` — один JSON-объект с финальным ответом, стоимостью, длительностью, идентификатором сессии и причиной остановки. Используйте, когда следующий шаг — `jq` или скрипт.
- `stream-json` — события, разделённые переводами строк, для инструментов, которые отображают прогресс или показывают события использования инструментов. Сочетайте с `--include-partial-messages` для дельт по токенам, с `--include-hook-events` — для жизненного цикла хуков.

`--input-format stream-json` позволяет родительскому процессу подавать в Claude Code структурированный поток событий — полезно, если ваш CI-оркестратор сам является агентом.

### `--json-schema` для структурированных замечаний

`--json-schema '<JSON Schema>'` (только в print-режиме) заставляет Claude Code выдавать финальный payload, *валидируемый* против указанной вами схемы. Это тот флаг, к которому экзамен ожидает обращения, когда следующий этап конвейера — «опубликовать каждое замечание как встроенный review-комментарий к PR в GitHub».

```bash
claude -p "Review changed files in $PR_DIFF for security issues" \
  --output-format json \
  --json-schema "$(cat .ci/review-schema.json)" \
  --max-turns 8 \
  > findings.json
```

Схема, чисто отображающаяся в payload `pulls/{pr}/comments` от GitHub, выглядит так:

```json
{
  "type": "object",
  "required": ["findings"],
  "properties": {
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["path", "line", "severity", "message"],
        "properties": {
          "path":     { "type": "string" },
          "line":     { "type": "integer", "minimum": 1 },
          "severity": { "enum": ["info", "warning", "error", "critical"] },
          "category": { "enum": ["security", "perf", "bug", "style"] },
          "message":  { "type": "string", "maxLength": 800 }
        }
      }
    }
  }
}
```

### Режимы разрешений для запусков без присмотра

Интерактивные запросы разрешений — вторая по частоте причина зависших CI-задач (после забытого `-p`). Ключевые флаги:

- `--allowedTools "Read,Bash(git diff *),Bash(gh pr view *)"` — узкий allowlist; предпочтительное значение по умолчанию.
- `--disallowedTools "Edit,Write,Bash(rm *)"` — блокирует только записи, когда задача — ревью только на чтение.
- `--permission-mode acceptEdits` (или `bypassPermissions`) — принимает изменения без запросов. `--dangerously-skip-permissions` — это явная форма «я понимаю»; оставьте её для песочничных раннеров без секретов в окружении.
- `--permission-prompt-tool <mcp-tool>` делегирует решения о разрешениях кастомному MCP-инструменту, когда каждый вызов должен логироваться перед одобрением.

### Флаги управления сессиями

- `--session-id <uuid>` закрепляет UUID, чтобы последующие шаги могли возобновить тот же диалог позже.
- `--resume <id>` / `-r` продолжает эту сессию — механизм, стоящий за «включить предыдущие замечания при повторном запуске».
- `--continue` / `-c` подхватывает самую свежую сессию в текущей директории.
- `--fork-session` возобновляет с ответвлением, чтобы повторные запуски не меняли каноническую ветку.
- `--no-session-persistence` ничего не пишет на диск; полезно в безсостояний раннерах, где рабочее пространство уничтожается между задачами.

## Эталонные паттерны

### 1. Проверка безопасности до merge (синхронная, блокирующая)

Блокирующие гейты должны использовать реальный вызов в режиме реального времени. Согласно вопросу 11, Message Batches API здесь — неподходящий инструмент: его скидка в 50% по стоимости даётся ценой задержки до 24 часов, что неприемлемо для разработчика, ждущего merge.

```yaml
- name: Claude security review
  run: |
    git fetch origin ${{ github.base_ref }}
    DIFF=$(git diff origin/${{ github.base_ref }}...HEAD)
    echo "$DIFF" | claude -p \
      "Find security issues only. Reply with the findings schema." \
      --output-format json \
      --json-schema "$(cat .ci/review-schema.json)" \
      --allowedTools "Read,Bash(git diff *)" \
      --max-turns 6 \
      --max-budget-usd 2.00 \
      > findings.json
```

### 2. Генерация тестов в ночном батче

Ночные задачи по тех-долгу тестов — ровно та нагрузка, на которой Message Batches API окупает свою скидку в 50%. Запускайте Claude Code с `-p`, передавайте существующие тестовые файлы в контекст, чтобы он не покрывал заново уже покрытые сценарии, и направляйте запросы на генерацию через Message Batches API (см. раздел 11).

```bash
ls tests/ | xargs -I{} cat tests/{} > .ci/existing-tests.txt

claude -p \
  "Generate missing tests for src/billing/. Existing tests are attached; do not duplicate scenarios already covered." \
  --append-system-prompt-file .ci/existing-tests.txt \
  --output-format json \
  --max-turns 12
```

### 3. Публикация встроенных комментариев к PR из JSON-вывода

С `findings.json`, провалидированным по схеме, одного shell-цикла достаточно, чтобы превратить файл в нативные review-комментарии:

```bash
jq -c '.findings[]' findings.json | while read f; do
  gh api -X POST "repos/$REPO/pulls/$PR/comments" \
    -f body="$(echo $f | jq -r .message)" \
    -f path="$(echo $f | jq -r .path)" \
    -F line="$(echo $f | jq -r .line)" \
    -f commit_id="$SHA" -f side=RIGHT
done
```

### 4. Перезапуск ревью на новых коммитах без дублирующих комментариев

Домен 3.6 прямо требует: при повторном пуше в PR включайте предыдущие замечания в контекст и инструктируйте Claude сообщать только о новых либо всё ещё нерешённых проблемах.

```bash
gh pr view $PR --json comments -q '.comments' \
  | claude -p \
      "Comments already posted are attached. Re-review the new diff and emit ONLY findings that are (a) new or (b) still unaddressed. Use the findings schema." \
      --resume "$REVIEW_SESSION_ID" \
      --output-format json \
      --json-schema "$(cat .ci/review-schema.json)" \
      > new-findings.json
```

## GitHub Action для Claude Code

### Быстрый старт

```yaml
name: Claude review
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  review:
    runs-on: ubuntu-latest
    permissions: { contents: read, pull-requests: write, issues: write }
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "Review this PR for security issues. Use inline comments."
          claude_args: >-
            --max-turns 10
            --model claude-sonnet-4-6
            --allowedTools "mcp__github_inline_comment__create_inline_comment,Bash(gh pr diff:*),Bash(gh pr view:*),Read"
```

### Входы и выходы

Ключевые входы из `action.yml`: `anthropic_api_key` (или `claude_code_oauth_token`, `use_bedrock`, `use_vertex`); `prompt`; `claude_args` (сырые аргументы CLI, дописываемые к лежащему ниже `claude -p`); `trigger_phrase` (по умолчанию `@claude`); `label_trigger` (по умолчанию `claude`); `track_progress` (рендерит трекинг-комментарий с чек-боксами); `use_sticky_comment` (один комментарий на всё); `classify_inline_comments` (Haiku-пре-фильтр, отбрасывающий пробные/тестовые комментарии); `branch_prefix` (по умолчанию `claude/`). Структурированный JSON, возвращаемый запуском, выставляется как выходы GitHub Action.

### Частые подводные камни

- MCP-инструменту встроенных комментариев нужен `confirmed: true`, чтобы публиковать сразу; иначе комментарии буферизуются в `/tmp/inline-comments-buffer.jsonl` для классификации через Haiku.
- Action требует прав токена `pull-requests: write` и `issues: write`, иначе он молча перестаёт публиковать.
- Выбор модели и лимит ходов передавайте через `claude_args`, а не через отдельные входы.

## CLAUDE.md для CI

### Разделы, которые окупаются

Для Claude, вызываемого из CI, разделы `CLAUDE.md` с наибольшим рычагом — это те, о которых модели иначе пришлось бы догадываться: команда раннера тестов, где лежат фикстуры, какую мокирующую библиотеку вы используете, что считается «ценным» тестом и какие критерии ревью вам важны — ровно те стандарты тестирования, фикстуры и критерии ревью, на которые указывают пункты «Навыки» для 3.6.

```markdown
## Testing
- Runner: `pnpm test --filter=$PKG`
- Mocks: use MSW only; never mock `fetch` directly
- Fixtures: tests/fixtures/*.json (load with `loadFixture()`)
- Valuable: state transitions, error branches, boundary inputs
- Low-value: getter coverage, framework re-tests

## Review criteria
- Block on: SQLi, SSRF, secrets, auth bypass, N+1 in hot paths
- Comment-only: style, naming, missing comments
```

### Избегание раздувания контекста

`CLAUDE.md` загружается при каждом вызове, поэтому за каждую его строку вы платите на каждом запуске CI. Удаляйте исторические решения, ссылайтесь на ADR вместо вставки, предпочитайте одну декларативную строку целому абзацу и не поддавайтесь искушению вставить весь стайл-гайд. В режиме `--bare` `CLAUDE.md` не загружается автоматически вовсе; подключайте его через `--append-system-prompt-file`, когда он вам нужен.

## Паттерн независимого ревьюера

### Почему отдельная сессия лучше самопроверки

Домен 3.6 явно выделяет *изоляцию контекста сессии*. Сессия, сгенерировавшая код, несёт собственную историю рассуждений — те самые оправдания, которые породили баг. Эта сессия предвзята в сторону подтверждения своих предыдущих решений. Свежая сессия видит только дифф, без приложенной нарративной истории, и ловит замечания, на которые сессия автора закрыла глаза. Собственный продукт Anthropic Code Review использует флот независимых агентов по той же причине.

На практике: никогда не `--continue` из сессии реализации в задачу ревью. Запускайте ревьюера с новым `--session-id`, пустым диалогом и только с диффом плюс `CLAUDE.md` в качестве контекста.

### Многопроходное ревью для больших PR (проход по файлу + проход интеграции)

Это вопрос 12. Один проход ревью по 14 файлам демонстрирует *размывание внимания* — глубина варьируется от файла к файлу, а идентичные паттерны получают противоречивые вердикты. Решение — два прохода:

1. **Проход по файлу.** Один вызов `claude -p` на каждый изменённый файл, ограниченный локальными замечаниями (баги, стиль, безопасность в этом файле).
2. **Проход интеграции.** Один дополнительный вызов получает сводку диффа и специально спрашивается о межфайловых вопросах: изменения API-границ, миграции схем, дрейф контрактов, мутация общего состояния.

Более крупная модель или большее контекстное окно эту проблему не решают; вопрос в распределении внимания, а не в бюджете токенов.

## Рычаги стоимости и задержки

### Prompt caching

Чтения из кэша обходятся примерно в 10 раз дешевле записи в кэш, и Claude Code построен вокруг агрессивного попадания в кэш. Чтобы максимизировать попадания в CI: ставьте стабильный контент (системный промпт, `CLAUDE.md`, соглашения репозитория) *в начало*, динамический (дифф, предыдущие замечания) — *в конец*; не меняйте модель или список инструментов посреди сессии; в Claude Code 2.1.108+ выставляйте `ENABLE_PROMPT_CACHING_1H=1` для запланированных задач, срабатывающих чаще, чем раз в пять минут. Используйте `--exclude-dynamic-system-prompt-sections` вместе с `-p`, чтобы не пускать машинно-специфичные детали в ключ кэша, когда одну и ту же задачу разделяют много раннеров.

### Batch API для неблокирующих нагрузок

Перекрёстная ссылка на раздел 11: Message Batches API даёт экономию 50% по стоимости, но SLA до 24 часов. Используйте его для ночного отчёта о техдолге; не используйте для гейта до merge. Это и есть различие, которое проверяет Sample Question 11.

### Выбор размера модели под задачу

Проверка безопасности до merge на горячем пути: Sonnet. Стиль-и-линт проход на PR с документами: Haiku. Архитектурное ревью на ежеквартальный рефакторинг: Opus, с ограничением через `--max-budget-usd`. Переключайте модель через `--model`; не переключайте посреди сессии, иначе инвалидируете кэш.

## Ключевые акценты для экзамена

- `-p` — это *единственный* документированный способ запустить Claude Code без присмотра. `CLAUDE_HEADLESS`, `--batch` и `</dev/null` — отвлекающие варианты.
- `--output-format json --json-schema <schema>` — каноничный рецепт для замечаний, парсимых конвейером.
- Блокирующие нагрузки (до merge) используют вызовы в реальном времени; неблокирующие ночные нагрузки — Message Batches API.
- Большие PR: проходы по файлу плюс один проход интеграции — не один мега-промпт и не большее контекстное окно.
- Сессия, написавшая код, — неподходящая сессия для ревью; поднимайте независимого ревьюера.
- Перезапуск ревью обязан включать предыдущие замечания в контекст, чтобы подавить дубликаты комментариев.
- `CLAUDE.md` несёт стандарты тестирования, фикстуры и критерии ревью — за каждую строку платится на каждом запуске CI, поэтому держите его компактным.

## Ссылки

- Справочник CLI Claude Code: <https://docs.claude.com/en/docs/claude-code/cli-usage>
- Запуск Claude Code программно (headless): <https://code.claude.com/docs/en/headless>
- Claude Agent SDK (Python / TypeScript): <https://code.claude.com/docs/en/agent-sdk/python>, <https://code.claude.com/docs/en/agent-sdk/typescript>
- Structured outputs: <https://code.claude.com/en/agent-sdk/structured-outputs>
- GitHub Action для Claude Code: <https://github.com/anthropics/claude-code-action> (использование: `docs/usage.md`, рецепты: `docs/solutions.md`)
- Документация GitHub Actions: <https://docs.anthropic.com/en/docs/claude-code/github-actions>
- Продукт Code Review: <https://code.claude.com/docs/en/code-review> и <https://claude.com/blog/code-review>
- Паттерн субагентов: <https://claude.com/blog/subagents-in-claude-code>
- Prompt caching: <https://claude.com/blog/lessons-from-building-claude-code-prompt-caching-is-everything> и <https://docs.claude.com/en/docs/build-with-claude/prompt-caching>
- Memory / CLAUDE.md: <https://code.claude.com/en/memory>
