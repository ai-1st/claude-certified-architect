---
title: "Section 1 — Vue d'ensemble de l'examen, structure et scénarios"
linkTitle: "1. Vue d'ensemble de l'examen"
weight: 1
description: "Pourquoi CCA-F existe, comment il est noté, les six scénarios et un parcours d'étude aligné sur les poids de domaines publiés."
---

## Ce que couvre cette section

Un socle pour comprendre *pourquoi* le Claude Certified Architect – Foundations (CCA-F) existe, *ce qu'il* évalue, *comment* il est structuré (domaines, notation, scénarios) et *comment* planifier un parcours d'étude qui corresponde proprement aux poids de domaines publiés.

## Matériel source (guide officiel)

**Objectif.** Le CCA-F valide que les praticiens savent prendre des décisions de compromis éclairées lorsqu'ils implémentent des solutions réelles avec Claude. L'examen couvre quatre technologies centrales : Claude Code, le Claude Agent SDK, la Claude API et le Model Context Protocol (MCP).

**Candidat cible.** Un architecte de solutions avec environ 6+ mois d'expérience pratique avec Claude, ayant personnellement :

- Construit des applications agentiques avec le Claude Agent SDK (orchestration, subagents, intégration d'outils, hooks de cycle de vie).
- Configuré Claude Code pour des équipes via `CLAUDE.md`, Agent Skills, serveurs MCP et plan mode.
- Conçu des interfaces d'outils et de ressources MCP pour de vrais backends.
- Ingénié des prompts produisant des sorties structurées fiables (schémas JSON, few-shot, motifs d'extraction).
- Géré des fenêtres de contexte sur de longs documents et des passages de relais multi-agents.
- Intégré Claude dans la CI/CD pour la revue de code, la génération de tests et les retours sur PR.
- Pris des décisions d'escalade et de fiabilité, y compris human-in-the-loop et auto-évaluation.

**Format des questions.** Choix multiple, une bonne réponse plus trois distracteurs. Il n'y a aucune pénalité pour deviner ; les questions sans réponse sont comptées comme incorrectes.

**Notation.** Réussite/échec avec un score échelonné de 100 à 1 000. Le score minimal de réussite est **720**. La notation échelonnée égalise les résultats entre des versions d'examen de difficulté légèrement différente.

**Domaines et pondérations.**

| # | Domaine | Poids |
|---|---|---|
| 1 | Agentic Architecture & Orchestration | 27% |
| 2 | Tool Design & MCP Integration | 18% |
| 3 | Claude Code Configuration & Workflows | 20% |
| 4 | Prompt Engineering & Structured Output | 20% |
| 5 | Context Management & Reliability | 15% |

**Scénarios.** Chaque examen tire au hasard **4 scénarios de production sur 6**. L'ensemble complet :

1. **Customer Support Resolution Agent** — agent Agent SDK sur des outils MCP (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`) ; objectif de 80%+ de résolution au premier contact avec une escalade saine.
2. **Code Generation with Claude Code** — `CLAUDE.md`, slash commands personnalisées, plan mode vs exécution directe.
3. **Multi-Agent Research System** — coordinateur déléguant à des subagents de recherche web, d'analyse documentaire, de synthèse et de génération de rapport.
4. **Developer Productivity with Claude** — Agent SDK sur outils intégrés (`Read`, `Write`, `Bash`, `Grep`, `Glob`) plus serveurs MCP pour l'exploration de codebase et la génération de boilerplate.
5. **Claude Code for Continuous Integration** — revue de code intégrée à la CI/CD, génération de tests et feedback PR ; minimiser les faux positifs.
6. **Structured Data Extraction** — extraction validée par schéma JSON depuis des documents non structurés, avec gestion élégante des cas limites.

**Hors périmètre.** Fine-tuning, facturation/auth/quotas, vision et computer use, détails internes de l'API streaming, embeddings/vector DBs, RLHF/Constitutional AI, détails internes du prompt caching, tokenisation, configurations cloud-provider spécifiques et sujets similaires de plomberie de déploiement.

## Contexte enrichi

### Écosystème de certification

Anthropic a lancé publiquement le CCA-F le **12 mars 2026** comme première certification technique officielle. Le format rapporté par plusieurs sources secondaires est **60 questions à choix multiple, 120 minutes, surveillé, livre fermé**, avec des résultats livrés sous environ 2 jours ouvrés, incluant les ventilations par section. Le coût est de **99 $ par tentative**, annulé pour les 5 000 premières tentatives allouées au Claude Partner Network.

L'accès est aujourd'hui **médié par l'organisation** via le Claude Partner Network plutôt que totalement en libre-service : les candidats travaillent dans une organisation partenaire ou s'alignent avec l'une d'elles, puis demandent un créneau d'examen sur `anthropic.skilljar.com`. Le programme de soutien — Anthropic Academy — est gratuit et hébergé sur Skilljar ; il comprend "Claude 101", "Building with the Claude API", des cours MCP introductifs et avancés, "Claude Code in Action", "Introduction to agent skills", "Introduction to subagents" et des cours supplémentaires de déploiement cloud.

Anthropic a indiqué que **des niveaux supplémentaires pour vendeurs, développeurs et architectes avancés** arriveront plus tard en 2026. Traitez le CCA-F comme le niveau d'entrée d'une pile de certifications à plusieurs niveaux, et non comme un badge isolé.

### Stratégie de pondération des domaines

Les cinq domaines sont assez rapprochés (15%–27%), donc aucun domaine ne peut être ignoré, mais le Domaine 1 (Agentic Architecture & Orchestration) est le levier le plus important. Un budget d'étude pragmatique basé sur les poids :

- **~27% du temps d'étude sur le Domaine 1** — boucles agentiques, gestion de `stop_reason`, délégation à des subagents, hooks de cycle de vie. Ce domaine apparaît aussi dans les scénarios 1, 3 et 4, donc son effet de levier dépasse son poids brut.
- **~20% chacun sur les Domaines 3 et 4** — configuration Claude Code (`CLAUDE.md` hierarchy, plan mode, slash commands, skills) et ingénierie de prompts/sorties structurées (schémas JSON, few-shot, boucles de retry).
- **~18% sur le Domaine 2** — descriptions d'outils MCP qui désambiguïsent des outils similaires, réponses d'erreur structurées avec drapeaux `retryable`, et conception de ressources.
- **~15% sur le Domaine 5** — gestion de la fenêtre de contexte, scratchpads, règles d'escalade, routage human-in-the-loop basé sur la confiance.

Heuristique utile : environ **deux tiers de l'examen** portent sur la conception des boucles agentiques, la configuration Claude Code et les décisions de prompt/sortie structurée. Investissez votre temps d'étude là-dessus avant d'optimiser la longue traîne.

### Les six scénarios — à quoi s'attendre

- **Scénario 1 — Customer Support Resolution Agent.** Attendez-vous à des questions sur *quand* l'agent escalade (lacune de politique, demande explicite du client, incapacité à progresser) plutôt que de résoudre en autonomie, et sur la façon dont `process_refund` doit exposer des erreurs retryable/non-retryable pour que l'agent puisse récupérer sans boucler.
- **Scénario 2 — Code Generation with Claude Code.** Attendez-vous à des arbitrages plan-mode-vs-direct-execution, à l'emplacement des règles spécifiques aux chemins (`.claude/rules/`) et au périmètre des slash commands et skills personnalisées (`context: fork`, `allowed-tools`).
- **Scénario 3 — Multi-Agent Research System.** Attendez-vous aux frontières coordinateur/subagent, au passage de contexte cité entre agents sans exploser la fenêtre, et à l'endroit où imposer des boucles evaluator/critic pour garder les citations exactes.
- **Scénario 4 — Developer Productivity with Claude.** Attendez-vous à des questions de sélection d'outils parmi les built-ins (`Read`, `Write`, `Bash`, `Grep`, `Glob`) plus les compléments MCP, et à des jugements sur quand garder le travail dans l'agent parent ou forker un subagent.
- **Scénario 5 — Claude Code for CI.** Attendez-vous à des questions de prompt engineering pour des commentaires de revue *actionnables*, à des critères de revue explicites pour supprimer les faux positifs, et à des motifs de revue multi-passe pour les gros diffs.
- **Scénario 6 — Structured Data Extraction.** Attendez-vous à de la conception de schémas avec champs nullable/optional, des boucles validation-retry sur `tool_use`, des arbitrages de traitement batch et une dégradation élégante quand un document est partiellement malformé.

### Certifications comparables

| Certification | Vendor | Focus | Notes |
|---|---|---|---|
| Claude Certified Architect – Foundations | Anthropic | Architecture Claude de production : agents, MCP, Claude Code, structured output | Très scénarisé, axé jugement ; accès médié par partenaires ; 99 $ |
| AWS Certified AI Practitioner (AIF-C01) | AWS | Culture IA/ML fondationnelle + concepts Bedrock | 85 questions / 120 min ; réussite 700/1000 ; plus large et conceptuel, moins pratique |
| Google Cloud Generative AI Leader | Google | Stratégie business pour GenAI sur Vertex AI / Gemini | Orienté leadership, pas implémentation |

Le contraste le plus net : AWS AI Practitioner et GCP Generative AI Leader testent **l'étendue et la culture générale** sur des services managés ; CCA-F teste **la profondeur et le jugement** dans une seule pile agentique fournisseur. Les architectes multi-cloud les empilent souvent plutôt que de choisir.

## Glossaire des termes clés

- **Agent SDK** — bibliothèque d'Anthropic pour construire des boucles agentiques avec appels d'outils, subagents et hooks de cycle de vie.
- **Claude Code** — agent de codage terminal-native d'Anthropic, configuré via `CLAUDE.md`, slash commands, Agent Skills et serveurs MCP.
- **MCP (Model Context Protocol)** — protocole ouvert exposant outils, ressources et prompts de systèmes backend à Claude.
- **Plan mode** — mode Claude Code qui produit un plan avant toute action avec effet de bord.
- **`CLAUDE.md`** — fichier de configuration hiérarchique niveau projet/utilisateur consommé automatiquement par Claude Code.
- **Scaled score** — score normalisé 100–1 000 qui égalise entre les versions d'examen ; le seuil est 720.
- **Scenario** — contexte de production réaliste qui encadre un groupe de questions liées ; 4 sur 6 apparaissent par examen.
- **Distractor** — mauvaise réponse plausible qui piège les candidats à l'expérience incomplète.

## Parcours de préparation recommandé

1. **Parcourez les 12 sections d'étude dans l'ordre**, en portant une attention particulière aux énoncés de tâches cités au début de chaque section — ce sont les objectifs d'apprentissage par domaine tirés directement du guide officiel.
2. **Terminez les cours gratuits Anthropic Academy sur Skilljar** dans cet ordre : Claude 101 → Building with the Claude API → Introduction to MCP → MCP Advanced Topics → Claude Code in Action → Introduction to agent skills → Introduction to subagents.
3. **Construisez un projet Agent SDK from scratch** qui exerce une boucle agentique complète : appels d'outils, gestion de `stop_reason`, délégation à des subagents, hooks de cycle de vie et politique erreur/retry.
4. **Configurez Claude Code pour un vrai dépôt** : `CLAUDE.md` en couches, règles spécifiques aux chemins `.claude/rules/`, au moins une slash command personnalisée, une skill avec `context: fork` et `allowed-tools`, et une intégration de serveur MCP.
5. **Concevez et stressez des outils MCP** : écrivez des descriptions d'outils qui désambiguïsent des quasi-doublons, renvoient des erreurs structurées avec drapeaux `retryable`, et vérifiez la sélection d'outils sur des prompts ambigus.
6. **Livrez un pipeline d'extraction structurée** avec schémas JSON, boucle validation-retry, champs optional/nullable et Message Batches API pour le débit.
7. **Travaillez les six scénarios** en écrivant un croquis d'architecture d'une page pour chacun — domaines principaux, inventaire d'outils, politique d'escalade, stratégie de contexte et motifs de fiabilité.
8. **Passez l'examen officiel d'entraînement en dernier**, relisez chaque explication (y compris celles que vous avez réussies) et ne réservez la session surveillée que lorsque vous obtenez confortablement plus de 720 sur du matériel simulé.

## Références

- [Anthropic Academy on Skilljar](https://anthropic.skilljar.com/) — Cours gratuits de préparation (Claude 101, Building with the Claude API, MCP, Claude Code, agent skills, subagents).
- [Anthropic Learn — Courses index](https://www.anthropic.com/learn/courses) — Catalogue officiel des cours de formation Anthropic.
- [Claude Certified Architect: How to Get Certified in 2026 (lowcode.agency)](https://www.lowcode.agency/blog/how-to-become-claude-certified-architect) — Date de lancement, format, coût et parcours d'accès rapportés.
- [How to become a Claude Certified Architect (datastudios.org)](https://www.datastudios.org/post/how-to-become-a-claude-certified-architect-current-access-path-partner-requirements-preparation) — Parcours d'accès Partner Network, domaines d'étude, éléments officiellement confirmés.
- [Claude Certified Architect: who can take the exam (datastudios.org)](https://www.datastudios.org/post/claude-certified-architect-who-can-take-the-exam-and-why-access-is-still-limited) — Éligibilité et limites d'accès.
- [Inside Anthropic's Claude Certified Architect Program (dev.to/mcrolly)](https://dev.to/mcrolly/inside-anthropics-claude-certified-architect-program-what-it-tests-and-who-should-pursue-it-1dk6) — Décomposition de ce que l'examen évalue et public cible.
- [How to Pass the CCA Foundations Exam (dev.to/sojs)](https://dev.to/sojs/how-to-pass-the-claude-certified-architect-cca-foundations-exam-3oa9) — Phasage de préparation sur 7 semaines et pièges courants rapportés par un candidat.
- [Claude Certified Architect Study Guide — 12-week plan (claudecertifications.com)](https://claudecertifications.com/claude-certified-architect/study-guide) — Chronologie alternative plus longue.
- [Claude Architect vs AWS AI Practitioner (flashgenius.net)](https://flashgenius.net/blog-article/claude-architect-vs-aws-ai-practitioner-2026-which-ai-certification-should-you-choose) — Comparaison avec la certification AWS AI Practitioner.
- [AWS vs Google AI Certifications 2026 (pertamapartners.com)](https://www.pertamapartners.com/insights/aws-google-ai-certifications) — Contexte de comparaison pour la certification GCP Generative AI Leader.
