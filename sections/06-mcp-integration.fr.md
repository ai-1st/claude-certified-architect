---
title: "Section 6 — Intégration MCP : erreurs, serveurs et ressources"
linkTitle: "6. Intégration MCP"
weight: 6
description: "Domaines 2.2, 2.4 — enveloppes d'erreur structurées (isError, isRetryable), portées .mcp.json et expansion env, ressources vs outils."
---

## Ce que couvre cette section

Deux domaines MCP critiques en production : comment les outils doivent *signaler les échecs* afin que l'agent récupère intelligemment (2.2), et comment serveurs et ressources doivent être câblés dans Claude Code afin qu'une équipe ait les bonnes capacités au bon périmètre (2.4). MCP est un contrat entre un serveur et un agent — la qualité de ce contrat (erreurs, descriptions, ressources) détermine si l'agent prend de bonnes décisions ou s'agite inutilement.

## Matériel source (guide officiel)

### 2.2 Réponses d'erreur structurées

Connaissances : les quatre classes d'erreur (transient, validation, business, permission) ; pourquoi les réponses uniformes `"Operation failed"` cassent la récupération ; retryable vs non-retryable.

Compétences : renvoyer `errorCategory`, `isRetryable` et une description lisible par humain ; marquer les violations de règle métier avec `retriable: false` plus une explication friendly client ; faire gérer aux subagents la récupération locale des erreurs transitoires et n'escalader que ce qui ne peut pas être résolu localement (avec résultats partiels et ce qui a été tenté) ; distinguer les *échecs d'accès* des *résultats vides valides*.

### 2.4 Intégration serveur

Connaissances : portée projet (`.mcp.json`) vs utilisateur (`~/.claude.json`) ; expansion `${VAR}` pour les secrets ; les outils de tous les serveurs sont découverts au moment de la connexion et disponibles simultanément ; les ressources exposent des catalogues de contenu pour réduire les appels d'outils exploratoires.

Compétences : configurer des serveurs partagés dans `.mcp.json` avec expansion de variables d'environnement ; configurer des serveurs personnels en portée utilisateur ; écrire des descriptions d'outils riches pour éviter que l'agent retombe sur des built-ins comme `Grep` ; préférer les serveurs communautaires (Jira, GitHub, Postgres) au custom ; exposer les catalogues de contenu comme ressources.

## MCP à 30 000 pieds

