Here's a revision sheet for Task Statement 3.3, kept strictly to exam scope.

## 1. Key terms

| Term | Meaning | Example |
|---|---|---|
| **`.claude/rules/`** | Directory holding rule files (Markdown) | `.claude/rules/testing.md` |
| **YAML frontmatter** | Settings block at the top of a rule file, between `---` lines | See section 2 |
| **`paths` field** | List of glob patterns deciding when the rule loads | `paths: ["**/*.test.tsx"]` |
| **Glob pattern** | Wildcard pattern matching file paths | `terraform/**/*` |
| **Path-scoped rule** | A rule with `paths`, loaded only for matching files | Testing rule loads only for test files |
| **Root `CLAUDE.md`** | Project-wide memory file, loads every session | Repo root `CLAUDE.md` |
| **Directory-level `CLAUDE.md`** | `CLAUDE.md` inside a subdirectory, applies to that directory only | `src/api/CLAUDE.md` |
| **Skill** | On-demand, task-style workflow | A "release checklist" procedure |

## 2. Rule file syntax

```markdown
---
paths: ["**/*.test.tsx"]
---
# Testing conventions
- Use React Testing Library, not Enzyme
- One describe block per component
```

Multiple patterns are allowed:

```yaml
paths: ["terraform/**/*", "**/*.tf"]
```

## 3. Glob patterns

| Pattern | Matches | Does not match |
|---|---|---|
| `**/*.test.tsx` | `Button.test.tsx`, `src/forms/Input.test.tsx` (any depth) | `mytest.tsx`, `Button.tsx` |
| `terraform/**/*` | Every file under `terraform/`, any depth | `infra/main.tf` |
| `api/**/*` | Every file under `api/`, any depth | `src/api-client.ts` |

- **`**/`** means any number of directories, including none.
- **`*`** means any characters within a filename.
- Glob patterns match **across the entire codebase**. This is the key advantage.

## 4. The core rules to remember

1. **Rules load by file path, not topic.** The trigger is a file matching the glob.
2. **Rules load only when editing matching files.** No match, no tokens.
3. **Root `CLAUDE.md` always loads**, every session, whatever you edit.
4. **Directory-level `CLAUDE.md` covers one directory only.**
5. **Rules are always-in-context conventions; skills are on-demand workflows.**
6. **Verify loaded memory with `/memory`** (exam answer).

## 5. Comparisons with examples

**Path-specific rules vs directory-level `CLAUDE.md`**
Scenario: test files co-located with source across 60 directories.
- Directory-level: 60 `CLAUDE.md` files, all duplicated, prone to drift, and new directories need new files. ❌
- Path-specific: one rule with `paths: ["**/*.test.tsx"]`. ✅

**Path-specific rules vs root `CLAUDE.md`**
Scenario: Terraform conventions, but you're editing React components.
- Root `CLAUDE.md`: Terraform conventions load anyway, wasting tokens. ❌
- Path-specific: `paths: ["terraform/**/*"]` loads nothing while you edit React. ✅

**Path-specific rules vs skills**
- "Every SQL migration must include a rollback section" is a background convention for a file type, so use a **rule**. ✅
- "Run our 6-step release process" is a multi-step procedure, so use a **skill**. ✅

## 6. Token efficiency example

Conventions: Terraform 2,000, React testing 1,500, API design 1,000 tokens. Session: editing only `Button.tsx`.

| Approach | Tokens loaded |
|---|---|
| All in root `CLAUDE.md` | 4,500 |
| Path-scoped rules | 0 (nothing matches) |

## 7. Anti-patterns

- Directory-level `CLAUDE.md` for conventions spanning many directories
- File-type-specific conventions in root `CLAUDE.md`
- Not using glob patterns when test or config files are scattered
- A `CLAUDE.md` in every directory

## 8. Exam traps

1. **"50+ directories" plus co-located tests** means a path-specific rule with a glob. Never one `CLAUDE.md` per directory.
2. **"Most token-efficient"** for file-type conventions means path-scoped rules, not root `CLAUDE.md`.
3. **"Automatically apply conventions when editing matching files"** means rules, not skills.
4. **Check the glob actually matches.** A file *related* to tests (`Button.tsx`) does not match `**/*.test.tsx`.
5. **"Verify which rules are loaded"** means `/memory`.

## 9. Quick decision guide

- Applies to **every** session? Root `CLAUDE.md`
- Applies to **one** directory only? Directory-level `CLAUDE.md`
- Applies to a **file type spread across many directories**? Path-specific rule with a glob
- A **procedure to run** on request? Skill

If you'd like this as a document you can keep, print or export, I can turn it into one.