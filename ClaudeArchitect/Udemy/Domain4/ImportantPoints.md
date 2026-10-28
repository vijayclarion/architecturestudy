# Clarity, Specificity, and the Cost of Ambiguity
- The principle is direct: be clear and specific about what you want, because ambiguity forces the model to guess and guesses are unreliable. Your notes frame this around a few ideas:
- Specify the task precisely. what you want done, in what form, with what constraints.
- Provide the necessary context
- Say what you do want, concretely. P
- Ambiguity has a cost, and it's often silent.

# Instruction Decomposition and Constraint Design
- Instruction decomposition = taking one complex, multi-requirement instruction and breaking it into multiple clear, separate, explicitly-stated instructions (a numbered list, discrete steps, an itemized set of requirements) — instead of leaving it as one dense, tangled paragraph.
- Constraint design — make constraints explicit, Specific and measurable where possible, and satisfiable
- Non-contradictory - conflicting constraints produce unpredictable resolution (the model can't satisfy both), so keep them coherent.
- Positively framed where possible — telling the model what to do is generally more reliable than a pile of prohibitions. Prefer "respond only with the JSON object" over a list of "don'ts."

# The System Prompt — Purpose, Structure, and Best Practices
- The system prompt is a separate, high-level instruction channel that sets up the model's role, behavior, context, and standing rules for the entire conversation,

Purpose — what the system prompt is for:

- Set the role/persona — who the model should act as (an expert code reviewer, a customer-support agent for a specific company). (Role assignment is the next topic in depth.)
- Establish standing behavior and rules — how it should always behave, constraints it must always respect, tone, boundaries. These persist across every turn, not just one.
- Provide durable context — background about the task, the domain, the company, that should frame the whole interaction.
- Set default output expectations — formats or conventions the model should follow throughout.

- use it for what should persist; use user turns for what's specific to the moment. Don't put one-off requests in the system prompt or standing rules in a single user turn.
- the system prompt is the dedicated channel for durable, high-level setup — role, standing behavior, context, and default format — that governs the entire conversation, separate from the per-turn requests in user messages


# Role Assignment, Persona Design, and Context Setting
- Role assignment (or persona prompting) is giving the model a specific role or identity to adopt — "You are an expert X" — which steers its behavior, tone, knowledge framing, and output toward what that role would produce. Your notes frame it around a few ideas:
- Telling the model to act as a specific expert or persona shifts it out of the generic default into a framing appropriate to that role. "
- Persona design shapes tone and style. Beyond expertise, the persona sets how the model communicates — formal vs. casual, concise vs. thorough, technical vs. plain-language.
- Context setting frames the task. Alongside the role, providing the situational context — the purpose, the audience, the domain, the constraints of the situation — frames how the model approaches the task.
- It belongs in the system prompt (standing setup). Because a role/persona should persist across the whole interaction, it's set in the system prompt (the last topic's principle). Set the role once, and it governs every turn.

