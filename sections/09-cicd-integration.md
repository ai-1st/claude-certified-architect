---
title: "Section 9 — Claude Code in CI/CD Pipelines"
linkTitle: "9. CI/CD Integration"
weight: 9
description: "Domain 3.6 — claude -p and --output-format json, --json-schema, prompt design for actionable, low-false-positive review."
---

## What this section covers

How to run Claude Code unattended inside a build system: the flags that prevent interactive hangs, the output shapes that downstream tools can parse, the project-level context that keeps the model on-policy, and the architectural patterns (independent reviewer, multi-pass review, batch-vs-sync) that the exam tests through Questions 10, 11, and 12.

## Source material (from official guide)

Task Statement 3.6 (guide lines 600–629) covers: `-p` / `--print` for non-interactive mode; `--output-format json` plus `--json-schema` for machine-parseable findings; `CLAUDE.md` as the project-context mechanism for CI-invoked Claude Code (testing standards, fixture conventions, review criteria); and session-context isolation — the session that *wrote* the code is the wrong session to *review* it. Skills: prevent interactive hangs with `-p`; produce structured findings for inline PR comments; feed prior review findings back in to suppress duplicate comments; pass existing test files to test-generation runs to avoid re-covering scenarios; document testing standards and fixtures in `CLAUDE.md`.

Sample Question 10 (`-p` is the correct headless flag; `CLAUDE_HEADLESS`, `--batch`, and `< /dev/null` are distractors), Question 11 (Batches API for the overnight tech-debt report, *not* the blocking pre-merge gate), and Question 12 (split a 14-file PR into per-file passes plus an integration pass) all key off this section.

## Headless / non-interactive Claude Code

### The `-p` flag

`claude -p "<prompt>"` (alias `--print`) makes Claude Code run as a one-shot SDK query: it consumes the prompt, streams or prints the response to stdout, and exits with a status code. No TTY check, no permission prompt loop, no waiting on stdin. This is the only documented mechanism for CI. The trap from Question 10 is that there is no `CLAUDE_HEADLESS` env var and no `--batch` flag, and redirecting stdin from `/dev/null` does not work because Claude Code still expects to render an interactive UI.

```bash
claude -p "Review the diff for SQL-injection and SSRF risks. Be terse." \
  --model claude-sonnet-4-6 \
  --max-turns 6
```

Add `--bare` (Claude Code 2.1.x) when you do not need hooks, plugins, MCP servers, or auto-discovered `CLAUDE.md`; it cuts startup latency and removes a class of "why did my CI grab my laptop's MCP config" failures.

### Output formats (text, json, stream-json)

`--output-format` controls how the response is serialized:

- `text` (default) — plain prose; good for humans, bad for parsers.
- `json` — one JSON object with the final response, cost, duration, session ID, stop reason. Use when the next step is `jq` or a script.
- `stream-json` — newline-delimited events for tools that render progress or surface tool-use events. Pair with `--include-partial-messages` for token-by-token deltas, `--include-hook-events` for hook lifecycle.

`--input-format stream-json` lets a parent process feed Claude Code a structured event stream — useful if your CI orchestrator is itself an agent.

### `--json-schema` for structured findings

`--json-schema '<JSON Schema>'` (print mode only) makes Claude Code emit a final payload that is *validated* against the schema you supply. This is the flag the exam expects you to reach for when the next stage of the pipeline is "post each finding as an inline GitHub PR review comment."

```bash
claude -p "Review changed files in $PR_DIFF for security issues" \
  --output-format json \
  --json-schema "$(cat .ci/review-schema.json)" \
  --max-turns 8 \
  > findings.json
```

A schema that maps cleanly to GitHub's `pulls/{pr}/comments` payload looks like:

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

### Permission modes for unattended runs

Interactive permission prompts are the second-most-common cause of stuck CI jobs (after forgetting `-p`). Key flags:

