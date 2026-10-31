# Exam Notes: Task 5.4 Codebase Exploration & Context Degradation

Oct 30, 2026 · @Vijay

## Overview

Task 5.4 tests whether you can keep an agent accurate during long codebase exploration. The big idea: the threat is losing grip on specific findings, not running out of tokens.

**Purpose of the task statement:** to recognise context degradation and choose the right mitigations: scratchpad files, subagent delegation, summary injection between phases, proactive /compact, and crash recovery via state manifests.

**Running example used in these notes:** an agent exploring a large Python codebase. Early on it finds `PaymentReconciliationService` in `src/billing/reconciliation.py`, which calls `match_ledger_entries()`.

| Concept | One-line purpose |
| --- | --- |
| Context degradation | The problem: specifics fade into generic "typical patterns" |
| Scratchpad files | Primary fix: store findings in a file, read before each step |
| Subagent delegation | Keep the main agent's context clean through isolation |
| Summary injection | Stop next-phase agents starting cold |
| /compact | Proactively condense context to keep quality high |
| State manifests | Resume from the last checkpoint after a crash |

## 1. Context degradation

**Purpose:** to name the core problem. In long sessions the model stops citing what it actually found and starts describing "typical patterns".

**Important points**

- The symptom is generic answers replacing specific class names, method names and file paths.
- The cause is verbose discovery output piling up: file contents, search results, directory listings.
- The earlier findings are often still in context; they are buried, not deleted.
- It is NOT a token limit problem.

**Rules**

- A larger context window does not fix it; it only gives noise more room to accumulate.
- Switching model is not the remedy either; the fix is how knowledge is managed during the session.

**Terms to remember:** context degradation · typical patterns · verbose discovery output

**Example**

- Healthy answer: "`PaymentReconciliationService` in `src/billing/reconciliation.py` calls `match_ledger_entries()`."
- Degraded answer, an hour later: "Billing systems typically have a reconciliation service that matches transactions against ledger records."
- Exam distractor: "Move to a model with a larger context window." Reject it.

## 2. Scratchpad files

**Purpose:** the primary mitigation for context degradation. Key findings live in a file outside the conversation, so they cannot get buried.

**Important points**

- The agent writes key findings to a file: class names, file paths, dependency chains.
- It reads the scratchpad back before each subsequent exploration step.
- Knowledge held in a file is immune to context degradation.

**Rules**

- Use scratchpads deliberately from the start of a session, not as a fallback once degradation appears.
- Read before every new exploration step, not just at the end.
- Running extended sessions without a scratchpad is an anti-pattern.

**Terms to remember:** scratchpad file · primary mitigation · persist outside context

**Example** (`notes/scratchpad.md`)

```markdown
## Billing module
- PaymentReconciliationService -> src/billing/reconciliation.py
- Calls match_ledger_entries() to match payments against ledger
- Stripe client -> src/billing/stripe_client.py

## Next
- Trace where match_ledger_entries() results are consumed
```

Before exploring the reporting module, the agent reads this file, so it still cites `PaymentReconciliationService` precisely.

## 3. Subagent delegation

**Purpose:** to keep the main agent's context clean by handing specific investigations to subagents with isolated context.

**Important points**

- Typical delegated tasks: find test files, trace dependency chains, identify external integrations.
- Each subagent's verbose output stays in its own context.
- Only a condensed result returns to the main agent.

**Rules**

- The primary benefit is context isolation, not parallelisation. Parallel speed is a side benefit at most.
- Having the main agent do all exploration directly is an anti-pattern: it floods its own context.

**Terms to remember:** subagent · isolated context · context isolation · main agent / coordinator

**Example**

The main agent spawns three subagents:

1. Test finder: locate all test files.
2. Dependency tracer: trace what `match_ledger_entries()` depends on.
3. Integration explorer: list external services.

The integration explorer reads dozens of files, yet returns only: "Stripe in `src/billing/stripe_client.py`; Twilio in `src/notifications/sms.py`." The main agent never sees the raw file dumps.

## 4. Summary injection between phases

**Purpose:** to stop next-phase subagents starting cold and repeating work already done.

**Important points**

- Next-phase subagents do not inherit earlier history. They start with a blank slate.
- Without help they hit the cold-start problem and duplicate Phase 1 exploration.
- The fix: summarise Phase 1 findings, then inject the summary into Phase 2's initial context.

**Rules**

- Always summarise before spawning the next phase's subagents.
- Never assume a new subagent "remembers" previous work.
- Not summarising between phases is an anti-pattern.

**Terms to remember:** cold-start problem · summary injection · initial context · phase

**Example**

Phase 1 mapped the billing module. Phase 2 must examine how billing feeds reporting. Its subagent prompt opens with:

