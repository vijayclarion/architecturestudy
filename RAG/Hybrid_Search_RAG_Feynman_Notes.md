# Hybrid Search RAG — Feynman Notes

*Notes written as if explaining to a smart friend. Companion to the Reranking RAG notes.*

---

## 1. The Problem Hybrid Search Solves

Naive RAG retrieves with **dense vector search** (cosine similarity). It's great at *meaning* — "fix my slow laptop" finds "improve computer performance" even with no shared words. But it has a blind spot: **exact tokens**.

Product codes, error codes, part numbers, names, acronyms, legal citations, function names — embeddings *smear* these into a fuzzy "roughly similar" region. A code like `XR-4400` has no meaning to lean toward; it just needs to be **matched literally**. Cosine search can rank the one chunk that literally contains it *below* chunks that are merely on-topic.

> Example: for "How do I fix error XR-4400?", the chunk that literally names XR-4400 can rank *3rd* on cosine — beaten by vaguely-related chunks — because the exact code earned no special credit.

Hybrid search fixes this by running a **second** search that *is* good at exact words, and fusing the two.

---

## 2. The Core Idea (One Sentence)

> Search by **meaning** and by **literal spelling** at the same time, then merge the two opinions into one list.

- **Dense search (cosine):** finds chunks similar in *meaning*.
- **Sparse search (BM25):** finds chunks containing the *exact words*.
- **Fusion:** a referee that reconciles the two rankings into one.

Crucially this is **not** "cosine *then* fusion" in a chain. It's **(cosine search) ∥ (keyword search) → fusion**. Two searches in parallel, then merged.

---

## 3. The Two-Layer Net Analogy (The Feynman Hook)

From the reranking notes: vector search is a net you cast to catch fish.

The trouble is a pure-meaning net has holes exactly where exact words live — it lets the `XR-4400` fish slip through because that fish doesn't "smell" semantically special.

**Hybrid search upgrades the net to two layers laid on top of each other:**

- Layer 1 (dense): catches *meaning* fish.
- Layer 2 (BM25): catches *exact-word* fish.

A fish caught by *either* layer is in your haul. The answer that meaning-search would have dropped gets caught by the word-search layer instead.

(Reranking is still the separate, later step: hand-inspecting whatever the net caught. See §7.)

---

## 4. The Two Searches In Detail

### Search A — Dense / vector (good at meaning)
1. Embed the query into one vector.
2. Score every chunk by **cosine similarity**.
3. Rank by meaning-closeness.
- **Weakness:** exact tokens (codes, names, acronyms) get blurred.

### Search B — Sparse / BM25 (good at exact words)
1. No embeddings. Count how often the query's exact words appear in each chunk.
2. Weight **rare** words heavily (a distinctive code counts far more than "the").
3. Rank by literal word overlap.
- **Weakness:** zero understanding of meaning — misses synonyms and paraphrases entirely.

The two are deliberately chosen because they **fail in opposite ways**. Where one is blind, the other sees.

---

## 5. Fusion — Combining Two Rankings

You now have **two ranked lists that disagree**, on **incompatible scales** (cosine ~0–1, BM25 unbounded like 9.2). You can't just add the raw scores. So fusion uses **rank position**, not raw score.

### Reciprocal Rank Fusion (RRF) — the standard method
For each chunk, in each list, score `1 / (k + rank)` (k is a constant, usually 60), then **sum across both lists**.

- A chunk ranked high in **both** lists floats to the top.
- A chunk only **one** method liked still gets partial credit — so a strong exact-match isn't thrown away just because cosine ignored it.

> Key point: fusion is **arithmetic**, not a model. No neural network reads the text here. It's a referee combining two opinions by their positions.

---

## 6. Worked Example (End To End)

**Knowledge base (5 chunks):**
- C1: "Error XR-4400 occurs when the payment gateway times out."  ← the real answer
- C2: "To resolve payment failures, restart the transaction service."
- C3: "The billing system processes refunds within 5 days."
- C4: "Gateway timeout issues are logged in the error console."
- C5: "Annual leave policy allows 24 days per year."

**Query:** "How do I fix error XR-4400?"

### Search A — Dense (cosine)
| Chunk | cosine | Rank |
|---|---|---|
| C4 | 0.81 | 1 |
| C2 | 0.78 | 2 |
| **C1** | 0.75 | **3** |
| C3 | 0.42 | 4 |
| C5 | 0.11 | 5 |

Problem: **C1 is the actual answer but cosine ranked it 3rd** — the exact code got no special credit.

### Search B — BM25 (exact words)
| Chunk | BM25 | Rank |
|---|---|---|
| **C1** | 9.2 | **1** |
| C4 | 2.1 | 2 |
| C2 | 0.0 | 3 |
| C3 | 0.0 | 4 |
| C5 | 0.0 | 5 |

BM25 **nailed C1 to the top** because "XR-4400" is a rare, distinctive token.

