---
title: "Sección 9 — Claude Code en pipelines CI/CD"
linkTitle: "9. Integración CI/CD"
weight: 9
description: "Dominio 3.6 — claude -p y --output-format json, --json-schema, diseño de prompts para revisión accionable y con pocos falsos positivos."
---

## Qué cubre esta sección

Cómo ejecutar Claude Code sin supervisión dentro de un sistema de build: los flags que previenen bloqueos interactivos, las formas de salida que pueden parsear las herramientas downstream, el contexto de proyecto que mantiene al modelo dentro de política, y los patrones arquitectónicos (revisor independiente, revisión multipaso, batch frente a sync) que el examen evalúa mediante las preguntas 10, 11 y 12.

## Material fuente (de la guía oficial)

Task Statement 3.6 (líneas 600–629 de la guía) cubre: `-p` / `--print` para modo no interactivo; `--output-format json` más `--json-schema` para hallazgos parseables por máquina; `CLAUDE.md` como mecanismo de contexto de proyecto para Claude Code invocado desde CI (estándares de pruebas, convenciones de fixtures, criterios de revisión); y aislamiento de contexto de sesión: la sesión que *escribió* el código es la sesión incorrecta para *revisarlo*. Habilidades: prevenir bloqueos interactivos con `-p`; producir hallazgos estructurados para comentarios inline en PR; alimentar hallazgos de revisiones previas para suprimir comentarios duplicados; pasar archivos de prueba existentes a ejecuciones de generación de tests para evitar cubrir de nuevo escenarios; documentar estándares de pruebas y fixtures en `CLAUDE.md`.

Sample Question 10 (`-p` es el flag headless correcto; `CLAUDE_HEADLESS`, `--batch` y `< /dev/null` son distractores), Question 11 (Batches API para el informe nocturno de deuda técnica, *no* para el gate pre-merge bloqueante), y Question 12 (dividir un PR de 14 archivos en pasadas por archivo más una pasada de integración) dependen de esta sección.

## Claude Code headless / no interactivo

### El flag `-p`

`claude -p "<prompt>"` (alias `--print`) hace que Claude Code se ejecute como una consulta SDK one-shot: consume el prompt, transmite o imprime la respuesta a stdout y sale con un status code. Sin chequeo de TTY, sin bucle de prompts de permiso, sin esperar stdin. Es el único mecanismo documentado para CI. La trampa de Question 10 es que no existe env var `CLAUDE_HEADLESS` ni flag `--batch`, y redirigir stdin desde `/dev/null` no funciona porque Claude Code aún espera renderizar una UI interactiva.

```bash
claude -p "Review the diff for SQL-injection and SSRF risks. Be terse." \
  --model claude-sonnet-4-6 \
  --max-turns 6
```

Añade `--bare` (Claude Code 2.1.x) cuando no necesites hooks, plugins, servidores MCP ni `CLAUDE.md` autodetectado; reduce latencia de arranque y elimina una clase de fallos tipo "por qué mi CI tomó la config MCP de mi portátil."

### Formatos de salida (text, json, stream-json)

`--output-format` controla cómo se serializa la respuesta:

- `text` (por defecto): prosa plana; buena para humanos, mala para parsers.
- `json`: un objeto JSON con la respuesta final, coste, duración, ID de sesión, stop reason. Úsalo cuando el siguiente paso es `jq` o un script.
- `stream-json`: eventos newline-delimited para herramientas que renderizan progreso o muestran eventos de tool-use. Combinar con `--include-partial-messages` para deltas token por token, `--include-hook-events` para ciclo de vida de hooks.

`--input-format stream-json` permite que un proceso padre alimente a Claude Code con un stream estructurado de eventos; útil si tu orquestador CI es a su vez un agente.

### `--json-schema` para hallazgos estructurados

`--json-schema '<JSON Schema>'` (solo print mode) hace que Claude Code emita un payload final que se *valida* contra el schema que suministras. Este es el flag que el examen espera cuando la siguiente etapa de la pipeline es "publicar cada hallazgo como comentario inline de revisión en GitHub PR."

```bash
claude -p "Review changed files in $PR_DIFF for security issues" \
  --output-format json \
  --json-schema "$(cat .ci/review-schema.json)" \
  --max-turns 8 \
  > findings.json
```

Un schema que mapea limpiamente al payload `pulls/{pr}/comments` de GitHub se ve así:

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

### Permission modes para ejecuciones sin supervisión

Los prompts interactivos de permisos son la segunda causa más común de jobs CI atascados (después de olvidar `-p`). Flags clave:

- `--allowedTools "Read,Bash(git diff *),Bash(gh pr view *)"`: allowlist estrecha; default preferido.
- `--disallowedTools "Edit,Write,Bash(rm *)"`: bloquear escrituras cuando el job es revisión de solo lectura.
- `--permission-mode acceptEdits` (o `bypassPermissions`): aceptar cambios sin prompts. `--dangerously-skip-permissions` es la forma explícita "sé lo que hago"; reservar para runners sandboxed sin secretos en env.
- `--permission-prompt-tool <mcp-tool>` delega decisiones de permisos a una herramienta MCP personalizada cuando cada llamada debe registrarse antes de aprobarse.

