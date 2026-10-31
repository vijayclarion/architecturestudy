# Exam Notes: Domain 5, Task Statement 5.2
## Escalation & Ambiguity Resolution

---

## 1. Purpose

The aim is for the agent to **escalate exactly the cases that genuinely need a human**, and resolve everything else itself. Getting this right protects **first-contact resolution** (solving the customer's issue in a single interaction) and makes the best use of human agents' time.

**Example:** A well-calibrated agent processes a simple late-delivery refund itself, but hands a competitor price-match request to a human because policy doesn't cover it.

---

## 2. Key Terms

| Term | Meaning | Example |
|---|---|---|
| **Escalation** | Handing a conversation from the agent to a human | Transferring a customer to a support team member |
| **First-contact resolution** | The issue is solved in one interaction, with no handoff or follow-up | The agent finds a missing refund and processes it in the same chat |
| **Calibration** | How well escalation decisions match what the case actually needs | Like a smoke alarm: it sounds for fires, not for toast |
| **Over-escalation** | Escalating cases the agent could have resolved | Passing a simple "where's my parcel?" query to a human |
| **Under-escalation** | Failing to escalate when a human is needed | The agent keeps retrying a broken tool instead of handing off |
| **Policy gap** | Policy is **silent** on the request; there is no documented answer | Policy covers own-site price matching but says nothing about competitors |
| **Policy violation** | Policy **explicitly says no**; a documented answer exists | "No refunds after 30 days", and the customer asks on day 45 |
| **Sentiment analysis** | Detecting the customer's emotional tone | Flagging "This is RIDICULOUS!" as angry |
| **Self-reported confidence** | The model rating its own certainty | "I am 8/10 confident in this answer" |
| **Poorly calibrated** (confidence) | Stated confidence doesn't match how often the model is right | The model says 9/10 on a case it gets wrong |
| **Heuristic** | A shortcut guess or rule of thumb | "Pick the James Smith who ordered most recently" |
| **Few-shot examples** | Worked example cases included in the prompt to teach a distinction | Showing one angry-but-simple case resolved and one calm policy-gap case escalated |

---

## 3. The Three Valid Escalation Triggers

**Memory aid: "Asks, Absent, Stuck"** (customer *asks* for a human, policy is *absent*, agent is *stuck*)

### Trigger 1: Customer explicitly requests a human
- **Rule:** Escalate **immediately**. Do not investigate, gather details or offer to help first.
- **Status:** Absolute rule with **no exceptions**.
- **Example:** *"I'd like to speak to a real person, please."* The agent transfers straight away, without saying "Let me just check your order first."

### Trigger 2: Policy exception or gap
- **Rule:** If the policy is **silent**, escalate. If the policy **says no**, explain the policy and do not escalate.
- **Example (gap, so escalate):** *"Will you match a competitor's price?"* when policy only covers own-site matching.
- **Example (violation, so don't escalate):** *"Can I have a refund?"* 45 days after delivery, when policy states no refunds after 30 days. The agent politely explains the policy.

### Trigger 3: Inability to make meaningful progress
- **Rule:** The agent **attempts** resolution first, then escalates once it cannot advance.
- **Causes:** Tool errors, missing system access, or a bug only engineering can fix.
- **Example:** The agent tries to update a delivery address, but the order system keeps returning an error and there is no other route. The agent escalates.

| Trigger | Try to resolve first? |
|---|---|
| Customer asks for a human | **No.** Escalate immediately |
| Policy gap | No documented answer exists, so escalate |
| Can't make progress | **Yes.** Escalate once genuinely stuck |

---

## 4. The Two Unreliable Triggers

### Unreliable trigger 1: Sentiment-based escalation
- **Why it fails:** Frustration **does not correlate with case complexity**.
- **Example:**
  - Furious customer with a parcel three days late: simple, so the agent should resolve it. A sentiment rule would wrongly escalate it.
  - Calm customer asking for a competitor price match: a policy gap, so it needs a human. A sentiment rule would wrongly miss it.
- **Result:** It fails in **both directions** at once.

### Unreliable trigger 2: Self-reported confidence scores
- **Why it fails:** LLM self-confidence is **poorly calibrated**.
  - **Hard cases:** often confidently wrong, so it doesn't escalate when it should.
  - **Easy cases:** often needlessly uncertain, so it escalates when it shouldn't.
- **Example:** A rule says "escalate if confidence is below 6/10". The model rates a tricky policy-gap case 9/10 and keeps it, then rates a simple tracking query 4/10 and escalates it.

---

## 5. The Frustration Nuance

**Principle:** The *case* decides whether to escalate. The *emotion* decides how warmly you respond.

| Situation | Agent's action | Example |
|---|---|---|
| Frustrated, resolvable issue, no request for a human | Acknowledge the frustration, then resolve directly | *"I'm sorry you've waited two weeks. I've found the problem and can process your refund now."* |
| Agent offers help, customer **reiterates** they want a human | Escalate now | *"No, I've had enough of bots. I want a human."* The agent transfers. |
| Customer says **"I want a human"** from the very start | Escalate immediately, with no investigation and no offer to help first | First message: *"Put me through to a person."* The agent transfers at once. |

---

## 6. Ambiguous Customer Matching

- **Rule:** When several customers match a search, **ask for an additional identifier**: email address, phone number or order number.
- **Never:** Select a match using a heuristic such as most recent order or most active account.
- **Why:** A wrong guess exposes another customer's personal data, which is a **privacy violation**.
- **Example:** *"I'm James Smith, can you check my order?"* There are three James Smiths, one of whom ordered yesterday. The agent replies: *"I've found more than one account under that name. Could you confirm your email address or order number?"* It does **not** assume the recent buyer is the right one.

---

## 7. Fixing Escalation Behaviour: Explicit Criteria with Few-Shot Examples

- **Rule:** The **proportionate first response** is to add explicit escalation criteria, with few-shot examples, to the system prompt.
- **Order to remember:** **Prompt optimisation first, architecture later.** Classifier models or sentiment analysis come only after prompt improvements have been tried.
- **Example of prompt content:**
  > *Escalate when: the customer asks for a human; the request is not covered by policy; you cannot make progress because of a system error.*
  > *Do not escalate merely because the customer is upset.*
  >
  > *Example 1: Angry customer, late parcel. Apologise, check tracking, resolve directly.*
  > *Example 2: Calm customer, competitor price match. Policy is silent, so escalate.*

---

## 8. Anti-Patterns and Their Corrections

| Anti-pattern | Correct approach | Example |
|---|---|---|
| Using sentiment analysis to trigger escalation | Escalate on the three valid triggers only | Resolve the angry late-delivery customer directly |
| Relying on self-reported confidence scores | Use explicit criteria, not the model's self-rating | Don't escalate just because the model says "5/10" |
| Investigating before honouring an explicit human request | Escalate immediately | Don't say "Let me check your order first" |
| Selecting from ambiguous matches with heuristics | Ask for email, phone or order number | Don't pick the James Smith who ordered yesterday |
| Deploying classifier models before prompt optimisation | Add explicit criteria with few-shot examples first | Improve the system prompt before building a classifier |

---

## 9. Exam Traps

1. **Sentiment sounds reasonable** but frustration doesn't track complexity. Reject answers that escalate because a customer is angry.
2. **Self-reported confidence** is confident on hard cases and uncertain on easy ones. Reject confidence-threshold answers.
3. **"I want a human" means escalate immediately.** Reject "helpful" options that gather details or attempt a fix first.
4. **Silent policy means escalate; policy that says no means explain.** Read the scenario carefully to see which it is.
5. **Heavy solutions are rarely the first answer.** If an option offers a classifier or sentiment model and another offers explicit prompt criteria with examples, the prompt option is usually correct.

---

## 10. Quick Decision Flow

1. Has the customer **asked for a human**? Yes: escalate immediately.
2. Do several customers **match**? Yes: ask for an extra identifier.
3. Is the request **covered by policy**?
   - Policy says yes: resolve it.
   - Policy says no: explain the policy.
   - Policy is silent: escalate.
4. Can the agent **make progress**? No (tool error, no access, bug): escalate.
5. Is the customer **frustrated**? Acknowledge it, but let the case decide the action.

---

I can also turn these into a doc you can edit, share, or download as Word or PDF, if you'd like.