Le [Model Context Protocol](https://modelcontextprotocol.io/specification/latest) est un protocole ouvert JSON-RPC 2.0 qui connecte les hôtes LLM à des fournisseurs de capacités (serveurs) via une session stateful avec négociation de capacités. Les serveurs exposent trois primitives :

| Primitive | Contrôlée par | Objectif | Exemple |
| --- | --- | --- | --- |
| **Tool** | Modèle | Action exécutable avec effets de bord | `create_issue`, `run_query`, `send_email` |
| **Resource** | Application / utilisateur | Contenu en lecture seule identifié par URI | `jira://issues/PROJ-123`, `postgres://schema/orders` |
| **Prompt** | Utilisateur | Modèles préconstruits invoqués par choix utilisateur (slash commands, menus) | `/code-review`, `/summarize-incident` |

Transports définis dans la spec : **stdio** (sous-processus local), **Streamable HTTP** (appelé `streamable-http` dans la spec, alias `http` dans la config Claude Code), et le transport historique **SSE**. Voir la [référence de schéma](https://modelcontextprotocol.io/specification/2025-11-25/schema) pour le format wire.

## Réponses d'erreur structurées

### Taxonomie des catégories d'erreur

| Catégorie | Signification | `isRetryable` typique | Ce que l'agent doit faire |
| --- | --- | --- | --- |
| `transient` | Timeout, 5xx, rate limit, connection reset | `true` | Backoff et retry local dans le subagent |
| `validation` | Schéma incompatible, mauvais enum, champ manquant | `false` | Reformuler l'entrée ; ne pas rejouer le même appel |
| `business` | Violation de politique/limite/état ("refund > $500", "ticket already closed") | `false` | Exposer l'explication client ; arrêter |
| `permission` | 401/403, scope manquant, RBAC refusé | `false` | Escalader vers coordinateur ou humain |
| `not_found` | Requête valide, zéro match | `false` (et `isError` est discutable — voir ci-dessous) | Traiter comme résultat vide légitime, pas échec |

### Drapeau `isError` — schéma et exemple

La forme officielle [`CallToolResult`](https://modelcontextprotocol.io/specification/2025-11-25/schema) :

```ts
interface CallToolResult {
  content: ContentBlock[];
  structuredContent?: { [key: string]: unknown };
  isError?: boolean;  // default false
  _meta?: { [key: string]: unknown };
}
```

La spec est explicite : *les erreurs venant de l'outil lui-même* doivent être signalées dans le résultat avec `isError: true`, **pas** comme erreur de protocole JSON-RPC — les erreurs de protocole sont invisibles au modèle, alors que le LLM doit voir l'erreur pour s'auto-corriger. Les erreurs de protocole sont réservées à "tool not found" ou "server doesn't support tool calls".

Un échec d'outil bien structuré ressemble à ceci :

```json
{
  "isError": true,
  "content": [
    { "type": "text", "text": "Refund of $750 exceeds the $500 single-transaction policy. Ask the customer to split the refund or open a manager-approval ticket." }
  ],
  "structuredContent": {
    "errorCategory": "business",
    "isRetryable": false,
    "code": "REFUND_LIMIT_EXCEEDED",
    "limit": 500,
    "requested": 750,
    "customerMessage": "We can only process refunds up to $500 in one transaction."
  }
}
```

Comparez à l'anti-pattern — `{ "isError": true, "content": [{ "type": "text", "text": "Operation failed" }] }` — qui force l'agent à réessayer aveuglément ou à abandonner.

### Arbre de décision retryable vs non-retryable

```
isError === true ?
├── No  → success path (or empty-result path if content is intentionally empty)
└── Yes
    ├── errorCategory === "transient" && isRetryable === true
    │     → backoff + retry locally (subagent), cap attempts (e.g. 3)
    ├── errorCategory === "validation"
    │     → re-plan with corrected input; do NOT replay the same call
    ├── errorCategory === "permission"
    │     → escalate to coordinator with the missing scope/role
    └── errorCategory === "business"
          → surface customerMessage; stop; do NOT retry
```

Les métadonnées structurées rendent cet arbre exécutable. Sans `errorCategory` et `isRetryable`, l'agent doit *deviner* si un échec mérite un retry, ce qui est la plus grande cause de boucles runaway de type "tool storm".

### Récupération locale par subagent vs escalade coordinateur

Un subagent possède la récupération locale des erreurs transitoires — retry avec backoff, fallback vers un endpoint secondaire, lecture cache stale — et n'escalade vers le haut que lorsque la classe d'erreur est non retryable *ou* que le budget de retry local est épuisé. Lorsqu'il escalade, il renvoie une enveloppe structurée incluant le `errorCategory`/`isRetryable` final, une description pour les logs du coordinateur, les **résultats partiels** collectés avant l'échec, et **ce qui a été tenté** (quels endpoints, combien de retries) afin que le coordinateur ne refasse pas le travail.

Point critique : un *résultat vide valide* (par exemple "aucune commande ne correspond à ce filtre") n'est **pas** une erreur. Confondre "je n'ai pas pu exécuter la requête" avec "la requête a renvoyé zéro ligne" pousse le coordinateur à réessayer inutilement ou à mal rapporter l'état à l'utilisateur.

## Configuration des serveurs MCP dans Claude Code

### Portées : local / project / user

La documentation MCP de [Claude Code](https://docs.claude.com/en/docs/claude-code/mcp) définit trois portées :

| Portée | Stockage | Partagé via git ? | Usage |
| --- | --- | --- | --- |
| `local` (par défaut) | `~/.claude.json`, indexé par chemin de projet courant | Non | Serveurs personnels/expérimentaux dans un projet, secrets à ne pas partager |
| `project` | `.mcp.json` à la racine du dépôt | **Oui** | Outillage partagé par l'équipe (Jira, APIs internes, bot de déploiement de l'équipe) |
| `user` | `~/.claude.json`, cross-project | Non | Utilitaires personnels partout (votre scratch DB, serveur de notes) |

Ajout via CLI :

```bash
claude mcp add --scope project --transport http jira https://mcp.atlassian.com/v1/sse
claude mcp add --scope user    --transport stdio notes -- npx -y @me/notes-mcp
claude mcp add --scope local   --transport stdio scratch -- python ./scratch_server.py
```

Quand le même nom de serveur existe à plusieurs portées, **local gagne, puis project, puis user**, puis les serveurs fournis par plugins, puis les connecteurs `claude.ai`. Cette précédence permet à un développeur de surcharger localement une config partagée par l'équipe sans éditer `.mcp.json`.

### Schéma `.mcp.json` (avec exemple)

Un `.mcp.json` à portée projet vit à la racine du dépôt et est committé. Schéma minimal :

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp",
      "headers": {
        "Authorization": "Bearer ${GITHUB_PERSONAL_ACCESS_TOKEN}"
      }
    },
    "postgres-readonly": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL_RO}"],
      "env": {
        "PGSSLMODE": "require"
      }
    },
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```

Le `type` de chaque entrée est l'un de `stdio`, `http` (alias `streamable-http`) ou `sse`. Pour `stdio`, vous fournissez `command` + `args` et `env` optionnels. Pour `http`/`sse`, vous fournissez `url` et `headers` optionnels. Les serveurs à portée projet demandent l'approbation utilisateur la première fois qu'une session les charge (réinitialiser avec `claude mcp reset-project-choices`).

### Expansion des variables d'environnement

Claude Code développe deux formes dans `.mcp.json` :

- `${VAR}` — valeur de `VAR` ; **échoue au parsing** si non définie
- `${VAR:-default}` — valeur de `VAR` si définie, sinon `default`

L'expansion fonctionne dans `command`, `args`, `env`, `url` et `headers`. Motif recommandé : committer `.mcp.json` avec des placeholders `${GITHUB_TOKEN}` et garder les vrais tokens dans le shell de chaque développeur, le runner CI ou 1Password CLI. Piège : `${CLAUDE_PROJECT_DIR}` est défini dans l'environnement du *serveur*, pas celui de Claude Code — le référencer dans un `command` ou `args` de niveau supérieur nécessite généralement `${CLAUDE_PROJECT_DIR:-.}` comme défaut.

### Vérifier la connexion (commande `/mcp`)

Dans Claude Code, `/mcp` liste chaque serveur connecté, le nombre d'outils/ressources/prompts annoncés, son état d'auth et tout backoff de reconnexion en cours. Les serveurs HTTP/SSE se reconnectent automatiquement jusqu'à cinq fois avec backoff exponentiel ; les serveurs stdio, étant des sous-processus locaux, non. `/mcp` est aussi l'endroit où lancer le flux OAuth pour les serveurs distants qui renvoient `401`/`403` avec un header `WWW-Authenticate`.

## Outils MCP vs ressources MCP

### Quand modéliser quelque chose comme outil

Les outils sont **contrôlés par le modèle** et peuvent avoir des effets de bord. Utilisez un outil lorsque l'agent doit *décider* de faire quelque chose :

- `create_jira_issue`, `update_pr_status`, `run_sql(query)`, `send_slack_message`
- Toute action qui mute l'état, appelle une API externe ou exige des arguments sur lesquels le modèle doit raisonner

### Quand le modéliser comme ressource

Les ressources sont **contrôlées par l'application**, en lecture seule, identifiées par URI et idéalement cacheables. Utilisez une ressource lorsque l'agent bénéficie d'un *contexte passif* sans devoir le demander :

- Un répertoire de tous les tickets Jira ouverts dans le sprint courant → `jira://sprints/current/issues`
- Le schéma Postgres de la base orders → `postgres://schema/public/orders`
- Le contenu d'un runbook ou design doc → `confluence://pages/12345`

