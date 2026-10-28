# Revision Sheet: Task Statement 1.3, Subagent Invocation and Context Passing

## 1. The Task Tool

**What it is:** The tool a coordinator calls to spawn a subagent.

**Key rule:** The coordinator's `allowedTools` **must include "Task"**. This is a hard requirement. Without it, the coordinator cannot spawn subagents at all.

**Example:**
```
❌ allowedTools: ["WebSearch", "Read"]          → coordinator does everything itself
✅ allowedTools: ["WebSearch", "Read", "Task"]  → coordinator can delegate
```

**Exam trap:** A setup has well-written subagent definitions, but no subagents ever run. Check `allowedTools` first.

---

## 2. AgentDefinition: Three Fields

| Field | Purpose | Example |
|---|---|---|
| **description** | Read by the **coordinator** to decide which subagent to pick | "Searches the web for recent sources and returns findings with URLs" |
| **system prompt** | The subagent's standing instructions | "You are a research agent. Return findings in structured format…" |
| **tool restrictions** | Limits the subagent's tools to its role | Web agent gets search tools, not file-writing tools |

**Remember:** A vague description ("Helper agent") leads to poor selection, because the description is written for the coordinator.

---

## 3. The Three Context Passing Rules

### Rule 1: Pass complete findings
Subagents start fresh. They **cannot look up** prior agents' results.

```
❌ "Write a report based on the research findings from earlier."
✅ "Write a report based on these findings: [full findings included here]"
```

### Rule 2: Use structured data that separates content from metadata
Remember the **five fields**:

```json
{
  "claim":         "Remote work increased productivity by 13%",
  "source_url":    "https://example.org/study",
  "document_name": "Stanford WFH Study.pdf",
  "page_number":   14,
  "confidence":    "high"
}
```

**Why it matters:** Without metadata, downstream agents **cannot attribute claims to sources**. The confidence field also helps the synthesis agent weigh conflicting claims and flag uncertain ones.

### Rule 3: Write goal-oriented prompts, not procedural ones

| Procedural ❌ | Goal-oriented ✅ |
|---|---|
| "Step 1: Search X. Step 2: Open first 3 results. Step 3: Copy one stat each." | "Find credible, recent evidence on X. Prioritise peer-reviewed studies. Return findings with source metadata." |
| Blindly follows steps, even when the results are poor | States the **goal + quality criteria** and lets the subagent decide how |

**Key phrase:** Goal-oriented prompts **enable subagent adaptability.**

---

## 4. Attribution Failure Pattern

**Symptom:** A synthesis agent produces claims with no sources.

**Root cause:** The coordinator passed content **without structured metadata**.

**Key phrase:** The synthesis agent **cannot cite sources it was never given.**

| Proposed fix | Correct? | Why |
|---|---|---|
| Add "Always cite sources" to the synthesis prompt | ❌ | An instruction can't create missing data |
| Give the synthesis agent direct search access | ❌ | Treats the symptom and bypasses the architecture |
| **Fix the coordinator's context passing to include metadata** | ✅ | Repairs the actual break |

**Diagnostic method (use elimination):** If the synthesis agent is unchanged and the research subagents are unchanged and still well-sourced, the fault lies in the **coordinator's handover**.

---

## 5. Parallel Spawning

**Rule:** Emit **multiple Task tool calls in a single coordinator response** to spawn independent subagents at the same time.

```
❌ Sequential:  Turn 1 → web agent → wait
               Turn 2 → doc agent → wait
               Turn 3 → db agent  → wait

✅ Parallel:   Turn 1 → [Task: web] [Task: doc] [Task: db]  → merge results
```

**Problem with sequential spawning:** Unnecessary **latency**.

**Caveat:** This applies only to **independent** subagents. A synthesis agent that depends on research results must still wait for them.

---

## 6. fork_session vs --resume

| | **fork_session** | **--resume** |
|---|---|---|
| What it does | Creates an **independent branch** from a shared baseline | **Continues** the same session |
| Use for | **Divergent exploration** (e.g. comparing two refactoring strategies) | **Continuation** of the same line of work |
| Isolation | Each fork is separate, so branches don't contaminate each other | A single thread, so later work inherits everything earlier |

**Memory aid:** *Fork to diverge, resume to continue.*

---

## Quick Anti-Pattern Checklist

When a scenario describes a problem, check for these in order:

1. ☐ Is **"Task"** missing from the coordinator's `allowedTools`?
2. ☐ Is content passed **without metadata**? (This causes attribution failures.)
3. ☐ Is the coordinator prompt **procedural** rather than goal-oriented?
4. ☐ Are independent subagents spawned **sequentially**?
5. ☐ Is **--resume** being used where **fork_session** is needed?

---

## Five Phrases to Memorise

1. *"Without Task in allowedTools, the coordinator cannot spawn subagents."*
2. *"Subagents cannot look up prior results, so pass complete findings."*
3. *"The synthesis agent cannot cite sources it was never given."*
4. *"Goal-oriented prompts enable subagent adaptability."*
5. *"Fork for divergent exploration, resume for continuation."*

If you'd like to keep this, I can turn it into a document you can save and print.