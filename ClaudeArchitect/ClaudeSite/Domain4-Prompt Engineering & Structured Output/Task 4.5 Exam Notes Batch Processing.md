# Task 4.5 Exam Notes: Batch Processing

Oct 27, 2026 · @VIjay

## Purpose

Task Statement 4.5 tests whether you can choose between the synchronous API and the Message Batches API, schedule batches to meet deadlines, and keep batch costs low through good prompts and targeted retries.

The exam rewards one core judgement: **batch saves money but gives up speed and interactivity, so use it only where neither is needed.**

**Example:** a company processes 10,000 customer feedback documents each month for a report read on the 1st. Nobody waits on each document, so batch halves the cost. The same company's live support chatbot must stay synchronous.

## Key terms

| Term | Meaning | Example |
| --- | --- | --- |
| Synchronous API | Send a request, wait, get the answer straight away | A chatbot replies to a customer within seconds |
| Message Batches API | Submit many requests together; results come back later at half the cost | 5,000 contracts submitted at 18:00, collected next day |
| Processing window | Maximum time a batch may take: 24 hours | Submitted Mon 10:00, guaranteed done by Tue 10:00 |
| Latency SLA | A promise of how fast results arrive. Batches have **none** | One batch finishes in 10 minutes, the next in 20 hours |
| `custom_id` | Your own label on each request, returned on its result | `invoice-1042` sent in, result tagged `invoice-1042` |
| Blocking workflow | Someone or something is waiting for the result | A pull request cannot merge until review passes |
| Latency-tolerant workflow | Results are used later; nobody waits | A weekly audit report read on Monday |
| SLA buffer | Deadline minus the 24-hour window | 30-hour SLA − 24 = 6 hours |
| First-pass success rate | Share of documents a prompt handles correctly on the first try | 900 of 1,000 correct = 90% |
| Chunking | Splitting an oversized document into smaller parts | A 60-page invoice sent as three 20-page parts |
| Multi-turn tool calling | Claude asks for a tool, your app runs it, Claude continues | Look up a customer in a database, then write a report |

## 1. Message Batches API constraints

Remember five facts: **50% cheaper, up to 24 hours, no latency SLA, no multi-turn tool calling, `custom_id` for matching.**

| Constraint | Rule to remember | Example |
| --- | --- | --- |
| Cost | 50% saving versus synchronous calls | 10,000 requests costing $200 synchronously cost about $100 in batch |
| Processing window | Up to 24 hours | Submitted Friday 17:00 → finished by Saturday 17:00 at the latest |
| No latency SLA | It may take minutes or the full 24 hours; you cannot predict which | Yesterday's batch took 15 minutes; today's takes 22 hours. Both are normal |
| No multi-turn tool calling | One request in, one response out; your app cannot run a tool mid-request | A workflow needing a database lookup halfway through cannot complete in batch |
| `custom_id` | Match each result to its request by ID, never by order | Results return as `doc-7`, `doc-2`, `doc-9`; you match each by its ID |

**Exam trap:** "Batches usually finish quickly, so it will be ready in an hour." Wrong. Always design for the full 24 hours.

## 2. The matching rule (most tested)

Ask one question: **is someone or something blocked until this result returns?** Yes → synchronous. No → batch.

| Workload | Who is waiting? | Use |
| --- | --- | --- |
| Pre-merge check on a pull request | The PR cannot merge | Synchronous |
| Real-time code review | The developer, now | Synchronous |
| Real-time customer support | The customer, in the chat | Synchronous |
| Overnight sales report | Nobody until morning | Batch |
| Weekly compliance audit | Nobody until the review meeting | Batch |
| Nightly unit-test generation | Developers review next day | Batch |
| Bulk extraction from 50,000 archived documents | Nobody; results go into a database | Batch |

**Exam trap:** "Use batch for pre-merge checks to save 50%." Wrong: a blocking workflow could wait up to 24 hours.

## 3. SLA calculation with the 24-hour window

Always plan for the full 24 hours. **Buffer = SLA − 24 hours.** Submit batches at least as often as the buffer (every 4–6 hours for a 30-hour SLA).

**Rules**

1. Buffer = SLA − 24. For 30 hours: 30 − 24 = **6 hours**.
2. Latest safe submission = deadline − 24 hours.
3. Count the full 30 hours from when a document enters your pipeline, not just 24 hours of processing. Time spent waiting for the next batch counts.
4. Batch cadence must be no longer than the buffer. **Submit every 4–6 hours.**

**Worked example:** documents arrive Monday 09:00; 30-hour SLA → deadline Tuesday 15:00. Latest safe submission = Tuesday 15:00 − 24 h = **Monday 15:00**. A document arriving at 09:05 just misses the 09:00 batch:

