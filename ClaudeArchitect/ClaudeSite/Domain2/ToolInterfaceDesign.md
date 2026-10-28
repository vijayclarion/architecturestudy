# Task Statement 2.1: Tool Interface Design – Revision Notes

## 1. Core rule: descriptions drive selection

**Rule:** The tool description is **the primary mechanism** an LLM uses to choose a tool. It is not supplementary metadata.

**Purpose:** The model never sees your code. It only sees the tool's **name and description**, so that is what it uses to decide.

**The five elements of a production-grade description** (memory hook: *Does, Inputs, Examples, Edges, Boundaries*):

| # | Element | Example for `lookup_order` |
|---|---|---|
| 1 | What it does | "Retrieves status, items and delivery tracking for orders." |
| 2 | Inputs (formats, types, constraints) | "Accepts an order ID (`ORD-12345`) or customer email plus approximate date." |
| 3 | Example queries | "Where is my delivery?", "Has my order shipped?" |
| 4 | Edge cases and limitations | "Does not return account details such as address or loyalty status." |
| 5 | Boundaries against similar tools | "For account details, use `get_customer`. Use this tool for order questions even if the user gives an email." |

❌ **Anti-pattern:** minimal descriptions like "Retrieves customer information".

## 2. The misrouting problem

**Term:** *Misrouting* means the agent calls the wrong tool because two tools look too alike.

**Cause:** Overlapping or near-identical descriptions.

**Example:**
- `get_customer`: "Retrieves customer information"
- `lookup_order`: "Retrieves order details"
- Query: "Where's my delivery? My email is priya@example.com" → agent picks `get_customer` because of the email.

**Fix (small toolkit):** Expand both descriptions with all five elements, especially **boundaries**.

## 3. Count the tools before choosing a remedy

**Rule:** Diagnose first, then prescribe.

| Situation | Problem | Remedy |
|---|---|---|
| Few tools, one confusable pair | Model can't tell two tools apart | Better descriptions or renaming |
| Many tools (guide example: **18 vs 4-5**), errors everywhere | Decision complexity | **Task Statement 2.3 work:** split by role, or consolidate variants |

**Exam note:** Use the guide's 18-versus-4-5 example. Anthropic's current tool search docs put the degradation point at 30-50 tools, but the exam follows the guide.

**Large-toolkit remedies:**
- **Split by role:** each agent gets only the tools it needs.
- **Consolidate into a parameterised tool:** `search_orders_by_date`, `search_orders_by_status` and `search_orders_by_email` → one `search_orders(filter_type, value)`.

**Key line:** Rewriting 22 descriptions leaves the decision-complexity problem untouched.

## 4. Tool splitting

**Rule:** Split generic tools with broad responsibilities into purpose-specific tools.

**Purpose:** Each tool gets one narrow, clearly described job and a **defined input/output contract**.

**Example:**
`analyse_document`: "Analyses a document and returns results" → split into:
- `extract_data_points`: document + fields → structured values
- `summarise_content`: document → summary
- `verify_claim_against_source`: claim + source → supported or not, plus evidence

**Memory hook:** If the description needs **"or"**, consider splitting.

**Distinction:** Better descriptions fix a *well-scoped but poorly described* tool. Splitting fixes a tool that is *too broad*.

## 5. Tool renaming

**Rule:** Renaming removes **functional overlap at the interface level**.

**Purpose:** Makes the purpose unambiguous **without changing the implementation**.

**Example:**
`analyse_content` (actually processes web results) vs `analyse_document` → rename to **`extract_web_results`** with a web-specific description.

**Why it works:** The model sees only the name and description, so fixing the interface fixes the selection.

## 6. System prompt interactions

**Rule:** Keyword-sensitive instructions in system prompts can create **unintended tool associations that override good descriptions**.

**Example 1:** "Always check customer details before proceeding" → the model links any customer query to `get_customer`, even "Where is my parcel?"

**Example 2:** "Whenever a user mentions a document, analyse it thoroughly" → "Find news documents about EV batteries" goes to `analyse_document` instead of `extract_web_results`.

**Fix:** After updating descriptions, **always review the system prompt for conflicts**. Replace keyword triggers with case-based guidance that matches the tool boundaries:
> "If the user has uploaded a file, use `analyse_document`. For information from the web, including news, use `extract_web_results`."

**Exam signal:** "Descriptions are fixed but misrouting continues" → the answer is **the system prompt**.

## 7. Anti-patterns (likely wrong answers)

| Anti-pattern | Why it's wrong |
|---|---|
| Minimal descriptions | Give the model nothing to choose with |
| Few-shot examples to fix misrouting | Not a sanctioned remedy for misselection |
| Routing classifier as a first step | Over-engineered, and keyed wrong for large tool sets |
| Consolidating tools as a first step | Skips diagnosis; for a small confusable pair, descriptions are the fix |
| Ignoring the system prompt after updating descriptions | Keyword associations can still override the descriptions |

## 8. Diagnostic flow

1. **Count the tools.** Too many? → Split by role or consolidate (2.3).
2. **Two tools confusable?** → Expand descriptions (five elements) or rename.
3. **One tool doing several jobs?** → Split into purpose-specific tools with I/O contracts.
4. **Descriptions fixed, still misrouting?** → Check the system prompt for keyword conflicts.

## 9. Key terms

- **Tool description:** text that tells the model what a tool does and when to use it; the primary selection mechanism.
- **Misrouting:** the agent calls the wrong tool.
- **Boundaries:** explicit statements of when to use this tool versus a similar one.
- **Input/output contract:** a defined specification of what a tool accepts and returns.
- **Interface level:** the name and description the model sees, as opposed to the implementation.
- **Parameterised tool:** one tool with a parameter that replaces several near-duplicate variants.
- **Unintended tool association:** a system prompt keyword that pulls the model towards a particular tool.

**Exam answer technique:** state the **cause**, then the **fix**, then **why the fix works**.