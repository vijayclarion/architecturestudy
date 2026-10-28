# Structured Output with tool\_use: Exam Notes (Task 4.3)

Oct 27, 2026 · @VIjay

## 1. Reliability hierarchy

**Rule:** when schema compliance matters, use `tool_use` with a JSON schema, not prompt-based JSON. `tool_use` eliminates JSON syntax errors entirely.

| Rank | Method | Syntax errors possible? |
| --- | --- | --- |
| 1 (most reliable) | `tool_use` + JSON schema | No |
| 2 | Prompt-based JSON ("Respond only with valid JSON…") | Yes |

**Why prompt-based JSON fails:** an instruction is a request, not a guarantee. At volume, some outputs break your parser.

**Examples of malformed prompt-based output:**

- Missing bracket: `{"items": [{"amount": 200}` (array never closed)
- Trailing comma: `{"total": 450.00,}`
- Unquoted key: `{supplier: "Acme Ltd"}`
- Prose wrapper: `Here is the data: {"total": 450}`

**Example of the reliable approach:** define an `extract_invoice` tool whose `input_schema` lists `supplier_name`, `invoice_date`, `line_items` and `total`. Claude returns the data as the tool call's input, already parsed and schema-compliant.

**Exam trap:** "Prompting for JSON is enough for a production pipeline" is wrong.

## 2. tool\_choice modes

**Rule:** the mode decides whether Claude may skip the tools and reply in text, and who picks the tool.

| Mode | Syntax | Must call a tool? | Who picks the tool? | Use when |
| --- | --- | --- | --- | --- |
| auto (default) | `{"type": "auto"}` | No: may reply in text | Model | Tool use is genuinely optional |
| any | `{"type": "any"}` | Yes | Model, from tools provided | Structured output needed, document type unknown |
| forced | `{"type": "tool", "name": "extract_metadata"}` | Yes | You name it | A specific step must happen, e.g. mandatory metadata extraction first |

**Decision rule:**

1. Must you get structured output every time? No → `auto`.
2. Yes, and Claude should choose between several tools → `any`.
3. Yes, and one specific tool must run → forced named tool.

**Examples:**

- **auto failure:** an invoice pipeline with one tool sometimes gets back "This appears to be a delivery note." The pipeline breaks.
- **any:** invoices, delivery notes and purchase orders arrive mixed. Tools `extract_invoice`, `extract_delivery_note`, `extract_purchase_order` plus `tool_choice: {"type": "any"}` always returns one tool call. Good descriptions decide which tool; `any` guarantees a tool.
- **forced:** every document must have sender, date and reference captured first, so force `extract_metadata` before type-specific extraction.

**Analogy:** auto = fill in a form or just tell me; any = you must fill in a form, pick which; forced = you must fill in this form.

**Exam trap:** using `auto` when guaranteed structured output is needed. Good descriptions do not make `auto` safe.

**Real-world note (as of 6 October 2026):** Claude Opus 5.5, Sonnet 5.5, Fable 5.1 and Mythos 5.1 reject `any` and forced selection with a 400 error. Manual extended thinking rejects both on any model. Docs point to `auto` with strict tool use, or structured outputs for a fixed JSON shape. The exam still tests all three modes, so answer with the table above.

## 3. Semantic errors schemas cannot prevent

**Rule:** `tool_use` eliminates syntax errors but not semantic errors. A schema checks structure (field names, types, presence), not meaning.

| Error | What it is | Example |
| --- | --- | --- |
| Sum discrepancy | Line items do not add up to the stated total | Items 200.00 + 50.00 = 250.00, but `total` is 300.00 |
| Field placement error | A valid value lands in the wrong field | Invoice number put in `purchase_order_number`; delivery date put in `invoice_date` |
| Fabrication | Model invents a value the source does not contain, often to fill a required field | `"purchase_order_number": "PO-88213"` when the invoice has no PO number |

**Worked example:** this output is valid JSON and passes the schema, yet contains two semantic errors.

```json
{
  "supplier_name": "Acme Ltd",
  "invoice_date": "2026-09-14",
  "line_items": [
    {"description": "Widgets", "amount": 200.00},
    {"description": "Delivery", "amount": 50.00}
  ],
  "total": 300.00,
  "purchase_order_number": "PO-88213"
}
```

- Sum discrepancy: 250.00 ≠ 300.00.
- Fabrication: the PO number does not exist in the source.

**Exam trap:** "tool\_use guarantees correct data" is wrong. It guarantees well-formed data only. You still need checks after extraction.

## 4. Optional and nullable fields

**Rule:** if source documents may lack a value, make that field optional or nullable. Required fields pressure the model to fabricate; optional or nullable fields allow an honest "not present".

**Before (causes fabrication):**

```json
"properties": {
  "purchase_order_number": {"type": "string"}
},
"required": ["supplier_name", "invoice_date", "total", "purchase_order_number"]
```

**Fix A: optional** (remove from `required`; Claude may omit the field):

```json
"required": ["supplier_name", "invoice_date", "total"]
```

**Fix B: nullable** (keep required, allow `null`; every record has the same keys):

```json
"purchase_order_number": {"type": ["string", "null"]}
```

Output when the PO number is missing: `"purchase_order_number": null`

| Choice | Key always present? | Missing value looks like |
| --- | --- | --- |
| Optional | No | Field absent |
| Nullable (required) | Yes | `null` |

**Keep required:** fields every document genuinely contains, e.g. `supplier_name`, `total`.

**Exam trap:** making all fields required when sources may lack information.

## 5. Enum patterns: unclear, other + detail, normalisation

