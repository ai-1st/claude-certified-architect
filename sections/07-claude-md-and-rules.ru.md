---
title: "Раздел 7 — Иерархия конфигурации CLAUDE.md и правила по путям"
linkTitle: "7. CLAUDE.md & Rules"
weight: 7
description: "Domains 3.1, 3.3 — иерархия user/project/directory CLAUDE.md, паттерны @import и соглашения с glob-шаблонами в .claude/rules/."
---

## Что покрывает этот раздел

Как Claude Code находит, сливает и применяет постоянные инструкции по областям видимости **управляемая политика / пользователь / проект / подкаталог**, как держать большие наборы инструкций модульными через `@import` и `.claude/rules/` и как использовать YAML-список `paths` с glob-шаблонами, чтобы соглашения загружались только когда Claude трогает совпадающие файлы. Экзамен проверяет два конкретных режима отказа: класть командные стандарты в пользовательскую область (и тогда новый коллега их не увидит) и использовать `CLAUDE.md` в подкаталоге для сквозных соглашений (тестовые файлы, разбросанные по дереву) вместо правила по путям. Sample Question 6 — прямой тест второго паттерна.

## Исходный материал (из официального руководства)

**3.1 — Иерархия и модульная организация.** Знать иерархию `CLAUDE.md` для пользователя / проекта / подкаталога; знать, что `~/.claude/CLAUDE.md` — для каждого пользователя (никогда не делится через VCS); знать `@import` для модульного контента; знать `.claude/rules/` как альтернативу монолитному файлу; диагностировать проблемы иерархии через `/memory`.

**3.3 — Правила по путям.** Знать, что `.claude/rules/*.md` используют YAML-frontmatter `paths` с glob-шаблонами, загружаются только когда Claude трогает совпадающие файлы и превосходят `CLAUDE.md` в подкаталоге для соглашений, пересекающих дерево (например, `**/*.test.tsx`).

## Иерархия CLAUDE.md

### Локации и приоритет

Claude Code загружает несколько файлов `CLAUDE.md` в определённом порядке. Ключевая ментальная модель: **файлы конкатенируются, а не переопределяются** — каждый применимый файл оказывается в контексте сессии. Порядок имеет значение, потому что инструкции, расположенные ближе к рабочему каталогу, появляются в контексте *позже*, что даёт им эффективный приоритет.

