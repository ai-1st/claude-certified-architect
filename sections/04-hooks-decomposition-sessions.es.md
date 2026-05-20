---
title: "Sección 4 — Hooks del Agent SDK, descomposición de tareas y gestión de sesiones"
linkTitle: "4. Hooks, descomposición y sesiones"
weight: 4
description: "Dominios 1.5–1.7 — hooks PostToolUse y de intercepción de llamadas, prompt chaining frente a descomposición adaptativa, fork_session y reanudación de sesiones nombradas."
---

## Qué cubre esta sección

Tres habilidades de nivel arquitecto estrechamente relacionadas que convierten un agente de Claude de un chatbot probabilístico en un sistema controlable y depurable:

1. **Hooks (1.5)** — callbacks deterministas de Python/TypeScript (o scripts shell) que interceptan el bucle del agente en puntos bien definidos del ciclo de vida para imponer políticas, normalizar salida de herramientas y auditar cada acción.
2. **Descomposición de tareas (1.6)** — saber cuándo cablear una pipeline secuencial (prompt chaining) frente a dejar que el modelo genere dinámicamente sus propias subtareas (orchestrator-workers / planes adaptativos).
3. **Gestión de sesiones (1.7)** — la disciplina operativa de `--continue`, `--resume` y `--fork-session`, más el criterio de *cuándo* desechar una sesión y empezar de nuevo con un resumen estructurado.

Criterio de aprobación: mirar un workflow y decir de inmediato "esto es un problema de hooks, no de prompts", "esto es prompt chaining, no orchestrator-workers" o "esta sesión está obsoleta, resume y reinicia."

## Material fuente (de la guía oficial)

### 1.5 Hooks para intercepción y normalización

`PostToolUse` intercepta *resultados* de herramientas y los transforma antes de que el modelo vea bytes raw. `PreToolUse` intercepta *llamadas* a herramientas y puede bloquear, modificar o redirigir; ejemplo canónico: "bloquear cualquier reembolso donde `amount > 500` y enrutar a escalación humana." Los hooks dan **garantías deterministas**; los prompts solo dan **cumplimiento probabilístico**. Habilidades: normalizar timestamps heterogéneos entre servidores MCP; bloquear acciones que violan políticas y redirigir a alternativas; elegir hooks sobre prompts cuando el cumplimiento debe garantizarse.

### 1.6 Estrategias de descomposición de tareas

**Prompt chaining** para workflows predecibles donde los pasos se conocen por adelantado frente a **descomposición dinámica adaptativa** para workflows abiertos donde las subtareas solo se descubren en runtime. Revisiones de código grandes: análisis local por archivo + una pasada separada de integración entre archivos para evitar dilución de atención.

### 1.7 Estado de sesión, reanudación y forking

`--resume <id-or-name>` continúa una conversación previa específica; `--continue`/`-c` reanuda la más reciente en este `cwd`. `--fork-session` (`fork_session: true` / `forkSession: true` en el SDK) crea una rama independiente desde una línea base compartida. Después de que los archivos cambien en disco, informa a un agente reanudado o sus resultados `Read` en caché estarán obsoletos; una sesión fresca inicializada con un resumen hecho a mano a veces es más fiable que reanudar con resultados de herramientas obsoletos.

## Referencia de hooks

### Tipos de eventos de hook