- `--allowedTools "Read,Bash(git diff *),Bash(gh pr view *)"` — narrow allowlist; preferred default.
- `--disallowedTools "Edit,Write,Bash(rm *)"` — block-only writes when the job is read-only review.
- `--permission-mode acceptEdits` (or `bypassPermissions`) — accept changes without prompts. `--dangerously-skip-permissions` is the explicit "I know" form; reserve for sandboxed runners with no secrets in env.
- `--permission-prompt-tool <mcp-tool>` delegates permission decisions to a custom MCP tool when every call must be logged before approval.

### Session control flags

- `--session-id <uuid>` pins a UUID so downstream steps can resume the same conversation later.
- `--resume <id>` / `-r` continues that session — the mechanism behind "include prior review findings when re-running."
- `--continue` / `-c` picks the most recent conversation in the current directory.
- `--fork-session` resumes-but-branches so retries do not mutate the canonical thread.
- `--no-session-persistence` writes nothing to disk; useful in stateless runners where the workspace is destroyed between jobs.

## Reference patterns

### 1. Pre-merge security review (synchronous, blocking)

Blocking gates must use a real-time call. Per Question 11, the Message Batches API is the wrong tool here — its 50%-cost discount comes at the price of up to 24-hour latency, which is not acceptable for a developer waiting on a merge.

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

### 2. Test generation in nightly batch

Overnight test-debt jobs are exactly the workload where the Batches API earns its 50% discount. Run Claude Code with `-p`, pass the existing test files in context so it does not re-cover scenarios, and route the generation requests through the Batches API (see Section 11).

```bash
ls tests/ | xargs -I{} cat tests/{} > .ci/existing-tests.txt

claude -p \
  "Generate missing tests for src/billing/. Existing tests are attached; do not duplicate scenarios already covered." \
  --append-system-prompt-file .ci/existing-tests.txt \
  --output-format json \
  --max-turns 12
```

### 3. Inline PR comment posting from JSON output

With a schema-validated `findings.json`, a single shell loop turns the file into native review comments:

```bash
jq -c '.findings[]' findings.json | while read f; do
  gh api -X POST "repos/$REPO/pulls/$PR/comments" \
    -f body="$(echo $f | jq -r .message)" \
    -f path="$(echo $f | jq -r .path)" \
    -F line="$(echo $f | jq -r .line)" \
    -f commit_id="$SHA" -f side=RIGHT
done
```

### 4. Re-running review on new commits without duplicate comments

Domain 3.6 is explicit: when the PR is re-pushed, include prior findings in context and instruct Claude to report only new or still-unaddressed issues.

```bash
gh pr view $PR --json comments -q '.comments' \
  | claude -p \
      "Comments already posted are attached. Re-review the new diff and emit ONLY findings that are (a) new or (b) still unaddressed. Use the findings schema." \
      --resume "$REVIEW_SESSION_ID" \
      --output-format json \
      --json-schema "$(cat .ci/review-schema.json)" \
      > new-findings.json
```

## The Claude Code GitHub Action

### Quick start

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

Key inputs from `action.yml`: `anthropic_api_key` (or `claude_code_oauth_token`, `use_bedrock`, `use_vertex`); `prompt`; `claude_args` (raw CLI args appended to the underlying `claude -p`); `trigger_phrase` (default `@claude`); `label_trigger` (default `claude`); `track_progress` (renders a tracking comment with checkboxes); `use_sticky_comment` (one comment for all); `classify_inline_comments` (Haiku pre-filter to drop probe/test comments); `branch_prefix` (default `claude/`). Structured JSON returned by the run is exposed as GitHub Action outputs.

### Common gotchas

- The MCP inline-comment tool needs `confirmed: true` to post immediately; otherwise comments are buffered to `/tmp/inline-comments-buffer.jsonl` for Haiku classification.
- The Action needs `pull-requests: write` and `issues: write` token permissions, otherwise it silently fails to post.
- Pass model selection and turn caps via `claude_args`, not via separate inputs.

