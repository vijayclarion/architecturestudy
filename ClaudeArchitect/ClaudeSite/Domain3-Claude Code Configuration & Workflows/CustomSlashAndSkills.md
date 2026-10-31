# Exam Revision Sheet: Task Statement 3.2, Custom Slash Commands and Skills

## 1. Key terms

| Term | Meaning |
|---|---|
| **Slash command** | A reusable instruction you run by typing `/name` |
| **Skill** | A directory containing a `SKILL.md` file that creates a `/command` |
| **Command file** | A flat `.md` file in `.claude/commands/` that creates a `/command` |
| **Project-scoped** | Stored in the repository and shared via git |
| **User-scoped** | Stored in your home directory (`~`), personal only |
| **Frontmatter** | Optional YAML config at the top of `SKILL.md`, between `---` lines |
| **CLAUDE.md** | Always-loaded file of universal standards |
| **On demand** | Loaded only when invoked or when Claude matches the task |

---

## 2. Unified Skills system

**Rule:** Skills and commands are merged. Both create **identical** `/commands`; only the file structure differs.

| Type | Structure | Name comes from | Example | Result |
|---|---|---|---|---|
| Skill | Directory + `SKILL.md` | **Directory name** | `.claude/skills/deploy/SKILL.md` | `/deploy` ✅ |
| Command | Flat `.md` file | **Filename** | `.claude/commands/deploy.md` | `/deploy` ✅ |
| Wrong | Flat file in skills folder | — | `.claude/skills/deploy.md` | No command ❌ |

- `.claude/skills/` is the **canonical** location.
- `.claude/commands/` is still supported for **backward compatibility**.
- Skills are **written by people**; Claude does not create them.

**Memory hook:** *Skills live in folders, commands live in files.*

---

## 3. Scoping: who gets the command?

| | Project-scoped | User-scoped |
|---|---|---|
| **Paths** | `.claude/skills/<name>/SKILL.md`<br>`.claude/commands/<name>.md` | `~/.claude/skills/<name>/SKILL.md`<br>`~/.claude/commands/<name>.md` |
| **Version-controlled** | Yes | No |
| **Shared with team** | Yes, on clone or pull | No, only you |
| **Purpose** | Team-wide standard workflows | Personal productivity |
| **Examples** | `/review`, `/deploy-check`, `/lint` | `/my-notes`, `/brainstorm` |

**Rule:** If **everyone** needs it, use a **project-scoped** path.

**Memory hook:** *`~` means "me".* If the path has a tilde, it is personal.

**Personal customisation:** To tweak a team skill for yourself, create a variant in `~/.claude/skills/` **with a different name**, for example `~/.claude/skills/my-review/SKILL.md`. This way you do not affect teammates.

---

## 4. Skills vs CLAUDE.md (most tested)

| | CLAUDE.md | Skills |
|---|---|---|
| **Loading** | Always, automatically, every session | On demand |
| **Content** | Universal standards | Task-specific workflows |
| **Example** | "Use British English", "Use pytest for tests", naming conventions | Step-by-step PR review, deployment checklist |

**Rules:**
- ❌ **Never** put task-specific procedures in CLAUDE.md. They waste context in every session.
- ❌ **Never** put universal standards in skills. They get missed when nobody invokes the skill.
- Conventions that should apply **automatically** belong in **CLAUDE.md** or `.claude/rules/`.

**Memory hook:** *CLAUDE.md is the rulebook you always carry. A skill is a tool you pick up when needed.*

---

## 5. Frontmatter options

```yaml
---
context: fork
allowed-tools: Read, Grep, Glob
argument-hint: [issue-number]
---
```

### `context: fork`
- **Purpose:** Runs the skill in an **isolated sub-agent context**.
- **Benefit:** Verbose output stays contained and the main conversation stays clean.
- **Use for:** codebase analysis, brainstorming, exploratory or noisy tasks.
- **Without it:** output fills the context window and **degrades later responses**.
- **Example:** `/analyse-codebase` scanning hundreds of files should use `context: fork`.
- **Memory hook:** *Do the messy work in another room.*

### `allowed-tools`
- **Exam answer:** **Restricts** the skill's tool access to the listed tools.
- **Current Claude Code (checked 6 October 2026):** pre-approves the listed tools so they run without a permission prompt. `disallowed-tools` is the actual security boundary.
- **Principle:** A read-only skill should **not** use `Write` or `Bash`.
- **Example:** A `/audit` skill that only inspects code: `allowed-tools: Read, Grep, Glob`.

### `argument-hint`
- **Exam answer:** **Prompts the developer** for required parameters when the skill is invoked without arguments.
- **Current Claude Code (checked 6 October 2026):** shown during autocomplete to indicate the expected arguments.
- **Audience:** the **human** developer, not the agent.
- **Example:** `argument-hint: [issue-number]` means the developer sees `/fix-issue [issue-number]` and types `/fix-issue 1234`.

---

## 6. Anti-patterns

| ❌ Anti-pattern | ✅ Fix |
|---|---|
| Task-specific procedures in CLAUDE.md | Move them into a skill |
| Universal standards in a skill | Move them into CLAUDE.md |
| Verbose or exploratory skill without `context: fork` | Add `context: fork` |
| Team command in `~/.claude/commands/` or `~/.claude/skills/` | Move it to `.claude/skills/` or `.claude/commands/` |
| Read-only skill with `Write` or `Bash` access | List safe tools only in `allowed-tools` |

---

## 7. Exam traps: quick-fire decision guide

| If the question says… | The answer is… |
|---|---|
| "available to every developer on the team" | Project-scoped path (`.claude/...`) |
| "personal, should not affect teammates" | User-scoped path (`~/.claude/...`) |
| "applied automatically to every session" | CLAUDE.md or `.claude/rules/` |
| "only needed for a specific task" | A skill |
| "output clutters the conversation" or "later responses get worse" | `context: fork` |
| "skill should only read, never modify" | `allowed-tools` with read-only tools |
| "developer forgets to pass the issue number" | `argument-hint` |
| `.claude/skills/review.md` | Invalid: it must be `.claude/skills/review/SKILL.md` |
| "Which is preferred, skills or commands?" | `.claude/skills/` is canonical; commands are kept for backward compatibility |

---

## 8. Worked scenario

**Team needs `/review`; you want a personal, verbose `/brainstorm`.**

```
my-repo/.claude/skills/review/SKILL.md        ← project-scoped, shared via git
~/.claude/skills/brainstorm/SKILL.md          ← user-scoped, personal
```

`brainstorm/SKILL.md`:
```yaml
---
context: fork
---
Generate and explore ideas for...
```

---

## 9. Your personal watch-list

From our session, double-check these two:
1. Skills are **authored files**, not created by Claude at runtime.
2. `argument-hint` is for the **developer**, not the agent.

Good luck with the exam. If you would like, I can turn this into a doc you can keep, or give you a short multiple-choice quiz to test yourself.