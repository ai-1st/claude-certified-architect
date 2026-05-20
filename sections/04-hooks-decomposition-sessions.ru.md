---
title: "Раздел 4 — Хуки Agent SDK, декомпозиция задач и управление сессиями"
linkTitle: "4. Hooks, Decomposition & Sessions"
weight: 4
description: "Domains 1.5–1.7 — хуки PostToolUse и перехват вызовов, последовательность промптов vs адаптивная декомпозиция, fork_session и возобновление именованной сессии."
---

## Что покрывает этот раздел

Три тесно связанных навыка уровня архитектора, которые превращают агента Claude из вероятностного чат-бота в управляемую, отладочную систему:

1. **Хуки (1.5)** — детерминированные коллбэки на Python/TypeScript (или shell-скрипты), которые перехватывают агентный цикл в чётко определённых точках жизненного цикла, чтобы обеспечивать соблюдение политик, нормализовать вывод инструментов и аудировать каждое действие.
2. **Декомпозиция задач (1.6)** — понимание, когда жёстко прошивать последовательный конвейер (последовательность промптов), а когда позволить модели динамически порождать собственные подзадачи (оркестратор–воркеры / адаптивные планы).
3. **Управление сессиями (1.7)** — операционная дисциплина `--continue`, `--resume` и `--fork-session`, плюс умение оценить, *когда* выбросить сессию и начать заново со структурированной сводкой.

Критерий прохождения: посмотреть на рабочий процесс и сразу сказать «это задача для хука, а не для промпта», «это последовательность промптов, а не оркестратор–воркеры» или «эта сессия устарела, нужна сводка и перезапуск».

## Исходный материал (из официального руководства)

### 1.5 Хуки для перехвата и нормализации

`PostToolUse` перехватывает *результаты* инструментов и преобразует их до того, как модель увидит сырые байты. `PreToolUse` перехватывает *вызовы* инструментов и может блокировать, изменять или перенаправлять — канонический пример: «блокировать любой возврат, где `amount > 500`, и направлять на человеческую эскалацию». Хуки дают **детерминированные гарантии**; промпты — только **вероятностное соблюдение**. Навыки: нормализовать разнородные временные метки от разных MCP-серверов; блокировать действия, нарушающие политику, и перенаправлять на альтернативы; выбирать хуки вместо промптов, когда соблюдение должно быть гарантировано.

### 1.6 Стратегии декомпозиции задач

**Последовательность промптов** для предсказуемых процессов, где шаги известны заранее, vs **динамическая адаптивная декомпозиция** для открытых процессов, где подзадачи можно обнаружить только во время выполнения. Большие ревью кода: пофайловый локальный анализ + отдельный кросс-файловый интеграционный проход, чтобы избежать размывания внимания.

### 1.7 Состояние сессии, возобновление и форк

`--resume <id-or-name>` продолжает конкретный прежний диалог; `--continue`/`-c` возобновляет самый свежий в текущем `cwd`. `--fork-session` (`fork_session: true` / `forkSession: true` в SDK) создаёт независимую ветку от общей базовой точки. После изменений в файлах на диске нужно сообщить возобновлённому агенту, иначе кэшированные результаты `Read` устареют; свежая сессия, засеянная вручную написанной сводкой, иногда надёжнее возобновления со старыми результатами инструментов.

## Справочник по хукам

### Типы событий хуков

