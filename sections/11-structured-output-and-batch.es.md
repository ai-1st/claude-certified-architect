---
title: "Sección 11 — Salida estructurada mediante tool_use, procesamiento Batch y revisión multipaso"
linkTitle: "11. Salida estructurada y Batch"
weight: 11
description: "Dominios 4.3, 4.5, 4.6 — JSON Schema con campos nullable/optional, bucles de validación-reintento y compromisos de Message Batches API."
---

## Qué cubre esta sección

Tres decisiones arquitectónicas relacionadas: **cómo** hacer que Claude emita salida parseable por máquina (`tool_use` + JSON schemas, `strict: true`, la función más nueva Structured Outputs), **cuándo** mover trabajo a **Message Batches API** por coste y throughput (y cuándo no), y **por qué** un revisor independiente o una división por archivo + integración supera la autorevisión.

El examen evalúa las tres con escenarios concretos: checks pre-merge frente a informes nocturnos, revisión de una sola pasada frente a revisión por archivo, prompts JSON-only frente a `tool_use`. Las respuestas correctas vienen de un principio: **ajustar la técnica a las restricciones de latencia, fiabilidad y presupuesto de atención del workload**.

## Material fuente (de la guía oficial)

### 4.3 Salida estructurada vía tool_use y JSON schemas
- `tool_use` con JSON schemas es el enfoque más fiable para salida garantizada conforme a schema; elimina errores de sintaxis JSON.
- Modos `tool_choice`: `"auto"` (el modelo puede devolver texto), `"any"` (debe llamar una herramienta, puede elegir cuál) o forzado `{"type": "tool", "name": "..."}`.
- Los schemas estrictos eliminan solo errores de **sintaxis**, no errores **semánticos** (line items que no suman el total, valores en campos equivocados, contenido fabricado para datos ausentes).
- Diseño de schema: required frente a optional/nullable, enums con `"other"` + string de detalle para extensibilidad, `"unclear"` para ambigüedad, reglas de normalización de formato en el prompt.

### 4.5 Estrategia de procesamiento batch
- Message Batches API: **50% de ahorro de coste**, ventana de procesamiento de hasta **24 horas**, **sin SLA de latencia**.
- Buena para workloads no bloqueantes y tolerantes a latencia (informes nocturnos, auditorías semanales, generación nocturna de tests). Mala para workflows bloqueantes (checks pre-merge).
- Según la guía: Batch API **no** soporta ejecución *agéntica* multiturno de herramientas dentro de una solicitud única; no puedes pausar a mitad de request para ejecutar una herramienta y devolver resultados.
- `custom_id` correlaciona request y response; `custom_id`s fallidos pueden reenviarse después de correcciones (por ejemplo, fragmentar documentos que excedieron contexto).

### 4.6 Revisión multiinstancia y multipaso
- Un modelo que conserva su razonamiento de generación es menos propenso a cuestionar sus propias decisiones, así que la **autorevisión** es estructuralmente débil.
- Instancias independientes de revisión detectan problemas que la autorevisión y extended thinking pasan por alto.
- Las revisiones multiarchivo deben dividirse en **pasadas locales por archivo** más una **pasada de integración entre archivos** para evitar dilución de atención y hallazgos contradictorios.
- Las pasadas de verificación pueden pedir al modelo que autoinforme **confianza por hallazgo** para habilitar enrutamiento calibrado.

## Salida estructurada con tool_use

### Por qué tool_use supera "responde solo con JSON"

Pedir al modelo "respond with JSON only" funciona la mayor parte del tiempo y falla lo suficiente para ser un peligro en producción: prosa suelta, smart quotes, trailing commas, fences markdown, una disculpa inicial. `tool_use` elimina toda esa clase de fallos: el modelo no emite texto libre, emite un bloque `tool_use` estructurado cuyo `input` está garantizado como objeto JSON.

