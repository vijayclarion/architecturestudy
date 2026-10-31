Here are your revision notes for Task Statement 1.7, kept strictly to the material we covered.

## 1. The three session options

**`--resume <session-name>`**
- **Purpose:** continue a specific named session where you left off.
- **What it restores:** the conversation history, including earlier tool results. A compacted session comes back from its **last summary**.
- **Use when:** prior context is still mostly valid and files haven't changed significantly.
- **Example:** you stopped analysing a module last night, nothing has changed, and you carry on this morning.

**`fork_session`**
- **Purpose:** create **independent branches from a shared analysis baseline**.
- **How it works:** both branches inherit everything up to the branching point, then develop separately.
- **Use when:** exploring divergent approaches.
- **Examples:** comparing two refactoring strategies, or a testing strategy against a documentation strategy.
- **Not the same as resume:** resume *continues* one line, while fork *splits* it.

**Fresh start with summary injection**
- **Purpose:** begin a new session with a **structured summary of prior findings** injected into the initial context.
- **Key benefit:** the conclusions carry over, but **no stale tool results** do.
- **Use when:**
  1. tool results are stale (files changed, APIs updated), or
  2. a long session has a cluttered, degraded history.
- **Example:** you edited several files overnight, so you start fresh with a summary of yesterday's analysis.

## 2. The stale context problem

- **What it is:** after code changes, the old tool results (showing the unfixed code) stay in the resumed history.
- **Why it happens:** the agent reasons from the stale data *alongside* any new data.
- **Symptoms (exam favourite):**
  - recommending changes that have **already been made**
  - referencing code that **no longer exists**
  - giving contradictory advice
- **Remember:** stale context means old information is **kept**, not lost. Adding a note on resume doesn't remove it; the history then holds two conflicting versions.

## 3. Targeted re-analysis vs full re-exploration

- **Rule:** tell the agent about the **specific** files that changed and re-analyse only those.
- **Don't** make it re-explore the whole codebase.
- **Why:** the prior summary still covers everything that hasn't changed.
- **Example:** 3 of 50 files changed, so you re-analyse the 3 and trust the summary for the other 47.

## 4. Decision matrix (memorise)

| Situation | Choice |
|---|---|
| No files changed, continuing work | `--resume` |
| Comparing two approaches | `fork_session` |
| Exploring testing vs documentation strategy | `fork_session` |
| Files modified since last session | Fresh start + summary |
| Long session with cluttered history | Fresh start + summary |

## 5. Anti-patterns (likely wrong answers)

1. **Resuming after file changes.** This brings stale tool results back, which leads to contradictory advice.
2. **Full re-exploration when only a few files changed.** This is wasteful; use targeted re-analysis instead.
3. **Using fork when resume is enough, or resume when you need to compare approaches.** Match the tool to the purpose.
4. **Using `fork_session` to fix stale context.** The fork **inherits** the stale data, and "separate" isn't the same as "clean".

## 6. Model answer for the classic scenario

> *3 core modules changed in a 50-file codebase; the resumed agent recommends changes already made and references removed code.*

- **What went wrong:** `--resume` restored stale tool results for the core modules, and the agent reasoned from them.
- **Correct approach:** start a **fresh session**, inject a **structured summary**, name the **3 changed files** for **targeted re-analysis**, and rely on the summary for the **47 unchanged files**.

## Quick memory hooks

- **Resume = same road, nothing moved.**
- **Fork = fork in the road, try both paths.**
- **Fresh + summary = new road, carry the map, leave the old photos behind.**
- **Changed files? Re-read only what changed.**

Would you like me to put these into a document you can keep and print, or move on to some rapid-fire practice questions?