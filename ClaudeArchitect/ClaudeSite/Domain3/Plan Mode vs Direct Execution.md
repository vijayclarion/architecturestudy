# Exam Notes: Plan Mode vs Direct Execution (Task 3.4)

Oct 26, 2026 · @VIjay

## 1. The golden rule: ambiguity, not difficulty

Choose the mode by asking one question: **do I already know the correct approach?** If yes, use direct execution. If no (there are design choices to make), use plan mode.

**Example A (hard but clear → direct execution):** a tricky race condition in one function. The stack trace points to the exact line and the cause is known. It is difficult, but there is nothing to decide.

**Example B (easy-looking but ambiguous → plan mode):** "Add notifications to the app." It sounds simple, but it could use email, push or in-app messages, and it touches several modules. You must choose an approach first.

Exam tip: questions often make a task *sound* hard to tempt you into plan mode. Ignore the difficulty; look for ambiguity, multiple valid approaches and scope.

## 2. Key terms

| Term | Meaning | Example |
| --- | --- | --- |
| Plan mode | Claude reads and analyses the codebase **without modifying files**, then designs an approach before any changes | Restructuring a monolith into microservices |
| Direct execution | Claude makes the change straight away; no exploration or design phase | Fixing a single-file bug with a clear stack trace |
| Explore subagent | A separate agent that does verbose discovery in its own context and **returns a summary** to the main conversation | Mapping every file that imports a payments library, returning a one-paragraph summary |
| Context window exhaustion | The main conversation fills up with verbose output, so later responses degrade | Pasting hundreds of grep results and file listings into the main chat before designing |
| Hybrid approach | Plan mode to investigate and decide, **then** direct execution to implement | Plan a 30-file library migration strategy, then apply it file by file |
| Multiple valid approaches | More than one reasonable way to solve the task, so a design choice is needed | Choosing between REST, message queue or gRPC for a new integration |
| Architectural decision | A choice about system structure that affects many parts of the code | Deciding where to draw service boundaries in a monolith |
| Well-understood change | The correct fix is already known and the scope is limited | Adding a date validation conditional to one form handler |

## 3. When to use each mode

Use **plan mode** when any of these signals appear; use **direct execution** when the approach is known and the scope is small.

| Signal | Mode | Example |
| --- | --- | --- |
| Large-scale change | Plan | Restructuring a monolith into microservices |
| Multi-file modification (10+ files) with multiple valid approaches | Plan | Library migration affecting 45+ files |
| Architectural decision | Plan | Choosing between integration approaches with different infrastructure requirements |
| Need to explore an unfamiliar codebase safely first | Plan | Understanding how authentication flows through an inherited service before changing it |
| Single file, clear cause | Direct | Bug fix with a stack trace pointing to one function |
| Small, well-defined addition | Direct | Adding a date validation conditional |
| Correct approach already known | Direct | Renaming a misspelt variable in one module |

Rule to remember: plan mode **does not change files**. It is safe exploration and design before you commit.

## 4. The Explore subagent

Its purpose is **protecting the main conversation's context window**, not just searching. It does verbose discovery in isolation and hands back a summary.

- **Problem:** file reads, directory listings and grep output are long. In the main conversation they fill the context window and later responses degrade.
- **Solution:** the Explore subagent runs the discovery in its own context and **returns only a summary**.
- **When:** multi-phase tasks with heavy exploration, especially in large or unfamiliar codebases.

**Example:** before splitting a monolith, you need to know which modules depend on the orders table. Without the subagent, 200 search results land in the main chat. With it, the main chat receives: "Orders table is used by 6 modules: billing, shipping, invoicing, reports, admin and the API layer." The planning work that follows still has plenty of context.

Memory hook: **Explore = isolate + summarise = save context.**

## 5. The hybrid approach: plan THEN execute

Plan and direct execution are **not mutually exclusive**. Complex tasks usually use both, in sequence: **plan THEN direct, not plan OR direct.**

1. **Plan mode:** explore the codebase (with the Explore subagent if output is verbose), compare valid approaches, choose a design. No files change.
2. **Switch to direct execution:** the ambiguity is now gone, so implement the chosen design.

**Example:** migrating from one HTTP library to another across 30 files.

1. Plan mode: find every usage, note the patterns (simple GET calls, retries, interceptors), and decide on a wrapper-first migration strategy.
2. Direct execution: create the wrapper, then update each file following the agreed pattern.

Rule: use the hybrid approach whenever a task needs analysis or a decision before the changes are made.

## 6. Exam traps

These four traps are the most likely to appear as scenario questions.

| Trap | Example scenario | Correct answer |
| --- | --- | --- |
| Multi-file changes (10+ files) with multiple valid approaches require plan mode | Migrate a logging library used in 30 files; several migration strategies are possible | **Plan mode** (then execute: hybrid) |
| A single-file bug fix with a clear stack trace does NOT need plan mode | A null-pointer error; the stack trace points to line 42 of one function and the fix is obvious | **Direct execution** |
| Verbose discovery in the main conversation fills the context window | You need to trace every caller of a function across a large repository | **Use the Explore subagent** so only a summary returns |
| When complexity is stated in the requirements, choose plan mode upfront | Requirements say "restructure the monolith into microservices"; a colleague says start direct and switch if it gets complicated | **Plan mode from the start**; the colleague is wrong because the complexity is already known |

The classic three-task question:

- Monolith to microservices → **plan mode** (architectural decisions, multiple valid approaches)
- Single-function bug with a clear stack trace → **direct execution** (known cause, limited scope)
- Library migration across 30 files → **plan mode** (multi-file, strategy to choose), then execute

## 7. Anti-patterns (wrong answers to spot)

| Anti-pattern | Example | Fix |
| --- | --- | --- |
| Direct execution for architectural decisions without exploring first | Immediately editing files to split out a "users" service with no analysis of dependencies | Plan mode first, then execute |
| Plan mode for a simple, well-understood bug fix | Entering plan mode to fix a typo in an error message | Direct execution; planning wastes time |
| Not using the Explore subagent for verbose discovery | Running dozens of searches in the main chat until responses get worse | Delegate discovery to the Explore subagent |
| Treating plan and direct as mutually exclusive | Insisting the whole migration must be done in plan mode, or entirely in direct mode | Hybrid: plan THEN direct |
| Starting direct and switching to plan only when complexity emerges, when it was already stated | "Restructure the monolith" begun with direct edits, switching to plan halfway | Choose plan mode upfront |

## 8. Exam-day decision checklist

Work through these in order on any scenario question; stop at the first match.

1. **Do the requirements already state complexity** (restructure, migrate, redesign)? → Plan mode **upfront**.
2. **Are there multiple valid approaches or an architectural decision?** → Plan mode.
3. **Does it touch many files (10+)?** → Plan mode, usually followed by direct execution (hybrid).
4. **Is the cause known, the scope one file or function, and the fix clear?** → Direct execution, even if it is hard.
5. **Will exploration produce lots of output?** → Add the Explore subagent so only summaries reach the main conversation.

**Worked example:** "Users report a crash when saving a profile; the stack trace points to `validateDate()` in one file." Step 1: no stated complexity. Step 2: one obvious fix. Step 3: one file. Step 4: yes → **direct execution**.

Five phrases to memorise:

- Ambiguity, not difficulty
- Plan mode reads and analyses; it does not modify files
- Explore = isolate + summarise = save context
- Plan THEN direct, not plan OR direct
- Complexity stated upfront → plan mode upfront