Añadir `"strict": true` (muestreo restringido por gramática) garantiza además que el JSON cumpla los tipos, enums y campos requeridos del schema. Strict mode está soportado en Opus 4.7/4.6/4.5, Sonnet 4.6/4.5 y Haiku 4.5 ([Strict tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/strict-tool-use)). La función más nueva **Structured Outputs** (`output_config.format = { "type": "json_schema", "schema": ... }`) entrega la misma garantía sin definir una herramienta falsa ([Structured outputs](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)); ambas comparten el mismo subconjunto de JSON Schema y la misma advertencia de errores semánticos.

```python
import anthropic

extract_invoice = {
    "name": "extract_invoice",
    "description": "Extract structured invoice data from the document.",
    "strict": True,
    "input_schema": {
        "type": "object",
        "additionalProperties": False,
        "required": ["vendor", "invoice_number", "line_items",
                     "stated_total", "calculated_total", "currency", "category"],
        "properties": {
            "vendor": {"type": "string"},
            "invoice_number": {"type": "string"},
            "issue_date": {"type": ["string", "null"], "format": "date"},
            "currency": {"type": "string", "enum": ["USD", "EUR", "GBP", "other"]},
            "currency_other": {"type": ["string", "null"]},
            "category": {"type": "string",
                         "enum": ["saas", "hardware", "travel",
                                  "professional_services", "other", "unclear"]},
            "category_other": {"type": ["string", "null"]},
            "line_items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "additionalProperties": False,
                    "required": ["description", "amount"],
                    "properties": {
                        "description": {"type": "string"},
                        "amount": {"type": "number"}
                    }
                }
            },
            "stated_total":     {"type": "number"},
            "calculated_total": {"type": "number"}
        }
    }
}

client = anthropic.Anthropic()
resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    tools=[extract_invoice],
    tool_choice={"type": "tool", "name": "extract_invoice"},
    messages=[{"role": "user", "content": INVOICE_TEXT}],
)
data = next(b.input for b in resp.content if b.type == "tool_use")
```

### Chuleta de tool_choice

| `tool_choice`                          | Comportamiento del modelo                             | Usar cuando                                                |
| -------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| `{"type": "auto"}`                     | Puede responder con texto *o* llamar cualquier herramienta | Bucles de agentes donde las respuestas de texto son válidas |
| `{"type": "any"}`                      | **Debe** llamar una herramienta; elige cuál           | Extracción multi-schema donde el tipo de documento es desconocido |
| `{"type": "tool", "name": "extract"}`  | **Debe** llamar esa herramienta específica            | Forzar un paso de extracción conocido antes de enriquecimiento |
| `{"type": "none"}`                     | No puede llamar ninguna herramienta                   | Deshabilitar herramientas por un turno sin reconstruir la solicitud |

Combina cualquiera con `"disable_parallel_tool_use": true` si necesitas como máximo una llamada de herramienta por turno: útil cuando el código downstream espera un único payload estructurado, o cuando llamadas concurrentes violarían invariantes de orden ([Parallel tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/parallel-tool-use)).

### Patrones de diseño de schema

- **Required frente a optional / nullable.** Los campos requeridos fuerzan al modelo a rellenarlos, causando fabricación cuando la fuente realmente carece del dato. Marca campos opcionalmente presentes como `"type": ["string", "null"]` para que el modelo pueda devolver `null` en vez de alucinar.
- **Enum con `"other"` + detalle.** Los enums cerrados son frágiles. Un enum `category` `[..., "other"]` + hermano `category_other: string` conserva información para categorías nuevas sin romper código downstream.
- **`"unclear"` para ambigüedad.** Un valor enum `"unclear"` permite al modelo señalar baja confianza; enruta esas filas a revisión humana.
- **Campos autovalidantes.** Emitir tanto `stated_total` (del documento) como `calculated_total` (suma de `line_items`); un check de postprocesamiento detecta errores semánticos que schemas estrictos no pueden.

### Subconjunto JSON Schema soportado

El compilador de strict-mode / structured-outputs acepta un subconjunto de JSON Schema. La mayoría de fallos "¿por qué mi schema da 400?" vienen de esta lista:

