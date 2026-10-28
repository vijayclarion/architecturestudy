# RAG, From First Principles
## Hierarchical RAG & Contextual RAG — Feynman-style working notes

> *"What I cannot create, I do not understand. Know how to solve every problem that has been solved."*
> — R. P. Feynman (on the blackboard, 1988)

**How to read these notes.** The rule here is simple: never accept a name as an explanation. We start each idea with the **problem it is forced to solve**, then build the machinery only as far as the problem demands, then walk one **concrete numerical example** all the way through. If you can't follow the example with a pencil, the explanation has failed.

---

## 0. The Problem Both Methods Are Solving

Before either method, there is plain ("naive") RAG. You chop a pile of documents into small chunks, turn each chunk into a vector, and at question-time you fetch the chunks whose vectors sit nearest the question's vector. Hand those to a language model and let it answer.

This works astonishingly often. So why complicate it? Because chopping has two nasty side-effects, and each advanced method is a direct cure for one of them.

> **The two diseases of a naive chunk**
>
> 1. **Amnesia (the orphaned-chunk problem).** A chunk torn from its page forgets where it came from. "Revenue grew 3% over the previous quarter" — whose revenue? which quarter? The vector encodes none of that, so the right chunk never surfaces for the question "What was ACME's Q2-2023 growth?"
> 2. **Flatness (the lost-structure problem).** All chunks live in one giant flat pile. A 300-page manual and a sticky-note are searched the same way. Questions that need the big picture ("summarise chapter 4") or need to hop across sections drown in thousands of tiny, equally-ranked fragments.

**Contextual RAG** cures disease #1 — amnesia. **Hierarchical RAG** cures disease #2 — flatness. Keep that mapping in your head and everything below is just detail.

---

## 1. Contextual RAG

### 1.1 The problem, stated sharply

Take a real chunk pulled from the middle of an annual report:

```
"Margins improved due to the new supplier
 agreement signed in March."
```

On its own this sentence is nearly useless to a search engine. There is no company, no year, no product line, no department. Embed it and it lands in a vague neighbourhood of "business-y" vectors. Ask *"Why did TechNova's hardware margins rise in 2024?"* and the embedding has nothing to grab onto. The chunk has **amnesia**.

### 1.2 The idea (in one breath)

> Before you store a chunk, ask a language model to read the **whole document** and write a one- or two-sentence note saying *where this chunk lives and what it is about*. Glue that note onto the front of the chunk, and store **that**. Now the chunk carries its own address.

Our amnesiac sentence becomes:

```
From TechNova Inc.'s FY2024 annual report,
Operating Costs section, explaining gross-margin
gains in the hardware division. Margins improved
due to the new supplier agreement signed in March.
```

Same facts, but now the entity, the year, the section and the topic are all *inside* the text that gets embedded. The cure is almost embarrassingly direct.

### 1.3 Step by step — INDEXING (the write path, done once)

1. **Load.** Read the entire source document into memory. You need all of it, because the context-note must see the whole thing.
2. **Chunk.** Split into ~200–500-token pieces (a little overlap is fine). Keep each raw chunk plus its position.
3. **Contextualise.** For each chunk, prompt an LLM with the whole document and that chunk, asking only for a short situating note. *Trick: cache the document in the prompt so all chunks of one document share its cost.*
4. **Enrich.** Prepend the note to the raw chunk → the "enriched chunk."
5. **Index into TWO stores.** Embed the enriched chunk into a vector DB, AND feed the same enriched text to a BM25 keyword index. Keep the original raw chunk mapped by id, so you can choose later what to show the model.

> **Why two stores, not one?** Embeddings are great at **meaning** ("car" ≈ "automobile") but clumsy with **exact tokens** — part numbers, error codes, surnames, dates. BM25 (classic keyword search) is the opposite: blind to meaning, surgical on exact strings. Run both and you get the best of each. This pairing is the heart of "Contextual Retrieval."

