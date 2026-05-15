---
title: "Section 11 — Structured Output via tool_use, Batch Processing & Multi-Pass Review"
linkTitle: "11. Structured Output & Batch"
weight: 11
description: "Domains 4.3, 4.5, 4.6 — JSON Schema with nullable/optional fields, validation-retry loops, and Message Batches API tradeoffs."
---

## What this section covers

Three related architectural decisions: **how** to make Claude emit machine-parseable output (`tool_use` + JSON schemas, `strict: true`, the newer Structured Outputs feature), **when** to move work onto the **Message Batches API** for cost and throughput (and when not to), and **why** an independent reviewer or a per-file + integration split beats self-review.

The exam tests all three with concrete scenarios — pre-merge checks versus overnight reports, single-pass versus per-file review, JSON-only prompts versus `tool_use`. The right answers come from one principle: **match the technique to the workload's latency, reliability, and attention-budget constraints**.

## Source material (from official guide)

### 4.3 Structured output via tool_use & JSON schemas
- `tool_use` with JSON schemas is the most reliable approach for guaranteed schema-compliant output; it eliminates JSON syntax errors.
- `tool_choice` modes: `"auto"` (model may return text), `"any"` (must call a tool, can pick which), or forced `{"type": "tool", "name": "..."}`.
- Strict schemas eliminate **syntax** errors only — not **semantic** errors (line items not summing to total, values in wrong fields, fabricated content for missing data).
- Schema design: required vs optional/nullable, enums with `"other"` + detail string for extensibility, `"unclear"` for ambiguity, format normalization rules in the prompt.

### 4.5 Batch processing strategy
- Message Batches API: **50% cost savings**, up to **24-hour** processing window, **no latency SLA**.
- Good for non-blocking, latency-tolerant workloads (overnight reports, weekly audits, nightly test generation). Bad for blocking workflows (pre-merge checks).
- Per the guide: the Batch API does **not** support multi-turn *agentic* tool execution within a single request — you cannot pause mid-request to run a tool and feed results back.
- `custom_id` correlates request and response; failed `custom_id`s can be resubmitted after fixes (e.g., chunking documents that exceeded context).

### 4.6 Multi-instance & multi-pass review
- A model that retains its generation reasoning is less likely to question its own decisions, so **self-review** is structurally weak.
- Independent review instances catch issues that self-review and extended thinking miss.
- Multi-file reviews should be split into **per-file local passes** plus a **cross-file integration pass** to avoid attention dilution and contradictory findings.
- Verification passes can ask the model to self-report **confidence per finding** to enable calibrated routing.

## Structured output with tool_use

### Why tool_use beats "respond with JSON only"

Asking the model to "respond with JSON only" works most of the time and fails just often enough to be a production hazard: stray prose, smart quotes, trailing commas, markdown fences, an apologetic preamble. `tool_use` removes that entire class of failures — the model isn't emitting free-form text, it's emitting a structured `tool_use` block whose `input` is guaranteed to be a JSON object.

Adding `"strict": true` (grammar-constrained sampling) further guarantees the JSON conforms to the schema's types, enums, and required fields. Strict mode is supported on Opus 4.7/4.6/4.5, Sonnet 4.6/4.5, and Haiku 4.5 ([Strict tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/strict-tool-use)). The newer **Structured Outputs** feature (`output_config.format = { "type": "json_schema", "schema": ... }`) delivers the same guarantee without defining a fake tool ([Structured outputs](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)); both share the same JSON Schema subset and the same semantic-error caveat.

```python
import anthropic

extract_invoice = {
    "name": "extract_invoice",
    "description": "Extract structured invoice data from the document.",
    "strict": True,
    "input_schema": {
        "type": "object",
        "additionalProperties": False,
        "required": ["vendor", "invoice_number", "line_items",
                     "stated_total", "calculated_total", "currency", "category"],
        "properties": {
            "vendor": {"type": "string"},
            "invoice_number": {"type": "string"},
            "issue_date": {"type": ["string", "null"], "format": "date"},
            "currency": {"type": "string", "enum": ["USD", "EUR", "GBP", "other"]},
            "currency_other": {"type": ["string", "null"]},
            "category": {"type": "string",
                         "enum": ["saas", "hardware", "travel",
                                  "professional_services", "other", "unclear"]},
            "category_other": {"type": ["string", "null"]},
            "line_items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "additionalProperties": False,
                    "required": ["description", "amount"],
                    "properties": {
                        "description": {"type": "string"},
                        "amount": {"type": "number"}
                    }
                }
            },
            "stated_total":     {"type": "number"},
            "calculated_total": {"type": "number"}
        }
    }
}

client = anthropic.Anthropic()
resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    tools=[extract_invoice],
    tool_choice={"type": "tool", "name": "extract_invoice"},
    messages=[{"role": "user", "content": INVOICE_TEXT}],
)
data = next(b.input for b in resp.content if b.type == "tool_use")
```

