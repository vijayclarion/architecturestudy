# Reranking RAG — Feynman Notes

*Notes written as if explaining to a smart friend. Companion to the main RAG notes.*

---

## 1. The Problem Reranking Solves

Naive RAG retrieves chunks using vector search (cosine similarity). It's fast — milliseconds across millions of chunks — but it's **rough**. Irrelevant chunks sneak into the top results because cosine similarity only measures whether two vectors *point in a similar direction*. It cannot tell whether a chunk actually *answers* the question.

> Example: for the query "How many weeks of maternity leave?", a chunk about *sick leave* can score high on cosine similarity — both are about "leave" — even though it's the wrong answer entirely.

Reranking adds a second, smarter pass that fixes this.

---

## 2. The Core Idea (One Sentence)

> Cast a wide net cheaply, then carefully pick the best fish.

- **Stage 1 — wide net:** fast vector search retrieves ~20 candidate chunks.
- **Stage 2 — careful pick:** a slow but accurate model re-scores those 20 and keeps the best 3.

You only pay the expensive cost on the small shortlist, never on the whole database.

---

## 3. The Fishing Analogy (The Feynman Hook)

Imagine fishing for the 3 best fish in a huge lake.

- You **can't** inspect every fish in the lake individually — too slow.
- So you cast a wide net and haul in 20 fish quickly (Stage 1 — vector search).
- Then you examine those 20 carefully, one by one, and keep the 3 best (Stage 2 — reranker).

The net is cheap and rough. The hand-inspection is expensive and precise. Used together, you get both speed and accuracy.

---

## 4. The Two Stages In Detail

### Stage 1 — Vector search (fast, rough)
1. Embed the query into one vector.
2. Find the top ~20 chunks by cosine similarity.
3. These are *candidates*, not final answers.

### Stage 2 — Reranking (slow, precise)
1. Take each (query, chunk) pair.
2. Feed it through a **cross-encoder** model that outputs a relevance score.
3. Sort all 20 by this new score.
4. Keep the top 3–5.
5. Send only those to the LLM to write the final answer.

The LLM never sees the noisy 20 — only the clean, reranked top 3.

---

## 5. Bi-Encoder vs Cross-Encoder (The Heart Of It)

The single most important distinction: **when does the query meet the chunk?**

### Bi-encoder — résumés on file
- Query and chunk are encoded **separately**, each into its own vector.
- Comparison is done afterward by measuring the angle between vectors (cosine similarity).
- Because chunk vectors are **precomputed once** and stored, you can scan millions in milliseconds.
- **Weakness:** the query and chunk never "see" each other during encoding, so nuance is lost.
- **Used for:** Stage 1 retrieval.

> Analogy: a dating agency matching people purely by comparing pre-filled questionnaires. Fast, but no real chemistry test.

### Cross-encoder — a live interview
- Query and chunk are glued into **one sequence** and processed **together**.
- The model can directly judge: *does this passage answer this question?*
- **Weakness:** nothing can be precomputed — every (query, chunk) pair needs a fresh full pass. Very slow at scale.
- **Used for:** Stage 2 reranking.

> Analogy: sitting two people down for an actual interview and watching the chemistry. Accurate, but you can't do it a million times.

### Why two stages exist
You can't run the slow cross-encoder on millions of chunks, and the fast bi-encoder alone lets junk through. So:

| | Bi-encoder (Stage 1) | Cross-encoder (Stage 2) |
|---|---|---|
| Speed | Fast | Slow |
| Accuracy | Rough | Precise |
| Precompute? | Yes (vectors stored) | No (fresh per pair) |
| Runs on | Millions of chunks | ~20 candidates |

The bi-encoder narrows millions → 20. The cross-encoder refines 20 → 3. Best of both.

---

## 6. Inside The Cross-Encoder (How It Actually Works)

Four steps from input to score.

### Step 1 — Glue query and chunk into one sequence
Concatenate with special marker tokens:

```
[CLS] how many weeks [SEP] 26 weeks paid [SEP]
       \___query___/        \__chunk__/
```

- `[SEP]` separates query from chunk so the model knows the boundary.
- `[CLS]` is an empty "notepad" token at the front that will collect a summary of the whole pair.

### Step 2 — Attention fires across the boundary
Because query and chunk are now in **one** sequence, attention lets the query word "weeks" look directly at the chunk's "26 weeks". The model computes things like *"the question asks for a duration, and this passage has a number + time unit → match."*

**This cross-talk is exactly what a bi-encoder cannot do**, because in a bi-encoder the two were sealed in separate boxes before they ever met.

### Step 3 — Read the [CLS] summary slot
After all transformer layers finish mingling the tokens, the model looks only at the vector sitting in the `[CLS]` position. It now encodes a rich summary of "how well does this chunk answer this query?"

### Step 4 — Scoring head squashes it to one number
The `[CLS]` vector is passed through a tiny final layer to produce a single relevance score between 0 and 1.

```
[CLS] vector → linear layer → sigmoid → 0.97
```

**Summary to remember:** query and chunk go in *together* → attention connects them → one score comes out.

---

## 7. How The Score Is Actually Computed