### 1.4 Step by step — RETRIEVAL (the read path, per question)

1. **Embed the question.** Turn the user query into a vector.
2. **Vector search.** Cosine similarity against the vector DB → top-K (say K = 150).
3. **BM25 search** *(in parallel, not after).* Keyword score against the BM25 index → its own top-K.
4. **Rank fusion.** Merge the two ranked lists by *position*, not raw score, using Reciprocal Rank Fusion (RRF). A chunk that ranks well in **both** lists rises to the top.
5. **Rerank** *(recommended).* A cross-encoder reranker reads the question against each surviving candidate and reorders them. Keep the top ~20.

The fusion arithmetic, so it isn't magic:

```
score(doc) = Σ   1 / (k + rank_in_list)     # k ≈ 60
             lists

# A doc at rank 2 in vectors and rank 5 in BM25:
# 1/(60+2) + 1/(60+5) = 0.0161 + 0.0154 = 0.0315
# beats a doc that was rank 1 in only ONE list (0.0164).
```

RRF works precisely *because* it throws away the raw scores. Cosine sits in [-1, 1]; BM25 can be any positive number. Averaging them is comparing temperature to weight. Ranks are a common currency.

### 1.5 Step by step — AUGMENTATION & GENERATION

1. **Select.** Take the top-N reranked chunks (often 5–20). Enriched or raw, your choice — enriched usually helps grounding.
2. **Assemble.** Concatenate them with source labels into one context block.
3. **Prompt.** "Answer using ONLY the context below; cite sources; if absent, say so." + context + question.
4. **Generate.** The LLM writes a grounded answer.
5. **Post-process.** Attach citations, optionally run a faithfulness check, return.

### 1.6 A full numerical walk-through

**Corpus:** three annual reports (ACME, TechNova, Globex), ~4,000 chunks total.
**Question:** *"Why did TechNova's hardware margins improve in FY2024?"*

| Stage | What happens to our sentence |
|---|---|
| Raw chunk | "Margins improved due to the new supplier agreement signed in March." — ranks ~800th. Buried. |
| + Context note | "From TechNova FY2024… hardware division…" prepended. Entity + year + division now embedded. |
| Vector search | Enriched chunk now ranks ~6th — close meaning to 'TechNova hardware margins 2024'. |
| BM25 search | Exact tokens 'TechNova', 'FY2024', 'supplier' ranks it ~3rd. |
| RRF fusion | Strong in both → rises to rank 2 overall. |
| Rerank | Cross-encoder confirms direct relevance → rank 1. Sent to LLM. |
| Generation | "TechNova's FY2024 hardware margins rose chiefly because of a new supplier agreement signed in March, which lowered input costs." |

*The single act of prepending an address moved the chunk from rank ~800 to rank 1. That is the whole payoff.*

### 1.7 When to use it

- **Chunks are ambiguous alone** — lots of pronouns, "the company," "last quarter," dangling references.
- **Entity- and number-heavy corpora** — financial filings, legal contracts, medical records, support tickets, codebases.
- **Exact-match matters** — error codes, SKUs, statute numbers — where BM25 earns its keep.
- **Documents are well-bounded** — each fits in an LLM context so the note-writer can see the whole thing.

### Pros and cons

| Pros | Cons |
|---|---|
| Directly kills the orphaned-chunk problem — large measured drops in retrieval failure. | Adds an LLM call per chunk at ingestion (cost + time). |
| Hybrid vector + BM25 catches both meaning and exact tokens. | Indexing pipeline is more complex (two indexes + fusion + reranker). |
| Retrieval stays a single round-trip — low query latency. | Prompt-caching mitigates but doesn't erase ingestion cost; big corpora still add up. |
| Works with any existing flat vector store; bolt-on. | If a document is larger than the context window, the 'whole-document' note degrades. |
| No change to query-time logic for the user. | Re-ingesting on document change re-runs the LLM notes. |