- **Soportado:** object/array/string/integer/number/boolean/null, `enum` (solo primitivos), `const`, `anyOf`/`allOf` (limitado), `$ref`/`$def` internos, `default`, `required`, `additionalProperties: false`, formatos (`date-time`, `date`, `email`, `uri`, `uuid`, …), `minItems` de 0 o 1.
- **No soportado:** `$ref` externo, schemas recursivos, tipos complejos en enums, constraints numéricas (`minimum`/`maximum`/`multipleOf`), longitud de strings, `additionalProperties` != `false`.
- **Regex:** quantifiers, character classes y groups funcionan; backreferences, lookarounds y word boundaries no.

### Lo que tool_use NO resuelve

Los schemas estrictos garantizan *parseabilidad*, no *corrección*. Emitirán sin problema una factura válida por schema donde los line items suman $812 mientras `stated_total` dice $1,200, o donde el nombre del proveedor aparece en `invoice_number`. Detectar esto requiere validación a nivel de aplicación (por ejemplo el truco de `calculated_total`), bucles retry-with-error-feedback (Domain 4.4) o una pasada de revisor independiente (Domain 4.6).

## Message Batches API

### Qué obtienes / qué cedes

| Dimensión                  | Messages API sincrónica           | Message Batches API                                          |
| -------------------------- | --------------------------------- | ------------------------------------------------------------ |
| Precio                     | Estándar                          | **50% off** input + output (por ejemplo Sonnet 4.6 $1.50/$7.50 por MTok) |
| Latencia                   | Segundos                          | Hasta **24 horas**; la mayoría de batches termina en 1 hora; sin SLA |
| Máx. requests / batch      | n/a                               | **100,000** requests **o 256 MB**, lo que ocurra primero     |
| Tool use                   | Bucle agéntico completo           | Se pueden definir herramientas; **no se soporta ejecución multiturno de herramientas a mitad de request** |
| Streaming                  | Sí                                | **No soportado** (los resultados se extraen cuando termina el batch) |
| Prompt caching             | Sí                                | Sí (best-effort; combinar con caché de 1 hora para contexto compartido) |
| Retención de resultados    | n/a                               | 29 días                                                      |
| Modelos                    | Todos los activos                 | Todos los activos                                            |