| Cadence | Next submission | Worst-case finish | Meets Tue 15:00? |
| --- | --- | --- | --- |
| Every 4 hours | Mon 13:00 | Tue 13:00 | Yes, 2 h spare |
| Every 6 hours | Mon 15:00 | Tue 15:00 | Yes, just |
| Every 8 hours | Mon 17:00 | Tue 17:00 | No, 2 h late |

**Exam trap:** "Submit 24 hours before the deadline." Incomplete: it ignores waiting time before submission. Plan against the whole 30-hour window.

## 4. Batch failure handling

Resubmit **only the failures**, identified by `custom_id`, with a targeted fix. **Never resubmit the entire batch.**

**The four steps**

1. Identify failed documents by `custom_id`.
2. Diagnose why each failed.
3. Apply a targeted fix.
4. Resubmit only those documents.

**The three targeted fixes**

| Cause | Fix | Example |
| --- | --- | --- |
| Document too large | Chunk it | `INV-1042` split into `INV-1042-part1`, `-part2`, `-part3`; results reassembled by ID |
| Prompt too complex | Simplify the prompt | Ask for 5 fields per request instead of 25 |
| Unfamiliar format | Add format-specific examples to the prompt | Include a sample of the new supplier's invoice layout showing where the total sits |

**Worked example:** 1,000 invoices, 940 succeed, 60 fail. Resubmit the 60, chunked or with improved prompts. Do not drop them and do not rerun the 940.

**Exam trap:** "Resubmit the whole batch to be safe." Wrong: it pays again for 940 successful documents.

## 5. Prompt refinement before batch submission

Test on a **representative sample of 5–10 documents**, iterate until accuracy is high, then submit the full batch.

**Steps**

1. Pick 5–10 documents that include the awkward cases (long, unusual layouts), not just clean ones.
2. Run the prompt and check every output.
3. Improve the prompt and rerun until the sample scores high.
4. Submit the full batch.

**Why it pays:**

| First-pass success rate | Failures on 1,000 docs | Retries |
| --- | --- | --- |
| 90% | 100 | 100 |
| 60% | 400 | 400 (4× the cost) |

**Example:** Prompt v1 says "Extract the key details" and gets 6 of 10 sample contracts right (60%). Prompt v2 names exact fields, sets a JSON format and adds an unusual-layout example; it gets 9 of 10 (90%). Submitting v2 saves about 300 retries, and each retry round can add another 24 hours.

**Exam trap:** "Submit everything now and fix failures afterwards." Wrong: skipped refinement multiplies retry cost and delay.

## 6. Multi-turn tool calling limitation

The Batches API does **not** support multi-turn tool calling within a single request. If the workflow needs a tool executed mid-processing, use the **synchronous API**.

**Why:** a batch item is one request in, one response out. There is no point where your application can run the tool and send the result back for Claude to continue.

**Example:** for each contract, Claude must (1) read it, (2) ask your app to look up the counterparty in an internal database, (3) receive the result, (4) write a risk assessment. In batch, every item stops at step 2 with an unexecuted tool request. Use synchronous calls.

**Exam trap:** "It's background work with no deadline, so batch is fine." Wrong: latency-tolerant is not enough if the workflow needs a tool loop.

*Real-world note (October 2026):* batches can now run server-side tools such as web search within one item, but tools your application executes still cannot complete a loop. **For the exam, answer: no multi-turn tool calling in batch.**

## Anti-patterns and quick decision checklist

If an exam answer matches any row below, it is the wrong answer.

| Anti-pattern | Correct approach |
| --- | --- |
| Batch for blocking workflows (pre-merge checks) | Synchronous for anything someone is waiting on |
| Assuming batch results arrive quickly | Design for the full 24 hours; there is no latency SLA |
| Resubmitting the entire batch | Resubmit only failures, found by `custom_id`, with targeted fixes |
| Skipping prompt refinement | Test on 5–10 representative documents first |
| Multi-turn tool calling in batch | Synchronous API for tool loops |

**Decision checklist:**

1. Is someone or something blocked waiting on the result? → **Synchronous.**
2. Does it need multi-turn tool calling mid-processing? → **Synchronous.**
3. Both answers "no" → **Batch**, then:
   1. Buffer = SLA − 24 h; submit batches at least that often.
   2. Refine the prompt on 5–10 sample documents first.
   3. Match results by `custom_id`; resubmit only failures with a targeted fix.

**Numbers to memorise:** 50% saving · 24-hour window · no SLA · 30 − 24 = 6 h buffer · submit every 4–6 h · 5–10 sample docs · 90% → 100 retries, 60% → 400 retries (4×).
