---
title: "Section 9 — Claude Code dans les pipelines CI/CD"
linkTitle: "9. Intégration CI/CD"
weight: 9
description: "Domaine 3.6 — claude -p et --output-format json, --json-schema, conception de prompts pour des revues actionnables à faible faux positif."
---

## Ce que couvre cette section

Comment exécuter Claude Code sans surveillance dans un système de build : les flags qui évitent les blocages interactifs, les formes de sortie que les outils aval peuvent parser, le contexte projet qui garde le modèle conforme aux règles, et les motifs architecturaux (reviewer indépendant, revue multi-passe, batch-vs-sync) que l'examen teste via les Questions 10, 11 et 12.

## Matériel source (guide officiel)

L'énoncé de tâche 3.6 (lignes 600–629 du guide) couvre : `-p` / `--print` pour le mode non interactif ; `--output-format json` plus `--json-schema` pour des findings parsables par machine ; `CLAUDE.md` comme mécanisme de contexte projet pour Claude Code invoqué par la CI (standards de test, conventions de fixtures, critères de revue) ; et l'isolation du contexte de session — la session qui a *écrit* le code est la mauvaise session pour le *relire*. Compétences : empêcher les blocages interactifs avec `-p` ; produire des findings structurés pour commentaires PR inline ; réinjecter les findings précédents afin de supprimer les doublons ; passer les fichiers de test existants aux runs de génération de tests pour éviter de recouvrir les mêmes scénarios ; documenter standards de test et fixtures dans `CLAUDE.md`.

