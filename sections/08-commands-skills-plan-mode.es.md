---
title: "Sección 8 — Slash commands personalizados, skills, plan mode y refinamiento iterativo"
linkTitle: "8. Comandos, skills y plan mode"
weight: 8
description: "Dominios 3.2, 3.4, 3.5 — .claude/commands/, .claude/skills/ con context: fork y allowed-tools, y plan mode frente a ejecución directa."
---

## Qué cubre esta sección

Extender Claude Code con **slash commands personalizados** y **agent skills**, elegir **plan mode** frente a ejecución directa (y cómo encaja el subagente **Explore**), y técnicas de **refinamiento iterativo**. Sample Question 4 evalúa dónde vive un comando `/review` compartido; Sample Question 5 evalúa elegir plan mode para una reestructuración de monolito a microservicios.

## Material fuente (de la guía oficial)

**3.2 Slash commands y skills.** Comandos de proyecto en `.claude/commands/` (compartidos vía VCS) frente a comandos de usuario en `~/.claude/commands/` (personales). Skills en `.claude/skills/<name>/SKILL.md` con frontmatter YAML que soporta `context: fork`, `allowed-tools`, `argument-hint`. `context: fork` ejecuta la skill en un contexto de subagente aislado para que la salida verbosa no contamine la conversación principal. Personalización personal: variantes en `~/.claude/skills/` con nombre distinto evitan afectar a compañeros. Las skills son bajo demanda y específicas de tarea; `CLAUDE.md` son estándares universales siempre cargados.

**3.4 Plan mode frente a ejecución directa.** Plan mode es para tareas complejas con cambios de gran escala, múltiples enfoques válidos, decisiones arquitectónicas, modificaciones multiarchivo. La ejecución directa es para cambios simples y bien acotados (una validación, un fix de un archivo con stack trace claro). Plan mode habilita exploración segura antes de comprometerse. El subagente Explore aísla descubrimiento verboso y devuelve resúmenes. Patrón común: planificar para investigar, luego ejecución directa para aplicar.

**3.5 Refinamiento iterativo.** Ejemplos concretos input/output superan a la prosa. Iteración guiada por pruebas: primero tests, luego iterar sobre fallos. Patrón de entrevista: hacer que Claude formule preguntas aclaratorias en dominios desconocidos. Casos de prueba específicos para edge cases (por ejemplo nulls en migraciones). Agrupar problemas interactuantes en un mensaje; secuenciar problemas independientes.

## Las cuatro primitivas de personalización

Claude Code expone cuatro mecanismos que se conectan en distintas partes del bucle de agente. Saber cuál elegir es un objetivo casi seguro del examen.

| Primitiva | Disparador | Aislamiento | Cuándo usar |
|---|---|---|---|
| **Slash command** (`.claude/commands/foo.md`) | El usuario escribe `/foo` | Contexto principal | Workflow interactivo reutilizable: `/review`, `/commit`, `/deploy-staging`. |
| **Skill** (`.claude/skills/foo/SKILL.md`) | El usuario escribe `/foo` *o* Claude la auto-invoca cuando la descripción coincide | Contexto principal; aislada con `context: fork` | Conocimiento o procedimiento específico de tarea con archivos de apoyo; permite que Claude decida *cuándo* aplicarlo. |
| **Subagent** (`.claude/agents/foo.md`) | Claude o el usuario delega una tarea | Siempre aislado; herramientas y modelo propios | Investigación verbosa, trabajo paralelo, especialistas como `code-reviewer` o `Explore`. |
| **Hook** (`hooks.json`) | Se dispara un evento de ciclo de vida (PreToolUse, PostToolUse, UserPromptSubmit, etc.) | Script shell, salida devuelta | Efectos secundarios deterministas: auto-lint, bloquear ediciones en rutas protegidas, añadir trailers de commit. |

