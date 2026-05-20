---
title: "Section 2 — Boucles agentiques et gestion de stop_reason"
linkTitle: "2. Boucles agentiques"
weight: 2
description: "Domaine 1.1 — flux de contrôle des boucles agentiques, valeurs stop_reason à connaître et anti-patterns canoniques."
---

## Ce que couvre cette section

Comment construire le flux de contrôle central d'un agent Claude : envoyer une requête, inspecter `stop_reason`, exécuter les outils demandés par Claude, ajouter les résultats à l'historique, itérer. Chaque motif de plus haut niveau (orchestrator-workers, subagents, evaluator-optimizer, Agent SDK) est construit au-dessus de cette boucle.

## Matériel source (guide officiel)

### Connaissances requises

- Le cycle de vie de la boucle agentique : envoyer une requête à Claude, inspecter `stop_reason` (`"tool_use"` vs `"end_turn"`), exécuter les outils demandés, renvoyer les résultats pour l'itération suivante.
- Comment les résultats d'outils sont ajoutés à l'historique de conversation afin que le modèle puisse raisonner sur l'action suivante.
- La distinction entre **prise de décision pilotée par le modèle** (Claude raisonne sur l'outil à appeler ensuite selon le contexte) et **arbres de décision préconfigurés** (le développeur code en dur la séquence d'outils).

### Compétences requises

- Implémenter un flux de contrôle de boucle agentique qui continue tant que `stop_reason == "tool_use"` et termine lorsque `stop_reason == "end_turn"`.
- Ajouter les résultats d'outils au contexte de conversation entre les itérations afin que le modèle puisse intégrer les nouvelles informations dans son raisonnement.
- Éviter les anti-patterns : parser des signaux en langage naturel pour terminer la boucle, utiliser des plafonds d'itération arbitraires comme mécanisme d'arrêt principal, ou vérifier le texte de l'assistant comme indicateur de fin.

## La boucle agentique, de bout en bout

La définition de travail d'Anthropic pour un agent est la plus simple du domaine : *"LLMs autonomously using tools in a loop."* Le LLM augmenté (modèle + outils + retrieval + mémoire) est le bloc fondamental — chaque motif de workflow (prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer) en est composé.

```text
  ┌──────────────────────────────────────────────────────────────┐
  │  user prompt + tool definitions  ─────────► messages array   │
  └──────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  POST /v1/messages  (Claude reasons about the next action)   │
  └──────────────────────────────────────────────────────────────┘
                              │
                              ▼
            ┌──────  inspect response.stop_reason  ──────┐
            │                                            │
   "tool_use"                                       "end_turn"
            │                                            │
            ▼                                            ▼
  ┌───────────────────────────┐               ┌─────────────────┐
  │ 1. append assistant turn  │               │ return final    │
  │    (incl. tool_use blocks)│               │ text to caller  │
  │ 2. execute each tool      │               └─────────────────┘
  │ 3. append a user turn     │
  │    with tool_result blocks│
  │ 4. loop back to /messages │
  └───────────────────────────┘
```

Déroulé d'une itération :

1. Envoyer `messages` plus le schéma `tools` à `POST /v1/messages`.
2. Claude renvoie un message `assistant`. Son `content` est une liste de blocs : zéro ou plusieurs blocs `text` et zéro ou plusieurs blocs `tool_use`. Le `stop_reason` de niveau supérieur résume pourquoi la génération s'est arrêtée.
3. Si `stop_reason == "tool_use"` : ajouter le tour assistant tel quel, exécuter chaque outil demandé, ajouter un nouveau tour `user` unique dont le contenu est une liste de blocs `tool_result` (un par `tool_use_id`), puis rappeler l'API avec l'historique mis à jour.
4. Si `stop_reason == "end_turn"` : le modèle a décidé que la tâche est terminée. Retourner.

Les résultats d'outils sont **ajoutés à l'historique de conversation**, pas résumés et perdus. Chaque nouvelle requête transporte tout l'historique, afin que Claude puisse chaîner le raisonnement sur de nombreux tours. Le modèle — pas votre code — décide quel outil appeler ensuite selon ce qu'il a observé. C'est la différence entre **prise de décision pilotée par le modèle** (Claude choisit l'outil N+1 depuis le contexte en cours) et **arbres de décision préconfigurés** (votre code appelle statiquement `tool_a()` → `tool_b()` → `tool_c()`). Les arbres de décision sont des workflows ; les boucles agentiques sont des agents. La recommandation publiée d'Anthropic est de préférer le workflow plus simple lorsque le chemin peut être codé en dur.

