---
title: "Secciones de Estudio"
linkTitle: "Secciones de estudio"
weight: 1
cascade:
  type: docs
sidebar:
  open: true
---

El material de estudio de CCA-F está dividido en **12 secciones** para que cada una quepa cómodamente en una conversación de Claude Code. Cuando inicias Mock Exam Mode, Claude lee exactamente una de estas por ronda; así las preguntas se mantienen específicas, los distractores se mantienen plausibles y el presupuesto de contexto se mantiene razonable.

Elige una sección para revisarla por encima, o salta a la [Sección 1 — Visión general del examen]({{< ref "01-exam-overview" >}}) para orientarte. Para practicar con Claude como supervisor, consulta la [página de inicio](/) para ver el prompt de arranque.

{{< cards >}}
  {{< card link="01-exam-overview" title="1. Visión general del examen" subtitle="Por qué existe CCA-F, puntuación, los 6 escenarios y una ruta de estudio mapeada a los pesos de dominio." >}}
  {{< card link="02-agentic-loops" title="2. Bucles de agentes" subtitle="Dominio 1.1 — flujo de control del bucle de agente, valores de stop_reason, antipatrones." >}}
  {{< card link="03-multi-agent-orchestration" title="3. Orquestación multiagente" subtitle="Dominios 1.2–1.4 — límites coordinador/subagente, herramienta Task, paso de contexto." >}}
  {{< card link="04-hooks-decomposition-sessions" title="4. Hooks, descomposición y sesiones" subtitle="Dominios 1.5–1.7 — hooks PostToolUse, prompt chaining frente a descomposición adaptativa, fork_session." >}}
  {{< card link="05-tool-design" title="5. Diseño de herramientas e integradas" subtitle="Dominios 2.1, 2.3, 2.5 — descripciones de herramientas, tool_choice, Read/Write/Edit/Bash/Grep/Glob." >}}
  {{< card link="06-mcp-integration" title="6. Integración MCP" subtitle="Dominios 2.2, 2.4 — sobres estructurados de error, ámbitos de .mcp.json, recursos frente a herramientas." >}}
  {{< card link="07-claude-md-and-rules" title="7. CLAUDE.md y reglas" subtitle="Dominios 3.1, 3.3 — jerarquía de CLAUDE.md, @imports, patrones glob de .claude/rules/." >}}
  {{< card link="08-commands-skills-plan-mode" title="8. Comandos, skills y modo plan" subtitle="Dominios 3.2, 3.4, 3.5 — slash commands, skills con context: fork, plan mode." >}}
  {{< card link="09-cicd-integration" title="9. Integración CI/CD" subtitle="Dominio 3.6 — claude -p, --output-format json, diseño de prompts para CI." >}}
  {{< card link="10-prompt-engineering" title="10. Ingeniería de prompts" subtitle="Dominios 4.1, 4.2, 4.4 — criterios explícitos, few-shot, revisión multipaso." >}}
  {{< card link="11-structured-output-and-batch" title="11. Salida estructurada y Batch" subtitle="Dominios 4.3, 4.5, 4.6 — JSON Schema, bucles de validación-reintento, Message Batches API." >}}
  {{< card link="12-context-and-reliability" title="12. Contexto y fiabilidad" subtitle="Dominio 5 — case-facts, disparadores de escalación, errores estructurados, mapeo claim–source." >}}
{{< /cards >}}

## Cómo usar estas secciones para el examen

1. **Revisa primero la sección 1**: establece la estructura del examen, los 6 escenarios y los pesos de dominio. Todo lo demás cuelga de ahí.
2. **Practica primero los dominios pesados.** D1 (Agentic Architecture) es el 27% del examen y aparece en tres escenarios; las secciones 2, 3 y 4 son tus horas de mayor rendimiento.
3. **Luego D3 + D4** (20% cada uno): secciones 7, 8 y 9 (Claude Code) y secciones 10 y 11 (ingeniería de prompts / salida estructurada).
4. **Termina con D2 y D5**: secciones 5 y 6 (herramientas / MCP) y 12 (contexto, escalación, procedencia; dominio pequeño pero transversal).
5. **Usa Mock Exam Mode en Claude Code** para practicar cada sección con preguntas de formato real hasta que puedas predecir el patrón de distractores antes de leer las opciones.
