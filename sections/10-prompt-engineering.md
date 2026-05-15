---
title: "Section 10 — Prompt Engineering: Explicit Criteria, Few-Shot & Validation Loops"
linkTitle: "10. Prompt Engineering"
weight: 10
description: "Domains 4.1, 4.2, 4.4 — explicit criteria over thresholds, few-shot for ambiguous scenarios, and multi-pass review for large diffs."
---

## What this section covers

Three prompt-level techniques the Foundations exam treats as core for moving a Claude feature from demo-quality to production-quality: **explicit categorical criteria** instead of vague instructions, **2–4 targeted few-shot examples**, and a **validation-retry loop** with self-correction signals baked into the schema. Domain 4.3 (tool use + JSON schemas) is covered in Section 11; this section closes the *semantic* gap that strict schemas don't.

## Source material (from official guide)

### 4.1 Explicit criteria over vague instructions

- Explicit categorical criteria beat vague instructions: *"flag comments only when claimed behavior contradicts actual code behavior"* outperforms *"check that comments are accurate"*. General instructions like *"be conservative"* or *"only report high-confidence findings"* do **not** improve precision; high false-positive rates in any one category erode trust across **all** categories.
- Skills: write specific report-when / skip-when rules per category; temporarily disable high-FP categories while improving them; define explicit severity criteria with concrete code examples per level.

### 4.2 Few-shot prompting

- Few-shot examples are the **most effective** technique for consistent formatted, actionable output when detailed instructions alone produce inconsistent results. They drive ambiguous-case handling (tool selection, escalation), enable generalization to novel patterns, and reduce hallucination in extraction over varied document structures.
- Skills: 2–4 targeted examples that show *reasoning* for the chosen action over plausible alternatives; examples demonstrating desired output format (location, issue, severity, suggested fix); examples that distinguish acceptable patterns from genuine issues; examples covering each structural variant of the input.

### 4.4 Validation, retry & feedback loops

