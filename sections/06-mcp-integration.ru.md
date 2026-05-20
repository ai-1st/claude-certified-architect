---
title: "Раздел 6 — Интеграция MCP: ошибки, серверы и ресурсы"
linkTitle: "6. MCP Integration"
weight: 6
description: "Domains 2.2, 2.4 — структурированные конверты ошибок (isError, isRetryable), области видимости и подстановка переменных окружения в .mcp.json, ресурсы vs инструменты."
---

## Что покрывает этот раздел

Две операционно критичные области MCP: как инструменты должны *сообщать о сбоях*, чтобы агент мог разумно восстанавливаться (2.2), и как серверы и ресурсы подключать к Claude Code, чтобы команда получила правильные возможности на правильной области видимости (2.4). MCP — это контракт между сервером и агентом; качество этого контракта (ошибки, описания, ресурсы) определяет, принимает ли агент разумные решения или мечется.

## Исходный материал (из официального руководства)

### 2.2 Структурированные ответы об ошибках

Знание: четыре класса ошибок (transient, validation, business, permission); почему однообразные ответы `"Operation failed"` ломают восстановление; retryable vs non-retryable.

Навыки: возвращать `errorCategory`, `isRetryable` и человекочитаемое описание; помечать нарушения бизнес-правил как `retriable: false` плюс понятным клиенту пояснением; поручать субагентам локальное восстановление от transient-ошибок и эскалировать только то, что нельзя разрешить локально (вместе с частичными результатами и описанием попыток); отличать *сбой доступа* от *валидного пустого результата*.

### 2.4 Интеграция сервера

Знание: область проекта (`.mcp.json`) vs пользователя (`~/.claude.json`); подстановка `${VAR}` для секретов; инструменты всех серверов обнаруживаются в момент подключения и доступны одновременно; ресурсы экспонируют каталоги контента, чтобы сократить разведывательные вызовы инструментов.

Навыки: настраивать общие серверы в `.mcp.json` с подстановкой переменных окружения; настраивать персональные серверы в пользовательской области; писать богатые описания инструментов, чтобы агент не скатывался к встроенным вроде `Grep`; выбирать community-серверы (Jira, GitHub, Postgres) вместо кастомных; экспонировать каталоги контента как ресурсы.

## MCP с высоты 30,000 футов