| Событие | Когда срабатывает | Типичное применение |
| --- | --- | --- |
| `SessionStart` | Сессия начинается или возобновляется (matchers: `startup`, `resume`, `clear`, `compact`) | Инициализация логирования, внедрение правил проекта |
| `SessionEnd` | Сессия завершается | Сброс логов, очистка |
| `UserPromptSubmit` | Пользователь отправляет промпт, до того как модель его увидит | Внедрение контекста, очистка PII, блокировка не по теме |
| `PreToolUse` | Перед выполнением любого вызова инструмента | **Принуждение политик**, переписывание ввода, перенаправление в песочницу |
| `PostToolUse` | После успешного вызова инструмента | **Нормализация данных**, аудит-логирование, конвертация форматов |
| `PostToolUseFailure` | После неуспешного вызова инструмента | Пользовательская обработка ошибок |
| `PostToolBatch` | Параллельный батч вызовов инструментов разрешён | Внедрение соглашений один раз на батч |
| `PermissionRequest` / `PermissionDenied` | Диалог разрешений или отказ в auto-режиме | Кастомный UX, решения о повторе |
| `SubagentStart` / `SubagentStop` | Субагент запускается / завершается | Отслеживание параллельной работы, агрегация результатов |
| `PreCompact` / `PostCompact` | Жизненный цикл уплотнения диалога | Архивирование транскрипта перед потерей данных при сводке |
| `Notification` | Агент выдаёт уведомление | Пересылка в Slack/PagerDuty |
| `Stop` / `StopFailure` | Ход завершается нормально / через ошибку API | Сохранение состояния, оповещение по rate-limit |
| `TaskCreated` / `TaskCompleted` | Жизненный цикл задачи | Соблюдение соглашений по ID тикетов, гейт по прохождению тестов |
| `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `FileChanged`, `WorktreeCreate/Remove`, `Setup` | Прочие события жизненного цикла | Аудит, перезагрузка конфига, реакция на внешние изменения |

`SessionStart`/`SessionEnd` существуют как коллбэки только в TS SDK; в Python они должны быть shell-хуками в `.claude/settings.json` плюс `setting_sources=["project"]`.

### Анатомия хука

Два пути регистрации: **коллбэки SDK** (`ClaudeAgentOptions.hooks` / `options.hooks`) или **shell-хуки команд** в `.claude/settings.json` — дочерние процессы, получающие JSON события на stdin.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }
        ]
      }
    ]
  }
}
```

**Matchers**: `*`/`""`/опущенный — совпадает со всем; буквы+цифры+`|` — точное совпадение или список через pipe (`Edit|Write`); всё остальное — JS-регулярное выражение (`^mcp__memory__`). Хуки на инструменты сопоставляются только по *имени* инструмента — фильтруйте `file_path` внутри обработчика.

**Ввод** — каждый хук получает `session_id`, `cwd`, `hook_event_name` плюс поля, специфичные для события (`tool_name`, `tool_input`, `tool_response`, …). Контекст субагента добавляет `agent_id`, `agent_type`.

**Вывод** — JSON в два слоя: верхний уровень (`systemMessage`, `continue` / `continue_`, `additionalContext`) и `hookSpecificOutput` (зависит от события). Для `PreToolUse`: `permissionDecision` ∈ `{"allow", "deny", "ask", "defer"}`, `permissionDecisionReason`, `updatedInput`. Для `PostToolUse`: `additionalContext` (добавить) или `updatedToolOutput` (заменить).

**Коды выхода shell-хука**: `0` = успех, парсить stdout как JSON; `2` = блокирующая ошибка, stderr передаётся модели (для `PreToolUse` блокирует вызов); любое другое ненулевое = неблокирующая ошибка.

Когда несколько хуков срабатывают на одном событии: **deny > defer > ask > allow** — один `deny` блокирует всё.

### Конкретные примеры

**1. PostToolUse, нормализующий разнородные форматы дат** — три MCP-сервера возвращают Unix epoch seconds, ISO 8601 строки и числовые миллисекунды. Принудительно приводим к одному представлению ISO 8601 до того, как модели придётся об этом рассуждать.

```python
from datetime import datetime, timezone

async def normalize_timestamps(input_data, tool_use_id, context):
    if input_data["hook_event_name"] != "PostToolUse":
        return {}
    response = input_data.get("tool_response", {})
    raw_ts = response.get("timestamp")
    if raw_ts is None:
        return {}

    if isinstance(raw_ts, (int, float)):
        ts = raw_ts / 1000 if raw_ts > 1e12 else raw_ts
        iso = datetime.fromtimestamp(ts, tz=timezone.utc).isoformat()
    else:
        iso = datetime.fromisoformat(str(raw_ts).replace("Z", "+00:00")).isoformat()

    response["timestamp"] = iso
    return {
        "hookSpecificOutput": {
            "hookEventName": "PostToolUse",
            "updatedToolOutput": response,
        }
    }

options = ClaudeAgentOptions(
    hooks={"PostToolUse": [HookMatcher(matcher="^mcp__", hooks=[normalize_timestamps])]}
)
```