### tool_choice cheat-sheet

| `tool_choice`                          | Model behavior                                         | Use when                                                    |
| -------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------- |
| `{"type": "auto"}`                     | May respond with text *or* call any tool               | Agent loops where text answers are valid                    |
| `{"type": "any"}`                      | **Must** call a tool; picks which                      | Multi-schema extraction where the document type is unknown  |
| `{"type": "tool", "name": "extract"}`  | **Must** call that specific tool                       | Force a known extraction step before enrichment             |
| `{"type": "none"}`                     | Cannot call any tool                                   | Disable tools for a turn without rebuilding the request     |

Pair any of these with `"disable_parallel_tool_use": true` if you need at most one tool call per turn — useful when downstream code expects a single structured payload, or when concurrent tool calls would violate ordering invariants ([Parallel tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/parallel-tool-use)).

### Schema design patterns

- **Required vs optional / nullable.** Required fields force the model to fill them, causing fabrication when the source genuinely lacks the data. Mark optionally-present fields as `"type": ["string", "null"]` so the model can return `null` instead of hallucinating.
- **Enum with `"other"` + detail.** Closed enums are brittle. `category` enum `[..., "other"]` + sibling `category_other: string` preserves information for new categories without breaking downstream code.
- **`"unclear"` for ambiguity.** An `"unclear"` enum value lets the model signal low confidence; route those rows to human review.
- **Self-validating fields.** Emit both `stated_total` (from the document) and `calculated_total` (summed from `line_items`); a post-processing check catches semantic errors strict schemas cannot.

### Supported JSON Schema subset

The strict-mode / structured-outputs compiler accepts a subset of JSON Schema. Most "why does my schema 400?" failures come from this list:

- **Supported:** object/array/string/integer/number/boolean/null, `enum` (primitives only), `const`, `anyOf`/`allOf` (limited), internal `$ref`/`$def`, `default`, `required`, `additionalProperties: false`, formats (`date-time`, `date`, `email`, `uri`, `uuid`, …), `minItems` of 0 or 1.
- **Not supported:** external `$ref`, recursive schemas, complex types in enums, numeric constraints (`minimum`/`maximum`/`multipleOf`), string length constraints, `additionalProperties` != `false`.
- **Regex:** quantifiers, character classes, and groups work; backreferences, lookarounds, and word boundaries do not.

### What tool_use does NOT solve

Strict schemas guarantee *parseability*, not *correctness*. They will happily emit a schema-valid invoice in which the line items sum to $812 while `stated_total` says $1,200, or in which the vendor name appears in `invoice_number`. Catching these requires application-level validation (e.g., the `calculated_total` trick above), retry-with-error-feedback loops (Domain 4.4), or an independent reviewer pass (Domain 4.6).

## Message Batches API

### What you get / what you give up

| Dimension                  | Synchronous Messages API           | Message Batches API                                          |
| -------------------------- | ---------------------------------- | ------------------------------------------------------------ |
| Pricing                    | Standard                           | **50% off** input + output (e.g., Sonnet 4.6 $1.50/$7.50 per MTok) |
| Latency                    | Seconds                            | Up to **24 hours**; most batches finish within 1 hour; no SLA |
| Max requests / batch       | n/a                                | **100,000** requests **or 256 MB**, whichever is first       |
| Tool use                   | Full agentic loop                  | Tools may be defined; **multi-turn tool execution mid-request is not supported** |
| Streaming                  | Yes                                | **Not supported** (results are pulled when the batch ends)   |
| Prompt caching             | Yes                                | Yes (best-effort; pair with 1-hour cache for shared context) |
| Result retention           | n/a                                | 29 days                                                      |
| Models                     | All active                         | All active models                                            |

