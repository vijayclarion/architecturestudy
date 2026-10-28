# Agent SDK Hooks: Exam Revision Sheet
*Domain 1, Task Statement 1.5*

## 1. Key terms

| Term | Meaning |
|---|---|
| **Hook** | Deterministic code that runs at a fixed point in the tool execution lifecycle |
| **PreToolUse hook** | Runs **before** a tool executes and intercepts the outgoing tool call |
| **PostToolUse hook** | Runs **after** a tool executes but **before** the model processes the result, and intercepts the tool result |
| **Normalisation** | Converting heterogeneous data formats into one consistent representation |
| **Policy enforcement** | Blocking, modifying or redirecting tool calls that would break a business rule |
| **Deterministic guarantee** | Runs every time, 100% (hooks) |
| **Probabilistic guidance** | Usually followed, but not guaranteed (prompts) |

## 2. The lifecycle

```
Model decides to call tool
        ↓
  [PreToolUse hook]  ← can BLOCK / MODIFY / REDIRECT the call
        ↓
   Tool executes     ← the action happens here (money moves, data changes)
        ↓
  [PostToolUse hook] ← can TRANSFORM the result
        ↓
Model processes result
```

## 3. PostToolUse: data normalisation

**Purpose:** give the model clean, consistent data every time.

**Why it matters:** with mixed formats, the model has to guess, and it may misinterpret the data.

**Example: dates from three MCP tools**

| Tool | Raw output | After the hook |
|---|---|---|
| A | `1729843200` (Unix) | `2024-10-25T08:00:00Z` |
| B | `2024-10-25T08:00:00Z` (ISO 8601) | `2024-10-25T08:00:00Z` |
| C | `25/10/2024` (DD/MM/YYYY) | `2024-10-25` (ISO 8601) |

**Example: status codes.** A numeric code `3` becomes the string `"shipped"`.

**Rule:** Unix timestamps convert to ISO 8601, and numeric codes convert to human-readable strings.

## 4. PreToolUse: policy enforcement

**Purpose:** stop a forbidden action **before** it happens.

**Possible actions:** block, modify or redirect the tool call.

**Example 1: a threshold rule**
- `process_refund(amount=750)` is intercepted.
- The amount is above $500, so the hook **blocks** the call and **redirects it to human escalation**.
- The hook only needs to inspect the **current call's parameters**.

**Example 2: a prerequisite rule**
- `international_transfer(...)` is intercepted.
- If the AML check has not passed, the hook **blocks** the transfer.
- The hook needs to know **what happened earlier**.

## 5. Decision framework: hooks or prompts?

| | Hooks | Prompts |
|---|---|---|
| Nature | Deterministic | Probabilistic |
| Use for | Business rules that must be followed 100% of the time | Preferences and soft rules |
| Examples | Refund limits, AML checks, compliance rules | Tone, greeting style, adding a gift note |

**Golden test:**
> If a **single failure** would cause **financial loss or legal risk**, use a **hook**. Otherwise a prompt is fine.

## 6. The critical direction distinction (heavily tested)

| Hook | Correct for | Wrong for | Why it's wrong |
|---|---|---|---|
| **PostToolUse** | Normalisation | Blocking actions | Too late: the action has already occurred |
| **PreToolUse** | Blocking actions | Normalisation | No result exists yet to transform |

## 7. Anti-patterns (common exam distractors)

1. **Using PostToolUse to block policy violations.** By the time it runs, the refund or transfer has already happened.
2. **Using prompts for mandatory compliance.** Prompts are probabilistic, so they can't guarantee 100% enforcement.
3. **Not normalising heterogeneous tool outputs.** The model then misinterprets inconsistent formats.
4. **Confusing hook directions.** Remember: Pre means *before execution*, and Post means *after execution, before the model*.

## 8. How to answer exam scenarios

1. **Is it about cleaning or transforming data the model receives?** Use PostToolUse.
2. **Is it about stopping or controlling an action?** Use PreToolUse.
3. **Must it work 100% of the time, with money or legal risk involved?** Use a hook, not a prompt.
4. **Is it just a preference?** A prompt is fine.
5. **Always rule out the distractors** by saying why the other options fail: wrong timing, or not reliable enough.

**Model answer:** *"The agent sometimes skips AML checks, and 100% enforcement is required."*
Use a **PreToolUse hook**, because it is deterministic and runs before the transfer executes.
- PostToolUse is too late, because the money has already moved.
- Prompts are probabilistic, so they can't guarantee 100%.

## 9. Memory hooks

> **Pre = Prevent** (policy enforcement)
> **Post = Polish** (data normalisation)
> **Prompts = Preferences** (soft rules)

> **Gate vs filter:** PreToolUse is a *gate* on actions; PostToolUse is a *filter* on results.

> **"Money or law? Hook it."**

I can turn this into a doc you can keep, print or export to PDF, or give you a rapid-fire round of scenario questions to test it.