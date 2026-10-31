# Exam Notes: Context Window Management (Task 5.1)

Oct 29, 2026 · @Vijay

## Purpose of Task 5.1

Context window management keeps the right facts in front of the model, in the right place, with no noise, for every request. It matters because the context window is limited, the API is stateless, and long inputs are processed unevenly.

**Three problems it solves**

- **Lost facts:** precise details (amounts, dates, IDs) disappear when history is condensed or truncated.
- **Missed information:** content buried in the middle of long inputs is under-attended.
- **Wasted tokens:** verbose tool results and reasoning chains crowd out useful content and are resent on every turn.

**Golden rule:** every fix in this domain is **structural** (what goes in, where it sits, what is protected), never "use a bigger window" or "tell the model to pay attention".

**Example:** a support agent on turn 40 must still know the customer wants £89.50 back for order #4471. Good context design guarantees that; a bigger window alone does not.

## 1. Progressive Summarisation Trap

**Purpose:** recognise why condensing history is dangerous for precise data.

**Important points**

- Summarisation keeps the gist and drops the precision.
- Most at risk: numbers, dates, percentages, identifiers and customer-stated expectations.
- The loss compounds: each round of condensing blurs the facts further.

**Rules**

- Never rely on summarised history to hold transactional facts.
- Fix: extract those facts into a **persistent case facts block** that is never summarised (see section 2).

**Terms to remember:** progressive summarisation, transactional facts, numerical precision, customer-stated expectations.

**Example**

| Before summarising | After summarising |
| --- | --- |
| Customer wants a refund of $247.83 for order #8891 placed on March 3rd | Customer wants a refund for a recent order |

Lost: the amount, the order number and the date.

**Exam trap:** progressive summarisation is NOT safe for transactional data.

## 2. Persistent Case Facts Block (most important pattern)

**Purpose:** guarantee that precise transactional facts survive however long the conversation runs.

**Important points**

- Facts (amounts, dates, order numbers, statuses) are extracted into a **structured** block.
- The block is **prepended to every prompt**, outside the summarised history.
- It is **never summarised or compacted**.
- In multi-issue sessions, **each issue gets its own entry**.
- It makes history management safe: older turns can be condensed because the critical facts are protected elsewhere.

**Rules**

- Structured fields, not free-text prose.
- Include a status per issue so the agent knows what is open or resolved.
- Label customer claims as customer-stated rather than confirmed.

**Terms to remember:** case facts block, prepended, outside summarised history, per-issue entries, status.

**Example**

```
CASE FACTS (never summarised; included in every request)

Issue 1 — Refund
  Order: #4471 | Amount: £89.50 | Status: refund pending approval

Issue 2 — Late delivery
  Order: #5520 | Promised by: 12 June | Status: awaiting courier update

Issue 3 — Billing query
  Discount claimed: 15% (customer-stated) | Status: verifying offer
```

**Exam tip:** an answer that only says "store the facts" is incomplete. Look for structured, prepended, never summarised and per-issue.

## 3. Lost in the Middle Effect

**Purpose:** make sure key findings in long inputs are actually used.

**Important points**

- Models reliably process the **beginning** and **end** of long inputs.
- Findings buried in the **middle** may be missed, even though they are present.
- It is a silent failure: the input looks complete when you inspect it.
- Longer inputs mean more middle, so a bigger window can make it worse.

**Rules**

- Fix it **structurally**:
  - place a **key findings summary at the beginning** of aggregated inputs;
  - use **explicit section headers** throughout.
- Do not rely on prompt reminders such as "pay attention to everything"; they are unreliable.

**Terms to remember:** lost in the middle, aggregated inputs, key findings summary, section headers, structural fix.

**Example**

Five subagents return research findings that are combined for a final agent. The critical result sits in subagent 3's output, mid-input.

- Wrong fix: add "Please read all findings carefully" to the prompt.
- Right fix: open with a "Key findings" summary listing the critical result, then give each subagent's output its own header.

**Exam trap:** the fix is structural (reorder inputs), not prompt-based.

## 4. Tool Result Trimming

**Purpose:** stop verbose tool outputs from wasting tokens and burying useful data.

**Important points**

- Lookups often return far more than needed (e.g. 40+ fields when only 5 matter).
- Because the API is stateless, a bloated result is resent in **every later request**, so the waste compounds.
- Extra fields are noise that hides the relevant ones.

**Rules**

- Trim **before** the result enters conversation history.
- Two places to trim:
  - **within the tool implementation** (when you control the tool);
  - a **PostToolUse hook** (when you cannot change the tool, e.g. a third-party API).
- Never keep everything "just in case".

**Terms to remember:** tool result trimming, PostToolUse hook, relevant fields, token cost.

**Example**

An order lookup returns 42 fields, including warehouse codes, internal SKUs and audit timestamps. The customer asks "Where is my order?" A PostToolUse hook keeps only order ID, status, shipped date, carrier and tracking number before the result is added to history.

**Exam trap:** trim BEFORE entering context, not afterwards and not "just in case".

## 5. Full Conversation History Requirement

**Purpose:** keep the conversation coherent, given that the model remembers nothing between requests.

**Important points**

- The Claude API is **stateless**: each request must include the complete conversation history.
- Anything omitted from a request does not exist for the model.
- Dropping earlier messages breaks coherence: later turns may refer to things the model can no longer see.
- Current documentation (October 2026) adds **context editing**, which clears chosen content **deliberately**. That is controlled clearing, not arbitrary truncation.

