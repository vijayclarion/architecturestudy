# Task 4.6 Exam Notes: Multi-Instance and Multi-Pass Review

Oct 27, 2026 · @VIjay

## Purpose of Task 4.6

Task 4.6 tests whether you can design review systems that reliably catch errors in Claude's output. The core idea: a single model, in a single pass, checking its own work, is not reliable. You fix this with three tools.

1. **Independence:** review with a separate instance, not the author.
2. **Focus:** split large reviews into per-file passes, then an integration pass.
3. **Calibrated routing:** send findings to developers or humans based on confidence scores you have tested against known answers.

**Exam tip:** most questions describe a symptom (missed bugs, contradictory findings, wrong findings reaching developers) and ask for the fix. Learn to map symptom → cause → fix.

## 1. Self-review limitation

**Purpose:** explain why a model reviewing its own output in the same session misses subtle issues, and what to do instead.

**Important points**

- In the same session, the model **retains its reasoning context**. It remembers *why* it made each decision.
- Because it remembers its justifications, it is **less likely to question** those decisions.
- An **independent instance** (a separate Claude invocation with no prior reasoning context) approaches the output fresh and catches more subtle issues.

**Rules**

- Never rely on same-session self-review for validation.
- Self-review instructions ("be extremely critical") are **not** a substitute.
- Extended thinking in the same session is **not** a substitute. It changes how hard the model looks, not what it already believes.
- Give the independent reviewer the **output and the requirements**, not the author's conversation or reasoning. Passing the reasoning reintroduces the bias.

**Terms to remember:** reasoning context, independent instance, self-review.

**Example**

- **Weak:** Claude writes a date-parsing function, then in the same chat you ask "review the code you just wrote". It confirms its own timezone handling because it remembers deciding to assume UTC.
- **Strong:** a new Claude invocation receives only the function and the spec ("must handle user-local timezones"). With no memory of the UTC assumption, it flags the mismatch.

**Remember the phrase:** "retains its reasoning context", not just "knows the conversation".

## 2. Multi-pass review architecture

**Purpose:** keep review depth consistent across large inputs (multi-file PRs, complex pipelines) and still catch problems that span files.

**Important points**

- A single pass over a large review suffers from **attention dilution**: attention is spread unevenly across the material.
- This happens **even when everything fits in context**. Nothing is cut off or chunked; it is a quality problem, not a capacity problem.
- **The three symptoms** (learn these; exams describe them without naming the cause):
  1. Detailed feedback on some files, superficial feedback on others.
  2. Obvious bugs missed, often in the middle.
  3. Contradictory findings: a pattern flagged in one file but identical code approved elsewhere.

**The fix: two kinds of pass**

1. **Per-file local analysis passes:** each file reviewed in its own focused pass, giving consistent depth.
2. **Cross-file integration pass:** run afterwards, looking at how files interact. It catches **data flow issues** and performs **contradiction detection**.

**Rules**

- Don't process all files in a single pass.
- Don't skip the integration pass. Per-file passes alone cannot see cross-file problems.

**Terms to remember:** attention dilution, per-file local analysis pass, cross-file integration pass, data flow issue, contradiction detection.

**Example**

A 25-file PR is reviewed in one pass. Early files get detailed comments; later files get one line. A null-check bug in file 12 is missed. Risky error handling is flagged in `payment.py` but approved in `refund.py`.

- **Per-file passes** fix the uneven depth and the missed bug in file 12.
- **Integration pass** spots that `payment.py` sends amounts in pence while `refund.py` expects pounds (data flow), and that the two files received conflicting verdicts (contradiction).

## 3. Why a larger context window does not fix attention dilution

**Purpose:** recognise a tempting but wrong fix that often appears as an exam distractor.

**Important points**

- A larger context window gives the model more **room**. It does not make the model spread its attention **evenly** across files.
- The problem is **attention quality, not context size**.
- Only **focused, per-file passes** ensure consistent depth.

**Rule:** don't switch to a larger-context model to fix attention dilution.

**Terms to remember:** attention quality vs context size.

**Example**

A team lead says: "Our 25 files are getting patchy reviews. Let's move to the model with the bigger context window and keep one pass." The 25 files already fit, so extra room changes nothing. The same three symptoms appear. The correct answer is per-file passes plus an integration pass.

**Exam trap:** any answer option framing the fix as "more context", "bigger window" or "stop files being cut off" is treating a quality problem as a capacity problem.

## 4. Confidence-based routing

**Purpose:** stop developers being overwhelmed by findings, while keeping a human safety net for uncertain ones.

**Important points**