| Область | Путь | Применение | Делится через VCS? |
| --- | --- | --- | --- |
| Управляемая политика | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`, Linux/WSL `/etc/claude-code/CLAUDE.md`, Windows `C:\Program Files\ClaudeCode\CLAUDE.md` | Правила уровня организации, развёрнутые ИТ; нельзя исключить | Нет — развёртывается через MDM / Group Policy / Ansible |
| Пользователь | `~/.claude/CLAUDE.md` | Личные предпочтения во всех проектах | Нет — никогда не коммитится |
| Проект | `./CLAUDE.md` *или* `./.claude/CLAUDE.md` | Командные соглашения, команды сборки, архитектура | Да — коммитится |
| Локальный проект | `./CLAUDE.local.md` | Личные песочные URL, тестовые данные | Нет — добавить в `.gitignore` |
| Подкаталог | `path/to/dir/CLAUDE.md` | Инструкции для одной части кодовой базы | Да, если закоммичено |
| Пользовательские правила | `~/.claude/rules/*.md` | Личные правила во всех проектах | Нет |
| Проектные правила | `.claude/rules/*.md` | Командные тематические или path-scoped правила | Да |

Источники: официальная документация по памяти Claude Code на [docs.anthropic.com/en/docs/claude-code/claude-md](https://docs.anthropic.com/en/docs/claude-code/claude-md) и зеркало на [code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory).

### Как файлы сливаются на старте сессии

Claude поднимается вверх по дереву каталогов от текущего рабочего каталога, подбирая каждый встретившийся `CLAUDE.md` и `CLAUDE.local.md`. Контент упорядочен **от корня файловой системы к рабочему каталогу**; внутри одного каталога `CLAUDE.local.md` дописывается после `CLAUDE.md`. Запуск в `foo/bar/` даёт примерно следующее:

```
1. managed-policy CLAUDE.md (if present)
2. ~/.claude/CLAUDE.md + ~/.claude/rules/*
3. /foo/CLAUDE.md
4. /foo/bar/CLAUDE.md
5. /foo/bar/CLAUDE.local.md
6. .claude/rules/*.md  (rules without `paths` load here)
```

Файлы `CLAUDE.md` в подкаталогах **ниже** вашего рабочего каталога **не** загружаются при старте — они загружаются лениво, когда Claude читает файл в этом подкаталоге. После `/compact` корневой проектный `CLAUDE.md` повторно подгружается с диска, но вложенные файлы `CLAUDE.md` перезагружаются только когда Claude в следующий раз затронет их поддерево.

### Диагностика через `/memory`

`/memory` — самый важный инструмент диагностики для этой области. Он выводит каждый `CLAUDE.md`, `CLAUDE.local.md` и файл правил, загруженный в текущей сессии, позволяет переключать авто-память и открывать любой файл в редакторе. Если ожидаемого файла нет в списке, Claude его не видит.

Стандартный порядок действий, когда «Claude не следует моему CLAUDE.md»:

1. Запустите `/memory`. Файл в списке?
2. Если нет, лежит ли он в одной из загружаемых локаций? `~/.claude/CLAUDE.md` применяется только к *вашим* сессиям; коллега его не увидит.
3. Если в списке, конкретны и непротиворечивы ли инструкции? Два правила с противоречивыми указаниями позволяют Claude выбирать произвольно.
4. Для правил по путям используйте хук `InstructionsLoaded`, чтобы логировать, когда и почему загружается каждый файл.

Каноническая ловушка иерархии: разработчик кладёт командные стандарты в `~/.claude/CLAUDE.md`, у него всё прекрасно работает, а потом новый сотрудник спрашивает: «почему Claude всё время использует неправильный test framework?». Исправление: перенесите эти правила в `./CLAUDE.md` или `./.claude/CLAUDE.md` и закоммитьте.

## Модульная организация через `@import`

### Синтаксис и ограничения

Внутри любого `CLAUDE.md` токен `@path/to/file` подставляет содержимое указанного файла на старте сессии.

- Относительные пути разрешаются **относительно файла, в котором написан import**, а не от рабочего каталога. Абсолютные пути и пути с `~/...` тоже работают.
- Импорты вкладываются на глубину до **5 уровней**: `CLAUDE.md` → `@docs/guide.md` → `@docs/sub/detail.md` → … (максимум пять уровней).
- Когда Claude Code в первый раз видит внешние импорты в проекте, появляется диалог одобрения со списком файлов. Откажетесь один раз — они останутся отключены.
- Импортирование **не** экономит токены — каждый импортируемый файл загружается целиком на старте. Импорты нужны для *организации*, а не для сокращения контекста. Для ленивой загрузки используйте `.claude/rules/` с `paths`.

### Пример: импорты по пакетам в монорепозитории

```markdown
# Monorepo CLAUDE.md (root)
This is a pnpm workspace. Run `pnpm install` at the root.

## Package-specific standards
- API service: @services/api/STANDARDS.md
- Web frontend: @apps/web/STANDARDS.md
- Shared types: @packages/types/STANDARDS.md

See @README.md for project overview and @package.json for scripts.

# Individual Preferences (cross-worktree personal notes)
- @~/.claude/my-project-instructions.md
```

Каждый мейнтейнер пакета владеет связанным STANDARDS-файлом, не заставляя остальные команды его читать. Импорт `~/.claude/...` — канонический обходной путь для того факта, что закоммиченный в `.gitignore` `CLAUDE.local.md` существует только в том worktree, где вы его создали.

## Тематические файлы в `.claude/rules/`

`.claude/rules/` — **документированная фича Claude Code**, а не Cursor-only-соглашение. Официальная документация по памяти описывает её в разделе «Organize rules with `.claude/rules/`». Все `.md`-файлы в этом каталоге обнаруживаются рекурсивно и существуют в двух разновидностях:

1. **Безусловные правила** — без frontmatter `paths`. Загружаются на старте с тем же приоритетом, что и `.claude/CLAUDE.md`.
2. **Path-scoped правила** — frontmatter `paths` с одним или несколькими glob-шаблонами. Загружаются лениво, когда Claude читает файл, совпадающий с одним из шаблонов.

Пользовательские правила в `~/.claude/rules/` применяются ко всем проектам на вашей машине и загружаются *до* проектных, поэтому проектные побеждают при конфликте.

### Схема frontmatter (YAML)

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "src/api/**/*.tsx"
---

