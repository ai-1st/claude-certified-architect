---
title: "Раздел 7 — Иерархия конфигурации CLAUDE.md и path-specific rules"
linkTitle: "7. CLAUDE.md & Rules"
weight: 7
description: "Domains 3.1, 3.3 — иерархия user/project/directory CLAUDE.md, @import patterns и glob-matched conventions в .claude/rules/."
---

## Что покрывает этот раздел

Как Claude Code finds, merges, and applies persistent instructions across **managed-policy / user / project / subdirectory** scopes, как держать large instruction sets modular with `@import` and `.claude/rules/`, и как использовать YAML `paths` globs, чтобы conventions load only when Claude touches matching files. Exam tests two failure modes: putting team standards in user-level scope (so a new teammate never sees them) and using subdirectory `CLAUDE.md` for cross-cutting conventions (test files spread across tree) instead of path-scoped rule. Sample Question 6 is a direct test of the second pattern.

## Исходный материал (из официального guide)

**3.1 — Hierarchy & modular organization.** Know user / project / subdirectory `CLAUDE.md` hierarchy; know that `~/.claude/CLAUDE.md` is per-user (never shared via VCS); know `@import` for modular content; know `.claude/rules/` as alternative to monolithic file; diagnose hierarchy issues with `/memory`.

**3.3 — Path-specific rules.** Know that `.claude/rules/*.md` use YAML frontmatter `paths` with globs, load only when Claude touches matching files, and beat subdirectory `CLAUDE.md` for conventions that cross-cut tree (e.g., `**/*.test.tsx`).

## Иерархия CLAUDE.md

### Locations & precedence

Claude Code loads multiple `CLAUDE.md` files in a defined order. Crucial mental model: **files are concatenated, not overridden** — every applicable file ends up in session context. Order matters because instructions closer to working directory appear *later* in context, giving them effective priority.