Модель всегда видит только ISO 8601, поэтому арифметика дат между инструментами просто работает.

**2. PreToolUse, блокирующий возвраты на крупные суммы** — гарантируем, что `process_refund` никогда не сработает свыше $500.

```typescript
import { HookCallback, PreToolUseHookInput } from "@anthropic-ai/claude-agent-sdk";

const refundGuard: HookCallback = async (input) => {
  if (input.hook_event_name !== "PreToolUse") return {};
  const pre = input as PreToolUseHookInput;
  if (pre.tool_name !== "mcp__billing__process_refund") return {};

  const amount = (pre.tool_input as { amount?: number }).amount ?? 0;
  if (amount > 500) {
    return {
      systemMessage: `Refund of $${amount} exceeds policy cap; escalating.`,
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny",
        permissionDecisionReason:
          "Refunds above $500 require human approval. Use mcp__support__create_escalation instead."
      }
    };
  }
  return {};
};
```

`permissionDecisionReason` — волшебный ингредиент: он сообщает модели, *почему* вызов был заблокирован и каким альтернативным инструментом воспользоваться, поэтому агент самокорректируется на следующем ходу, а не зацикливается на том же отклонённом вызове.

### Когда НЕ использовать хук (использовать промпт)

| Ситуация | Хук | Промпт |
| --- | --- | --- |
| Жёсткое регуляторное правило («никогда не логировать SSN») | Да | Нет |
| Контракт по форме данных («всегда ISO 8601») | Да | Нет |
| Аудит-лог каждого вызова инструмента | Да | Нет |
| Квота / потолок стоимости на вызов | Да (PreToolUse) | Нет |
| Мягкое стилевое предпочтение («предпочитать отступ в 2 пробела») | Нет | Да |
| Персона / тон («будь краток, без эмодзи») | Нет | Да |

Правило большого пальца: если нарушение этого — инцидент уровня P0, оно живёт в хуке.

## Паттерны декомпозиции

### Последовательность промптов (sequential, predictable)

Фиксированный N-шаговый конвейер, где шаг *k* подаёт данные в шаг *k+1*. Каждый вызов LLM решает более лёгкую задачу, чем один мега-промпт. Из [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents): *«идеально, когда задача легко и чисто декомпозируется на фиксированные подзадачи. Главная цель — обменять задержку на повышенную точность, делая каждый вызов LLM проще».*

Добавляйте программные *гейты* между шагами для быстрого отказа: набросок → проверка рубрики (гейт) → написание документа; пофайловый lint-анализ → кросс-файловое интеграционное ревью.

### Динамическая адаптивная декомпозиция

Также известная как **оркестратор–воркеры**. LLM-оркестратор смотрит на ввод, решает, какие подзадачи нужны (заранее это знать было невозможно), запускает воркеров и синтезирует результаты. *«Подходит для сложных задач, в которых нельзя предсказать необходимые подзадачи (в разработке количество файлов, требующих изменений, и характер изменений в каждом, как правило, зависят от задачи)».*

В Agent SDK это отображается на паттерн `Task` / субагент, опционально отслеживаемый хуками `SubagentStart`/`SubagentStop`.

### Матрица решений

| Форма рабочего процесса | Паттерн |
| --- | --- |
| Шаги известны заранее, одинаковы для каждого входа | Последовательность промптов |
| Независимые подзадачи *известной* формы | Параллелизация (sectioning) |
| Одна задача, нужно N голосов для уверенности | Параллелизация (voting) |
| Различимые категории, каждая со своим специалистом | Маршрутизация |
| Число/форма подзадач зависят от ввода | Оркестратор–воркеры (адаптивно) |
| Выход выигрывает от цикла критики | Оценщик–оптимизатор |
| Открытый, многошаговый, с неизвестным горизонтом | Полный агентный цикл |

### Разобранный пример — пофайловое + кросс-файловое ревью кода

PR из 40 файлов, втиснутый в один промпт, страдает от размывания внимания: модель пробегает поверх и пропускает баги. Декомпозируем:

**Фаза 1 — пофайловый локальный анализ (цепочка, параллелизуется)**: каждый файл получает свой контекст, ответвлённый от общей базовой сессии, в которую уже загружены `CLAUDE.md`, описание PR и diff-stat. Форк избавляет от повторной оплаты токенов на восстановление контекста на каждом файле.

