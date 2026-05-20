---
title: "Sección 10 — Ingeniería de prompts: criterios explícitos, few-shot y bucles de validación"
linkTitle: "10. Ingeniería de prompts"
weight: 10
description: "Dominios 4.1, 4.2, 4.4 — criterios explícitos sobre umbrales, few-shot para escenarios ambiguos y revisión multipaso para diffs grandes."
---

## Qué cubre esta sección

Tres técnicas a nivel de prompt que el examen Foundations trata como centrales para llevar una feature de Claude de calidad demo a calidad producción: **criterios categóricos explícitos** en vez de instrucciones vagas, **2–4 ejemplos few-shot dirigidos** y un **bucle de validación-reintento** con señales de autocorrección incorporadas en el schema. Domain 4.3 (tool use + JSON schemas) se cubre en la Sección 11; esta sección cierra la brecha *semántica* que los schemas estrictos no cubren.

## Material fuente (de la guía oficial)

### 4.1 Criterios explícitos sobre instrucciones vagas

- Los criterios categóricos explícitos superan instrucciones vagas: *"flag comments only when claimed behavior contradicts actual code behavior"* rinde mejor que *"check that comments are accurate"*. Instrucciones generales como *"be conservative"* o *"only report high-confidence findings"* **no** mejoran la precisión; altas tasas de falsos positivos en una categoría erosionan la confianza en **todas** las categorías.
- Habilidades: escribir reglas específicas report-when / skip-when por categoría; deshabilitar temporalmente categorías con altos FP mientras se mejoran; definir criterios explícitos de severidad con ejemplos concretos de código por nivel.

### 4.2 Few-shot prompting

- Los ejemplos few-shot son la técnica **más efectiva** para salida formateada, accionable y consistente cuando las instrucciones detalladas solas producen resultados inconsistentes. Impulsan el manejo de casos ambiguos (selección de herramientas, escalación), habilitan generalización a patrones nuevos y reducen alucinación en extracción sobre estructuras documentales variadas.
- Habilidades: 2–4 ejemplos dirigidos que muestren *razonamiento* para la acción elegida frente a alternativas plausibles; ejemplos que demuestren el formato de salida deseado (ubicación, problema, severidad, fix sugerido); ejemplos que distingan patrones aceptables de problemas reales; ejemplos que cubran cada variante estructural del input.

### 4.4 Validación, reintento y bucles de feedback

- Reintento con feedback de error: añade el error de validación **específico** al siguiente prompt para guiar la autocorrección. Reintentar no sirve cuando la información requerida está **ausente de la fuente**. Distingue errores de validación **semánticos** (valores no suman, campo equivocado) de errores de **sintaxis de schema** (ya eliminados por tool use).
- Habilidades: solicitudes de seguimiento que incluyan documento original + extracción fallida + errores específicos; añadir `detected_pattern` para habilitar análisis de falsos positivos; diseñar autocorrección extrayendo `calculated_total` junto a `stated_total`; añadir booleanos `conflict_detected` para datos inconsistentes en la fuente.

## Escribir criterios explícitos

### Vago frente a específico: reescrituras antes/después

El movimiento más importante de prototipo a producción es reemplazar instrucciones borrosas ("be careful", "only report high-confidence findings", "be conservative") por **criterios categóricos que definan comportamiento**. La confianza autoinformada está mal calibrada: un modelo que se equivoca con confianza en casos difíciles no se salvará con *"only report high-confidence findings"*; seguirá reportando las mismas cosas incorrectas.

| Vago (lo que los equipos escriben primero) | Específico (lo que parecen los prompts de producción) |
| --- | --- |
| "Check that comments are accurate." | "Flag a comment **only** when its claimed behaviour contradicts the actual code behaviour. Do **not** flag comments that are merely vague, outdated stylistically, or under-detailed." |
| "Find security issues." | "Report only: SQL string concatenation reaching a query call, `eval`/`exec` on attacker-controlled input, secrets in source, missing auth checks on routes under `/api/admin/*`. Skip: defensive `assert` patterns, hardcoded test fixtures, anything in `tests/`, `fixtures/`, `examples/`." |
| "Be conservative when escalating." | "Escalate when (a) the customer asks for a policy exception, (b) the claim value > $X, or (c) more than 2 prior contacts on the same case. Resolve autonomously when the photo evidence matches a standard damage SKU and value < $X." |
| "Only report high-confidence findings." | "Report a finding only if you can quote the exact line from the source and name the specific rule it violates. If you cannot quote a line, do not report." |

