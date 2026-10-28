# Exam Notes: Validation, Retry & Feedback Loops (Task 4.4)

Oct 27, 2026 · @VIjay

## Purpose and cheat sheet

Task 4.4 tests whether you can make extraction pipelines catch and fix their own mistakes: validate the output, retry only when a retry can help, and feed back exactly what went wrong.

The five ideas in one line each:

1. **Retry with error feedback:** send the document, the failed extraction and the specific error.
2. **Retry boundary:** retry fixable mistakes; never retry for data the source doesn't contain.
3. **Self-correction fields:** `calculated_total` + `stated_total`, and `conflict_detected`, make errors visible in the output itself.
4. **`detected_pattern`:** record what triggered each finding so you can analyse dismissals and fix the prompt.
5. **Syntax vs semantic errors:** `tool_use` fixes structure; validation logic and retries fix wrong values.

Memory hook: **structure is free with tool\_use; correctness is your job.**

## 1. Retry-with-error-feedback pattern

A useful retry tells the model exactly what failed; a naive retry (same prompt, same document) usually reproduces the same mistake.

**Rule: send back three things.**

1. The **original document**, so the model can re-read the source.
2. The **failed extraction**, so it sees what it produced.
3. The **specific validation error**, so it knows what to fix.

**Why it works:** the error message turns a blind guess into a targeted correction. Without it, the model has no guidance.

**Example.** Validation finds the line items sum to £4,250.00 but `total_amount` is £4,520.00.

- Naive retry (weak): resend the original prompt and PDF.
- Error-feedback retry (strong):

```
<document> [original invoice PDF] </document>
<previous_extraction> {"line_items": [...], "total_amount": 4520.00} </previous_extraction>
<validation_error>
Line items sum to 4250.00 but total_amount is 4520.00.
Re-examine the line items and the stated total. Check for missed items.
</validation_error>
```

**Exam trap:** an option that "refines the prompt" or "adds more instructions" without the specific error is still a blind retry.

## 2. Retry effectiveness boundary

Retry only when the correct answer exists in the source and the model got it wrong; if the data is genuinely absent, no retry can fix it.

**Decision test:** "Is the correct value actually in the document?" Yes means retry with feedback. No means accept null or route for human review.

| Situation | Example | Retry helps? |
| --- | --- | --- |
| Format mismatch | Date returned as "3rd March" when the schema expects YYYY-MM-DD | Yes |
| Structural output error | Line items nested under the wrong key | Yes |
| Misplaced value | Supplier phone number put in `vat_number` | Yes |
| Maths error | Missed line item, so the sum is wrong | Yes |
| Information absent from source | `po_number` null; invoice has no PO number anywhere | **No** |

**Why retrying absent data is harmful, not just wasteful:** pressing the model with "this field must not be null" can push it to **fabricate** a plausible value, such as an invented VAT number.

**Correct handling of absent data:** keep the null. If the field is business-critical, flag the document for human review.

**Exam trap:** the exam shows both kinds of failure side by side. Check whether the value exists in the source before choosing "retry".

## 3. Self-correction schema fields

Add fields that make the model expose its own discrepancies, so semantic validation works on the output alone without external logic.

**Rule A: extract both totals, not a difference.**

- `stated_total`: the total as printed on the document.
- `calculated_total`: the sum of the extracted line items.
- If they differ, the discrepancy flags itself, and the two numbers become the retry error message.
- A single `difference` field is weaker: `0` can't prove the model checked, and a non-zero value loses the numbers you need to diagnose.

```json
{
  "line_items": [{"desc": "Licences", "amount": 3000.00}, {"desc": "Support", "amount": 1250.00}],
  "stated_total": 4520.00,
  "calculated_total": 4250.00
}
```

Here `calculated_total` is lower, which suggests a missed line item: a retry-fixable maths error.

**Rule B: add a `conflict_detected` boolean for contradictory source data.**

- With only one `payment_terms` field, the model is forced to **silently pick one** value, hiding the contradiction.
- A boolean is machine-checkable; free text needs parsing. Use free text only as supporting detail.

Example: Clause 4.2 says "payment due: 30 days"; Clause 9.1 says "net 60" (due within 60 days of the invoice date).

```json
{
  "payment_terms": "30 days",
  "conflict_detected": true,
  "conflict_details": "Clause 4.2 states 30 days; Clause 9.1 states net 60"
}
```

**Handling:** a contradiction in the source is not a model error, so route it for human review rather than retrying.

## 4. detected\_pattern fields

