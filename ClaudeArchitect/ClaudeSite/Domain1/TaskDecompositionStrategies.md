Here's a revision sheet for **Task Statement 1.6: Task Decomposition Strategies**.

## 1. Key terms

| Term | Meaning |
|---|---|
| **Task decomposition** | Breaking a large task into smaller subtasks an agent can handle |
| **Fixed sequential pipeline (prompt chaining)** | Predetermined steps run in order; each step takes the previous step's output as its input |
| **Dynamic adaptive decomposition** | Subtasks are generated from what is discovered at each step; the plan evolves as the agent learns |
| **Attention dilution** | Processing too many items in a single pass, which produces inconsistent depth of analysis |
| **Per-item local analysis pass** | Each item is analysed on its own, with the full attention budget |
| **Cross-item integration pass** | A separate pass across all items to check cross-cutting concerns (data flow, pattern consistency) |
| **Multi-pass architecture** | Per-item passes plus an integration pass; the structural fix for attention dilution |

## 2. Fixed sequential pipeline (prompt chaining)

**Purpose:** consistent, repeatable processing of structured work.

- **Best for:** code reviews, document processing, data extraction
- **Advantages:** consistent, reliable, easy to debug
- **Limitation:** **cannot adapt to unexpected findings**
- **Example:** invoice extraction. Read the invoice, then extract the number, date and total, then validate, then output. The steps never change.
- **Failure example:** a fixed bug-fix pipeline inspects the named function and proposes a fix, even though the real cause is a misconfigured queue in another service.

## 3. Dynamic adaptive decomposition

**Purpose:** thorough investigation when the scope is unknown.

- **Best for:** legacy system exploration, debugging unfamiliar systems, security audits
- **Advantages:** adapts to the problem; more thorough for open-ended tasks
- **Limitations:** **less predictable; harder to estimate time**
- **Loop:** plan, act, observe, reassess (keep the plan, change it, or stop)
- **Example:** exploring an undocumented 15-year-old codebase. Each discovery decides what to examine next.

## 4. Selection rules

> **Steps known in advance and structured → fixed pipeline.**
> **Open-ended, unknown scope → dynamic decomposition.**

| Task | Pattern |
|---|---|
| Multi-file code review | Fixed pipeline |
| Document extraction | Fixed pipeline |
| Legacy codebase exploration | Dynamic decomposition |
| Debugging an unfamiliar system | Dynamic decomposition |

⚠️ **Exam trap:** tool use doesn't make a task open-ended. The test is whether the *steps* are known in advance.

## 5. Attention dilution

**Cause:** too many items in one pass. It is a **structural** problem, not a capacity problem. It happens even when everything fits in context.

**Three symptoms to recognise:**
1. Detailed feedback for early files, shallow feedback for later ones
2. A pattern flagged in one file but approved in another
3. Obvious bugs missed in some files

## 6. Multi-pass architecture (the fix)

1. **Per-item local passes** catch local issues consistently, because every item gets full attention.
2. **Cross-item integration pass** catches cross-file relationships, such as data flow and pattern consistency.

**Example:** File A now returns `null` instead of an empty list, and File B loops over the result. Each file looks fine alone; only the integration pass catches the crash.

## 7. Anti-patterns (frequent wrong answers)

| Anti-pattern | Why it's wrong |
|---|---|
| Single-pass review of a large codebase | Causes attention dilution |
| Fixed pipeline for open-ended investigation | Cannot adapt to findings |
| Dynamic decomposition for structured tasks | Unnecessary unpredictability, inconsistent results |
| "Use a bigger context window" or "write a better prompt" | Attention dilution is structural, so neither fixes it |
| Batching without a cross-file integration pass | Still misses cross-cutting issues |

## 8. Classic exam scenario

> *14 files reviewed: detailed feedback for files 1–5, obvious bugs missed in 10–14, and a `forEach` loop flagged in one file but approved elsewhere.*

- **Diagnosis:** attention dilution from a single-pass review
- **Fix:** per-file local passes (consistent depth) plus a cross-file integration pass (consistent patterns, data flow)
- **Not the fix:** larger context, stricter prompt, or batching alone

## Memory hooks

- **"Known steps, fixed chain. Unknown scope, adaptive plan."**
- **"Dilution is structural, so fix the structure, not the window."**
- **"Local passes for depth, integration pass for links."**

If you'd like this as a document you can keep and print, I can turn it into one.