Observa el patrón: cada versión "buena" responde dos preguntas: *¿qué cuenta como positivo?* y *¿qué cuenta como no-problema que debo omitir?* Sample Question 3 de la guía hace el mismo punto en forma de agente: 55% de resolución en primer contacto mejora añadiendo **criterios explícitos de escalación con ejemplos few-shot**, no añadiendo un score de confianza autoinformado.

### Definiciones de severidad con ejemplos de código

Para cualquier clasificador que emita un campo `severity`, define cada nivel con un ejemplo concreto de código *dentro del prompt*. Si no, distintas ejecuciones colapsan la diferencia entre `low` y `medium` en ruido.

```xml
<severity_definitions>
  <critical>Data loss, auth bypass, or RCE in prod.
    Example: db.query("SELECT * FROM users WHERE id=" + req.params.id)</critical>
  <high>Bug producing incorrect output for at least one realistic input.
    Example: off-by-one in pagination dropping the last page.</high>
  <medium>Resource leak or correctness issue only under load.
    Example: file handle opened in a loop without `with` / `defer close`.</medium>
  <low>Style or readability only — do NOT include unless explicitly asked.</low>
</severity_definitions>
```

Anclar cada nivel a un ejemplo fuerza clasificación consistente entre ejecuciones y revisores.

### Deshabilitar y reconstruir para categorías con muchos FP

Los falsos positivos en una categoría erosionan la confianza en todas. La guía recomienda un patrón deliberado disable-then-rebuild: cuando "comment accuracy" corre con 40% FP, es mejor eliminar esa categoría por completo que dejarla activa y degradar la confianza en hallazgos de seguridad que sí accionan los desarrolladores. Itera la categoría mala en un prompt lateral hasta que la precisión supere tu umbral y luego reactívala.

### Por qué "be conservative" no funciona

Calibración. Adverbios como "carefully" o "only when sure" no dan al modelo nada que pueda verificar mecánicamente. En cambio, *"do not report a finding unless you can quote the exact source line and name the rule"* es una regla verificable: el modelo puede comprobarla antes de emitir salida.

## Profundización en few-shot prompting

### ¿Cuántos ejemplos?

La guía publicada de Anthropic y la guía de certificación convergen en el mismo número: **2–4 ejemplos dirigidos**. Menos de 2 no establece patrón; más de ~4 quema tokens con rendimientos decrecientes y arriesga que el modelo sobreajuste a rasgos superficiales de los ejemplos. Para prompts con extended-thinking, Anthropic señala específicamente que multishot sigue ayudando, pero debe mostrar *patrones de razonamiento*, no solo pares input→output.

### Anatomía de un ejemplo

Los ejemplos efectivos llevan tres campos, no dos:

1. **Input**: un fragmento que refleje input real de producción.
2. **Razonamiento**: *por qué* la acción elegida es correcta frente a alternativas plausibles.
3. **Output**: exactamente la forma (XML o JSON) que quieres en inferencia.

```xml
<example>
  <input>
    User: "My order from last Tuesday hasn't arrived and I want my money back."
  </input>
  <reasoning>
    This is a delivery exception within standard policy (lost package, &lt; 30 days,
    refund value within auto-approve threshold). No policy exception is being
    requested. Resolve autonomously with refund + reship.
  </reasoning>
  <output>
    {"action": "resolve", "tool": "issue_refund", "escalate": false}
  </output>
</example>
```

### Ejemplos para casos ambiguos, formato y reducción de FP

- **Casos ambiguos.** Elige 2–4 ejemplos de tu eval set donde el modelo se comportó mal e incluye un **par de contraste**: un ejemplo que se parece pero debe manejarse de la forma **opuesta**. Los pares de contraste enseñan generalización en vez de memorización.
- **Consistencia de formato.** Si quieres cada hallazgo como `{location, issue, severity, suggested_fix}`, muestra tres hallazgos exactamente con esa forma. Las instrucciones solas filtran capitalización inconsistente, campos faltantes y prosa final.
- **Reducción de falsos positivos.** Empareja un ejemplo de "issue" con un ejemplo de "acceptable pattern" que comparte rasgos superficiales. Para un revisor de SQL injection, muestra una concat vulnerable *y* una llamada parametrizada etiquetada `not_a_finding`: así evitas que el modelo marque cada `db.query(...)`.
- **Estructuras documentales variadas.** Para facturas con line items desordenados, papers con citas inline frente a bibliografía o metodología en narrativa, da un ejemplo *por variante estructural*. Sin cobertura de cada forma, la extracción devuelve null en formatos no vistos.

