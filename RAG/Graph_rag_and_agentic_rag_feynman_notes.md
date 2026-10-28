# Graph RAG & Agentic RAG — Feynman-Style Notes

> *"If you can't explain it simply, you don't understand it well enough."*
> These notes start from the dumbest possible version of the idea and build up. No jargon until it's earned.

---

## First, the thing we're all fixing: plain old RAG

Imagine you have a giant pile of documents and a question. You can't read all the documents every time, so you do a trick:

1. Chop the documents into small pieces ("chunks").
2. Turn each chunk into a list of numbers that captures its meaning ("embedding").
3. When a question comes in, turn the question into numbers too, and grab the chunks whose numbers are closest.
4. Hand those chunks + the question to the LLM and say "answer using this."

That's it. That's RAG. It's basically *"find the paragraphs that sound similar to the question, then let the AI read them."*

**The two things it's bad at — and each fix has a name:**

- It can't **think about its own search**. It searches once, takes whatever comes back, and answers — even if the result was garbage or incomplete. → **Agentic RAG fixes this.**
- It only finds text that's **worded similarly**. It can't follow a *chain of connections* between facts that are scattered across documents. → **Graph RAG fixes this.**

Keep these two weaknesses in mind. Everything below is just a clever patch for one of them.

---

# Part 1 — Graph RAG

## The purpose (why it exists)

Here's the problem in one sentence: **similarity is not the same as connection.**

Suppose you ask: *"Who funds the company that bought the startup Alice founded?"*

Plain RAG looks for chunks that *sound like* that sentence. But the answer isn't sitting in one similar-sounding paragraph. It's spread across three separate facts:

- Alice founded Acme.
- Beta Corp bought Acme.
- VC-X funds Beta Corp.

To answer, you have to **hop** from Alice → her company → who bought it → who funds *them*. No single chunk contains that hop. Similarity search just shrugs.

Graph RAG exists to make those hops possible. The whole idea: **stop storing knowledge as a bag of loose paragraphs, and start storing it as a web of connected things.**

## How it works, step by step

Think of it in two phases, like building a subway system and then riding it.

### Phase A — Building the map (offline, done once)

**Step 1 — Chop the documents into chunks** (same as plain RAG — we keep these).

**Step 2 — Read each chunk and pull out the "things" and "connections."**
An LLM reads the text and extracts:
- **Things** = entities = nouns that matter (people, companies, places). These become **dots (nodes)**.
- **Connections** = relationships, usually verbs. These become **lines (edges)** between dots.

From *"Alice founded Acme; Beta Corp acquired Acme; VC-X funds Beta Corp"* you get:

```
(Alice) ──FOUNDED──> (Acme) <──ACQUIRED── (Beta Corp) <──FUNDS── (VC-X)
```

Notice: most words got thrown away. "the," "in," "because" aren't things or connections. Only the meaningful dots and lines survive.

**Step 3 — Store the map (the knowledge graph)** in a graph database, AND keep the original chunks too, with each dot/line remembering *which chunk it came from*. The graph is the map; the chunks are the detailed street-level photos behind each point on the map.

**Step 4 (optional but powerful) — Summarize neighborhoods.**
Group tightly-connected dots into "communities" and pre-write a summary of each. This lets you later answer big-picture questions like *"What are the main themes here?"* — something plain RAG is hopeless at.

### Phase B — Riding the map (when a question arrives)

**Step 5 — Find the starting dots ("anchoring").**
Take the question, pull out the entity *mentions* in it (the LLM reads only the question for this), then match those mentions to real dots in the graph using embeddings/keyword lookup. *"Alice"* and *"startup"* → land on the dots `Alice` and `Acme`.

**Step 6 — Walk the connections (traversal).**
Starting from those dots, follow the lines:
`Alice → FOUNDED → Acme → ACQUIRED-by → Beta Corp → FUNDED-by → VC-X.`
You've now collected a small relevant slice of the map — a **subgraph**.

**Step 7 — Grab the photos behind the dots.**
Follow the links from those dots/lines back to their original chunks, so the LLM has real source text, not just a skeleton.

