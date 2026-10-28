# Task Statement 1.4: Exam Revision Notes

## 1. Enforcement spectrum

**Purpose:** to control what an agent does, and decide how strongly to enforce it.

| | Prompt-based guidance | Programmatic enforcement |
|---|---|---|
| **What it is** | Instructions in the system prompt | Hooks, prerequisite gates, code-level checks |
| **Behaviour** | **Probabilistic**: works most of the time | **Deterministic**: works every time |
| **Failure rate** | Non-zero | Zero (the model cannot bypass it) |
| **Who decides** | The model chooses whether to comply | The code decides |

**Example:** "Always verify identity before refunding" in the prompt is probabilistic. A gate that blocks `process_refund` until verification succeeds is deterministic.

**Remember:** prompts *guide*; code *guarantees*.

---

## 2. The decision rule

**Purpose:** to decide which approach a given rule needs.

| Category | Examples | Approach |
|---|---|---|
| **Financial** | Refunds, transfers, payments, payment limits | Programmatic |
| **Security** | Identity verification, access control | Programmatic |
| **Compliance** | AML checks, regulatory requirements | Programmatic |
| **Low-stakes** | Formatting, tone, style guidelines | Prompt-based is acceptable |

**Why low-stakes can use prompts:** an occasional miss (a forgotten bullet point) causes no real harm.

**Memory hook:** money, security or law, so use code. Style, so a prompt is fine.

---

## 3. Prerequisite gates

**Purpose:** to enforce workflow **order**, so a tool cannot run until a prior step has completed.

**How it works:**
1. The model calls `process_refund`.
2. The gate checks whether `get_customer` has returned a **verified customer ID**.
3. If not, the tool returns an error and **does not execute**.
4. The model reads the error, calls `get_customer`, then retries successfully.

**Key points:**
- The gate is **code, not a prompt instruction**.
- The model **cannot bypass** it, however convincing its reasoning.
- It checks that a **step happened**, not just general data validity.
- A blocked call **steers the agent back** into the correct order; the agent doesn't simply stop.

---

## 4. The 8% failure scenario (classic exam question)

**Scenario:** the prompt says "verify ownership first", but the agent skips it in 8% of refunds, leading to refunds on wrong accounts.

| Proposed fix | Correct? | Why |
|---|---|---|
| Enhanced system prompt | ❌ | Still probabilistic; reduces failures but can't eliminate them |
| Few-shot examples | ❌ | Still probabilistic, for the same reason |
| Routing classifier | ❌ | Fixes *which agent* handles it, not *how* that agent behaves |
| **Prerequisite gate on `process_refund`** | ✅ | Deterministic; blocks refunds until verification succeeds |

**Remember:** if the requirement is "every time", the answer is code.

---

## 5. Multi-concern request handling

**Purpose:** to resolve compound requests completely, without dropping items.

**Pattern:**
1. **Decompose** the request into distinct items.
2. **Investigate in parallel** using **shared context** (same customer, same order).
3. **Synthesise** one **unified resolution** covering every item.

**Example:** "Damaged item + double charge + address change" means three investigations in parallel, then one reply resolving all three.

**Why shared context matters:** related issues (damage and double charge on the same order) must not produce conflicting or duplicate actions, such as refunding twice.

**Trap:** handling items **one at a time** across separate replies ("Anything else?"), or forgetting minor items like the address update.

---

## 6. Structured handoff protocol

**Purpose:** to let a human agent act immediately on an escalation.

**Critical fact:** the human agent **does NOT have access to the conversation transcript**, so the handoff must be **self-contained**.

**Five required fields:**

| Field | What it gives the human |
|---|---|
| **Customer ID** | The correct account to look up |
| **Conversation summary** | What happened so far |
| **Root cause analysis** | *Why* the problem occurred, not just what |
| **Refund amount** (if applicable) | The exact figure to act on |
| **Recommended action** | What the human should do next |

**Trap:** handoff options that look reasonable but are **missing one field**, often the refund amount or the root cause.

---

## Exam traps at a glance

1. **Better prompts or few-shot examples for high-stakes failures**: wrong, because they're still probabilistic.
2. **Routing classifier for a compliance failure**: wrong, because routing doesn't change agent behaviour.
3. **Handling multi-concern requests sequentially**: wrong; decompose, run in parallel, synthesise.
4. **Incomplete handoff**: wrong; check for all five fields.

## Two-question drill for every scenario

1. **Is it financial, security or compliance?** If so, choose programmatic enforcement.
2. **Is there a handoff?** If so, check for the customer ID, summary, root cause, refund amount and recommended action.