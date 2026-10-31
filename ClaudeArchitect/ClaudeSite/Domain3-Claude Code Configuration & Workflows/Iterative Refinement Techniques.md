# Task 3.5 Exam Notes: Iterative Refinement Techniques

Oct 26, 2026 · @VIjay

## Quick decision guide

Match the problem to the technique. Most exam questions in Task 3.5 test exactly this mapping.

| The problem you see | The technique to use | Remember it as |
| --- | --- | --- |
| Claude Code interprets a transformation differently each time | Concrete input/output examples (2-3 before/after pairs) | Inconsistent → show, don't tell |
| A complex transformation with many behaviours to get right | Test-driven iteration (write tests first) | Complex → tests first |
| An unfamiliar domain where you may be missing requirements | Interview pattern (Claude asks questions first) | Unfamiliar → let Claude ask |
| Several issues where fixing one changes another | Batch them in a single message | Interacting → together |
| Several issues that don't affect each other | Fix them sequentially, one at a time | Independent → one by one |

## Key terms

| Term | Meaning | Example |
| --- | --- | --- |
| Iterative refinement | Improving Claude Code's output over several rounds of feedback | First draft of a parser, then fixes driven by examples or failing tests |
| Transformation | Turning an input into a specific output | `03/14/2025` → `2025-03-14` |
| Inconsistent interpretation | The same prose instruction gives different results on different runs | "Normalise the names" gives `smith, john` once and `John Smith` the next time |
| Before/after pair | One exact input with its exact expected output | `john SMITH` → `John Smith` |
| Generalisation | The model applies the rule shown in examples to new, unseen inputs | After three name examples, it also converts `mary-ann o'brien` → `Mary-Ann O'Brien` correctly |
| Happy path | The normal, expected input | A valid email address passed to a validator |
| Edge case | An unusual but valid input at the limits | An empty string, a 255-character name, a leap-year date |
| Error case | Invalid input that should be rejected or handled | `"abc"` passed where a number is expected |
| Test failure feedback | A failing test's message shared with Claude Code | `Expected "2024-02-29", got "2024-03-01"` |
| Interview pattern | Claude asks you about requirements, edge cases and constraints before implementing | "Should refunds be partial or full? How should currencies be rounded?" |
| Interacting (interdependent) issues | Fixing one changes what the correct fix for another is | Error response shape and the logging format that records its fields |
| Independent issues | Fixing one has no effect on the others | Variable naming convention vs error response shape |
| Batch feedback | All issues sent together in a single message | One message covering both the error shape and the logging format |
| Sequential feedback | Issues sent one at a time, each fixed and checked before the next | Fix naming first, confirm it, then move on |

## 1. Concrete input/output examples

This is the first-line fix when Claude Code interprets a transformation inconsistently. Show the result rather than describing it.

**Rules to remember**

- Use **2-3** well-chosen before/after pairs. That is sufficient.
- The model **generalises from examples more reliably than from prose**.
- Examples beat rewriting the prose with more precise language.
- They work for any transformation, not only formatting.
- Make the examples **varied**. Near-identical examples don't show which features matter.

**Workflow**

1. Observe the inconsistency.
2. Provide 2-3 before/after pairs.
3. Verify that the model generalises to new inputs.
4. Add edge-case examples only if it still gets some wrong.

**Example**

Prose that keeps failing: *"Convert the dates to a standard format."* One run gives `14/03/2025`, another gives `March 14, 2025`.

Switch to examples instead:

```
03/14/2025   →  2025-03-14
1 Jan 2024   →  2024-01-01
2023.12.05   →  2023-12-05
```

If it then mishandles two-digit years, add one edge case: `5/6/99 → 1999-05-06`.

## 2. Test-driven iteration

Use this for complex transformations. Write the tests first, then share the failures with Claude Code.

**Rules to remember**

- Write the tests **before** the implementation.
- Cover three categories: **happy path**, **edge cases** and **error cases**.
- Share the test failures with Claude Code and iterate until they pass.
- Failures are concrete and unambiguous: **"Expected X, got Y"** leaves no room for interpretation.
- Your tests define what "correct" means. Claude Code doesn't decide it.

**Examples vs tests**

- Examples: a transformation read inconsistently, where a few pairs are enough.
- Tests: a complex transformation with many behaviours, where you need a full, checkable specification.

**Example**

Task: a function that parses currency strings into numbers.

```
# Happy path
parse_amount("£1,250.50") == 1250.50
# Edge cases
parse_amount("£0.00") == 0.0
parse_amount("₹1,00,000") == 100000.0
# Error case
parse_amount("twelve pounds") raises ValueError
```

Feedback you share: `test_indian_grouping FAILED: expected 100000.0, got 1.0`. Claude Code knows exactly what to fix.

## 3. Interview pattern

Use this in unfamiliar domains. Have Claude ask questions before it implements anything.

**Rules to remember**

