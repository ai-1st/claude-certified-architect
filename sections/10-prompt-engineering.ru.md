---
title: "Раздел 10 — Prompt Engineering: Explicit Criteria, Few-Shot и Validation Loops"
linkTitle: "10. Prompt Engineering"
weight: 10
description: "Domains 4.1, 4.2, 4.4 — explicit criteria вместо thresholds, few-shot для ambiguous scenarios и multi-pass review для large diffs."
---

## Что покрывает этот раздел

Три prompt-level техники, которые Foundations exam считает core для перевода Claude feature из demo-quality в production-quality: **explicit categorical criteria** instead of vague instructions, **2–4 targeted few-shot examples**, and **validation-retry loop** with self-correction signals baked into schema. Domain 4.3 (tool use + JSON schemas) covered in Section 11; this section closes the *semantic* gap that strict schemas don't.

## Исходный материал (из официального guide)

### 4.1 Explicit criteria over vague instructions

- Explicit categorical criteria beat vague instructions: *"flag comments only when claimed behavior contradicts actual code behavior"* outperforms *"check that comments are accurate"*. General instructions like *"be conservative"* or *"only report high-confidence findings"* do **not** improve precision; high false-positive rates in any one category erode trust across **all** categories.
- Skills: write specific report-when / skip-when rules per category; temporarily disable high-FP categories while improving them; define explicit severity criteria with concrete code examples per level.

### 4.2 Few-shot prompting

- Few-shot examples are the **most effective** technique for consistent formatted, actionable output when detailed instructions alone produce inconsistent results. They drive ambiguous-case handling (tool selection, escalation), enable generalization to novel patterns, and reduce hallucination in extraction over varied document structures.
- Skills: 2–4 targeted examples that show *reasoning* for chosen action over plausible alternatives; examples demonstrating desired output format (location, issue, severity, suggested fix); examples distinguishing acceptable patterns from genuine issues; examples covering each structural variant of input.

### 4.4 Validation, retry & feedback loops

