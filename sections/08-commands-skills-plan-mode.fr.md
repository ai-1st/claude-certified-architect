---
title: "Section 8 — Slash commands personnalisées, Skills, Plan Mode et raffinement itératif"
linkTitle: "8. Commandes, Skills et Plan Mode"
weight: 8
description: "Domaines 3.2, 3.4, 3.5 — .claude/commands/, .claude/skills/ avec context: fork et allowed-tools, et plan mode vs exécution directe."
---

## Ce que couvre cette section

Étendre Claude Code avec des **slash commands personnalisées** et des **agent skills**, choisir le **plan mode** vs l'exécution directe (et comprendre où le subagent **Explore** s'insère), et appliquer des techniques de **raffinement itératif**. Sample Question 4 teste où vit une commande `/review` partagée ; Sample Question 5 teste le choix du plan mode pour restructurer un monolithe en microservices.

## Matériel source (guide officiel)

**3.2 Slash commands et skills.** Commandes projet dans `.claude/commands/` (partagées via VCS) vs commandes utilisateur dans `~/.claude/commands/` (personnelles). Skills dans `.claude/skills/<name>/SKILL.md` avec frontmatter YAML supportant `context: fork`, `allowed-tools`, `argument-hint`. `context: fork` exécute la skill dans un contexte de subagent isolé afin que la sortie verbeuse ne pollue pas la conversation principale. Personnalisation personnelle : des variantes dans `~/.claude/skills/` avec un nom différent évitent d'affecter les coéquipiers. Les skills sont à la demande et spécifiques à une tâche ; `CLAUDE.md` est un standard universel toujours chargé.

**3.4 Plan mode vs exécution directe.** Le plan mode sert aux tâches complexes avec grands changements, plusieurs approches valides, décisions architecturales, modifications multi-fichiers. L'exécution directe convient aux changements simples et bien cadrés (une validation, une correction d'un fichier avec stack trace claire). Le plan mode permet une exploration sûre avant engagement. Le subagent Explore isole la découverte verbeuse et renvoie des résumés. Motif courant : plan pour investiguer, puis exécution directe pour appliquer.

**3.5 Raffinement itératif.** Les exemples entrée/sortie concrets battent la prose. Itération pilotée par tests : tests d'abord, puis itérer sur les échecs. Motif interview : demander à Claude de poser des questions clarifiantes dans les domaines inconnus. Cas de test spécifiques pour les edge cases (p. ex. nulls dans les migrations). Grouper les problèmes qui interagissent dans un message ; séquencer les problèmes indépendants.

## Les quatre primitives de personnalisation

Claude Code expose quatre mécanismes qui se branchent à différents endroits de la boucle agentique. Savoir lequel choisir est une cible presque certaine de l'examen.

| Primitive | Déclencheur | Isolation | Quand l'utiliser |
|---|---|---|---|
| **Slash command** (`.claude/commands/foo.md`) | L'utilisateur tape `/foo` | Contexte principal | Workflow interactif réutilisable — `/review`, `/commit`, `/deploy-staging`. |
| **Skill** (`.claude/skills/foo/SKILL.md`) | L'utilisateur tape `/foo` *ou* Claude l'auto-invoque quand la description correspond | Contexte principal ; isolée avec `context: fork` | Connaissance ou procédure spécifique à une tâche avec fichiers de support ; laisse Claude décider *quand* l'appliquer. |
| **Subagent** (`.claude/agents/foo.md`) | Claude ou l'utilisateur délègue une tâche | Toujours isolé ; ses propres outils et modèle | Recherche verbeuse, travail parallèle, spécialistes comme `code-reviewer` ou `Explore`. |
| **Hook** (`hooks.json`) | Un événement de cycle de vie se déclenche automatiquement (PreToolUse, PostToolUse, UserPromptSubmit, etc.) | Script shell, sortie réinjectée | Effets de bord déterministes — auto-lint, bloquer les éditions de chemins protégés, ajouter des trailers de commit. |