**Step 8 — Hand the LLM both** the subgraph (the connections) and the chunks (the source words), and let it write the answer: *"VC-X — it funds Beta Corp, which acquired Alice's startup Acme."*

### The one-line summary of how it works
> Turn text into a web of connected things, then *walk the web* to find facts that are related but not similarly worded, and read the source text behind them.

## Pros

- **Multi-hop reasoning.** It can chain facts across many documents — the thing plain RAG simply cannot do.
- **Big-picture questions.** With community summaries it can answer "what are the overall themes?" across a whole corpus.
- **Explainable.** You can literally see the path it walked: Alice → Acme → Beta → VC-X. Great for trust and auditing.
- **Connected domains shine.** Org charts, medical (drug→disease→symptom), legal, supply chains, citations — anything where *relationships are the point*.

## Cons

- **Expensive to build.** Extracting clean entities and relationships from messy text (Step 2) takes a lot of LLM calls and careful prompting. This is the big cost.
- **The graph can be wrong.** If extraction misreads a relationship, the graph carries that mistake forever. Garbage in, garbage graph.
- **Goes stale.** When documents change, you have to re-extract and update the graph — more maintenance than just re-embedding chunks.
- **Overkill for simple lookups.** If your questions are "what does our refund policy say?", a graph is a heavy hammer for a small nail.

## When to USE Graph RAG

- Questions need **multiple hops** ("who is connected to whom, through what").
- Your domain is **inherently relational** (people/orgs/deals, biology, law, networks).
- You need **whole-corpus / thematic** answers, not just a single fact.
- **Explainability matters** — you must show *why* an answer is true.

## When NOT to use Graph RAG

- Your questions are **simple, single-fact lookups** ("what's the deadline?").
- Your documents are **flat and unconnected** (e.g., independent FAQ entries).
- You're **early/prototyping** and just need something working cheaply — start with plain RAG.
- The relationships in your data are **fuzzy or not worth modeling** — forcing a graph adds cost for no gain.

---

# Part 2 — Agentic RAG

## The purpose (why it exists)

Plain RAG is like a student who, given any question, walks to *one* shelf, grabs the *first* book that looks relevant, copies a paragraph, and hands it in — **without ever checking whether it's the right book or whether they're missing half the answer.**

Agentic RAG turns that student into a **thoughtful researcher** who:
- breaks the question into parts,
- decides *which* shelf (tool) to use for each part,
- *checks* whether what they found is actually good and complete,
- goes back for more if it isn't,
- and only answers once they're confident.

The purpose: **add a brain that controls the search**, instead of doing one blind grab.

The key realization: the fix isn't better chunks — it's better *decisions about searching*.

## How it works, step by step

The agent runs a loop. People call it **Reason → Act → Observe → Reflect**, but really it's just *think, do, look, check — repeat.*

Let's use a real question:
> *"I'm relocating to Germany next month — am I still eligible for the remote-work stipend, and how much is it in euros?"*

**Step 1 — Plan (break it up).**
The agent thinks: *"Three parts. (a) Does the policy allow international relocation? (b) Am I personally eligible — tenure, role? (c) Convert the dollar amount to euros at today's rate."*

**Step 2 — Act (pick the right tool for part a).**
It chooses the **HR policy search** (not a generic search) and looks up the stipend rule.
→ Finds: *"Eligible internationally if you've completed 12 months of employment; stipend is $200/month."*

