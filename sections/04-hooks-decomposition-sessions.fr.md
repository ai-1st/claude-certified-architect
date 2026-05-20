---
title: "Section 4 — Hooks Agent SDK, décomposition de tâches et gestion de sessions"
linkTitle: "4. Hooks, décomposition et sessions"
weight: 4
description: "Domaines 1.5–1.7 — hooks PostToolUse et d'interception d'appels, prompt chaining vs décomposition adaptative, fork_session et reprise de session nommée."
---

## Ce que couvre cette section

Trois compétences de niveau architecte étroitement liées, qui transforment un agent Claude de chatbot probabiliste en système contrôlable et débogable :

1. **Hooks (1.5)** — callbacks Python/TypeScript déterministes (ou scripts shell) qui interceptent la boucle agentique à des points de cycle de vie bien définis pour imposer une politique, normaliser la sortie d'outils et auditer chaque action.
2. **Décomposition de tâches (1.6)** — savoir quand câbler en dur un pipeline séquentiel (prompt chaining) vs laisser le modèle générer dynamiquement ses propres sous-tâches (orchestrator-workers / plans adaptatifs).
3. **Gestion de sessions (1.7)** — discipline opérationnelle autour de `--continue`, `--resume` et `--fork-session`, plus le jugement de *quand* jeter une session et repartir avec un résumé structuré.

Critère de réussite : regarder un workflow et dire immédiatement "c'est un problème de hook, pas de prompt", "c'est du prompt chaining, pas de l'orchestrator-workers", ou "cette session est périmée, résumer et redémarrer".

## Matériel source (guide officiel)

### 1.5 Hooks pour interception et normalisation

`PostToolUse` intercepte les *résultats* d'outils et les transforme avant que le modèle voie les octets bruts. `PreToolUse` intercepte les *appels* d'outils et peut bloquer, modifier ou rediriger — exemple canonique : "bloquer tout remboursement où `amount > 500` et router vers une escalade humaine." Les hooks donnent des **garanties déterministes** ; les prompts ne donnent qu'une **conformité probabiliste**. Compétences : normaliser des timestamps hétérogènes entre serveurs MCP ; bloquer les actions violant une politique et rediriger vers des alternatives ; choisir les hooks plutôt que les prompts quand la conformité doit être garantie.

### 1.6 Stratégies de décomposition des tâches

**Prompt chaining** pour les workflows prévisibles où les étapes sont connues à l'avance vs **décomposition adaptative dynamique** pour les workflows ouverts où les sous-tâches ne peuvent être découvertes qu'à l'exécution. Grandes revues de code : analyse locale par fichier + passe d'intégration cross-file séparée pour éviter la dilution de l'attention.

### 1.7 État de session, reprise et forking

`--resume <id-or-name>` continue une conversation antérieure précise ; `--continue`/`-c` reprend la plus récente dans ce `cwd`. `--fork-session` (`fork_session: true` / `forkSession: true` dans le SDK) crée une branche indépendante depuis une base partagée. Après des changements de fichiers sur disque, informez un agent repris ou ses résultats `Read` mis en cache seront périmés ; une session fraîche alimentée par un résumé fait main est parfois plus fiable qu'une reprise avec des résultats d'outils périmés.

## Référence des hooks

### Types d'événements de hooks

