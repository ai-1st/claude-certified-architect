---
title: "Sección 5 — Diseño de interfaces de herramientas, distribución y herramientas integradas"
linkTitle: "5. Diseño de herramientas e integradas"
weight: 5
description: "Dominios 2.1, 2.3, 2.5 — escribir descripciones distintivas de herramientas, tool_choice, acceso acotado a herramientas y las integradas Read/Write/Edit/Bash/Grep/Glob."
---

## Qué cubre esta sección

Tres temas muy relacionados del Domain 2:

- **2.1** — Cómo escribir definiciones de herramientas (`name`, `description`, `input_schema`, `input_examples`) para que Claude elija de forma fiable la herramienta correcta, incluyendo cómo reconocer y corregir enrutamiento erróneo causado por descripciones.
- **2.3** — Cómo acotar qué herramientas ve cada agente o subagente, y cómo usar el parámetro `tool_choice` (`auto`, `any`, herramienta forzada, `none`) más `disable_parallel_tool_use` para controlar el comportamiento de invocación.
- **2.5** — Cómo aplicar el conjunto de herramientas integradas de Claude Code (`Read`, `Write`, `Edit`, `Bash`, `Grep`, `Glob`, además de `WebFetch`, `WebSearch`, `NotebookEdit`, `Agent`, `Task*`) al trabajo real en codebases, incluido el patrón canónico Edit-fails-on-non-unique-anchor → Read+Write fallback.

Dos ítems puntuados se apoyan directamente en este material: Sample Question 2 (descripciones mínimas de `get_customer` / `lookup_order`) y Sample Question 9 (acotar `verify_fact` al agente de síntesis).

## Material fuente (de la guía oficial)

### 2.1 Interfaces de herramientas con descripciones claras

Las descripciones de herramientas son *el* mecanismo principal que usa el LLM para elegir herramientas. Descripciones mínimas causan selección poco fiable entre herramientas similares: `analyze_content` frente a `analyze_document`, o `get_customer` frente a `lookup_order`. Las descripciones deben incluir formatos de input, consultas de ejemplo, casos límite y declaraciones explícitas de límites ("usa esta herramienta cuando…, no la uses cuando…"). La redacción del system prompt es sensible a palabras clave y puede sobreescribir una buena descripción, así que el system prompt forma parte de la superficie de enrutamiento de herramientas. Correcciones esperadas: diferenciar descripciones por propósito / inputs / outputs / cuándo usar, **renombrar** por solapamiento (`analyze_content` → `extract_web_results`), **dividir** herramientas genéricas en herramientas específicas de propósito (`analyze_document` → `extract_data_points` + `summarize_content` + `verify_claim_against_source`) y auditar system prompts por fugas de keywords.

### 2.3 Distribución de herramientas y tool_choice

Dar a un agente 18 herramientas en vez de 4–5 degrada de forma medible la fiabilidad de selección, y los agentes con herramientas fuera de su especialización tienden a usarlas mal (un agente de síntesis intentando búsquedas web). La corrección es **acceso acotado a herramientas** más un pequeño número de **herramientas transversales acotadas** para necesidades frecuentes (por ejemplo, `verify_fact` conectado al agente de síntesis para que no tenga que hacer round-trip por el coordinador). `tool_choice` tiene cuatro valores: `auto`, `any`, `{"type": "tool", "name": "..."}` y `none`, cubiertos en detalle abajo.

### 2.5 Herramientas integradas (Read/Write/Edit/Bash/Grep/Glob)

`Grep` busca contenido de archivos (regex); `Glob` coincide con *rutas* de archivos (`**/*.test.tsx`). `Read`/`Write` son operaciones de archivo completo; `Edit` realiza reemplazo de una cadena única y **falla** si `old_string` no es único o el archivo no se ha leído en la sesión actual; en ese punto el fallback correcto es `Read` y luego `Write`. Construye comprensión incrementalmente: `Grep` para puntos de entrada, `Read` para seguir imports, luego traza uso enumerando nombres exportados y haciendo `Grep` de cada uno.

## Escribir una gran descripción de herramienta

La guía "Define tools" de Anthropic es inequívoca: **las descripciones detalladas son, con diferencia, el factor más importante en el rendimiento de herramientas**, con un objetivo de "al menos 3–4 frases por descripción de herramienta, más si la herramienta es compleja." ([Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools))