---

## 2. Hierarchical RAG

### 2.1 The problem, stated sharply

Now forget amnesia; the chunks are fine. The trouble is **flatness**. Imagine 4,000 chunks in one pile and the question *"Summarise the water-damage rules across all our policies."* No single chunk answers it. The word "deductible" appears 200 times across unrelated policies. A flat top-K search returns a confetti of fragments with no sense of which document or section they belong to, and no overview anywhere.

What's missing is **altitude**. Sometimes you want a satellite view (the whole policy), sometimes street level (one paragraph). A flat index only ever gives you street level.

### 2.2 The idea (in one breath)

> Build a **tree**. Leaves are the raw chunks. Group similar leaves and have an LLM write a **summary** of each group — those summaries are the next level up. Summarise the summaries, and again, until a single root describes the whole corpus. Now you can retrieve at *any altitude*, and you can fly top-down so most of the tree is never even searched.

The popular concrete recipe for this is **RAPTOR** (Recursive Abstractive Processing for Tree-Organised Retrieval). We'll use it as our example.

### 2.3 Picture the tree

```
Level 3 (Root)   :  whole-corpus gist            [1 node]
                         /            \
Level 2          :  per-document summaries        [~3]
                      /      |      \
Level 1          :  per-section summaries         [~400]
                   /   |   |   |   \
Level 0 (Leaves) :  raw chunks                    [~4000]
```

### 2.4 Step by step — INDEXING (build the tree, offline)

1. **Chunk & embed leaves.** Split into ~200–400-token leaf chunks (Level 0); embed each.
2. **Cluster.** Group semantically similar leaves. RAPTOR reduces dimensions (UMAP) then clusters with a Gaussian Mixture Model, allowing *soft* membership — a chunk can join more than one cluster.
3. **Summarise.** Feed each cluster's members to an LLM → a concise summary. Each summary is a Level-1 node; embed it.
4. **Recurse.** Treat the Level-1 summaries as new input; cluster and summarise them → Level 2; repeat until one node remains.
5. **Index every node.** Store all nodes at all levels with embedding, level, and parent/child links. Two storage choices below.

> **Two ways to store & search the tree**
>
> - **Tree traversal:** keep levels separate; the query walks down level by level, pruning branches.
> - **Collapsed tree:** flatten *every* node from every level into one index and search them all together; the search itself picks the right altitude. RAPTOR found this usually wins and is simpler.

### 2.5 Step by step — RETRIEVAL (per question)

**Embed the question**, then pick one mode:

**Mode A — Tree traversal (top-down)**

1. **Search the top level** (root / document summaries); keep the most relevant node(s).
2. **Descend** into only the children of those nodes; keep the best.
3. **Repeat to the leaves**, pruning irrelevant branches; collect nodes along the path. Most of the tree is never touched → fast, scoped.

**Mode B — Collapsed tree (recommended)**

1. **One search over all nodes** (leaves + every summary level) → top-K. A broad question naturally pulls summaries; a specific one pulls leaves.

**Then (optional):** rerank the retrieved nodes and keep top-N.

### 2.6 Step by step — AUGMENTATION & GENERATION

1. **Select.** Top-N nodes — typically a **mix** of leaf chunks and summaries. That mix is the entire point.
2. **Assemble & label.** Mark each as 'Summary' or 'Detail' so the model knows the altitude of each piece.
3. **Prompt.** "Summaries give broad context; details give specifics. Answer using only the context."
4. **Generate, then post-process** — trace each summary back to its child leaves for citations.

### 2.7 A full numerical walk-through

