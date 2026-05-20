---
title: "Sección 2 — Bucles de agentes y manejo de stop_reason"
linkTitle: "2. Bucles de agentes"
weight: 2
description: "Dominio 1.1 — flujo de control del bucle de agente, valores de stop_reason que debes conocer y los antipatrones canónicos."
---

## Qué cubre esta sección

Cómo construir el flujo de control central de un agente de Claude: enviar una solicitud, inspeccionar `stop_reason`, ejecutar cualquier herramienta que Claude haya pedido, añadir resultados al historial e iterar. Todo patrón de nivel superior (orchestrator-workers, subagentes, evaluator-optimizer, Agent SDK) se construye sobre este bucle.

## Material fuente (de la guía oficial)

### Conocimiento requerido

- El ciclo de vida del bucle de agente: enviar solicitud a Claude, inspeccionar `stop_reason` (`"tool_use"` frente a `"end_turn"`), ejecutar las herramientas solicitadas y devolver resultados para la siguiente iteración.
- Cómo se añaden los resultados de herramientas al historial conversacional para que el modelo pueda razonar sobre la siguiente acción.
- La distinción entre **toma de decisiones impulsada por el modelo** (Claude razona qué herramienta llamar a continuación según el contexto) y **árboles de decisión preconfigurados** (el desarrollador codifica de forma fija la secuencia de herramientas).

### Habilidades requeridas

- Implementar flujo de control de bucle de agente que continúa mientras `stop_reason == "tool_use"` y termina cuando `stop_reason == "end_turn"`.
- Añadir resultados de herramientas al contexto conversacional entre iteraciones para que el modelo pueda incorporar nueva información en su razonamiento.
- Evitar antipatrones: parsear señales de lenguaje natural para terminar el bucle, usar límites arbitrarios de iteraciones como mecanismo principal de parada o revisar el texto del asistente como indicador de finalización.

## El bucle de agente, de extremo a extremo

La definición operativa de Anthropic para un agente es la más simple del campo: *"LLMs autonomously using tools in a loop."* El LLM aumentado (modelo + herramientas + retrieval + memoria) es el bloque fundacional; todo patrón de workflow (prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer) se compone a partir de él.

```text
  ┌──────────────────────────────────────────────────────────────┐
  │  user prompt + tool definitions  ─────────► messages array   │
  └──────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  POST /v1/messages  (Claude reasons about the next action)   │
  └──────────────────────────────────────────────────────────────┘
                              │
                              ▼
            ┌──────  inspect response.stop_reason  ──────┐
            │                                            │
   "tool_use"                                       "end_turn"
            │                                            │
            ▼                                            ▼
  ┌───────────────────────────┐               ┌─────────────────┐
  │ 1. append assistant turn  │               │ return final    │
  │    (incl. tool_use blocks)│               │ text to caller  │
  │ 2. execute each tool      │               └─────────────────┘
  │ 3. append a user turn     │
  │    with tool_result blocks│
  │ 4. loop back to /messages │
  └───────────────────────────┘
```

Recorrido de una iteración:

1. Envía `messages` más el schema de `tools` a `POST /v1/messages`.
2. Claude devuelve un mensaje `assistant`. Su `content` es una lista de bloques: cero o más bloques `text` y cero o más bloques `tool_use`. El `stop_reason` de nivel superior resume por qué se detuvo la generación.
3. Si `stop_reason == "tool_use"`: añade el turno del asistente literalmente, ejecuta cada herramienta solicitada, añade un único turno nuevo de `user` cuyo contenido sea una lista de bloques `tool_result` (uno por `tool_use_id`) y llama de nuevo a la API con el historial actualizado.
4. Si `stop_reason == "end_turn"`: el modelo decidió que la tarea terminó. Devuelve.

Los resultados de herramientas se **añaden al historial conversacional**, no se resumen y desaparecen. Cada nueva solicitud lleva todo el historial, así que Claude puede encadenar razonamiento a través de muchos turnos. El modelo, no tu código, decide qué herramienta llamar después según lo observado. Esta es la diferencia entre **toma de decisiones impulsada por el modelo** (Claude elige la herramienta N+1 a partir del contexto en curso) y **árboles de decisión preconfigurados** (tu código llama estáticamente a `tool_a()` → `tool_b()` → `tool_c()`). Los árboles de decisión son workflows; los bucles de agentes son agentes. La guía publicada de Anthropic recomienda preferir el workflow más simple siempre que la ruta pueda codificarse de forma fija.

