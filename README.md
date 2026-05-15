# Claude Certified Architect — Foundations Prep

A practice repo for the **Claude Certified Architect — Foundations (CCA-F)** exam.

The repo turns Claude itself into your **proctor and question-writer**. Open the repo in [Claude Code](https://docs.claude.com/en/docs/claude-code) (or any Claude session that respects `CLAUDE.md`), say *"start a mock exam"*, and Claude will quiz you in the same scenario-based, four-option multiple-choice format used on the real exam — one section at a time, so a single round stays inside Claude's context window.

The same study material that powers the mock exam is published as a static website (Hugo + Hextra), suitable for Netlify hosting.

## How it works

Mock Exam Mode is defined entirely in [`CLAUDE.md`](./CLAUDE.md). The short version:

1. Claude greets you and asks which **section** (1–12) or which **mode** you want:
   - **Section drill** — pick one section, get questions until you stop.
   - **Scenario block** — 5 questions tied to one of the 6 official scenarios.
   - **Full mock** — 60 questions across 4 random scenarios, ~2 hours.
2. Claude reads **only that one section file** from [`sections/`](./sections/). This is a deliberate context-budget choice — the corpus is too big to load all at once, and one section is enough to ground rich, anti-pattern-aware distractors.
3. Claude generates one question at a time in the official format (scenario lead-in → stem → four options A/B/C/D), waits for your letter, then reveals the correct answer with a one-paragraph explanation in the official-guide voice.
4. At session end, Claude prints your score, per-domain breakdown, and the list of concepts you missed (with section pointers).

Hard rules baked into the spec: one section per round, never reveal the answer before you reply, distractors must be real anti-patterns from the loaded section, exactly four options, no web search, no code execution.

## The 12 sections

| # | Section | Domain coverage |
| --- | --- | --- |
| 1 | [Exam Overview, Structure & Scenarios](./sections/01-exam-overview.md) | meta |
| 2 | [Agentic Loops & `stop_reason` Handling](./sections/02-agentic-loops.md) | D1.1 |
| 3 | [Multi-Agent Orchestration](./sections/03-multi-agent-orchestration.md) | D1.2, D1.3 |
| 4 | [Hooks, Decomposition, Sessions & Forking](./sections/04-hooks-decomposition-sessions.md) | D1.4–D1.7 |
| 5 | [Tool Interface Design, Distribution & Built-ins](./sections/05-tool-design.md) | D2.1, D2.3, D2.5 |
| 6 | [MCP Integration & Structured Errors](./sections/06-mcp-integration.md) | D2.2, D2.4 |
| 7 | [`CLAUDE.md` Hierarchy & `.claude/rules/`](./sections/07-claude-md-and-rules.md) | D3.1, D3.2 |
| 8 | [Slash Commands, Skills & Plan Mode](./sections/08-commands-skills-plan-mode.md) | D3.3–D3.5 |
| 9 | [Claude Code in CI/CD](./sections/09-cicd-integration.md) | D3 (CI slice) |
| 10 | [Prompt Engineering](./sections/10-prompt-engineering.md) | D4.1, D4.2 |
| 11 | [Structured Output & Message Batches](./sections/11-structured-output-and-batch.md) | D4.3–D4.5 |
| 12 | [Context, Escalation, Provenance & Reliability](./sections/12-context-and-reliability.md) | D5 |

Domain weights on the real exam: **D1 27% · D2 18% · D3 20% · D4 20% · D5 15%.**

## Quick start (mock exam)

```bash
git clone https://github.com/ai-1st/claude-certified-architect.git
cd claude-certified-architect
claude            # or open the folder in Cursor / Claude Code
# then say:  Start a mock exam.
```

Claude will pick up `CLAUDE.md` automatically and walk you through Mock Exam Mode.

Other phrasings Claude understands out of the box:

- `Quiz me on section 5.`
- `Scenario block on customer support.`
- `Full mock — I have two hours.`
- `Just quiz me.`

## Static site (Hugo + Hextra)

The same `sections/` directory is also served as a browsable docs site using the [Hextra](https://imfing.github.io/hextra/) theme via Hugo modules. The `sections/` folder is mounted into Hugo at `/docs/` so there is no duplication.

### Local preview

```bash
hugo mod tidy
hugo server
```

Then open <http://localhost:1313>.

### Deploy on Netlify

The repo ships with `netlify.toml` configured for the Hextra build. In Netlify: **Add new site → Import an existing project → connect this repo** — defaults will pick up the build command and Hugo version automatically.

## Repo layout

```
.
├── CLAUDE.md             # Mock Exam Mode spec — this is the heart of the repo
├── README.md
├── LICENSE               # MIT
├── netlify.toml          # Hugo build for Netlify
├── hugo.yaml             # Hextra config + section mount
├── go.mod / go.sum       # Hugo modules (Hextra theme)
├── content/              # Site-only pages: home, about
└── sections/             # 12 study sections (single source of truth)
```

## Disclaimer

Unofficial. This repo is not affiliated with or endorsed by Anthropic. The CCA-F exam, its content domains, scenarios, and scoring are Anthropic's. The study material in `sections/` is an independent re-statement built from the publicly-available exam guide and community sources, organized for context-efficient practice with Claude.

## License

MIT — see [`LICENSE`](./LICENSE).

## Author

**Dmitry Degtyarev**
[ddegtyarev@gmail.com](mailto:ddegtyarev@gmail.com) · +7 702 730 6962 (WhatsApp) · [linkedin.com/in/mitek](https://linkedin.com/in/mitek)