- The model **self-reports a confidence score** for each finding.
- **High-confidence** findings go **directly to developers**.
- **Low-confidence** findings are routed to **human review** first.
- **The catch:** raw self-reported confidence is **poorly calibrated**. "90% confident" does not reliably mean "correct 90% of the time".

**Why uncalibrated scores are dangerous**

- A wrong finding with high confidence skips human review and reaches developers, wasting time and eroding trust.
- A correct, important finding with low confidence gets delayed in the human queue.

**Rule:** don't use uncalibrated confidence scores for automated routing decisions.

**Terms to remember:** self-reported confidence, routing threshold, poorly calibrated.

**Example**

You set "confidence ≥ 0.8 → developers". In practice, the model scores almost every finding between 0.85 and 0.95, including false alarms. Nearly everything bypasses human review, and developers receive many wrong findings. The routing idea was sound; trusting the raw number was the mistake.

## 5. Calibrating confidence thresholds

**Purpose:** turn unreliable raw confidence scores into evidence-based routing thresholds.

**Important points**

- Use a **labelled validation set**: cases where you already know the correct answer.
- Run the reviewer on it and record each finding's **reported confidence** alongside whether it was **actually correct**.
- Measure the **relationship between reported confidence and actual accuracy**.
- **Adjust routing thresholds** based on this calibration data, not on the raw number.

**The process**

1. Build a labelled validation set with known correct answers.
2. Run the reviewer and collect confidence scores per finding.
3. Compare confidence against actual accuracy, band by band.
4. Set the threshold where accuracy is reliably high enough.
5. Route findings above it to developers and the rest to human review.

**Rule:** uncalibrated confidence scores are unreliable for automated decisions. Calibrate first.

**Terms to remember:** labelled validation set, calibration, confidence vs accuracy, routing threshold.

**Example**

| Reported confidence | Actual accuracy on validation set |
| --- | --- |
| 0.95 and above | 94% correct |
| 0.85–0.94 | 71% correct |
| Below 0.85 | 48% correct |

The raw scores look high everywhere, but the data shows only findings at 0.95 or above are reliable. You set the developer threshold at **0.95**, not 0.8, and route everything below it to human review.

## Anti-patterns to recognise

Each of these is a likely wrong answer option. Know the example and the fix.

| Anti-pattern | Example | Why it fails | Fix |
| --- | --- | --- | --- |
| Model reviews its own output in the same session | "Now review the code you just wrote" | Retains reasoning context; defends its own decisions | Independent instance with the output and requirements only |
| All files in a single pass | One review prompt for a 25-file PR | Attention dilution: uneven depth, missed bugs, contradictions | Per-file local analysis passes |
| Skipping the integration pass | Per-file passes only, then ship | Cross-file data flow issues and contradictions go unseen | Add a cross-file integration pass |
| Larger context window to fix dilution | "Switch to the bigger-window model" | Attention quality, not context size, is the problem | Per-file passes plus an integration pass |
| Uncalibrated confidence for routing | "Anything ≥ 0.8 goes straight to devs" | Self-reported confidence is poorly calibrated | Calibrate thresholds on a labelled validation set |

## Glossary and exam checklist

| Term | Meaning | Quick example |
| --- | --- | --- |
| Reasoning context | The model's memory of why it made its decisions in the current session | It remembers choosing UTC, so it won't question it |
| Independent instance | A separate Claude invocation with no prior reasoning context | A fresh call that sees only the code and the spec |
| Attention dilution | Uneven attention across a large input reviewed in one pass | File 3 gets ten comments, file 20 gets one |
| Per-file local analysis pass | A focused review of one file at a time | 25 files → 25 focused passes |
| Cross-file integration pass | A pass checking how files interact, after the per-file passes | Spots pence vs pounds between two files |
| Data flow issue | A bug in how data moves between files | One file sends a format the other doesn't expect |
| Contradiction detection | Finding inconsistent verdicts on identical code | Flagged in `payment.py`, approved in `refund.py` |
| Confidence-based routing | Sending findings to developers or humans by confidence score | High → developers, low → human review |
| Calibration | Measuring how reported confidence relates to actual accuracy | 0.9 scores turn out 71% correct |
| Labelled validation set | Test cases where the correct answer is already known | 200 past PRs with confirmed bugs |

**Last-minute checklist**

- [ ] I can explain why self-review fails using the phrase "retains its reasoning context".
- [ ] I know extended thinking and "be critical" instructions don't fix self-review.
- [ ] I can list the three symptoms of attention dilution.
- [ ] I know attention dilution happens even when everything fits in context.
- [ ] I know the architecture: per-file passes, then a cross-file integration pass.
- [ ] I know a larger context window is the wrong fix.
- [ ] I know raw self-reported confidence is poorly calibrated.
- [ ] I can describe calibration with a labelled validation set and threshold adjustment.
