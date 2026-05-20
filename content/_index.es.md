---
title: "Claude Certified Architect — Foundations"
toc: false
---

{{< hextra/hero-badge >}}
  <div class="hx:w-2 hx:h-2 hx:rounded-full hx:bg-primary-400"></div>
  No oficial · Licencia MIT · código abierto
{{< /hextra/hero-badge >}}

{{< hextra/hero-headline >}}
  Practica para el examen CCA-F, con Claude como tu supervisor.
{{< /hextra/hero-headline >}}

{{< hextra/hero-subtitle >}}
  Abre el repositorio en Claude Code. Di *"start a mock exam."* &nbsp;
  Claude lee una sección de estudio a la vez y te hace preguntas con el mismo
  formato basado en escenarios y de cuatro opciones que se usa en el examen real
  Claude Certified Architect — Foundations.
{{< /hextra/hero-subtitle >}}

{{< hextra/hero-button text="Empieza con las 12 secciones de estudio" link="docs" >}}
&nbsp;
{{< hextra/hero-button text="Ver en GitHub" link="https://github.com/ai-1st/claude-certified-architect" style="background: linear-gradient(90deg, #555 0%, #333 100%);" >}}

<div class="hx:mt-8 hx:flex hx:flex-wrap hx:items-center hx:justify-center hx:gap-2">
  <span class="hx:text-sm hx:font-semibold hx:opacity-70 hx:mr-2">Lee esta guía en:</span>
  <a href="/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-neutral-700 hx:text-current hx:no-underline hx:transition hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800">English</a>
  <a href="/es/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:bg-neutral-900 hx:text-white hx:dark:bg-white hx:dark:text-neutral-900 hx:font-semibold hx:no-underline hx:transition hx:hover:opacity-90">Español</a>
  <a href="/fr/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-neutral-700 hx:text-current hx:no-underline hx:transition hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800">Français</a>
  <a href="/ru/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-neutral-700 hx:text-current hx:no-underline hx:transition hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800">Русский</a>
</div>

<div class="hx:mt-6"></div>

## Qué es este repositorio

Un compañero de estudio para el examen **Claude Certified Architect — Foundations (CCA-F)**, la primera certificación técnica oficial de Anthropic, lanzada en marzo de 2026. El examen evalúa el criterio práctico para crear aplicaciones de Claude listas para producción: bucles de agentes, herramientas MCP, configuración de Claude Code, ingeniería de prompts y salidas estructuradas, y gestión del contexto.

El repositorio contiene dos cosas:

1. Un archivo `CLAUDE.md` que pone a Claude en **Mock Exam Mode**: un supervisor de cuestionarios sección por sección que genera preguntas en el formato oficial, espera tu respuesta y luego explica por qué cada opción es correcta o incorrecta.
2. **Doce secciones de estudio** (`sections/01..12.md`) que cubren todos los dominios del examen. Los mismos archivos impulsan tanto el simulacro de examen de Claude como este sitio web.

## Cómo funciona Mock Exam Mode

{{< cards >}}
  {{< card link="docs/01-exam-overview" title="1. Elige una sección" subtitle="Claude te saluda y pregunta cuál de las 12 secciones, o cuál de los 6 escenarios oficiales, quieres practicar." >}}
  {{< card link="docs/02-agentic-loops" title="2. Un archivo a la vez" subtitle="Claude solo lee el archivo de la sección elegida. El contexto se mantiene ligero y las preguntas se mantienen fundamentadas." >}}
  {{< card link="docs" title="3. Preguntas con formato real" subtitle="Introducción de escenario → enunciado → exactamente cuatro opciones. Una respuesta correcta. Tres distractores plausibles tomados de antipatrones reales." >}}
  {{< card link="docs" title="4. Respuesta y luego explicación" subtitle="Claude espera tu letra y luego explica por qué la opción correcta lo es y por qué cada distractor es incorrecto, citando por nombre el concepto de la sección." >}}
{{< /cards >}}

## El examen de un vistazo

| | |
| --- | --- |
| Formato | 60 preguntas de opción múltiple, una correcta + tres distractores |
| Tiempo | 120 minutos |
| Puntuación | Escalada 100–1000; aprobado = **720** |
| Escenarios | **4 de 6** seleccionados al azar por intento |
| Dominios | D1 Agentic Architecture & Orchestration **27%** · D2 Tool Design & MCP **18%** · D3 Claude Code Config **20%** · D4 Prompt Engineering & Structured Output **20%** · D5 Context Management & Reliability **15%** |

## Inicio rápido

```bash
git clone https://github.com/ai-1st/claude-certified-architect.git
cd claude-certified-architect
claude            # or open the folder in Cursor / Claude Code
# then say:  Start a mock exam.
```

Claude detecta `CLAUDE.md` automáticamente y te guía por Mock Exam Mode.

## Aviso

No oficial. No está afiliado ni avalado por Anthropic. El examen, sus dominios de contenido, escenarios y puntuación pertenecen a Anthropic. Las secciones de estudio de este repositorio son una reformulación independiente de material disponible públicamente, organizada para practicar con Claude de forma eficiente en contexto.