### Anatomía de una descripción (los 5 elementos)

1. **Qué hace la herramienta**: la acción, en una frase.
2. **Cuándo usarla (y cuándo no)**: la declaración de límites que desambigua frente a herramientas vecinas.
3. **Inputs**: qué significa cada parámetro, formatos aceptados, ejemplos (`AAPL` para ticker), requerido frente a opcional.
4. **Outputs**: qué devuelve la herramienta y qué deliberadamente no devuelve.
5. **Advertencias / limitaciones**: rate limits, frescura, alcance regional, unidades.

Más el array opcional `input_examples`: inputs de ejemplo validados contra schema que ayudan con formas de parámetros complejas o anidadas. Los ejemplos cuestan ~20–50 tokens para inputs simples y ~100–200 para objetos anidados.

### Ejemplos antes/después (malo frente a bueno)

Malo: el ejemplo canónico de Anthropic:

```json
{
  "name": "get_stock_price",
  "description": "Gets the stock price for a ticker.",
  "input_schema": {
    "type": "object",
    "properties": { "ticker": { "type": "string" } },
    "required": ["ticker"]
  }
}
```

Bueno: el contraejemplo canónico de Anthropic:

```json
{
  "name": "get_stock_price",
  "description": "Retrieves the current stock price for a given ticker symbol. The ticker symbol must be a valid symbol for a publicly traded company on a major US stock exchange like NYSE or NASDAQ. The tool will return the latest trade price in USD. It should be used when the user asks about the current or most recent price of a specific stock. It will not provide any other information about the stock or company.",
  "input_schema": {
    "type": "object",
    "properties": {
      "ticker": {
        "type": "string",
        "description": "The stock ticker symbol, e.g. AAPL for Apple Inc."
      }
    },
    "required": ["ticker"]
  },
  "input_examples": [
    { "ticker": "AAPL" },
    { "ticker": "MSFT" }
  ]
}
```

La buena descripción le dice a Claude *qué*, *cuándo*, *qué devuelve* y *qué no devuelve*: exactamente los cuatro huecos que hacen que `get_customer` se elija en vez de `lookup_order` en Sample Question 2.

## Nomenclatura y descomposición de herramientas

### Dividir una herramienta demasiado amplia

Una sola herramienta `analyze_document` con una descripción vaga obliga a Claude a adivinar. Descompón en herramientas específicas de propósito cuyos nombres *son* el desambiguador:

| Original | Reemplazo |
| --- | --- |
| `analyze_document` | `extract_data_points`, `summarize_content`, `verify_claim_against_source` |
| `fetch_url` (genérico) | `load_document` (valida que la URL sea un documento) |
| `analyze_content` (solapa web + doc) | `extract_web_results` (solo web) |