> Phase 1 findings: `PaymentReconciliationService` (`src/billing/reconciliation.py`) calls `match_ledger_entries()`. Stripe client in `src/billing/stripe_client.py`. Do not re-explore billing. Your task: trace how reconciliation results reach the reporting module.

Phase 2 starts warm and goes straight to the reporting module.

## 5. The /compact command

**Purpose:** a Claude Code command that reduces context usage during extended sessions, keeping context quality high.

**Important points**

- It condenses the conversation so far, cutting accumulated noise.
- Degradation begins long before the token limit, so waiting for the limit is too late.

**Rules**

- Use /compact proactively, at regular points in a long session.
- Using it only when you hit the limit is an anti-pattern.

**Terms to remember:** /compact · proactive use · context quality

**Example**

The agent finishes a burst of exploration across 30 billing files. Before starting on the reporting module, the user runs `/compact`. The noise from those 30 file reads is condensed, and the next phase begins with a lean context.

Wrong approach for comparison: carrying on until Claude Code warns that the context is nearly full, then compacting. By then the specific findings may already have faded.

## 6. Crash recovery via structured state manifests

**Purpose:** to let a multi-agent exploration resume from its last checkpoint after a crash, instead of starting from scratch.

**Important points**

- Each agent exports structured state to a manifest file: explored paths, key findings, next steps.
- State is written continuously during the run, not at the moment of failure; a crashed agent may never get to report.
- On resume, the coordinator loads the manifest and injects it into the agents' prompts.
- In practice, the coordinator assigns each subagent its own manifest path when spawning it, so it knows where to look.

**Rules**

- Write state regularly while working.
- One manifest per agent avoids agents overwriting each other.
- Lacking any crash recovery mechanism is an anti-pattern.

**Terms to remember:** state manifest · explored paths · key findings · next steps · checkpoint · coordinator

**Example** (`manifests/billing-explorer.json`)

```json
{
  "agent": "billing-explorer",
  "explored_paths": ["src/billing/reconciliation.py", "src/billing/stripe_client.py"],
  "key_findings": ["PaymentReconciliationService calls match_ledger_entries()"],
  "next_steps": ["Check src/billing/invoices/"]
}
```

The agent times out. The coordinator reads this file and starts a fresh subagent with: "Resuming. Already explored: … Findings: … Continue with: check `src/billing/invoices/`."

Flow to memorise: the coordinator assigns the path → the subagent writes → the file survives the crash → the coordinator reads and injects.

## Anti-patterns, exam traps and quick revision

**Anti-patterns and their fixes**

| Anti-pattern | Correct practice | Example |
| --- | --- | --- |
| Extended session with no scratchpad | Write findings to a scratchpad from the start; read before each step | `notes/scratchpad.md` lists `PaymentReconciliationService` and its path |
| Main agent does all exploration itself | Delegate to subagents for context isolation | Integration explorer returns a two-line summary, not 40 file dumps |
| No summary between phases | Summarise Phase 1, inject into Phase 2's initial context | Phase 2 prompt opens with "Phase 1 findings: …" |
| No crash recovery | Each agent writes a state manifest continuously | `manifests/billing-explorer.json` with explored paths and next steps |
| /compact only at the limit | Run /compact proactively | Compact after finishing the billing module, before reporting |

**Exam traps**

| If an option says… | The truth |
| --- | --- |
| "Increase the context window to fix degradation" | Wrong. Degradation is losing grip on findings, not running out of tokens |
| "Subagents' main benefit is parallel speed" | Wrong. The primary benefit is context isolation |
| "Phase 2 agents can see Phase 1's work" | Wrong. They inherit nothing; summarise and inject |
| "Use scratchpads once degradation appears" | Wrong. Use them deliberately from the start |
| "Use /compact when the context is full" | Wrong. Use it proactively for quality |
| "The agent reports its state when it crashes" | Wrong. State is saved continuously to a manifest beforehand |

**Glossary**

| Term | Meaning |
| --- | --- |
| Context degradation | Specific findings drift into generic "typical patterns" as verbose output builds up |
| Verbose discovery output | File contents, search results and directory listings that fill context |
| Scratchpad file | A file of key findings, read back before each exploration step |
| Context isolation | Each subagent works in its own context, keeping the main agent's clean |
| Cold-start problem | A new-phase agent knows nothing and repeats earlier exploration |
| Summary injection | Placing earlier findings in the next phase's initial context |
| /compact | Claude Code command that condenses context during a session |
| State manifest | File holding explored paths, key findings and next steps |
| Checkpoint | The last saved state an agent can resume from |

**Memory aid:** write it down, split it up, pass it on, compact early.

- Write it down: scratchpad files and state manifests
- Split it up: subagents for context isolation
- Pass it on: summary injection between phases
- Compact early: proactive /compact
