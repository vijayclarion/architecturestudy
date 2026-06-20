# Naive RAG — Feynman Notes

*Notes written as if explaining to a smart friend. Built from a deep walkthrough — covers the why, the full mechanism, and the maths underneath.*

---

## 1. The Purpose — Why Naive RAG Exists

An LLM is like a student who studied **one book** for an exam.

- If the exam question is **inside the book** → the student answers well.
- If the question is **outside the book** → the student is stuck, or worse, **makes something up**.

That "one book" is the LLM's training data. It creates four problems RAG was built to solve:

**Problem 1 — Knowledge cutoff.**
The LLM was trained up to a certain date. Anything after that, it simply doesn't know.

**Problem 2 — No access to your private data.**
Your company's HR policy, internal wiki, contracts — the LLM never saw these. They weren't in its "book."

**Problem 3 — Hallucination.**
When asked something it doesn't know, the LLM doesn't say "I don't know." It confidently **invents** a plausible-sounding but false answer — like a student bluffing on an exam.

**Problem 4 — Token limit + lost in the middle.**
The naive fix is "just paste all the documents into the prompt." This fails because:
- Huge documents blow past the token limit.
- Even when they fit, the LLM **loses important details buried in the middle** of a long context.

> **RAG's purpose:** give the LLM exactly the right "pages" from your documents at question time — so it answers from real information instead of guessing. This kills hallucination, sidesteps the knowledge cutoff, and avoids dumping everything into the prompt.

RAG = **R**etrieval **A**ugmented **G**eneration.

---

## 2. The Big Picture (The Book Analogy)

A student with an **index** beats a student without one.

- **Without an index:** read all 500 pages hoping to stumble on the answer. Slow, tiring, easy to miss.
- **With an index:** look up "maternity leave" → jump to page 234 → answer instantly.

RAG gives the LLM an index into *your* documents. When a question comes, we **look up the relevant pages first**, hand them to the LLM, and say "answer using only these."

---

## 3. The Two Big Phases

### Phase A — Indexing (done once, upfront / offline)
1. Break documents into **chunks**.
2. Convert each chunk into numbers → an **embedding (vector)**.
3. Store every vector in a **vector database**.

### Phase B — Query time (every time a user asks)
1. Convert the **query** into a vector (same embedding model).
2. Find the stored chunks whose vectors are **closest** (cosine similarity).
3. Take the **top-k** chunks.
4. Build a prompt: instructions + retrieved chunks + question.
5. LLM generates the answer from that context.

---

## 4. What Is a Chunk? (and why chunk at all)

A chunk is a **small piece** of a document.

Why break documents into chunks instead of storing them whole?
- LLMs have token limits — can't fit entire documents.
- Smaller, focused chunks → **more precise** retrieval (you fetch only the relevant piece).
- Avoids the "lost in the middle" problem.

So a 50-page HR policy might become ~100 chunks, each a paragraph or section.

---

## 5. What Is an Embedding? (text → numbers)

**An embedding is the process of turning text into a list of numbers.** The output list IS the embedding.

### Why numbers?
Surface answer: machine learning models work with numbers, not raw words.

**But the real reason is deeper:** the numbers **represent meaning**.

### Why a *list* of numbers, not just one?
If you mapped "cat" = 5, "dog" = 7, that's just labels — it tells you nothing about *meaning*. A single number can't capture that a cat is similar to a kitten but different from a car.

So instead, each piece of text becomes a **list** of numbers — and each number is a **feature** of meaning.

### The "human features" analogy
Describe a person with numbers: height, weight, age, introversion, love of music… With enough such features, you can describe anyone, and **two similar people end up with similar numbers**.