La guía [writing-tools-for-agents](https://www.anthropic.com/engineering/writing-tools-for-agents) de Anthropic hace el punto inverso: evita proliferación de una herramienta por endpoint (`list_users`, `list_events`, `create_event`) cuando una `schedule_event` consolidada encaja mejor con el workflow real. La regla es *una herramienta por subdivisión natural de una tarea*, no una herramienta por llamada de API.

### Renombrar para claridad

Usa **prefijos de servicio** (`github_list_prs`, `slack_send_message`, `asana_search`) cuando el mismo agente tenga herramientas de varios sistemas. Anthropic señala explícitamente que la namespacing por prefijo frente a sufijo tiene efectos medibles en la precisión de evaluación. Renombra herramientas que se solapan en significado hasta que cada nombre sea inequívoco por sí solo: `analyze_content` y `analyze_document` son una colisión de manual; `extract_web_results` y `summarize_pdf` no lo son.

## Distribuir herramientas entre agentes

### Por qué menos herramientas = mejor selección

Las evaluaciones MCP de Anthropic en Opus 4 midieron que la precisión de selección de herramientas se desploma a medida que crece el conteo: 50+ herramientas cargadas al inicio dieron 49% de precisión, recuperándose a 74% solo cuando [Tool Search](https://code.claude.com/docs/en/agent-sdk/tool-search) habilitó carga diferida. El encuadre de la guía "18 vs 4–5" coincide con esta curva: cada herramienta adicional aumenta la probabilidad de enrutamiento erróneo y quema contexto (50 definiciones pueden costar 10–20K tokens).

La conclusión arquitectónica: da a cada subagente el conjunto más pequeño de herramientas que le permita terminar su rol, y deja que el coordinador sostenga la superficie mayor. Cuando un toolset debe seguir siendo grande, habilita Tool Search para que solo 3–5 definiciones relevantes se materialicen por turno.

### Herramientas transversales acotadas

El patrón del agente de síntesis de Sample Question 9 es el caso canónico del examen. La síntesis necesita legítimamente *algo* de lookup de hechos (85% de sus verificaciones son simples), pero darle todo el toolset de búsqueda web la sobreaprovisiona. La corrección es una sola herramienta `verify_fact` **restringida**: inputs estrechos, outputs estrechos, sin fetch general. Maneja el caso común, mientras el 15% de investigaciones complejas sigue pasando por el coordinador hacia el agente dedicado de búsqueda web. Es el principio de mínimo privilegio aplicado a herramientas.

### Tabla de configuración tool_choice

| `tool_choice` | Comportamiento | Usar cuando |
| --- | --- | --- |
| `{"type": "auto"}` | Por defecto con herramientas presentes. Claude decide si llamar alguna herramienta, posiblemente varias en paralelo. | Bucles de agentes generales donde el modelo debe razonar si las herramientas hacen falta. |
| `{"type": "any"}` | Claude debe llamar alguna herramienta (cualquiera de las provistas). No se emite texto conversacional. | Agentes de salida estructurada donde las respuestas de texto libre son un error. Combinar con `strict: true` para inputs garantizados por schema. |
| `{"type": "tool", "name": "extract_metadata"}` | Fuerza a Claude a llamar una herramienta específica. El mensaje del asistente se prellena como un bloque `tool_use`. | Pipelines de "ejecuta X primero" (por ejemplo `extract_metadata` antes de enriquecimiento). Procesar pasos posteriores en turnos de seguimiento. |
| `{"type": "none"}` | Por defecto cuando no se proveen herramientas. Claude no puede llamar herramientas. | Turnos de chat puro dentro de una sesión equipada con herramientas. |
| `disable_parallel_tool_use: true` | Modificador de cualquiera de los anteriores. Con `auto`, Claude llama *como máximo una* herramienta; con `any`/`tool`, exactamente una. | Cuando herramientas downstream tienen dependencias de orden o rate limits. |

Tres restricciones para memorizar:

- `tool_choice: any` y el modo de herramienta forzada **no son compatibles con extended thinking**: solo `auto` y `none` funcionan en ese modo.
- Cambiar `tool_choice` invalida bloques de mensajes cacheados bajo prompt caching (herramientas y system prompt permanecen cacheados).
- `disable_parallel_tool_use` debe ponerse en la solicitud que devuelve el bloque `tool_use`; ponerlo en un seguimiento no tiene efecto retroactivo.

Referencias: [Define tools — Forcing tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools#forcing-tool-use) y [Parallel tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use).

## Chuleta de herramientas integradas

La superficie integrada actual de Claude Code ([Tools reference](https://docs.claude.com/en/docs/claude-code/tools-reference)):

| Herramienta | Usar cuando | Evitar cuando | Fallback / nota |
| --- | --- | --- | --- |
| `Read` | Cargar una ruta de archivo conocida; ver imágenes, PDFs, notebooks. Requerida antes de cualquier `Edit` o `Write` a un archivo existente. | Listar un directorio (usa `Bash ls` o `Glob`). | Errores en archivos demasiado grandes: reintentar con `offset`/`limit`. |
| `Write` | Crear archivos nuevos; sobrescribir después de un `Read`. | Ediciones dirigidas dentro de un archivo grande (usa `Edit`). | Falla en archivos existentes no leídos en la sesión. |
| `Edit` | Reemplazo dirigido de una cadena única en un archivo leído previamente. | La cadena ancla no es única, o el archivo cambió en disco después de leerlo. | Leer otra vez y luego `Write` del contenido completo actualizado; o definir `replace_all: true` si realmente quieres todas las ocurrencias. |
| `Bash` | Ejecutar scripts, package managers, git, formatters. Procesos largos vía `run_in_background: true`. | Leer o escribir archivos (usa `Read`/`Write`/`Edit` para que permisos y elegibilidad de edición apliquen). | Timeout por defecto de 2 min, elevable a 10 min; salida limitada a 30K chars (archivo escrito para el resto). |
| `Grep` | Búsqueda de contenido en el codebase (nombres de funciones, strings de error, imports). Basado en ripgrep: escapa metacaracteres regex. | Localizar archivos solo por patrón de nombre (usa `Glob`). | Respeta `.gitignore`; pasa una ruta explícita para saltarlo. Usa `multiline: true` para cruzar límites de línea. |
| `Glob` | Coincidencia de patrones de nombre de archivo: `**/*.test.tsx`, `src/**/*.ts`. | Buscar dentro del contenido de archivos (usa `Grep`). | Limitado a 100 resultados, ordenado por mtime; no respeta `.gitignore` por defecto. |
| `WebFetch` | Extraer respuestas desde una URL conocida mediante un modelo extractor pequeño. | Necesitar HTML raw o selectores específicos (usa `Bash curl`). | Con pérdida por diseño: re-fetch con prompt de extracción más específico si hace falta. Cacheado 15 min por URL. |
| `WebSearch` | Descubrir URLs por consulta; hasta 8 búsquedas backend por llamada; soporta `allowed_domains` *o* `blocked_domains` (no ambos). | Leer las páginas resultado (encadena `WebFetch` después). | La regla de permiso no tiene especificador: allow/deny de toda la herramienta. |
| `NotebookEdit` | Modificar una celda de Jupyter por `cell_id` (`replace`, `insert`, `delete`). | Reemplazo de texto entre celdas. | Las reglas de permiso usan el formato de ruta `Edit(...)`. |
| `Agent` | Crear un subagente acotado. | Operaciones rutinarias que el padre puede hacer directamente. | El frontmatter `tools` / `disallowedTools` del subagente acota su superficie. |
| `Task*` (`TaskCreate` / `TaskList` / `TaskUpdate`) | Seguimiento de trabajo multipaso en sesiones interactivas. | Tareas triviales one-shot. | `TodoWrite` es el equivalente de modo headless. |

El patrón relevante para el examen es el **bucle de recuperación de fallo de Edit**: `Edit` requiere que el ancla aparezca exactamente una vez en un archivo ya leído en esta sesión. Cuando el ancla se repite, común en archivos de config, líneas de import repetidas o boilerplate, la respuesta correcta es `Read` (archivo completo) → `Write` (archivo completo actualizado), no trucos regex cada vez más creativos.

## Puntos de enfoque para el examen

- "Both tools have minimal descriptions" → la primera corrección es **expandir las descripciones** (inputs, ejemplos, casos límite, límites). Few-shot prompts, capas de routing y consolidación de herramientas son distractores como primer paso. (Sample Q2.)
- "Synthesis agent needs frequent simple verifications" → dale una **herramienta `verify_fact` acotada**; no le entregues todo el toolset de búsqueda web. Batching y caching especulativo son incorrectos. (Sample Q9.)
- `tool_choice: any` ≠ "cualquier herramienta que yo quiera": significa *Claude debe llamar alguna herramienta*, no texto libre.
- El uso forzado de herramientas (`{"type": "tool", "name": "..."}`) no es compatible con extended thinking.
- `Glob` encuentra *archivos*, `Grep` encuentra *contenido*. `**/*.test.tsx` es un patrón Glob.
- Fallos de `Edit` por anclas no únicas → `Read` + `Write`, no reintentos repetidos de `Edit`.
- El tamaño del toolset importa: 18 herramientas son demasiadas para un agente; 4–5 es el objetivo. Más allá de 30–50 en la sesión, habilita Tool Search.
- `disable_parallel_tool_use: true` pertenece a la solicitud que emite el bloque `tool_use`.

## Referencias

- [Define tools — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- [Tool reference — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference)
- [Parallel tool use — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use)
- [Strict tool use — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use)
- [Writing effective tools for AI agents — Anthropic Engineering](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Tools reference — Claude Code Docs](https://docs.claude.com/en/docs/claude-code/tools-reference)
- [Scale to many tools with tool search — Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/tool-search)
- [Configure permissions — Claude Code Docs](https://code.claude.com/docs/en/permissions)
- [anthropics/skills — claude-api/shared/tool-use-concepts.md](https://github.com/anthropics/skills/blob/main/skills/claude-api/shared/tool-use-concepts.md)
