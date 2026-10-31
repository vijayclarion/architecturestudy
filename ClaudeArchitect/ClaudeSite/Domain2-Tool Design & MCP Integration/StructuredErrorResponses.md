# Exam revision sheet: Task Statement 2.2, Structured Error Responses

## 1. The `isError` flag

**Purpose:** It tells the model that **tool execution failed**, so the model reasons about **recovery** instead of treating the error text as a successful result.

**Rule:** Every tool failure must set `isError: true`. Never return error text with `isError: false`.

**Example:**
- ❌ `isError: false`, text: "Database connection timed out". The agent treats this as real data.
- ✅ `isError: true`, text: "Database connection timed out". The agent knows to recover.

## 2. The four error categories

| Category | Examples | `isRetryable` | Recovery |
|---|---|---|---|
| **Transient** | Timeout, service unavailable, rate limit | **true** | Retry after a brief delay |
| **Validation** | Invalid format, missing required field | false | Correct the input and send a **new** call |
| **Business** | Policy violation, limit exceeded | false | Alternative workflow or escalate |
| **Permission** | Access denied, insufficient credentials | false | Escalate or use different credentials |

**Memory aid: "TVBP": "Try, Verify, Break-rule, Permit"**
- **T**ransient: **T**ry again
- **V**alidation: **V**erify and fix the input
- **B**usiness: **B**roke a rule
- **P**ermission: no **P**ermit

**Rules:**
- **Only transient errors are retryable.**
- Validation is `isRetryable: false` because resending the *identical* call fails the same check. Fix the input, then make a new call.
- **Business vs validation test:** Is the input *malformed*? That's validation. Is it *valid but not allowed*? That's business.

**Examples:**
- Date sent as "32/13/2026": **validation**
- A £700 refund against a £500 policy limit: **business**
- HTTP 429 "Too many requests": **transient**
- "User lacks admin role": **permission**

## 3. Structured error metadata (three fields)

| Field | Purpose |
|---|---|
| `errorCategory` | Which of the four categories, so the agent picks the right strategy |
| `isRetryable` | Boolean saying whether resending the same call could work |
| `description` | Human-readable explanation **plus recovery guidance** |

**Rule:** The category and `isRetryable` must agree. Transient means true; the other three mean false.

**Anti-pattern:** A generic `"Operation failed"`. The agent can't decide whether to retry, fix or escalate.

**Example:**
```
isError: true
errorCategory: "business"
isRetryable: false
description: "Refund of £700 exceeds the £500 policy limit.
              Do not retry. Escalate to a human agent."
```

## 4. Access failure vs valid empty result ⭐ (most critical)

| | Access failure | Valid empty result |
|---|---|---|
| Meaning | Tool **could not reach** the data | Tool **queried successfully** and found nothing |
| Cause | Timeout, auth failure | Genuinely no matches |
| `isError` | **true** | **false** |
| Retry? | Consider retrying | **No.** The answer *is* "no results" |

**Key question:** *"Did the tool successfully get an answer, even if it's empty?"* If yes, it's **not an error**.

**Example:** Searching orders for customer 12345:
- Database timed out: **access failure**, `isError: true`, transient
- Query ran and found 0 orders: **valid empty result**, `isError: false`

**What goes wrong if you confuse them:**
- Empty result marked as an error leads to **wasted retries** and **incorrect escalations**.
- Access failure marked as an empty result leads to **false answers**, such as "You have no orders".

## 5. Error propagation in multi-agent systems

**Rules:**
1. **Subagents recover locally** from transient failures, for example by backing off and retrying.
2. **Only propagate** errors that **cannot be resolved locally**.
3. When propagating, include:
   - **Partial results** gathered so far
   - **What was attempted**, such as retries made and which queries ran
4. A failed retry **doesn't change the category**. A rate limit stays transient.

**Example (good propagation):**
```
isError: true
errorCategory: "transient"
isRetryable: true
description: "Rate limit persisted after 3 retries with backoff.
              Completed 4 of 10 queries; 5–10 not run."
partialResults: [results from queries 1–4]
```

**The two anti-patterns:**

| Anti-pattern | What happens | Result |
|---|---|---|
| **Silently suppressing errors** | Subagent returns an empty list as success | The coordinator is misled and the report is **wrong but looks complete** |
| **Terminating the whole workflow** | One subagent fails, so everything is aborted | **Good results are lost** |

✅ **Correct approach:** Propagate honestly with structure and partial results. The coordinator **continues with what succeeded**, notes the gaps, and retries or reassigns.

## Anti-patterns checklist (likely exam distractors)

1. ❌ Generic "Operation failed" with no metadata
2. ❌ Retrying on a valid empty result
3. ❌ Treating business errors as retryable
4. ❌ Silently suppressing subagent errors by returning empty results as success
5. ❌ Terminating the entire workflow on a single subagent failure

## Quick decision flow

1. **Did the tool get an answer, even an empty one?** If yes, use `isError: false` and don't retry.
2. If no, use `isError: true`, then ask **is it temporary?**
   - If yes, it's **transient**: retry with backoff.
   - If no, ask **is the input malformed?**
     - If yes, it's **validation**: fix the input and make a new call.
     - If no, ask **is it a policy or limit?**
       - If yes, it's **business**: escalate or use an alternative.
       - If no, it's **permission** (access denied): escalate or use different credentials.

---

Would you like me to save this as a document you can keep and print, or try the quick-fire round of practice scenarios to test yourself?