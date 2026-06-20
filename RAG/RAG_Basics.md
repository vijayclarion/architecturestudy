# RAG (Retrieval Augmented Generation) — Feynman Notes

*Notes written as if explaining to a smart non-technical friend.*

---

## 1. The Problem We're Solving

Large Language Models (LLMs) like ChatGPT or Claude have two big limitations:

**Limitation 1 — Knowledge Cutoff.**
They were trained on data up to a certain date. Anything that happened after that date, they don't know. If you ask "what's the latest iPhone?", they might give you outdated info or make something up (hallucinate).

**Limitation 2 — They don't know your private data.**
They have no clue about your company's HR policies, your internal documents, your team's wiki. Their training data was the public internet, not your company's SharePoint.

**The naive workaround that doesn't work well:**
Just paste the whole document into the prompt and ask. This fails because:
- Documents are huge → hits the token limit
- Even when it fits, the LLM gets "lost in the middle"—important details buried in a long prompt get ignored
- Costs a lot per query (you pay for every token)

So we need a smarter approach.

---

## 2. The Book Analogy (The Feynman Hook)

Imagine you're studying for a history exam. Your textbook is 500 pages.

- **Without an index**: You read cover-to-cover hoping to find the topic. You get tired, you lose focus, you might miss the right page.
- **With an index**: You look up "French Revolution," it says page 234, you jump straight there. Fast and accurate.

LLMs are like a student who has read many books but not yours. **RAG gives the LLM an index into your specific documents.** When a question comes in, we look up the right "pages" first, hand them to the LLM, and ask it to answer.

That's RAG in one sentence:
> Find the relevant pages first. Then ask the LLM to answer using only those pages.

---

## 3. The Three Phases of RAG

### Phase A — Indexing (done once, offline)
1. Take your documents
2. Break them into smaller pieces (chunks)
3. Convert each chunk into numbers (embeddings)
4. Store those numbers in a special database (vector database)

### Phase B — Retrieval (happens per query)
1. User asks a question
2. Convert the question into numbers too (using the same embedding model)
3. Find the chunks whose numbers are closest to the question's numbers
4. Return the top few (e.g., top 3 chunks)

### Phase C — Generation (happens per query)
1. Build a prompt: instructions + retrieved chunks + the user's question
2. Send to LLM
3. LLM writes an answer using the chunks
4. Return to user

---

## 4. Chunking — Breaking Documents Into Pieces

### Why chunk at all?
- LLMs have token limits—can't fit whole docs
- Smaller, focused chunks → more precise retrieval
- Avoids "lost in the middle"

### The chunking strategies

**Fixed-size, no overlap**
- "Cut every 500 tokens, regardless of content."
- Simple. Predictable. But it can cut mid-sentence.
- Example: chunk ends with "Maternity leave is 26..." and next starts with "weeks paid."
- Neither chunk fully answers the question.

**Fixed-size with overlap**
- Same as above, but last 50 tokens of chunk N repeat at start of chunk N+1.
- Why overlap matters: if you say "I am playing in the | ground" with a hard break, you lose the connection between the two halves of the thought. Overlap is the bridge.
- Trade-off: slightly more storage, sometimes two chunks return the same info.
- **Good default choice.**

**Sentence-based**
- Break at sentence boundaries.
- Never cuts mid-sentence, but sentences vary wildly in length.
- "Yes." is one chunk. A 200-word legal sentence is another.

**Structural (paragraphs / sections)**
- Respects the document's own structure (headings, sections, bullets).
- Great if the document is well-formatted (e.g., markdown, manuals).
- Fails if a section is huge—then you're back to "lost in the middle."

**Semantic chunking (using embeddings)**
- Embed each sentence, group consecutive sentences that mean similar things, break when the topic shifts.
- Natural, meaningful chunks.
- Expensive—you embed twice (once to chunk, once to store).

**LLM-based chunking**
- Send the document to an LLM and ask: "split into self-contained topic chunks."
- The LLM understands meaning, so it splits beautifully.
- Slow and expensive—one LLM call per document.
- Some versions also rewrite each chunk to be self-contained (resolving pronouns, adding context).

### How to choose
| Situation | Strategy |
|---|---|
| Prototype, simple text | Fixed-size with overlap |
| Well-structured docs (markdown, manuals) | Structural |
| Messy/unstructured text | Semantic |
| Mission-critical accuracy | LLM-based |
| Legal/transcripts | Sentence-based |

---