**Step 3 — Reflect (check what came back).**
*"Good — but this introduces a condition: 12 months employed. I don't know this person's tenure. Missing info. I must look it up."*
(This noticing-a-gap step is the whole point. Plain RAG would've stopped and answered "$200" here — and been wrong.)

**Step 4 — Act (different tool for part b).**
It queries the **employee database** (structured data, not text): hire date = Jan 2024, role = engineer.

**Step 5 — Reflect + reason.**
*"Today is mid-2026, so ~2.5 years employed — over 12 months. Eligible: yes. Now part c: I need a *live* exchange rate, so I'll use web search."*

**Step 6 — Act (yet another tool for part c).**
**Web search:** "USD to EUR today" → 1 USD ≈ 0.92 EUR.

**Step 7 — Reflect (am I done?).**
*"Policy ✓, eligibility ✓, conversion 200 × 0.92 = €184 ✓. Nothing missing. Now I can answer."*
If something were still missing, it would **loop back** to Step 2 instead of answering. That loop is what makes it "agentic."

**Step 8 — Synthesize.**
*"Yes, you're still eligible (you've passed the 12-month mark). The stipend is $200/month, about €184 at today's rate."*

### What made that "agentic" (vs plain RAG)
- It **split** one question into sub-questions.
- It **picked different tools** for each (policy text vs database vs live web) — this is *routing*.
- It **rewrote** the query for searching instead of searching the raw sentence.
- It **checked** results and caught a hidden requirement.
- It was ready to **loop and search again**.
- It only answered **after verifying completeness.**

### The one-line summary of how it works
> Put an LLM in charge as a decision-maker that plans, picks tools, checks its own work, and keeps searching until the answer is actually complete.

## Pros

- **Handles complex, multi-part questions** that need several steps or several sources.
- **Uses many tools** — vector search, databases, calculators, live web, even other RAG pipelines — and picks the right one.
- **Self-correcting.** It can notice bad/missing retrieval and try again, instead of confidently answering with garbage.
- **Adapts.** Easy questions get a quick path; hard ones get more steps. It scales effort to difficulty.

## Cons

- **Slow and expensive.** Each "think/check" step is another LLM call. One answer might cost many calls.
- **Can loop or wander.** A poorly-designed agent may keep searching, second-guess itself, or get stuck.
- **Harder to debug and predict.** More moving parts, more non-determinism — the same question can take different paths.
- **Overkill for simple questions.** If a single lookup answers it, all this machinery is wasted cost and latency.

## When to USE Agentic RAG

- Questions are **multi-step or multi-part** ("compare X and Y, then compute Z").
- You have **several different sources/tools** and need to choose among them.
- The task needs **live data or computation** (today's price, a calculation, an API call).
- **Retrieval quality must be verified** before answering (high-stakes correctness).

## When NOT to use Agentic RAG

- Questions are **simple, single-fact lookups** — the loop just adds cost and delay.
- You need **fast, cheap, predictable** responses (high volume, low latency).
- There's only **one source** and no decisions to make — there's nothing for the agent to route or reflect on.
- You can't tolerate **variable behavior** — agents are less deterministic than a fixed pipeline.

---

# Part 3 — Seeing them side by side

| Question | Plain RAG | Agentic RAG | Graph RAG |
|---|---|---|---|
| **What problem does it fix?** | (the baseline) | "I searched once, blindly, and didn't check." | "Similar text ≠ connected facts." |
| **The core trick** | Find similar chunks | A brain that plans, routes, and re-checks | A web of connected things you can walk |
| **What it searches** | Chunks (by similarity) | Whatever tool it decides to use | The graph (by following connections) |
| **Best at** | Simple grounded Q&A | Multi-step, multi-tool, verify-then-answer | Multi-hop relationships & big-picture themes |
| **Main cost** | Cheap | Many LLM calls (slow) | Expensive to build & maintain the graph |
| **Mental image** | A photocopier | A thoughtful researcher | A subway map you ride between stations |

## The deepest takeaway

These two aren't rivals — they fix **different** weaknesses:

- **Agentic RAG** improves *how you decide to search* (the control/brain).
- **Graph RAG** improves *how knowledge is stored and connected* (the data/map).

So the most powerful real systems **combine them**: an agent that *decides* to consult a graph, *walks* it, *checks* the result, and searches elsewhere if needed. The brain (agentic) drives the map (graph).

> **Plain RAG** = find similar text.
> **Graph RAG** = find *connected* facts by walking a map.
> **Agentic RAG** = a researcher who plans, checks, and won't answer until it's sure.