## Valeurs stop_reason à connaître

`stop_reason` fait partie de chaque réponse Messages API réussie. C'est le seul signal sur lequel vous devez brancher pour décider de continuer la boucle. L'ensemble documenté complet est ci-dessous.

| Valeur | Signification | Ce que votre boucle doit faire |
| --- | --- | --- |
| `end_turn` | Claude a terminé naturellement sa réponse. | Sortir de la boucle. Renvoyer les blocs texte `response.content` à l'appelant. |
| `tool_use` | La réponse contient un ou plusieurs blocs `tool_use` ; Claude s'attend à ce que vous les exécutiez. | Ajouter le tour assistant, exécuter chaque bloc `tool_use`, ajouter un tour `user` avec les blocs `tool_result` correspondants (utiliser le même `tool_use_id`), puis rappeler l'API. |
| `max_tokens` | La sortie a atteint le paramètre `max_tokens`. La réponse est tronquée et peut contenir un bloc `tool_use` **incomplet**. | Détecter la troncature au milieu d'un appel d'outil en vérifiant `type == "tool_use"` sur le dernier bloc de contenu ; réessayer avec un `max_tokens` plus élevé. Sinon, demander une continuation ou exposer un avertissement de troncature. |
| `stop_sequence` | La sortie a correspondu à une chaîne personnalisée dans `stop_sequences`. La séquence correspondante est dans `response.stop_sequence`. | Traiter comme un arrêt terminal réussi pour ce motif. Continuer ou finaliser selon votre protocole. |
| `pause_turn` | La boucle d'échantillonnage côté serveur a atteint son plafond d'itérations pendant l'exécution de **server tools** (web search, web fetch, code execution, etc.). La réponse peut contenir un bloc `server_tool_use` sans `server_tool_result` correspondant. | Ajouter la réponse assistant **inchangée** et rappeler l'API avec les mêmes outils. Répéter jusqu'à obtenir un stop reason différent de `pause_turn`. |
| `refusal` | Le modèle a refusé pour raisons de sécurité (filtre de sécurité API Sonnet 4.5+ / Opus 4.1+). | Ne pas boucler. Exposer un refus à l'appelant ; éventuellement reformuler, router vers un autre modèle (p. ex. Haiku 4.5) ou escalader. |
| `model_context_window_exceeded` | La génération s'est arrêtée car la réponse a atteint la fenêtre de contexte complète du modèle (pas `max_tokens`). Sonnet 4.5+ par défaut ; les modèles antérieurs nécessitent un beta header. | Traiter de façon similaire à `max_tokens` — la réponse est valide mais plafonnée. Continuer, résumer ou compacter le contexte. |

Brancher sur `stop_reason` est le **seul** test de terminaison correct. Ne parsez pas des textes comme "I'm done" ou "Final answer:" — c'est l'anti-pattern canonique ci-dessous.

## Implémentations de référence

### Python — boucle brute Messages API

