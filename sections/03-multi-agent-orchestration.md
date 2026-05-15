---
title: "Section 3 — Multi-Agent Orchestration: Coordinator–Subagent Patterns"
linkTitle: "3. Multi-Agent Orchestration"
weight: 3
description: "Domains 1.2–1.4 — coordinator/subagent boundaries, the Task tool, explicit context passing, parallel spawning, and structured handoffs."
---

## What this section covers

How to design a hub-and-spoke coordinator that decomposes work, spawns isolated subagents via the `Task`/`Agent` tool, passes complete context in each prompt, and enforces deterministic prerequisites (hooks, gates) so multi-step workflows hand off cleanly — to other agents or to humans — without losing state.

## Source material (from the official guide)

### 1.2 Coordinator–subagent patterns

- **Hub-and-spoke architecture**: a single coordinator agent manages all inter-subagent communication, error handling, and information routing.
- **Context isolation**: subagents do **not** inherit the coordinator's conversation history. Each starts with a fresh window.
- **Coordinator responsibilities**: task decomposition, delegation, result aggregation, dynamic selection of which subagents to invoke based on query complexity (rather than blindly routing through the full pipeline).
- **Key risk**: overly narrow decomposition. The canonical exam example is the "creative industries" query that gets split into *digital art, graphic design, photography* and silently omits music, writing, and film.
- **Skills tested**: dynamic subagent selection, partitioning scope to minimize duplication, iterative refinement loops (re-delegating when synthesis reveals gaps), routing every call through the coordinator for observability.

### 1.3 Subagent invocation & context passing

