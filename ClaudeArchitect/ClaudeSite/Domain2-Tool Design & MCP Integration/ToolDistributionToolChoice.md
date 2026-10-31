# Task Statement 2.3: Tool Distribution and Tool Choice, Revision Sheet

## 1. Tool overload

**Rule:** Keep each agent to **4-5 tools**, scoped to its role. The exam contrasts this with **18 tools** on one agent.

**Why:** Every extra tool increases **decision complexity**, so selection reliability drops. Agents given tools **outside their specialisation tend to misuse them**.

**Example:**
- ❌ A synthesis agent with `search_web` starts searching instead of synthesising.
- ❌ A web search agent with document analysis tools.
- ✅ Each agent only has the tools for its own job.

*Real-world note: Anthropic's current docs say accuracy degrades beyond 30-50 tools. On the exam, use 18 vs 4-5.*

---

## 2. Fixing an oversized toolkit: split or consolidate

| Situation | Fix | Example |
|---|---|---|
| Tools are **genuinely different jobs** | **Split** across role-scoped agents | `query_database`, `export_report`, `send_notification` go to different agents |
| Tools are **variations on one job**, same shape | **Consolidate** into one parameterised tool | 19 transformation tools become `transform_data(operation="convert_to_csv")` |

**Rules:**
- Consolidation reduces the number of choices; it gives **no new powers**, so least privilege is intact.
- ❌ **Never the fix:** moving tools to a second MCP server. Server boundaries are **invisible to the model**, so 22 tools across two servers is still 22 tools.

---

## 3. `tool_choice` modes

| Mode | Must call a tool? | Who picks? | Use when | Risk it removes |
|---|---|---|---|---|
| `"auto"` (default) | No | Model | General operation | — |
| `"any"` | **Yes** | Model | Guaranteed structured output from **one of several** schemas | Plain-text reply instead of a tool call |
| `{type: "tool", name: "X"}` | **Yes** | **You** | **Mandatory first step**; cannot be skipped or reordered | Wrong tool, or skipped step |

**Memory hook:**
- `"auto"` risk → **no** tool call
- `"any"` risk → **wrong** tool call
- forced → exactly one tool, guaranteed

**Examples:**
- Extraction pipeline receiving invoices, contracts and receipts, each with its own schema → **`"any"`** (forcing one schema would apply the wrong one to the other types).
- Metadata must be extracted before any analysis → **forced** `{type: "tool", name: "extract_metadata"}`.

**Anti-pattern:** using `"auto"` when structured output is required.

*Real-world note (October 2026): Opus 5.5, Sonnet 5.5, Fable 5.1 and Mythos 5.1 reject `"any"` and forced selection with a 400 error, as does manual extended thinking on any model; the docs recommend `"auto"` plus strict tool use. The exam still tests all three modes.*

---

## 4. Scoped cross-role tools

**Problem:** Routing every simple request through the coordinator adds **2-3 round trips** and up to **40% latency**.

**Fix:** Give the agent a **scoped, constrained** version of another role's capability for the simple cases only.

**Example:** Synthesis agent gets `verify_fact` (scoped):
- **85%** simple, single-source lookups → handled directly
- **15%** complex, multi-source verifications → still routed through the coordinator

**Key word:** *scoped*. It is a constrained slice, not the other role's full toolkit (not `search_web`).

**Anti-pattern:** routing all simple operations through the coordinator.

---

## 5. Constrained tools instead of generic ones

**Principle:** **Least privilege** applied to tool design: each tool does exactly what the agent needs, nothing more.

**Example:**
- ❌ `fetch_url`: fetches anything from anywhere
- ✅ `load_document`: validates and only accepts document URLs

**Remember:** the benefit comes from the **constraint/validation**, not the name. Renaming `fetch_url` without adding validation achieves nothing.

**Anti-pattern:** using generic tools when constrained alternatives exist.

---

## 6. Reference tool distribution

| Agent | Tools |
|---|---|
| Web search | `search_web`, `fetch_page`, `extract_links`, `save_snippet` |
| Document analysis | `extract_metadata`, `extract_data_points`, `summarise_content`, `verify_claim` |
| Synthesis | `compile_report`, `verify_fact` (scoped), `format_citation`, `assess_coverage` |
| Coordinator | `Task`, `review_output`, `request_revision` |

**Rule:** The coordinator **manages** work (delegate, review, request revision); it doesn't **produce** it. `format_citation` belongs to **synthesis**, not the coordinator.

---

## 7. Anti-patterns checklist

1. One agent with 18+ tools
2. Agents holding tools outside their specialisation
3. Routing all simple operations through the coordinator
4. Generic tools where constrained alternatives exist
5. `tool_choice: "auto"` when structured output is required
6. Splitting tools across MCP servers to "reduce" the toolkit

---

## 8. Quick exam decision guide

- "Agent keeps picking the wrong tool / has too many tools" → **reduce to 4-5, role-scoped** (split or consolidate)
- "Many near-identical tools" → **consolidate** with a parameter
- "Must always get structured output, several document types" → **`"any"`**
- "Step X must always happen first" → **forced tool**
- "Too much latency from coordinator round trips" → **scoped cross-role tool**
- "Agent has broad access it doesn't need" → **constrained tool / least privilege**

Would you like this saved as a document you can keep and print, or shall we move on to some exam-style practice questions?