| Evento | Cuándo se dispara | Uso típico |
| --- | --- | --- |
| `SessionStart` | La sesión empieza o se reanuda (matchers: `startup`, `resume`, `clear`, `compact`) | Inicializar logging, inyectar reglas de proyecto |
| `SessionEnd` | La sesión termina | Vaciar logs, limpiar recursos |
| `UserPromptSubmit` | El usuario envía un prompt, antes de que el modelo lo vea | Inyectar contexto, limpiar PII, bloquear prompts fuera de tema |
| `PreToolUse` | Antes de ejecutar cualquier llamada de herramienta | **Enforcement de políticas**, reescritura de inputs, redirección de sandbox |
| `PostToolUse` | Después de que una llamada de herramienta tiene éxito | **Normalización de datos**, audit logging, conversión de formato |
| `PostToolUseFailure` | Después de que falla una llamada de herramienta | Manejo de errores personalizado |
| `PostToolBatch` | Se resuelve un batch paralelo de llamadas de herramientas | Inyectar convenciones una vez por batch |
| `PermissionRequest` / `PermissionDenied` | Diálogo de permiso o denegación en modo automático | UX personalizada, decisiones de reintento |
| `SubagentStart` / `SubagentStop` | Un subagente se crea / termina | Rastrear trabajo paralelo, agregar resultados |
| `PreCompact` / `PostCompact` | Ciclo de vida de compactación de conversación | Archivar transcripción antes de un resumen con pérdida |
| `Notification` | El agente emite una notificación | Reenviar a Slack/PagerDuty |
| `Stop` / `StopFailure` | El turno termina normalmente / vía error de API | Guardar estado, alertar por rate-limit |
| `TaskCreated` / `TaskCompleted` | Ciclo de vida de tareas | Imponer convenciones de ticket-ID, gate en pruebas pasando |
| `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `FileChanged`, `WorktreeCreate/Remove`, `Setup` | Ciclo de vida misceláneo | Auditar, recargar config, reaccionar a cambios externos |

`SessionStart`/`SessionEnd` son callbacks solo del TS-SDK; en Python deben ser hooks shell en `.claude/settings.json` más `setting_sources=["project"]`.

### Anatomía de un hook

Dos rutas de registro: **callbacks del SDK** (`ClaudeAgentOptions.hooks` / `options.hooks`) o **hooks de comando shell** en `.claude/settings.json`: procesos hijo que reciben JSON de evento por stdin.

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

**Matchers**: `*`/`""`/omitido coincide con todo; letras+dígitos+`|` es exacto / lista con pipe (`Edit|Write`); cualquier otra cosa es una regex JS (`^mcp__memory__`). Los hooks de herramientas coinciden solo por *nombre* de herramienta: filtra `file_path` dentro del handler.

**Input** — cada hook recibe `session_id`, `cwd`, `hook_event_name` más campos específicos del evento (`tool_name`, `tool_input`, `tool_response`, …). El contexto de subagente añade `agent_id`, `agent_type`.

**Output** — JSON en dos capas: nivel superior (`systemMessage`, `continue` / `continue_`, `additionalContext`) y `hookSpecificOutput` (dependiente del evento). Para `PreToolUse`: `permissionDecision` ∈ `{"allow", "deny", "ask", "defer"}`, `permissionDecisionReason`, `updatedInput`. Para `PostToolUse`: `additionalContext` (añadir) o `updatedToolOutput` (reemplazar).

**Códigos de salida de hook shell**: `0` = éxito, parsear stdout como JSON; `2` = error bloqueante, stderr se alimenta al modelo (para `PreToolUse` bloquea la llamada); cualquier otro distinto de cero = error no bloqueante.

Cuando múltiples hooks se disparan en el mismo evento: **deny > defer > ask > allow**; un solo `deny` bloquea.

### Ejemplos concretos

**1. PostToolUse normalizando formatos de fecha heterogéneos** — tres servidores MCP devuelven Unix epoch seconds, cadenas ISO 8601 e enteros numéricos en milisegundos. Fuerza una representación ISO 8601 antes de que el modelo tenga que razonar sobre ella.

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

El modelo solo ve ISO 8601, así que la aritmética de fechas entre herramientas simplemente funciona.

**2. PreToolUse bloqueando reembolsos de alto valor** — garantiza que `process_refund` nunca pueda ejecutarse por encima de $500.

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

`permissionDecisionReason` es el ingrediente mágico: le dice al modelo *por qué* se bloqueó la llamada y qué herramienta alternativa usar, así que el agente se autocorrige en el siguiente turno en vez de repetir la misma llamada denegada.

### Cuándo NO usar un hook (usar un prompt)

| Situación | Hook | Prompt |
| --- | --- | --- |
| Regla regulatoria dura ("never log SSNs") | Sí | No |
| Contrato de forma de datos ("always ISO 8601") | Sí | No |
| Audit log de cada llamada de herramienta | Sí | No |
| Cuota/coste por llamada | Sí (`PreToolUse`) | No |
| Preferencia suave de estilo ("prefer 2-space indent") | No | Sí |
| Persona / tono ("be concise, no emojis") | No | Sí |

Regla práctica: si violarlo es un incidente P0, va en un hook.

## Patrones de descomposición

### Prompt chaining (secuencial, predecible)

Una pipeline fija de N pasos donde el paso *k* alimenta al paso *k+1*. Cada llamada LLM hace una tarea más fácil que un megaprompt único. De [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents): *"ideal for situations where the task can be easily and cleanly decomposed into fixed subtasks. The main goal is to trade off latency for higher accuracy, by making each LLM call an easier task."*

Añade *gates* programáticos entre pasos para fallar rápido: outline → revisar rúbrica (gate) → escribir documento; análisis lint por archivo → revisión de integración entre archivos.

### Descomposición dinámica adaptativa

También llamada **orchestrator-workers**. El LLM orquestador mira el input, decide qué subtareas hacen falta (no podía saberlo por adelantado), crea workers y sintetiza resultados. *"Well-suited for complex tasks where you can't predict the subtasks needed (in coding, the number of files that need to be changed and the nature of the change in each file likely depend on the task)."*

En Agent SDK esto se mapea al patrón `Task` / subagente, opcionalmente rastreado mediante hooks `SubagentStart`/`SubagentStop`.

### Matriz de decisión

| Forma del workflow | Patrón |
| --- | --- |
| Pasos conocidos por adelantado, idénticos para cada input | Prompt chaining |
| Subtareas independientes de forma *conocida* | Paralelización (seccionamiento) |
| Misma tarea, quieres N votos para confianza | Paralelización (votación) |
| Categorías distintas, cada una con especialista | Routing |
| El número/forma de subtareas depende del input | Orchestrator-workers (adaptativo) |
| La salida se beneficia de un bucle crítico | Evaluator-optimizer |
| Abierto, multiturno, horizonte desconocido | Bucle completo de agente |

### Ejemplo trabajado — revisión de código por archivo + entre archivos

Un PR de 40 archivos metido en un solo prompt sufre dilución de atención: el modelo hojea y pierde bugs. Descompón:

**Fase 1 — análisis local por archivo (encadenado, paralelizable)**: cada archivo recibe su propio contexto, bifurcado desde una sesión base compartida que ya cargó `CLAUDE.md`, la descripción del PR y el diff stat. Forking evita pagar tokens de nuevo para restablecer contexto por archivo.

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

**Fase 2 — pasada de integración entre archivos (una sola llamada):**

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

Prompt chaining (Fase 1 → Fase 2) sobre paralelización dentro de la Fase 1.

**Variante abierta — "add tests to a legacy codebase":** *adaptativa*, no encadenada. Mapear estructura → identificar módulos sin pruebas de alto impacto → construir backlog priorizado → tomar el primer ítem, escribir pruebas, descubrir una dependencia oculta, devolverla al backlog, repetir. El paso 4..N solo se conoce en runtime: orchestrator-workers, no chaining.

## Gestión de sesiones

### `--resume` frente a `--continue` frente a sesión nueva

| Necesidad | Usar | Cómo encuentra la sesión |
| --- | --- | --- |
| Sesión más reciente en este directorio | `claude -c` / `continue: true` | La más nueva en `~/.claude/projects/<cwd-slug>/` |
| Sesión específica con nombre o ID | `claude -r "auth-refactor"` / `resume: "<id>"` | ID exacto o lookup de `--name` |
| Conversación totalmente nueva | `claude` | ID de sesión fresco |
| One-shot, sin persistencia en disco (solo TS) | `persistSession: false` | Solo en memoria |

Las sesiones viven en `~/.claude/projects/<slugified-cwd>/<session-id>.jsonl`, donde el slug es el directorio de trabajo absoluto con cada carácter no alfanumérico reemplazado por `-`. **Un `cwd` distinto significa que `resume` no puede encontrar el archivo**: la causa #1 de "por qué resume devuelve una sesión fresca."

Captura el ID de sesión desde el `ResultMessage` (Python) / `SDKResultMessage` (TS) en cada ejecución si pretendes reanudar programáticamente. En TS también está en el `SystemMessage` init.

### `fork_session`: cuándo y cómo

Forking copia la transcripción existente a un *nuevo* ID de sesión y permite que diverja. El original queda intacto.

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

Casos de uso: comparar A/B dos enfoques de refactorización desde una línea base compartida; bake-off de estrategias de pruebas; exploración arriesgada con retorno garantizado al padre.

Advertencia: forking ramifica la *conversación*, no el *filesystem*. Si ambos forks editan archivos en el mismo repo, esas ediciones chocan. Combínalo con [file checkpointing](https://code.claude.com/docs/en/agent-sdk/file-checkpointing) o worktrees de git (`claude -w <name>`) para aislamiento real.

### Árbol de decisión de contexto obsoleto

Después de un cambio de código, antes de reanudar una investigación:

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

Una sesión limpia con un resumen curado a menudo supera una reanudación porque la transcripción reanudada todavía contiene salidas `Read` obsoletas en las que el modelo confía: puede "recordar" firmas de funciones que ya no existen y alucinar llamadas. Patrón: mantén una **sesión nombrada** de larga vida (`claude -n design-review`) para contexto arquitectónico estable y haz **fork** por investigación. Desecha los forks.

## Puntos de enfoque para el examen

- **Los hooks son deterministas; los prompts son probabilísticos.** Cualquier regla que deba cumplirse el 100% del tiempo va en `PreToolUse` (saliente) o `PostToolUse` (entrante).
- Memoriza valores de `permissionDecision`: `allow`, `deny`, `ask`, `defer`. Memoriza prioridad: **deny > defer > ask > allow**.
- `PostToolUse.updatedToolOutput` *reemplaza* lo que ve el modelo; `additionalContext` *añade*. `PreToolUse.updatedInput` reescribe el input de herramienta, pero solo si también devuelves `permissionDecision: "allow"`.
- Códigos de salida de shell-hook: `0` = parsear JSON; `2` = bloquear (`PreToolUse`) / alimentar stderr al modelo; cualquier otro = error no bloqueante.
- Selector de descomposición: forma conocida → prompt chaining; forma desconocida → orchestrator-workers. Revisión de código grande = pasada por archivo + pasada de integración entre archivos.
- `--continue` no necesita ID pero solo encuentra la sesión más reciente en el `cwd` *actual*. `--resume` necesita un ID o `--name`. `--fork-session` requiere `--resume`/`--continue` y produce un nuevo ID.
- Almacenamiento de sesiones: `~/.claude/projects/<slugified-cwd>/<session-id>.jsonl`. Un `cwd` incorrecto es la razón #1 por la que resume devuelve silenciosamente una sesión fresca.
- Los resultados de herramientas obsoletos en una sesión reanudada pueden perjudicar más de lo que ayudan: a veces una sesión fresca con resumen curado rinde mejor que reanudar.
- Fork para comparar; resume para continuar; reinicia-con-resumen cuando el contexto se pudrió.

## Referencias

- [Intercept and control agent behavior with hooks](https://code.claude.com/docs/en/agent-sdk/hooks) — referencia completa de hooks de Agent SDK, tabla de eventos, forma de callbacks, ejemplos.
- [Hooks reference](https://code.claude.com/docs/en/hooks) — cada evento, patrones de matcher, salida JSON, semántica de códigos de salida, variantes shell/HTTP/MCP-tool.
- [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide) — quickstart con ejemplos trabajados.
- [Configure hooks (Anthropic blog)](https://claude.com/blog/how-to-configure-hooks) — recorrido avanzado de `settings.json`.
- [Work with sessions](https://code.claude.com/docs/en/agent-sdk/sessions) — `continue`, `resume`, `fork_session`, captura de IDs, advertencias cross-host.
- [CLI reference](https://code.claude.com/docs/en/cli-reference) — `--continue`, `--resume`, `--fork-session`, `--name`, `--session-id`, `--from-pr`.
- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — taxonomía canónica de Anthropic: prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer.
- [Anthropic cookbook — agents patterns](https://github.com/anthropics/claude-cookbooks/tree/main/patterns/agents) — notebooks ejecutables para cada patrón.
- [How the agent loop works](https://code.claude.com/docs/en/agent-sdk/agent-loop) — modelo conceptual de turnos, llamadas a herramientas y dónde encajan los hooks.
- [Permissions](https://code.claude.com/docs/en/agent-sdk/permissions) — mecanismo complementario a `PreToolUse` para control de acceso a herramientas.
