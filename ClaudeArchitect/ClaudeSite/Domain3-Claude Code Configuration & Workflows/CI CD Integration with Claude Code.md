# CI/CD Integration with Claude Code — Exam Notes (Task 3.6)

Oct 26, 2026 · @VIjay

## Cheat sheet

Six concepts cover Task Statement 3.6. If you remember only one thing: **a CI pipeline that hangs is always fixed with `-p`.**

| # | Concept | The rule to memorise | One-line example |
| --- | --- | --- | --- |
| 1 | `-p` (`--print`) flag | Non-interactive mode; without it the CI job hangs indefinitely | `claude -p "Analyse this pull request"` |
| 2 | Structured output | `--output-format json` **plus** `--json-schema` for machine-parseable output | Findings parsed into inline PR comments by severity |
| 3 | Session context isolation | Same session is less effective at reviewing its own code; use an independent `claude -p` instance | Generate in one call, review in a fresh call |
| 4 | Incremental review context | Include prior findings; report only new or still-unaddressed issues | Re-run after new commits without duplicate comments |
| 5 | `CLAUDE.md` in CI | Read in CI just as in interactive mode; document testing standards, fixtures, coverage, review criteria | Test generation reuses existing fixtures |
| 6 | Batch API vs real-time | Batch: up to 24 hours, no latency SLA; never for blocking pre-merge checks | Weekly audit → Batch; PR review → real-time |

## 1. The `-p` (`--print`) flag

**Rule:** always run Claude Code in CI with `-p`. It is the single most directly testable fact in Domain 3 (Question 10 in the official sample set).

**What it does:** runs Claude Code in non-interactive mode. It takes the prompt, processes it, prints the result to standard output and exits.

**Why it matters:** a CI runner has no human at the keyboard. Without `-p`, Claude Code starts an interactive session and waits for input that never arrives, so the job hangs indefinitely until the CI platform's timeout kills it.

**Example (correct):**

```bash
claude -p "Analyse this pull request"
```

**Example (wrong, hangs):**

```bash
claude "Analyse this pull request"
```

**Distractors to reject:**

| Option | Why it is wrong |
| --- | --- |
| `CLAUDE_HEADLESS=true` | Does not exist. Fake, made plausible by the word "headless" |
| `--batch` | Does not exist. Do not confuse with the real Message Batches API |
| Redirecting stdin from `/dev/null` | Not the supported mechanism; `-p` is |

**Memory hook:** *Pipeline hangs → `-p`. Every time.*

## 2. Structured output

**Rule:** use `--output-format json` **together with** `--json-schema` when CI must process Claude's output programmatically.

**Why both flags:**

- `--output-format json` makes the output JSON instead of plain text.
- `--json-schema` fixes the **shape** of that JSON: required fields and their types.
- JSON without a schema can change shape between runs (`"severity"` one time, `"level"` the next), which breaks your parsing code.

**What it enables:** posting inline PR comments, filtering by severity, and tracking findings across runs.

**Example:**

```bash
claude -p "Review this pull request for bugs and security issues" \
  --output-format json \
  --json-schema '{
    "type": "object",
    "properties": {
      "findings": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "file":     { "type": "string" },
            "line":     { "type": "integer" },
            "severity": { "type": "string", "enum": ["low", "medium", "high"] },
            "message":  { "type": "string" }
          },
          "required": ["file", "line", "severity", "message"]
        }
      }
    }
  }'
```

The pipeline can now keep only `medium` and `high` findings and post each one as an inline comment on its `file` and `line`.

**Memory hook:** *JSON gives the format; the schema gives the guarantee.*

## 3. Session context isolation (independent review)

**Rule:** never let the session that generated code also review it. Use an **independent review instance**: a separate `claude -p` invocation.

**Why:** the generating session retains its own reasoning context (why it chose an approach, what it assumed). That context **biases it towards its own decisions**, so it confirms its choices instead of questioning them. Same-session self-review is therefore **less effective** than independent review.

**Analogy:** asking an author to proofread their own essay versus handing it to a colleague with fresh eyes.

**Example pipeline:**

1. **Generate:** one invocation makes the change.

   ```bash
   claude -p "Fix the failing test in tests/test_payments.py"
   ```
2. **Review:** a new, separate invocation sees only the diff, not the generator's reasoning.

   ```bash
   git diff origin/main...HEAD | claude -p "Review this diff for bugs and security issues" \
     --output-format json --json-schema "$REVIEW_SCHEMA"
   ```
3. **Post:** the pipeline parses the JSON and adds inline PR comments.

**Wrong version:** continuing the generation session with "Now review what you just changed".

**Memory hook:** *Fresh session, fresh eyes.*

## 4. Incremental review context

**Rule:** when re-running a review after new commits, do two things:

1. **Include the prior review findings** in the context.
2. **Instruct Claude to report only new or still-unaddressed issues.**

Both halves are needed. Context without the instruction can still produce repeats.

**Why:** a review that starts from scratch re-raises issues already flagged, including ones the developer has fixed. Duplicate comments make the review look like noise, developers stop reading it, and real findings get ignored. This **erodes developer trust**.

**Example:**

```bash
claude -p "Previous review findings:
$(cat previous_findings.json)

Review the updated changes on this branch.
Report ONLY new issues, or previous issues that remain unaddressed.
Do not repeat issues that have been fixed." \
  --output-format json --json-schema "$REVIEW_SCHEMA"
```

**Scenario:** the first review flags 5 issues. The developer fixes 3 and pushes. The re-run should report the 2 unfixed issues plus anything new, not all 5 again.

