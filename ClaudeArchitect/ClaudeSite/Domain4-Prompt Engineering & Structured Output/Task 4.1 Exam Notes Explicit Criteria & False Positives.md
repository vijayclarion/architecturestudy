# Task 4.1 Exam Notes: Explicit Criteria & False Positives

Oct 26, 2026 · @VIjay

One-line summary: replace vague adjectives with named categories and code examples, protect trust by disabling noisy categories, and never let self-reported confidence be your main filter.

## Key terms

| Term | Meaning | Example |
| --- | --- | --- |
| False positive | A finding the model reports that is not actually a problem | Flagging a "missing null check" when the value is checked two lines earlier |
| Precision | Of everything reported, the share that is correct | 49 correct out of 50 reported = 98% precision |
| Vague instruction | An instruction with no actionable decision boundary | "Be conservative", "Only report high-confidence findings", "Use your best judgement" |
| Decision boundary | A clear rule that tells the model, per finding, whether it counts | "Report SQL injection risks; skip variable naming" |
| Explicit categorical criteria | Named categories to flag and named categories to skip | Flag: bugs, security vulnerabilities. Skip: style, local patterns |
| Trust destruction | Errors in one category make developers distrust all categories | Wrong documentation findings lead people to ignore accurate security findings |
| Disable and iterate | Temporarily switch off a noisy category, fix its prompt, then switch it back on | Turn off "documentation mismatch" until its precision improves |
| Severity calibration | Defining severity levels so they are applied consistently | Each level anchored to a code snippet |
| Calibration (of confidence) | Whether "X% confident" really means right X% of the time | Model says 92 but is wrong; says 55 and is right = poorly calibrated |
| Confidence-based routing | Using confidence as a secondary signal to decide where a finding goes, not to delete it | Low-confidence findings sent for extra human review |

## 1. Explicit categorical criteria vs vague instructions

**Rule:** Tell the model exactly what to flag and what to skip. Adjectives like "conservative" give it nothing to apply to a specific line of code.

**Why vague instructions fail**

- "Conservative" and "high-confidence" have no defined meaning, so the model falls back on its own general interpretation.
- That interpretation drifts between reviews, so results stay inconsistent.
- You get fewer findings, not fewer *wrong* findings.

**Wrong (vague):**

```
Review this code. Be conservative and only report high-confidence findings.
Use your best judgement.
```

**Right (explicit categories):**

```
Report ONLY these categories:
- Bugs: logic errors, off-by-one errors, null dereferences, unhandled exceptions
- Security vulnerabilities: unsanitised user input in SQL or shell commands,
  hard-coded secrets, missing authentication checks

Do NOT report:
- Style preferences: naming, brace placement, line length
- Local patterns: conventions used consistently elsewhere in this codebase
```

**Exam memory hook:** The model should be able to ask "Which category is this?", not "How cautious should I be?"

## 2. False positives destroy trust across ALL categories

**Rule:** Trust is not category-specific. A high false positive rate in one category makes developers distrust the whole tool.

**Key points**

- This is about **people**, not the model. Developers don't keep a separate trust score per category.
- After seeing repeated wrong findings, they think "this tool is often wrong" and start skimming or ignoring everything.
- Your most accurate, most valuable findings (usually security) get ignored along with the noise.

**Example**

| Category | Precision | What developers do |
| --- | --- | --- |
| Documentation mismatch | 60% (wrong 40% of the time) | Learn to ignore the tool |
| Security vulnerability | 98% | Also ignored, even though it is almost always right |

**Common wrong answer:** "Developers can just ignore the noisy category and keep trusting the good one." They don't. Trust bleeds across the entire output.

**Exam memory hook:** One noisy category can make your best category worthless in practice.

## 3. Temporarily disable, iterate, re-enable

**Rule:** When a category has a high false positive rate, switch it off **now**, fix its prompt, and switch it back on only once precision improves.

**The three steps**

1. **Disable** the noisy category immediately, so it stops damaging trust.
2. **Improve** its prompt, ideally with concrete code examples of real issues vs acceptable code.
3. **Re-enable** it once precision has improved.

**Principle:** System-wide trust comes before category completeness.

**Example:** Documentation mismatch findings are wrong 40% of the time. Remove that category from the system prompt this week. Add code examples showing a genuine mismatch (a docstring that says the function returns a list when it returns a dict) and a non-issue (a docstring that is brief but accurate). Test, then bring it back.

**Options compared**

