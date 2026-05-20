---
title: "Раздел 8 — Custom Slash Commands, Skills, Plan Mode и Iterative Refinement"
linkTitle: "8. Commands, Skills & Plan Mode"
weight: 8
description: "Domains 3.2, 3.4, 3.5 — .claude/commands/, .claude/skills/ с context: fork и allowed-tools, plan mode vs direct execution."
---

## Что покрывает этот раздел

Расширение Claude Code через **custom slash commands** и **agent skills**, выбор **plan mode** vs direct execution (и роль **Explore subagent**), а также техники **iterative refinement**. Sample Question 4 проверяет, где должен жить shared `/review` command; Sample Question 5 проверяет выбор plan mode for monolith-to-microservices restructure.

## Исходный материал (из официального guide)

**3.2 Slash commands and skills.** Project commands in `.claude/commands/` (shared via VCS) vs user commands in `~/.claude/commands/` (personal). Skills in `.claude/skills/<name>/SKILL.md` with YAML frontmatter supporting `context: fork`, `allowed-tools`, `argument-hint`. `context: fork` runs skill in isolated subagent context so verbose output doesn't pollute main conversation. Personal customization: variants in `~/.claude/skills/` with different name avoid affecting teammates. Skills are on-demand task-specific; `CLAUDE.md` is always-loaded universal standards.

**3.4 Plan mode vs direct execution.** Plan mode is for complex tasks with large-scale changes, multiple valid approaches, architectural decisions, multi-file modifications. Direct execution is for simple, well-scoped changes (one validation check, one-file fix with clear stack trace). Plan mode enables safe exploration before commitment. Explore subagent isolates verbose discovery and returns summaries. Common pattern: plan to investigate, then direct execution to apply.

**3.5 Iterative refinement.** Concrete input/output examples beat prose. Test-driven iteration: tests first, then iterate on failures. Interview pattern: have Claude ask clarifying questions in unfamiliar domains. Specific test cases for edge cases (e.g., nulls in migrations). Bundle interacting issues in one message; sequence independent issues.

## Four customization primitives

Claude Code exposes four mechanisms that plug into different parts of agentic loop. Knowing which to reach for is a near-certain exam target.

| Primitive | Trigger | Isolation | When to use |
|---|---|---|---|
| **Slash command** (`.claude/commands/foo.md`) | User types `/foo` | Main context | Reusable interactive workflow — `/review`, `/commit`, `/deploy-staging`. |
| **Skill** (`.claude/skills/foo/SKILL.md`) | User types `/foo` *or* Claude auto-invokes when description matches | Main context; isolated with `context: fork` | Task-specific knowledge or procedure with supporting files; lets Claude decide *when* to apply it. |
| **Subagent** (`.claude/agents/foo.md`) | Claude or user delegates a task | Always isolated; own tools and model | Verbose research, parallel work, specialists like `code-reviewer` or `Explore`. |
| **Hook** (`hooks.json`) | Lifecycle event fires automatically (PreToolUse, PostToolUse, UserPromptSubmit, etc.) | Shell script, output fed back | Deterministic side effects — auto-lint, block protected-path edits, append commit trailers. |