Fuentes: [Batch processing](https://docs.anthropic.com/en/docs/build-with-claude/message-batches).

### Cuándo usar / cuándo no

**Usa Batches API cuando:**

- El workload no es bloqueante: informes nocturnos de deuda técnica, auditorías semanales, generación de pruebas de regresión, backlog de moderación de contenido, evaluaciones masivas.
- Los volúmenes son suficientemente grandes para que el descuento del 50% importe materialmente.
- Cada request es **autocontenida** (no necesitas inyectar resultados de herramientas entre turnos).

**No la uses cuando:**

- Un humano espera el resultado (checks pre-merge, chat, UIs interactivas).
- El workflow necesita que el modelo llame herramientas, vea resultados y continúe razonando en la misma request.
- Necesitas streaming o latencia subminuto.

Esta es exactamente la estructura de **Sample Question 11**: mover el informe nocturno de deuda técnica a batch (A), mantener el check pre-merge bloqueante en la API sincrónica. Las respuestas incorrectas implican esperar que batches "normalmente terminen suficientemente rápido" o añadir un timeout fallback; ninguna es aceptable cuando el SLA es "el desarrollador está mirando la pantalla."

### custom_id y manejo de fallos

Cada request en un batch lleva un `custom_id` (1–64 chars, `[a-zA-Z0-9_-]`). Es el **único** mecanismo para correlacionar resultados con inputs, ya que el orden de salida no está garantizado. Incluye suficiente metadata en `custom_id` para buscar el registro original: `doc_42891-v3-2026q1` está bien; `req_001` te perseguirá.

Al recuperar resultados, cada entrada tiene un `result` de `succeeded`, `errored`, `canceled` o `expired`. Patrón estándar de fallo:

1. Extrae el stream de resultados y particiona por `result.type`.
2. Para entradas `errored`, inspecciona el código de error: fragmenta documentos demasiado grandes, corrige params inválidos y reenvía **solo los `custom_id`s fallidos** como un nuevo batch más pequeño.
3. Para entradas `expired` (el batch no terminó en 24h), reenvía con tamaño menor o fuera de horas pico.

### Ejemplo trabajado: extracción nocturna de 100 documentos

```python
from anthropic import Anthropic
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

client = Anthropic()

requests = [
    Request(
        custom_id=f"invoice-{doc.id}",
        params=MessageCreateParamsNonStreaming(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            tools=[extract_invoice],
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=[{"role": "user", "content": doc.text}],
        ),
    )
    for doc in documents
]

batch = client.messages.batches.create(requests=requests)

while True:
    batch = client.messages.batches.retrieve(batch.id)
    if batch.processing_status == "ended":
        break
    time.sleep(60)

for entry in client.messages.batches.results(batch.id):
    if entry.result.type == "succeeded":
        msg = entry.result.message
        payload = next(b.input for b in msg.content if b.type == "tool_use")
        store(entry.custom_id, payload)
    else:
        log_failure(entry.custom_id, entry.result)
```

Observa la combinación: `tool_use` para salida segura por schema **dentro** de una request batch. La extracción single-shot no tiene llamadas a herramientas a mitad de request, así que la restricción de Batch API no muerde.

### Matemática de SLA: con qué frecuencia enviar

Si el SLA de negocio es "resultados en N horas" y Batch API puede tardar hasta 24 horas, debes enviar con una cadencia tal que **demora de envío + 24h de procesamiento ≤ N**. Para N = 30h, enviar cada **4 horas** (peor caso: 4h de espera hasta el siguiente batch, más 24h de procesamiento = 28h, cómodo dentro de 30h). Para 26h, cada hora. Para 24h, no puedes cumplirlo solo con Batches API: cae a llamadas sincrónicas o acepta incumplir SLAs.

## Arquitectura de revisión multiinstancia y multipaso

### La trampa de la autorevisión

Cuando la misma instancia de Claude que escribió el código también lo revisa, el razonamiento de generación sigue en contexto: el modelo trata sus decisiones anteriores como premisas en vez de hipótesis a desafiar. Incluso una instrucción explícita "critique your previous response" es más débil que una **instancia fresca** sin compromiso previo que defender.

El sistema Code Review de Anthropic refleja esto: despacha **múltiples agentes especializados en paralelo** (prompts distintos para lógica, seguridad, edge cases) y ejecuta un **paso de verificación** contra comportamiento real del código para filtrar falsos positivos antes de publicar hallazgos ([Claude Code Code Review](https://claude.com/blog/code-review)). Divide el trabajo, usa contexto independiente y reconcilia al final.

### Patrón por archivo + integración entre archivos

Esta es la respuesta correcta a **Sample Question 12** (PR de 14 archivos con profundidad inconsistente y hallazgos contradictorios):

```
                    ┌────────────────────────────┐
                    │  Per-file local passes     │
PR (14 files) ──►  │  - one Claude call per file │  ──► findings_local[]
                    │  - focused prompt           │
                    │  - no other files in ctx    │
                    └────────────────────────────┘
                                  │
                                  ▼
                    ┌────────────────────────────┐
                    │  Integration pass           │
                    │  - all diffs + module map   │  ──► findings_integration[]
                    │  - cross-file data flow     │
                    │  - API contracts, types     │
                    └────────────────────────────┘
                                  │
                                  ▼
                          dedupe + rank + post
```

Las pasadas por archivo dan a cada archivo el **mismo presupuesto de atención**, eliminando el modo de fallo "profundo en archivo 1, superficial en archivo 14". La pasada de integración se prompta específicamente para preocupaciones entre archivos (drift de firma caller/callee, cambios de schema compartidos, invariantes transaccionales), así no rehace lo que ya hicieron las pasadas locales.

### Patrón de revisor independiente

Para artefactos single de alto riesgo (una migración generada, un informe para cliente), ejecuta **dos llamadas a Claude**:

1. **Generador**: produce el artefacto con razonamiento completo.
2. **Revisor**: una *nueva* request, *nuevo* system prompt, *sin* transcripción del generador, dado solo el artefacto y la spec. Su único trabajo es encontrar defectos.

La falta de contexto del revisor es la función, no un bug: no puede racionalizar decisiones que nunca tomó.

### Pasadas de verificación anotadas con confianza

Haz que el revisor devuelva hallazgos con un enum explícito `confidence` (`"high"`, `"medium"`, `"low"`) y una `rationale` de una línea:

```json
{
  "findings": [
    {"severity": "high", "confidence": "high",
     "file": "billing.py", "line": 142,
     "issue": "Off-by-one in proration when subscription starts on month boundary",
     "rationale": "Integration test billing_test.py:88 covers mid-month only."}
  ]
}
```

Luego enruta por confianza: `high` confidence + `high` severity va directo al PR como comentario bloqueante; hallazgos `low` confidence van a una cola de triage o disparan una pasada de desempate. Este es el **enrutamiento de revisión calibrado** que referencia la skill oficial.

## Matriz de decisión: qué técnica para qué job

| Workload                                | Necesidad de latencia | Profundidad de revisión | Stack recomendado                                                       |
| --------------------------------------- | --------------------- | ----------------------- | ----------------------------------------------------------------------- |
| Check pre-merge bloqueante              | Segundos              | Por archivo + integración | Sync Messages API + `tool_use(strict)` + revisión multipaso             |
| Informe nocturno de deuda técnica       | Horas                 | Por archivo + integración | **Batches API** + `tool_use(strict)` + revisión multipaso               |
| Extracción de campos de 100k documentos | Nocturna              | QC por muestra          | **Batches API** + `tool_choice` forzado + campos autovalidantes         |
| Chat interactivo con paso de extracción | Segundos              | Ninguna                 | Sync Messages API + `tool_choice` forzado + campos nullable             |
| QC de documento regulatorio             | Minutos–horas         | Revisor independiente   | Sync (o batch) + split generador/revisor + enrutamiento por confianza   |
| Auditoría semanal cross-repo            | Días                  | Solo por repo           | **Batches API** + `tool_use` + omitir pasada de integración             |

## Puntos de enfoque para el examen

- **`tool_use` frente a "respond with JSON":** la respuesta correcta siempre empuja hacia `tool_use` / schemas estrictos para parseabilidad garantizada. Prompts bare-JSON son distractor.
- **Selección de `tool_choice`:** `"any"` para tipo de documento desconocido entre varias herramientas de extracción; forzado `{"type":"tool","name":"..."}` para garantizar que una herramienta específica corra antes de enriquecimiento; `"auto"` solo cuando respuestas de texto son legítimas.
- **Diseño de schema:** nullable cuando la fuente puede carecer de datos (previene fabricación); `"other"` + detalle para extensibilidad; `"unclear"` para ambigüedad; campos autovalidantes para checks semánticos.
- **Encaje de Batches API:** solo workloads **no bloqueantes, tolerantes a latencia, ≤24h**. Checks pre-merge son el mal encaje canónico. 50% off, límite 100k req / 256 MB, resultados válidos 29 días, sin streaming, sin ejecución de herramientas a mitad de request.
- **`custom_id`:** obligatorio para correlación; reenvía solo IDs fallidos después de corregir la causa.
- **Revisión multipaso:** por archivo local + integración entre archivos supera una sola pasada en PRs multiarchivo; instancias independientes superan autorevisión; hallazgos anotados con confianza habilitan enrutamiento.
- **Ideas equivocadas que evitar:** "una ventana de contexto más grande arregla dilución de atención" (no), "tres pasadas completas + voto mayoritario" (suprime bugs reales detectados intermitentemente), "timeout-fallback de batch a sync" (sobrecomplejo; elige la API correcta por workload).

## Referencias

- [Tool use overview](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- [Define tools](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/define-tools)
- [Strict tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/strict-tool-use)
- [Parallel tool use & `disable_parallel_tool_use`](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/parallel-tool-use)
- [Structured outputs (`output_config.format` JSON Schema)](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)
- [Message Batches API — batch processing](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Create / list / retrieve Message Batches (API reference)](https://docs.anthropic.com/en/api/creating-message-batches)
- [Prompt caching with batches (1-hour cache duration)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Claude Code: multi-agent Code Review system](https://claude.com/blog/code-review)
- [Claude Code Code Review docs](https://docs.anthropic.com/en/docs/claude-code/code-review)