**Corpus:** a 300-page, 3-policy insurance manual, ~4,000 leaf chunks.
**Question:** *"What is the deductible for water-damage claims on the homeowner policy?"* (We'll trace tree-traversal mode.)

| Step | What the search does |
|---|---|
| Level 2 (docs) | Compare question to 3 document summaries. 'Homeowner Policy HX-200' wins; Auto and Life are pruned — never searched again. |
| Level 1 (sections) | Among HX-200's section summaries, 'Section 4: Property Damage Coverage' matches 'water damage'. |
| Level 0 (leaves) | Inside Section 4 only, retrieve the paragraph: 'Water-damage deductible: $1,500 per incident.' |
| Augment | Send that leaf PLUS the Section-4 summary (so the model has both the number and its surrounding rules). |
| Generate | "On the homeowner policy (HX-200), the water-damage deductible is $1,500 per incident, under Section 4 Property Damage Coverage." |

*Note what flatness would have done: 'deductible' hits 200 chunks across all three policies, and the auto-policy's collision-deductible could easily out-rank the right answer. The tree pruned two whole policies in one step.*

**And a question that wants altitude:** *"Give me an overview of what the homeowner policy covers."* Here the collapsed-tree search simply returns the Level-2 summary node directly — no need to stitch 50 leaf chunks together. A flat index literally cannot do this; it has no summary to return.

### 2.8 When to use it

- **Long, structured documents** — manuals, policies, books, research corpora with real hierarchy.
- **Big-picture or multi-hop questions** — "summarise chapter 4," "compare across sections," "what's the overall theme."
- **Mixed-altitude needs** — the same system must answer both "what's the exact deductible" and "summarise the coverage."
- **Large search space** — top-down traversal prunes most of the corpus, keeping precision high as the pile grows.

### Pros and cons

| Pros | Cons |
|---|---|
| Retrieves at multiple altitudes — can answer both detail and overview questions. | Building the tree is expensive: clustering + many LLM summarisation calls. |
| Top-down traversal prunes most of the corpus — precision holds at scale. | Tree can go stale; updating documents may need partial rebuilds. |
| Summaries enable genuine multi-hop / cross-section reasoning. | More moving parts: clustering params (UMAP/GMM) need tuning. |
| Collapsed-tree mode keeps query-time logic simple while still offering altitude. | Summaries can lose or distort detail — an LLM summary may drop the very fact you need. |
| Scales gracefully to very large, structured corpora. | Tree-traversal mode adds sequential lookups → higher latency than a flat search. |

---

## 3. Putting Them Side by Side

Strip away the vocabulary and the choice is about **which disease you have.** One fixes amnesia in the chunk; the other fixes flatness in the pile. They are not rivals.

| Question | Contextual RAG | Hierarchical RAG |
|---|---|---|
| Disease it cures | Amnesia (orphaned chunks) | Flatness (lost structure) |
| Indexing trick | LLM writes a context note per chunk | Recursively cluster + summarise into a tree |
| What gets indexed | Enriched chunks (one level) | Nodes at every altitude |
| Retrieval | Vector + BM25 → RRF → rerank | Tree traversal, or one search over collapsed tree |
| Reaches the LLM | Leaf chunks (now self-describing) | A mix of summaries and leaf chunks |
| Best questions | Specific, entity/number-heavy | Broad, multi-hop, overview, summarise |
| Main cost | One LLM call per chunk at ingest | Clustering + many summary calls; rebuilds |
| Query latency | Single round-trip (low) | Low (collapsed) to higher (traversal) |

### 3.1 You can have both

The methods compose cleanly. A strong production system often builds a RAPTOR-style tree **and** contextualises the leaf chunks — so each leaf is self-describing (cures amnesia) *and* sits in a tree you can fly over (cures flatness). Add hybrid search + reranking on top and you've covered both diseases at once.

> **The one-sentence memory hook**
>
> **Contextual** gives every chunk an **address** so it can be found. **Hierarchical** gives the corpus an **altimeter** so you can choose how high to fly. Different problems, different cures — and on a hard corpus, you want both.

---

*End of notes. If you can rebuild each method from its problem statement with a pencil, you understand it.*