Parce que les ressources sont adressables par URI et ne modifient pas l'état, elles coûtent moins en retries et sont plus favorables au cache que le même contenu emballé en appel d'outil.

### Exemples (catalogue d'issues, introspection de schéma)

Le motif "content catalog" du guide : au lieu de forcer l'agent à appeler `list_issues` puis `get_issue` pour chaque correspondance (un N+1 exploratoire), le serveur expose une ressource `issues://summary` que l'hôte peut attacher automatiquement. L'agent voit le catalogue dans le contexte et saute directement à l'appel `get_issue` pertinent. La même idée vaut pour l'introspection de schéma — exposer `db://schema` comme ressource transforme "quelles colonnes a `orders` ?" en réponse zéro appel d'outil.

## Choisir serveurs communautaires vs custom

### Quand construire le vôtre

Construisez un serveur MCP custom uniquement lorsque l'intégration est **spécifique à l'équipe** et qu'aucune option communautaire maintenue n'existe : microservice interne, pipeline de déploiement, CRM maison. Le coût d'un serveur custom n'est pas le code — c'est la maintenance continue : dérive de schéma, refresh d'auth, mises à niveau de transport, revue sécurité.

### Serveurs populaires à connaître

Le dépôt [`modelcontextprotocol/servers`](https://github.com/modelcontextprotocol/servers) contient les implémentations de référence et des liens vers le [MCP Registry](https://registry.modelcontextprotocol.io/) officiel. Noms à retenir : **Filesystem** (ops fichier sandboxées), **Fetch** (URL → texte lisible par LLM), **Git**, **Memory** (persistance knowledge-graph), **Sequential Thinking**, plus **GitHub** hébergé vendeur (`api.githubcopilot.com/mcp`), **Postgres**, **Slack**, **Sentry**, **Notion**, **Atlassian/Jira**, **Stripe**, **PayPal**, **HubSpot**, **Asana** et **Playwright**. Les répertoires communautaires ([mcpforge.org](https://www.mcpforge.org/directory), [mcpindex.net](https://mcpindex.net/), [mcpdir.dev](https://mcpdir.dev/)) cataloguent des milliers d'autres, mais le registre officiel est la source de vérité pour la provenance.

Dernière compétence appelée par le guide : **améliorer les descriptions d'outils MCP**. Si votre outil Jira MCP dit "search Jira" tandis que le built-in `Grep` est décrit avec précision, le modèle choisira `Grep` sur un cache local. Les descriptions d'outils sont le seul signal du modèle pour choisir l'outil — écrivez-les comme la docstring d'une fonction qu'il n'a jamais vue.

## Points d'attention style examen

- `isError: true` vit dans `CallToolResult`, **pas** comme erreur JSON-RPC — parce que le modèle doit *voir* l'échec pour récupérer.
- Incluez toujours `errorCategory` et `isRetryable` dans `structuredContent` ; "Operation failed" est la mauvaise réponse canonique.
- Distinguez les échecs d'accès des résultats vides ; `isError: false` avec `content` vide est un cas valide et courant.
- `.mcp.json` est à portée projet et committé ; `~/.claude.json` est local/user et privé.
- L'expansion `${VAR}` et `${VAR:-default}` sert à garder les secrets hors git, pas à faire de la logique de config runtime.
- Tools = contrôlés par le modèle, avec effets de bord ; Resources = contrôlées par l'application, lecture seule, adressables par URI.
- Préférez les serveurs communautaires ; les serveurs custom servent seulement aux workflows spécifiques à l'équipe.
- Les subagents récupèrent localement les erreurs transitoires ; seules les erreurs non récupérables (avec résultats partiels et log des tentatives) remontent au coordinateur.

## Références

- MCP Specification (latest): https://modelcontextprotocol.io/specification/latest
- MCP Schema (2025-11-25): https://modelcontextprotocol.io/specification/2025-11-25/schema
- MCP Server Primitives Overview: https://modelcontextprotocol.io/specification/2025-11-25/server
- Claude Code — Connect to tools via MCP: https://docs.claude.com/en/docs/claude-code/mcp
- Claude Code — MCP installation scopes: https://docs.claude.com/en/docs/claude-code/mcp#mcp-installation-scopes
- Claude Code — Environment variable expansion in `.mcp.json`: https://docs.claude.com/en/docs/claude-code/mcp#environment-variable-expansion-in-mcp-json
- Reference servers: https://github.com/modelcontextprotocol/servers
- MCP Registry: https://registry.modelcontextprotocol.io/