### Dónde poner ejemplos: XML tagging

Los docs de prompt engineering de Anthropic son explícitos: envuelve ejemplos en tags XML (`<example>`, `<examples>`, `<input>`, `<output>`) y colócalos **después** de instrucciones y criterios de tarea pero **antes** del input real.

```xml
<task>...explicit criteria here...</task>
<severity_definitions>...</severity_definitions>
<examples>
  <example>...</example>
  <example>...</example>
  <example>...</example>
</examples>
<input>{{actual_document_to_process}}</input>
```

## Validación, reintento y bucles de feedback

### Reintento con feedback de error

El patrón es: validar programáticamente la salida del modelo y, al fallar, enviar un *nuevo* turno que contenga el input original, la salida fallida y el error de validación **específico**. Un "try again" genérico nunca ayuda; el texto de error concreto sí.

```python
from anthropic import Anthropic

client = Anthropic()
MODEL = "claude-opus-4-7"

def extract_with_retry(document: str, max_attempts: int = 2) -> dict:
    messages = [{"role": "user", "content": build_extraction_prompt(document)}]
    for attempt in range(max_attempts + 1):
        resp = client.messages.create(
            model=MODEL, max_tokens=2048,
            tools=[INVOICE_TOOL],
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=messages,
        )
        extraction = next(b.input for b in resp.content if b.type == "tool_use")

        errors = validate(extraction)
        if not errors:
            return extraction
        if not is_recoverable(errors):
            raise ExtractionError(errors, extraction)

        messages.append({"role": "assistant", "content": resp.content})
        messages.append({"role": "user", "content":
            f"Your previous extraction failed validation:\n"
            f"{format_errors(errors)}\n\n"
            f"Re-examine the original document and call extract_invoice again, "
            f"correcting only the listed errors."
        })
    raise ExtractionError(errors, extraction)
```

Tres cosas hacen que este bucle funcione: (1) el turno del asistente se añade *literalmente* (misma regla que el bucle de agente de la Sección 2), (2) el seguimiento nombra los campos fallidos **específicos** y (3) el reintento usa tool use con herramienta forzada, así la forma de respuesta se mantiene estable entre intentos.

### Cuándo el reintento NO ayudará

Los reintentos corrigen **errores de formato y estructura** (nombres de campo equivocados, valores mal ubicados, line items que no suman). No corrigen información genuinamente ausente de la fuente. Si el PDF de factura no contiene tax ID, ningún reintento producirá uno; y un bucle agresivo puede presionar al modelo a alucinar uno para pasar validación. Detecta "ausente en la fuente" en tu validador y detén el bucle con un null estructurado + razón en vez de reintentar.

### Señales de autocorrección en el schema

Incorpora validación *dentro de la propia extracción* para que el modelo enfrente contradicciones durante la generación:

- **`calculated_total` junto a `stated_total`**: el modelo emite ambos; tu validador marca discrepancias. El modelo a menudo se autocorrige en la misma respuesta porque calcular el valor lo obliga a revisar cada line item.
- **Booleanos `conflict_detected`**: para documentos donde dos secciones discrepan (fecha del encabezado frente al pie, conteo de resumen frente a lista de ítems), haz que el modelo emita un booleano por familia de conflicto. Convierte ambigüedad en una señal estructurada sobre la que puedes enrutar.

```json
{
  "line_items": [{"desc": "Widget A", "amount": 150.00},
                 {"desc": "Widget B", "amount": 300.00}],
  "calculated_total": 450.00,
  "stated_total": 500.00,
  "total_discrepancy": true,
  "conflict_detected": false,
  "detected_pattern": "missing_tax_line"
}
```

### `detected_pattern` para analizar descartes

Añade un campo `detected_pattern` (o `rule_id`) a cada hallazgo. Cuando los desarrolladores descarten hallazgos en tu UI, agrupa descartes por `detected_pattern` para ver qué categorías son ruidosas. Sin este campo, tu única señal es la tasa agregada de FP, que te dice *que* algo va mal, no *qué* corregir.

### Errores semánticos frente a sintácticos

Tool use con un JSON schema estricto (Domain 4.3) elimina errores de **sintaxis**: JSON malformado, campos requeridos faltantes, tipos incorrectos. No elimina errores **semánticos**: valores que no suman, fechas fuera del rango del documento, contenido plausible pero incorrecto. El bucle de validación-reintento es la capa que maneja errores semánticos. Trampa común de examen: "añadimos un JSON schema, ¿por qué siguen mal los line items?" Porque los schemas no hacen aritmética.