## 5. Embeddings — Turning Text Into Numbers

### What is an embedding?
A neural network reads a piece of text and outputs **a list of numbers**—a vector.

- Whole chunk goes in → **one vector** comes out (NOT one number per word).
- The vector is typically 384, 768, or 1536 numbers long.
- The same model is used for documents AND queries—so they land in the same "meaning space."

### What do the 1536 numbers mean?
Each number is a "dimension" of meaning the model learned during training.

**Analogy**: Imagine describing a person with numbers—height, age, introversion, love of music, years of education. With enough such dimensions, you can describe anyone meaningfully. Two similar people have similar numbers.

Embeddings work the same way for text. The model decided what each dimension captures—not humans. Dimension 47 might roughly mean "is this about time?", dimension 312 might mean "is this about work?" But we don't need to inspect them. What matters is the model is consistent: similar meanings → similar numbers.

### Why this is powerful
The phrases:
- "Maternity leave is 26 weeks"
- "Female employees get six months off after childbirth"

share almost no words, but their embeddings are very close. **Embeddings capture meaning, not word overlap.**

---

## 6. Vector Database — Where Embeddings Live

Each chunk is stored as a row:

```
id: 47
text: "Maternity leave at Wipro is 26 weeks..."
vector: [0.024, -0.183, 0.451, ..., 0.117]  (1536 numbers)
metadata: {source: "HR_Policy.pdf", page: 12}
```

The DB uses a special index (HNSW—a graph of nearest neighbors) so that searching across millions of vectors takes milliseconds. Without this index, every query would compare against every stored vector—too slow.

---

## 7. Cosine Similarity — How "Closeness" Is Measured

### The concept
Each vector is an arrow from origin (0,0,…) pointing somewhere. Cosine similarity measures the **angle** between two arrows—not their length.

- Small angle → arrows point similar direction → similar meaning → score near 1.0
- 90° angle → arrows perpendicular → unrelated → score near 0.0
- 180° → arrows opposite → opposite meaning → score near -1.0

### The math (2D example)
Vector A = [3, 4], Vector B = [4, 3]

**Step 1 — Dot product:**
```
A · B = (3 × 4) + (4 × 3) = 12 + 12 = 24
```

**Step 2 — Lengths:**
```
|A| = √(3² + 4²) = √25 = 5
|B| = √(4² + 3²) = √25 = 5
```

**Step 3 — Cosine similarity:**
```
cos_sim = 24 / (5 × 5) = 0.96  → very similar
```

### Higher dimensions
The formula is identical for 1536 dimensions—you just sum 1536 products instead of 2. You can't visualize the angle, but the math still gives a number between -1 and 1 with the same meaning.

### Why cosine, not distance?
Cosine cares about direction (meaning), not magnitude (length). Two chunks can be very different lengths but mean the same thing. Cosine handles that; raw distance doesn't.

---

## 8. The Complete End-to-End Flow

### Setup (one-time, offline)

```
HR Policy PDF (50 pages)
        ↓
    Extract text
        ↓
    Chunk (~100 chunks)
        ↓
    Embed each chunk → 100 vectors
        ↓
    Store in vector DB with metadata
```

### Query time

User asks: **"How long is maternity leave?"**

```
Query "How long is maternity leave?"
        ↓
Embed the WHOLE query (one vector)
        ↓
Vector DB finds top-3 closest chunks (by cosine similarity)
        ↓
Top results:
  [1] "Maternity leave at Wipro is 26 weeks..."        (0.91)
  [2] "Leave categories include casual, sick, mat..."  (0.78)
  [3] "Extended maternity options include 8 weeks..."  (0.74)
        ↓
Build the prompt:

    "You are an HR assistant. Answer using only the context below.
     
     Context:
     [1] Maternity leave at Wipro is 26 weeks of paid leave...
     [2] Leave categories at Wipro include...
     [3] Extended maternity options include...
     
     Question: How long is maternity leave?"
        ↓
Send to LLM
        ↓
LLM generates:
"Maternity leave at Wipro is 26 weeks of fully paid leave.
 You're eligible after 80 days of service. There's also an
 option for 8 additional weeks of unpaid leave."
        ↓
Return to user
```

### Crucial points to remember
- One query → **one** embedding (not per word)
- Same embedding model for docs AND query (so they live in same space)
- The LLM **never sees the vector database**—it only sees the retrieved text chunks placed in the prompt
- The "smart retrieval" is done by the vector DB, not the LLM

