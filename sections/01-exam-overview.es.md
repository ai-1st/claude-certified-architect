---
title: "Sección 1 — Visión general, estructura y escenarios del examen"
linkTitle: "1. Visión general del examen"
weight: 1
description: "Por qué existe CCA-F, cómo se puntúa, los seis escenarios y una ruta de estudio mapeada a los pesos de dominio publicados."
---

## Qué cubre esta sección

Una base sobre *por qué* existe Claude Certified Architect – Foundations (CCA-F), *qué* evalúa, *cómo* está estructurado (dominios, puntuación, escenarios) y *cómo* planificar una ruta de estudio que se alinee limpiamente con los pesos de dominio publicados.

## Material fuente (de la guía oficial)

**Propósito.** CCA-F valida que los profesionales puedan tomar decisiones informadas de compromiso al implementar soluciones reales con Claude. El examen cubre cuatro tecnologías centrales: Claude Code, Claude Agent SDK, Claude API y Model Context Protocol (MCP).

**Candidato objetivo.** Un arquitecto de soluciones con aproximadamente 6+ meses de experiencia práctica con Claude que personalmente haya:

- Construido aplicaciones de agentes con Claude Agent SDK (orquestación, subagentes, integración de herramientas, hooks de ciclo de vida).
- Configurado Claude Code para equipos mediante `CLAUDE.md`, Agent Skills, servidores MCP y plan mode.
- Diseñado interfaces de herramientas y recursos MCP contra backends reales.
- Diseñado prompts que producen salida estructurada fiable (JSON schemas, few-shot, patrones de extracción).
- Gestionado ventanas de contexto en documentos largos y traspasos multiagente.
- Integrado Claude en CI/CD para revisión de código, generación de pruebas y feedback en PRs.
- Tomado decisiones de escalación y fiabilidad, incluyendo human-in-the-loop y autoevaluación.

**Formato de preguntas.** Opción múltiple, una respuesta correcta más tres distractores. No hay penalización por adivinar; las preguntas sin responder se puntúan como incorrectas.

**Puntuación.** Aprobado/suspenso con una puntuación escalada de 100–1,000. La puntuación mínima para aprobar es **720**. La puntuación escalada equipara resultados entre versiones del examen con dificultad ligeramente distinta.

**Dominios y pesos.**

| # | Dominio | Peso |
|---|---|---|
| 1 | Agentic Architecture & Orchestration | 27% |
| 2 | Tool Design & MCP Integration | 18% |
| 3 | Claude Code Configuration & Workflows | 20% |
| 4 | Prompt Engineering & Structured Output | 20% |
| 5 | Context Management & Reliability | 15% |

**Escenarios.** Cada examen extrae **4 de 6** escenarios de producción al azar. El conjunto completo:

1. **Customer Support Resolution Agent** — agente Agent SDK sobre herramientas MCP (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`); objetivo de 80%+ de resolución en primer contacto con escalación sólida.
2. **Code Generation with Claude Code** — `CLAUDE.md`, slash commands personalizados, plan mode frente a ejecución directa.
3. **Multi-Agent Research System** — coordinador que delega en subagentes de búsqueda web, análisis documental, síntesis y generación de informes.
4. **Developer Productivity with Claude** — Agent SDK sobre herramientas integradas (`Read`, `Write`, `Bash`, `Grep`, `Glob`) más servidores MCP para exploración de codebase y generación de boilerplate.
5. **Claude Code for Continuous Integration** — revisión de código integrada en CI/CD, generación de pruebas y feedback de PR; minimizar falsos positivos.
6. **Structured Data Extraction** — extracción validada con JSON schema desde documentos no estructurados con manejo elegante de casos límite.

**Fuera de alcance.** Fine-tuning, facturación/auth/cuotas, visión y computer use, detalles internos de streaming API, embeddings/vector DBs, RLHF/Constitutional AI, detalles internos de prompt caching, tokenización, configuraciones específicas de proveedores cloud y temas similares de infraestructura de despliegue.

## Contexto enriquecido

### Ecosistema de certificación

Anthropic lanzó públicamente CCA-F el **12 de marzo de 2026** como su primera certificación técnica oficial. El formato reportado por varias fuentes secundarias es de **60 preguntas de opción múltiple, 120 minutos, supervisado, cerrado**, con resultados entregados en ~2 días hábiles e incluyendo desglose por secciones. El coste es de **$99 por intento**, exonerado para los primeros 5,000 intentos asignados a Claude Partner Network.

El acceso hoy está **mediado por organizaciones** a través de Claude Partner Network, no es completamente autoservicio: los candidatos trabajan en una organización partner o se alinean con una, y luego solicitan un cupo de examen en `anthropic.skilljar.com`. El currículo de apoyo, Anthropic Academy, es gratuito y vive en Skilljar; incluye "Claude 101," "Building with the Claude API," cursos introductorios y avanzados de MCP, "Claude Code in Action," "Introduction to agent skills," "Introduction to subagents," y cursos adicionales de despliegue cloud.

Anthropic ha señalado que **niveles adicionales para sellers, developers y advanced architects** llegarán más adelante en 2026. Trata CCA-F como el nivel de entrada de una pila de credenciales multinivel, no como una insignia aislada.

### Estrategia según pesos de dominio

Los cinco dominios están bastante agrupados (15%–27%), así que no se puede omitir ninguno, pero Domain 1 (Agentic Architecture & Orchestration) es la palanca más grande. Un presupuesto de estudio pragmático basado en pesos:

- **~27% del tiempo de estudio en Domain 1**: bucles de agentes, manejo de `stop_reason`, delegación en subagentes, hooks de ciclo de vida. Este dominio también aparece dentro de los escenarios 1, 3 y 4, así que su rendimiento es mayor de lo que su peso bruto sugiere.
- **~20% cada uno en Domains 3 y 4**: configuración de Claude Code (`CLAUDE.md` hierarchy, plan mode, slash commands, skills) e ingeniería de prompts/salida estructurada (JSON schemas, few-shot, bucles de reintento).
- **~18% en Domain 2**: descripciones de herramientas MCP que desambiguan herramientas similares, respuestas de error estructuradas con flags `retryable` y diseño de recursos.
- **~15% en Domain 5**: gestión de ventana de contexto, scratchpads, reglas de escalación, enrutamiento human-in-the-loop basado en confianza.

Una heurística útil: aproximadamente **dos tercios del examen** giran alrededor del diseño de bucles de agentes, configuración de Claude Code y decisiones de prompts/salida estructurada. Invierte tiempo ahí antes de afinar la cola larga.

### Los seis escenarios: qué esperar

- **Scenario 1 — Customer Support Resolution Agent.** Espera preguntas sobre *cuándo* el agente escala (brecha de política, solicitud explícita del cliente, incapacidad de avanzar) frente a cuándo resuelve de forma autónoma, y cómo `process_refund` debe exponer errores reintentables/no reintentables para que el agente pueda recuperarse sin entrar en bucle.
- **Scenario 2 — Code Generation with Claude Code.** Espera compromisos entre plan mode y ejecución directa, dónde viven las reglas específicas de ruta (`.claude/rules/`) y cómo se acotan los slash commands y skills personalizados (`context: fork`, `allowed-tools`).
- **Scenario 3 — Multi-Agent Research System.** Espera límites coordinador/subagente, cómo pasar contexto citado entre agentes sin agotar la ventana y dónde imponer bucles evaluador/crítico para mantener citas precisas.
- **Scenario 4 — Developer Productivity with Claude.** Espera preguntas de selección de herramientas entre integradas (`Read`, `Write`, `Bash`, `Grep`, `Glob`) más extensiones MCP, y juicios sobre cuándo mantener el trabajo en el agente padre frente a crear un subagente.
- **Scenario 5 — Claude Code for CI.** Espera preguntas de ingeniería de prompts para comentarios de revisión *accionables*, criterios de revisión explícitos para suprimir falsos positivos y patrones de revisión multipaso para diffs grandes.
- **Scenario 6 — Structured Data Extraction.** Espera diseño de schemas con campos nullable/optional, bucles de validación-reintento en `tool_use`, compromisos de procesamiento batch y degradación elegante cuando un documento está parcialmente malformado.

### Certificaciones comparables

| Certificación | Proveedor | Enfoque | Notas |
|---|---|---|---|
| Claude Certified Architect – Foundations | Anthropic | Arquitectura de producción con Claude: agentes, MCP, Claude Code, salida estructurada | Muy basada en escenarios y criterio; acceso mediado por partners; $99 |
| AWS Certified AI Practitioner (AIF-C01) | AWS | Alfabetización fundacional AI/ML + conceptos de Bedrock | 85 preguntas / 120 min; aprobado 700/1000; más amplia y conceptual, menos práctica |
| Google Cloud Generative AI Leader | Google | Estrategia de negocio para GenAI en Vertex AI / Gemini | Orientada a liderazgo, no a implementación |

El contraste más claro: AWS AI Practitioner y GCP Generative AI Leader evalúan **amplitud y alfabetización** en servicios gestionados; CCA-F evalúa **profundidad y criterio** dentro de la pila de agentes de un solo proveedor. Los arquitectos multicloud a menudo las apilan en vez de elegir.

## Glosario de términos clave

- **Agent SDK** — biblioteca de Anthropic para construir bucles de agentes con llamadas a herramientas, subagentes y hooks de ciclo de vida.
- **Claude Code** — agente de programación nativo de terminal de Anthropic, configurado mediante `CLAUDE.md`, slash commands, Agent Skills y servidores MCP.
- **MCP (Model Context Protocol)** — protocolo abierto para exponer herramientas, recursos y prompts desde sistemas backend hacia Claude.
- **Plan mode** — modo de Claude Code que produce un plan antes de ejecutar cualquier acción con efectos secundarios.
- **`CLAUDE.md`** — archivo jerárquico de configuración a nivel de proyecto/usuario consumido automáticamente por Claude Code.
- **Scaled score** — puntuación normalizada de 100–1,000 que equipara versiones del examen; el punto de corte es 720.
- **Scenario** — contexto realista de producción que enmarca un bloque de preguntas relacionadas; aparecen 4 de 6 por examen.
- **Distractor** — respuesta incorrecta de apariencia plausible que atrapa a candidatos con experiencia incompleta.

## Ruta de preparación recomendada

1. **Revisa las 12 secciones de estudio en orden**, prestando atención extra a los enunciados de tareas citados al inicio de cada sección: son los objetivos de aprendizaje por dominio tomados directamente de la guía oficial del examen.
2. **Completa los cursos gratuitos de Anthropic Academy en Skilljar** en este orden: Claude 101 → Building with the Claude API → Introduction to MCP → MCP Advanced Topics → Claude Code in Action → Introduction to agent skills → Introduction to subagents.
3. **Construye un proyecto Agent SDK desde cero** que ejercite un bucle de agente completo: llamadas a herramientas, manejo de `stop_reason`, delegación en subagentes, hooks de ciclo de vida y política de error/reintento.
4. **Configura Claude Code para un repo real**: `CLAUDE.md` por capas, reglas específicas de ruta en `.claude/rules/`, al menos un slash command personalizado, una skill con `context: fork` y `allowed-tools`, y una integración de servidor MCP.
5. **Diseña y somete a estrés herramientas MCP**: escribe descripciones que desambigüen casi duplicados, devuelve errores estructurados con flags `retryable` y verifica la selección de herramientas en prompts ambiguos.
6. **Envía una pipeline de extracción estructurada** con JSON schemas, un bucle de validación-reintento, campos optional/nullable y Message Batches API para throughput.
7. **Practica los seis escenarios** escribiendo un bosquejo arquitectónico de una página para cada uno: dominios principales, inventario de herramientas, política de escalación, estrategia de contexto y patrones de fiabilidad.
8. **Toma al final el examen oficial de práctica**, revisa cada explicación (incluidas las que acertaste) y reserva el examen supervisado solo cuando estés puntuando cómodamente por encima de 720 en material simulado.

## Referencias

- [Anthropic Academy on Skilljar](https://anthropic.skilljar.com/) — cursos gratuitos de preparación (Claude 101, Building with the Claude API, MCP, Claude Code, agent skills, subagents).
- [Anthropic Learn — Courses index](https://www.anthropic.com/learn/courses) — catálogo oficial de cursos de Anthropic.
- [Claude Certified Architect: How to Get Certified in 2026 (lowcode.agency)](https://www.lowcode.agency/blog/how-to-become-claude-certified-architect) — fecha de lanzamiento, formato, coste y ruta de acceso reportados.
- [How to become a Claude Certified Architect (datastudios.org)](https://www.datastudios.org/post/how-to-become-a-claude-certified-architect-current-access-path-partner-requirements-preparation) — ruta de acceso por Partner Network, áreas de estudio y qué está confirmado oficialmente.
- [Claude Certified Architect: who can take the exam (datastudios.org)](https://www.datastudios.org/post/claude-certified-architect-who-can-take-the-exam-and-why-access-is-still-limited) — elegibilidad y limitaciones de acceso.
- [Inside Anthropic's Claude Certified Architect Program (dev.to/mcrolly)](https://dev.to/mcrolly/inside-anthropics-claude-certified-architect-program-what-it-tests-and-who-should-pursue-it-1dk6) — desglose de qué evalúa el examen y audiencia objetivo.
- [How to Pass the CCA Foundations Exam (dev.to/sojs)](https://dev.to/sojs/how-to-pass-the-claude-certified-architect-cca-foundations-exam-3oa9) — fases de preparación de 7 semanas reportadas por candidatos y errores comunes.
- [Claude Certified Architect Study Guide — 12-week plan (claudecertifications.com)](https://claudecertifications.com/claude-certified-architect/study-guide) — cronograma alternativo de preparación más largo.
- [Claude Architect vs AWS AI Practitioner (flashgenius.net)](https://flashgenius.net/blog-article/claude-architect-vs-aws-ai-practitioner-2026-which-ai-certification-should-you-choose) — comparación con la certificación AWS AI Practitioner.
- [AWS vs Google AI Certifications 2026 (pertamapartners.com)](https://www.pertamapartners.com/insights/aws-google-ai-certifications) — contexto comparativo para la credencial GCP Generative AI Leader.