- **Spawning mechanism**: the `Task` tool (renamed `Agent` in Claude Code v2.1.63 — see [SDK note](#a-note-on-task-vs-agent)). The coordinator must list this tool in `allowedTools`.
- **Explicit context**: subagent context is whatever you put in the prompt string. No automatic inheritance of parent conversation, tool results, or memory.
- **`AgentDefinition`**: per-subagent configuration object containing `description` (when to invoke), `prompt` (system prompt), `tools` / `disallowedTools` (capability restrictions), and optional `model`, `skills`, `mcpServers`, `permissionMode`.
- **`fork_session`**: branches a session into a new one that shares prior history up to a chosen message — used for divergent exploration from a shared baseline.
- **Skills tested**: passing complete prior findings in the spawn prompt, using structured formats (content + metadata: URLs, doc names, page numbers), emitting multiple `Task` calls in **one** coordinator response for parallelism, and writing goal/quality-criteria-style prompts rather than step-by-step procedures.

### 1.4 Workflows with enforcement & handoff

- **Programmatic enforcement (hooks, prerequisite gates)** vs **prompt-based guidance**: when deterministic compliance is required (identity verification before a financial transaction), prompts alone have a non-zero failure rate.
- **Structured handoff protocols** for mid-process escalation: customer ID, root-cause analysis, recommended action, evidence trail.
- **Skills tested**: blocking `process_refund` until `get_customer` has returned a verified ID, decomposing multi-concern requests into parallel investigations sharing context, and compiling escalation summaries for humans who lack the transcript.

## Architecture deep-dive

### Hub-and-spoke topology

The coordinator is the *only* node that talks to subagents. Subagents never address each other directly. Every result returns to the hub, which decides what to do next.

```mermaid
flowchart TD
    User([User query]) --> Coord[Coordinator agent<br/>Opus / lead model<br/>holds plan + state]
    Coord -->|Task call #1<br/>scope: subtopic A| S1[Subagent A<br/>fresh context<br/>tools: Read, Grep, WebSearch]
    Coord -->|Task call #2<br/>scope: subtopic B| S2[Subagent B<br/>fresh context<br/>tools: Read, Grep, WebSearch]
    Coord -->|Task call #3<br/>scope: subtopic C| S3[Subagent C<br/>fresh context<br/>tools: Read, Grep, WebSearch]
    S1 -->|final message only| Coord
    S2 -->|final message only| Coord
    S3 -->|final message only| Coord
    Coord -->|synthesis + gap check| Coord
    Coord -->|optional re-delegate| S1
    Coord --> Citation[Citation / verification subagent]
    Citation --> Coord
    Coord --> User
```

Two properties follow from this topology: **observability** (every action flows through one node, so coordinator-side logging captures the full causal graph) and **bounded blast radius** (a misbehaving subagent corrupts only its own context; its final message is the only thing the hub sees, and the hub can reject or re-delegate).

Anthropic's Research feature uses exactly this pattern: a `LeadResearcher` (Opus) plans, persists the plan to memory (the 200k window can be truncated mid-task), spawns parallel subagents (Sonnet), and hands findings to a `CitationAgent` that re-attributes every claim. Anthropic reports a **90.2% lift** over single-agent Claude Opus 4 on internal evals, at **~15× the tokens of a chat** — economical only when task value is high. ([Anthropic engineering blog](https://www.anthropic.com/engineering/multi-agent-research-system))

### Why context isolation matters

Each subagent's window starts fresh. The Agent SDK documents the boundary precisely ([Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)):

| The subagent receives | The subagent does **not** receive |
| --- | --- |
| Its own `AgentDefinition.prompt` | The parent's conversation history or tool results |
| The string passed in the `Task`/`Agent` tool call | The parent's system prompt |
| Tool definitions (inherited or restricted via `tools`) | Preloaded skills, unless declared in `AgentDefinition.skills` |
| Project `CLAUDE.md` (when `settingSources` is enabled) | Memory the parent built up across turns |

Three practical consequences: **(1) Compression is the feature** — a subagent can read fifty files; only its final message returns to the parent, keeping the lead's context clean. **(2) Anything the subagent needs must be in the prompt** — file paths, prior URLs, prior decisions, the user constraint that "we already ruled out option B." All of it is the coordinator's responsibility to forward. **(3) No back-channel** — if subagents need shared state, write it to a filesystem or external store and pass references back to the coordinator. Anthropic calls this the "artifact" pattern; it sidesteps the *game of telephone* where each handoff degrades fidelity.

## Spawning subagents with the Task tool

A coordinator that can spawn subagents needs three things: the `Task` (or `Agent`) tool in `allowedTools`, one or more `AgentDefinition`s, and a prompt that invites delegation. The example below defines two specialized subagents with restricted toolsets and then issues parallel calls from a single coordinator turn.

```typescript
import { query, type AgentDefinition } from "@anthropic-ai/claude-agent-sdk";

const webResearcher: AgentDefinition = {
  description: "Web research specialist. Breadth-first search across the web.",
  prompt: `GOAL: JSON list of {url, title, claim, snippet, retrieved_at}.
QUALITY: prefer primary sources; 5-15 per facet; never invent URLs;
start broad then narrow (don't lead with overly specific queries).`,
  tools: ["WebSearch", "WebFetch", "Read"],
  model: "sonnet",
};

const docAnalyst: AgentDefinition = {
  description: "Document analyst. Extract structured facts from supplied files.",
  prompt: `OUTPUT: JSON list of {source_doc, page, quote, normalized_claim}.
Never paraphrase a number; quote verbatim with its page.`,
  tools: ["Read", "Grep", "Glob"],
  model: "sonnet",
};

for await (const message of query({
  prompt: `Research "the impact of AI on creative industries". Cover full
domain breadth: visual arts AND music AND writing AND film/TV AND performing
arts. Decompose into AT LEAST one subagent per medium and emit the Task calls
in a SINGLE response so they run in parallel. After synthesis, check for
omitted media and re-delegate if any are missing.`,
  options: {
    allowedTools: ["Read", "Grep", "Glob", "WebSearch", "WebFetch", "Task"],
    agents: { "web-researcher": webResearcher, "doc-analyst": docAnalyst },
  },
})) {
  if ("result" in message) console.log(message.result);
}
```

Key points the exam tests:

- **`"Task"` (or `"Agent"`) must be in `allowedTools`** on the coordinator, or Claude cannot spawn anything.
- **Never put `Task` / `Agent` in a subagent's `tools`** — the SDK documents this as a hard rule to prevent recursive spawning.
- **Parallelism = multiple tool calls in one assistant turn**, not separate turns. Prompt the lead to "emit the Task calls in a single response." Anthropic reports 3–5 parallel subagents (each making 3+ parallel tool calls) cut research time by up to 90% on complex queries.
- **Prompts specify goals and quality criteria, not procedures.** "Objective + output format + tool guidance + task boundaries" gave Anthropic the largest single quality lift; vague instructions caused duplication and silent gaps.

### `fork_session` for divergent exploration

When two subagents should try *different approaches from the same baseline* — e.g., one optimization branch tries a SQL rewrite while another tries an index addition — use `fork_session` rather than re-spawning from scratch ([Sessions docs](https://code.claude.com/docs/en/agent-sdk/sessions)). Fork copies the conversation up to a chosen message, remaps UUIDs to avoid collisions, and tags each entry with `forkedFrom` for lineage. Each fork resumes independently; the original is preserved — ideal for A/B exploration without polluting the baseline.

### A note on `Task` vs `Agent`

The certification guide calls the spawning mechanism the **`Task` tool**. The SDK renamed it to **`Agent`** in Claude Code v2.1.63. Current SDK releases emit `"Agent"` in new `tool_use` blocks but still emit `"Task"` in `system:init` tools list and `permission_denials[].tool_name`. For the exam: treat `Task` as canonical (that's the wording in the questions). In production 2026 code, match **both** names defensively (`block.name in ("Task", "Agent")`).

## Context-passing patterns

Because subagents inherit nothing automatically, the coordinator's job is to pack a self-contained briefing into each spawn. The principle the exam rewards: **separate content from metadata, in a structured format, so attribution survives handoff.**

A high-quality spawn prompt is structured JSON with content separated from metadata:

```json
{
  "task": "Extend findings on AI's impact on the music industry.",
  "goal": "5-10 sourced 2025-2026 claims on production, distribution, royalties.",
  "prior_findings": [
    {
      "claim": "Major labels sued Suno and Udio in June 2024.",
      "source_url": "https://example.org/riaa-suno-2024",
      "source_title": "RIAA files suit against Suno",
      "retrieved_at": "2026-05-10",
      "confidence": "high"
    },
    {
      "claim": "AI-generated tracks: ~18M streams/day on Deezer in Q1 2025.",
      "source_url": "https://example.org/deezer-q1-2025",
      "page": 14, "retrieved_at": "2026-04-29", "confidence": "medium"
    }
  ],
  "open_questions": ["EU/US 2026 regulatory developments?"],
  "output_format": "list of {claim, source_url, source_title, page?, retrieved_at, confidence}",
  "do_not": ["duplicate prior findings", "rely on a single source for a number", "invent URLs"]
}
```

Why this format is rewarded: **metadata travels with content** (`source_url`, `page`, `retrieved_at` survive the next hop, so a downstream synthesis agent has the page reference rather than a paraphrase); **open questions partition the scope** so the subagent doesn't redo work; **explicit "do not" lines** are cheaper than retries; and **the schema is machine-checkable** by the coordinator before re-delegation.

## Enforcement: hooks vs prompts

Prompt-based guidance — "always call `get_customer` before `process_refund`" — is *probabilistic*. Even a well-tuned Claude 4 model has a non-zero failure rate, unacceptable for financial, security, or compliance flows. A `PreToolUse` hook turns the rule into a deterministic gate.

```typescript
import { query, type HookCallback, type PreToolUseHookInput } from
  "@anthropic-ai/claude-agent-sdk";

const verifiedCustomers = new Set<string>();

const requireVerifiedCustomer: HookCallback = async (input) => {
  const pre = input as PreToolUseHookInput;
  const args = pre.tool_input as Record<string, unknown>;

  if (pre.tool_name === "get_customer" && args.verified === true) {
    verifiedCustomers.add(String(args.customer_id));
    return {};
  }
  if (pre.tool_name === "process_refund" &&
      !verifiedCustomers.has(String(args.customer_id))) {
    return { hookSpecificOutput: {
      hookEventName: pre.hook_event_name,
      permissionDecision: "deny",
      permissionDecisionReason: "Refund blocked: customer not verified. " +
        "Call get_customer first and obtain a verified ID.",
    }};
  }
  return {};
};

for await (const message of query({
  prompt: "Refund order #88421 for customer C-1042.",
  options: {
    allowedTools: ["get_customer", "process_refund", "Task"],
    hooks: { PreToolUse: [{ matcher: "get_customer|process_refund",
                            hooks: [requireVerifiedCustomer] }] },
  },
})) {
  if ("result" in message) console.log(message.result);
}
```

### When to use which

| Concern | Prompt-based guidance | Programmatic enforcement (hooks / gates) |
| --- | --- | --- |
| Style, tone, formatting | Yes — flexible, cheap | Overkill |
| Tool ordering preferences | Yes | Only if compliance-bound |
| Identity verification before financial action | **No** — non-zero failure rate is unsafe | **Yes** — `PreToolUse` deny |
| Writes to protected paths (`.env`, `/etc`) | No | Yes |
| Audit logging of every tool call | Optional | Yes — `PostToolUse` |
| Cross-subagent prerequisites | No | Yes — hook state across `SubagentStart` / `SubagentStop` |
| Approval routing to a human | Possible but unreliable | Yes — `PermissionRequest` / `canUseTool` |

Rule of thumb: **if a wrong answer is irreversible or regulated, the rule belongs in code, not a prompt.** Hooks are also the right place to *normalize* data flowing between subagents (Domain 1.5) — by the time content reaches a downstream agent, it's been schema-checked.

## Handoff to humans

A human agent picking up an escalation has none of the conversation transcript. A well-formed handoff summary is therefore part of the coordinator's contract. The template below is what the exam scenarios reward:

```yaml
handoff:
  type: human_escalation
  reason: policy_exception_required
  urgency: medium
  customer:
    id: C-1042
    verified: true
    verification_method: email_otp
    verified_at: 2026-05-15T14:22:11Z
  case:
    ticket_id: T-58219
    concerns:
      - {type: refund_request, order: O-88421, amount: 249.00, status: blocked_by_policy}
      - {type: account_merge, target: C-0997, status: needs_review}
  root_cause:
    summary: >
      Duplicate charge on O-88421 from a payments retry after a 504. Refund
      automation cannot fire because the duplicate is on a different account
      (C-0997) the customer also owns.
    evidence: [pay_log/2026-05-14T22:14Z#retry-3, orders/O-88421/events#charge-retry]
  attempted_actions:
    - {tool: get_customer, result: verified}
    - {tool: lookup_order, result: duplicate_charge_confirmed}
    - {tool: process_refund, result: blocked,
       blocked_by: prerequisite_gate (cross-account refund needs approval)}
  recommended_action:
    - merge C-1042 and C-0997 (manual review queue)
    - issue refund of 249.00 against the merged account
    - apply goodwill credit of 25.00 per playbook PB-17
  policy_refs: [PB-17, SEC-3]
```

The human needs **identity** (don't re-verify), **case state** (don't redo lookups), **root cause** (don't re-investigate), **attempted actions and why they failed** (don't repeat or undo them), and **recommended action + policy refs** (consistency with prior cases). The same structure also works for subagent-to-subagent escalation.

## Common failure modes (and fixes)

| Failure mode | Symptom | Fix |
| --- | --- | --- |
| **Narrow decomposition** (the "creative industries → only visual arts" trap from Question 7) | All subagents complete successfully, but the final report silently omits entire domains. Coordinator logs show the decomposition was already incomplete. | Coordinator prompt must demand **domain-breadth check before delegation** and a **gap audit after synthesis** with re-delegation when gaps are found. Tag the decomposition with the canonical list of subdomains and reject if any are missing. |
| **Over-provisioned subagent** (the Question 9 trap) | A synthesis subagent given full web-search tools so it never needs to round-trip. Solves latency, breaks separation of concerns; the synthesis agent is now also a researcher. | Apply least privilege: scope-restricted `verify_fact` tool for the 85% simple-lookup case; keep coordinator-routed delegation for the 15% deep cases. |
| **Sequential where parallel was possible** | Latency scales linearly with subagent count. Coordinator emits one `Task` call, waits, emits the next. | Coordinator prompt must explicitly instruct emitting multiple `Task` calls in a **single response**. Confirm via tracing that the assistant turn contained N tool_use blocks. |
| **Missing prerequisite gate** | `process_refund` occasionally fires without a verified customer, especially under prompt drift or model upgrades. | Move the rule into a `PreToolUse` hook that denies the downstream tool until a state flag set by the prerequisite tool is present. |
| **Lossy handoff to humans** | Human agent re-verifies identity, re-investigates, makes a different decision from the agent's recommendation. | Standardize a structured handoff schema (see above) and validate it at the escalation boundary. |
| **Subagent context starvation** | Subagent invents URLs, repeats prior searches, or contradicts prior findings. | Coordinator forgot it must *explicitly* pass prior findings + metadata. Use structured JSON briefings, not paraphrased prose. |
| **Telephone-game decomposition** | Splitting one feature across planner / implementer / tester / reviewer subagents; coordination tokens exceed actual work tokens. | Use **context-centric** decomposition: split by context boundary, not by job title. An agent that owns a feature also owns its tests. Reserve multi-agent for truly parallel, low-coupling work. ([Anthropic guidance](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)) |
| **Coordinator drift on long runs** | After 100+ turns the lead loses its plan. | Persist the plan to memory at the start (Research-feature pattern); on context pressure, spawn a fresh coordinator with the plan + summary handoff. |

## Exam-style focus points

- **Hub-and-spoke** is the default topology; subagents never talk to each other.
- **Subagents inherit nothing** — no conversation, no tool results, no parent system prompt. Pack what they need into the `Task` prompt.
- **`Task` in coordinator's `allowedTools`**, never in a subagent's `tools` (recursion).
- **`AgentDefinition`** = `description`, `prompt`, `tools`/`disallowedTools`, plus optional `model`, `skills`, `mcpServers`, `permissionMode`.
- **Parallelism = multiple `Task` calls in one coordinator turn.** Sequential is the failure mode.
- **Prompts specify goals and quality criteria**, not procedural steps.
- **Pass prior findings with metadata** (URL, doc, page, retrieval timestamp) in structured form.
- **`fork_session`** = divergent exploration from a shared baseline. Not the same as parallel decomposition.
- **Hooks > prompts for deterministic compliance.** Identity verification before financial ops belongs in a `PreToolUse` hook.
- **Structured handoff** to humans: customer ID, verification status, case state, root cause, attempted actions, recommended action, policy refs.
- **"Creative industries" trap**: all subagents succeed but coverage is incomplete → the coordinator's *decomposition* is the root cause.
- **"Over-provisioned synthesis agent" trap**: scope new tools to the 85% case (least privilege); keep coordinator-routed delegation for the 15%.
- Multi-agent costs **3–15× more tokens** than single-agent. Justified only for breadth-first, parallelizable, high-value tasks.

## References

- [Anthropic — How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Anthropic — When to use multi-agent systems (and when not to)](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)
- [Claude Agent SDK — Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- [Claude Agent SDK — Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Agent SDK — Work with sessions (incl. `fork_session`)](https://code.claude.com/docs/en/agent-sdk/sessions)
- [Claude Agent SDK — Intercept and control agent behavior with hooks](https://code.claude.com/docs/en/agent-sdk/hooks)
- [Claude Agent SDK — Handle approvals and user input](https://code.claude.com/docs/en/agent-sdk/user-input)
- [Claude Agent SDK — TypeScript reference (`AgentDefinition`)](https://code.claude.com/docs/en/sdk/sdk-typescript)
- [Anthropic cookbook — Agent workflow patterns](https://platform.claude.com/cookbook/patterns-agents-basic-workflows)