**Rule:** add `"unclear"` for genuinely ambiguous cases and `"other"` paired with a freeform detail string for categories you did not list. State format normalisation rules in the prompt alongside the schema.

| Value | Meaning | Example |
| --- | --- | --- |
| `unclear` | Not enough information to decide | Line item reads only "Services rendered" |
| `other` + detail | Clear category, but not on the list | Catering invoice → `other`, detail `"catering"` |

**Schema example:**

```json
"expense_category": {
  "type": "string",
  "enum": ["travel", "software", "office_supplies", "consulting", "other", "unclear"]
},
"category_detail": {
  "type": ["string", "null"],
  "description": "If expense_category is 'other', name the category, e.g. 'catering'."
}
```

**Why the detail field matters:** `other` alone loses the information. With the detail, you can spot frequent values and promote them to real enum options. That makes the pattern extensible.

**Why `unclear` matters:** without it, ambiguous items are forced into a confident but arbitrary category. With it, you can route them to human review.

**Format normalisation rules (in the prompt):**

- "Dates in ISO 8601: YYYY-MM-DD" → `2026-09-14`, never `14/09/2026` or `Sept 14th`
- "Amounts as numbers without currency symbols" → `450.00`, never `"£450"`

A schema accepts any string, so without these rules you get valid but inconsistent formats.

**Exam traps:** omitting `unclear` or `other`; using `other` without a detail field; confusing `unclear` (cannot decide) with `other` (decided, not listed).

## 6. Anti-patterns and key terms

**The five anti-patterns the exam tests:**

| Anti-pattern | Example | Correct approach |
| --- | --- | --- |
| Assuming `tool_use` prevents all errors | Trusting a schema-valid total without checking line items | Add semantic checks: sums, field placement, fabrication |
| Making all fields required | `purchase_order_number` required on invoices that often lack it | Optional or nullable fields |
| Using `auto` when output is mandatory | Pipeline breaks on "This looks like a delivery note" | `any` (unknown type) or forced named tool |
| Omitting `unclear` / `other` | "Services rendered" forced into `consulting` | Add `unclear`, and `other` + detail string |
| Relying on prompt-based JSON | "Respond only with valid JSON" feeding `json.loads()` | `tool_use` + JSON schema |

**Glossary:**

| Term | Definition | Example |
| --- | --- | --- |
| `tool_use` | Claude returns structured data as a tool call's input, matching the tool's schema | `extract_invoice` called with `{"total": 250.00, …}` |
| JSON schema / `input_schema` | Defines field names, types, required fields and enums | `"total": {"type": "number"}` |
| `tool_choice` | Controls whether and which tool Claude must call | `{"type": "any"}` |
| Syntax error | Malformed JSON that will not parse | `{"total": 450,}` |
| Semantic error | Well-formed JSON with wrong meaning | `total` 300 when items sum to 250 |
| Fabrication | Invented value absent from the source | Made-up `PO-88213` |
| Field placement error | Right kind of value, wrong field | Delivery date in `invoice_date` |
| Sum discrepancy | Parts do not match the stated whole | 200 + 50 ≠ 300 |
| Nullable field | Field that may hold `null` | `{"type": ["string", "null"]}` |
| Optional field | Field not listed in `required` | Removed from `required` array |
| `unclear` | Enum value for genuine ambiguity | Vague line item |
| `other` + detail | Enum value plus freeform string for unlisted categories | `other` + `"catering"` |
| Format normalisation | Prompt rules for consistent value formats | Dates as YYYY-MM-DD |

## 7. Worked example and quick recall

**Scenario:** extract customer name, order number (often missing), complaint type (known list, new types appear) and refund amount (sometimes not stated) from complaint letters. Every letter must produce output. Only one document type.

| Decision | Choice | Why |
| --- | --- | --- |
| Extraction method | `tool_use` + JSON schema | Eliminates syntax errors |
| `tool_choice` | Forced: `{"type": "tool", "name": "extract_complaint"}` | One tool, output mandatory |
| Customer name | Required string | Every letter has one |
| Order number | Required + nullable | Same keys every record; `null` when missing |
| Refund amount | Optional or nullable number | Prevents invented figures |
| Complaint type | Enum + `other` + `unclear` | Handles new and ambiguous types |
| Complaint detail | Nullable string | Captures what `other` means |
| Prompt rules | e.g. amounts as numbers, no currency symbols | Consistent formats |
| Post-extraction check | Refund not taken from order total | Catches field placement errors |

```json
{
  "name": "extract_complaint",
  "input_schema": {
    "type": "object",
    "properties": {
      "customer_name": {"type": "string"},
      "order_number": {"type": ["string", "null"]},
      "complaint_type": {
        "type": "string",
        "enum": ["late_delivery", "damaged_item", "wrong_item", "billing", "other", "unclear"]
      },
      "complaint_detail": {"type": ["string", "null"]},
      "refund_amount": {"type": ["number", "null"]}
    },
    "required": ["customer_name", "order_number", "complaint_type"]
  }
}
```

**Quick-recall rules:**

1. Schema compliance critical → `tool_use`, never prompt-based JSON.
2. `tool_use` stops syntax errors, not semantic errors.
3. Three semantic errors: sum discrepancy, field placement, fabrication.
4. Output mandatory + type unknown → `any`.
5. Specific step must run → forced named tool.
6. Tools optional → `auto` (the default, no guarantee).
7. Value may be missing → optional or nullable, never plain required.
8. Ambiguous → `unclear`; unlisted → `other` + detail.
9. Formats → normalisation rules in the prompt.
10. Real API today may reject `any` / forced; the exam does not.