| Scope | Path | Use case | Shared via VCS? |
| --- | --- | --- | --- |
| Managed policy | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`, Linux/WSL `/etc/claude-code/CLAUDE.md`, Windows `C:\Program Files\ClaudeCode\CLAUDE.md` | Org-wide rules deployed by IT; cannot be excluded | No — deployed via MDM / Group Policy / Ansible |
| User | `~/.claude/CLAUDE.md` | Personal preferences across all projects | No — never committed |
| Project | `./CLAUDE.md` *or* `./.claude/CLAUDE.md` | Team conventions, build commands, architecture | Yes — committed |
| Local project | `./CLAUDE.local.md` | Personal sandbox URLs, test data | No — `.gitignore` it |
| Subdirectory | `path/to/dir/CLAUDE.md` | Instructions for one part of codebase | Yes if committed |
| User rules | `~/.claude/rules/*.md` | Personal rules across all projects | No |
| Project rules | `.claude/rules/*.md` | Team topic-specific or path-scoped rules | Yes |

Sources: official Claude Code memory docs at [docs.anthropic.com/en/docs/claude-code/claude-md](https://docs.anthropic.com/en/docs/claude-code/claude-md) and mirror at [code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory).

### How files are merged at session start

Claude walks up directory tree from current working directory, picking up every `CLAUDE.md` and `CLAUDE.local.md` it finds. Content is ordered **filesystem root → working directory**; within directory, `CLAUDE.local.md` appended after `CLAUDE.md`. Launching in `foo/bar/` yields roughly:

```
1. managed-policy CLAUDE.md (if present)
2. ~/.claude/CLAUDE.md + ~/.claude/rules/*
3. /foo/CLAUDE.md
4. /foo/bar/CLAUDE.md
5. /foo/bar/CLAUDE.local.md
6. .claude/rules/*.md  (rules without `paths` load here)
```

Subdirectory `CLAUDE.md` files **below** working directory are **not** loaded at launch — they load lazily when Claude reads a file in that subdirectory. After `/compact`, project-root `CLAUDE.md` is re-injected from disk, but nested `CLAUDE.md` files reload only when Claude next touches their subtree.

### Diagnosing with `/memory`

`/memory` — most important debugging tool for this domain. It lists every `CLAUDE.md`, `CLAUDE.local.md`, and rules file loaded in current session, lets you toggle auto memory, and opens any file in editor. If expected file isn't listed, Claude cannot see it.

Standard flow when "Claude isn't following my CLAUDE.md":

1. Run `/memory`. Is the file listed?
2. If not, is it in a loaded location? `~/.claude/CLAUDE.md` applies only to *your* sessions; teammate won't see it.
3. If listed, are instructions specific and non-conflicting? Two contradictory rules let Claude pick arbitrarily.
4. For path-scoped rules, use `InstructionsLoaded` hook to log when each file loads and why.

Canonical hierarchy trap: developer puts team standards in `~/.claude/CLAUDE.md`, it works great for them, then new hire asks "why does Claude keep using the wrong test framework?" Fix: move those rules into `./CLAUDE.md` or `./.claude/CLAUDE.md` and commit them.

## Modular organization with `@import`

### Syntax & limitations

Inside any `CLAUDE.md`, token `@path/to/file` expands referenced file inline at session start.

- Relative paths resolve **from the file containing the import**, not working directory. Absolute paths and `~/...` paths also work.
- Imports nest up to **5 hops deep**: `CLAUDE.md` → `@docs/guide.md` → `@docs/sub/detail.md` → … (five total levels max).
- First time Claude Code sees external imports in project, approval dialog lists files. Decline once and they stay disabled.
- Importing does **not** save tokens — every imported file loads in full at launch. Imports are for *organization*, not context reduction. Use `.claude/rules/` with `paths` for lazy loading.

### Example: per-package imports in a monorepo

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

Each package maintainer owns linked standards file without forcing every other team to read it. `~/.claude/...` import is canonical workaround for the fact that gitignored `CLAUDE.local.md` only exists in worktree where you created it.

## Topic files in `.claude/rules/`

`.claude/rules/` is a **documented Claude Code feature**, not Cursor-only convention. Official memory docs describe it under "Organize rules with `.claude/rules/`". All `.md` files in that directory are discovered recursively, in two flavors:

1. **Unconditional rules** — no `paths` frontmatter. Loaded at launch with same priority as `.claude/CLAUDE.md`.
2. **Path-scoped rules** — `paths` frontmatter with one or more globs. Load lazily when Claude reads a file matching one of patterns.

User-level rules in `~/.claude/rules/` apply to every project on your machine and load *before* project rules, so project rules win on conflict.

### Frontmatter schema (YAML)

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

Notes from spec and known YAML bug ([anthropics/claude-code#13905](https://github.com/anthropics/claude-code/issues/13905)): `paths` must be a YAML list; patterns starting with `*` or `{` **must be quoted** to be valid YAML; omitting `paths` makes rule load unconditionally.

### Glob `paths` semantics

Standard glob matchers — `*` matches within one path segment, `**` recurses across directories:

| Pattern | Matches |
| --- | --- |
| `*.md` | Markdown files in project root only |
| `**/*.ts` | TypeScript files in any directory |
| `**/*.test.tsx` | React test files anywhere in tree |
| `src/**/*` | Every file under `src/` |
| `src/components/*.tsx` | Direct children only |
| `src/**/*.{ts,tsx}` | TypeScript and TSX under `src/` (brace expansion) |
| `terraform/**/*` | Everything under `terraform/` |

Canonical exam example `**/*.test.tsx` works — textbook case for path-scoped rule.

### Example: cross-cutting tests rule (sample Question 6)

`.claude/rules/tests.md` covers test files that live next to source files (`Button.test.tsx` beside `Button.tsx`) anywhere in repo:

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

Swap `paths` for `["terraform/**/*", "**/*.tf"]` and you have same trick for infrastructure files anywhere in tree.

## CLAUDE.md vs subdirectory CLAUDE.md vs path-rules — when to use which

| Need | Best choice |
| --- | --- |
| Universal team standard (build, repo etiquette) | Root `CLAUDE.md` or `.claude/CLAUDE.md` |
| Personal preferences across projects | `~/.claude/CLAUDE.md` or `~/.claude/rules/` |
| Personal preferences in one repo | `CLAUDE.local.md` (gitignored) |
| Convention scoped to one folder | Subdirectory `path/to/CLAUDE.md` (lazy) |
| Convention spread across many folders (tests, `.tf`, `.sql`) | `.claude/rules/*.md` with `paths` glob |
| Big monolithic instructions | Split into `.claude/rules/` or `@import`s |
| Must run at a specific lifecycle point | A **hook**, not `CLAUDE.md` |
| Org-wide policy users can't opt out of | Managed-policy `CLAUDE.md` + managed `settings.json` |

Decisive heuristic: **does convention follow file *type/pattern* or file *location*?** Pattern → path-rule with glob. Location → subdirectory `CLAUDE.md`. Exactly what Question 6 tests.

## A reference layout

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

Each mechanism is in its sweet spot: small root file, modular topic rules, path scoping for cross-cutting conventions, subdirectory `CLAUDE.md` for genuinely location-bound rules, and `~/.claude/` for personal preferences.

## Anti-patterns & exam traps

- **Team standards in `~/.claude/CLAUDE.md`.** They follow you, not repo; new teammates get nothing. Move shared conventions to `./CLAUDE.md` or `./.claude/CLAUDE.md` and commit them.
- **One giant 2,000-line `CLAUDE.md`.** Everything in `CLAUDE.md` enters context window in full at launch; Anthropic recommends under ~200 lines per file. Split into `.claude/rules/` topic files and prefer path-scoped rules so most content stays out of context until needed.
- **Subdirectory `CLAUDE.md` for a cross-cutting convention.** Test files next to source across dozens of folders cannot be covered by directory-bound files without absurd duplication. Use single `.claude/rules/tests.md` with `paths: ["**/*.test.tsx"]`. This is sample Question 6.
- **Treating `@import` as a token-saving optimization.** It isn't — imports load in full at launch. Token saver is `.claude/rules/` with `paths` (lazy load).
- **Machine-enforceable rules in `CLAUDE.md`.** `CLAUDE.md` is advisory context, not enforcement. Anything that *must* happen (lint before commit, block writes to `secrets/`) belongs in hook or `permissions.deny`.
- **Assuming Claude reads `AGENTS.md`.** It doesn't, natively. If repo uses `AGENTS.md`, put `@AGENTS.md` at top of `CLAUDE.md` (recommended; portable) or `ln -s AGENTS.md CLAUDE.md` (Unix; Windows needs Admin/Developer Mode). `/init` in repo with `AGENTS.md` reads it and incorporates relevant parts.
- **Conflicting rules across files.** Claude may pick arbitrarily. Audit periodically and use `claudeMdExcludes` in `.claude/settings.local.json` to skip irrelevant teams' files in monorepo.