Embeddings do the same for text. Each dimension captures some feature of meaning (the model decides what — humans don't label them). The result: **text with similar meaning gets similar numbers.**

### The payoff — meaning, not word-matching
This is the magic. These two phrases:
- "Maternity leave is 26 weeks"
- "Female employees get six months off after childbirth"

share **almost no words**, yet their embeddings are **close**, because they *mean* the same thing. Embeddings capture **meaning, not word overlap.**

So when a user asks "How long is maternity leave?" and the document says "26 weeks of paid leave after childbirth" — different words — the embeddings still land near each other and the match is found.

### Where does this "knowledge of relationships" come from?
The embedding model **learned** it during training, from billions of sentences where words like "maternity," "childbirth," "leave," and "time off" appeared in similar contexts. Nobody hand-coded a dictionary. The model figured out the relationships from patterns in massive text data.

> Consequence: if the model had **never** seen "maternity" and "childbirth" used in related contexts during training, it would NOT place them close together, and cosine similarity would fail to match them.

---

## 6. Embedding vs Vector vs Vector Database (the confusion, cleared)

These three are related but distinct. The photo analogy makes it click:

| Concept | What it is | Photo analogy |
|---|---|---|
| **Embedding** | The *process* of turning text into numbers (and the output) | Taking the photo |
| **Vector** | An ordered list of numbers, e.g. `[0.2, -0.5, 0.8, ...]` | The photo file |
| **Vector database** | A special store designed to hold and search vectors fast | The photo album |

**Key point:** an embedding *is* a vector — same thing. When you embed text, you get a vector. The vector database is just where those vectors live.

The relationship as a flow:
```
Text document
   ↓  embedding model (the process)
Vector (list of numbers — the output)
   ↓  store it
Vector database (where vectors live)
```

---

## 7. What Is a Vector Database, and Why Not Just SQL?

A **regular (SQL) database** matches **exact words**:
```
ID | Text
1  | "Employees get 26 weeks leave after childbirth"
2  | "Sick leave is 12 days per year"
```
Searching needs the exact keyword. "maternity" wouldn't find row 1.

A **vector database** stores the *meaning vectors* and searches by **closeness in space**:
```
ID | Vector
1  | [0.2, -0.5, 0.8, 0.1, -0.3, ...]
2  | [0.1, -0.3, 0.7, 0.05, -0.2, ...]
```
It finds vectors **near** the query vector — so "maternity leave?" finds the childbirth chunk even with different words.

### Why a special index (not a B-tree)?
SQL uses a **B-tree** index — great for sorted, exact lookups (like a phonebook sorted by name).

But vectors live in a **multi-dimensional space** (think of a map). You don't want exact matches — you want **"what's nearby in meaning?"** A B-tree can't do "nearest in 768-dimensional space."

So vector DBs use special indexes built for spatial nearest-neighbour search:
- **HNSW** (a graph of nearest neighbours)
- **IVF** (partition the space into regions)

These are **approximate nearest neighbour (ANN)** methods — they avoid comparing against every single vector, so search across millions of vectors takes **milliseconds**.

### The 2D Map — "Find Nearby Location" (the example to remember)

This is the single best mental model for a spatial index. **Shrink the 768-dimensional space down to a 2D map** — same idea, just easier to picture.

Imagine every document chunk is a **pin on a city map**. Chunks about similar topics sit near each other, exactly like shops of the same kind cluster in the same neighbourhood:

```
        (maternity / childbirth corner)
        📍 "26 weeks paid leave after childbirth"
        📍 "maternity benefit eligibility"
        📍 "extended maternity 8 weeks unpaid"

                                  (sick-leave corner — far away)
                                  📍 "sick leave 12 days/year"
                                  📍 "medical certificate rules"

   (payroll corner — somewhere else entirely)
   📍 "salary credited on 1st"
   📍 "tax form-16 issuance"
```

Now a user asks **"How long is maternity leave?"** → the query becomes its own pin and **drops onto the map** right in the maternity corner:

```
        📍 "26 weeks paid leave after childbirth"   ← 0.92
   ★ QUERY lands here
        📍 "maternity benefit eligibility"          ← 0.88
        📍 "extended maternity 8 weeks unpaid"       ← 0.75
```

**Finding the answer = finding the pins nearest to the star.** Just like asking your phone "coffee shops near me" — you don't measure the distance from your location to every shop on Earth. The map is **organised by location**, so the app jumps straight to your neighbourhood and only checks the handful of pins around you.

That "jump straight to the neighbourhood instead of checking everything" is **exactly** what the spatial index (HNSW / IVF) does:
- **Without an index:** compare the query to all 1,000,000 pins, one by one → slow.
- **With a spatial index:** zoom to the right region of the map, check only the ~50 pins nearby → milliseconds.

> One-liner to remember: **a vector database is a map where meaning is location, and search means "find the nearest pins."**

(The only difference from a real map: a real map has 2 dimensions, north–south and east–west. The vector map has 768 dimensions — but "find the nearest pins" works identically, the maths just runs in more dimensions.)

---

## 8. How Documents Are Saved (Indexing — the one-time step)

```
HR Policy PDF (50 pages)
   ↓ extract text
   ↓ chunk (~100 chunks)
   ↓ embed each chunk  → ~100 vectors
   ↓ store in vector DB (vector + original text + metadata)
```

Each stored row looks like:
```
id: 47
text: "Maternity leave is 26 weeks of paid leave..."
vector: [0.024, -0.183, 0.451, ..., 0.117]
metadata: { source: "HR_Policy.pdf", page: 12 }
```

**All chunk vectors are stored.** This is the upfront, one-time preprocessing step.

---

## 9. How a Query Is Retrieved (Query time)

Crucial fact: **the document chunks and the query are embedded SEPARATELY, never together.**
- Chunks were embedded once and stored.
- The query is embedded fresh each time — and is **NOT stored** in the database. It's only used to search.

Step by step:

```
Query: "How long is maternity leave?"
   ↓ embed the WHOLE query → ONE query vector
   ↓ vector DB compares query vector against stored chunk vectors (cosine similarity)
   ↓ sort by score, take top-k
Top results:
   [1] "Maternity leave is 26 weeks..."        (0.92)
   [2] "Leave categories: sick, maternity..."   (0.88)
   [3] "Extended maternity options..."          (0.75)
```

### top-k — how many chunks do we send?
In Naive RAG, **top-k is a fixed number YOU set in your code** — not a hyperparameter of the embedding model.
```python
results = vector_db.search(query_vector, top_k=3)   # YOU choose 3, 5, 10...
```
- It has **nothing to do with the embedding model**. The embedding model only converts text to vectors. Your RAG pipeline decides how many to retrieve.
- Typically top-k = 3 to 5.

---

## 10. Cosine Similarity — How "Closeness" Is Measured

### The idea
Each vector is an **arrow** from the origin pointing somewhere. Cosine similarity measures the **angle** between two arrows — *not* their length.

- Small angle → arrows point the same way → similar meaning → score near **1.0**
- 90° angle → perpendicular → unrelated → score near **0.0**
- 180° → opposite → opposite meaning → score near **−1.0**

> Map analogy: two cities close together have a small angle between their direction vectors; two far-apart cities have a bigger angle.

### How do you get an angle from two lists of numbers? (the maths)

Three steps. Example with small vectors:
```
Document chunk vector: [0.2, -0.5, 0.8, 0.1]
Query vector:          [0.18, -0.52, 0.79, 0.12]
```

**Step 1 — Dot product** (multiply matching positions, add them up):
```
(0.2×0.18) + (-0.5×-0.52) + (0.8×0.79) + (0.1×0.12)
= 0.036 + 0.26 + 0.632 + 0.012
= 0.94
```

**Step 2 — Length of each vector** (√ of sum of squares):
```
|doc|   = √(0.2² + (-0.5)² + 0.8² + 0.1²)      = √0.94  ≈ 0.97
|query| = √(0.18² + (-0.52)² + 0.79² + 0.12²)  = √0.9413 ≈ 0.97
```

**Step 3 — Divide:**
```
cosine_similarity = dot_product / (|doc| × |query|)
                  = 0.94 / (0.97 × 0.97)
                  ≈ 1.0   → almost identical direction → very similar
```

### Reading the score
| Score | Meaning |
|---|---|
| 1.0 | identical direction (perfect match) |
| ~0.9 | very close (strong match) |
| ~0.5 | somewhat related |
| 0.0 | perpendicular (unrelated) |
| −1.0 | opposite meaning |

In RAG we usually treat ~0.7–0.8+ as relevant.

### Quick sanity check
Take `[1, 0, 0, 0]` and `[0, 1, 0, 0]`:
- Dot product = (1×0)+(0×1)+(0×0)+(0×0) = **0**
- So cosine similarity = 0 → **perpendicular → completely unrelated.**

### Why cosine and not plain distance?
Cosine cares about **direction (meaning)**, not **magnitude (length)**. Two chunks of very different lengths can mean the same thing — cosine handles that; raw distance would be thrown off by the length difference.

### Higher dimensions
Real embeddings have 384 / 768 / 1536 numbers. The formula is **identical** — you just sum 1536 products instead of 4. You can't picture the angle, but the maths still returns a number between −1 and 1 with the same meaning. (And a computer does those multiplications extremely fast.)

---

## 11. How It's Sent to the LLM (Generation)

Once the top-k chunks are retrieved, we build a prompt:

```
You are an HR assistant. Answer using ONLY the context below.

Context:
[1] Maternity leave is 26 weeks of fully paid leave...
[2] Leave categories include sick, annual, maternity...
[3] Extended maternity options include 8 weeks unpaid...

Question: How long is maternity leave?
```

Send to LLM → it generates:
```
"Maternity leave is 26 weeks of fully paid leave, with an option
 for 8 additional weeks of unpaid leave."
```

**The instruction "answer only from the context" is what suppresses hallucination** — the LLM is told not to invent anything beyond the provided chunks.

### Crucial points to remember
- One query → **one** embedding (not one per word).
- **Same embedding model** for documents AND query (so they live in the same space).
- The LLM **never sees the vector database** — it only sees the retrieved text chunks in the prompt.
- The "smart retrieval" is done by the **vector DB**, not the LLM.

---

## 12. Cons of Naive RAG (with Examples)

Naive RAG relies on **two things both going right**:
1. **Cosine similarity must rank correctly** — the right chunk has to land in the top-k.
2. **The LLM must be smart enough** to pick the right answer out of a mix of chunks.

If either fails → wrong answer. Here are the specific ways it breaks, each with an example.

### Con 1 — Cosine similarity is fooled by surface words
Cosine can rank an **irrelevant** chunk highly just because it shares common words.

> **Query:** "How long is maternity leave?"
> ```
> [A] "Maternity leave is 26 weeks"   0.92  ← actually relevant
> [B] "Sick leave is 12 days"         0.85  ← high ONLY because of the word "leave"
> ```
> Chunk B is noise but still gets retrieved. Cosine can't tell that B doesn't **answer** the question — it only sees that "leave" is shared.

### Con 2 — The top-k trade-off has no good setting
- **top-k too small (e.g. 3):** fast and clean, but you **miss** the right chunk if it happened to rank 4th.
- **top-k too large (e.g. 100):** you won't miss it, but you flood the LLM with noise, waste tokens, and bring back "lost in the middle."

> **Example:** the correct answer chunk ranks 6th. With `top_k=5`, it's **never retrieved** — the LLM answers from the 5 wrong chunks and confidently gives a wrong answer. Bumping to `top_k=50` fixes that one case but drowns every other query in junk.

### Con 3 — No exact-keyword matching
Embeddings capture meaning, but **blur exact tokens** like codes, IDs, and names.

> **Query:** "What's the penalty under clause 7B?"
> The chunk containing "clause 7B" may rank *below* a chunk that merely *talks about penalties in general*, because "7B" carries little semantic weight. Naive RAG has no keyword search to force an exact match. (This is what Hybrid Search later fixes.)

### Con 4 — Chunks lose their context
A chunk ripped out of its document can become ambiguous on its own.

> **Chunk retrieved:** "The fine is ₹50,000."
> Fine for *what*? The section heading ("Late PF filing") was in a different chunk and never got retrieved. The LLM gets a number with no idea what it refers to. (This is what Contextual / Parent-Child RAG later fix.)

### Con 5 — One-shot retrieval can't handle multi-step questions
Naive RAG retrieves **once** and answers. It can't reason across multiple lookups.

> **Query:** "How does our 2024 maternity policy differ from 2023?"
> This needs **two** retrievals (2023 policy *and* 2024 policy) and then a comparison. A single top-k search can't do it — it grabs a muddle of both years and the LLM can't reliably separate them. (This is what Agentic RAG later fixes.)

### Con 6 — It reduces hallucination but doesn't eliminate it
If retrieval returns nothing relevant, a weaker LLM may still **invent** an answer instead of saying "not found."

> **Query:** "What's the work-from-home policy?" — but no such document was ever indexed.
> Retrieval returns the 3 *least bad* chunks (about office attendance), all low similarity (~0.4). A careful LLM says "I don't have that information"; a weaker one stitches the office-attendance chunks into a confident, **made-up** WFH policy. Naive RAG has **no self-check** to catch this. (This is what Corrective / Self-RAG later fixes.)

### Con 7 — Retrieval quality is invisible without testing
The system returns *something* every time, with a similarity score that *looks* confident — even when it's wrong.

> A 0.85 similarity looks reassuring, but if it came from Con 1 (shared word "leave"), the answer is wrong despite the high number. Without a test set of question→correct-answer pairs, **you can't tell good retrievals from bad ones** just by watching scores.

### Summary table

| # | Con | Example | Later fix |
|---|---|---|---|
| 1 | Fooled by shared words | "sick leave" matches "maternity leave" | Reranking |
| 2 | top-k trade-off | right chunk ranks 6th, `top_k=5` misses it | Reranking |
| 3 | No exact-keyword match | "clause 7B" ranks below generic text | Hybrid Search |
| 4 | Chunks lose context | "The fine is ₹50,000" — for what? | Contextual / Parent-Child |
| 5 | One-shot, no multi-step | "compare 2023 vs 2024 policy" | Agentic RAG |
| 6 | Still hallucinates on empty retrieval | invents a WFH policy that was never indexed | Corrective / Self-RAG |
| 7 | Quality is invisible | confident 0.85 score on a wrong match | evaluation / testing |

> The golden rule: **start with Naive RAG, then add exactly the fix for the failure you actually measure** — don't pile on complexity before you've seen which con is hurting you. Cons 1 and 2 point straight to **Reranking RAG** as the usual first upgrade (covered in separate notes).

---

## 13. The Feynman Test — Can You Explain These Simply?

1. Why can't we just give the LLM the whole document?
2. What's a chunk, and why do we make them?
3. What does it mean to "embed" text, and why a *list* of numbers?
4. Where did the embedding model's sense of meaning come from?
5. What's the difference between an embedding, a vector, and a vector database?
6. Why use a vector database instead of SQL?
7. Are the query and the chunks embedded together or separately?
8. How do you get a cosine angle out of two lists of numbers?
9. Who sets top-k, and what's the trade-off?
10. What does the LLM actually see when it answers?
11. When and why does Naive RAG fail?

---

## 14. One-Line Summary

> Naive RAG chops your documents into chunks, turns each into a meaning-vector, and stores them in a vector database; at question time it turns the query into a vector too, uses cosine similarity to grab the closest top-k chunks, and hands only those to the LLM with an instruction to answer strictly from them — giving the model the right "pages" instead of letting it guess.