Sources: [Batch processing](https://docs.anthropic.com/en/docs/build-with-claude/message-batches).

### When to use / when not to

**Use the Batches API when:**

- The workload is non-blocking — nightly technical-debt reports, weekly audits, regression test generation, content moderation backlogs, bulk evaluations.
- Volumes are large enough that the 50% discount materially matters.
- Each request is **self-contained** (no need to inject tool results between turns).

**Do not use it when:**

- A human is waiting for the result (pre-merge checks, chat, interactive UIs).
- The workflow needs the model to call tools, see results, then continue reasoning in the same request.
- You need streaming or sub-minute latency.

This is exactly the structure of **Sample Question 11**: switch the overnight technical-debt report to batch (A), keep the blocking pre-merge check on the synchronous API. The wrong answers all involve hoping batches "usually finish fast enough" or adding a timeout fallback — neither is acceptable when the SLA is "developer is staring at the screen."

### custom_id and failure handling

Every request in a batch carries a `custom_id` (1–64 chars, `[a-zA-Z0-9_-]`). It is the **only** mechanism for correlating results to inputs, since output order is not guaranteed. Embed enough metadata in the `custom_id` to look the original record up — `doc_42891-v3-2026q1` is fine; `req_001` will haunt you.

When you retrieve results, each entry has a `result` of `succeeded`, `errored`, `canceled`, or `expired`. Standard failure pattern:

1. Pull the results stream and partition by `result.type`.
2. For `errored` entries, inspect the error code: chunk oversized documents, fix invalid params, then resubmit **only the failed `custom_id`s** as a new smaller batch.
3. For `expired` entries (batch did not finish within 24h), resubmit at a lower batch size or off-peak.

### Worked example: 100-document overnight extraction

```python
from anthropic import Anthropic
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

client = Anthropic()

requests = [
    Request(
        custom_id=f"invoice-{doc.id}",
        params=MessageCreateParamsNonStreaming(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            tools=[extract_invoice],
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=[{"role": "user", "content": doc.text}],
        ),
    )
    for doc in documents
]

batch = client.messages.batches.create(requests=requests)

while True:
    batch = client.messages.batches.retrieve(batch.id)
    if batch.processing_status == "ended":
        break
    time.sleep(60)

for entry in client.messages.batches.results(batch.id):
    if entry.result.type == "succeeded":
        msg = entry.result.message
        payload = next(b.input for b in msg.content if b.type == "tool_use")
        store(entry.custom_id, payload)
    else:
        log_failure(entry.custom_id, entry.result)
```

Note the combination: `tool_use` for schema-safe output **inside** a batch request. The single-shot extraction has no mid-request tool calls, so the Batch API constraint doesn't bite.

### SLA math: how often to submit

If the business SLA is "results within N hours" and the Batch API can take up to 24 hours, you must submit on a cadence such that **submission delay + 24h processing ≤ N**. For an N = 30h SLA, submit every **4 hours** (worst-case wait 4h before the next batch picks the record up, plus 24h processing = 28h, comfortably inside 30h). For a 26h SLA, every hour. For a 24h SLA, you cannot meet it with the Batches API alone — fall back to synchronous calls or accept missed SLAs.

## Multi-instance & multi-pass review architecture

### The self-review trap

When the same Claude instance that wrote the code also reviews it, the generation reasoning is still in context — the model treats its earlier choices as premises rather than hypotheses to challenge. Even an explicit "critique your previous response" instruction is weaker than a **fresh instance** with no prior commitment to defend.

Anthropic's own Code Review system reflects this: it dispatches **multiple specialized agents in parallel** (distinct prompts for logic, security, edge cases) and runs a **verification step** against actual code behavior to filter false positives before posting findings ([Claude Code Code Review](https://claude.com/blog/code-review)). Split the work, use independent context, reconcile at the end.

### Per-file + cross-file integration pattern

This is the correct answer to **Sample Question 12** (14-file PR with inconsistent depth and contradictory findings):

```
                    ┌────────────────────────────┐
                    │  Per-file local passes     │
PR (14 files) ──►  │  - one Claude call per file │  ──► findings_local[]
                    │  - focused prompt           │
                    │  - no other files in ctx    │
                    └────────────────────────────┘
                                  │
                                  ▼
                    ┌────────────────────────────┐
                    │  Integration pass           │
                    │  - all diffs + module map   │  ──► findings_integration[]
                    │  - cross-file data flow     │
                    │  - API contracts, types     │
                    └────────────────────────────┘
                                  │
                                  ▼
                          dedupe + rank + post
```

Per-file passes give every file the **same attention budget**, eliminating the "deep on file 1, superficial on file 14" failure mode. The integration pass is specifically prompted for cross-file concerns (caller/callee signature drift, shared schema changes, transactional invariants) so it doesn't redo what the local passes already did.

### Independent reviewer pattern

For high-stakes single artifacts (a generated migration, a customer-facing report), run **two Claude calls**:

1. **Generator** — produces the artifact with full reasoning.
2. **Reviewer** — a *new* request, *new* system prompt, *no* generator transcript, given only the artifact and the spec. Its only job is to find defects.

The reviewer's lack of context is the feature, not a bug — it cannot rationalize away decisions it never made.

### Confidence-annotated verification passes

Have the reviewer return findings with an explicit `confidence` enum (`"high"`, `"medium"`, `"low"`) and a one-line `rationale`:

```json
{
  "findings": [
    {"severity": "high", "confidence": "high",
     "file": "billing.py", "line": 142,
     "issue": "Off-by-one in proration when subscription starts on month boundary",
     "rationale": "Integration test billing_test.py:88 covers mid-month only."}
  ]
}
```

Then route by confidence: `high` confidence + `high` severity goes straight into the PR as a blocking comment; `low` confidence findings go to a triage queue or trigger a tie-breaker pass. This is the **calibrated review routing** the official skill references.

## Decision matrix: which technique for which job

| Workload                                | Latency need   | Review depth           | Recommended stack                                                       |
| --------------------------------------- | -------------- | ---------------------- | ----------------------------------------------------------------------- |
| Blocking pre-merge check                | Seconds        | Per-file + integration | Sync Messages API + `tool_use(strict)` + multi-pass review              |
| Overnight technical-debt report         | Hours          | Per-file + integration | **Batches API** + `tool_use(strict)` + multi-pass review                |
| 100k document field extraction          | Overnight      | Sample QC only         | **Batches API** + forced `tool_choice` + self-validating fields         |
| Interactive chat with extraction step   | Seconds        | None                   | Sync Messages API + forced `tool_choice` + nullable fields              |
| Regulatory document QC                  | Minutes–hours  | Independent reviewer   | Sync (or batch) + generator/reviewer split + confidence routing         |
| Weekly cross-repo audit                 | Days           | Per-repo only          | **Batches API** + `tool_use` + skip integration pass                    |

## Exam-style focus points

- **`tool_use` vs "respond with JSON":** the right answer always pushes toward `tool_use` / strict schemas for guaranteed parseability. Bare-JSON prompts are a distractor.
- **`tool_choice` selection:** `"any"` for unknown document type across multiple extraction tools; forced `{"type":"tool","name":"..."}` to guarantee a specific tool runs before enrichment; `"auto"` only when text answers are legitimate.
- **Schema design:** nullable when source may lack data (prevents fabrication); `"other"` + detail for extensibility; `"unclear"` for ambiguity; self-validating fields for semantic checks.
- **Batches API fit:** **non-blocking, latency-tolerant, ≤24h** workloads only. Pre-merge checks are the canonical wrong fit. 50% off, 100k req / 256 MB cap, results valid 29 days, no streaming, no mid-request tool execution.
- **`custom_id`:** mandatory for correlation; resubmit only failed IDs after fixing the underlying issue.
- **Multi-pass review:** per-file local + cross-file integration beats single-pass on multi-file PRs; independent instances beat self-review; confidence-annotated findings enable routing.
- **Misconceptions to avoid:** "bigger context window fixes attention dilution" (no), "three full passes + majority vote" (suppresses real bugs caught intermittently), "timeout-fallback from batch to sync" (over-complex; pick the right API per workload).

## References

- [Tool use overview](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- [Define tools](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/define-tools)
- [Strict tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/strict-tool-use)
- [Parallel tool use & `disable_parallel_tool_use`](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/parallel-tool-use)
- [Structured outputs (`output_config.format` JSON Schema)](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)
- [Message Batches API — batch processing](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Create / list / retrieve Message Batches (API reference)](https://docs.anthropic.com/en/api/creating-message-batches)
- [Prompt caching with batches (1-hour cache duration)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Claude Code: multi-agent Code Review system](https://claude.com/blog/code-review)
- [Claude Code Code Review docs](https://docs.anthropic.com/en/docs/claude-code/code-review)