## CLAUDE.md for CI

### Sections that pay off

For a CI-invoked Claude, the highest-leverage `CLAUDE.md` sections are the ones the model otherwise has to guess at: the test runner command, where fixtures live, which mocking library you use, what counts as a "valuable" test, and the review criteria you care about — exactly the testing standards, fixtures, and review criteria the Skills bullets for 3.6 call out.

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

### Avoiding context bloat

`CLAUDE.md` is loaded on every invocation, so every line is paid for on every CI run. Strip historical decisions, link out to ADRs, prefer one declarative line over a paragraph, and resist the urge to paste the entire style guide. In `--bare` mode `CLAUDE.md` is not auto-loaded at all; opt-in via `--append-system-prompt-file` when you want it.

## Independent reviewer pattern

### Why a separate session beats self-review

Domain 3.6 explicitly calls out *session context isolation*. The session that generated the code carries its own reasoning history — the very justifications that produced the bug. That session is biased toward confirming its earlier decisions. A fresh session sees only the diff, with no narrative attached, and catches issues the author's session anchored away from. Anthropic's own Code Review product uses a fleet of independent agents for the same reason.

In practice: never `--continue` from the implementation session into the review job. Spawn the reviewer with a new `--session-id`, an empty conversation, and only the diff plus `CLAUDE.md` as context.

### Multi-pass for large PRs (per-file + cross-file integration)

This is Question 12. A single review pass over 14 files exhibits *attention dilution* — depth varies file-to-file and identical patterns get contradictory verdicts. The fix is two passes:

1. **Per-file pass.** One `claude -p` invocation per changed file, scoped to local issues (bugs, style, security in this file).
2. **Integration pass.** One additional invocation given the diff summary and asked specifically about cross-file concerns: API boundary changes, schema migrations, contract drift, shared-state mutation.

A larger model or bigger context window does not solve this; the issue is allocation of attention, not token budget.

## Cost & latency knobs

### Prompt caching

Cache reads cost roughly 10x less than cache writes, and Claude Code is built around hitting the cache aggressively. To maximize hits in CI: put stable content (system prompt, `CLAUDE.md`, repo conventions) *first*, dynamic content (diff, prior findings) *last*; do not change the model or tool list mid-session; on Claude Code 2.1.108+, set `ENABLE_PROMPT_CACHING_1H=1` for scheduled jobs that fire more than every five minutes. Use `--exclude-dynamic-system-prompt-sections` with `-p` to keep per-machine details out of the cache key when many runners share the same task.

### Batch API for non-blocking workloads

Cross-link to Section 11: the Message Batches API gives 50% cost savings but up to 24-hour SLA. Use it for the overnight tech-debt report; do not use it for the pre-merge gate. This is the distinction Sample Question 11 tests.

### Choosing model size by job

Pre-merge security review on a hot path: Sonnet. Style-and-lint pass on docs PRs: Haiku. Architecture review on the quarterly refactor: Opus, gated by `--max-budget-usd`. Switch model via `--model`; do not switch mid-session or you invalidate the cache.

## Exam-style focus points

- `-p` is the *only* documented way to run Claude Code unattended. `CLAUDE_HEADLESS`, `--batch`, and `</dev/null` are distractors.
- `--output-format json --json-schema <schema>` is the canonical recipe for pipeline-parseable findings.
- Blocking workloads (pre-merge) use real-time calls; non-blocking overnight workloads use the Batches API.
- Large PRs: per-file passes plus one integration pass — not a single mega-prompt and not a bigger context window.
- The session that wrote the code is the wrong session to review it; spawn an independent reviewer.
- Re-running review must include prior findings in context to suppress duplicate comments.
- `CLAUDE.md` carries testing standards, fixtures, and review criteria — every line is paid for on every CI run, so keep it lean.

## References

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