- Retry-with-error-feedback: append the **specific** validation error to next prompt to guide self-correction. Retry is ineffective when required information is **absent from source**. Distinguish **semantic** validation errors (values don't sum, wrong field placement) from **schema syntax** errors (already eliminated by tool use).
- Skills: follow-up requests including original document + failed extraction + specific errors; add `detected_pattern` to enable false-positive analysis; design self-correction by extracting `calculated_total` alongside `stated_total`; add `conflict_detected` booleans for inconsistent source data.

## Writing explicit criteria

### Vague vs specific — before/after rewrites

Single most important move from prototype to production is replacing fuzzy instructions ("be careful", "only report high-confidence findings", "be conservative") with **categorical, behaviour-defining criteria**. Self-reported confidence is poorly calibrated: model wrongly confident on hard cases will not be saved by *"only report high-confidence findings"* — it will keep reporting same wrong things.

| Vague (what teams write first) | Specific (what production prompts look like) |
| --- | --- |
| "Check that comments are accurate." | "Flag a comment **only** when its claimed behaviour contradicts the actual code behaviour. Do **not** flag comments that are merely vague, outdated stylistically, or under-detailed." |
| "Find security issues." | "Report only: SQL string concatenation reaching a query call, `eval`/`exec` on attacker-controlled input, secrets in source, missing auth checks on routes under `/api/admin/*`. Skip: defensive `assert` patterns, hardcoded test fixtures, anything in `tests/`, `fixtures/`, `examples/`." |
| "Be conservative when escalating." | "Escalate when (a) the customer asks for a policy exception, (b) the claim value > $X, or (c) more than 2 prior contacts on the same case. Resolve autonomously when the photo evidence matches a standard damage SKU and value < $X." |
| "Only report high-confidence findings." | "Report a finding only if you can quote the exact line from the source and name the specific rule it violates. If you cannot quote a line, do not report." |

Notice pattern: every "good" version answers two questions — *what counts as a positive?* and *what counts as a non-issue I must skip?* Sample Question 3 in guide makes same point in agent form: 55% first-contact resolution improves by adding **explicit escalation criteria with few-shot examples**, not self-reported confidence score.

### Severity definitions with code examples

For any classifier that emits `severity` field, define each level with concrete code example *inside prompt*. Otherwise different runs collapse distinction between `low` and `medium` into noise.

```xml
<severity_definitions>
  <critical>Data loss, auth bypass, or RCE in prod.
    Example: db.query("SELECT * FROM users WHERE id=" + req.params.id)</critical>
  <high>Bug producing incorrect output for at least one realistic input.
    Example: off-by-one in pagination dropping the last page.</high>
  <medium>Resource leak or correctness issue only under load.
    Example: file handle opened in a loop without `with` / `defer close`.</medium>
  <low>Style or readability only — do NOT include unless explicitly asked.</low>
</severity_definitions>
```

Anchoring each level to example forces consistent classification across runs and reviewers.

### Disable-then-rebuild for high-FP categories

False positives in any one category erode trust in every category. Guide recommends deliberate disable-then-rebuild pattern: when "comment accuracy" runs at 40% FP, better remove that category entirely than leave it on and degrade trust in security findings developers actually act on. Iterate bad category in side prompt until precision clears threshold, then re-enable.

### Why "be conservative" doesn't work

Calibration. Adverbs like "carefully" or "only when sure" give model nothing mechanically checkable. By contrast, *"do not report a finding unless you can quote the exact source line and name the rule"* is checkable rule — model can verify it itself before emitting output.

## Few-shot prompting deep-dive

### How many examples?

Anthropic guidance and certification guide converge on same number: **2–4 targeted examples**. Fewer than 2 doesn't establish pattern; more than ~4 burns tokens for diminishing returns and risks over-fitting to surface features. For extended-thinking prompts, Anthropic notes multishot still helps but should show *reasoning patterns*, not just input→output pairs.

### Example anatomy

Effective examples carry three fields, not two:

1. **Input** — snippet that mirrors real production input.
2. **Reasoning** — *why* chosen action is correct over plausible alternatives.
3. **Output** — exactly shape (XML or JSON) you want at inference time.

```xml
<example>
  <input>
    User: "My order from last Tuesday hasn't arrived and I want my money back."
  </input>
  <reasoning>
    This is a delivery exception within standard policy (lost package, &lt; 30 days,
    refund value within auto-approve threshold). No policy exception is being
    requested. Resolve autonomously with refund + reship.
  </reasoning>
  <output>
    {"action": "resolve", "tool": "issue_refund", "escalate": false}
  </output>
</example>
```

### Examples for ambiguous cases, format, and FP reduction

- **Ambiguous cases.** Pick 2–4 examples from eval set where model previously misbehaved, and include a **contrast pair** — example that looks similar but should be handled the **opposite** way. Contrast pairs teach generalization rather than memorization.
- **Format consistency.** If you want every finding shaped as `{location, issue, severity, suggested_fix}`, show three findings in exactly that shape. Instructions alone leak inconsistent capitalization, missing fields, and trailing prose.
- **False-positive reduction.** Pair an "issue" example with an "acceptable pattern" example sharing surface features. For SQL-injection reviewer, show one vulnerable concat *and* one parameterized call labelled `not_a_finding` — this stops model from flagging every `db.query(...)`.
- **Varied document structures.** For invoices with messy line items, papers with inline vs bibliography citations, or methodology embedded in narrative, give one example *per structural variant*. Without coverage of each shape, extraction returns null on unseen formats.

### Where to place examples — XML tagging

Anthropic prompt engineering docs are explicit: wrap examples in XML tags (`<example>`, `<examples>`, `<input>`, `<output>`) and place them **after** task instructions and criteria but **before** live input.

```xml
<task>...explicit criteria here...</task>
<severity_definitions>...</severity_definitions>
<examples>
  <example>...</example>
  <example>...</example>
  <example>...</example>
</examples>
<input>{{actual_document_to_process}}</input>
```

## Validation, retry & feedback loops

### Retry-with-error-feedback

Pattern: validate model output programmatically, and on failure send a *new* turn containing original input, failed output, and **specific** validation error. Generic "try again" never helps — concrete error text does.

```python
from anthropic import Anthropic

client = Anthropic()
MODEL = "claude-opus-4-7"

def extract_with_retry(document: str, max_attempts: int = 2) -> dict:
    messages = [{"role": "user", "content": build_extraction_prompt(document)}]
    for attempt in range(max_attempts + 1):
        resp = client.messages.create(
            model=MODEL, max_tokens=2048,
            tools=[INVOICE_TOOL],
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=messages,
        )
        extraction = next(b.input for b in resp.content if b.type == "tool_use")

        errors = validate(extraction)
        if not errors:
            return extraction
        if not is_recoverable(errors):
            raise ExtractionError(errors, extraction)

        messages.append({"role": "assistant", "content": resp.content})
        messages.append({"role": "user", "content":
            f"Your previous extraction failed validation:\n"
            f"{format_errors(errors)}\n\n"
            f"Re-examine the original document and call extract_invoice again, "
            f"correcting only the listed errors."
        })
    raise ExtractionError(errors, extraction)
```

Three things make this loop work: (1) assistant turn appended *verbatim* (same rule as agentic loop in Section 2), (2) follow-up names **specific** failed fields, and (3) retry uses tool use with forced tool, so response shape stays stable across attempts.

### When retry will NOT help

Retries fix **format and structural errors** (wrong field names, misplaced values, line items that don't sum). They do **not** fix information genuinely absent from source. If invoice PDF doesn't contain tax ID, no retry will produce one — aggressive retry loop can pressure model into hallucinating one to pass validation. Detect "absent in source" in validator and stop loop with structured null + reason instead of retrying.

### Self-correction signals in the schema

Bake validation *into extraction itself* so model reckons with contradictions during generation:

- **`calculated_total` alongside `stated_total`** — model emits both; validator flags discrepancies. Model often self-corrects in same response because computing calculated value forces it to inspect every line item.
- **`conflict_detected` booleans** — for documents where two sections disagree (header date vs footer date, summary count vs item-list length), have model emit boolean per conflict family. Converts ambiguity into structured signal you can route on.

```json
{
  "line_items": [{"desc": "Widget A", "amount": 150.00},
                 {"desc": "Widget B", "amount": 300.00}],
  "calculated_total": 450.00,
  "stated_total": 500.00,
  "total_discrepancy": true,
  "conflict_detected": false,
  "detected_pattern": "missing_tax_line"
}
```

### `detected_pattern` for analyzing dismissals

Add `detected_pattern` (or `rule_id`) field to every finding. When developers dismiss findings in UI, group dismissals by `detected_pattern` to see which categories are noisy. Without this field, your only signal is aggregate FP rate — which tells you *that* something is wrong, not *what* to fix.

### Semantic vs syntax errors

Tool use with strict JSON schema (Domain 4.3) eliminates **syntax** errors — malformed JSON, missing required fields, wrong types. It does **not** eliminate **semantic** errors — values that don't sum, dates outside document range, plausible-but-wrong content. Validation-retry loop is layer that handles semantic errors. Common exam trap: "we added JSON schema, why are line items still wrong?" — because schemas don't do arithmetic.

## Putting it together: a precision-tuning workflow

1. **Baseline FP/FN per category.** Run eval set; group findings by `detected_pattern`; record precision/recall per category, not just overall.
2. **Rewrite vague instructions as explicit criteria** for top FP categories — "report when… / skip when…" rules with code examples for each side.
3. **Add 2–4 few-shot examples per ambiguous scenario,** including at least one **contrast pair** (looks like issue, isn't) per noisy category. Wrap in `<examples>` XML tags placed after instructions, before live input.
4. **Wrap in validation-retry loop.** Use tool use to force schema; validate semantic constraints (`calculated_total == stated_total`, required quoted source line); on failure append specific error and retry once.
5. **Track `detected_pattern` and dismissals.** Sort by FP rate weekly; temporarily disable any category above threshold while iterating.
6. **Evaluate before and after each change** with Anthropic Console's [Evaluation tool](https://console.anthropic.com/docs/en/test-and-evaluate/eval-tool) or [Promptfoo](https://www.promptfoo.dev/docs/getting-started/).

## Exam-style focus points

- Given prompt that says "be conservative" or "only report high-confidence findings", recognize this is **not** right fix for high false-positive rates — categorical criteria + few-shot are.
- Pick right number of few-shot examples: **2–4 targeted** (not 10, not 1).
- Recognize that few-shot examples should include **reasoning** (why this action over alternatives), not just input/output, especially for ambiguous-case handling like escalation or tool selection.
- Identify when retry will help (format/structural error, line-items-don't-sum) vs when it won't (information genuinely absent from source). Don't retry latter.
- Distinguish **semantic** validation errors from **schema syntax** errors. Tool use fixes second; validation-retry loop fixes first.
- Know role of self-correction fields: `calculated_total` vs `stated_total`, `conflict_detected`, `detected_pattern`.
- For Sample Question 3 (insurance agent escalation): correct answer is **explicit escalation criteria + few-shot examples**, not self-reported confidence scores, not separate classifier model, not sentiment analysis.

## References

- [Prompt engineering overview — Claude API docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — canonical landing page; lists technique stack (be clear and direct → multishot → chain-of-thought → XML tags → system prompts → prefill → chain prompts) in priority order.
- [Be clear, contextual, and specific](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/be-clear-and-direct) — Anthropic's foundational guidance behind Domain 4.1.
- [Use examples (multishot prompting)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting) — official source for 2–4 example recommendation, example anatomy, and XML wrapping.
- [Let Claude think (chain of thought)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/chain-of-thought) and [Use XML tags](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags) — separate reasoning from output and keep examples, instructions, and live input from bleeding into each other.
- [Extended thinking tips](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/extended-thinking-tips) — multishot still works with extended thinking; prefer high-level instructions over prescriptive step-by-step.
- [Increase output consistency (prefill)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/increase-consistency) and [Reduce hallucinations](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-hallucinations) — techniques behind "force a starting JSON brace" and "require a quoted source line".
- [Evaluate prompts in the developer console](https://www.anthropic.com/news/evaluate-prompts) and [Using the Evaluation tool](https://console.anthropic.com/docs/en/test-and-evaluate/eval-tool) — Anthropic Workbench for test cases, side-by-side comparison, and 5-point human grading.
- [Promptfoo getting started](https://www.promptfoo.dev/docs/getting-started/) — third-party eval framework with code-graded, model-graded, and classification evals across 60+ providers.
- [Building effective agents — Anthropic Engineering](https://www.anthropic.com/engineering/building-effective-agents) — frames prompt engineering as simpler step before reaching for ML infrastructure (same logic behind Sample Question 3's correct answer).