---

## 9. Types of RAG — When to Use What

Start naive. Add complexity only when you measure a specific failure.

### Naive / Standard RAG
The baseline. Embed → search → top-k → prompt → answer.
**Use when**: any starting point; moderate accuracy needs.

### Hybrid Search RAG
Combines vector search (semantic) with keyword search (BM25).
**Use when**: your domain has product codes, IDs, specific names ("Form-16", "SKU-A2837")—things that need exact matching.

### Reranking RAG
Retrieve more chunks (~20), then re-score them with a specialized reranker model. Keep top 3-5.
**Use when**: initial retrieval is noisy. Two-stage retrieval improves quality.

### Multi-Query RAG
LLM rewrites the query into 3-5 variations, searches for each, merges results.
**Use when**: user questions are short, vague, or ambiguous.

### HyDE (Hypothetical Document Embeddings)
LLM generates a fake answer first; embed that fake answer; search using it.
**Use when**: short queries don't match well against long answer chunks. Counterintuitive but powerful.

### Parent-Child / Hierarchical RAG
Embed small chunks for precise matching, but return larger parent chunks for context.
**Use when**: precise retrieval matters but the LLM needs more surrounding context to answer.

### Contextual RAG
Before embedding, prepend a short context line to each chunk (e.g., "Section 4.2 of HR policy, under Maternity Leave:").
**Use when**: chunks lose meaning without context. Significantly improves accuracy.

### Agentic RAG
LLM treats retrieval as a tool, decides when/what to search, can do multi-step reasoning.
**Use when**: questions need multi-hop reasoning. "Compare our 2023 and 2024 maternity policies."

### Graph RAG
Build a knowledge graph from documents; traverse it for context.
**Use when**: information is highly interconnected (org structures, legal precedents).

### Corrective / Self-RAG
LLM evaluates if retrieved chunks actually answer the question; re-searches if not.
**Use when**: hallucination is unacceptable—legal, medical, financial.

### The progression
```
Naive RAG
   ↓ (keyword queries failing?)
+ Hybrid Search
   ↓ (retrieval still noisy?)
+ Reranker
   ↓ (vague queries failing?)
+ Multi-query or HyDE
   ↓ (chunks losing context?)
+ Contextual RAG or Parent-Child
   ↓ (need reasoning?)
+ Agentic RAG
   ↓ (hallucination risk?)
+ Corrective RAG
```

---

## 10. Common Misconceptions to Address in Talk

| Wrong | Right |
|---|---|
| "The LLM searches the vector database." | The DB does the search. The LLM only sees retrieved text. |
| "Each word is embedded separately." | Whole chunk → one vector. |
| "Bigger embedding dimensions = always better." | Trade-off with cost, speed, storage. |
| "Cosine similarity only works in 2D." | Works in any dimensions; math is identical. |
| "RAG eliminates hallucination." | It reduces it; doesn't eliminate it. Use corrective RAG if critical. |
| "More retrieved chunks = better answer." | Too many chunks cause "lost in the middle" again. 3-5 is usually optimal. |

---

## 11. The Feynman Test — Can You Explain These Simply?

If you can answer all of these in plain language, you understand RAG:

1. Why can't we just give the LLM the whole document?
2. What's a chunk? Why do we overlap chunks?
3. What does it mean to "embed" text?
4. What is the vector database storing?
5. How does the system know which chunks are relevant to a query?
6. What does the LLM actually see when it answers?
7. When would naive RAG fail, and what would you add?

---

## 12. Quick Talk Outline

1. **Hook** (2 min) — Studying for an exam without an index. That's an LLM without RAG.
2. **The Problem** (3 min) — Knowledge cutoff, private data, "lost in the middle."
3. **The Three Phases** (5 min) — Indexing, retrieval, generation.
4. **Chunking** (5 min) — Why, strategies, trade-offs. Use the "playing in the ground" example for overlap.
5. **Embeddings** (5 min) — Text as numbers. The person-with-dimensions analogy.
6. **Vector DB + Cosine Similarity** (5 min) — Arrows in space, the angle between them.
7. **Live walk-through** (5 min) — Trace "How long is maternity leave?" end to end.
8. **Types of RAG** (5 min) — Progression based on failure modes.
9. **Q&A** (5 min)

Total: ~40 minutes plus Q&A.
---

## 13. Things i don't know
1. How embedding convert to text
2. Detail of each RAG type like Reranking, Multi Model Query 