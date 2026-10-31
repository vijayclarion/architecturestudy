# Task 3.1 Revision Notes: CLAUDE.md Hierarchy, Scoping and Modular Organisation

## 1. Key terms

| Term | Meaning |
|---|---|
| **CLAUDE.md** | A Markdown file of instructions that Claude Code reads automatically |
| **User-level** | Personal config in your home directory, applied to you only |
| **Project-level** | Team config inside the repo, shared via git |
| **Directory-level** | A CLAUDE.md inside a subdirectory, applied when working there |
| **@ import** | `@path/to/file.md` on its own line; pulls another file into CLAUDE.md |
| **`.claude/rules/`** | A folder of topic-specific instruction files |
| **Path scoping** | Optional YAML frontmatter in a rules file that limits it to matching files |
| **`/memory`** | Diagnostic command showing which memory files are loaded |

---

## 2. The three levels: purpose, rule and example

### User-level: `~/.claude/CLAUDE.md`
- **Purpose:** personal preferences.
- **Rule:** **not** version-controlled and **not** shared via git. New team members never receive it.
- **Example:**
  ```markdown
  Keep explanations brief. Show diffs rather than full files.
  ```
- **Memory hook:** *"User = just me."*

### Project-level: `.claude/CLAUDE.md` **or** root `CLAUDE.md`
- **Purpose:** team-wide standards such as naming conventions, testing requirements and architecture decisions.
- **Rule:** version-controlled and shared through git. Everyone who clones gets it. **Both locations are valid.**
- **Example:**
  ```markdown
  Use camelCase for variables, PascalCase for classes.
  Every API handler must have an integration test.
  ```
- **Memory hook:** *"Project = whole team, via git."*

### Directory-level: `packages/payments/CLAUDE.md`
- **Purpose:** package-specific conventions that **differ** from the root.
- **Rule:** applies when working in **that specific directory**, not necessarily all nested subdirectories.
- **Example:**
  ```markdown
  All monetary values use the Decimal type, never float.
  ```
- **Memory hook:** *"Directory = this folder's exceptions."*

---

## 3. @ import syntax

- **Purpose:** modular organisation and **no duplication**. Each package imports only the standards it needs.
- **Rule:** `@` followed by a path, on its own line. **There is no `@import` keyword.**
- **Example** (in `packages/payments/CLAUDE.md`):
  ```markdown
  @docs/standards/testing.md
  @docs/standards/api-conventions.md

  Payments-specific: always use Decimal for money.
  ```
- ✅ `@docs/testing.md`  ❌ `@import docs/testing.md`
- **Memory hook:** *"At-sign and path, nothing in between."*

---

## 4. `.claude/rules/` directory

- **Purpose:** an alternative to one massive, monolithic CLAUDE.md; standards split by topic.
- **Rule:** files contain **instructions (guidance)**, not checks or validations. Each can optionally have YAML frontmatter with path scoping.
- **Example layout:**
  ```
  .claude/rules/
  ├── testing.md
  ├── api-conventions.md
  └── deployment.md
  ```
- **Example with path scoping** (`testing.md`):
  ```markdown
  ---
  paths:
    - "**/*.test.ts"
  ---
  Use describe/it blocks. One assertion focus per test.
  ```
- **When to use:** CLAUDE.md has grown large, or a convention applies to files scattered across many directories.

---

## 5. `/memory` command

- **Purpose:** a diagnostic tool to verify which memory files are loaded in the current session.
- **Does:** shows loaded memory files and helps debug inconsistent behaviour across sessions.
- **Does NOT:** trigger loading. Configuration loads **automatically based on location**.
- **Example:** Claude ignores the testing rules in one session → run `/memory` → see whether the expected file appears.
- **Current tooling (October 2026):** `/context` shows what loaded at launch, but **`/memory` is the exam answer**.
- **Memory hook:** *"/memory verifies; location determines."*

---

## 6. Anti-patterns (spot these in exam options)

| Anti-pattern | Correct approach |
|---|---|
| Team standards in user-level config | Move them to project-level and commit |
| One massive monolithic CLAUDE.md | Split into `.claude/rules/` topic files |
| Not using `/memory` when behaviour is inconsistent | Run `/memory` to verify loaded files |
| Same instructions copied into many directory CLAUDE.md files | One shared file, imported with `@` |

---

## 7. Exam traps

1. **New joiner doesn't get the conventions** → the instructions are in **user-level** config, not project-level. *(Not randomness, not model variance, not the joiner's own file.)*
2. **Deleting the new joiner's user file fixes nothing.** The fix is to move the conventions into project-level config and commit.
3. **`.claude/CLAUDE.md` and root `CLAUDE.md`** are **both** valid project-level locations.
4. **Directory-level** applies to work in that directory, not automatically to all nested subdirectories.
5. **`/memory` does not load anything.**
6. **`@import` is wrong.** The correct form is `@path`.
7. **Rules files are instructions**, not validations or linters.
8. **Conventions across 50+ directories** → one path-scoped `.claude/rules/` file, not 50 directory-level CLAUDE.md files.

---

## 8. Quick decision guide

```
Personal preference only?              → User-level  (~/.claude/CLAUDE.md)
Whole team needs it?                   → Project-level (.claude/CLAUDE.md or root CLAUDE.md)
One package differs from the root?     → Directory-level CLAUDE.md
Same standard needed in many packages? → Shared file + @ import
CLAUDE.md getting huge?                → Split into .claude/rules/
Applies to file types across the repo? → Path-scoped rules file
Behaviour inconsistent?                → /memory to verify
```

---

## 9. One-line summary

> **User = me (not in git). Project = team (in git). Directory = this folder. `@path` to share without duplicating. `rules/` to split by topic. `/memory` checks; it never loads.**

Would you like me to turn these notes into a doc you can keep and print, or test you with some exam-style multiple-choice questions?