Anthropic **merged custom slash commands into skills** in late 2025: file at `.claude/commands/deploy.md` and skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and behave identically. Skills are recommended form because they add supporting files, invocation control, dynamic context injection, and subagent execution ([slash-commands docs](https://docs.anthropic.com/en/docs/claude-code/slash-commands)).

## Custom slash commands

### Project vs user scope

- **`.claude/commands/`** — project scope, version-controlled, team-wide workflows (`/review`, `/security-review`, `/migrate-route`).
- **`~/.claude/commands/`** — user scope, personal-only shortcuts (`/scratch`, `/jira-link`).

Sample Question 4 hinges on this: `/review` that "should be available to every developer when they clone or pull the repository" goes in **`.claude/commands/`** — not `~/.claude/commands/`, not `CLAUDE.md`, not fictional `.claude/config.json`.

### File layout and frontmatter

Command is a markdown file with optional YAML frontmatter:

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

Key frontmatter fields ([reference](https://docs.anthropic.com/en/docs/claude-code/slash-commands#frontmatter-reference)):

| Field | Purpose |
|---|---|
| `description` | Summary shown in `/help`; also drives auto-invocation. Keep under ~60 chars. |
| `argument-hint` | Autocomplete hint like `[issue-number]` or `[filename] [format]`. |
| `allowed-tools` | Tools usable without per-call approval. Accepts globbed Bash patterns like `Bash(git:*)`. |
| `model` | Override model for this command (`haiku`/`sonnet`/`opus`/`inherit`). |
| `disable-model-invocation` | `true` forces human-triggered only — for side-effecting workflows like `/deploy`. |

All fields are optional.

### Argument handling, bash execution, file references

Three substitution mechanisms ride inside markdown body:

- **`$ARGUMENTS`** — full argument string. `$ARGUMENTS[N]` indexes by position; `$N` is shorthand.
- **`` !`cmd` ``** — executes bash command at load time and inlines stdout (e.g., `` !`git diff HEAD` ``).
- **`@path`** — inlines file contents; `@$0` inlines file whose path is first argument.

> Exam note: older docs used `$1` for the *first* argument. Current docs renumber to 0-based (`$0`, `$1`, `$2`). Recognize either; prefer `$ARGUMENTS` when order doesn't matter.

### Example: `/review`

Project-scoped `/review` for Sample Question 4 — commit this to `.claude/commands/review.md` and every teammate gets `/review` on next pull:

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

### File layout

```
.claude/skills/pr-summary/
  SKILL.md          # required: frontmatter + instructions
  checklist.md      # optional supporting files
  example-good.md
```

Skills discovered at four levels — enterprise, user (`~/.claude/skills/`), project (`.claude/skills/`), and plugin. Enterprise overrides personal, personal overrides project; plugin skills live in `plugin-name:skill-name` namespace ([skills docs](https://docs.anthropic.com/en/docs/claude-code/skills)).

### Frontmatter reference

Beyond shared command fields, skills add:

| Field | Purpose |
|---|---|
| `name` | Skill identifier (defaults to directory name). |
| `description` | What skill does and when to use it; drives auto-invocation. `description` + `when_to_use` cap at 1,536 chars combined. |
| `when_to_use` | Extra trigger phrases appended to `description`. |
| `arguments` | Named positional args, e.g. `arguments: [issue, branch]` → `$issue`, `$branch`. |
| `user-invocable` | `false` hides from `/` menu; Claude can still use it as background knowledge. |
| `disable-model-invocation` | `true` blocks auto-invocation — pair with side-effecting skills (`/deploy`). |
| `context` | `fork` to run inside isolated subagent. |
| `agent` | Subagent type for forked skills (`Explore`, `Plan`, custom). |
| `hooks` | Lifecycle hooks scoped to this skill. |
| `paths` | Glob patterns that gate auto-invocation to matching files. |

### `context: fork` — what it does and when to use it

By default skill runs inline, so its output — file listings, intermediate thoughts, search results — eats into parent context window. `context: fork` **dispatches skill to a fresh subagent**: skill content becomes subagent prompt, subagent runs with own tools and permissions, and only final summary returns to parent.

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

Fork when skill is **verbose** (codebase analysis, large-diff summary), **exploratory** (brainstorming alternatives), or **independent** (returns a summary, not raw artifacts parent edits). Don't fork pure "use these conventions" skills — subagent gets guidelines but no actionable task and returns empty.

### Personal-variant pattern

To customize shared skill without affecting teammates, copy it under different name into `~/.claude/skills/` (e.g., `pr-summary-detailed/`). This avoids "I edited shared skill and now everyone gets my flavor" antipattern. Same renaming trick works for slash commands in `~/.claude/commands/`.

### Skills vs CLAUDE.md decision matrix

| Need | Pick |
|---|---|
| Universal standards active every session (tech stack, style) | `CLAUDE.md` |
| On-demand, task-specific procedure with supporting files | Skill |
| Interactive workflow with no auto-invocation | Skill with `disable-model-invocation: true` |
| Per-file-type convention gated on path globs | Path-scoped rule (`.claude/rules/`) or skill with `paths:` |
| Convention that must run as code, not advice | Hook |
| Verbose discovery that would flood context | Skill with `context: fork`, or delegate to Explore |

`CLAUDE.md` is **always-on context** — cheap to load, expensive at scale. Skills are **on-demand context** — heavier per use but free when unused.

## Plan mode vs direct execution

### How to enter plan mode

Plan mode is one of six permission modes; Claude reads files and runs read-only commands but cannot edit source ([permission-modes docs](https://docs.anthropic.com/en/docs/claude-code/permission-modes)). Four entry points:

1. **Cycle in-session** with `Shift+Tab`: `default → acceptEdits → plan`.
2. **At startup**: `claude --permission-mode plan` (also works with `-p` for headless).
3. **Project default**: `permissions.defaultMode: "plan"` in `.claude/settings.json`.
4. **Per-prompt**: prefix a message with `/plan`.

When plan ready Claude offers to (a) approve and switch to auto, (b) approve and review each edit, or (c) keep planning. `Ctrl+G` opens plan in editor; `Shift+Tab` exits without approving.

### When plan mode pays off

Reach for plan mode when any of:

- Change spans **many files or architectural seams** (microservices restructure, library migration affecting 45+ files).
- **Multiple valid approaches** with real tradeoffs (Redis vs in-memory vs file cache; webhook vs polling).
- **Scope is unknown** — question is "how big?" not "implement this."
- **High blast radius**: schema migrations, auth refactors, cross-team contracts.

Sample Question 5 is first bullet: restructuring monolith into microservices spans dozens of files plus boundary decisions — textbook plan mode. Wrong answer ("let implementation reveal the boundaries") loses to rework as soon as hidden dependency surfaces.

### When direct execution is correct

- Single-file bug with clear stack trace and obvious fix.
- One validation check, one log line, one feature flag.
- Mechanical refactor with known target (rename across N files — use `acceptEdits`).
- You already have a plan from prior turn.

Plan mode has overhead — extra exploration tokens for Claude, plan review for developer. Don't pay it on three-line changes.

### The Explore subagent for verbose discovery

Even outside plan mode, delegate discovery phase to built-in **Explore** subagent — read-only, Haiku-backed, access to Glob/Grep/Read/Bash but no Write/Edit. Each invocation runs in own context window and returns only summary ([sub-agents docs](https://docs.anthropic.com/en/docs/claude-code/sub-agents)).

Use Explore (directly, or via skill with `context: fork` and `agent: Explore`) for "where is X defined?" / "how does Y work?" questions, multi-phase tasks that would blow context budget on discovery, and broad searches whose raw output you won't reread. Specify thoroughness: `quick`, `medium`, or `very thorough`.

### Combined pattern: plan then execute

Recommended workflow for non-trivial work is **explore → plan → execute → commit** ([best practices](https://docs.anthropic.com/en/docs/claude-code/best-practices)): delegate survey to Explore, have Claude write plan, review/edit it with `Ctrl+G`, approve into `acceptEdits`, then commit. Plan mode for investigation, direct execution for edits.

## Iterative refinement playbook

### 1. Provide input/output examples

When prose ambiguity is bottleneck (extraction, transformation, formatting), supply **2–3 concrete input/output pairs** instead of more adjectives. Claude generalizes from examples more reliably than from prose like "friendly but professional." Examples double as regression cases.

### 2. Test-driven iteration

Tests are unambiguous verification signal. Loop: have Claude write suite covering happy path, edge cases, and performance budgets *before* implementation exists; run it and share failures; have Claude implement minimum needed to flip failing test green; repeat until green; refactor with suite as safety net. This is "give Claude a way to verify its work" principle made concrete.

### 3. The interview pattern

In unfamiliar domains, ask Claude to interview you before writing code: *"Before you implement caching, list the questions you'd need answered (eviction, invalidation, failure modes, memory budget, multi-tenant isolation) and ask each one."* This surfaces considerations developer didn't anticipate. Use it for cross-cutting concerns: caching, retries, auth, billing, migrations.

### 4. Batch interacting fixes vs sequence independent fixes

When multiple issues surface, decide whether they interact:

- **Interacting** — a fix in one place changes the right answer elsewhere (changing date format affects parsing, display, DB writes). Put all in a **single detailed message** so Claude reasons holistically.
- **Independent** — typo in helper, missing null check elsewhere. Fix **sequentially**, one per turn — small diffs, fresh context, attributable failures.

For single failing edge case (null in migration), supply specific input/expected output for that case — don't rewrite whole script.

## Exam-style focus points

- Team-shared commands live in **`.claude/commands/<name>.md`** in repo. Not `~/.claude/commands/`, not `CLAUDE.md`, not fictional `config.json`.
- Skills are **on-demand**, `CLAUDE.md` is **always-loaded**. Persistent standards go in `CLAUDE.md`; task-specific workflows go in skills.
- `context: fork` isolates verbose or exploratory skill output in subagent; pair with `agent: Explore` for read-only discovery.
- `allowed-tools` pre-approves a tool list while skill is active; it does not block other tools — permission rules still apply.
- `argument-hint` powers autocomplete; `$ARGUMENTS`/`$N` substitute actual values into prompt body.
- Plan mode entered via `Shift+Tab` cycle, `--permission-mode plan`, `/plan` prefix, or `defaultMode: "plan"`. It is read-only.
- Pick plan mode for architectural / multi-file / multi-approach work; pick direct execution for single-file, well-scoped changes.
- Explore subagent is built-in, Haiku-backed, read-only, and returns summaries — use it to keep discovery out of main context.
- Iteration: input/output examples beat prose; tests are highest-leverage feedback loop; interview in unfamiliar domains; batch interacting fixes, sequence independent ones.

## References

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