# API development rules
- All API endpoints must include input validation.
- Use the standard error response format.
```

Заметки из спецификации и известной YAML-баги ([anthropics/claude-code#13905](https://github.com/anthropics/claude-code/issues/13905)): `paths` должен быть YAML-списком; шаблоны, начинающиеся с `*` или `{`, **должны быть в кавычках**, чтобы быть валидным YAML; отсутствие `paths` делает правило загружаемым безусловно.

### Семантика glob-шаблонов в `paths`

Стандартные glob-операторы — `*` совпадает внутри одного сегмента пути, `**` рекурсивно проходит по каталогам:

| Шаблон | Что совпадает |
| --- | --- |
| `*.md` | Markdown-файлы только в корне проекта |
| `**/*.ts` | TypeScript-файлы в любом каталоге |
| `**/*.test.tsx` | React-тесты в любом месте дерева |
| `src/**/*` | Все файлы под `src/` |
| `src/components/*.tsx` | Только прямые дочерние |
| `src/**/*.{ts,tsx}` | TypeScript и TSX под `src/` (раскрытие фигурных скобок) |
| `terraform/**/*` | Всё под `terraform/` |

Канонический экзаменационный пример `**/*.test.tsx` работает — это учебный случай для правила по путям.

### Пример: сквозное правило для тестов (Sample Question 6)

`.claude/rules/tests.md` покрывает тестовые файлы, лежащие рядом с исходниками (`Button.test.tsx` рядом с `Button.tsx`) в любом месте репозитория:

```markdown
---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "**/*.spec.ts"
---

# Test conventions
- Use `describe` / `it` (Jest style), never `test()`.
- One assertion per `it` block; use `it.each` for tables.
- Mock with `vi.mock`; never mutate `globalThis`.
```

Поменяйте `paths` на `["terraform/**/*", "**/*.tf"]` — и у вас тот же трюк для инфраструктурных файлов в любом месте дерева.

## CLAUDE.md vs CLAUDE.md в подкаталоге vs правила по путям — когда что

| Что нужно | Лучший выбор |
| --- | --- |
| Универсальный командный стандарт (сборка, этикет репозитория) | Корневой `CLAUDE.md` или `.claude/CLAUDE.md` |
| Личные предпочтения во всех проектах | `~/.claude/CLAUDE.md` или `~/.claude/rules/` |
| Личные предпочтения в одном репозитории | `CLAUDE.local.md` (в .gitignore) |
| Соглашение в рамках одной папки | `CLAUDE.md` в подкаталоге `path/to/CLAUDE.md` (ленивая загрузка) |
| Соглашение, размазанное по многим папкам (тесты, `.tf`, `.sql`) | `.claude/rules/*.md` с glob в `paths` |
| Большие монолитные инструкции | Разбить на `.claude/rules/` или `@import` |
| Должно выполняться в конкретной точке жизненного цикла | **Хук**, а не `CLAUDE.md` |
| Политика уровня организации, от которой пользователи не могут отказаться | Управляемая `CLAUDE.md` + управляемый `settings.json` |

Решающая эвристика: **соглашение следует за *типом/шаблоном* файла или за его *расположением*?** Шаблон → правило по путям с glob. Расположение → `CLAUDE.md` в подкаталоге. Именно это и проверяет Question 6.

## Эталонная раскладка

```
your-monorepo/
├── CLAUDE.md                          # High-level architecture, build commands
├── CLAUDE.local.md                    # Gitignored personal notes
├── .claude/
│   └── rules/
│       ├── tests.md                   # paths: ["**/*.test.{ts,tsx}"]
│       ├── terraform.md               # paths: ["terraform/**/*", "**/*.tf"]
│       ├── api-conventions.md         # paths: ["src/api/**/*.ts"]
│       └── security.md                # No paths -> loads every session
├── services/billing/
│   ├── CLAUDE.md                      # Lazy-loaded inside services/billing/
│   └── STANDARDS.md                   # @services/billing/STANDARDS.md from root
└── apps/web/CLAUDE.md                 # Lazy-loaded for the web app subtree

~/.claude/
├── CLAUDE.md                          # Personal preferences across projects
└── rules/preferences.md               # Personal coding preferences
```

Каждый механизм находится на своём месте: маленький корневой файл, модульные тематические правила, ограничение по путям для сквозных соглашений, `CLAUDE.md` в подкаталоге для действительно привязанных к локации правил и `~/.claude/` для личных предпочтений.

## Анти-паттерны и экзаменационные ловушки

- **Командные стандарты в `~/.claude/CLAUDE.md`.** Они следуют за вами, а не за репозиторием; новые коллеги не получают ничего. Перенесите общие соглашения в `./CLAUDE.md` или `./.claude/CLAUDE.md` и закоммитьте.
- **Один гигантский `CLAUDE.md` на 2000 строк.** Всё в `CLAUDE.md` попадает в контекстное окно целиком на старте; Anthropic рекомендует держать ниже ~200 строк на файл. Разбейте на тематические файлы в `.claude/rules/` и предпочитайте правила по путям, чтобы большая часть контента не попадала в контекст, пока не понадобится.
- **`CLAUDE.md` в подкаталоге для сквозного соглашения.** Тестовые файлы рядом с исходниками в десятках папок невозможно покрыть директорными файлами без абсурдного дублирования. Используйте единый `.claude/rules/tests.md` с `paths: ["**/*.test.tsx"]`. Это и есть Sample Question 6.
- **Трактовка `@import` как оптимизации, экономящей токены.** Это не так — импорты загружаются целиком на старте. Токены экономит `.claude/rules/` с `paths` (ленивая загрузка).
- **Машинно-принудительные правила в `CLAUDE.md`.** `CLAUDE.md` — это совещательный контекст, а не принуждение. Всё, что *должно* произойти (линт перед коммитом, блокировка записи в `secrets/`), живёт в хуке или `permissions.deny`.
- **Предположение, что Claude читает `AGENTS.md`.** Нативно — нет. Если ваш репозиторий использует `AGENTS.md`, поставьте `@AGENTS.md` в начало `CLAUDE.md` (рекомендуется; переносимо) или сделайте `ln -s AGENTS.md CLAUDE.md` (Unix; на Windows нужны права администратора или Developer Mode). `/init` в репозитории с `AGENTS.md` читает его и включает релевантные части.
- **Конфликтующие правила между файлами.** Claude может выбрать произвольно. Периодически проводите аудит и используйте `claudeMdExcludes` в `.claude/settings.local.json`, чтобы в монорепозитории пропускать файлы чужих команд.

## Экзаменационные акценты

- Знайте все четыре области плюс управляемую политику и какие из них делятся через VCS. `~/.claude/CLAUDE.md` — ловушка: per-user, никогда не коммитится.
- Файлы **конкатенируются** от корня файловой системы к рабочему каталогу; `CLAUDE.local.md` дописывается после `CLAUDE.md` в том же каталоге.
- `@import` допускает относительные, абсолютные и пути с префиксом `~`; вложенность максимум **5 уровней**; токены **не** экономит.
- `.claude/rules/*.md` с frontmatter `paths` — **единственный** механизм, обеспечивающий ленивую загрузку соглашений по glob. Без `paths` правило загружается в каждой сессии.
- Соглашение следует за *типом/шаблоном* файла → правило по путям. Соглашение следует за *расположением* файла → `CLAUDE.md` в подкаталоге.
- `/memory` отвечает на вопрос «что именно загрузил Claude?»; хук `InstructionsLoaded` добавляет детализацию по правилам путей. Целевой ориентир — менее **200 строк** на `CLAUDE.md`.

## References

- Anthropic, *How Claude remembers your project* — [docs.anthropic.com/en/docs/claude-code/claude-md](https://docs.anthropic.com/en/docs/claude-code/claude-md) (mirror: [code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory))
- Anthropic, *Claude Code best practices* — [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)
- Anthropic, slash commands reference — [code.claude.com/docs/en/commands.md](https://code.claude.com/docs/en/commands.md)
- `anthropics/claude-cookbooks` CLAUDE.md example — [github.com/anthropics/claude-cookbooks/blob/main/CLAUDE.md](https://github.com/anthropics/claude-cookbooks/blob/main/CLAUDE.md)
- `paths` frontmatter YAML quoting bug — [github.com/anthropics/claude-code/issues/13905](https://github.com/anthropics/claude-code/issues/13905)
- Memory vs. Settings precedence inconsistency — [github.com/anthropics/claude-code/issues/18964](https://github.com/anthropics/claude-code/issues/18964)
- AGENTS.md open standard — [agents.md](https://agents.md/); v1.1 proposal [github.com/agentsmd/agents.md/issues/135](https://github.com/agentsmd/agents.md/issues/135)