## Valores de stop_reason que debes conocer

`stop_reason` forma parte de cada respuesta correcta de Messages API. Es la única señal sobre la que debes ramificar para decidir si seguir iterando. El conjunto completo de valores documentados aparece abajo.

| Valor | Significado | Qué debe hacer tu bucle |
| --- | --- | --- |
| `end_turn` | Claude terminó su respuesta de forma natural. | Salir del bucle. Devolver al llamador los bloques de texto de `response.content`. |
| `tool_use` | La respuesta contiene uno o más bloques `tool_use`; Claude espera que los ejecutes. | Añadir el turno del asistente, ejecutar cada bloque `tool_use`, añadir un turno `user` con bloques `tool_result` correspondientes (usar el mismo `tool_use_id`) y llamar otra vez a la API. |
| `max_tokens` | La salida alcanzó el parámetro `max_tokens`. La respuesta está truncada y puede contener un bloque `tool_use` **incompleto**. | Detectar truncamiento a mitad de llamada de herramienta revisando si el último bloque de contenido tiene `type == "tool_use"`; reintentar con un `max_tokens` mayor. En otro caso, pedir continuación o exponer una advertencia de truncamiento. |
| `stop_sequence` | La salida coincidió con una cadena personalizada en `stop_sequences`. La secuencia coincidente está en `response.stop_sequence`. | Tratarlo como una parada terminal correcta para ese patrón. Continuar o finalizar según tu protocolo. |
| `pause_turn` | El bucle de muestreo del servidor alcanzó su límite de iteraciones mientras ejecutaba **server tools** (web search, web fetch, code execution, etc.). La respuesta puede contener un bloque `server_tool_use` sin `server_tool_result` correspondiente. | Añadir la respuesta del asistente **sin cambios** y llamar de nuevo a la API con las mismas herramientas. Repetir hasta obtener un stop reason que no sea `pause_turn`. |
| `refusal` | El modelo rechazó por razones de seguridad (filtro de seguridad API en Sonnet 4.5+ / Opus 4.1+). | No iterar. Exponer un rechazo al llamador; opcionalmente reformular, enrutar a otro modelo (por ejemplo Haiku 4.5) o escalar. |
| `model_context_window_exceeded` | La generación se detuvo porque la respuesta alcanzó la ventana de contexto completa del modelo (no `max_tokens`). Sonnet 4.5+ por defecto; modelos anteriores necesitan un beta header. | Tratarlo de forma similar a `max_tokens`: la respuesta es válida pero limitada. Continuar, resumir o compactar contexto. |

Ramificar sobre `stop_reason` es la **única** prueba correcta de terminación. No parsees texto como "I'm done" o "Final answer:"; ese es el antipatrón canónico de abajo.

## Implementaciones de referencia

### Python — bucle raw de Messages API