**Memory hook:** *Prior findings in, only new or unfixed out.*

## 5. `CLAUDE.md` for CI context

**Rule:** Claude Code reads `CLAUDE.md` in CI **just as it does in interactive mode**. Running with `-p` does not change this, so `CLAUDE.md` is where you give CI-invoked Claude your project's context.

**What to document:**

| Item | Fixes this symptom |
| --- | --- |
| Testing standards (framework, naming, structure) | Tests ignore team conventions |
| Available fixtures | Claude re-creates fixtures you already have |
| Existing test coverage | Tests duplicate what is already covered |
| Review criteria | Reviews miss what your team cares about |

**Why:** without this documentation, CI-invoked test generation produces **low-value boilerplate**.

**Example `CLAUDE.md`:**

```markdown
# Testing standards
- Use pytest; name test files test_<module>.py
- One behaviour per test; use Arrange / Act / Assert

# Available fixtures (tests/conftest.py)
- db_session: rolled-back database session
- sample_customer: customer with a verified email

# Existing coverage
- payments/refunds.py is fully covered; do not add refund tests

# Review criteria
- Flag any SQL built with string formatting
- Flag missing error handling on external API calls
```

**Exam answer to watch for:** if a question asks where to give CI-invoked Claude Code project context, the answer is **`CLAUDE.md`**.

**Memory hook:** *CI reads `CLAUDE.md` too.*

## 6. Batch API vs real-time

**Rule:** blocking workflows use **real-time** responses; non-blocking, latency-tolerant workflows can use the **Message Batches API**.

**Key fact:** the Batch API has **up to 24 hours** of processing time and **no latency SLA**. "Up to" means it might finish quickly, but nothing is guaranteed.

**Why it matters:** a pre-merge check is **blocking**: the developer cannot merge until it finishes. On the Batch API, a pull request could sit waiting for hours.

| Workload | Blocking? | Use | Why |
| --- | --- | --- | --- |
| Pre-merge code review on a PR | Yes | Real-time | Developer is waiting to merge |
| Pre-merge test generation check | Yes | Real-time | Gates the merge |
| Overnight code-quality report | No | Batch API | Needed by the morning; nobody is waiting |
| Weekly codebase security audit | No | Batch API | Latency-tolerant; cost saving with no downside |

**Exam scenario:** "Move pre-merge review to the Batch API to save costs." Answer: **no**. Cost savings never justify the Batch API for a blocking workflow.

**Memory hook:** *Someone waiting → real-time. Nobody waiting → Batch.*

## Anti-patterns, exam traps and terms

### Anti-patterns (and the fix)

| Anti-pattern | Consequence | Fix |
| --- | --- | --- |
| Running Claude Code in CI without `-p` | Job hangs indefinitely | `claude -p "..."` |
| Same session for generation and review | Self-review bias; issues missed | Separate `claude -p` review instance |
| Re-running reviews without prior findings | Duplicate comments; lost developer trust | Include prior findings; report only new or unaddressed issues |
| No testing standards in `CLAUDE.md` | Low-value boilerplate tests | Document standards, fixtures, coverage, review criteria |
| Batch API for pre-merge blocking checks | PRs blocked for up to 24 hours | Real-time API for blocking checks |

### Exam traps

- **CI pipeline hanging** → the answer is always `-p`.
- **`CLAUDE_HEADLESS=true`** and **`--batch`** are fake. They do not exist.
- **Redirecting stdin from `/dev/null`** is not the answer to a hang.
- **Same-session self-review** retains reasoning context, so it is less effective than an independent review.
- **Batch API** has up to 24 hours of processing and no latency SLA, so it is unsuitable for pre-merge checks, even to save money.
- **`--output-format json` alone** is incomplete; pair it with `--json-schema`.

### Scenario → answer

| If the question describes… | The answer is… |
| --- | --- |
| A CI job that hangs or never completes | `-p` flag |
| Output that must be parsed for inline comments or severity filtering | `--output-format json` + `--json-schema` |
| A review that misses flaws in code the same session wrote | Independent review instance (separate `claude -p`) |
| Repeated comments after new commits | Include prior findings; report only new or still-unaddressed issues |
| Generic tests that ignore conventions and fixtures | Document context in `CLAUDE.md` |
| A cost-saving proposal to batch pre-merge checks | Reject it; use real-time |
| Overnight reports or weekly audits | Batch API is suitable |

### Glossary

| Term | Meaning | Example |
| --- | --- | --- |
| Non-interactive mode | Runs a prompt, prints the result, exits; no human input needed | `claude -p "Summarise this diff"` |
| Structured output | Machine-parseable output in a defined shape | JSON findings with `file`, `line`, `severity` |
| JSON schema | Definition of required fields and types in the JSON | `"required": ["file", "line"]` |
| Self-review bias | A session favouring its own earlier decisions when reviewing them | Generator approves its own flawed fix |
| Independent review instance | A separate invocation with no generation reasoning | Second `claude -p` given only the diff |
| Incremental review | Re-review that reports only new or unaddressed issues | Second run reports 2 of 5 earlier issues |
| Developer trust | Developers' willingness to act on review comments | Lost when duplicates pile up |
| `CLAUDE.md` | Project context file read in CI and interactive mode | Lists fixtures and testing standards |
| Message Batches API | Asynchronous processing, up to 24 hours, no latency SLA | Weekly audit job |
| Blocking workflow | A step that must finish before work continues | Pre-merge check |
| Latency-tolerant workload | Work where a delayed result is acceptable | Overnight report |
