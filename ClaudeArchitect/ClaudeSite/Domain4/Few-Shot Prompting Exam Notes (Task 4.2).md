# Few-Shot Prompting: Exam Notes (Task 4.2)

Oct 26, 2026 · @VIjay

## 1. The core rule

When output is inconsistent, few-shot examples are the first tool to reach for. Not more instructions, not confidence thresholds, not temperature changes.

**Why:** detailed instructions tell Claude the rule, but it already knows the rule. What it lacks is a concrete picture of what a correct output looks like. Examples supply that picture.

**Example:** a prompt asks for ticket summaries as a heading plus three bullets. Outputs vary between bullets, tables and prose.

- Wrong fix: rewrite the instructions with "You MUST use bullet points".
- Wrong fix: lower the temperature. You may get the wrong format consistently.
- Right fix: add 2-3 example tickets paired with correctly formatted summaries.

**Exam pattern:** "detailed instructions still give inconsistent results" → choose few-shot examples, never "add more instructions".

## 2. Key terms

| Term | Meaning | Quick example |
| --- | --- | --- |
| Few-shot examples | Sample input-output pairs placed in the prompt to show the expected result | Two tickets, each paired with its correct summary |
| Deployment trigger | A symptom telling you few-shot is the right fix | Same pattern flagged "critical" in one file, "low" in another |
| Reasoning (in an example) | A line explaining why this answer was chosen over the alternatives | "It is a test file, but the key is live, so its reach outweighs its location" |
| Pattern-matching | Copying the literal surface of examples | "Queries with order numbers go to lookup\_order" |
| Generalisation | Applying the underlying principle to unseen cases | "Specific identifiers route to specific lookup tools" |
| Ambiguous case | An input the instructions don't clearly settle | A live API key inside a test fixture |
| Narrative extraction | Pulling a value from prose rather than a labelled field | "sixty days' written notice" → `termination_notice_days: 60` |
| Nullable field | A schema field allowed to be empty (`null`) | `"governing_law": {"type": ["string", "null"]}` |
| Validation-retry loop | Code checks the output, sends the error back, Claude retries | Line items sum to £1,240 but total says £1,420 → retry |

## 3. The three deployment triggers

The exam describes symptoms, not causes. Learn to name the trigger from the symptom.

| Trigger | Symptom you'll see | Example | Few-shot fix |
| --- | --- | --- | --- |
| 1. Inconsistent formatting | Same task, different output shapes | Summaries come back as bullets, tables or prose | Show 2-3 outputs in the exact format wanted |
| 2. Inconsistent judgement calls | Same pattern classified differently across inputs | Hard-coded key in a test fixture flagged critical, low, or not at all | Show reasoned examples deciding the borderline case |
| 3. Empty fields for data that exists | Extraction returns null, but the value is in the text | Notice period in a table extracts fine; the same fact in a paragraph comes back empty | Show an example extracting the value from narrative prose |

**Remember on trigger 2:** the inconsistency appears on borderline cases, while clear cases stay consistent. Ambiguity is why the probabilities are spread out. Lowering the temperature gives consistency without correctness.

## 4. Construction rules: 2-4 examples, each with reasoning

**Rule 1: use 2-4 targeted examples.**

- Fewer than 2: no pattern is established. Claude can't tell which features matter (the key prefix? the file name? the length?).
- More than 4: wastes tokens on every call without a proportional gain.
- Pick examples that differ from each other, so together they cover different decisions.

**Rule 2: every example shows reasoning for why that action beat the alternatives.**

Weak example (no reasoning):

```
Input:  test_config.py contains API_KEY = "sk-live-8f3a..."
Output: Severity: Critical
```

Strong example (with reasoning):

```
Input:     test_config.py contains API_KEY = "sk-live-8f3a..."
Reasoning: It is a test fixture, which suggests low severity. But the
           sk-live prefix shows a real production credential, usable by
           anyone with repository access. Reach outweighs location.
Output:    Severity: Critical
```

**A well-chosen set of three:**