# Using XML Tags to Structure Complex Prompts
- XML tags delimit and label the distinct parts of a prompt, giving the model clear structural boundaries so it can tell instructions from data from examples from format specs. Your notes establish:
-  Tags mark boundaries and roles. Wrapping each part in named tags — <instructions>…</instructions>, <context>…</context>, <document>…</document>, <example>…</example>, <data>…</data> — tells the model exactly what each section is and where it starts and stops.
- Claude responds especially well to XML tags. This is a Claude-specific best practice your notes flag: Claude is trained to recognize and use XML-style tags for structure, so using them is a particularly effective way to organize prompts for Claude. It's the recommended structuring convention.
- Separating data from instructions is the key win. The most important use: clearly distinguishing the content to be processed from the instructions about what to do with it. When you wrap the document in <document> tags and the task in <instructions> tags
- Tags enable precise reference. Once parts are tagged, you can refer to them in your instructions ("summarize the text in <document>," "f
- XML tags give a complex prompt clear, labeled structure — separating instructions, context, data, examples, and format — so the model parses each part correctly, and Claude in particular is optimized to use them.

# Document Structure Patterns for Multi-Part Tasks
- It's the set of patterns for organizing longer, multi-part prompts so the model handles all parts reliably. Your notes emphasize:
- Organize into clear, logical sections
- Order matters — put things where they land best
- Handle multiple documents/inputs distinctly
- Use consistent structure and reference
- Keep long prompts navigable

# Why Examples Work — Pattern Matching and Format Anchoring
- This topic explains the mechanism behind few-shot prompting: providing examples of the desired input→output behavior lets the model pattern-match to them, producing output that follows the demonstrated pattern. Your notes frame the "why" around two ideas:
- pattern matching. Models are powerful pattern matchers. When you show input→output examples, the model infers the pattern connecting them
- Format anchoring. Examples anchor the output format concretely. Rather than describing the format ("return a JSON object with these fields, in this order, with this style…"), you show one, and the model matches its structure exactly.
- Pros Precision, Consistency and Disambiguation

- Zero-shot (description only):
- Few-shot (with examples):

# Designing High-Quality Few-Shot Examples
- Make examples representative of the real task. The examples should reflect the actual inputs the model will face and the outputs you actually want.
- Cover the range — including edge cases and the harder cases. Don't show only the easy, obvious cases. Include examples that demonstrate how to handle the boundaries and the tricky situations
- Keep examples consistent with each other. All examples must demonstrate the same pattern and format.
- Ensure examples are correct. Every example must be a correct demonstration of what you want — the right output, the right format, the right handling.
- Cover the classes/categories you care about (for classification). If the task has multiple categories or output types, include examples of each — especially so the model sees each class demonstrated. Showing only some classes biases the model toward those. 
- Use enough examples, but quality over quantity. A few well-chosen, representative, correct, varied examples beat many redundant or poor ones.

# Chain-of-Thought — Prompting for Visible Reasoning
- For simple tasks the model can jump straight to an answer. But for tasks requiring reasoning — multi-step math, logic, complex analysis, decisions with several dependent considerations
- Chain-of-thought (CoT) prompting instructs the model to reason step-by-step / show its work before giving the final answer, which improves accuracy on tasks that require reasoning. Your notes frame it around:
- Reasoning first, answer after.
- When to use it. CoT helps on tasks that require reasoning — math, logic, multi-step analysis, complex decisions, anything with dependent steps. It's not needed for simple,
- Structuring the reasoning. You can guide how the model reasons — asking for specific steps, or having it put its reasoning in tags (e.g., <thinking>) separate from the final answer (e.g., <answer>)
- chain-of-thought prompts the model to produce its reasoning before its answer, improving accuracy on reasoning-heavy tasks by making it work through intermediate steps — used when the task requires reasoning, skipped when it's simple and direct.


# Extended Thinking:
- Extended Thinking reaches a similar goal a different way. You don't change the prompt. You turn on a feature in the API request. Claude then reasons internally first and sends that reasoning back as separate thinking blocks, before the final answer.
- On current models you set thinking to adaptive mode. Claude then decides for itself how much reasoning each request needs. You guide that with an effort level: low, medium, high (the default), x-high, or max
- When to use it 
    Extended thinking is a good fit for:
    multi-step reasoning where CoT isn't good enough,
    problems that require exploring several solution paths before committing,
    synthesising many constraints or large amounts of data,
    high-stakes decisions where quality matters more than speed.


# Prefilling the Assistant Turn to Control Output Format.

- Normally your messages array ends with a user turn, and Claude writes a fresh assistant reply from nothing.With prefilling, you end the array with a partial assistant turn, meaning the first few characters of the reply you want. Claude doesn't start over. It continues from exactly where your text stops.
- Prefilling is inserting a partial assistant turn at the end of the messages array so the output is constrained from the first token.
- Output steering is the broad category: any technique that guides output structure before generation begins. Prefilling is one of these. Explicit format instructions and example-driven constraints (few-shot) are others.
- Format anchoring is the specific practice of using an opening token, an XML wrapper, or a structural marker to lock in a format. The course calls it the most reliable way to enforce JSON, markdown tables, or lists.


# Steering Model Output

- Steering is actively directing the model's output toward the desired form and content, using a combination of techniques your notes have been building toward. The key steering levers:'
- Examples (few-shot) to anchor the pattern. D
- Explicit instructions about form and content.
- XML tags / structure for output


# Temperature, top_p, and top_k Explained
- Temperature. The primary randomness dial. The model, at each step, has a probability distribution over possible next tokens. Temperature controls how that distribution is sampled:
- Low temperature (→0) makes generation more deterministic/focused
- High temperature makes generation more random/creative/varied
- The rule: low temperature for consistency/accuracy/determinism; high temperature for creativity/variety.
- top_p (nucleus sampling). Restricts token choice to the smallest set of top tokens whose cumulative probability reaches p. cut by CUMULATIVE PROBABILITY (mass).
- top_k. Restricts token choice to the top k most probable tokens — a fixed count cutoff. top_k = 10 means "only ever consider the 10 most likely next tokens. cut by COUNT (rank)

# Designing Effective Multi-Turn Interactions
- It's the principles for designing interactions that work well across multiple turns. Your notes emphasize:
- Maintain coherence and continuity across turns.
- Manage the growing context.
- Keep each turn clear in context.
- Design the flow deliberately.
- designing multi-turn interactions means maintaining coherence/consistency across turns (primarily via a persistent system prompt), managing the context that accumulates as the conversation grows (with a plan for when it gets long), and keeping each turn clear within that growing history
- Poorly designed for multi-turn: no system prompt — the role and rules are stated only in the first user message. By turn 10, that initial framing is buried far up

# Conversation State, Context Accumulation, and Turn Management
- It explains the state model of a conversation and how to manage its growth. Your notes center on:
- The model is stateless; state is the re-sent history.
- The conversation history is included in each request
- Context accumulation is therefore inherent.
- Turn management — strategies to control the growth
   - Summarize / compact earlier turns — replace verbose past exchang
   - Trim / filter — drop content that's no longer needed (stale tool outputs, resolved tangents) rather than carrying everything forever.
   - Reference instead of inline — keep large data out of the running history, referring to it rather than re-sending it each turn.
   - Carry forward the essentials — ensure the important durable facts (the user's goal, key decisions, critical context) persist, whether via the persistent system prompt, a fact block, or the summary — so they don't get lost when older turns are trimmed.
- the model is stateless, so conversation "state" is the history re-sent each turn — which means context inherently accumulates as the conversation grows, and must be actively managed via summarization/compaction, trimming, referencing, and carrying forward essentials, to keep the conversation coherent without exceeding the context budget.


# JSON Mode and Schema-Constrained Output
- The problem it solves. These are mechanisms to force the model's output to be valid, well-formed structured data rather than relying on prompting alone. Your notes frame two related levels:-
- JSON mode — constrains the model to produce valid JSON output. Instead of hoping the prompt yields parseable JSON, JSON mode guarantees the output is syntactically
- Schema-constrained output — goes further: constrains the output to conform to a specific schema you define (t
- JSON mode guarantees valid JSON; schema-constrained output guarantees JSON conforming to a specified schema — both are constrained generation that structurally enforce format (not merely prompt for it), giving reliable machine-parseable output — but they guarantee structure, not the correctness of the values.

# Schema Design for Reliable Extraction
- It's the principles for designing schemas that produce accurate extraction, not just valid structure. Your notes emphasize:
- Clear, descriptive field names and descriptions. The field name (and any field-level description) tells the model what goes there. customer_email is clearer than field2;
- Appropriate, specific types. Use the right type for each field — string, number, boolean, array, nested object, enum — so the model is constrained toward the correct form.
- Model the schema on the actual data.
- Handle optionality and missing data correctly.
- design the extraction schema so it guides accurate extraction — clear field names/descriptions (which instruct the model what to pull), specific/appropriate types (enums for fixed sets, correct formats), a shape that matches the real data, and correct handling of optional/missing fields (so the model reports absence rather than hallucinating) — because a well-constrained schema reduces wrong-but-valid output, while a vague one guarantees it. 

# Structured Outputs Limits
- . It's the set of boundaries and failure modes of structured/schema-constrained output. Your notes cover:
- Structure is guaranteed; correctness is not — the headline limit. conforming structure — it does nothing to guarantee the values are right
- Schema complexity has limits / costs. V
- Forcing structure can constrain/degrade the model's output.
- It doesn't fit every task
- Still need to handle failures/edge cases
- structured output has real limits — it guarantees structure but not correctness (validation is still needed), it degrades with excessive schema complexity, it can hurt output when structure is forced where it doesn't fit, and it suits only genuinely-structured programmatic tasks — so it's a format-reliability tool, not a correctness or universal solution

# Using Tool Definitions to Extract Structured Data
- It's a technique to get reliable structured output by defining a tool whose input schema matches the data you want, and having the model "call" it — using the tool-use mechanism as the structured-extraction channel. Your notes frame it around:


# Validation Loops and Hallucination Prevention
- It's the practice of validating model output to catch errors and prevent hallucinations, in a loop that detects problems and corrects them rather than trusting the first output. Your notes frame it around:
- Why validation is necessary — correctness isn't guaranteed. Models hallucinate and make content errors; structured output doesn't fix thi
- The validation loop — generate → validate → correct → re-validate. Instead of accepting the first output, you validate it against criteria; if it fails, you feed the failure back and have the model (or the system) correct
- Hallucination-prevention techniques. Yo
   - Ground the output in provided source / require citations.
   - Let the model say "I don't know" / not-present
   - Validate claims/values against the source or against external truth.
   - Cross-check / verify
   - Validation can be programmatic or model-based.

# Validation Gates and Retry Strategy
- . It's the design of the gate-and-retry mechanics of a validation loop. Your notes frame it around:
- Validation gates — the checkpoint output must pass. A validation gate is a point where output is checked against criteria and only allowed through if it passes. It's the enforcement checkpoint: output that fails the gate is not accepted — it's rejected and sent back for correction (or escalated)
- Retry strategy — what happens on failure. When output fails the gate, you retry: feed back the failure (what was wrong) and have the model regenerate, aiming to pass the gate this time. The design questions — straight from Domain 1's retry/abort material:
  - Feed back why it failed
  - Bound the retries. Never retry infinitely. Set a maximum number of attempts
  - Have a defined failure path (abort/escalate). When retries are exhausted, you need a designed outcome: abort, escalate to a human, return a safe fallback, or flag the output as
  - Know when retry won't help.
- validation loop is built from a gate (output must pass defined criteria to be accepted, else it's rejected) and a bounded retry strategy (feed back the failure, regenerate, cap the attempts, and on exhaustion take a designed path — abort or escalate, especially to human review for high-stakes output).

# Multi-Pass Review — When One Pass Is Not Enough
- Multi-pass review uses more than one pass over the output — typically a generation pass followed by one or more review/critique/refinement passes — to catch errors and improve quality that a single pass misses. Your notes frame it around:
- Separate generation from review.
- Why multiple passes help.
- When to use it — match to complexity/stakes. Multi-pass review is for complex, high-stakes, or quality-critical output
- Forms it takes. A review pass can be the same model critiquing its own output (self-review/self-crit

# Building a Prompt Evaluation Workflow
- It's the practice of systematically evaluating a prompt's quality against a set of test cases, with defined success criteria, so you can measure performance and know whether changes improve it. Your notes frame the workflow around a few components:
- A test dataset — many representative cases, not a few. The foundation: a collection of test inputs (ideally with expected/acceptable outputs) that represents the real range of cases the prompt will face
- Defined success criteria — what "good" means, measurably.
- A grading method — how each output is scored. Run the prompt across the dataset and grade each output against the criteria. Grading can be:
    - Programmatic/deterministic — code checks (exact match, contains X, valid format, value in range). Objective and cheap, good for checkable criteria
    - Model-based grading — use a model to judge output quality against criteria, for things hard to check with code (is this summary accurate? is this response helpful?). (Detailed in the next topic.)
    - Human grading — for subjective/high-stakes quality, though it doesn't scale as well.
    - Measure, then iterate. Run the eval to get a score/metric across the dataset. N


# Test Dataset Generation and Model-Based Grading
- Building a good eval dataset by hand is slow and tends to under-cover the range. The technique: use a model to generate test cases — produce a diverse set of representative inputs (and expected outputs where applicable), including edge cases and varied scenarios, far faster than writing them manually. Your notes' points:
- Model-based grading (LLM-as-judge). For criteria you can't check programmatically, use a model to grade/evaluate the outputs against the criteria. Your notes' points: