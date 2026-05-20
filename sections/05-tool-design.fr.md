---
title: "Section 5 — Conception d'interfaces d'outils, distribution et outils intégrés"
linkTitle: "5. Conception d'outils et built-ins"
weight: 5
description: "Domaines 2.1, 2.3, 2.5 — rédiger des descriptions d'outils distinctives, tool_choice, accès outillé cadré et built-ins Read/Write/Edit/Bash/Grep/Glob."
---

## Ce que couvre cette section

Trois sujets étroitement liés du Domaine 2 :

- **2.1** — Comment écrire des définitions d'outils (`name`, `description`, `input_schema`, `input_examples`) afin que Claude choisisse de façon fiable le bon outil, y compris comment reconnaître et corriger un mauvais routage dû aux descriptions.
- **2.3** — Comment cadrer les outils visibles par chaque agent ou subagent, et comment utiliser le paramètre `tool_choice` (`auto`, `any`, outil forcé, `none`) plus `disable_parallel_tool_use` pour contrôler le comportement d'invocation.
- **2.5** — Comment appliquer le jeu d'outils intégré de Claude Code (`Read`, `Write`, `Edit`, `Bash`, `Grep`, `Glob`, plus `WebFetch`, `WebSearch`, `NotebookEdit`, `Agent`, `Task*`) au travail réel sur codebase, y compris le motif canonique Edit-échec-sur-ancre-non-unique → fallback Read+Write.

Deux items notés s'appuient directement sur ce contenu : Sample Question 2 (descriptions minimales de `get_customer` / `lookup_order`) et Sample Question 9 (cadrer `verify_fact` pour l'agent de synthèse).

## Matériel source (guide officiel)

### 2.1 Interfaces d'outils avec descriptions claires

Les descriptions d'outils sont *le* mécanisme principal que le LLM utilise pour choisir les outils. Des descriptions minimales rendent la sélection peu fiable entre outils similaires — `analyze_content` vs `analyze_document`, ou `get_customer` vs `lookup_order`. Les descriptions doivent inclure formats d'entrée, exemples de requêtes, cas limites et frontières explicites ("use this tool when…, do **not** use this tool when…"). Le wording du prompt système est sensible aux mots-clés et peut surclasser une bonne description ; le prompt système fait donc partie de la surface de routage d'outils. Corrections attendues : différencier les descriptions par objectif / entrées / sorties / quand utiliser, **renommer** en cas de chevauchement (`analyze_content` → `extract_web_results`), **diviser** les outils génériques en outils spécifiques (`analyze_document` → `extract_data_points` + `summarize_content` + `verify_claim_against_source`), et auditer les prompts système pour les fuites de mots-clés.

### 2.3 Distribution des outils et tool_choice

Donner 18 outils à un agent au lieu de 4–5 dégrade mesurablement la fiabilité de sélection, et les agents avec des outils hors spécialisation tendent à les utiliser à mauvais escient (un agent de synthèse qui tente des recherches web). La correction est **l'accès outillé cadré** plus un petit nombre d'**outils cross-role cadrés** pour les besoins fréquents (p. ex. `verify_fact` câblé dans l'agent de synthèse afin qu'il ne fasse pas d'aller-retour via le coordinateur). `tool_choice` a quatre valeurs — `auto`, `any`, `{"type": "tool", "name": "..."}` et `none` — détaillées ci-dessous.

### 2.5 Outils intégrés (Read/Write/Edit/Bash/Grep/Glob)

`Grep` cherche dans le contenu des fichiers (regex) ; `Glob` matche les *chemins* de fichiers (`**/*.test.tsx`). `Read`/`Write` sont des opérations fichier complet ; `Edit` effectue un remplacement de chaîne unique et **échoue** si `old_string` n'est pas unique ou si le fichier n'a pas été lu dans la session courante — le fallback correct est alors `Read` puis `Write`. Construisez la compréhension incrémentalement : `Grep` pour les points d'entrée, `Read` pour suivre les imports, puis tracer les usages en énumérant les noms exportés et en faisant un `Grep` sur chacun.

## Écrire une excellente description d'outil

Le guide "Define tools" d'Anthropic est explicite : **les descriptions détaillées sont de loin le facteur le plus important de performance des outils**, avec une cible de "au moins 3–4 phrases par description d'outil, davantage si l'outil est complexe." ([Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools))

### Anatomie d'une description (les 5 éléments)

1. **Ce que fait l'outil** — l'action, en une phrase.
2. **Quand l'utiliser (et quand ne pas l'utiliser)** — la frontière qui désambiguïse les outils voisins.
3. **Entrées** — ce que signifie chaque paramètre, formats acceptés, exemples (`AAPL` pour ticker), requis vs optionnel.
4. **Sorties** — ce que l'outil renvoie et ce qu'il ne renvoie volontairement *pas*.
5. **Mises en garde / limites** — rate limits, fraîcheur, périmètre régional, unités.

Plus le tableau optionnel `input_examples` — exemples d'entrées validés par schéma qui aident pour des formes de paramètres complexes ou imbriquées. Les exemples coûtent ~20–50 tokens pour des entrées simples et ~100–200 pour des objets imbriqués.

### Exemples avant/après (mauvais vs bon)

Mauvais — l'exemple canonique d'Anthropic :

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

Bon — le contre-exemple canonique d'Anthropic :

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

La bonne description dit à Claude *quoi*, *quand*, *ce que cela renvoie* et *ce que cela ne renvoie pas* — exactement les quatre lacunes qui font choisir `get_customer` plutôt que `lookup_order` dans Sample Question 2.

## Nommage et décomposition des outils

### Diviser un outil trop large

Un seul outil `analyze_document` avec une description vague force Claude à deviner. Décomposez en outils spécifiques dont les noms *sont* le désambiguïsateur :

| Original | Remplacement |
| --- | --- |
| `analyze_document` | `extract_data_points`, `summarize_content`, `verify_claim_against_source` |
| `fetch_url` (générique) | `load_document` (valide que l'URL est un document) |
| `analyze_content` (chevauche web + doc) | `extract_web_results` (web seulement) |

Le guide [writing-tools-for-agents](https://www.anthropic.com/engineering/writing-tools-for-agents) d'Anthropic souligne l'inverse : éviter la prolifération un-outil-par-endpoint (`list_users`, `list_events`, `create_event`) quand un `schedule_event` consolidé correspond mieux au workflow réel. La règle est *un outil par subdivision naturelle d'une tâche*, pas un outil par appel API.

### Renommer pour clarifier

Utilisez des **préfixes de service** (`github_list_prs`, `slack_send_message`, `asana_search`) lorsque le même agent a des outils couvrant plusieurs systèmes. Anthropic note explicitement que le namespacing par préfixe vs suffixe a des effets mesurables sur la précision d'évaluation. Renommez les outils qui se chevauchent sémantiquement jusqu'à ce que chaque nom soit non ambigu isolément — `analyze_content` et `analyze_document` sont une collision textbook ; `extract_web_results` et `summarize_pdf` ne le sont pas.

## Distribuer les outils entre agents

### Pourquoi moins d'outils = meilleure sélection

Les évaluations MCP d'Anthropic sur Opus 4 ont mesuré l'effondrement de la précision de sélection d'outils à mesure que le nombre d'outils augmente : 50+ outils chargés en amont arrivaient à 49% de précision, remontant à 74% seulement quand [Tool Search](https://code.claude.com/docs/en/agent-sdk/tool-search) activait le chargement différé. Le cadrage "18 vs 4–5" du guide correspond à cette courbe — chaque outil supplémentaire augmente le risque de mauvais routage et consomme du contexte (50 définitions d'outils peuvent coûter 10–20K tokens).

Conclusion architecturale : donnez à chaque subagent le plus petit jeu d'outils qui lui permet de terminer son rôle, et laissez le coordinateur porter la surface plus large. Quand un jeu d'outils doit rester grand, activez Tool Search afin que seules 3–5 définitions pertinentes soient matérialisées par tour.

### Outils cross-role cadrés

Le motif de l'agent de synthèse dans Sample Question 9 est le cas canonique d'examen. La synthèse a légitimement besoin de *quelques* vérifications de faits (85% de ses vérifications sont simples), mais lui donner tout le jeu web-search le surprovisionne. La correction est un seul outil `verify_fact` **contraint** — entrées étroites, sorties étroites, pas de fetch général — qui gère le cas courant, tandis que les 15% d'investigations complexes restent routées via le coordinateur vers l'agent de recherche web dédié. C'est le principe du moindre privilège appliqué aux outils.

### Table de configuration tool_choice

| `tool_choice` | Comportement | À utiliser quand |
| --- | --- | --- |
| `{"type": "auto"}` | Par défaut avec outils présents. Claude décide d'appeler ou non un outil, éventuellement plusieurs en parallèle. | Boucles agentiques générales où le modèle doit raisonner sur la nécessité des outils. |
| `{"type": "any"}` | Claude doit appeler un outil (n'importe lequel des fournis). Aucun texte conversationnel n'est émis. | Agents de sortie structurée où les réponses en texte libre sont une erreur. À combiner avec `strict: true` pour des entrées garanties par schéma. |
| `{"type": "tool", "name": "extract_metadata"}` | Force Claude à appeler un outil précis. Le message assistant est prérempli en bloc `tool_use`. | Pipelines "exécuter X d'abord" (p. ex. `extract_metadata` avant enrichissement). Traiter les étapes suivantes dans des tours ultérieurs. |
| `{"type": "none"}` | Par défaut quand aucun outil n'est fourni. Claude ne peut pas appeler d'outils. | Tours de chat purs dans une session équipée d'outils. |
| `disable_parallel_tool_use: true` | Modificateur sur tous les modes ci-dessus. Avec `auto`, Claude appelle *au plus un* outil ; avec `any`/`tool`, exactement un. | Quand les outils aval ont des dépendances d'ordre ou des rate limits. |

Trois contraintes à mémoriser :

- `tool_choice: any` et le mode outil forcé ne sont **pas compatibles avec extended thinking** — seuls `auto` et `none` fonctionnent dans ce mode.
- Changer `tool_choice` invalide les blocs de messages cachés sous prompt caching (outils et prompt système restent cachés).
- `disable_parallel_tool_use` doit être défini sur la requête qui renvoie le bloc `tool_use` ; le mettre sur un follow-up n'a aucun effet rétroactif.

Références : [Define tools — Forcing tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools#forcing-tool-use) et [Parallel tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use).

## Aide-mémoire des outils intégrés

La surface actuelle des built-ins Claude Code ([Tools reference](https://docs.claude.com/en/docs/claude-code/tools-reference)) :

| Outil | À utiliser quand | À éviter quand | Fallback / note |
| --- | --- | --- | --- |
| `Read` | Charger un chemin de fichier connu ; voir images, PDFs, notebooks. Requis avant tout `Edit` ou `Write` sur un fichier existant. | Lister un répertoire (utiliser `Bash ls` ou `Glob`). | Erreurs sur fichiers trop grands — réessayer avec `offset`/`limit`. |
| `Write` | Créer de nouveaux fichiers ; écraser après un `Read`. | Éditions ciblées dans un gros fichier (utiliser `Edit`). | Échoue sur fichiers existants non lus dans la session. |
| `Edit` | Remplacement ciblé d'une chaîne unique dans un fichier déjà lu. | La chaîne d'ancrage n'est pas unique, ou le fichier a changé sur disque après lecture. | Relire, puis `Write` le contenu complet mis à jour ; ou `replace_all: true` si vous voulez réellement chaque occurrence. |
| `Bash` | Lancer scripts, package managers, git, formatters. Processus longs via `run_in_background: true`. | Lire ou écrire des fichiers (utiliser `Read`/`Write`/`Edit` afin que permissions et contrôles d'éligibilité d'édition s'appliquent). | Timeout 2 min par défaut, relevable à 10 min ; sortie plafonnée à 30K chars (fichier écrit pour le reste). |
| `Grep` | Recherche de contenu dans la codebase (noms de fonctions, chaînes d'erreur, imports). Basé sur ripgrep — échapper les métacaractères regex. | Localiser des fichiers uniquement par motif de nom (utiliser `Glob`). | Respecte `.gitignore` ; passer un chemin explicite pour contourner. Utiliser `multiline: true` pour franchir les lignes. |
| `Glob` | Matching de noms de fichiers : `**/*.test.tsx`, `src/**/*.ts`. | Chercher dans le contenu des fichiers (utiliser `Grep`). | Plafonné à 100 résultats, triés par mtime ; ne respecte *pas* `.gitignore` par défaut. |
| `WebFetch` | Récupérer une URL connue et extraire des réponses via un petit modèle extracteur. | Besoin du HTML brut ou de sélecteurs précis (utiliser `Bash curl`). | Lossy par conception — refetcher avec un prompt d'extraction plus spécifique si besoin. Cache 15 min par URL. |
| `WebSearch` | Découvrir des URLs par requête ; jusqu'à 8 recherches backend par appel ; supporte `allowed_domains` *ou* `blocked_domains` (pas les deux). | Lire les pages de résultats (enchaîner `WebFetch` après). | La règle de permission n'a pas de spécificateur — autoriser/refuser tout l'outil. |
| `NotebookEdit` | Modifier une cellule Jupyter par `cell_id` (`replace`, `insert`, `delete`). | Remplacement de texte cross-cell. | Les règles de permission utilisent le format de chemin `Edit(...)`. |
| `Agent` | Lancer un subagent cadré. | Opérations routinières que le parent peut faire directement. | Le frontmatter `tools` / `disallowedTools` du subagent cadre sa surface. |
| `Task*` (`TaskCreate` / `TaskList` / `TaskUpdate`) | Suivi de travail multi-étapes dans les sessions interactives. | Tâches triviales one-shot. | `TodoWrite` est l'équivalent headless-mode. |

Le motif pertinent pour l'examen est la **boucle de récupération d'échec Edit** : `Edit` exige que l'ancre apparaisse exactement une fois dans un fichier déjà lu dans cette session. Quand l'ancre se répète — courant dans les configs, lignes d'import répétées ou boilerplate — la bonne réponse est `Read` (fichier complet) → `Write` (fichier complet mis à jour), pas des astuces regex de plus en plus créatives.

## Points d'attention style examen

- "Les deux outils ont des descriptions minimales" → la première correction est **développer les descriptions** (entrées, exemples, cas limites, frontières). Few-shot prompts, couches de routage et consolidation d'outils sont des distracteurs de mauvais premier pas. (Sample Q2.)
- "L'agent de synthèse a besoin de vérifications simples fréquentes" → lui donner un **outil `verify_fact` cadré** ; ne pas lui remettre tout le jeu web search. Batching et caching spéculatif sont faux. (Sample Q9.)
- `tool_choice: any` ≠ "n'importe quel outil que je veux" — cela signifie *Claude doit appeler un outil*, pas du texte libre.
- L'outil forcé (`{"type": "tool", "name": "..."}`) n'est pas compatible avec extended thinking.
- `Glob` trouve des *fichiers*, `Grep` trouve du *contenu*. `**/*.test.tsx` est un motif Glob.
- Échecs `Edit` dus à des ancres non uniques → `Read` + `Write`, pas des retries `Edit` répétés.
- La taille du jeu d'outils compte : 18 outils, c'est trop pour un agent ; 4–5 est la cible. Au-delà de 30–50 sur la session, activez Tool Search.
- `disable_parallel_tool_use: true` appartient à la requête qui émet le bloc `tool_use`.

## Références

- [Define tools — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- [Tool reference — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference)
- [Parallel tool use — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use)
- [Strict tool use — Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use)
- [Writing effective tools for AI agents — Anthropic Engineering](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Tools reference — Claude Code Docs](https://docs.claude.com/en/docs/claude-code/tools-reference)
- [Scale to many tools with tool search — Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/tool-search)
- [Configure permissions — Claude Code Docs](https://code.claude.com/docs/en/permissions)
- [anthropics/skills — claude-api/shared/tool-use-concepts.md](https://github.com/anthropics/skills/blob/main/skills/claude-api/shared/tool-use-concepts.md)