Sample Question 10 (`-p` est le bon flag headless ; `CLAUDE_HEADLESS`, `--batch` et `< /dev/null` sont des distracteurs), Question 11 (Batches API pour le rapport technique overnight, *pas* le gate pré-merge bloquant), et Question 12 (diviser une PR de 14 fichiers en passes par fichier plus une passe d'intégration) s'appuient toutes sur cette section.

## Claude Code headless / non interactif

### Le flag `-p`

`claude -p "<prompt>"` (alias `--print`) fait exécuter Claude Code comme une requête SDK one-shot : il consomme le prompt, stream ou imprime la réponse sur stdout, et sort avec un code de statut. Pas de vérification TTY, pas de boucle de permission prompt, pas d'attente sur stdin. C'est le seul mécanisme documenté pour la CI. Le piège de la Question 10 est qu'il n'existe pas de variable env `CLAUDE_HEADLESS` ni de flag `--batch`, et rediriger stdin depuis `/dev/null` ne fonctionne pas parce que Claude Code s'attend encore à rendre une UI interactive.

```bash
claude -p "Review the diff for SQL-injection and SSRF risks. Be terse." \
  --model claude-sonnet-4-6 \
  --max-turns 6
```

Ajoutez `--bare` (Claude Code 2.1.x) lorsque vous n'avez pas besoin de hooks, plugins, serveurs MCP ou `CLAUDE.md` auto-découvert ; cela réduit la latence de démarrage et supprime une classe d'échecs du type "pourquoi ma CI a pris la config MCP de mon laptop".

### Formats de sortie (text, json, stream-json)

`--output-format` contrôle la sérialisation de la réponse :

- `text` (défaut) — prose brute ; bien pour les humains, mauvais pour les parsers.
- `json` — un objet JSON avec réponse finale, coût, durée, ID de session, stop reason. À utiliser lorsque l'étape suivante est `jq` ou un script.
- `stream-json` — événements newline-delimited pour les outils qui affichent le progrès ou exposent les événements tool-use. À combiner avec `--include-partial-messages` pour les deltas token par token, `--include-hook-events` pour le cycle de vie des hooks.

`--input-format stream-json` permet à un processus parent d'alimenter Claude Code avec un flux d'événements structuré — utile si votre orchestrateur CI est lui-même un agent.

### `--json-schema` pour findings structurés

`--json-schema '<JSON Schema>'` (print mode seulement) force Claude Code à émettre un payload final *validé* contre le schéma fourni. C'est le flag que l'examen attend lorsque l'étape suivante du pipeline est "poster chaque finding comme commentaire inline de revue GitHub PR".

```bash
claude -p "Review changed files in $PR_DIFF for security issues" \
  --output-format json \
  --json-schema "$(cat .ci/review-schema.json)" \
  --max-turns 8 \
  > findings.json
```

Un schéma qui mappe proprement vers le payload GitHub `pulls/{pr}/comments` ressemble à :

```json
{
  "type": "object",
  "required": ["findings"],
  "properties": {
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["path", "line", "severity", "message"],
        "properties": {
          "path":     { "type": "string" },
          "line":     { "type": "integer", "minimum": 1 },
          "severity": { "enum": ["info", "warning", "error", "critical"] },
          "category": { "enum": ["security", "perf", "bug", "style"] },
          "message":  { "type": "string", "maxLength": 800 }
        }
      }
    }
  }
}
```

### Modes de permission pour runs sans surveillance

Les prompts de permission interactifs sont la deuxième cause la plus fréquente de jobs CI bloqués (après l'oubli de `-p`). Flags clés :

- `--allowedTools "Read,Bash(git diff *),Bash(gh pr view *)"` — allowlist étroite ; défaut recommandé.
- `--disallowedTools "Edit,Write,Bash(rm *)"` — bloquer seulement les écritures quand le job est une revue read-only.
- `--permission-mode acceptEdits` (ou `bypassPermissions`) — accepter les changements sans prompts. `--dangerously-skip-permissions` est la forme explicite "je sais ce que je fais" ; à réserver à des runners sandboxés sans secrets dans l'env.
- `--permission-prompt-tool <mcp-tool>` délègue les décisions de permission à un outil MCP custom lorsque chaque appel doit être journalisé avant approbation.

### Flags de contrôle de session

- `--session-id <uuid>` fixe un UUID afin que les étapes aval puissent reprendre la même conversation plus tard.
- `--resume <id>` / `-r` continue cette session — mécanisme derrière "inclure les findings de revue précédents lors du rerun".
- `--continue` / `-c` choisit la conversation la plus récente dans le répertoire courant.
- `--fork-session` reprend mais branche afin que les retries ne mutent pas le thread canonique.
- `--no-session-persistence` n'écrit rien sur disque ; utile dans les runners stateless où le workspace est détruit entre jobs.

## Motifs de référence

### 1. Revue de sécurité pré-merge (synchrone, bloquante)

Les gates bloquants doivent utiliser un appel temps réel. Comme dans la Question 11, Message Batches API est le mauvais outil ici — sa réduction de coût de 50% se paie par une latence pouvant aller jusqu'à 24 h, inacceptable pour un développeur qui attend un merge.

```yaml
- name: Claude security review
  run: |
    git fetch origin ${{ github.base_ref }}
    DIFF=$(git diff origin/${{ github.base_ref }}...HEAD)
    echo "$DIFF" | claude -p \
      "Find security issues only. Reply with the findings schema." \
      --output-format json \
      --json-schema "$(cat .ci/review-schema.json)" \
      --allowedTools "Read,Bash(git diff *)" \
      --max-turns 6 \
      --max-budget-usd 2.00 \
      > findings.json
```

### 2. Génération de tests en batch nocturne

Les jobs nocturnes de dette de tests sont exactement la charge où Batches API gagne sa réduction de 50%. Exécutez Claude Code avec `-p`, passez les fichiers de test existants dans le contexte afin qu'il ne recouvre pas des scénarios déjà testés, et routez les requêtes de génération via Batches API (voir Section 11).

```bash
ls tests/ | xargs -I{} cat tests/{} > .ci/existing-tests.txt

claude -p \
  "Generate missing tests for src/billing/. Existing tests are attached; do not duplicate scenarios already covered." \
  --append-system-prompt-file .ci/existing-tests.txt \
  --output-format json \
  --max-turns 12
```

### 3. Poster des commentaires PR inline depuis une sortie JSON

Avec un `findings.json` validé par schéma, une simple boucle shell transforme le fichier en commentaires de revue natifs :

```bash
jq -c '.findings[]' findings.json | while read f; do
  gh api -X POST "repos/$REPO/pulls/$PR/comments" \
    -f body="$(echo $f | jq -r .message)" \
    -f path="$(echo $f | jq -r .path)" \
    -F line="$(echo $f | jq -r .line)" \
    -f commit_id="$SHA" -f side=RIGHT
done
```

### 4. Relancer la revue sur de nouveaux commits sans commentaires doublons

Le Domaine 3.6 est explicite : lorsque la PR est repoussée, inclure les findings précédents dans le contexte et demander à Claude de ne rapporter que les problèmes nouveaux ou encore non résolus.

```bash
gh pr view $PR --json comments -q '.comments' \
  | claude -p \
      "Comments already posted are attached. Re-review the new diff and emit ONLY findings that are (a) new or (b) still unaddressed. Use the findings schema." \
      --resume "$REVIEW_SESSION_ID" \
      --output-format json \
      --json-schema "$(cat .ci/review-schema.json)" \
      > new-findings.json
```

## La GitHub Action Claude Code

### Démarrage rapide

```yaml
name: Claude review
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  review:
    runs-on: ubuntu-latest
    permissions: { contents: read, pull-requests: write, issues: write }
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "Review this PR for security issues. Use inline comments."
          claude_args: >-
            --max-turns 10
            --model claude-sonnet-4-6
            --allowedTools "mcp__github_inline_comment__create_inline_comment,Bash(gh pr diff:*),Bash(gh pr view:*),Read"
```

### Inputs / outputs

Inputs clés de `action.yml` : `anthropic_api_key` (ou `claude_code_oauth_token`, `use_bedrock`, `use_vertex`) ; `prompt` ; `claude_args` (args CLI bruts ajoutés au `claude -p` sous-jacent) ; `trigger_phrase` (défaut `@claude`) ; `label_trigger` (défaut `claude`) ; `track_progress` (rend un commentaire de suivi avec cases à cocher) ; `use_sticky_comment` (un commentaire pour tout) ; `classify_inline_comments` (pré-filtre Haiku pour supprimer les commentaires probe/test) ; `branch_prefix` (défaut `claude/`). Le JSON structuré renvoyé par le run est exposé comme outputs GitHub Action.

### Pièges courants

- L'outil MCP inline-comment a besoin de `confirmed: true` pour poster immédiatement ; sinon les commentaires sont bufferisés dans `/tmp/inline-comments-buffer.jsonl` pour classification Haiku.
- L'Action a besoin des permissions de token `pull-requests: write` et `issues: write`, sinon elle échoue silencieusement à poster.
- Passer le choix de modèle et les plafonds de tours via `claude_args`, pas via des inputs séparés.

## CLAUDE.md pour la CI

### Sections qui rapportent

Pour un Claude invoqué par CI, les sections `CLAUDE.md` les plus rentables sont celles que le modèle devrait sinon deviner : commande du runner de test, emplacement des fixtures, bibliothèque de mocking utilisée, ce qui compte comme test "valuable", et critères de revue importants — exactement les standards de test, fixtures et critères de revue appelés par les bullets Skills pour 3.6.

```markdown
## Testing
- Runner: `pnpm test --filter=$PKG`
- Mocks: use MSW only; never mock `fetch` directly
- Fixtures: tests/fixtures/*.json (load with `loadFixture()`)
- Valuable: state transitions, error branches, boundary inputs
- Low-value: getter coverage, framework re-tests

## Review criteria
- Block on: SQLi, SSRF, secrets, auth bypass, N+1 in hot paths
- Comment-only: style, naming, missing comments
```

### Éviter le bloat de contexte

`CLAUDE.md` est chargé à chaque invocation, donc chaque ligne est payée à chaque run CI. Supprimez les décisions historiques, liez vers les ADRs, préférez une ligne déclarative à un paragraphe, et résistez à coller tout le guide de style. En mode `--bare`, `CLAUDE.md` n'est pas auto-chargé du tout ; optez pour `--append-system-prompt-file` lorsque vous le voulez.

## Motif reviewer indépendant

### Pourquoi une session séparée bat l'auto-revue

Le Domaine 3.6 appelle explicitement *l'isolation du contexte de session*. La session qui a généré le code porte son propre historique de raisonnement — les justifications mêmes qui ont produit le bug. Cette session est biaisée vers la confirmation de ses décisions antérieures. Une session fraîche voit seulement le diff, sans narration attachée, et attrape des problèmes que la session auteur aurait ignorés. Le produit Code Review d'Anthropic utilise une flotte d'agents indépendants pour la même raison.

En pratique : ne jamais faire `--continue` depuis la session d'implémentation vers le job de revue. Lancez le reviewer avec un nouveau `--session-id`, une conversation vide, et seulement le diff plus `CLAUDE.md` comme contexte.

### Multi-passe pour grosses PRs (par fichier + intégration cross-file)

C'est la Question 12. Une passe unique sur 14 fichiers montre une *dilution de l'attention* — la profondeur varie fichier par fichier et des motifs identiques reçoivent des verdicts contradictoires. La correction se fait en deux passes :

1. **Passe par fichier.** Une invocation `claude -p` par fichier modifié, limitée aux problèmes locaux (bugs, style, sécurité dans ce fichier).
2. **Passe d'intégration.** Une invocation supplémentaire recevant le résumé de diff et demandée explicitement sur les préoccupations cross-file : changements de frontières API, migrations de schéma, dérive de contrat, mutation d'état partagé.

Un modèle plus grand ou une fenêtre de contexte plus large ne résout pas cela ; le problème est l'allocation de l'attention, pas le budget de tokens.

## Boutons coût et latence

### Prompt caching

Les lectures de cache coûtent environ 10× moins que les écritures de cache, et Claude Code est conçu pour toucher agressivement le cache. Pour maximiser les hits en CI : mettez le contenu stable (prompt système, `CLAUDE.md`, conventions repo) *en premier*, le contenu dynamique (diff, findings précédents) *en dernier* ; ne changez pas le modèle ou la liste d'outils en milieu de session ; sur Claude Code 2.1.108+, définissez `ENABLE_PROMPT_CACHING_1H=1` pour les jobs planifiés lancés plus souvent que toutes les cinq minutes. Utilisez `--exclude-dynamic-system-prompt-sections` avec `-p` pour garder les détails machine hors de la clé de cache quand plusieurs runners partagent la même tâche.

### Batch API pour charges non bloquantes

Lien vers la Section 11 : Message Batches API donne 50% d'économies mais une SLA jusqu'à 24 h. Utilisez-la pour le rapport technique overnight ; ne l'utilisez pas pour le gate pré-merge. C'est la distinction testée par Sample Question 11.

### Choisir la taille de modèle par job

Revue de sécurité pré-merge sur hot path : Sonnet. Passe style-and-lint sur PR docs : Haiku. Revue d'architecture sur le refactor trimestriel : Opus, borné par `--max-budget-usd`. Changez de modèle via `--model` ; ne changez pas en milieu de session ou vous invalidez le cache.

## Points d'attention style examen

- `-p` est le *seul* moyen documenté d'exécuter Claude Code sans surveillance. `CLAUDE_HEADLESS`, `--batch` et `</dev/null` sont des distracteurs.
- `--output-format json --json-schema <schema>` est la recette canonique pour des findings parsables par pipeline.
- Les charges bloquantes (pré-merge) utilisent des appels temps réel ; les charges overnight non bloquantes utilisent Batches API.
- Grosses PRs : passes par fichier plus une passe d'intégration — pas un méga-prompt unique ni une plus grande fenêtre de contexte.
- La session qui a écrit le code est la mauvaise session pour le relire ; lancez un reviewer indépendant.
- Une relance de revue doit inclure les findings précédents dans le contexte pour supprimer les doublons.
- `CLAUDE.md` porte standards de test, fixtures et critères de revue — chaque ligne est payée à chaque run CI, donc gardez-le léger.

## Références

- Claude Code CLI reference: <https://docs.claude.com/en/docs/claude-code/cli-usage>
- Run Claude Code programmatically (headless): <https://code.claude.com/docs/en/headless>
- Agent SDK (Python / TypeScript): <https://code.claude.com/docs/en/agent-sdk/python>, <https://code.claude.com/docs/en/agent-sdk/typescript>
- Structured outputs: <https://code.claude.com/en/agent-sdk/structured-outputs>
- Claude Code GitHub Action: <https://github.com/anthropics/claude-code-action> (usage: `docs/usage.md`, recipes: `docs/solutions.md`)
- GitHub Actions docs: <https://docs.anthropic.com/en/docs/claude-code/github-actions>
- Code Review product: <https://code.claude.com/docs/en/code-review> and <https://claude.com/blog/code-review>
- Subagents pattern: <https://claude.com/blog/subagents-in-claude-code>
- Prompt caching: <https://claude.com/blog/lessons-from-building-claude-code-prompt-caching-is-everything> and <https://docs.claude.com/en/docs/build-with-claude/prompt-caching>
- Memory / CLAUDE.md: <https://code.claude.com/en/memory>