1. Live key in a test file → Critical (location doesn't reduce risk)
2. Placeholder `API_KEY = "your-key-here"` → Don't flag (not a real credential)
3. Real password in a config file → Critical (same principle, different form)

Three examples, three decisions, one principle. That covers far more cases than twelve unreasoned examples would.

## 5. Generalisation vs pattern-matching

Reasoning turns an example from a single case into a principle. That is what lets Claude handle inputs it has never seen.

**Analogy:** a maths book showing "12 × 15 = 180" teaches one fact. One showing the working (12 × 10 + 12 × 5) teaches a method that also solves 14 × 17. The reasoning line is the working.

|  | Without reasoning | With reasoning |
| --- | --- | --- |
| What Claude learns | "Queries mentioning order numbers go to lookup\_order" | "Specific identifiers mean the user wants their own record, so use a lookup tool" |
| New case: "What's the status of ticket T-882?" | Likely misrouted: no order number present | Routed correctly: a ticket ID is a specific identifier |
| New case: DB password in Docker Compose | Missed: not an sk-live key, not a test file | Flagged critical: it is a real credential |

**Exam line:** examples without reasoning teach literal pattern-matching; examples with reasoning teach the decision principle.

## 6. Hallucination reduction through varied document structures

Examples showing correct handling of different formats greatly reduce hallucination in extraction tasks. Once Claude has seen each layout handled properly, it guesses less when a document is unusual.

**Vary the structure across your examples:**

- Inline citations vs a bibliography at the end
- Narrative prose vs tables
- Labelled fields vs facts buried in a paragraph

**Example (narrative extraction):**

```
Text:      "Either party may end this agreement by giving not less
            than ninety days' written notice."
Reasoning: No field is labelled "notice period", but this sentence
           states the notice needed to terminate, so it maps to
           termination_notice_days. "Ninety days" = 90.
Output:    termination_notice_days: 90
```

Pair it with a second example where the same field comes from a table ("Notice Period: 60 days"). Together they show the field can appear in either form.

## 7. Anti-patterns (wrong answers the exam offers)

| Anti-pattern | Example | Why it fails |
| --- | --- | --- |
| Adding more detailed instructions | "You MUST ALWAYS use bullet points" after bullets are already requested | Claude already knows the rule; it needs to see the target |
| Confidence thresholds for consistency | "Only flag issues you are over 80% sure of" | Doesn't resolve the ambiguity; Claude may be confidently wrong |
| Examples without reasoning | Input → Output with no "why" | Teaches pattern-matching; fails on new cases |
| Too many examples (more than 4) | Twelve examples, one per credential type | Wastes tokens without proportional benefit |
| Lowering the temperature | Temperature set to 0 to stop format drift | Consistency without correctness |

A related trap from the session: telling Claude "don't add anything you're unsure of" to stop fabricated values. That's still an instruction-based fix. The structural fix is a nullable schema field (see section 8).

## 8. Technique comparison table (must memorise)

Few-shot fixes problems of understanding (format, judgement, recognition). Structural problems (syntax, required fields, arithmetic) need structural fixes (schemas, tools, code).

| Symptom | Technique | Example scenario |
| --- | --- | --- |
| Inconsistent output formatting | Few-shot examples | Summaries vary between bullets, tables and prose |
| Malformed JSON | `tool_use` with JSON schemas | Missing closing brackets or unquoted keys crash the parser |
| Fabricated values for missing fields | Optional/nullable schema fields | Contract has no governing law, but Claude fills in "England and Wales" |
| Wrong tool selection | Better tool descriptions first, then few-shot | "Where's my order #A4471?" goes to `search_knowledge_base` |
| Information missed in narrative text | Few-shot examples showing narrative extraction | Notice period in a paragraph comes back empty |
| Extracted sum doesn't match total | Validation-retry loop | Line items total £1,240; invoice says £1,420 |

**Mirror-image pair to watch for:**

- Field empty, but the data is in the document → few-shot narrative extraction.
- Field filled, but the data isn't in the document → nullable schema field.

**Tool selection order:** first rewrite vague descriptions (`lookup_order: Looks up orders.` → say when to use it and when not to). Only add reasoned few-shot examples if errors remain.

## 9. Last-minute checklist

Ask these in order when reading an exam scenario:

1. **Is the problem understanding or structure?** Understanding → few-shot. Structure → schema, tool or code.
2. **Which trigger is it?** Formatting, judgement on ambiguous cases, or missed narrative data.
3. **Is an option offering more instructions, a confidence threshold or temperature?** Usually the trap.
4. **How many examples?** 2-4. Not 1, not 12.
5. **Do the examples include reasoning?** If an option shows examples without a "why", it teaches pattern-matching only.

**Your personal weak spots from the session:**

- [ ] Name the trigger before jumping to the fix
- [ ] Don't blame randomness: ambiguity causes the spread
- [ ] Reasoning line in every example
- [ ] 2-4 examples, never 1
- [ ] Prefer structural fixes over prompt instructions where one exists
- [ ] Memorise the section 8 table
