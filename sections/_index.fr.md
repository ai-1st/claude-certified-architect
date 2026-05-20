---
title: "Sections d'étude"
linkTitle: "Sections d'étude"
weight: 1
cascade:
  type: docs
sidebar:
  open: true
---

Le matériel d'étude CCA-F est découpé en **12 sections** afin que chacune tienne confortablement dans une conversation Claude Code. Lorsque vous lancez le Mock Exam Mode, Claude lit exactement une section par tour — c'est ce qui garde les questions précises, les distracteurs plausibles et le budget de contexte sain.

Choisissez une section à parcourir, ou commencez par la [Section 1 — Vue d'ensemble de l'examen]({{< ref "01-exam-overview" >}}) pour vous orienter. Pour vous entraîner avec Claude comme examinateur, consultez la [page d'accueil](/) pour l'invite de démarrage.

{{< cards >}}
  {{< card link="01-exam-overview" title="1. Vue d'ensemble de l'examen" subtitle="Pourquoi CCA-F existe, la notation, les 6 scénarios et un parcours d'étude aligné sur les poids des domaines." >}}
  {{< card link="02-agentic-loops" title="2. Boucles agentiques" subtitle="Domaine 1.1 — flux de contrôle des boucles agentiques, valeurs stop_reason, anti-patterns." >}}
  {{< card link="03-multi-agent-orchestration" title="3. Orchestration multi-agent" subtitle="Domaines 1.2–1.4 — frontières coordinateur/subagent, outil Task, passage de contexte." >}}
  {{< card link="04-hooks-decomposition-sessions" title="4. Hooks, décomposition et sessions" subtitle="Domaines 1.5–1.7 — hooks PostToolUse, prompt chaining vs décomposition adaptative, fork_session." >}}
  {{< card link="05-tool-design" title="5. Conception d'outils et built-ins" subtitle="Domaines 2.1, 2.3, 2.5 — descriptions d'outils, tool_choice, Read/Write/Edit/Bash/Grep/Glob." >}}
  {{< card link="06-mcp-integration" title="6. Intégration MCP" subtitle="Domaines 2.2, 2.4 — enveloppes d'erreur structurées, portées .mcp.json, ressources vs outils." >}}
  {{< card link="07-claude-md-and-rules" title="7. CLAUDE.md et règles" subtitle="Domaines 3.1, 3.3 — hiérarchie CLAUDE.md, @imports, motifs glob .claude/rules/." >}}
  {{< card link="08-commands-skills-plan-mode" title="8. Commandes, Skills et Plan Mode" subtitle="Domaines 3.2, 3.4, 3.5 — slash commands, skills avec context: fork, plan mode." >}}
  {{< card link="09-cicd-integration" title="9. Intégration CI/CD" subtitle="Domaine 3.6 — claude -p, --output-format json, conception de prompts pour la CI." >}}
  {{< card link="10-prompt-engineering" title="10. Ingénierie de prompts" subtitle="Domaines 4.1, 4.2, 4.4 — critères explicites, few-shot, revue multi-passe." >}}
  {{< card link="11-structured-output-and-batch" title="11. Sorties structurées et batch" subtitle="Domaines 4.3, 4.5, 4.6 — JSON Schema, boucles validation-retry, Message Batches API." >}}
  {{< card link="12-context-and-reliability" title="12. Contexte et fiabilité" subtitle="Domaine 5 — case-facts, déclencheurs d'escalade, erreurs structurées, mapping claim–source." >}}
{{< /cards >}}

## Comment les utiliser pour l'examen

1. **Parcourez d'abord la section 1** — elle pose la structure de l'examen, les 6 scénarios et les poids des domaines. Tout le reste s'y rattache.
2. **Travaillez d'abord les domaines les plus lourds.** D1 (Agentic Architecture) représente 27% de l'examen et apparaît dans trois scénarios — les sections 2, 3 et 4 sont vos heures les plus rentables.
3. **Passez ensuite à D3 + D4** (20% chacun) — sections 7, 8, 9 (Claude Code) et sections 10, 11 (prompt engineering / structured output).
4. **Terminez par D2 et D5** — sections 5, 6 (outils / MCP) et 12 (contexte, escalade, provenance — petit domaine mais transversal).
5. **Utilisez le Mock Exam Mode dans Claude Code** pour travailler chaque section avec des questions au format réel jusqu'à pouvoir prédire le schéma des distracteurs avant de lire les options.
