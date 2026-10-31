# Revision sheet: Task Statement 1.2, Orchestration Patterns

## 1. Hub-and-spoke architecture
- **Coordinator** sits at the centre (the hub). **Subagents** are the spokes and handle specialised tasks.
- **All communication flows through the coordinator.**
- **Subagents never communicate directly with each other.**

**Three benefits of centralisation:**

| Term | Meaning |
|---|---|
| **Observability** | Log and monitor every message in one place |
| **Consistent error handling** | Uniform recovery policies across all subagents |
| **Controlled information flow** | Coordinator decides what context each subagent receives |

## 2. Critical isolation principle (most misunderstood concept)
- Subagents **do not inherit** the coordinator's conversation history.
- Subagents **do not share memory** between invocations.
- **Every piece of information a subagent needs must be explicitly included in its prompt.**
- Example: if the web search subagent is called twice, the second call knows nothing of the first.
- **Diagnostic reflex:** when a subagent "ignores" context, check whether the context was ever passed to it. Don't blame the model.

## 3. Coordinator responsibilities

| Term | Meaning |
|---|---|
| **Dynamic subagent selection** | Analyse the query and invoke only the subagents it needs |
| **Research scope partitioning** | Assign distinct subtopics to minimise duplication |
| **Iterative refinement loops** | Evaluate output for gaps, then re-delegate |
| **Centralised communication routing** | Supports observability and error handling |

## 4. Narrow decomposition failure
- The coordinator decomposes a broad topic **too narrowly** and misses entire categories.
- Example: "AI in creative industries" split only into visual arts, missing music, writing and film.
- **Root cause is the coordinator's decomposition**, not any downstream subagent.
- **Trace failures to their origin**, not to where they first become visible.
- **Fix: better decomposition**, not more subagents.

## 5. Anti-patterns to recognise

| Anti-pattern | Why it's wrong |
|---|---|
| Direct subagent-to-subagent communication | Breaks observability |
| Assuming subagents share context or memory | They don't, so context must be explicit |
| Always routing through the full pipeline | Ignores query complexity and wastes effort |
| Adding more subagents to fix a decomposition problem | The fix is better decomposition |

## Quick exam heuristics
- Output **missing whole categories** while subagents did thorough work → **coordinator decomposition**.
- Subagent **ignores earlier context** → **context not passed in the prompt**.
- **Overlapping, duplicated results** → **poor scope partitioning**.
- **Gap noticed after results return** → **iterative refinement**: re-delegate, don't just filter.
- A **simple query** sent through every subagent → missing **dynamic selection**.

Would you like this saved as a document you can keep, or would you rather test yourself with some exam-style questions?