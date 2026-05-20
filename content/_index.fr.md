---
title: "Claude Certified Architect — Foundations"
toc: false
---

{{< hextra/hero-badge >}}
  <div class="hx:w-2 hx:h-2 hx:rounded-full hx:bg-primary-400"></div>
  Non officiel · licence MIT · open source
{{< /hextra/hero-badge >}}

{{< hextra/hero-headline >}}
  Entraînez-vous à l'examen CCA-F, avec Claude comme examinateur.
{{< /hextra/hero-headline >}}

{{< hextra/hero-subtitle >}}
  Ouvrez le dépôt dans Claude Code. Dites *"start a mock exam."* &nbsp;
  Claude lit une section d'étude à la fois et vous interroge dans le même
  format à scénarios, avec quatre options, que le véritable examen Claude Certified
  Architect — Foundations.
{{< /hextra/hero-subtitle >}}

{{< hextra/hero-button text="Commencer par les 12 sections d'étude" link="docs" >}}
&nbsp;
{{< hextra/hero-button text="Voir sur GitHub" link="https://github.com/ai-1st/claude-certified-architect" style="background: linear-gradient(90deg, #555 0%, #333 100%);" >}}

<div class="hx:mt-8 hx:flex hx:flex-wrap hx:items-center hx:justify-center hx:gap-2">
  <span class="hx:text-sm hx:font-semibold hx:opacity-70 hx:mr-2">Lire ce guide en :</span>
  <a href="/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-neutral-700 hx:text-current hx:no-underline hx:transition hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800">English</a>
  <a href="/es/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-neutral-700 hx:text-current hx:no-underline hx:transition hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800">Español</a>
  <a href="/fr/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:bg-neutral-900 hx:text-white hx:dark:bg-white hx:dark:text-neutral-900 hx:font-semibold hx:no-underline hx:transition hx:hover:opacity-90">Français</a>
  <a href="/ru/" class="hx:inline-flex hx:items-center hx:px-4 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-neutral-700 hx:text-current hx:no-underline hx:transition hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800">Русский</a>
</div>

<div class="hx:mt-6"></div>

## Ce qu'est ce dépôt

Un compagnon d'étude pour l'examen **Claude Certified Architect — Foundations (CCA-F)** — la première certification technique officielle d'Anthropic, lancée en mars 2026. L'examen évalue le jugement pratique nécessaire pour construire des applications Claude de niveau production : boucles agentiques, outils MCP, configuration de Claude Code, ingénierie de prompts et de sorties structurées, gestion du contexte.

Le dépôt contient deux éléments :

1. Un fichier `CLAUDE.md` qui place Claude en **Mock Exam Mode** — un examinateur section par section qui génère des questions au format officiel, attend votre réponse, puis explique pourquoi chaque option est correcte ou incorrecte.
2. **Douze sections d'étude** (`sections/01..12.md`) couvrant tous les domaines de l'examen. Les mêmes fichiers alimentent à la fois l'examen blanc de Claude et ce site web.

## Fonctionnement du Mock Exam Mode

{{< cards >}}
  {{< card link="docs/01-exam-overview" title="1. Choisir une section" subtitle="Claude vous accueille et vous demande laquelle des 12 sections (ou lequel des 6 scénarios officiels) vous voulez travailler." >}}
  {{< card link="docs/02-agentic-loops" title="2. Un fichier à la fois" subtitle="Claude lit uniquement le fichier de section choisi. Le contexte reste léger ; les questions restent ancrées." >}}
  {{< card link="docs" title="3. Questions au format réel" subtitle="Mise en situation → énoncé → exactement quatre options. Une seule bonne réponse. Trois distracteurs plausibles tirés d'anti-patterns réels." >}}
  {{< card link="docs" title="4. Réponse, puis explication" subtitle="Claude attend votre lettre, puis explique pourquoi la bonne option est correcte et pourquoi chaque distracteur est faux, en citant le concept de la section par son nom." >}}
{{< /cards >}}

## L'examen en bref

| | |
| --- | --- |
| Format | 60 questions à choix multiple, une bonne réponse + trois distracteurs |
| Durée | 120 minutes |
| Score | Score échelonné 100–1000 ; réussite = **720** |
| Scénarios | **4 sur 6** tirés au hasard par session |
| Domaines | D1 Agentic Architecture & Orchestration **27%** · D2 Tool Design & MCP **18%** · D3 Claude Code Config **20%** · D4 Prompt Engineering & Structured Output **20%** · D5 Context Management & Reliability **15%** |

## Démarrage rapide

```bash
git clone https://github.com/ai-1st/claude-certified-architect.git
cd claude-certified-architect
claude            # or open the folder in Cursor / Claude Code
# then say:  Start a mock exam.
```

Claude charge automatiquement `CLAUDE.md` et vous guide dans le Mock Exam Mode.

## Avertissement

Non officiel. Non affilié à Anthropic et non approuvé par Anthropic. L'examen, ses domaines de contenu, ses scénarios et sa notation appartiennent à Anthropic. Les sections d'étude de ce dépôt sont une reformulation indépendante de ressources publiquement disponibles, organisée pour un entraînement efficace en contexte avec Claude.