| Option | Verdict | Why |
| --- | --- | --- |
| Keep it running to collect more error data | Wrong (anti-pattern) | Developers keep seeing bad findings and trust keeps draining |
| Remove it permanently | Wrong | You lose a useful category for good; the fix is meant to be temporary |
| Fix the prompt while it stays live | Incomplete | Right improvement step, but trust damage continues meanwhile |
| Disable, improve with code examples, re-enable | Correct | Protects trust while the category is fixed |

**Exam memory hook:** Losing one category for a few weeks costs less than developers ignoring your security findings.

## 4. Severity calibration with CODE examples

**Rule:** Define each severity level with an actual code snippet, not a prose description.

**Why prose fails:** Phrases like "could cause system failures" or "significantly affect functionality" are as vague as "be conservative". The model must interpret them, so the same issue lands at Critical one day and Medium the next.

**Wrong (prose only):**

```
Critical: issues that could cause system failures
High: issues that significantly affect functionality
Medium: issues that may cause problems under some conditions
```

**Also not enough (prose example):** "Critical, e.g. a SQL injection risk." Better than nothing, but still open to interpretation.

**Right (code anchors each level):**

```python
# CRITICAL: unsanitised user input in a SQL query
query = f"SELECT * FROM users WHERE id = '{request.args['id']}'"
cursor.execute(query)

# HIGH: resource never closed, leaks under load
f = open(path)
data = f.read()   # no close(), no 'with' block

# MEDIUM: off-by-one skips the last item in an edge case
for i in range(len(items) - 1):
    process(items[i])
```

**Exam memory hook:** "Code examples, not prose." A snippet removes the ambiguity entirely.

## 5. Why confidence-based filtering fails

**Rule:** LLM self-reported confidence is poorly calibrated, so it cannot be your primary filter. Explicit criteria first, confidence-based routing second.

**What "poorly calibrated" means:** A well-calibrated forecaster who says "80% chance of rain" is right on about 80 of 100 such days. The model's score is just more generated text, not a measured probability. It can be very confident about wrong findings and unsure about correct ones.

**Example: filtering at "confidence above 80"**

| Finding | Model's confidence | Actually correct? | Result of the filter |
| --- | --- | --- | --- |
| "Missing null check on line 42" | 92 | No, checked two lines earlier | Kept (false positive gets through) |
| "Possible race condition in cache update" | 55 | Yes, a real bug | Dropped (real bug lost) |

The filter does the opposite of what you wanted.

**The hierarchy**

1. **Explicit categorical criteria first:** decide what is in scope (flag bugs and security, skip style). This does the precision work.
2. **Confidence-based routing second:** a secondary signal for *where* an in-scope finding goes (for example, lower-confidence findings sent for extra human review). Route, don't delete.

**Exam memory hook:** Criteria decide *what* counts; confidence only helps decide *where it goes*.

## Anti-patterns to spot in exam questions

| Anti-pattern | Example of the wrong approach | Correct approach |
| --- | --- | --- |
| Vague instructions | "Be conservative and only report high-confidence findings." | "Flag bugs and security vulnerabilities. Skip style and local patterns." |
| Confidence-based filtering as the main control | "Only show findings with confidence above 80." | Categorical criteria first; confidence only for routing (e.g. to human review) |
| Prose-only severity levels | "Critical: issues that could cause system failures." | A code snippet per level, e.g. unsanitised SQL input = Critical |
| Keeping a noisy category live while iterating | "Leave documentation checks on so we gather more error data." | Disable it now, improve it with code examples, re-enable later |

## Golden rules and exam traps

**Five golden rules**

1. Name categories to flag and skip; never rely on adjectives.
2. Trust bleeds across categories; one noisy category hurts them all.
3. Disable noisy categories temporarily; fix with code examples; re-enable.
4. Severity = code examples, not prose.
5. Explicit criteria first, confidence-based routing second.

**Exam traps (answers that sound right but are wrong)**

| Trap answer | Why it is wrong |
| --- | --- |
| "Add more context and constraints to the prompt" | Too generic. The tested idea is explicit flag and skip categories. |
| "Developers will just ignore the bad category" | Trust is not category-specific; they ignore everything. |
| "Keep the category running to collect data" | Trust keeps draining while you iterate. |
| "Remove the noisy category permanently" | The fix is temporary; re-enable once precision improves. |
| "Give examples for each severity level" (prose) | They must be code examples. |
| "Filter out findings below a confidence threshold" | Confidence is poorly calibrated; route, don't delete. |
| "Confidence fails because of training data" | The precise reason is poor calibration: confident when wrong, unsure when right. |