| Événement | Quand il se déclenche | Usage typique |
| --- | --- | --- |
| `SessionStart` | La session commence ou reprend (matchers : `startup`, `resume`, `clear`, `compact`) | Initialiser les logs, injecter les règles projet |
| `SessionEnd` | La session se termine | Flusher les logs, nettoyer |
| `UserPromptSubmit` | L'utilisateur soumet un prompt, avant que le modèle le voie | Injecter du contexte, supprimer la PII, bloquer les prompts hors sujet |
| `PreToolUse` | Avant l'exécution de tout appel d'outil | **Enforcement de politique**, réécriture d'entrée, redirection sandbox |
| `PostToolUse` | Après la réussite d'un appel d'outil | **Normalisation des données**, journal d'audit, conversion de format |
| `PostToolUseFailure` | Après l'échec d'un appel d'outil | Gestion d'erreur personnalisée |
| `PostToolBatch` | Un lot parallèle d'appels d'outils se résout | Injecter des conventions une fois par lot |
| `PermissionRequest` / `PermissionDenied` | Dialogue d'autorisation ou refus auto-mode | UX personnalisée, décisions de retry |
| `SubagentStart` / `SubagentStop` | Un subagent démarre / termine | Suivre le travail parallèle, agréger les résultats |
| `PreCompact` / `PostCompact` | Cycle de vie de compaction de conversation | Archiver la transcription avant résumé lossy |
| `Notification` | L'agent émet une notification | Relayer vers Slack/PagerDuty |
| `Stop` / `StopFailure` | Le tour se termine normalement / via erreur API | Sauvegarder l'état, alerter sur rate-limit |
| `TaskCreated` / `TaskCompleted` | Cycle de vie des tâches | Imposer les conventions d'ID ticket, gate sur tests réussis |
| `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `FileChanged`, `WorktreeCreate/Remove`, `Setup` | Cycle de vie divers | Auditer, recharger la config, réagir aux changements externes |

`SessionStart`/`SessionEnd` sont des callbacks TS-SDK uniquement ; en Python ils doivent être des hooks shell dans `.claude/settings.json` plus `setting_sources=["project"]`.

### Anatomie d'un hook

Deux chemins d'enregistrement : **callbacks SDK** (`ClaudeAgentOptions.hooks` / `options.hooks`) ou **hooks shell-command** dans `.claude/settings.json` — processus enfants qui reçoivent le JSON d'événement sur stdin.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }
        ]
      }
    ]
  }
}
```

**Matchers** : `*`/`""`/omis correspond à tout ; lettres+chiffres+`|` est exact / liste pipe (`Edit|Write`) ; tout le reste est une regex JS (`^mcp__memory__`). Les hooks d'outils correspondent seulement au *nom* de l'outil — filtrez `file_path` dans le handler.

**Entrée** — chaque hook reçoit `session_id`, `cwd`, `hook_event_name` plus des champs spécifiques à l'événement (`tool_name`, `tool_input`, `tool_response`, …). Le contexte subagent ajoute `agent_id`, `agent_type`.