**Rules**

- Send complete history with every request.
- Never silently or selectively truncate older messages to save tokens.
- Protect critical facts in the case facts block so any history management stays safe.

**Terms to remember:** stateless API, complete history, conversational coherence, selective truncation, context editing.

**Example**

On turn 1 the customer gives order #4471 and a £89.50 refund amount. A developer drops the oldest five messages on turn 30 to save tokens. The agent then asks the customer for their order number again, or worse, acts on a wrong guess.

**Exam trap:** ad hoc truncation loses facts; deliberate context editing is a different, controlled mechanism.

## 6. Upstream Agent Optimisation

**Purpose:** stop upstream agents from flooding downstream agents that have limited context.

**Important points**

- Verbose reasoning chains ("First I considered X, then Y…") waste the downstream agent's context.
- Upstream agents should return **structured data**: key facts, citations and relevance scores.
- Structured outputs must include **metadata**: dates, source locations and methodological context.

**Rules**

- Modify the upstream agent's output format, rather than asking the downstream agent to cope.
- Prefer structured extraction over a prose "summary", which risks the summarisation trap.
- Make metadata required, not optional.

**Terms to remember:** upstream agent, downstream agent, structured output, citations, relevance scores, metadata, methodological context.

**Example**

| Verbose (avoid) | Structured (use) |
| --- | --- |
| "I searched several sources, considered the 2023 report, then compared it with…" | Fact: market grew 12% · Source: 2023 industry report, p. 14 · Date: 2023 · Method: survey of 400 firms · Relevance: 0.9 |

**Exam tip:** when you see "summarise and pass on", look for the option that says "structured output with metadata" instead.

## Exam Traps and Anti-Patterns

If an answer option matches the left column, it is almost certainly wrong.

| Anti-pattern (wrong) | Correct approach | Example |
| --- | --- | --- |
| Summarise transactional facts into vague descriptions | Persistent case facts block, never summarised | "$247.83 for #8891" kept in the block, not "a recent order" |
| Tell the model to "pay attention to everything" | Key findings summary at the beginning, plus section headers | "Key findings" opener before five subagent outputs |
| Place critical findings only in the middle of long inputs | Move them to the beginning | Critical result lifted from subagent 3 into the opening summary |
| Append full verbose tool results "just in case" | Trim before entering context (tool or PostToolUse hook) | 42 order fields cut to 5 |
| Omit or truncate earlier messages to save tokens | Send complete history; use deliberate context editing if clearing is needed | Turn 1 refund details still present on turn 30 |
| Let upstream agents return verbose reasoning chains | Structured output with facts, citations, relevance scores and metadata | Fact + source + date + method + score |
| "Use a model with a bigger context window" | Structural fixes; a bigger window only delays or worsens the problems | Longer input means more middle to get lost in |

## Glossary of Terms

| Term | Meaning | Example |
| --- | --- | --- |
| Context window | The limited amount of text the model can process in one request | A 40-turn chat plus tool results filling the window |
| Stateless API | The API keeps no memory between requests | Turn 2 must resend turn 1 |
| Progressive summarisation | Repeatedly condensing history, losing precision each time | "$247.83" becomes "a refund" |
| Transactional facts | Precise, actionable data: amounts, dates, IDs, statuses | Order #4471, £89.50, 12 June |
| Customer-stated expectation | What the customer says they were promised or expect | "I was offered 15% off" |
| Case facts block | Structured, never-summarised block of facts prepended to every prompt | Per-issue entries with status |
| Lost in the middle | Under-attention to content in the middle of long inputs | Key finding missed in subagent 3's output |
| Aggregated input | Several outputs combined into one input | Five subagent reports merged |
| Key findings summary | Short list of the most important points placed first | "Key findings: X, Y, Z" opener |
| Tool result trimming | Cutting tool output to relevant fields before it enters history | 42 fields cut to 5 |
| PostToolUse hook | Code that runs after a tool returns and before its result enters history | Strips audit fields from a third-party API |
| Selective truncation | Dropping some messages from history (unsafe) | Removing the oldest 5 turns |
| Context editing | Deliberate, controlled clearing of chosen content (current docs, October 2026) | Clearing an old, no-longer-needed tool result on purpose |
| Upstream / downstream agent | The agent producing output / the agent consuming it | Research subagent / final writer agent |
| Metadata | Context about a fact: date, source location, method | "2023 report, p. 14, survey of 400 firms" |

## Quick Self-Check Before the Exam

Tick each one when you can explain it in one sentence without notes.

- [ ] Why progressive summarisation is unsafe for transactional data
- [ ] What goes in a case facts block, where it sits, and how it handles multiple issues
- [ ] Why the lost-in-the-middle fix is structural, not prompt-based
- [ ] The two places to trim tool results, and when to use each
- [ ] Why the stateless API makes silent truncation dangerous, and how context editing differs
- [ ] What structured output an upstream agent should return, including its metadata
- [ ] Why a bigger context window does not fix any of these problems

**Personal reminders from your practice session**

- Give full answers: list every property (structured, prepended, never summarised, per-issue, status).
- Lost in the middle = summary at the **beginning** + section headers.
- Treat the word "summarise" as a warning sign; prefer "structured extraction with metadata".