```python
async def review_file(path: str, baseline_sid: str) -> dict:
    async for msg in query(
        prompt=f"Review {path}. Find correctness bugs, missing error handling, "
               f"and security issues. Output JSON {{'file','findings'}}.",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Grep"],
            resume=baseline_sid,
            fork_session=True,
            max_turns=8,
        ),
    ):
        if isinstance(msg, ResultMessage) and msg.subtype == "success":
            return json.loads(msg.result)
```

**Фаза 2 — кросс-файловый интеграционный проход (один вызов):**

```python
findings = await asyncio.gather(*(review_file(p, baseline_sid) for p in changed_files))
async for msg in query(
    prompt=f"Per-file findings: {json.dumps(findings)}. "
           f"Identify cross-cutting issues: API contract drift, type mismatches, "
           f"missing call-site updates, security holes spanning files.",
    options=ClaudeAgentOptions(resume=baseline_sid, allowed_tools=["Read", "Grep"]),
):
    ...
```

Последовательность промптов (Фаза 1 → Фаза 2), наложенная поверх параллелизации внутри Фазы 1.

**Открытый вариант — «добавить тесты в legacy-кодовую базу»**: *адаптивный*, не цепочечный. Картографирование структуры → выявление модулей высокого влияния без тестов → построение приоритизированного бэклога → берём верхний элемент, пишем тесты, обнаруживаем скрытую зависимость, кладём её обратно в бэклог, повторяем. Шаги 4..N узнаваемы только во время выполнения — это оркестратор–воркеры, а не цепочка.

## Управление сессиями

### `--resume` vs `--continue` vs новая сессия

| Что нужно | Что использовать | Как находит сессию |
| --- | --- | --- |
| Самая свежая сессия в этом каталоге | `claude -c` / `continue: true` | Самая новая в `~/.claude/projects/<cwd-slug>/` |
| Конкретная именованная или ID-сессия | `claude -r "auth-refactor"` / `resume: "<id>"` | Точный ID или поиск по `--name` |
| Совершенно новый диалог | `claude` | Свежий ID сессии |
| Одноразовый запуск без записи на диск (только TS) | `persistSession: false` | Только в памяти |

Сессии живут в `~/.claude/projects/<slugified-cwd>/<session-id>.jsonl`, где slug — это абсолютный рабочий каталог, в котором каждый небуквенно-цифровой символ заменён на `-`. **Другой `cwd` означает, что `resume` не найдёт файл** — причина №1 «почему resume возвращает свежую сессию».

Фиксируйте ID сессии из `ResultMessage` (Python) / `SDKResultMessage` (TS) при каждом запуске, если планируете возобновлять программно. В TS он также есть в инициализирующем `SystemMessage`.

### `fork_session`: когда и как

Форк копирует существующий транскрипт в *новый* ID сессии и позволяет ей расходиться. Оригинал не трогается.

```python
forked_id = None
async for message in query(
    prompt="Try OAuth2 instead of JWT for the auth module",
    options=ClaudeAgentOptions(resume=session_id, fork_session=True),
):
    if isinstance(message, ResultMessage):
        forked_id = message.session_id
```

```typescript
for await (const message of query({
  prompt: "Try OAuth2 instead of JWT for the auth module",
  options: { resume: sessionId, forkSession: true }
})) {
  if (message.type === "system" && message.subtype === "init") {
    forkedId = message.session_id;
  }
}
```

Случаи использования: A/B-сравнение двух подходов к рефакторингу из общей базовой точки; bake-off стратегий тестирования; рискованное исследование с гарантированным откатом к родителю.

