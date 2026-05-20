---
title: "Раздел 11 — Structured Output через tool_use, Batch Processing и Multi-Pass Review"
linkTitle: "11. Structured Output & Batch"
weight: 11
description: "Domains 4.3, 4.5, 4.6 — JSON Schema с nullable/optional fields, validation-retry loops и tradeoffs Message Batches API."
---

## Что покрывает этот раздел

Три related architectural decisions: **как** заставить Claude emit machine-parseable output (`tool_use` + JSON schemas, `strict: true`, newer Structured Outputs feature), **когда** переносить работу на **Message Batches API** for cost and throughput (and when not to), and **почему** independent reviewer или per-file + integration split beats self-review.

Exam tests all three with concrete scenarios — pre-merge checks versus overnight reports, single-pass versus per-file review, JSON-only prompts versus `tool_use`. Right answers come from one principle: **match technique to workload's latency, reliability, and attention-budget constraints**.

## Исходный материал (из официального guide)

### 4.3 Structured output via tool_use & JSON schemas
- `tool_use` with JSON schemas is most reliable approach for guaranteed schema-compliant output; it eliminates JSON syntax errors.
- `tool_choice` modes: `"auto"` (model may return text), `"any"` (must call a tool, can pick which), or forced `{"type": "tool", "name": "..."}`.
- Strict schemas eliminate **syntax** errors only — not **semantic** errors (line items not summing to total, values in wrong fields, fabricated content for missing data).
- Schema design: required vs optional/nullable, enums with `"other"` + detail string for extensibility, `"unclear"` for ambiguity, format normalization rules in prompt.

### 4.5 Batch processing strategy
- Message Batches API: **50% cost savings**, up to **24-hour** processing window, **no latency SLA**.
- Good for non-blocking, latency-tolerant workloads (overnight reports, weekly audits, nightly test generation). Bad for blocking workflows (pre-merge checks).
- Per guide: Batch API does **not** support multi-turn *agentic* tool execution within a single request — you cannot pause mid-request to run a tool and feed results back.
- `custom_id` correlates request and response; failed `custom_id`s can be resubmitted after fixes (e.g., chunking documents that exceeded context).

### 4.6 Multi-instance & multi-pass review
- A model that retains its generation reasoning is less likely to question its own decisions, so **self-review** structurally weak.
- Independent review instances catch issues that self-review and extended thinking miss.
- Multi-file reviews should be split into **per-file local passes** plus **cross-file integration pass** to avoid attention dilution and contradictory findings.
- Verification passes can ask model to self-report **confidence per finding** to enable calibrated routing.

## Structured output with tool_use

### Why tool_use beats "respond with JSON only"

Asking model to "respond with JSON only" works most of the time and fails just often enough to be production hazard: stray prose, smart quotes, trailing commas, markdown fences, apologetic preamble. `tool_use` removes that entire class of failures — model isn't emitting free-form text, it's emitting structured `tool_use` block whose `input` is guaranteed to be JSON object.

Adding `"strict": true` (grammar-constrained sampling) further guarantees JSON conforms to schema types, enums, and required fields. Strict mode is supported on Opus 4.7/4.6/4.5, Sonnet 4.6/4.5, and Haiku 4.5 ([Strict tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/strict-tool-use)). Newer **Structured Outputs** feature (`output_config.format = { "type": "json_schema", "schema": ... }`) delivers same guarantee without defining fake tool ([Structured outputs](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs)); both share same JSON Schema subset and semantic-error caveat.

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

Pair any with `"disable_parallel_tool_use": true` if you need at most one tool call per turn — useful when downstream code expects single structured payload, or when concurrent tool calls would violate ordering invariants ([Parallel tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/parallel-tool-use)).

### Schema design patterns

- **Required vs optional / nullable.** Required fields force model to fill them, causing fabrication when source genuinely lacks data. Mark optionally-present fields as `"type": ["string", "null"]` so model can return `null` instead of hallucinating.
- **Enum with `"other"` + detail.** Closed enums are brittle. `category` enum `[..., "other"]` + sibling `category_other: string` preserves information for new categories without breaking downstream code.
- **`"unclear"` for ambiguity.** `"unclear"` enum value lets model signal low confidence; route those rows to human review.
- **Self-validating fields.** Emit both `stated_total` (from document) and `calculated_total` (summed from `line_items`); post-processing check catches semantic errors strict schemas cannot.

### Supported JSON Schema subset

Strict-mode / structured-outputs compiler accepts subset of JSON Schema. Most "why does my schema 400?" failures come from this list:

- **Supported:** object/array/string/integer/number/boolean/null, `enum` (primitives only), `const`, `anyOf`/`allOf` (limited), internal `$ref`/`$def`, `default`, `required`, `additionalProperties: false`, formats (`date-time`, `date`, `email`, `uri`, `uuid`, …), `minItems` of 0 or 1.
- **Not supported:** external `$ref`, recursive schemas, complex types in enums, numeric constraints (`minimum`/`maximum`/`multipleOf`), string length constraints, `additionalProperties` != `false`.
- **Regex:** quantifiers, character classes, and groups work; backreferences, lookarounds, and word boundaries do not.

### What tool_use does NOT solve