## Exam-style focus points

- Know all four scopes plus managed policy and which are shared via VCS. `~/.claude/CLAUDE.md` is trap — per-user, never committed.
- Files **concatenate** from filesystem root toward working directory; `CLAUDE.local.md` appends after `CLAUDE.md` in same directory.
- `@import` allows relative, absolute, and `~`-prefixed paths; nests **5 hops** max; does **not** save tokens.
- `.claude/rules/*.md` with `paths` frontmatter is the **only** mechanism providing lazy, glob-based convention loading. Without `paths`, rule loads every session.
- Convention follows file *type/pattern* → path-scoped rule. Convention follows file *location* → subdirectory `CLAUDE.md`.
- `/memory` answers "what did Claude actually load?"; `InstructionsLoaded` hook adds deeper path-scoped detail. Target under **200 lines** per `CLAUDE.md`.

## References

- Anthropic, *How Claude remembers your project* — [docs.anthropic.com/en/docs/claude-code/claude-md](https://docs.anthropic.com/en/docs/claude-code/claude-md) (mirror: [code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory))
- Anthropic, *Claude Code best practices* — [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)
- Anthropic, slash commands reference — [code.claude.com/docs/en/commands.md](https://code.claude.com/docs/en/commands.md)
- `anthropics/claude-cookbooks` CLAUDE.md example — [github.com/anthropics/claude-cookbooks/blob/main/CLAUDE.md](https://github.com/anthropics/claude-cookbooks/blob/main/CLAUDE.md)
- `paths` frontmatter YAML quoting bug — [github.com/anthropics/claude-code/issues/13905](https://github.com/anthropics/claude-code/issues/13905)
- Memory vs. Settings precedence inconsistency — [github.com/anthropics/claude-code/issues/18964](https://github.com/anthropics/claude-code/issues/18964)
- AGENTS.md open standard — [agents.md](https://agents.md/); v1.1 proposal [github.com/agentsmd/agents.md/issues/135](https://github.com/agentsmd/agents.md/issues/135)