### Flags de control de sesión

- `--session-id <uuid>` fija un UUID para que pasos downstream puedan reanudar la misma conversación después.
- `--resume <id>` / `-r` continúa esa sesión: el mecanismo detrás de "incluir hallazgos de revisión previos al volver a ejecutar."
- `--continue` / `-c` elige la conversación más reciente en el directorio actual.
- `--fork-session` reanuda-pero-ramifica para que los reintentos no muten el hilo canónico.
- `--no-session-persistence` no escribe nada a disco; útil en runners stateless donde el workspace se destruye entre jobs.

## Patrones de referencia

### 1. Revisión de seguridad pre-merge (sincrónica, bloqueante)

Los gates bloqueantes deben usar una llamada en tiempo real. Según Question 11, Message Batches API es la herramienta incorrecta aquí: su descuento de 50% de coste llega al precio de una latencia de hasta 24 horas, inaceptable para un desarrollador esperando merge.

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

### 2. Generación de pruebas en batch nocturno

Los jobs nocturnos de deuda de pruebas son exactamente el workload donde Batches API gana su descuento del 50%. Ejecuta Claude Code con `-p`, pasa los archivos de prueba existentes en contexto para que no recubra escenarios y enruta las solicitudes de generación por Batches API (ver Sección 11).

```bash
ls tests/ | xargs -I{} cat tests/{} > .ci/existing-tests.txt

claude -p \
  "Generate missing tests for src/billing/. Existing tests are attached; do not duplicate scenarios already covered." \
  --append-system-prompt-file .ci/existing-tests.txt \
  --output-format json \
  --max-turns 12
```

### 3. Publicación de comentarios inline de PR desde salida JSON

Con un `findings.json` validado contra schema, un único bucle shell convierte el archivo en comentarios de revisión nativos:

```bash
jq -c '.findings[]' findings.json | while read f; do
  gh api -X POST "repos/$REPO/pulls/$PR/comments" \
    -f body="$(echo $f | jq -r .message)" \
    -f path="$(echo $f | jq -r .path)" \
    -F line="$(echo $f | jq -r .line)" \
    -f commit_id="$SHA" -f side=RIGHT
done
```

### 4. Reejecutar revisión en commits nuevos sin duplicar comentarios

Domain 3.6 es explícito: cuando se vuelve a empujar el PR, incluye hallazgos previos en contexto e instruye a Claude a reportar solo problemas nuevos o aún no resueltos.

```bash
gh pr view $PR --json comments -q '.comments' \
  | claude -p \
      "Comments already posted are attached. Re-review the new diff and emit ONLY findings that are (a) new or (b) still unaddressed. Use the findings schema." \
      --resume "$REVIEW_SESSION_ID" \
      --output-format json \
      --json-schema "$(cat .ci/review-schema.json)" \
      > new-findings.json
```

## La GitHub Action de Claude Code

### Inicio rápido

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

### Inputs / outputs

Inputs clave de `action.yml`: `anthropic_api_key` (o `claude_code_oauth_token`, `use_bedrock`, `use_vertex`); `prompt`; `claude_args` (args CLI raw añadidos al `claude -p` subyacente); `trigger_phrase` (default `@claude`); `label_trigger` (default `claude`); `track_progress` (renderiza comentario de seguimiento con checkboxes); `use_sticky_comment` (un comentario para todo); `classify_inline_comments` (prefiltro Haiku para descartar comentarios de prueba/sonda); `branch_prefix` (default `claude/`). El JSON estructurado devuelto por la ejecución se expone como outputs de GitHub Action.

### Gotchas comunes

- La herramienta MCP de comentarios inline necesita `confirmed: true` para publicar inmediatamente; si no, los comentarios se almacenan en `/tmp/inline-comments-buffer.jsonl` para clasificación Haiku.
- La Action necesita permisos `pull-requests: write` e `issues: write` del token; si no, falla silenciosamente al publicar.
- Pasa selección de modelo y límites de turnos vía `claude_args`, no mediante inputs separados.

## CLAUDE.md para CI

### Secciones que dan rendimiento

Para un Claude invocado desde CI, las secciones de `CLAUDE.md` de mayor rendimiento son las que el modelo de otro modo tendría que adivinar: el comando del test runner, dónde viven los fixtures, qué librería de mocks usas, qué cuenta como prueba "valiosa" y qué criterios de revisión te importan; exactamente los estándares de pruebas, fixtures y criterios de revisión que las habilidades de 3.6 mencionan.

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

### Evitar bloat de contexto

`CLAUDE.md` se carga en cada invocación, así que cada línea se paga en cada ejecución CI. Elimina decisiones históricas, enlaza a ADRs, prefiere una línea declarativa sobre un párrafo y resiste pegar toda la guía de estilo. En modo `--bare`, `CLAUDE.md` no se autocarga; opta explícitamente con `--append-system-prompt-file` cuando lo quieras.