Strict schemas guarantee *parseability*, not *correctness*. They will happily emit schema-valid invoice where line items sum to $812 while `stated_total` says $1,200, or where vendor name appears in `invoice_number`. Catching these requires application-level validation (e.g., `calculated_total` trick above), retry-with-error-feedback loops (Domain 4.4), or independent reviewer pass (Domain 4.6).

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

- Workload is non-blocking — nightly technical-debt reports, weekly audits, regression test generation, content moderation backlogs, bulk evaluations.
- Volumes are large enough that 50% discount materially matters.
- Each request is **self-contained** (no need to inject tool results between turns).

**Do not use it when:**

- Human is waiting for result (pre-merge checks, chat, interactive UIs).
- Workflow needs model to call tools, see results, then continue reasoning in same request.
- You need streaming or sub-minute latency.

This is exactly structure of **Sample Question 11**: switch overnight technical-debt report to batch (A), keep blocking pre-merge check on synchronous API. Wrong answers all involve hoping batches "usually finish fast enough" or adding timeout fallback — neither acceptable when SLA is "developer is staring at the screen."

### custom_id and failure handling

Every request in batch carries `custom_id` (1–64 chars, `[a-zA-Z0-9_-]`). It is **only** mechanism for correlating results to inputs, since output order is not guaranteed. Embed enough metadata in `custom_id` to look original record up — `doc_42891-v3-2026q1` is fine; `req_001` will haunt you.

When retrieving results, each entry has `result` of `succeeded`, `errored`, `canceled`, or `expired`. Standard failure pattern:

1. Pull results stream and partition by `result.type`.
2. For `errored` entries, inspect error code: chunk oversized documents, fix invalid params, then resubmit **only failed `custom_id`s** as new smaller batch.
3. For `expired` entries (batch did not finish within 24h), resubmit at lower batch size or off-peak.

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

Note combination: `tool_use` for schema-safe output **inside** batch request. Single-shot extraction has no mid-request tool calls, so Batch API constraint doesn't bite.

### SLA math: how often to submit

If business SLA is "results within N hours" and Batch API can take up to 24 hours, submit on cadence such that **submission delay + 24h processing ≤ N**. For N = 30h SLA, submit every **4 hours** (worst-case wait 4h before next batch picks record up, plus 24h processing = 28h, comfortably inside 30h). For 26h SLA, every hour. For 24h SLA, you cannot meet it with Batches API alone — fall back to synchronous calls or accept missed SLAs.

## Multi-instance & multi-pass review architecture

### The self-review trap

When same Claude instance that wrote code also reviews it, generation reasoning is still in context — model treats earlier choices as premises rather than hypotheses to challenge. Even explicit "critique your previous response" instruction is weaker than **fresh instance** with no prior commitment to defend.

Anthropic's own Code Review system reflects this: it dispatches **multiple specialized agents in parallel** (distinct prompts for logic, security, edge cases) and runs **verification step** against actual code behavior to filter false positives before posting findings ([Claude Code Code Review](https://claude.com/blog/code-review)). Split work, use independent context, reconcile at end.

### Per-file + cross-file integration pattern

This is correct answer to **Sample Question 12** (14-file PR with inconsistent depth and contradictory findings):

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

Per-file passes give every file **same attention budget**, eliminating "deep on file 1, superficial on file 14" failure mode. Integration pass is specifically prompted for cross-file concerns (caller/callee signature drift, shared schema changes, transactional invariants) so it doesn't redo what local passes already did.

### Independent reviewer pattern

For high-stakes single artifacts (generated migration, customer-facing report), run **two Claude calls**:

1. **Generator** — produces artifact with full reasoning.
2. **Reviewer** — *new* request, *new* system prompt, *no* generator transcript, given only artifact and spec. Its only job is to find defects.

Reviewer's lack of context is feature, not bug — it cannot rationalize away decisions it never made.

### Confidence-annotated verification passes

Have reviewer return findings with explicit `confidence` enum (`"high"`, `"medium"`, `"low"`) and one-line `rationale`:

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

Then route by confidence: `high` confidence + `high` severity goes straight into PR as blocking comment; `low` confidence findings go to triage queue or trigger tie-breaker pass. This is **calibrated review routing** official skill references.

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

- **`tool_use` vs "respond with JSON":** right answer always pushes toward `tool_use` / strict schemas for guaranteed parseability. Bare-JSON prompts are distractor.
- **`tool_choice` selection:** `"any"` for unknown document type across multiple extraction tools; forced `{"type":"tool","name":"..."}` to guarantee specific tool runs before enrichment; `"auto"` only when text answers legitimate.
- **Schema design:** nullable when source may lack data (prevents fabrication); `"other"` + detail for extensibility; `"unclear"` for ambiguity; self-validating fields for semantic checks.
- **Batches API fit:** **non-blocking, latency-tolerant, ≤24h** workloads only. Pre-merge checks are canonical wrong fit. 50% off, 100k req / 256 MB cap, results valid 29 days, no streaming, no mid-request tool execution.
- **`custom_id`:** mandatory for correlation; resubmit only failed IDs after fixing underlying issue.
- **Multi-pass review:** per-file local + cross-file integration beats single-pass on multi-file PRs; independent instances beat self-review; confidence-annotated findings enable routing.
- **Misconceptions to avoid:** "bigger context window fixes attention dilution" (no), "three full passes + majority vote" (suppresses real bugs caught intermittently), "timeout-fallback from batch to sync" (over-complex; pick right API per workload).

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