[Model Context Protocol](https://modelcontextprotocol.io/specification/latest) — открытый протокол на JSON-RPC 2.0, соединяющий LLM-хосты с провайдерами возможностей (серверами) через сессию с состоянием и согласованием возможностей. Серверы выставляют три примитива:

| Примитив | Управляется | Назначение | Пример |
| --- | --- | --- | --- |
| **Tool** | Моделью | Исполняемое действие с побочными эффектами | `create_issue`, `run_query`, `send_email` |
| **Resource** | Приложением / пользователем | Read-only контент, идентифицируемый URI | `jira://issues/PROJ-123`, `postgres://schema/orders` |
| **Prompt** | Пользователем | Заготовленные шаблоны, вызываемые по выбору пользователя (slash-команды, пункты меню) | `/code-review`, `/summarize-incident` |

Транспорты, определённые в спецификации: **stdio** (локальный подпроцесс), **Streamable HTTP** (`streamable-http` в спецификации, в конфигурации Claude Code алиас `http`) и устаревший транспорт **SSE**. Формат провода см. в [schema reference](https://modelcontextprotocol.io/specification/2025-11-25/schema).

## Структурированные ответы об ошибках

### Таксономия категорий ошибок

| Категория | Что означает | Типичный `isRetryable` | Что должен делать агент |
| --- | --- | --- | --- |
| `transient` | Таймаут, 5xx, rate limit, сброс соединения | `true` | Отступить и повторить локально внутри субагента |
| `validation` | Несовпадение схемы, неверный enum, отсутствующее поле | `false` | Переформулировать ввод; не повторять тот же вызов |
| `business` | Нарушение политики/лимита/состояния («refund > $500», «ticket already closed») | `false` | Показать пояснение для клиента; остановиться |
| `permission` | 401/403, отсутствующий scope, отказ RBAC | `false` | Эскалировать координатору или человеку |
| `not_found` | Валидный запрос, ноль совпадений | `false` (и `isError` спорен — см. ниже) | Трактовать как легитимный пустой результат, а не как сбой |

### Флаг `isError` — схема и пример

Официальная форма [`CallToolResult`](https://modelcontextprotocol.io/specification/2025-11-25/schema):

```ts
interface CallToolResult {
  content: ContentBlock[];
  structuredContent?: { [key: string]: unknown };
  isError?: boolean;  // default false
  _meta?: { [key: string]: unknown };
}
```

Спецификация явна: *ошибки, исходящие от самого инструмента*, должны сообщаться внутри результата с `isError: true`, **не** как протокольная ошибка JSON-RPC — протокольные ошибки невидимы модели, а LLM нужно увидеть ошибку, чтобы самокорректироваться. Протокольные ошибки зарезервированы для «tool not found» или «сервер не поддерживает вызовы инструментов».

Хорошо структурированный сбой инструмента выглядит так:

```json
{
  "isError": true,
  "content": [
    { "type": "text", "text": "Refund of $750 exceeds the $500 single-transaction policy. Ask the customer to split the refund or open a manager-approval ticket." }
  ],
  "structuredContent": {
    "errorCategory": "business",
    "isRetryable": false,
    "code": "REFUND_LIMIT_EXCEEDED",
    "limit": 500,
    "requested": 750,
    "customerMessage": "We can only process refunds up to $500 in one transaction."
  }
}
```

Сравните с анти-паттерном — `{ "isError": true, "content": [{ "type": "text", "text": "Operation failed" }] }` — который вынуждает агента либо повторять вслепую, либо сдаваться.

### Дерево решений retryable vs non-retryable

```
isError === true ?
├── No  → success path (or empty-result path if content is intentionally empty)
└── Yes
    ├── errorCategory === "transient" && isRetryable === true
    │     → backoff + retry locally (subagent), cap attempts (e.g. 3)
    ├── errorCategory === "validation"
    │     → re-plan with corrected input; do NOT replay the same call
    ├── errorCategory === "permission"
    │     → escalate to coordinator with the missing scope/role
    └── errorCategory === "business"
          → surface customerMessage; stop; do NOT retry
```

Структурированные метаданные делают это дерево исполнимым. Без `errorCategory` и `isRetryable` агенту приходится *гадать*, стоит ли сбой повторной попытки, и это главная причина «tool storm» — раскрученных циклов.

### Локальное восстановление у субагента vs эскалация к координатору

Субагент отвечает за локальное восстановление при transient-сбоях — повтор с backoff, переход на резервный endpoint, выдача устаревшего кэшированного чтения — и эскалирует наверх только тогда, когда класс ошибки не подлежит повтору *или* локальный бюджет повторов исчерпан. Когда он эскалирует, он возвращает структурированный конверт, включающий итоговые `errorCategory`/`isRetryable`, описание для логов координатора, **частичные результаты**, собранные до сбоя, и **что было предпринято** (какие endpoint-ы, сколько повторов), чтобы координатор не повторял работу.

Важно: *валидный пустой результат* (например, «no orders match this filter») — **не** ошибка. Смешение «я не смог выполнить запрос» с «запрос вернул ноль строк» приводит к тому, что координатор зря повторяет операцию или некорректно сообщает состояние пользователю.

## Конфигурация MCP-сервера в Claude Code

### Области видимости: local / project / user

[Документация Claude Code по MCP](https://docs.claude.com/en/docs/claude-code/mcp) определяет три области видимости:

| Область | Хранилище | Делится через git? | Для чего использовать |
| --- | --- | --- | --- |
| `local` (по умолчанию) | `~/.claude.json`, привязано к пути текущего проекта | Нет | Личные/экспериментальные серверы в одном проекте, секреты, которыми не хочется делиться |
| `project` | `.mcp.json` в корне репозитория | **Да** | Командные инструменты (Jira, внутренние API, бот деплоя команды) |
| `user` | `~/.claude.json`, общие для всех проектов | Нет | Личные утилиты, которые нужны везде (ваша scratch-БД, ваш сервер заметок) |

Добавление через CLI:

```bash
claude mcp add --scope project --transport http jira https://mcp.atlassian.com/v1/sse
claude mcp add --scope user    --transport stdio notes -- npx -y @me/notes-mcp
claude mcp add --scope local   --transport stdio scratch -- python ./scratch_server.py
```

Когда одно и то же имя сервера существует в нескольких областях, **побеждает local, затем project, затем user**, затем серверы плагинов, затем коннекторы `claude.ai`. Эта приоритетность позволяет разработчику локально переопределить общекомандную конфигурацию, не редактируя `.mcp.json`.

### Схема `.mcp.json` (с примером)

`.mcp.json` уровня проекта лежит в корне репозитория и коммитится. Минимальная схема:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp",
      "headers": {
        "Authorization": "Bearer ${GITHUB_PERSONAL_ACCESS_TOKEN}"
      }
    },
    "postgres-readonly": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL_RO}"],
      "env": {
        "PGSSLMODE": "require"
      }
    },
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```

У каждой записи `type` — один из `stdio`, `http` (алиас `streamable-http`) или `sse`. Для `stdio` указывайте `command` плюс опциональные `args` и `env`. Для `http`/`sse` указывайте `url` и опциональные `headers`. Серверы уровня проекта при первой загрузке сессии запрашивают подтверждение пользователя (сбрасывается через `claude mcp reset-project-choices`).

### Подстановка переменных окружения

Claude Code расширяет внутри `.mcp.json` две формы:

- `${VAR}` — значение `VAR`; **парсинг падает**, если переменная не задана
- `${VAR:-default}` — значение `VAR`, если задано, иначе `default`

Подстановка работает в `command`, `args`, `env`, `url` и `headers`. Рекомендуемый паттерн: коммитить `.mcp.json` с плейсхолдерами `${GITHUB_TOKEN}` и держать реальные токены в shell каждого разработчика, в CI-раннере или в 1Password CLI. Тонкость: `${CLAUDE_PROJECT_DIR}` устанавливается в окружении *сервера*, а не самого Claude Code — для использования на верхнем уровне `command` или `args` обычно нужен дефолт `${CLAUDE_PROJECT_DIR:-.}`.

### Проверка подключения (`/mcp`)

Внутри Claude Code `/mcp` выводит список каждого подключённого сервера, количество объявленных им инструментов/ресурсов/промптов, состояние аутентификации и любой текущий backoff на переподключение. HTTP/SSE-серверы автоматически переподключаются до пяти раз с экспоненциальным backoff; stdio-серверы, будучи локальными подпроцессами, — нет. `/mcp` также то место, где запускают OAuth-флоу для удалённых серверов, возвращающих `401`/`403` с заголовком `WWW-Authenticate`.

## MCP-инструменты vs MCP-ресурсы

### Когда моделировать как инструмент

Инструменты **управляются моделью** и могут иметь побочные эффекты. Используйте инструмент, когда агент должен *решать*, делать ли что-то:

- `create_jira_issue`, `update_pr_status`, `run_sql(query)`, `send_slack_message`
- Всё, что меняет состояние, вызывает внешний API или требует аргументов, о которых модель должна рассуждать

### Когда моделировать как ресурс

Ресурсы **управляются приложением**, доступны только для чтения, идентифицируются URI и идеально кэшируются. Используйте ресурс, когда агент выигрывает от *пассивного контекста*, не запрашивая его:

- Каталог всех открытых тикетов Jira в текущем спринте → `jira://sprints/current/issues`
- Схема Postgres для базы заказов → `postgres://schema/public/orders`
- Содержимое runbook или design-doc → `confluence://pages/12345`

Поскольку ресурсы адресуются URI и не меняют состояние, они дешевле в повторах и дружелюбнее к кэшированию, чем те же данные, обёрнутые в вызов инструмента.

### Примеры (каталог тикетов, интроспекция схемы)

Паттерн руководства «content catalog»: вместо того чтобы вынуждать агента вызывать `list_issues`, а затем `get_issue` для каждого совпадения (исследовательский N+1), сервер выставляет ресурс `issues://summary`, который хост может прикрепить автоматически. Агент видит каталог в контексте и сразу делает релевантный вызов `get_issue`. Та же идея применима к интроспекции схемы — выставление `db://schema` как ресурса превращает вопрос «какие столбцы у `orders`?» в ответ без единого вызова инструмента.

## Выбор community vs кастомных серверов

### Когда писать свой

Создавайте кастомный MCP-сервер только когда интеграция **специфична для команды** и нет поддерживаемого community-варианта: внутренний микросервис, ваш конвейер деплоя, самописная CRM. Стоимость кастомного сервера — не сам код, а постоянное обслуживание: дрейф схемы, обновление аутентификации, апгрейды транспорта, security-ревью.

### Популярные серверы, которые стоит знать

Репозиторий [`modelcontextprotocol/servers`](https://github.com/modelcontextprotocol/servers) содержит эталонные реализации и ссылки на официальный [MCP Registry](https://registry.modelcontextprotocol.io/). Имена, которые стоит запомнить: **Filesystem** (файловые операции в песочнице), **Fetch** (URL → текст для LLM), **Git**, **Memory** (хранилище в виде графа знаний), **Sequential Thinking**, плюс хостинг от вендоров — **GitHub** (`api.githubcopilot.com/mcp`), **Postgres**, **Slack**, **Sentry**, **Notion**, **Atlassian/Jira**, **Stripe**, **PayPal**, **HubSpot**, **Asana** и **Playwright**. Community-каталоги ([mcpforge.org](https://www.mcpforge.org/directory), [mcpindex.net](https://mcpindex.net/), [mcpdir.dev](https://mcpdir.dev/)) перечисляют тысячи других, но официальный реестр — источник истины по происхождению.

Последний навык, который выделяет руководство: **усиливать описания MCP-инструментов**. Если ваш Jira MCP-инструмент описан как «search Jira», а встроенный `Grep` — описан подробно и живо, модель потянется за `Grep` по локальному кэшу. Описания инструментов — единственный сигнал модели для выбора инструмента; пишите их как docstring функции, которую модель ни разу не видела.

## Экзаменационные акценты

- `isError: true` живёт внутри `CallToolResult`, **а не** как ошибка JSON-RPC — модель обязана *увидеть* сбой, чтобы восстановиться.
- Всегда включайте `errorCategory` и `isRetryable` в `structuredContent`; «Operation failed» — канонический неправильный ответ.
- Отличайте сбой доступа от пустого результата; `isError: false` с пустым `content` — валидный, обычный случай.
- `.mcp.json` — это область проекта и коммитится; `~/.claude.json` — local/user-область и приватный.
- Подстановка `${VAR}` и `${VAR:-default}` — для удержания секретов вне git, а не для рантайм-логики конфигурации.
- Инструменты = управляются моделью, с побочными эффектами; ресурсы = управляются приложением, только для чтения, адресуются URI.
- Предпочитайте community-серверы; кастомные существуют только для команд-специфичных процессов.
- Субагенты восстанавливаются от transient-ошибок локально; только неустранимые ошибки (с частичными результатами и журналом попыток) поднимаются к координатору.

## References

- MCP Specification (latest): https://modelcontextprotocol.io/specification/latest
- MCP Schema (2025-11-25): https://modelcontextprotocol.io/specification/2025-11-25/schema
- MCP Server Primitives Overview: https://modelcontextprotocol.io/specification/2025-11-25/server
- Claude Code — Connect to tools via MCP: https://docs.claude.com/en/docs/claude-code/mcp
- Claude Code — MCP installation scopes: https://docs.claude.com/en/docs/claude-code/mcp#mcp-installation-scopes
- Claude Code — Environment variable expansion in `.mcp.json`: https://docs.claude.com/en/docs/claude-code/mcp#environment-variable-expansion-in-mcp-json
- Reference servers: https://github.com/modelcontextprotocol/servers
- MCP Registry: https://registry.modelcontextprotocol.io/