## Integrarlo todo: workflow de ajuste de precisión

1. **Baseline FP/FN por categoría.** Ejecuta un eval set; agrupa hallazgos por `detected_pattern`; registra precision/recall por categoría, no solo global.
2. **Reescribe instrucciones vagas como criterios explícitos** para las principales categorías de FP: reglas "report when… / skip when…" con ejemplos de código para cada lado.
3. **Añade 2–4 ejemplos few-shot por escenario ambiguo,** incluyendo al menos un **par de contraste** (parece un problema, no lo es) por categoría ruidosa. Envuelve en tags XML `<examples>` colocados después de instrucciones y antes del input real.
4. **Envuelve con un bucle de validación-reintento.** Usa tool use para forzar el schema; valida constraints semánticas (`calculated_total == stated_total`, línea fuente citada requerida); al fallar, añade el error específico y reintenta una vez.
5. **Rastrea `detected_pattern` y descartes.** Ordena por tasa de FP semanalmente; deshabilita temporalmente cualquier categoría sobre tu umbral mientras iteras.
6. **Evalúa antes y después de cada cambio** con la [Evaluation tool](https://console.anthropic.com/docs/en/test-and-evaluate/eval-tool) de Anthropic Console o [Promptfoo](https://www.promptfoo.dev/docs/getting-started/).

## Puntos de enfoque para el examen

- Dado un prompt que dice "be conservative" u "only report high-confidence findings", reconoce que **no** es la corrección adecuada para tasas altas de falsos positivos: criterios categóricos + few-shot sí.
- Elige el número correcto de ejemplos few-shot: **2–4 dirigidos** (no 10, no 1).
- Reconoce que los ejemplos few-shot deben incluir **razonamiento** (por qué esta acción frente a alternativas), no solo input/output, especialmente para casos ambiguos como escalación o selección de herramientas.
- Identifica cuándo un reintento ayudará (error de formato/estructura, line-items que no suman) frente a cuándo no (información genuinamente ausente de la fuente). No reintentes lo segundo.
- Distingue errores de validación **semánticos** de errores de **sintaxis de schema**. Tool use corrige lo segundo; un bucle de validación-reintento corrige lo primero.
- Conoce el rol de campos de autocorrección: `calculated_total` frente a `stated_total`, `conflict_detected`, `detected_pattern`.
- Para Sample Question 3 (escalación de agente de seguros): la respuesta correcta es **criterios explícitos de escalación + ejemplos few-shot**, no scores de confianza autoinformada, no un modelo clasificador separado, no análisis de sentimiento.

## Referencias

- [Prompt engineering overview — Claude API docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — landing page canónica; lista la pila de técnicas (be clear and direct → multishot → chain-of-thought → XML tags → system prompts → prefill → chain prompts) en orden de prioridad.
- [Be clear, contextual, and specific](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/be-clear-and-direct) — guía fundacional de Anthropic detrás de Domain 4.1.
- [Use examples (multishot prompting)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting) — fuente oficial para la recomendación de 2–4 ejemplos, anatomía del ejemplo y envoltura XML.
- [Let Claude think (chain of thought)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/chain-of-thought) and [Use XML tags](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags) — separar razonamiento de salida y evitar que ejemplos, instrucciones e input real se mezclen.
- [Extended thinking tips](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/extended-thinking-tips) — multishot sigue funcionando con extended thinking; preferir instrucciones de alto nivel sobre paso a paso prescriptivo.
- [Increase output consistency (prefill)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/increase-consistency) and [Reduce hallucinations](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-hallucinations) — técnicas detrás de "forzar una llave JSON inicial" y "requerir una línea fuente citada."
- [Evaluate prompts in the developer console](https://www.anthropic.com/news/evaluate-prompts) and [Using the Evaluation tool](https://console.anthropic.com/docs/en/test-and-evaluate/eval-tool) — Anthropic Workbench para casos de prueba, comparación lado a lado y calificación humana de 5 puntos.
- [Promptfoo getting started](https://www.promptfoo.dev/docs/getting-started/) — framework eval de terceros con evaluaciones calificadas por código, modelo y clasificación en 60+ proveedores.
- [Building effective agents — Anthropic Engineering](https://www.anthropic.com/engineering/building-effective-agents) — enmarca prompt engineering como el paso más simple antes de llegar a infraestructura ML (la misma lógica detrás de la respuesta correcta de Sample Question 3).