The model does **not** "decide" a rating like a person. The score falls out of arithmetic: a learned weighted sum, then a squash.

### The [CLS] vector = a list of feature values
Each dimension captures some learned feature. (Real models have hundreds; the model learned what each means — humans don't label them.) Illustrative example:

| Dimension | Value | Learned weight |
|---|---|---|
| about duration? | 0.80 | +2.1 |
| has a number?   | 0.70 | +1.6 |
| about leave?    | 0.90 | +1.8 |
| off-topic?      | 0.10 | −2.4 |

- **Positive weight** = this feature votes "relevant."
- **Negative weight** = this feature pushes the score *down* (e.g. "off-topic?").
- **Size of weight** = how loud the vote is.

### Move 1 — Weighted sum (a dot product)
Multiply each value by its weight, add them all, plus a bias constant:

```
z = (0.80 × 2.1) + (0.70 × 1.6) + (0.90 × 1.8) + (0.10 × −2.4) + (−0.5)
z = 1.68 + 1.12 + 1.62 − 0.24 − 0.5
z = 3.68
```

`z` is the raw, unbounded "evidence tally." This is the same dot-product idea from cosine similarity — multiply matching positions, sum them — but with **learned** weights instead of another vector.

### Move 2 — Sigmoid squash
`z` could be any number, but a score must live between 0 and 1. The sigmoid bends it into range:

```
score = sigmoid(z) = 1 / (1 + e^(−z))
score = 1 / (1 + e^(−3.68)) ≈ 0.98
```

- Big positive z → close to 1.
- Big negative z → close to 0.
- z = 0 → exactly 0.5.

### The takeaway
The cross-encoder doesn't reason in words. It computes `dot product → sigmoid`. All the "intelligence" lives in the learned weights, tuned during training on millions of (query, good-chunk, bad-chunk) examples until the math reliably produces high numbers for good matches.

---

## 8. Worked Example (End To End)

**Query:** "How many weeks of maternity leave do employees get?"

After Stage 1 (vector search), candidates ranked by cosine similarity:

| Rank | Chunk | cosine |
|---|---|---|
| 1 | Maternity leave is 26 weeks fully paid... | 0.91 |
| 2 | Extended maternity: 8 more unpaid weeks... | 0.88 |
| 3 | Leave categories: casual, sick, maternity... | 0.85 |
| 4 | Sick leave is 12 days per year... | 0.82 |
| ... | ... | ... |
| 7 | Eligible under Maternity Benefit Act §5... | 0.74 |

After Stage 2 (cross-encoder reranking):

| New rank | Chunk | reranker | What changed |
|---|---|---|---|
| 1 | Maternity leave is 26 weeks fully paid... | 0.97 | stayed top |
| 2 | Extended maternity: 8 more unpaid weeks... | 0.89 | stayed |
| 3 | Eligible under Maternity Benefit Act §5... | 0.86 | **jumped up** (directly answers) |
| ... | Leave categories list... | 0.61 | dropped (too generic) |
| bottom | Sick leave is 12 days... | 0.28 | **collapsed** (wrong leave type) |

Only the top 3 reranked chunks go to the LLM. The sick-leave chunk that fooled cosine similarity is now correctly discarded.

---

## 9. When To Use Reranking

**Use when:**
- Initial vector search feels noisy — unrelated chunks in the top-K.
- You need high precision in the final answer.
- It's an easy, high-value add-on to an existing Naive RAG pipeline.

**Costs / trade-offs:**
- Adds latency (~100–500ms per query) since the cross-encoder runs per candidate.
- Only helps if Stage 1 retrieval is decent — if the right chunk isn't in the top 20, reranking can't recover it.
- Choosing how many candidates to retrieve (20? 50?) is a tuning knob.

---

## 10. Common Misconceptions

| Wrong | Right |
|---|---|
| "Reranking searches the database again." | It only re-scores the ~20 candidates Stage 1 already found. |
| "The reranker uses cosine similarity too." | No — it uses a cross-encoder that reads query + chunk together. |
| "Bi-encoder and cross-encoder are the same model." | Different designs. Bi = separate vectors; cross = joint pass. |
| "The model decides a score by reasoning." | It computes a learned weighted sum, then a sigmoid. |
| "More candidates is always better." | More candidates = more cross-encoder calls = more latency. |
| "Reranking can fix bad Stage 1 retrieval." | No — it can only reorder what Stage 1 retrieved. |

---

## 11. The Feynman Test

If you can answer these in plain language, you understand Reranking RAG:

1. Why isn't vector search alone good enough?
2. What are the two stages, and why two?
3. What's the difference between a bi-encoder and a cross-encoder?
4. When do the query and chunk "meet" in each one?
5. What do the `[CLS]` and `[SEP]` tokens do?
6. How does the cross-encoder turn a chunk into a single 0–1 score?
7. Why is reranking slow, and why is that acceptable?

---

## 12. One-Line Summary

> Vector search casts a cheap wide net (bi-encoder, fast); the reranker hand-inspects the catch (cross-encoder, accurate) by reading query and chunk *together* and squashing the result into a relevance score — keeping only the best few for the LLM.