Anthropic a **fusionné les slash commands personnalisées dans les skills** fin 2025 : un fichier `.claude/commands/deploy.md` et une skill `.claude/skills/deploy/SKILL.md` créent tous deux `/deploy` et se comportent de façon identique. Les skills sont la forme recommandée parce qu'elles ajoutent fichiers de support, contrôle d'invocation, injection de contexte dynamique et exécution subagent ([slash-commands docs](https://docs.anthropic.com/en/docs/claude-code/slash-commands)).

## Slash commands personnalisées

### Portée projet vs utilisateur

- **`.claude/commands/`** — portée projet, versionnée, workflows d'équipe (`/review`, `/security-review`, `/migrate-route`).
- **`~/.claude/commands/`** — portée utilisateur, raccourcis personnels uniquement (`/scratch`, `/jira-link`).

Sample Question 4 repose là-dessus : un `/review` qui "should be available to every developer when they clone or pull the repository" va dans **`.claude/commands/`** — pas `~/.claude/commands/`, pas `CLAUDE.md`, pas un `.claude/config.json` fictif.

### Layout de fichier et frontmatter

Une commande est un fichier markdown avec frontmatter YAML optionnel :

```markdown
---
description: Run the team code-review checklist on the current diff
argument-hint: [path-or-PR]
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git log:*)
model: sonnet
---

Review the changes in $ARGUMENTS using our checklist:

## Diff
!`git diff HEAD`

## Style guide
@docs/CODE_REVIEW.md

For each finding, output: severity, file:line, problem, suggested fix.
```

Champs frontmatter clés ([référence](https://docs.anthropic.com/en/docs/claude-code/slash-commands#frontmatter-reference)) :

| Champ | Objectif |
|---|---|
| `description` | Résumé affiché dans `/help` ; pilote aussi l'auto-invocation. Garder sous ~60 caractères. |
| `argument-hint` | Indication autocomplete comme `[issue-number]` ou `[filename] [format]`. |
| `allowed-tools` | Outils utilisables sans approbation par appel. Accepte des motifs Bash globbés comme `Bash(git:*)`. |
| `model` | Surcharger le modèle pour cette commande (`haiku`/`sonnet`/`opus`/`inherit`). |
| `disable-model-invocation` | `true` force le déclenchement humain uniquement — pour des workflows avec effets de bord comme `/deploy`. |

Tous les champs sont optionnels.

### Arguments, exécution bash, références de fichiers

Trois mécanismes de substitution vivent dans le corps markdown :

- **`$ARGUMENTS`** — chaîne complète d'arguments. `$ARGUMENTS[N]` indexe par position ; `$N` est le raccourci.
- **`` !`cmd` ``** — exécute une commande bash au chargement et inline son stdout (p. ex. `` !`git diff HEAD` ``).
- **`@path`** — inline le contenu d'un fichier ; `@$0` inline un fichier dont le chemin est le premier argument.

> Note examen : les anciennes docs utilisaient `$1` pour le *premier* argument. Les docs actuelles passent à l'indexation 0-based (`$0`, `$1`, `$2`). Reconnaissez les deux ; préférez `$ARGUMENTS` quand l'ordre n'importe pas.

### Exemple : `/review`

Un `/review` à portée projet pour Sample Question 4 — committer ceci dans `.claude/commands/review.md` et chaque coéquipier obtient `/review` au prochain pull :

```markdown
---
description: Run our PR review checklist on a diff or PR
argument-hint: [pr-number-or-path]
allowed-tools: Read, Grep, Glob, Bash(gh pr view:*), Bash(git diff:*)
---

## Diff
!`git diff origin/main...HEAD`

## Checklist
@.claude/docs/review-checklist.md

Apply each checklist item. For every finding, report severity
(block/major/nit), file:line, the issue, and a concrete fix.
End with a one-paragraph overall verdict.
```

## Agent skills

### Layout de fichier

```
.claude/skills/pr-summary/
  SKILL.md          # required: frontmatter + instructions
  checklist.md      # optional supporting files
  example-good.md
```

Les skills sont découvertes à quatre niveaux — enterprise, user (`~/.claude/skills/`), project (`.claude/skills/`) et plugin. Enterprise surclasse personal, personal surclasse project ; les skills de plugin vivent dans un namespace `plugin-name:skill-name` ([skills docs](https://docs.anthropic.com/en/docs/claude-code/skills)).

### Référence frontmatter

Au-delà des champs communs aux commandes, les skills ajoutent :

| Champ | Objectif |
|---|---|
| `name` | Identifiant de skill (par défaut, nom du répertoire). |
| `description` | Ce que fait la skill et quand l'utiliser ; pilote l'auto-invocation. `description` + `when_to_use` plafonnés à 1 536 caractères combinés. |
| `when_to_use` | Phrases déclencheuses supplémentaires ajoutées à `description`. |
| `arguments` | Args positionnels nommés, p. ex. `arguments: [issue, branch]` → `$issue`, `$branch`. |
| `user-invocable` | `false` masque du menu `/` ; Claude peut toujours l'utiliser comme connaissance de fond. |
| `disable-model-invocation` | `true` bloque l'auto-invocation — à combiner avec les skills à effets de bord (`/deploy`). |
| `context` | `fork` pour exécuter dans un subagent isolé. |
| `agent` | Type de subagent pour skills forkées (`Explore`, `Plan`, custom). |
| `hooks` | Hooks de cycle de vie limités à cette skill. |
| `paths` | Motifs glob qui restreignent l'auto-invocation aux fichiers correspondants. |

### `context: fork` — ce que cela fait et quand l'utiliser

Par défaut, une skill s'exécute inline, donc sa sortie — listes de fichiers, pensées intermédiaires, résultats de recherche — consomme la fenêtre de contexte parent. `context: fork` **envoie la skill à un subagent frais** : le contenu de la skill devient le prompt du subagent, le subagent s'exécute avec ses propres outils et permissions, et seul son résumé final revient au parent.

```yaml
---
name: pr-summary
description: Summarize a PR's changes and risk surface
context: fork
agent: Explore
allowed-tools: Bash(gh *), Read, Grep, Glob
---

Summarize PR $ARGUMENTS. Read the diff with `gh pr diff $ARGUMENTS`,
group changes by subsystem, call out risky touches (auth, billing,
migrations), and end with a 3-bullet reviewer checklist.
```

Forkez lorsque la skill est **verbeuse** (analyse de codebase, résumé de gros diff), **exploratoire** (brainstorming d'alternatives) ou **indépendante** (renvoie un résumé, pas des artefacts bruts que le parent édite). Ne forkez pas les skills pures "use these conventions" — le subagent reçoit des guidelines mais aucune tâche actionnable et renvoie vide.

### Motif de variante personnelle

Pour personnaliser une skill partagée sans affecter les coéquipiers, copiez-la sous un nom différent dans `~/.claude/skills/` (p. ex. `pr-summary-detailed/`). Cela évite l'antipattern "j'ai édité la skill partagée et maintenant tout le monde a mon style". La même astuce de renommage vaut pour les slash commands dans `~/.claude/commands/`.

### Matrice de décision Skills vs CLAUDE.md

| Besoin | Choisir |
|---|---|
| Standards universels actifs à chaque session (stack technique, style) | `CLAUDE.md` |
| Procédure spécifique à une tâche, à la demande, avec fichiers de support | Skill |
| Workflow interactif sans auto-invocation | Skill avec `disable-model-invocation: true` |
| Convention par type de fichier gated par globs de chemins | Règle path-scoped (`.claude/rules/`) ou skill avec `paths:` |
| Convention qui doit s'exécuter comme code, pas conseil | Hook |
| Découverte verbeuse qui inonderait le contexte | Skill avec `context: fork`, ou déléguer à Explore |

`CLAUDE.md` est un **contexte always-on** — bon marché à charger, coûteux à grande échelle. Les skills sont du **contexte à la demande** — plus lourdes par usage mais gratuites quand inutilisées.

## Plan mode vs exécution directe

### Comment entrer en plan mode

Le plan mode est l'un des six modes de permission ; Claude lit les fichiers et lance des commandes read-only mais ne peut pas éditer le source ([permission-modes docs](https://docs.anthropic.com/en/docs/claude-code/permission-modes)). Quatre points d'entrée :

1. **Cycle en session** avec `Shift+Tab` : `default → acceptEdits → plan`.
2. **Au démarrage** : `claude --permission-mode plan` (fonctionne aussi avec `-p` en headless).
3. **Défaut projet** : `permissions.defaultMode: "plan"` dans `.claude/settings.json`.
4. **Par prompt** : préfixer un message avec `/plan`.

Quand le plan est prêt, Claude propose (a) approuver et passer en auto, (b) approuver et revoir chaque édition, ou (c) continuer à planifier. `Ctrl+G` ouvre le plan dans votre éditeur ; `Shift+Tab` sort sans approuver.

### Quand le plan mode paie

Utilisez le plan mode lorsque :

- Le changement traverse **beaucoup de fichiers ou de frontières architecturales** (restructuration microservices, migration de bibliothèque touchant 45+ fichiers).
- Il existe **plusieurs approches valides** avec de vrais compromis (Redis vs in-memory vs cache fichier ; webhook vs polling).
- **Le périmètre est inconnu** — la question est "quelle taille ?" et non "implémente ceci".
- **Rayon d'impact élevé** : migrations de schéma, refactors auth, contrats cross-team.

Sample Question 5 est le premier cas : restructurer un monolithe en microservices traverse des dizaines de fichiers plus des décisions de frontières — plan mode textbook. La mauvaise réponse ("laisser l'implémentation révéler les frontières") perd en rework dès qu'une dépendance cachée apparaît.

### Quand l'exécution directe est correcte

- Bug d'un seul fichier avec stack trace claire et correction évidente.
- Une validation, une ligne de log, un feature flag.
- Refactor mécanique avec cible connue (renommage sur N fichiers — utiliser `acceptEdits`).
- Vous avez déjà un plan d'un tour précédent.

Le plan mode a un coût — tokens d'exploration supplémentaires pour Claude, revue de plan pour le développeur. Ne le payez pas sur des changements de trois lignes.

### Le subagent Explore pour la découverte verbeuse

Même hors plan mode, déléguez la phase de découverte au subagent intégré **Explore** — read-only, basé sur Haiku, accès à Glob/Grep/Read/Bash mais pas Write/Edit. Chaque invocation s'exécute dans sa propre fenêtre de contexte et ne renvoie qu'un résumé ([sub-agents docs](https://docs.anthropic.com/en/docs/claude-code/sub-agents)).

Utilisez Explore (directement, ou via une skill avec `context: fork` et `agent: Explore`) pour les questions "où est défini X ?" / "comment fonctionne Y ?", les tâches multi-phases qui exploseraient le budget de contexte en découverte, et les recherches larges dont vous ne relirez pas la sortie brute. Spécifiez la profondeur : `quick`, `medium` ou `very thorough`.

### Motif combiné : plan puis exécuter

Le workflow recommandé pour le travail non trivial est **explore → plan → execute → commit** ([best practices](https://docs.anthropic.com/en/docs/claude-code/best-practices)) : déléguer le survey à Explore, faire écrire le plan par Claude, le relire/éditer avec `Ctrl+G`, approuver vers `acceptEdits`, puis committer. Plan mode pour l'investigation, exécution directe pour les éditions.

## Playbook de raffinement itératif

### 1. Fournir des exemples entrée/sortie

Quand l'ambiguïté prose est le goulot (extraction, transformation, formatage), fournissez **2–3 paires entrée/sortie concrètes** plutôt que plus d'adjectifs. Claude généralise plus fiablement depuis des exemples que depuis de la prose comme "friendly but professional." Les exemples servent aussi de cas de régression.

### 2. Itération pilotée par tests

Les tests sont un signal de vérification non ambigu. La boucle : faire écrire à Claude une suite couvrant happy path, edge cases et budgets de performance *avant* toute implémentation ; l'exécuter et partager les échecs ; faire implémenter à Claude le minimum pour faire passer un test rouge au vert ; répéter jusqu'au vert ; refactorer avec la suite comme filet de sécurité. C'est le principe "donner à Claude un moyen de vérifier son travail" rendu concret.

### 3. Motif interview

Dans les domaines inconnus, demandez à Claude de vous interviewer avant d'écrire le code : *"Before you implement caching, list the questions you'd need answered (eviction, invalidation, failure modes, memory budget, multi-tenant isolation) and ask each one."* Cela fait émerger des considérations que le développeur n'avait pas anticipées. Utilisez-le pour les préoccupations transversales : caching, retries, auth, billing, migrations.

### 4. Grouper les corrections qui interagissent vs séquencer les indépendantes

Quand plusieurs problèmes apparaissent, décidez s'ils interagissent :

- **Interagissent** — une correction quelque part change la bonne réponse ailleurs (modifier un format de date affecte parsing, affichage, écritures DB). Mettez tout dans un **seul message détaillé** afin que Claude raisonne globalement.
- **Indépendants** — faute de frappe dans un helper, null check manquant ailleurs. Corriger **séquentiellement**, un par tour — petits diffs, contexte frais, échecs attribuables.

Pour un seul edge case en échec (null dans une migration), fournissez l'entrée/sortie attendue spécifique à ce cas — ne réécrivez pas tout le script.

## Points d'attention style examen

- Les commandes partagées par l'équipe vivent dans **`.claude/commands/<name>.md`** dans le repo. Pas `~/.claude/commands/`, pas `CLAUDE.md`, pas un `config.json` fictif.
- Les skills sont **à la demande**, `CLAUDE.md` est **toujours chargé**. Les standards persistants vont dans `CLAUDE.md` ; les workflows spécifiques à une tâche vont dans des skills.
- `context: fork` isole la sortie verbeuse ou exploratoire d'une skill dans un subagent ; combinez avec `agent: Explore` pour de la découverte read-only.
- `allowed-tools` préapprouve une liste d'outils pendant que la skill est active ; cela ne bloque pas les autres outils — les règles de permission s'appliquent encore.
- `argument-hint` alimente l'autocomplete ; `$ARGUMENTS`/`$N` substituent les valeurs réelles dans le corps du prompt.
- On entre en plan mode via cycle `Shift+Tab`, `--permission-mode plan`, préfixe `/plan` ou `defaultMode: "plan"`. Il est read-only.
- Choisir plan mode pour le travail architectural / multi-fichiers / multi-approches ; choisir exécution directe pour les changements single-file bien cadrés.
- Le subagent Explore est intégré, basé Haiku, read-only et renvoie des résumés — utilisez-le pour garder la découverte hors du contexte principal.
- Itération : les exemples entrée/sortie battent la prose ; les tests sont la boucle de feedback la plus efficace ; interviewer dans les domaines inconnus ; grouper les corrections qui interagissent, séquencer les indépendantes.

## Références

- Slash commands — <https://docs.anthropic.com/en/docs/claude-code/slash-commands>
- Skills — <https://docs.anthropic.com/en/docs/claude-code/skills>
- Permission modes (plan mode) — <https://docs.anthropic.com/en/docs/claude-code/permission-modes>
- Subagents (Explore) — <https://docs.anthropic.com/en/docs/claude-code/sub-agents>
- Hooks — <https://docs.anthropic.com/en/docs/claude-code/hooks>
- Best practices — <https://docs.anthropic.com/en/docs/claude-code/best-practices>
- Extend Claude Code — <https://docs.anthropic.com/en/docs/claude-code/features-overview>
- Open Agent Skills standard — <https://agentskills.io/>
- Claude Code repo — <https://github.com/anthropics/claude-code>
- Official plugins — <https://github.com/anthropics/claude-plugins-official>
