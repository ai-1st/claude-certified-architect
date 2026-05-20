---
title: "Section 7 — Hiérarchie de configuration CLAUDE.md et règles spécifiques aux chemins"
linkTitle: "7. CLAUDE.md et règles"
weight: 7
description: "Domaines 3.1, 3.3 — hiérarchie CLAUDE.md utilisateur/projet/répertoire, motifs @import et conventions .claude/rules/ appariées par glob."
---

## Ce que couvre cette section

Comment Claude Code trouve, fusionne et applique des instructions persistantes entre les portées **managed-policy / user / project / subdirectory**, comment garder de grands jeux d'instructions modulaires avec `@import` et `.claude/rules/`, et comment utiliser des globs YAML `paths` pour que les conventions ne se chargent que lorsque Claude touche les fichiers correspondants. L'examen teste deux modes d'échec précis : mettre des standards d'équipe au niveau utilisateur (donc un nouveau coéquipier ne les voit jamais) et utiliser un `CLAUDE.md` de sous-répertoire pour des conventions transversales (fichiers de test répartis dans l'arbre) au lieu d'une règle path-scoped. Sample Question 6 teste directement le second motif.

## Matériel source (guide officiel)

**3.1 — Hiérarchie et organisation modulaire.** Connaître la hiérarchie `CLAUDE.md` user / project / subdirectory ; savoir que `~/.claude/CLAUDE.md` est par utilisateur (jamais partagé via VCS) ; connaître `@import` pour le contenu modulaire ; connaître `.claude/rules/` comme alternative à un fichier monolithique ; diagnostiquer les problèmes de hiérarchie avec `/memory`.

**3.3 — Règles spécifiques aux chemins.** Savoir que `.claude/rules/*.md` utilise du frontmatter YAML `paths` avec des globs, se charge seulement quand Claude touche des fichiers correspondants, et bat un `CLAUDE.md` de sous-répertoire pour les conventions transversales dans l'arbre (p. ex. `**/*.test.tsx`).

## La hiérarchie CLAUDE.md

### Emplacements et précédence

Claude Code charge plusieurs fichiers `CLAUDE.md` dans un ordre défini. Le modèle mental crucial est que **les fichiers sont concaténés, pas remplacés** — chaque fichier applicable finit dans le contexte de session. L'ordre compte parce que les instructions plus proches de votre répertoire de travail apparaissent *plus tard* dans le contexte, ce qui leur donne une priorité effective.

| Portée | Chemin | Cas d'usage | Partagé via VCS ? |
| --- | --- | --- | --- |
| Managed policy | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`, Linux/WSL `/etc/claude-code/CLAUDE.md`, Windows `C:\Program Files\ClaudeCode\CLAUDE.md` | Règles org-wide déployées par IT ; impossibles à exclure | Non — déployé via MDM / Group Policy / Ansible |
| User | `~/.claude/CLAUDE.md` | Préférences personnelles sur tous les projets | Non — jamais committé |
| Project | `./CLAUDE.md` *ou* `./.claude/CLAUDE.md` | Conventions d'équipe, commandes de build, architecture | Oui — committé |
| Local project | `./CLAUDE.local.md` | URLs sandbox personnelles, données de test | Non — à mettre dans `.gitignore` |
| Subdirectory | `path/to/dir/CLAUDE.md` | Instructions pour une partie de la codebase | Oui si committé |
| User rules | `~/.claude/rules/*.md` | Règles personnelles sur tous les projets | Non |
| Project rules | `.claude/rules/*.md` | Règles d'équipe par sujet ou par chemin | Oui |

Sources : documentation officielle Claude Code memory sur [docs.anthropic.com/en/docs/claude-code/claude-md](https://docs.anthropic.com/en/docs/claude-code/claude-md) et miroir [code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory).

### Comment les fichiers sont fusionnés au démarrage de session

Claude remonte l'arborescence depuis votre répertoire de travail courant, récupérant chaque `CLAUDE.md` et `CLAUDE.local.md` trouvé. Le contenu est ordonné **racine du filesystem → répertoire de travail** ; dans un même répertoire, `CLAUDE.local.md` est ajouté après `CLAUDE.md`. Lancer dans `foo/bar/` donne approximativement :

```
1. managed-policy CLAUDE.md (if present)
2. ~/.claude/CLAUDE.md + ~/.claude/rules/*
3. /foo/CLAUDE.md
4. /foo/bar/CLAUDE.md
5. /foo/bar/CLAUDE.local.md
6. .claude/rules/*.md  (rules without `paths` load here)
```

Les fichiers `CLAUDE.md` de sous-répertoires **sous** votre répertoire de travail ne sont **pas** chargés au lancement — ils se chargent paresseusement lorsque Claude lit un fichier dans ce sous-répertoire. Après `/compact`, le `CLAUDE.md` racine projet est réinjecté depuis le disque, mais les `CLAUDE.md` imbriqués ne se rechargent que lorsque Claude retouche leur sous-arbre.

### Diagnostiquer avec `/memory`

`/memory` est l'outil de debugging le plus important pour ce domaine. Il liste chaque `CLAUDE.md`, `CLAUDE.local.md` et fichier de règles chargé dans la session courante, permet de basculer auto memory et ouvre tout fichier dans votre éditeur. Si un fichier attendu n'est pas listé, Claude ne peut pas le voir.

Flux standard quand "Claude ne suit pas mon CLAUDE.md" :

1. Lancer `/memory`. Le fichier est-il listé ?
2. Sinon, est-il dans un emplacement chargé ? `~/.claude/CLAUDE.md` s'applique seulement à *vos* sessions ; un coéquipier ne le verra pas.
3. S'il est listé, les instructions sont-elles spécifiques et non conflictuelles ? Deux règles contradictoires laissent Claude choisir arbitrairement.
4. Pour les règles path-scoped, utilisez le hook `InstructionsLoaded` pour logger quand chaque fichier se charge et pourquoi.

Piège canonique de hiérarchie : un développeur met les standards d'équipe dans `~/.claude/CLAUDE.md`, cela fonctionne très bien pour lui, puis un nouvel arrivant demande "pourquoi Claude utilise-t-il le mauvais framework de test ?" Correction : déplacer ces règles dans `./CLAUDE.md` ou `./.claude/CLAUDE.md` et les committer.

## Organisation modulaire avec `@import`

### Syntaxe et limites

Dans tout `CLAUDE.md`, le token `@path/to/file` développe le fichier référencé inline au démarrage de session.

- Les chemins relatifs se résolvent **depuis le fichier contenant l'import**, pas depuis le répertoire de travail. Les chemins absolus et `~/...` fonctionnent aussi.
- Les imports s'imbriquent jusqu'à **5 sauts** : `CLAUDE.md` → `@docs/guide.md` → `@docs/sub/detail.md` → … (cinq niveaux totaux max).
- La première fois que Claude Code voit des imports externes dans un projet, une boîte d'approbation liste les fichiers. Refuser une fois les garde désactivés.
- Importer ne **sauve pas** de tokens — chaque fichier importé est chargé en entier au lancement. Les imports servent à *organiser*, pas à réduire le contexte. Utilisez `.claude/rules/` avec `paths` pour du chargement paresseux.

### Exemple : imports par package dans un monorepo

```markdown
# Monorepo CLAUDE.md (root)
This is a pnpm workspace. Run `pnpm install` at the root.

## Package-specific standards
- API service: @services/api/STANDARDS.md
- Web frontend: @apps/web/STANDARDS.md
- Shared types: @packages/types/STANDARDS.md

See @README.md for project overview and @package.json for scripts.

# Individual Preferences (cross-worktree personal notes)
- @~/.claude/my-project-instructions.md
```

Chaque mainteneur de package possède le fichier de standards lié sans forcer toutes les autres équipes à le lire. L'import `~/.claude/...` est le contournement canonique du fait qu'un `CLAUDE.local.md` gitignoré n'existe que dans le worktree où vous l'avez créé.

## Fichiers de sujet dans `.claude/rules/`

`.claude/rules/` est une **fonction documentée de Claude Code**, pas une convention propre à Cursor. La documentation officielle memory la décrit sous "Organize rules with `.claude/rules/`". Tous les fichiers `.md` de ce répertoire sont découverts récursivement, en deux variantes :

1. **Règles inconditionnelles** — pas de frontmatter `paths`. Chargées au lancement avec la même priorité que `.claude/CLAUDE.md`.
2. **Règles path-scoped** — frontmatter `paths` avec un ou plusieurs globs. Chargées paresseusement lorsque Claude lit un fichier correspondant à l'un des motifs.

Les règles niveau utilisateur dans `~/.claude/rules/` s'appliquent à chaque projet de votre machine et se chargent *avant* les règles projet, donc les règles projet gagnent en cas de conflit.

### Schéma frontmatter (YAML)

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "src/api/**/*.tsx"
---

# API development rules
- All API endpoints must include input validation.
- Use the standard error response format.
```

Notes de la spec et d'un bug YAML connu ([anthropics/claude-code#13905](https://github.com/anthropics/claude-code/issues/13905)) : `paths` doit être une liste YAML ; les motifs commençant par `*` ou `{` **doivent être entre guillemets** pour être du YAML valide ; omettre `paths` charge la règle inconditionnellement.

### Sémantique des globs `paths`

Matchers glob standards — `*` matche dans un seul segment de chemin, `**` recurse dans les répertoires :

| Motif | Matche |
| --- | --- |
| `*.md` | Fichiers Markdown à la racine du projet seulement |
| `**/*.ts` | Fichiers TypeScript dans tout répertoire |
| `**/*.test.tsx` | Fichiers de test React partout dans l'arbre |
| `src/**/*` | Chaque fichier sous `src/` |
| `src/components/*.tsx` | Enfants directs seulement |
| `src/**/*.{ts,tsx}` | TypeScript et TSX sous `src/` (brace expansion) |
| `terraform/**/*` | Tout sous `terraform/` |

L'exemple canonique de l'examen `**/*.test.tsx` fonctionne — c'est le cas textbook pour une règle path-scoped.

### Exemple : règle transversale de tests (sample Question 6)

`.claude/rules/tests.md` couvre les fichiers de test vivant à côté des fichiers source (`Button.test.tsx` à côté de `Button.tsx`) partout dans le dépôt :

```markdown
---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "**/*.spec.ts"
---

# Test conventions
- Use `describe` / `it` (Jest style), never `test()`.
- One assertion per `it` block; use `it.each` for tables.
- Mock with `vi.mock`; never mutate `globalThis`.
```

Remplacez les `paths` par `["terraform/**/*", "**/*.tf"]` et vous avez la même astuce pour les fichiers d'infrastructure partout dans l'arbre.

## CLAUDE.md vs CLAUDE.md de sous-répertoire vs path-rules — quand utiliser quoi

| Besoin | Meilleur choix |
| --- | --- |
| Standard d'équipe universel (build, etiquette repo) | `CLAUDE.md` racine ou `.claude/CLAUDE.md` |
| Préférences personnelles entre projets | `~/.claude/CLAUDE.md` ou `~/.claude/rules/` |
| Préférences personnelles dans un repo | `CLAUDE.local.md` (gitignoré) |
| Convention limitée à un dossier | `path/to/CLAUDE.md` de sous-répertoire (lazy) |
| Convention répartie dans de nombreux dossiers (tests, `.tf`, `.sql`) | `.claude/rules/*.md` avec glob `paths` |
| Grandes instructions monolithiques | Découper en `.claude/rules/` ou `@import`s |
| Doit s'exécuter à un point précis du cycle de vie | Un **hook**, pas `CLAUDE.md` |
| Politique org-wide que les utilisateurs ne peuvent pas désactiver | `CLAUDE.md` managed-policy + `settings.json` managé |

Heuristique décisive : **la convention suit-elle le *type/motif* de fichier ou la *localisation* ?** Motif → path-rule avec glob. Localisation → `CLAUDE.md` de sous-répertoire. C'est exactement ce que teste la Question 6.

## Layout de référence

```
your-monorepo/
├── CLAUDE.md                          # High-level architecture, build commands
├── CLAUDE.local.md                    # Gitignored personal notes
├── .claude/
│   └── rules/
│       ├── tests.md                   # paths: ["**/*.test.{ts,tsx}"]
│       ├── terraform.md               # paths: ["terraform/**/*", "**/*.tf"]
│       ├── api-conventions.md         # paths: ["src/api/**/*.ts"]
│       └── security.md                # No paths -> loads every session
├── services/billing/
│   ├── CLAUDE.md                      # Lazy-loaded inside services/billing/
│   └── STANDARDS.md                   # @services/billing/STANDARDS.md from root
└── apps/web/CLAUDE.md                 # Lazy-loaded for the web app subtree

~/.claude/
├── CLAUDE.md                          # Personal preferences across projects
└── rules/preferences.md               # Personal coding preferences
```

Chaque mécanisme est dans sa zone idéale : petit fichier racine, règles modulaires par sujet, scoping par chemin pour les conventions transversales, `CLAUDE.md` de sous-répertoire pour les règles réellement liées à un emplacement, et `~/.claude/` pour les préférences personnelles.

## Anti-patterns et pièges d'examen

- **Standards d'équipe dans `~/.claude/CLAUDE.md`.** Ils vous suivent vous, pas le repo ; les nouveaux coéquipiers ne reçoivent rien. Déplacer les conventions partagées vers `./CLAUDE.md` ou `./.claude/CLAUDE.md` et les committer.
- **Un `CLAUDE.md` géant de 2 000 lignes.** Tout `CLAUDE.md` entre en entier dans la fenêtre de contexte au lancement ; Anthropic recommande moins de ~200 lignes par fichier. Découper en fichiers de sujet `.claude/rules/` et préférer les règles path-scoped afin que la plupart du contenu reste hors contexte jusqu'au besoin.
- **`CLAUDE.md` de sous-répertoire pour une convention transversale.** Des fichiers de test à côté des sources dans des dizaines de dossiers ne peuvent pas être couverts par des fichiers liés aux répertoires sans duplication absurde. Utilisez un seul `.claude/rules/tests.md` avec `paths: ["**/*.test.tsx"]`. C'est Sample Question 6.
- **Traiter `@import` comme optimisation de tokens.** Ce n'en est pas une — les imports se chargent en entier au lancement. Le gain de tokens vient de `.claude/rules/` avec `paths` (chargement lazy).
- **Règles machine-enforceable dans `CLAUDE.md`.** `CLAUDE.md` est du contexte consultatif, pas de l'enforcement. Tout ce qui *doit* arriver (lint avant commit, bloquer les écritures dans `secrets/`) appartient à un hook ou `permissions.deny`.
- **Supposer que Claude lit `AGENTS.md`.** Il ne le fait pas nativement. Si votre repo utilise `AGENTS.md`, mettez `@AGENTS.md` en haut de `CLAUDE.md` (recommandé ; portable) ou `ln -s AGENTS.md CLAUDE.md` (Unix ; Windows nécessite Admin/Developer Mode). `/init` dans un repo avec `AGENTS.md` le lit et incorpore les parties pertinentes.
- **Règles conflictuelles entre fichiers.** Claude peut choisir arbitrairement. Auditez périodiquement et utilisez `claudeMdExcludes` dans `.claude/settings.local.json` pour ignorer les fichiers d'équipes non pertinentes dans un monorepo.

## Points d'attention style examen

- Connaître les quatre portées plus managed policy et lesquelles sont partagées via VCS. `~/.claude/CLAUDE.md` est le piège — par utilisateur, jamais committé.
- Les fichiers se **concatènent** depuis la racine du filesystem vers le répertoire de travail ; `CLAUDE.local.md` s'ajoute après `CLAUDE.md` dans le même répertoire.
- `@import` accepte chemins relatifs, absolus et préfixés `~` ; s'imbrique sur **5 sauts** max ; ne sauve **pas** de tokens.
- `.claude/rules/*.md` avec frontmatter `paths` est le **seul** mécanisme fournissant un chargement de conventions lazy basé sur glob. Sans `paths`, la règle se charge à chaque session.
- Convention suit le *type/motif* de fichier → règle path-scoped. Convention suit la *localisation* → `CLAUDE.md` de sous-répertoire.
- `/memory` répond à "qu'est-ce que Claude a réellement chargé ?" ; le hook `InstructionsLoaded` ajoute plus de détail path-scoped. Ciblez moins de **200 lignes** par `CLAUDE.md`.

## Références

- Anthropic, *How Claude remembers your project* — [docs.anthropic.com/en/docs/claude-code/claude-md](https://docs.anthropic.com/en/docs/claude-code/claude-md) (mirror: [code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory))
- Anthropic, *Claude Code best practices* — [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)
- Anthropic, slash commands reference — [code.claude.com/docs/en/commands.md](https://code.claude.com/docs/en/commands.md)
- `anthropics/claude-cookbooks` CLAUDE.md example — [github.com/anthropics/claude-cookbooks/blob/main/CLAUDE.md](https://github.com/anthropics/claude-cookbooks/blob/main/CLAUDE.md)
- `paths` frontmatter YAML quoting bug — [github.com/anthropics/claude-code/issues/13905](https://github.com/anthropics/claude-code/issues/13905)
- Memory vs. Settings precedence inconsistency — [github.com/anthropics/claude-code/issues/18964](https://github.com/anthropics/claude-code/issues/18964)
- AGENTS.md open standard — [agents.md](https://agents.md/); v1.1 proposal [github.com/agentsmd/agents.md/issues/135](https://github.com/agentsmd/agents.md/issues/135)