Оговорка: форк ветвит *диалог*, а не *файловую систему*. Если обе ветки правят файлы в одном репозитории, эти правки сталкиваются. Комбинируйте с [file checkpointing](https://code.claude.com/docs/en/agent-sdk/file-checkpointing) или git worktrees (`claude -w <name>`) для настоящей изоляции.

### Дерево решений для устаревшего контекста

После изменения кода, прежде чем возобновлять расследование:

```
Did the agent's last tool calls touch files that have since been edited?
├── No  → Resume normally.
└── Yes →
    Is the affected surface small AND were the edits surgical?
    ├── Yes → Resume + tell the agent: "Files X, Y were edited since
    │         you last saw them. Re-Read them before continuing."
    └── No  → Throw the session away. Start fresh with a structured
              summary: (a) goal, (b) decisions made, (c) current
              state of the code, (d) open questions.
```

Чистая сессия с тщательно подобранной сводкой часто превосходит возобновление, потому что в возобновлённом транскрипте всё ещё содержатся устаревшие результаты `Read`, которым модель доверяет — она может «помнить» сигнатуры функций, которых больше нет, и галлюцинировать вызовы. Паттерн: держите долгоживущую **именованную сессию** (`claude -n design-review`) для устойчивого архитектурного контекста и **ответвляйте** её на каждое расследование. Форки выбрасывайте.

## Экзаменационные акценты

- **Хуки детерминированы; промпты вероятностны.** Любое правило, которое должно соблюдаться на 100%, живёт в `PreToolUse` (исходящий поток) или `PostToolUse` (входящий поток).
- Запомните значения `permissionDecision`: `allow`, `deny`, `ask`, `defer`. Запомните приоритет: **deny > defer > ask > allow**.
- `PostToolUse.updatedToolOutput` *заменяет* то, что видит модель; `additionalContext` *добавляет*. `PreToolUse.updatedInput` переписывает ввод инструмента — но только если вы также возвращаете `permissionDecision: "allow"`.
- Коды выхода shell-хука: `0` = парсить JSON; `2` = блокировать (PreToolUse) / передать stderr модели; всё остальное = неблокирующая ошибка.
- Выбор декомпозиции: форма известна → последовательность промптов; форма неизвестна → оркестратор–воркеры. Большое ревью кода = пофайловый проход + кросс-файловый интеграционный проход.
- `--continue` не требует ID, но находит только самую свежую сессию в *текущем* `cwd`. `--resume` требует ID или `--name`. `--fork-session` требует `--resume`/`--continue` и порождает новый ID.
- Хранилище сессий: `~/.claude/projects/<slugified-cwd>/<session-id>.jsonl`. Неверный `cwd` — причина №1 того, что resume молча возвращает свежую сессию.
- Устаревшие результаты инструментов в возобновлённой сессии могут навредить сильнее, чем помочь — иногда свежая сессия с тщательно подобранной сводкой превосходит возобновление.
- Форк — чтобы сравнить; resume — чтобы продолжить; перезапуск со сводкой — когда контекст протух.

## References

- [Intercept and control agent behavior with hooks](https://code.claude.com/docs/en/agent-sdk/hooks) — полный справочник по хукам Agent SDK, таблица событий, форма коллбэков, примеры.
- [Hooks reference](https://code.claude.com/docs/en/hooks) — каждое событие, шаблоны matchers, JSON-вывод, семантика кодов выхода, варианты shell/HTTP/MCP-инструментальных хуков.
- [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide) — быстрый старт с разобранными примерами.
- [Configure hooks (Anthropic blog)](https://claude.com/blog/how-to-configure-hooks) — обзор `settings.json` для опытных пользователей.
- [Work with sessions](https://code.claude.com/docs/en/agent-sdk/sessions) — `continue`, `resume`, `fork_session`, фиксация ID, кросс-хостовые оговорки.
- [CLI reference](https://code.claude.com/docs/en/cli-reference) — `--continue`, `--resume`, `--fork-session`, `--name`, `--session-id`, `--from-pr`.
- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — каноническая таксономия Anthropic: последовательность промптов, маршрутизация, параллелизация, оркестратор–воркеры, оценщик–оптимизатор.
- [Anthropic cookbook — agents patterns](https://github.com/anthropics/claude-cookbooks/tree/main/patterns/agents) — выполнимые ноутбуки по каждому паттерну.
- [How the agent loop works](https://code.claude.com/docs/en/agent-sdk/agent-loop) — концептуальная модель ходов, вызовов инструментов и места, где встраиваются хуки.
- [Permissions](https://code.claude.com/docs/en/agent-sdk/permissions) — сопутствующий механизм рядом с `PreToolUse` для контроля доступа на уровне инструмента.