### Fusion (RRF, k=60)
| Chunk | dense rank | sparse rank | RRF score | Fused rank |
|---|---|---|---|---|
| C4 | 1 | 2 | 1/61 + 1/62 = 0.0325 | 1 |
| **C1** | 3 | 1 | 1/63 + 1/61 = 0.0323 | **2** |
| C2 | 2 | 3 | 1/62 + 1/63 = 0.0320 | 3 |
| C3 | 4 | 4 | 0.0313 | 4 |
| C5 | 5 | 5 | 0.0308 | 5 |

**The rescue:** cosine alone buried C1 at position 3. Because BM25 put it at position 1, fusion lifted C1 to position 2 — right at the top. *Neither search alone gave the best ordering; the fusion did.*

---

## 7. Hybrid Search vs Reranking (The Confusion To Kill)

Both "combine scores," so they get mixed up. They are **different jobs at different stages** — and good pipelines use **both**.

| | Hybrid Search | Reranking |
|---|---|---|
| What it combines | Two *retrieval methods* (meaning + exact words) | One candidate list, re-scored |
| When it runs | **IS** the retrieval step | **After** retrieval |
| How it scores | Arithmetic (RRF on rank positions) | A trained **cross-encoder** reads query+chunk together |
| Job it fixes | **Recall** — getting the answer *into* the list | **Precision** — getting the answer *to the top* of the list |
| Per-query cost | Cheap (two index lookups + formula) | Pricey (model pass per candidate, ~hundreds of ms) |

### The iron rule that makes them non-substitutes
> **Reranking can only reorder what retrieval already found.**

If the right chunk isn't in the candidate list, no reranker can recover it.

**Proof with C1:** suppose pure cosine retrieved only the top 3 → C4, C2, C3. C1 ranked 4th on cosine, so it's **not in the list**. Now rerank C4, C2, C3 with a brilliant cross-encoder — it carefully reorders them, but **C1 was never there**. The real answer is gone and reranking *cannot* bring it back.

Hybrid search is what puts C1 in the list in the first place (BM25 caught the exact code). *Then* reranking has something worth ranking.

### Full pipeline order
```
Query
  → (cosine search  ∥  BM25 search)     ← hybrid retrieval
  → fusion merges the two lists (RRF)
  → top ~20 candidates
  → [optional cross-encoder reranker]   ← reranking, separate & later
  → top 3
  → LLM
```

---

## 8. When To Use Hybrid Search

**Use when:**
- Queries contain **exact tokens** embeddings smear: product/error codes, part numbers, names, acronyms, legal citations, code identifiers.
- You suspect a **recall** problem — the right chunk sometimes never gets retrieved.
- Your corpus mixes natural language with structured identifiers (support tickets, technical docs, legal text).

**Reach for reranking instead (or as well) when:**
- The right chunk *is* retrieved but sits **below plausible-looking distractors** (a precision problem) — e.g. sick-leave outranking maternity-leave because both are "about leave."

**Decision rule:**
> Worried the answer won't be **retrieved at all**? → Hybrid search (recall).
> Worried the answer is retrieved but **ranked below junk**? → Reranking (precision).
> Most real corpora have **both** problems → use **both**: hybrid to widen the net, reranker to clean the catch.

**Costs / trade-offs:**
- Maintains a **second index** (BM25 alongside vectors) — setup cost, but cheap per query.
- Fusion adds a **weighting/`k` knob** to tune between the two methods.
- More moving parts than single-vector search; usually worth it when exact terms matter.

---

## 9. Common Misconceptions

| Wrong | Right |
|---|---|
| "Hybrid does cosine, then fusion." | Cosine is *one* of two parallel searches; fusion combines it with a *second* (BM25) search. |
| "Fusion is a smart model judging relevance." | Fusion is arithmetic (RRF) combining *rank positions*. No model reads text. |
| "Hybrid search and reranking are alternatives." | Different stages, different jobs. Often used together. |
| "Reranking can fix a missing chunk." | No — it can only reorder what retrieval already found. |
| "BM25 understands meaning." | No — it matches exact words. Its strength is precisely that it *doesn't* rely on meaning. |
| "You add the cosine and BM25 scores." | Scales differ (0–1 vs unbounded); RRF uses ranks, not raw scores. |

---

## 10. The Feynman Test

If you can answer these in plain language, you understand Hybrid Search RAG:

1. What blind spot of cosine similarity does hybrid search fix?
2. What are the two searches, and how do they fail in *opposite* ways?
3. Why can't you just add the cosine and BM25 scores together?
4. What does Reciprocal Rank Fusion actually combine?
5. In the XR-4400 example, *why* did C1 get rescued?
6. Why is hybrid search NOT the same as reranking?
7. What's the difference between a recall problem and a precision problem — and which technique fixes which?

---

## 11. One-Line Summary

> Hybrid search runs a meaning-search (cosine) and an exact-word-search (BM25) in parallel and fuses their rankings (RRF) so the answer gets *into* the candidate list — fixing recall; reranking is a separate later step that reorders that list — fixing precision. Hybrid widens the net; reranking inspects the catch; you usually want both.