Record which code construct or document feature triggered each finding, so you can see which patterns get dismissed and refine the prompt for those specifically.

**Rule:** every finding carries a `detected_pattern` describing *what in the input* caused it.

**Why not log the prompt instead?** The prompt is identical for every finding, so it can't explain why some are dismissed and others aren't. What varies is the trigger.

**Example finding (code review tool):**

```json
{
  "finding": "Possible SQL injection",
  "detected_pattern": "f-string inside cursor.execute()",
  "severity": "high"
}
```

**Dismissal analysis after a month:**

| detected\_pattern | Findings | Dismissed |
| --- | --- | --- |
| string concatenation in ORM .filter() | 80 | 85% |
| f-string inside cursor.execute() | 50 | 4% |

The ORM pattern is mostly false positives (the ORM already parameterises queries), so refine the prompt for that pattern only. The f-string pattern is working; leave it alone.

**Rule of thumb:** when developers consistently dismiss a specific pattern, that pattern needs prompt refinement.

**Document example:** for an extraction, `detected_pattern` could be "total taken from footer summary box" or "date parsed from email header".

## 5. Schema syntax errors vs semantic validation errors

`tool_use` eliminates syntax errors but not semantic ones, so you still need validation logic and retry loops.

|  | Syntax errors | Semantic errors |
| --- | --- | --- |
| What it is | Output has the wrong shape | Shape is right, content is wrong |
| Examples | Malformed JSON; **missing required field**; string where a number is expected | Wrong sum; value in the wrong field; fabricated data; due date before invoice date |
| Fixed by tool\_use? | Yes | No |
| How to catch | Schema / parsing | Custom validation logic |
| How to fix | Prevented up front by tool\_use | Retry with error feedback (if the data is in the source) |

**Quick test:** "Would a schema reject this?" If not, it's semantic.

**Pydantic: the standard Python validation layer.**

- **Parsing** enforces the schema: types and required fields.
- **Custom validators** enforce rules a JSON schema can't express, such as cross-field sums and date ordering.
- **`ValidationError`** yields per-field messages, which you quote in the retry prompt.

```python
from datetime import date
from pydantic import BaseModel, model_validator

class Invoice(BaseModel):
    invoice_date: date
    due_date: date
    line_items: list[float]
    stated_total: float

    @model_validator(mode="after")
    def check_semantics(self):
        if abs(sum(self.line_items) - self.stated_total) > 0.01:
            raise ValueError(
                f"Line items sum to {sum(self.line_items):.2f} "
                f"but stated_total is {self.stated_total:.2f}")
        if self.due_date < self.invoice_date:
            raise ValueError("due_date is earlier than invoice_date")
        return self
```

The `ValidationError` text becomes item 3 of the retry-with-error-feedback pattern.

**Worked classification:**

1. Due date earlier than invoice date, correct dates visible in the PDF: semantic, retry fixes it.
2. Phone number in `vat_number`: semantic (misplaced value), retry fixes it.
3. `po_number` null, none in the invoice: not an error; don't retry.

## Anti-patterns, glossary and exam traps

The exam rewards spotting these four anti-patterns in answer options.

| Anti-pattern | Example | Correct approach |
| --- | --- | --- |
| Retrying without the specific error | Resending the same prompt after a sum mismatch | Send document + failed extraction + specific error |
| Retrying for absent information | Retrying until `po_number` is filled when the invoice has none | Accept null; human review if critical |
| Omitting `detected_pattern` | Findings logged with no trigger, so 40% dismissals are unexplained | Record the triggering construct per finding |
| Schema validation only | "We use tool\_use, so no validation needed" | Add semantic validators (e.g. Pydantic) + retry loop |

**Glossary**

- **Naive retry:** resending the same request with no new information.
- **Validation error:** the specific, quotable reason an extraction failed a check.
- **`stated_total` / `calculated_total`:** printed total vs sum of extracted items.
- **`conflict_detected`:** boolean set when the source contradicts itself.
- **`detected_pattern`:** the input feature that triggered a finding.
- **Syntax error:** wrong shape (malformed JSON, missing required field, wrong type).
- **Semantic error:** right shape, wrong content (bad sum, misplaced or fabricated value).
- **Fabrication:** the model inventing a plausible value that isn't in the source.

**Traps to remember**

- Missing required fields are **syntax** errors, which tool\_use eliminates.
- Null is the correct answer when the source lacks the value.
- A contradictory source goes to human review, not a retry.
- "Refine the prompt" without the error message is still a blind retry.