- Retry-with-error-feedback: append the **specific** validation error to the next prompt to guide self-correction. Retry is ineffective when the required information is **absent from the source**. Distinguish **semantic** validation errors (values don't sum, wrong field placement) from **schema syntax** errors (already eliminated by tool use).
- Skills: follow-up requests including original document + failed extraction + specific errors; add `detected_pattern` to enable false-positive analysis; design self-correction by extracting `calculated_total` alongside `stated_total`; add `conflict_detected` booleans for inconsistent source data.

## Writing explicit criteria

### Vague vs specific — before/after rewrites

The single most important move from prototype to production is replacing fuzzy instructions ("be careful", "only report high-confidence findings", "be conservative") with **categorical, behaviour-defining criteria**. Self-reported confidence is poorly calibrated: a model that is wrongly confident on hard cases will not be saved by *"only report high-confidence findings"* — it will keep reporting the same wrong things.

| Vague (what teams write first) | Specific (what production prompts look like) |
| --- | --- |
| "Check that comments are accurate." | "Flag a comment **only** when its claimed behaviour contradicts the actual code behaviour. Do **not** flag comments that are merely vague, outdated stylistically, or under-detailed." |
| "Find security issues." | "Report only: SQL string concatenation reaching a query call, `eval`/`exec` on attacker-controlled input, secrets in source, missing auth checks on routes under `/api/admin/*`. Skip: defensive `assert` patterns, hardcoded test fixtures, anything in `tests/`, `fixtures/`, `examples/`." |
| "Be conservative when escalating." | "Escalate when (a) the customer asks for a policy exception, (b) the claim value > $X, or (c) more than 2 prior contacts on the same case. Resolve autonomously when the photo evidence matches a standard damage SKU and value < $X." |
| "Only report high-confidence findings." | "Report a finding only if you can quote the exact line from the source and name the specific rule it violates. If you cannot quote a line, do not report." |

Notice the pattern: every "good" version answers two questions — *what counts as a positive?* and *what counts as a non-issue I must skip?* Sample Question 3 in the guide makes the same point in agent form: 55% first-contact resolution improves by adding **explicit escalation criteria with few-shot examples**, not by adding a self-reported confidence score.

### Severity definitions with code examples

For any classifier that emits a `severity` field, define each level with a concrete code example *inside the prompt*. Otherwise different runs collapse the distinction between `low` and `medium` into noise.

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

Anchoring each level to an example forces consistent classification across runs and across reviewers.

### Disable-then-rebuild for high-FP categories

False positives in any one category erode trust in every category. The guide recommends a deliberate disable-then-rebuild pattern: when "comment accuracy" runs at 40% FP, it's better to remove that category entirely than to leave it on and degrade trust in the security findings developers actually act on. Iterate the bad category in a side prompt until precision clears your threshold, then re-enable.

### Why "be conservative" doesn't work

Calibration. Adverbs like "carefully" or "only when sure" give the model nothing it can mechanically check. By contrast, *"do not report a finding unless you can quote the exact source line and name the rule"* is a checkable rule — the model can verify it itself before emitting output.

## Few-shot prompting deep-dive

### How many examples?

Anthropic's published guidance and the certification guide converge on the same number: **2–4 targeted examples**. Fewer than 2 doesn't establish a pattern; more than ~4 burns tokens for diminishing returns and risks the model over-fitting to surface features of the examples. For extended-thinking prompts, Anthropic specifically notes that multishot still helps but should show *reasoning patterns*, not just input→output pairs.

### Example anatomy

Effective examples carry three fields, not two:

1. **Input** — a snippet that mirrors real production input.
2. **Reasoning** — *why* the chosen action is correct over the plausible alternatives.
3. **Output** — exactly the shape (XML or JSON) you want at inference time.

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

- **Ambiguous cases.** Pick 2–4 examples from your eval set where the model previously misbehaved, and include a **contrast pair** — an example that looks similar but should be handled the **opposite** way. Contrast pairs are what teach generalization rather than memorization.
- **Format consistency.** If you want every finding shaped as `{location, issue, severity, suggested_fix}`, show three findings in exactly that shape. Instructions alone leak inconsistent capitalization, missing fields, and trailing prose.
- **False-positive reduction.** Pair an "issue" example with an "acceptable pattern" example that shares surface features. For a SQL-injection reviewer, show one vulnerable concat *and* one parameterized call labelled `not_a_finding` — this is how you stop the model from flagging every `db.query(...)`.
- **Varied document structures.** For invoices with messy line items, papers with inline vs bibliography citations, or methodology embedded in narrative, give one example *per structural variant*. Without coverage of each shape, extraction returns null on the unseen formats.

### Where to place examples — XML tagging

Anthropic's prompt engineering docs are explicit: wrap examples in XML tags (`<example>`, `<examples>`, `<input>`, `<output>`) and place them **after** task instructions and criteria but **before** the live input.

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

The pattern is: validate the model's output programmatically, and on failure send a *new* turn that contains the original input, the failed output, and the **specific** validation error. Generic "try again" never helps — concrete error text does.

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

Three things make this loop work: (1) the assistant turn is appended *verbatim* (same rule as the agentic loop in Section 2), (2) the follow-up names the **specific** failed fields, and (3) the retry uses tool use with a forced tool, so the response shape stays stable across attempts.

### When retry will NOT help

Retries fix **format and structural errors** (wrong field names, misplaced values, line items that don't sum). They do **not** fix information that is genuinely absent from the source. If the invoice PDF doesn't contain a tax ID, no retry will produce one — and an aggressive retry loop can pressure the model into hallucinating one to pass validation. Detect "absent in source" in your validator and stop the loop with a structured null + reason instead of retrying.

### Self-correction signals in the schema

Bake validation *into the extraction itself* so the model reckons with contradictions during generation:

- **`calculated_total` alongside `stated_total`** — the model emits both; your validator flags discrepancies. The model often self-corrects in the same response because computing the calculated value forces it to look at every line item.
- **`conflict_detected` booleans** — for documents where two sections disagree (header date vs footer date, summary count vs item-list length), have the model emit a boolean per conflict family. Converts ambiguity into a structured signal you can route on.

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

Add a `detected_pattern` (or `rule_id`) field to every finding. When developers dismiss findings in your UI, group dismissals by `detected_pattern` to see which categories are noisy. Without this field, your only signal is aggregate FP rate — which tells you *that* something is wrong, not *what* to fix.

### Semantic vs syntax errors

Tool use with a strict JSON schema (Domain 4.3) eliminates **syntax** errors — malformed JSON, missing required fields, wrong types. It does **not** eliminate **semantic** errors — values that don't sum, dates outside the document's range, plausible-but-wrong content. The validation-retry loop is the layer that handles semantic errors. A common exam trap: "we added a JSON schema, why are line items still wrong?" — because schemas don't do arithmetic.

## Putting it together: a precision-tuning workflow

1. **Baseline FP/FN per category.** Run an eval set; group findings by `detected_pattern`; record precision/recall per category, not just overall.
2. **Rewrite vague instructions as explicit criteria** for the top FP categories — "report when… / skip when…" rules with code examples for each side.
3. **Add 2–4 few-shot examples per ambiguous scenario,** including at least one **contrast pair** (looks like an issue, isn't) per noisy category. Wrap in `<examples>` XML tags placed after instructions, before live input.
4. **Wrap in a validation-retry loop.** Use tool use to force the schema; validate semantic constraints (`calculated_total == stated_total`, required quoted source line); on failure append the specific error and retry once.
5. **Track `detected_pattern` and dismissals.** Sort by FP rate weekly; temporarily disable any category above your threshold while you iterate.
6. **Evaluate before and after each change** with the Anthropic Console's [Evaluation tool](https://console.anthropic.com/docs/en/test-and-evaluate/eval-tool) or [Promptfoo](https://www.promptfoo.dev/docs/getting-started/).

## Exam-style focus points

- Given a prompt that says "be conservative" or "only report high-confidence findings", recognize that this is **not** the right fix for high false-positive rates — categorical criteria + few-shot are.
- Pick the right number of few-shot examples: **2–4 targeted** (not 10, not 1).
- Recognize that few-shot examples should include **reasoning** (why this action over alternatives), not just input/output, especially for ambiguous-case handling like escalation or tool selection.
- Identify when a retry will help (format/structural error, line-items-don't-sum) vs when it won't (information genuinely absent from the source). Don't retry the latter.
- Distinguish **semantic** validation errors from **schema syntax** errors. Tool use fixes the second; a validation-retry loop fixes the first.
- Know the role of self-correction fields: `calculated_total` vs `stated_total`, `conflict_detected`, `detected_pattern`.
- For Sample Question 3 (insurance agent escalation): the correct answer is **explicit escalation criteria + few-shot examples**, not self-reported confidence scores, not a separate classifier model, not sentiment analysis.

## References

- [Prompt engineering overview — Claude API docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — the canonical landing page; lists the technique stack (be clear and direct → multishot → chain-of-thought → XML tags → system prompts → prefill → chain prompts) in priority order.
- [Be clear, contextual, and specific](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/be-clear-and-direct) — Anthropic's foundational guidance behind Domain 4.1.
- [Use examples (multishot prompting)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting) — the official source for the 2–4 example recommendation, example anatomy, and XML wrapping.
- [Let Claude think (chain of thought)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/chain-of-thought) and [Use XML tags](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags) — separate reasoning from output and keep examples, instructions, and live input from bleeding into each other.
- [Extended thinking tips](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/extended-thinking-tips) — multishot still works with extended thinking; prefer high-level instructions over prescriptive step-by-step.
- [Increase output consistency (prefill)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/increase-consistency) and [Reduce hallucinations](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-hallucinations) — the techniques behind "force a starting JSON brace" and "require a quoted source line".
- [Evaluate prompts in the developer console](https://www.anthropic.com/news/evaluate-prompts) and [Using the Evaluation tool](https://console.anthropic.com/docs/en/test-and-evaluate/eval-tool) — Anthropic Workbench for test cases, side-by-side comparison, and 5-point human grading.
- [Promptfoo getting started](https://www.promptfoo.dev/docs/getting-started/) — third-party eval framework with code-graded, model-graded, and classification evals across 60+ providers.
- [Building effective agents — Anthropic Engineering](https://www.anthropic.com/engineering/building-effective-agents) — frames prompt engineering as the simpler step before reaching for ML infrastructure (the same logic behind Sample Question 3's correct answer).