**Sortie** — JSON en deux couches : niveau supérieur (`systemMessage`, `continue` / `continue_`, `additionalContext`) et `hookSpecificOutput` (dépendant de l'événement). Pour `PreToolUse` : `permissionDecision` ∈ `{"allow", "deny", "ask", "defer"}`, `permissionDecisionReason`, `updatedInput`. Pour `PostToolUse` : `additionalContext` (ajouter) ou `updatedToolOutput` (remplacer).

**Codes de sortie shell-hook** : `0` = succès, parser stdout comme JSON ; `2` = erreur bloquante, stderr fourni au modèle (pour `PreToolUse`, bloque l'appel) ; tout autre non-zéro = erreur non bloquante.

Lorsque plusieurs hooks se déclenchent sur le même événement : **deny > defer > ask > allow** — un seul `deny` bloque.

### Exemples concrets

**1. PostToolUse normalisant des formats de date hétérogènes** — trois serveurs MCP renvoient des secondes Unix epoch, des chaînes ISO 8601 et des entiers numériques en millisecondes. Forcer une représentation ISO 8601 unique avant que le modèle ait à raisonner dessus.

```python
from datetime import datetime, timezone

async def normalize_timestamps(input_data, tool_use_id, context):
    if input_data["hook_event_name"] != "PostToolUse":
        return {}
    response = input_data.get("tool_response", {})
    raw_ts = response.get("timestamp")
    if raw_ts is None:
        return {}

    if isinstance(raw_ts, (int, float)):
        ts = raw_ts / 1000 if raw_ts > 1e12 else raw_ts
        iso = datetime.fromtimestamp(ts, tz=timezone.utc).isoformat()
    else:
        iso = datetime.fromisoformat(str(raw_ts).replace("Z", "+00:00")).isoformat()

    response["timestamp"] = iso
    return {
        "hookSpecificOutput": {
            "hookEventName": "PostToolUse",
            "updatedToolOutput": response,
        }
    }

options = ClaudeAgentOptions(
    hooks={"PostToolUse": [HookMatcher(matcher="^mcp__", hooks=[normalize_timestamps])]}
)
```

Le modèle ne voit jamais que de l'ISO 8601, donc l'arithmétique de dates cross-tool fonctionne directement.

**2. PreToolUse bloquant les remboursements à forte valeur** — garantir que `process_refund` ne puisse jamais s'exécuter au-dessus de 500 $.

```typescript
import { HookCallback, PreToolUseHookInput } from "@anthropic-ai/claude-agent-sdk";

const refundGuard: HookCallback = async (input) => {
  if (input.hook_event_name !== "PreToolUse") return {};
  const pre = input as PreToolUseHookInput;
  if (pre.tool_name !== "mcp__billing__process_refund") return {};

  const amount = (pre.tool_input as { amount?: number }).amount ?? 0;
  if (amount > 500) {
    return {
      systemMessage: `Refund of $${amount} exceeds policy cap; escalating.`,
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny",
        permissionDecisionReason:
          "Refunds above $500 require human approval. Use mcp__support__create_escalation instead."
      }
    };
  }
  return {};
};
```

`permissionDecisionReason` est l'ingrédient magique — il indique au modèle *pourquoi* l'appel a été bloqué et quel outil alternatif utiliser, afin que l'agent s'auto-corrige au tour suivant au lieu de boucler sur le même appel refusé.

### Quand NE PAS utiliser un hook (utiliser un prompt)

| Situation | Hook | Prompt |
| --- | --- | --- |
| Règle réglementaire stricte ("never log SSNs") | Oui | Non |
| Contrat de forme des données ("always ISO 8601") | Oui | Non |
| Journal d'audit de chaque appel d'outil | Oui | Non |
| Quota / plafond de coût par appel | Oui (`PreToolUse`) | Non |
| Préférence de style souple ("prefer 2-space indent") | Non | Oui |
| Persona / ton ("be concise, no emojis") | Non | Oui |

Règle pratique : si sa violation est un incident P0, cela va dans un hook.

## Motifs de décomposition

### Prompt chaining (séquentiel, prévisible)

Un pipeline fixe en N étapes où l'étape *k* alimente l'étape *k+1*. Chaque appel LLM fait une tâche plus simple qu'un méga-prompt unique. Depuis [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) : *"ideal for situations where the task can be easily and cleanly decomposed into fixed subtasks. The main goal is to trade off latency for higher accuracy, by making each LLM call an easier task."*

Ajoutez des *gates* programmatiques entre les étapes pour échouer vite : outline → vérifier la rubrique (gate) → rédiger le document ; analyse lint par fichier → revue d'intégration cross-file.

### Décomposition adaptative dynamique

Aussi appelée **orchestrator-workers**. Le LLM orchestrateur regarde l'entrée, décide quelles sous-tâches sont nécessaires (il ne pouvait pas le savoir à l'avance), lance des workers et synthétise les résultats. *"Well-suited for complex tasks where you can't predict the subtasks needed (in coding, the number of files that need to be changed and the nature of the change in each file likely depend on the task)."*

Dans l'Agent SDK, cela correspond au motif `Task` / subagent, éventuellement suivi via des hooks `SubagentStart`/`SubagentStop`.

### Matrice de décision

| Forme du workflow | Motif |
| --- | --- |
| Étapes connues à l'avance, identiques pour chaque entrée | Prompt chaining |
| Sous-tâches indépendantes de forme *connue* | Parallelization (sectioning) |
| Même tâche, on veut N votes pour la confiance | Parallelization (voting) |
| Catégories distinctes, chacune avec un spécialiste | Routing |
| Nombre/forme des sous-tâches dépend de l'entrée | Orchestrator-workers (adaptive) |
| La sortie bénéficie d'une boucle critique | Evaluator-optimizer |
| Ouvert, multi-tour, horizon inconnu | Full agent loop |

### Exemple travaillé — revue de code par fichier + cross-file

Une PR de 40 fichiers comprimée dans un seul prompt souffre de dilution de l'attention : le modèle survole et manque des bugs. Décomposer :

**Phase 1 — analyse locale par fichier (chaînée, parallélisable)** : chaque fichier reçoit son propre contexte, forké depuis une session de base partagée qui a déjà chargé `CLAUDE.md`, la description de PR et le diff stat. Le forking évite de repayer les tokens pour réétablir le contexte par fichier.

```python
async def review_file(path: str, baseline_sid: str) -> dict:
    async for msg in query(
        prompt=f"Review {path}. Find correctness bugs, missing error handling, "
               f"and security issues. Output JSON {{'file','findings'}}.",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Grep"],
            resume=baseline_sid,
            fork_session=True,
            max_turns=8,
        ),
    ):
        if isinstance(msg, ResultMessage) and msg.subtype == "success":
            return json.loads(msg.result)
```

**Phase 2 — passe d'intégration cross-file (appel unique) :**

```python
findings = await asyncio.gather(*(review_file(p, baseline_sid) for p in changed_files))
async for msg in query(
    prompt=f"Per-file findings: {json.dumps(findings)}. "
           f"Identify cross-cutting issues: API contract drift, type mismatches, "
           f"missing call-site updates, security holes spanning files.",
    options=ClaudeAgentOptions(resume=baseline_sid, allowed_tools=["Read", "Grep"]),
):
    ...
```

Prompt chaining (Phase 1 → Phase 2) superposé à la parallélisation dans la Phase 1.

**Variante ouverte — "add tests to a legacy codebase" :** *adaptative*, pas chaînée. Cartographier la structure → identifier les modules non testés à fort impact → construire un backlog priorisé → prendre le premier item, écrire des tests, découvrir une dépendance cachée, le remettre dans le backlog, répéter. Les étapes 4..N ne sont connaissables qu'à l'exécution — orchestrator-workers, pas chaining.

## Gestion de sessions

### `--resume` vs `--continue` vs nouvelle session

| Besoin | Utiliser | Comment la session est trouvée |
| --- | --- | --- |
| Session la plus récente dans ce répertoire | `claude -c` / `continue: true` | La plus récente dans `~/.claude/projects/<cwd-slug>/` |
| Session nommée ou identifiée précise | `claude -r "auth-refactor"` / `resume: "<id>"` | ID exact ou lookup `--name` |
| Conversation neuve | `claude` | ID de session frais |
| One-shot, pas de persistance disque (TS uniquement) | `persistSession: false` | Mémoire seulement |

Les sessions vivent dans `~/.claude/projects/<slugified-cwd>/<session-id>.jsonl`, où le slug est le répertoire de travail absolu avec chaque caractère non alphanumérique remplacé par `-`. **Un `cwd` différent signifie que `resume` ne peut pas trouver le fichier** — la cause n°1 de "pourquoi resume renvoie une session fraîche".

Capturez l'ID de session depuis le `ResultMessage` (Python) / `SDKResultMessage` (TS) à chaque exécution si vous comptez reprendre par programme. En TS, il est aussi sur le `SystemMessage` init.

### `fork_session` : quand et comment

Forker copie la transcription existante vers un *nouvel* ID de session et la laisse diverger. L'original est intact.

```python
forked_id = None
async for message in query(
    prompt="Try OAuth2 instead of JWT for the auth module",
    options=ClaudeAgentOptions(resume=session_id, fork_session=True),
):
    if isinstance(message, ResultMessage):
        forked_id = message.session_id
```

```typescript
for await (const message of query({
  prompt: "Try OAuth2 instead of JWT for the auth module",
  options: { resume: sessionId, forkSession: true }
})) {
  if (message.type === "system" && message.subtype === "init") {
    forkedId = message.session_id;
  }
}
```

Cas d'usage : comparer A/B deux approches de refactoring depuis une base partagée ; bake-off de stratégie de tests ; exploration risquée avec retour garanti au parent.

Mise en garde : le forking branche la *conversation*, pas le *filesystem*. Si les deux forks éditent des fichiers dans le même dépôt, ces éditions entrent en collision. Combinez avec [file checkpointing](https://code.claude.com/docs/en/agent-sdk/file-checkpointing) ou des git worktrees (`claude -w <name>`) pour une vraie isolation.

### Arbre de décision contexte périmé

Après un changement de code, avant de reprendre une investigation :

```
Did the agent's last tool calls touch files that have since been edited?
├── No  → Resume normally.
└── Yes →
    Is the affected surface small AND were the edits surgical?
    ├── Yes → Resume + tell the agent: "Files X, Y were edited since
    │         you last saw them. Re-Read them before continuing."
    └── No  → Throw the session away. Start fresh with a structured
              summary: (a) goal, (b) decisions made, (c) current
              state of the code, (d) open questions.
```

Une session propre avec un résumé sélectionné bat souvent une reprise parce que la transcription reprise contient encore des sorties `Read` périmées auxquelles le modèle fait confiance — il peut "se souvenir" de signatures de fonctions qui n'existent plus et halluciner des appels. Motif : garder une **session nommée** longue durée (`claude -n design-review`) pour le contexte architectural stable et la **forker** par investigation. Jeter les forks.

## Points d'attention style examen

- **Les hooks sont déterministes ; les prompts sont probabilistes.** Toute règle qui doit tenir 100% du temps va dans `PreToolUse` (sortant) ou `PostToolUse` (entrant).
- Mémorisez les valeurs `permissionDecision` : `allow`, `deny`, `ask`, `defer`. Mémorisez la priorité : **deny > defer > ask > allow**.
- `PostToolUse.updatedToolOutput` *remplace* ce que le modèle voit ; `additionalContext` *ajoute*. `PreToolUse.updatedInput` réécrit l'entrée d'outil — mais seulement si vous renvoyez aussi `permissionDecision: "allow"`.
- Codes de sortie shell-hook : `0` = parser JSON ; `2` = bloquer (`PreToolUse`) / fournir stderr au modèle ; tout autre = erreur non bloquante.
- Sélecteur de décomposition : forme connue → prompt chaining ; forme inconnue → orchestrator-workers. Grosse revue de code = passe par fichier + passe d'intégration cross-file.
- `--continue` n'a pas besoin d'ID mais trouve seulement la session la plus récente dans le `cwd` *actuel*. `--resume` a besoin d'un ID ou `--name`. `--fork-session` nécessite `--resume`/`--continue` et produit un nouvel ID.
- Stockage de session : `~/.claude/projects/<slugified-cwd>/<session-id>.jsonl`. Un mauvais `cwd` est la raison n°1 pour laquelle resume renvoie silencieusement une session fraîche.
- Des résultats d'outils périmés dans une session reprise peuvent faire plus de mal que de bien — parfois une session fraîche avec résumé sélectionné surpasse une reprise.
- Fork pour comparer ; resume pour continuer ; redémarrer avec résumé quand le contexte a pourri.

## Références

- [Intercept and control agent behavior with hooks](https://code.claude.com/docs/en/agent-sdk/hooks) — référence complète des hooks Agent SDK, table d'événements, forme des callbacks, exemples.
- [Hooks reference](https://code.claude.com/docs/en/hooks) — chaque événement, motifs de matcher, sortie JSON, sémantique des codes de sortie, variantes shell/HTTP/MCP-tool.
- [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide) — quickstart avec exemples travaillés.
- [Configure hooks (Anthropic blog)](https://claude.com/blog/how-to-configure-hooks) — walkthrough power-user `settings.json`.
- [Work with sessions](https://code.claude.com/docs/en/agent-sdk/sessions) — `continue`, `resume`, `fork_session`, capture d'IDs, mises en garde cross-host.
- [CLI reference](https://code.claude.com/docs/en/cli-reference) — `--continue`, `--resume`, `--fork-session`, `--name`, `--session-id`, `--from-pr`.
- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — taxonomie canonique d'Anthropic : prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer.
- [Anthropic cookbook — agents patterns](https://github.com/anthropics/claude-cookbooks/tree/main/patterns/agents) — notebooks exécutables pour chaque motif.
- [How the agent loop works](https://code.claude.com/docs/en/agent-sdk/agent-loop) — modèle conceptuel des turns, appels d'outils, où les hooks s'insèrent.
- [Permissions](https://code.claude.com/docs/en/agent-sdk/permissions) — mécanisme compagnon de `PreToolUse` pour le contrôle d'accès aux outils.