- Don't prescribe a solution. Let Claude ask about **requirements, edge cases and constraints** first.
- It **surfaces considerations you would miss**, because you don't know what you don't know.
- It is for **unfamiliar domains**, NOT for inconsistent interpretation.
- Watch the wording: "ambiguous output" or "inconsistent results" points to examples, not an interview.

**Example**

You've never built a payment integration. Instead of saying *"Add Stripe refunds"*, you say:

> "Before implementing refunds, ask me any questions you need about requirements, edge cases and constraints."

Claude might ask:

- Should refunds be full only, or partial too?
- What happens if a refund is requested twice (idempotency)?
- How should currency rounding be handled?
- Is there a time limit on refunds?

These are considerations you might never have thought to specify.

## 4. Batch vs sequential feedback

The deciding factor is whether the issues **interact**. It is not about complexity, and not about how many files a fix touches.

**Rules to remember**

- **Batch** (single message) when fixes interact: changing one affects the others. The model needs to see all the interacting constraints at once.
- **Sequential** when issues are independent: fixing one doesn't affect the others. Batching independent issues can confuse the model.
- One fix applied across many files is still **one** issue.

**The test question:** *"If Claude fixes issue A, does that change the right fix for issue B?"* Yes means batch. No means sequential.

**Example: batch**

The API's date field changes from `"14/03/2025"` to ISO format, and the frontend parser that reads that field is also wrong. Fixing the API changes what the parser must handle. Send both in one message.

**Example: sequential**

A typo in a help message, and a missing index on a database table. Neither affects the other. Fix the typo, confirm it, then send the index fix.

**Example: mixed (the classic exam scenario)**

Three issues: (1) error response shape, (2) logging format that records fields from the error response, (3) inconsistent variable naming.

- Message A: issues 1 and 2 together, because they interact.
- Message B: issue 3 on its own, before or after, because it is independent.

Answer: **{1 + 2} batched, 3 separately.**

## Anti-patterns

Exam options often describe one of these as a plausible answer. Each is wrong.

| Anti-pattern | Example of the mistake | Do this instead |
| --- | --- | --- |
| Repeating or rewriting prose when interpretation is inconsistent | Rewording "normalise the names" five times, getting a different result each time | Give 2-3 before/after pairs |
| Not writing tests first for a complex transformation | Asking for a CSV-to-JSON converter with nested fields and fixing bugs by description | Write happy-path, edge and error tests first, then share failures |
| Batching independent issues | One message fixing a README typo, a slow query and a colour in the UI | Send each one separately |
| Sequential feedback on interdependent issues | Fixing the error shape, then separately fixing logging against the old shape | Send both in one message |
| Prescribing solutions in an unfamiliar domain | "Build the HL7 lab-results importer like this" with no domain knowledge | Ask Claude to interview you first |

## Exam traps

Three traps account for most wrong answers on this task statement.

**Trap 1: more detailed prose**

- Question: *Claude Code formats phone numbers differently on each run. What should the developer do first?*
- Trap answer: rewrite the instruction with more precise language.
- Correct answer: provide 2-3 concrete before/after examples.

**Trap 2: batch vs sequential**

- Question: *You find that the database schema is wrong and the migration script depends on it, plus a typo in a log message. How do you give feedback?*
- Trap answer: everything in one message, or everything one at a time.
- Correct answer: schema and migration together in one message, and the typo separately.

**Trap 3: interview pattern vs examples**

- Question: *A developer new to tax law must build a VAT calculation module and is worried about missing rules. Which technique?*
- Trap answer: concrete examples.
- Correct answer: the interview pattern, because it is an unfamiliar domain.
- The reverse also appears: inconsistent output in a familiar task calls for examples, not an interview.

**Keyword cues**

| If the question says… | Think… |
| --- | --- |
| "differently each time", "inconsistent", "varies between runs" | Concrete examples |
| "complex transformation", "many edge cases" | Tests first |
| "unfamiliar", "new to the domain", "might miss requirements" | Interview pattern |
| "depends on", "affects", "changing one changes the other" | Batch |
| "unrelated", "independent", "separate concern" | Sequential |

## Last-minute checklist

Tick these off when you can recall each one without looking.

- [ ] Inconsistent interpretation → concrete examples, not more prose
- [ ] 2-3 well-chosen, varied examples are enough
- [ ] Examples workflow: observe → 2-3 pairs → verify generalisation → add edge cases if needed
- [ ] Complex transformation → write tests first (happy path, edge, error)
- [ ] Test failures give "Expected X, got Y" feedback with no room for interpretation
- [ ] Unfamiliar domain → interview pattern (Claude asks first)
- [ ] Interview pattern and examples solve different problems
- [ ] Interacting issues → batch in one message
- [ ] Independent issues → sequential, one at a time
- [ ] Deciding test: does fixing A change the right fix for B?
- [ ] Classic scenario: error shape + logging batched, naming separate