## Patrón de revisor independiente

### Por qué una sesión separada supera la autorevisión

Domain 3.6 llama explícitamente al *aislamiento de contexto de sesión*. La sesión que generó el código lleva su propio historial de razonamiento: las justificaciones que produjeron el bug. Esa sesión está sesgada a confirmar sus decisiones previas. Una sesión fresca ve solo el diff, sin narrativa adjunta, y detecta problemas que la sesión autora racionalizó. El producto Code Review de Anthropic usa una flota de agentes independientes por la misma razón.

En práctica: nunca hagas `--continue` desde la sesión de implementación hacia el job de revisión. Crea el revisor con un `--session-id` nuevo, conversación vacía y solo el diff más `CLAUDE.md` como contexto.

### Multipaso para PRs grandes (por archivo + integración entre archivos)

Esto es Question 12. Una sola pasada de revisión sobre 14 archivos presenta *dilución de atención*: la profundidad varía de archivo a archivo y patrones idénticos reciben veredictos contradictorios. La corrección son dos pasadas:

1. **Pasada por archivo.** Una invocación `claude -p` por archivo cambiado, acotada a problemas locales (bugs, estilo, seguridad en este archivo).
2. **Pasada de integración.** Una invocación adicional con el resumen del diff, preguntando específicamente por preocupaciones entre archivos: cambios en límites API, migraciones de schema, contract drift, mutación de estado compartido.

Un modelo más grande o una ventana de contexto mayor no solucionan esto; el problema es la asignación de atención, no el presupuesto de tokens.

## Perillas de coste y latencia

### Prompt caching

Las lecturas de caché cuestan aproximadamente 10x menos que las escrituras, y Claude Code está construido para acertar la caché agresivamente. Para maximizar hits en CI: pon contenido estable (system prompt, `CLAUDE.md`, convenciones del repo) *primero*, contenido dinámico (diff, hallazgos previos) *al final*; no cambies modelo ni lista de herramientas a mitad de sesión; en Claude Code 2.1.108+, define `ENABLE_PROMPT_CACHING_1H=1` para jobs programados que se ejecutan más de cada cinco minutos. Usa `--exclude-dynamic-system-prompt-sections` con `-p` para mantener detalles por máquina fuera de la clave de caché cuando muchos runners comparten la misma tarea.

### Batch API para workloads no bloqueantes

Enlace cruzado a Sección 11: Message Batches API da 50% de ahorro de coste pero hasta 24 horas de SLA. Úsala para el informe nocturno de deuda técnica; no para el gate pre-merge. Esta es la distinción que evalúa Sample Question 11.

### Elegir tamaño de modelo por job

Revisión de seguridad pre-merge en una ruta hot: Sonnet. Pasada de estilo-y-lint en PRs de docs: Haiku. Revisión arquitectónica en el refactor trimestral: Opus, limitado con `--max-budget-usd`. Cambia modelo vía `--model`; no lo cambies a mitad de sesión o invalidas la caché.

## Puntos de enfoque para el examen

- `-p` es la *única* forma documentada de ejecutar Claude Code sin supervisión. `CLAUDE_HEADLESS`, `--batch` y `</dev/null` son distractores.
- `--output-format json --json-schema <schema>` es la receta canónica para hallazgos parseables por pipelines.
- Workloads bloqueantes (pre-merge) usan llamadas en tiempo real; workloads nocturnos no bloqueantes usan Batches API.
- PRs grandes: pasadas por archivo más una pasada de integración, no un megaprompt único ni una ventana de contexto más grande.
- La sesión que escribió el código es la sesión incorrecta para revisarlo; crea un revisor independiente.
- Reejecutar una revisión debe incluir hallazgos previos en contexto para suprimir comentarios duplicados.
- `CLAUDE.md` lleva estándares de pruebas, fixtures y criterios de revisión; cada línea se paga en cada ejecución CI, así que mantenlo ligero.

## Referencias

- Claude Code CLI reference: <https://docs.claude.com/en/docs/claude-code/cli-usage>
- Run Claude Code programmatically (headless): <https://code.claude.com/docs/en/headless>
- Agent SDK (Python / TypeScript): <https://code.claude.com/docs/en/agent-sdk/python>, <https://code.claude.com/docs/en/agent-sdk/typescript>
- Structured outputs: <https://code.claude.com/en/agent-sdk/structured-outputs>
- Claude Code GitHub Action: <https://github.com/anthropics/claude-code-action> (usage: `docs/usage.md`, recipes: `docs/solutions.md`)
- GitHub Actions docs: <https://docs.anthropic.com/en/docs/claude-code/github-actions>
- Code Review product: <https://code.claude.com/docs/en/code-review> and <https://claude.com/blog/code-review>
- Subagents pattern: <https://claude.com/blog/subagents-in-claude-code>
- Prompt caching: <https://claude.com/blog/lessons-from-building-claude-code-prompt-caching-is-everything> and <https://docs.claude.com/en/docs/build-with-claude/prompt-caching>
- Memory / CLAUDE.md: <https://code.claude.com/en/memory>