Forme minimale et exécutable avec le SDK Python `anthropic` (le même motif de boucle qu'Anthropic montre dans [sa documentation](https://docs.anthropic.com/en/docs/build-with-claude/handling-stop-reasons#handling-tool-use-workflows)).

```python
from anthropic import Anthropic

client = Anthropic()
MODEL = "claude-opus-4-7"

tools = [{
    "name": "get_weather",
    "description": "Get current weather for a city.",
    "input_schema": {
        "type": "object",
        "properties": {"location": {"type": "string"}},
        "required": ["location"],
    },
}]

def run_tool(name: str, tool_input: dict) -> str:
    if name == "get_weather":
        return f"Weather in {tool_input['location']}: 72F, clear"
    raise ValueError(f"unknown tool: {name}")

def agent_loop(user_prompt: str) -> str:
    messages = [{"role": "user", "content": user_prompt}]
    while True:
        resp = client.messages.create(
            model=MODEL, max_tokens=4096, tools=tools, messages=messages,
        )
        if resp.stop_reason == "end_turn":
            return "".join(b.text for b in resp.content if b.type == "text")
        if resp.stop_reason == "pause_turn":
            messages.append({"role": "assistant", "content": resp.content})
            continue
        if resp.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": resp.content})
            tool_results = [
                {"type": "tool_result", "tool_use_id": b.id,
                 "content": run_tool(b.name, b.input)}
                for b in resp.content if b.type == "tool_use"
            ]
            messages.append({"role": "user", "content": tool_results})
            continue
        raise RuntimeError(f"unhandled stop_reason: {resp.stop_reason}")
```

Notes : le tour assistant est ajouté **verbatim** (les blocs `tool_use` doivent survivre dans l'historique). Les résultats d'outils sont renvoyés dans un seul message `user` dont le `content` est une **liste** de blocs `tool_result`, un par `tool_use_id`. `pause_turn` exige de renvoyer le contenu assistant inchangé ; ne synthétisez pas de résultat d'outil.

### TypeScript — Claude Agent SDK

Pour le [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/agent-loop) de plus haut niveau d'Anthropic (`@anthropic-ai/claude-agent-sdk`), la boucle est déjà implémentée pour vous. Vous consommez un flux asynchrone de messages typés et vérifiez le `ResultMessage` terminal.

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const stream = query({
  prompt: "Find the failing tests in auth.ts and fix them.",
  options: {
    model: "claude-opus-4-7",
    maxTurns: 20,
    maxBudgetUsd: 1.0,
    permissionMode: "acceptEdits",
    allowedTools: ["Read", "Edit", "Bash", "Grep", "Glob"],
  },
});

for await (const message of stream) {
  if (message.type === "assistant") {
    console.log(`turn: ${message.message.content.length} blocks`);
  }
  if (message.type === "result") {
    if (message.subtype === "success") {
      console.log("done:", message.result);
    } else {
      console.error("stopped early:", message.subtype);
    }
  }
}
```

Le SDK exécute en interne la même boucle pilotée par `stop_reason` : Claude évalue, demande des outils, le SDK les exécute, les résultats reviennent automatiquement, et un tour Claude complet + exécution d'outil est ce que le SDK appelle un *turn*. La boucle se termine lorsque Claude produit un message assistant sans blocs `tool_use`. `maxTurns` et `maxBudgetUsd` sont des **garde-fous**, pas le mécanisme d'arrêt principal — ils produisent un `ResultMessage` avec le sous-type `error_max_turns` ou `error_max_budget_usd` lorsqu'ils se déclenchent.

## Anti-patterns à éviter

- **Parser des signaux en langage naturel pour terminer la boucle.** Chercher "Final answer:" ou "DONE" dans `response.content` est fragile — le modèle peut formuler la fin d'une infinité de façons et peut encore vouloir appeler un outil. **Correct :** brancher uniquement sur `response.stop_reason`.
- **Utiliser un plafond d'itération comme mécanisme d'arrêt principal.** Coder en dur `for _ in range(10):` et sortir au plafond signifie que vous terminerez en milieu de tâche sur des problèmes difficiles et gaspillerez des tokens sur les faciles. **Correct :** laisser `stop_reason == "end_turn"` terminer la boucle ; garder les plafonds d'itération et `max_budget_usd` comme garde-fous de sécurité uniquement.
- **Vérifier le texte assistant pour décider que c'est terminé.** Un tour peut contenir *à la fois* des blocs `text` et `tool_use` (Claude peut narrer tout en demandant un outil). Traiter "il y a du texte" comme "terminé" abandonne des appels d'outils. **Correct :** inspecter `stop_reason` ; itérer sur les blocs de contenu par `type`.
- **Oublier le tour assistant lorsque vous ajoutez les résultats d'outils.** Envoyer des résultats d'outils sans ajouter d'abord le tour assistant `tool_use` produit un tableau `messages` invalide et une erreur API. **Correct :** ajouter le tour assistant verbatim, puis ajouter un seul tour user de blocs `tool_result`.
- **Ajouter du texte supplémentaire après des blocs `tool_result`.** Des blocs `text` en fin du même tour user apprennent à Claude à attendre du texte utilisateur après chaque appel d'outil, ce qui cause des réponses `end_turn` vides. **Correct :** le tour user après un `tool_use` doit contenir uniquement des blocs `tool_result`.
- **Ignorer `pause_turn`.** Avec des server tools, le serveur atteint son propre plafond de 10 itérations et renvoie `pause_turn` sans `tool_result` à produire. Le traiter comme `end_turn` tronque l'agent. **Correct :** ajouter la réponse assistant inchangée et rappeler.
- **Ignorer une troncature `max_tokens` dans un bloc `tool_use`.** Si `stop_reason == "max_tokens"` et le dernier bloc est `tool_use`, l'entrée JSON est incomplète et réessayer avec la même limite échoue encore. **Correct :** détecter le cas et réessayer avec un `max_tokens` plus élevé.
- **Coder en dur la séquence d'outils.** Appeler `read_file → search → write_file` depuis votre propre code sans raisonnement model-in-the-loop est un **workflow**, pas un agent. Très bien quand le chemin est connu — mais n'attendez pas qu'il récupère sur des entrées nouvelles.

## Points d'attention style examen

- À partir d'une valeur `stop_reason`, identifier l'action de boucle correcte (continuer avec les résultats d'outils, append-and-resend pour `pause_turn`, sortir sur `end_turn`, réessayer avec un budget plus grand sur `max_tokens` au milieu d'un outil, exposer le refus).
- Identifier quelles mutations de `messages` sont requises entre les itérations : ajouter le tour assistant verbatim (incluant les blocs `tool_use`), puis ajouter un tour user de blocs `tool_result` indexés par `tool_use_id`.
- Distinguer la prise de décision pilotée par le modèle des arbres de décision préconfigurés, et choisir le bon motif pour une tâche décrite (tâche ouverte = agent ; chemin fixe bien défini = workflow).
- Repérer les anti-patterns dans un exemple de code : parsing de texte pour la fin, plafond d'itération comme arrêt, tour assistant manquant, blocs `text` supplémentaires après `tool_result`, `pause_turn` ignoré.
- Savoir que `max_turns` / `max_budget_usd` dans le Claude Agent SDK sont des garde-fous — le terminateur principal de la boucle reste "pas de blocs `tool_use` dans la réponse assistant".

## Références

- [Handling stop reasons — Claude API docs](https://docs.anthropic.com/en/docs/build-with-claude/handling-stop-reasons) — liste autoritative de chaque valeur `stop_reason` avec exemples de gestion en Python, TypeScript, Go, Java, C#, PHP et Ruby.
- [Messages API reference — Create a Message](https://docs.anthropic.com/en/api/messages) — schéma requête/réponse incluant les types de blocs de contenu `text`, `tool_use` et `tool_result`.
- [Tool use overview](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview) — définition des outils et échange `tool_use` / `tool_result`.
- [Building effective agents — Anthropic Engineering (Dec 19, 2024)](https://www.anthropic.com/engineering/building-effective-agents) — article canonique définissant workflows vs agents et le bloc LLM augmenté. Lecture obligatoire pour l'examen.
- [Effective context engineering for AI agents — Anthropic Engineering (Sep 29, 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — reformule la définition de travail d'un agent comme "LLMs autonomously using tools in a loop" ; couvre compaction et motifs subagent pour les boucles longues.
- [How the agent loop works — Claude Agent SDK docs](https://code.claude.com/docs/en/agent-sdk/agent-loop) — description officielle du cycle de vie SDK turn/message, `max_turns`, `max_budget_usd` et sous-types `ResultMessage`.
- [Agent SDK reference — Python](https://code.claude.com/docs/en/sdk/sdk-python) and [TypeScript](https://code.claude.com/docs/en/sdk/sdk-typescript) — `query()` vs `ClaudeSDKClient`, types de messages et flux d'événements typé.
- [anthropics/claude-cookbooks — tool_use](https://github.com/anthropics/claude-cookbooks/tree/main/tool_use) — exemples exécutables de la boucle, `tool_choice` et appel d'outils programmatique.
- [Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python) and [TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript) — formes actuelles des clients utilisées dans les boucles ci-dessus.
