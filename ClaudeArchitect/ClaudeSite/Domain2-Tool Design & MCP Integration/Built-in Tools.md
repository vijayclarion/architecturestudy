Here's a revision sheet for **Task Statement 2.5: Built-in Tools**.

---

## 1. The tools at a glance

| Tool | Purpose | Searches / acts on | Changes files? |
|---|---|---|---|
| **Grep** | Find text *inside* files | File **contents** | No |
| **Glob** | Find files by *name* | File **paths** | No |
| **Read** | Load a file into context | One whole file | No |
| **Edit** | Targeted change to specific text | Unique text match | Yes |
| **Write** | Output a complete file | Whole file | Yes |

**Golden rule:** *Grep finds what is INSIDE files; Glob finds files by their NAMES.*

---

## 2. Grep vs Glob

**Use Grep for content:**
- Function callers: `processRefund(`
- Error messages: `"Payment declined"`
- Import statements: `from 'lodash'`
- Variable assignments: `MAX_RETRIES =`

**Use Glob for paths:**
- Test files: `**/*.test.tsx`
- Config files: `**/config.*`
- All MDX files in a directory: `docs/**/*.mdx`

**Trap to avoid:** using Glob `**/*lodash*` to find files that import lodash. That only matches file *names*. Use Grep instead.

---

## 3. Edit vs Read + Write

**Rule:** Try **Edit first**, and fall back to **Read + Write** only when Edit fails.

- **Edit** changes only the text you identify. It's fast, precise, and uses little context.
  - *Example:* `const MAX_RETRIES = 3` → `const MAX_RETRIES = 5`. The text is unique, so Edit works.
- **Why Edit fails:** the text is **not unique**, so it appears more than once.
  - *Example:* `retries: 3` appears in several config objects, so Edit can't tell which one you mean.
- **Fallback:** **Read** the full file, then **Write** the complete modified file.

**Trap to avoid:** defaulting to Read + Write for every change. It wastes context tokens, and the exam penalises it.

*Current practice (not the exam answer):* the live tools reference also suggests widening the match with surrounding context, or using `replace_all: true`.

---

## 4. Incremental codebase understanding

**Rule:** Never read all files upfront, because it's a **context-budget killer**.

**The four-step pattern:**
1. **Grep** to find entry points, such as function names or error messages.
2. **Read** those files and follow their imports.
3. **Grep again** if you find wrappers or re-exports.
4. **Read** only what earlier steps justify.

**Key principle:** each file you Read should be justified by the previous step.

*Example:* to understand `processRefund` in a 400-file project, Grep for `processRefund`, Read `refunds/core.ts`, then follow only its imports.

---

## 5. Tracing usage across wrapper modules

**Problem:** a function can be re-exported under a new name. A Grep for the original name then misses the **indirect consumers**.

*Example:*
```ts
// refunds/core.ts
export function processRefund() { ... }

// payments/index.ts (wrapper / barrel file)
export { processRefund as issueRefund } from '../refunds/core';

// checkout.ts
issueRefund(order);   // ← missed by Grep "processRefund"
```

**Method:**
1. **Grep** for the definition (`processRefund`).
2. **Read** the defining and wrapper files to find the exported names (`issueRefund`).
3. **Grep** for **each exported name**.
4. If there's a **barrel file**, **Grep** for imports of that module (`from '../payments'`).

**Remember:** follow the *names*, not just the original name.

---

## 6. The deprecation scenario: Grep, then Glob

**Task:** find all callers of deprecated `legacyAuth()` **and** their test files.

1. **Grep** `legacyAuth`, plus any aliases. This content search gives you the callers, such as `auth/login.ts` and `billing/invoice.ts`.
2. **Glob** for matching tests, such as `**/login.test.*` and `**/invoice.test.*`. This path search gives you the test files.

**Why this order:** you need the caller names before you know which test files to look for.

**Why Grep alone isn't enough:** `login.test.ts` tests `login()` and may never contain the text `legacyAuth`. It still needs updating, though.

---

## 7. Key terms

- **Content search:** looking inside files (Grep).
- **Path matching:** matching file names, extensions and directories (Glob).
- **Unique text matching:** how Edit locates what to change. It fails if the text isn't unique.
- **Context budget / context tokens:** the limited space for what Claude has loaded. Reading unnecessary files wastes it.
- **Entry point:** where you start tracing, such as a function name or error message.
- **Re-export / alias:** making a function available under a different name or from a different module.
- **Barrel file:** a module (often `index.ts`) that re-exports things from other modules.
- **Indirect consumer:** code that uses a function through a wrapper or alias, not its original name.

---

## 8. Anti-patterns the exam tests

1. ❌ Glob to find function callers → ✅ Grep
2. ❌ Grep to find files by extension or name → ✅ Glob
3. ❌ Reading every file upfront → ✅ Grep → Read incrementally
4. ❌ Read + Write for every change → ✅ Edit first
5. ❌ Grep only for the original function name → ✅ trace aliases and barrel files

---

**Memory hooks:**
- **"Grep = Guts, Glob = Labels"**: Grep looks in a file's guts; Glob reads its labels (names).
- **"Edit first, rewrite last."**
- **"Justify every Read."**
- **"Follow the names."**
- **"Callers then tests: Grep then Glob."**

Would you like me to turn this into a doc you can keep and print, or try a few practice questions on your weaker areas?