Forma mínima y ejecutable usando el SDK Python `anthropic` (el mismo patrón de bucle que Anthropic muestra en [sus docs](https://docs.anthropic.com/en/docs/build-with-claude/handling-stop-reasons#handling-tool-use-workflows)).

```python
from anthropic import Anthropic

client = Anthropic()
MODEL = "claude-opus-4-7"

tools = [{
    "name": "get_weather",
    "description": "Get current weather for a city.",
    "input_schema": {
        "type": "object",
        "properties": {"location": {"type": "string"}},
        "required": ["location"],
    },
}]

def run_tool(name: str, tool_input: dict) -> str:
    if name == "get_weather":
        return f"Weather in {tool_input['location']}: 72F, clear"
    raise ValueError(f"unknown tool: {name}")

def agent_loop(user_prompt: str) -> str:
    messages = [{"role": "user", "content": user_prompt}]
    while True:
        resp = client.messages.create(
            model=MODEL, max_tokens=4096, tools=tools, messages=messages,
        )
        if resp.stop_reason == "end_turn":
            return "".join(b.text for b in resp.content if b.type == "text")
        if resp.stop_reason == "pause_turn":
            messages.append({"role": "assistant", "content": resp.content})
            continue
        if resp.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": resp.content})
            tool_results = [
                {"type": "tool_result", "tool_use_id": b.id,
                 "content": run_tool(b.name, b.input)}
                for b in resp.content if b.type == "tool_use"
            ]
            messages.append({"role": "user", "content": tool_results})
            continue
        raise RuntimeError(f"unhandled stop_reason: {resp.stop_reason}")
```

Notas: el turno del asistente se añade **literalmente** (los bloques `tool_use` deben sobrevivir en el historial). Los resultados de herramientas se devuelven en un único mensaje `user` cuyo `content` es una **lista** de bloques `tool_result`, uno por `tool_use_id`. `pause_turn` requiere reenviar el contenido del asistente sin cambios; no sintetices un resultado de herramienta.

### TypeScript — Claude Agent SDK

Para el [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/agent-loop) de nivel superior de Anthropic (`@anthropic-ai/claude-agent-sdk`), el bucle ya está implementado por ti. Consumes un stream asíncrono de mensajes tipados y revisas el `ResultMessage` terminal.

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const stream = query({
  prompt: "Find the failing tests in auth.ts and fix them.",
  options: {
    model: "claude-opus-4-7",
    maxTurns: 20,
    maxBudgetUsd: 1.0,
    permissionMode: "acceptEdits",
    allowedTools: ["Read", "Edit", "Bash", "Grep", "Glob"],
  },
});

for await (const message of stream) {
  if (message.type === "assistant") {
    console.log(`turn: ${message.message.content.length} blocks`);
  }
  if (message.type === "result") {
    if (message.subtype === "success") {
      console.log("done:", message.result);
    } else {
      console.error("stopped early:", message.subtype);
    }
  }
}
```

El SDK ejecuta internamente el mismo bucle impulsado por `stop_reason`: Claude evalúa, solicita herramientas, el SDK las ejecuta, los resultados vuelven automáticamente y un turno completo de Claude + ejecución de herramientas es lo que el SDK llama un *turn*. El bucle termina cuando Claude produce un mensaje de asistente sin bloques `tool_use`. `maxTurns` y `maxBudgetUsd` son **guardrails**, no el mecanismo principal de parada: producen un `ResultMessage` con subtipo `error_max_turns` o `error_max_budget_usd` cuando se disparan.

## Antipatrones que evitar

- **Parsear señales de lenguaje natural para terminar el bucle.** Buscar "Final answer:" o "DONE" en `response.content` es frágil: el modelo puede expresar la finalización de infinitas formas y aun así querer llamar otra herramienta. **Correcto:** ramificar solo sobre `response.stop_reason`.
- **Usar un límite de iteraciones como mecanismo principal de parada.** Codificar `for _ in range(10):` y salir al llegar al límite significa terminar a mitad de tarea en problemas difíciles y gastar tokens en tareas fáciles. **Correcto:** deja que `stop_reason == "end_turn"` termine el bucle; mantén límites de iteración y `max_budget_usd` solo como guardrails de seguridad.
- **Revisar el texto del asistente para decidir finalización.** Un turno puede contener *ambos*, bloques `text` y `tool_use` a la vez (Claude puede narrar mientras solicita una herramienta). Tratar "tiene texto" como "terminó" descarta llamadas a herramientas. **Correcto:** inspeccionar `stop_reason`; iterar sobre bloques de contenido por `type`.
- **Omitir el turno del asistente al añadir resultados de herramientas.** Enviar resultados sin añadir primero el turno `tool_use` del asistente produce un array `messages` inválido y un error de API. **Correcto:** añadir literalmente el turno del asistente y luego un único turno de usuario con bloques `tool_result`.
- **Añadir texto extra después de bloques `tool_result`.** Bloques `text` finales en el mismo turno de usuario enseñan a Claude a esperar texto de usuario después de cada llamada de herramienta, causando respuestas `end_turn` vacías. **Correcto:** el turno de usuario después de un `tool_use` debe contener solo bloques `tool_result`.
- **Ignorar `pause_turn`.** Con server tools, el servidor alcanza su propio límite de 10 iteraciones y devuelve `pause_turn` sin que debas producir `tool_result`. Tratarlo como `end_turn` trunca el agente. **Correcto:** añade la respuesta del asistente sin cambios y llama otra vez.
- **Ignorar truncamiento de `max_tokens` dentro de un bloque `tool_use`.** Si `stop_reason == "max_tokens"` y el último bloque es `tool_use`, el input JSON está incompleto y reintentar con el mismo límite fallará otra vez. **Correcto:** detecta el caso y reintenta con un `max_tokens` mayor.
- **Codificar de forma fija la secuencia de herramientas.** Llamar `read_file → search → write_file` desde tu propio código sin razonamiento model-in-the-loop es un **workflow**, no un agente. Está bien cuando la ruta es conocida, pero no esperes que se recupere de entradas nuevas.

## Puntos de enfoque para el examen

- Dado un valor de `stop_reason`, identificar la acción correcta del bucle (continuar con resultados de herramientas, añadir-y-reenviar para `pause_turn`, salir en `end_turn`, reintentar con mayor presupuesto en `max_tokens` a mitad de herramienta, exponer rechazo).
- Identificar qué mutaciones de `messages` son requeridas entre iteraciones: añadir literalmente el turno del asistente (incluidos bloques `tool_use`) y luego añadir un turno de usuario con bloques `tool_result` enlazados por `tool_use_id`.
- Distinguir toma de decisiones impulsada por el modelo de árboles de decisión preconfigurados, y escoger el patrón correcto para una tarea descrita (tarea abierta = agente; ruta fija bien definida = workflow).
- Detectar antipatrones en una muestra de código: parseo de texto para completar, límite de iteraciones como parada, falta del turno del asistente, bloques `text` extra después de `tool_result`, ignorar `pause_turn`.
- Saber que `max_turns` / `max_budget_usd` en Claude Agent SDK son guardrails: el terminador primario del bucle sigue siendo "sin bloques `tool_use` en la respuesta del asistente."

## Referencias

- [Handling stop reasons — Claude API docs](https://docs.anthropic.com/en/docs/build-with-claude/handling-stop-reasons) — lista autoritativa de cada valor de `stop_reason` con ejemplos de manejo en Python, TypeScript, Go, Java, C#, PHP y Ruby.
- [Messages API reference — Create a Message](https://docs.anthropic.com/en/api/messages) — schema de request/response incluyendo tipos de bloques de contenido `text`, `tool_use` y `tool_result`.
- [Tool use overview](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview) — definición de herramientas y el intercambio `tool_use` / `tool_result`.
- [Building effective agents — Anthropic Engineering (Dec 19, 2024)](https://www.anthropic.com/engineering/building-effective-agents) — post canónico que define workflows frente a agentes y el bloque LLM aumentado. Lectura obligatoria para el examen.
- [Effective context engineering for AI agents — Anthropic Engineering (Sep 29, 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — reafirma la definición operativa de agente como "LLMs autonomously using tools in a loop"; cubre compaction y patrones de subagentes para bucles de largo horizonte.
- [How the agent loop works — Claude Agent SDK docs](https://code.claude.com/docs/en/agent-sdk/agent-loop) — descripción oficial del ciclo de vida de turnos/mensajes del SDK, `max_turns`, `max_budget_usd` y subtipos de `ResultMessage`.
- [Agent SDK reference — Python](https://code.claude.com/docs/en/sdk/sdk-python) and [TypeScript](https://code.claude.com/docs/en/sdk/sdk-typescript) — `query()` frente a `ClaudeSDKClient`, tipos de mensajes y stream tipado de eventos.
- [anthropics/claude-cookbooks — tool_use](https://github.com/anthropics/claude-cookbooks/tree/main/tool_use) — ejemplos ejecutables del bucle, `tool_choice` y llamadas programáticas a herramientas.
- [Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python) and [TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript) — formas actuales de clientes usadas en los bucles anteriores.