Anthropic **fusionó custom slash commands en skills** a finales de 2025: un archivo en `.claude/commands/deploy.md` y una skill en `.claude/skills/deploy/SKILL.md` crean ambos `/deploy` y se comportan igual. Las skills son la forma recomendada porque añaden archivos de apoyo, control de invocación, inyección dinámica de contexto y ejecución en subagente ([slash-commands docs](https://docs.anthropic.com/en/docs/claude-code/slash-commands)).

## Slash commands personalizados

### Ámbito proyecto frente a usuario

- **`.claude/commands/`** — ámbito proyecto, version-controlled, workflows de todo el equipo (`/review`, `/security-review`, `/migrate-route`).
- **`~/.claude/commands/`** — ámbito usuario, shortcuts solo personales (`/scratch`, `/jira-link`).

Sample Question 4 depende de esto: un `/review` que "should be available to every developer when they clone or pull the repository" va en **`.claude/commands/`**, no en `~/.claude/commands/`, no en `CLAUDE.md`, no en un `.claude/config.json` ficticio.

### Layout de archivo y frontmatter

Un comando es un archivo markdown con frontmatter YAML opcional:

```markdown
---
description: Run the team code-review checklist on the current diff
argument-hint: [path-or-PR]
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git log:*)
model: sonnet
---

Review the changes in $ARGUMENTS using our checklist:

## Diff
!`git diff HEAD`

## Style guide
@docs/CODE_REVIEW.md

For each finding, output: severity, file:line, problem, suggested fix.
```

Campos clave de frontmatter ([referencia](https://docs.anthropic.com/en/docs/claude-code/slash-commands#frontmatter-reference)):

| Campo | Propósito |
|---|---|
| `description` | Resumen mostrado en `/help`; también impulsa auto-invocación. Mantener bajo ~60 caracteres. |
| `argument-hint` | Hint de autocompletado como `[issue-number]` o `[filename] [format]`. |
| `allowed-tools` | Herramientas utilizables sin aprobación por llamada. Acepta patrones Bash con glob como `Bash(git:*)`. |
| `model` | Sobrescribe el modelo para este comando (`haiku`/`sonnet`/`opus`/`inherit`). |
| `disable-model-invocation` | `true` fuerza solo disparo humano; para workflows con efectos secundarios como `/deploy`. |

Todos los campos son opcionales.

### Manejo de argumentos, ejecución bash, referencias de archivo

Tres mecanismos de sustitución viven dentro del cuerpo markdown:

- **`$ARGUMENTS`** — cadena completa de argumentos. `$ARGUMENTS[N]` indexa por posición; `$N` es la abreviatura.
- **`` !`cmd` ``** — ejecuta un comando bash al cargar e inserta su stdout (por ejemplo `` !`git diff HEAD` ``).
- **`@path`** — inserta el contenido de un archivo; `@$0` inserta un archivo cuya ruta es el primer argumento.

> Nota de examen: docs antiguos usaban `$1` para el *primer* argumento. Los docs actuales renumeran a base 0 (`$0`, `$1`, `$2`). Reconoce ambos; prefiere `$ARGUMENTS` cuando el orden no importa.

### Ejemplo: `/review`

Un `/review` de ámbito proyecto para Sample Question 4: commitea esto en `.claude/commands/review.md` y cada compañero recibe `/review` en el siguiente pull:

```markdown
---
description: Run our PR review checklist on a diff or PR
argument-hint: [pr-number-or-path]
allowed-tools: Read, Grep, Glob, Bash(gh pr view:*), Bash(git diff:*)
---

## Diff
!`git diff origin/main...HEAD`

## Checklist
@.claude/docs/review-checklist.md

Apply each checklist item. For every finding, report severity
(block/major/nit), file:line, the issue, and a concrete fix.
End with a one-paragraph overall verdict.
```

## Agent skills

### Layout de archivo

```
.claude/skills/pr-summary/
  SKILL.md          # required: frontmatter + instructions
  checklist.md      # optional supporting files
  example-good.md
```

Las skills se descubren en cuatro niveles: enterprise, user (`~/.claude/skills/`), project (`.claude/skills/`) y plugin. Enterprise sobreescribe personal, personal sobreescribe project; las plugin skills viven en un namespace `plugin-name:skill-name` ([skills docs](https://docs.anthropic.com/en/docs/claude-code/skills)).

### Referencia de frontmatter

Además de los campos compartidos con comandos, las skills añaden:

| Campo | Propósito |
|---|---|
| `name` | Identificador de skill (por defecto el nombre del directorio). |
| `description` | Qué hace la skill y cuándo usarla; impulsa auto-invocación. `description` + `when_to_use` se limitan a 1,536 chars combinados. |
| `when_to_use` | Frases disparadoras extra añadidas a `description`. |
| `arguments` | Args posicionales nombrados, por ejemplo `arguments: [issue, branch]` → `$issue`, `$branch`. |
| `user-invocable` | `false` la oculta del menú `/`; Claude aún puede usarla como conocimiento de fondo. |
| `disable-model-invocation` | `true` bloquea auto-invocación; combinar con skills con efectos secundarios (`/deploy`). |
| `context` | `fork` para ejecutar dentro de un subagente aislado. |
| `agent` | Tipo de subagente para skills forked (`Explore`, `Plan`, custom). |
| `hooks` | Hooks de ciclo de vida acotados a esta skill. |
| `paths` | Patrones glob que limitan auto-invocación a archivos coincidentes. |

### `context: fork`: qué hace y cuándo usarlo

Por defecto, una skill se ejecuta inline, así que su salida — listados de archivos, pensamientos intermedios, resultados de búsqueda — consume la ventana de contexto del padre. `context: fork` **despacha la skill a un subagente fresco**: el contenido de la skill se convierte en el prompt del subagente, el subagente corre con sus propias herramientas y permisos, y solo su resumen final vuelve al padre.

```yaml
---
name: pr-summary
description: Summarize a PR's changes and risk surface
context: fork
agent: Explore
allowed-tools: Bash(gh *), Read, Grep, Glob
---

Summarize PR $ARGUMENTS. Read the diff with `gh pr diff $ARGUMENTS`,
group changes by subsystem, call out risky touches (auth, billing,
migrations), and end with a 3-bullet reviewer checklist.
```

Usa fork cuando la skill es **verbosa** (análisis de codebase, resumen de diff grande), **exploratoria** (brainstorming de alternativas) o **independiente** (devuelve un resumen, no artefactos raw que el padre edita). No hagas fork de skills puras de "usa estas convenciones": el subagente recibe guías pero ninguna tarea accionable y devuelve vacío.

### Patrón de variante personal

Para personalizar una skill compartida sin afectar a compañeros, cópiala con un nombre distinto en `~/.claude/skills/` (por ejemplo `pr-summary-detailed/`). Esto evita el antipatrón "edité la skill compartida y ahora todos reciben mi variante." El mismo truco de renombrado funciona para slash commands en `~/.claude/commands/`.

### Matriz de decisión skills frente a CLAUDE.md

| Necesidad | Elegir |
|---|---|
| Estándares universales activos en cada sesión (tech stack, estilo) | `CLAUDE.md` |
| Procedimiento bajo demanda específico de tarea con archivos de apoyo | Skill |
| Workflow interactivo sin auto-invocación | Skill con `disable-model-invocation: true` |
| Convención por tipo de archivo limitada por globs de ruta | Regla con alcance de ruta (`.claude/rules/`) o skill con `paths:` |
| Convención que debe ejecutarse como código, no consejo | Hook |
| Descubrimiento verboso que inundaría el contexto | Skill con `context: fork`, o delegar a Explore |

`CLAUDE.md` es **contexto siempre activo**: barato de cargar, caro a escala. Las skills son **contexto bajo demanda**: más pesadas por uso pero gratis cuando no se usan.

## Plan mode frente a ejecución directa

### Cómo entrar en plan mode

Plan mode es uno de seis permission modes; Claude lee archivos y ejecuta comandos de solo lectura pero no puede editar código fuente ([permission-modes docs](https://docs.anthropic.com/en/docs/claude-code/permission-modes)). Cuatro puntos de entrada:

1. **Ciclar en sesión** con `Shift+Tab`: `default → acceptEdits → plan`.
2. **Al inicio**: `claude --permission-mode plan` (también funciona con `-p` para headless).
3. **Default de proyecto**: `permissions.defaultMode: "plan"` en `.claude/settings.json`.
4. **Por prompt**: prefijar un mensaje con `/plan`.

Cuando el plan está listo, Claude ofrece (a) aprobar y cambiar a auto, (b) aprobar y revisar cada edición, o (c) seguir planificando. `Ctrl+G` abre el plan en tu editor; `Shift+Tab` sale sin aprobar.

### Cuándo compensa plan mode

Usa plan mode cuando se cumpla cualquiera:

- El cambio cruza **muchos archivos o límites arquitectónicos** (reestructura a microservicios, migración de librería que afecta 45+ archivos).
- **Múltiples enfoques válidos** con compromisos reales (Redis frente a in-memory frente a file cache; webhook frente a polling).
- **El alcance es desconocido**: la pregunta es "¿qué tan grande?" no "implementa esto."
- **Alto radio de impacto**: migraciones de schema, refactors de auth, contratos entre equipos.

Sample Question 5 es el primer bullet: reestructurar un monolito a microservicios cruza docenas de archivos y decisiones de límites; plan mode de manual. La respuesta incorrecta ("dejar que la implementación revele los límites") pierde frente al retrabajo en cuanto aparece una dependencia oculta.

### Cuándo la ejecución directa es correcta

- Bug de un solo archivo con stack trace claro y fix obvio.
- Una validación, una línea de log, un feature flag.
- Refactor mecánico con objetivo conocido (renombrar en N archivos: usa `acceptEdits`).
- Ya tienes un plan de un turno previo.

Plan mode tiene overhead: tokens extra de exploración para Claude, revisión del plan para el desarrollador. No lo pagues en cambios de tres líneas.

### El subagente Explore para descubrimiento verboso

Incluso fuera de plan mode, delega la fase de descubrimiento al subagente integrado **Explore**: solo lectura, respaldado por Haiku, con acceso a Glob/Grep/Read/Bash pero sin Write/Edit. Cada invocación corre en su propia ventana de contexto y devuelve solo un resumen ([sub-agents docs](https://docs.anthropic.com/en/docs/claude-code/sub-agents)).

Usa Explore (directamente o vía una skill con `context: fork` y `agent: Explore`) para preguntas "¿dónde se define X?" / "¿cómo funciona Y?", tareas multifase que agotarían el presupuesto de contexto en descubrimiento, y búsquedas amplias cuya salida raw no volverás a leer. Especifica thoroughness: `quick`, `medium` o `very thorough`.

### Patrón combinado: planificar y luego ejecutar

El workflow recomendado para trabajo no trivial es **explore → plan → execute → commit** ([best practices](https://docs.anthropic.com/en/docs/claude-code/best-practices)): delega el sondeo a Explore, haz que Claude escriba el plan, revísalo/edítalo con `Ctrl+G`, apruébalo hacia `acceptEdits` y luego commitea. Plan mode para investigación, ejecución directa para las ediciones.

## Playbook de refinamiento iterativo

### 1. Proporciona ejemplos input/output

Cuando la ambigüedad de prosa es el cuello de botella (extracción, transformación, formato), aporta **2–3 pares input/output concretos** en vez de más adjetivos. Claude generaliza desde ejemplos de forma más fiable que desde prosa como "friendly but professional." Los ejemplos también sirven como casos de regresión.

### 2. Iteración guiada por pruebas

Las pruebas son una señal de verificación inequívoca. El bucle: haz que Claude escriba una suite que cubra happy path, edge cases y presupuestos de rendimiento *antes* de que exista implementación; ejecútala y comparte fallos; haz que Claude implemente lo mínimo para volver verde una prueba fallida; repite hasta verde; refactoriza con la suite como red de seguridad. Es el principio "dar a Claude una forma de verificar su trabajo" hecho concreto.

### 3. El patrón entrevista

En dominios desconocidos, pide a Claude que te entreviste antes de escribir código: *"Before you implement caching, list the questions you'd need answered (eviction, invalidation, failure modes, memory budget, multi-tenant isolation) and ask each one."* Esto saca a la luz consideraciones que el desarrollador no anticipó. Úsalo para preocupaciones transversales: caching, retries, auth, billing, migrations.

### 4. Agrupar fixes interactuantes frente a secuenciar fixes independientes

Cuando aparecen varios problemas, decide si interactúan:

- **Interactuantes**: un fix en un lugar cambia la respuesta correcta en otro (cambiar formato de fecha afecta parsing, display y escrituras DB). Pon todo en un **único mensaje detallado** para que Claude razone holísticamente.
- **Independientes**: typo en un helper, null check faltante en otro lugar. Arregla **secuencialmente**, uno por turno: diffs pequeños, contexto fresco, fallos atribuibles.

Para un solo edge case fallido (null en una migración), aporta el input/expected output específico de ese caso; no reescribas todo el script.

## Puntos de enfoque para el examen

- Los comandos compartidos por el equipo viven en **`.claude/commands/<name>.md`** en el repo. No en `~/.claude/commands/`, no en `CLAUDE.md`, no en un `config.json` ficticio.
- Las skills son **bajo demanda**, `CLAUDE.md` está **siempre cargado**. Los estándares persistentes van en `CLAUDE.md`; workflows específicos de tarea van en skills.
- `context: fork` aísla salida verbosa o exploratoria de una skill en un subagente; combínalo con `agent: Explore` para descubrimiento de solo lectura.
- `allowed-tools` preaprueba una lista de herramientas mientras la skill está activa; no bloquea otras herramientas: las reglas de permisos aún aplican.
- `argument-hint` alimenta autocompletado; `$ARGUMENTS`/`$N` sustituyen valores reales en el cuerpo del prompt.
- Plan mode entra vía ciclo `Shift+Tab`, `--permission-mode plan`, prefijo `/plan` o `defaultMode: "plan"`. Es de solo lectura.
- Elige plan mode para trabajo arquitectónico / multiarchivo / multienfoque; elige ejecución directa para cambios de un archivo y bien acotados.
- El subagente Explore es integrado, respaldado por Haiku, de solo lectura y devuelve resúmenes: úsalo para mantener el descubrimiento fuera del contexto principal.
- Iteración: ejemplos input/output superan la prosa; las pruebas son el bucle de feedback de mayor rendimiento; entrevista en dominios desconocidos; agrupa fixes interactuantes, secuencia los independientes.

## Referencias

- Slash commands — <https://docs.anthropic.com/en/docs/claude-code/slash-commands>
- Skills — <https://docs.anthropic.com/en/docs/claude-code/skills>
- Permission modes (plan mode) — <https://docs.anthropic.com/en/docs/claude-code/permission-modes>
- Subagents (Explore) — <https://docs.anthropic.com/en/docs/claude-code/sub-agents>
- Hooks — <https://docs.anthropic.com/en/docs/claude-code/hooks>
- Best practices — <https://docs.anthropic.com/en/docs/claude-code/best-practices>
- Extend Claude Code — <https://docs.anthropic.com/en/docs/claude-code/features-overview>
- Open Agent Skills standard — <https://agentskills.io/>
- Claude Code repo — <https://github.com/anthropics/claude-code>
- Official plugins — <https://github.com/anthropics/claude-plugins-official>
