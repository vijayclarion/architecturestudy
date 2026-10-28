# Two RAG Tricks, Explained the Feynman Way

> *"If you can't explain it simply, you don't understand it well enough. And if it sounds complicated, somebody is hiding something from you — usually themselves."*

Let me tell you about two clever tricks people use to make a computer find the right page in a giant library. But first, let's be honest about the problem, because if you don't feel the problem in your bones, the tricks will just sound like magic words.

---

## First, the actual problem (no jargon allowed)

Imagine a library where, instead of organizing books by topic, you organize them by *vibe*. Every book gets turned into a single dot floating in space. Books with similar vibes float near each other. Cooking books cluster over here, physics books cluster over there. This "vibe-space" is what nerds call **embedding space**, and the dots are **embeddings**. Don't let the word scare you — it's just "turn the meaning of some text into a location."

Now you walk in and ask a question. The computer turns *your question* into a dot too, and grabs whatever books are floating nearby.

Here's the trouble. **A question and an answer have different vibes.**

Watch:

- Your question: *"Why is the sky blue?"* — tiny, curious, pokes at the air.
- The real answer in the book: *"Sunlight scatters off air molecules, and shorter blue wavelengths scatter more strongly than red, a phenomenon called Rayleigh scattering..."* — long, confident, stuffed with vocabulary.

Same topic. Totally different *shape*. So your little question-dot lands slightly off in the wrong neighborhood, and the computer hands you so-so books instead of the perfect one.

That's the whole disease. Both tricks below are different medicines for it.

---

## Trick #1: Multi-Query RAG — *"Ask the same thing five ways"*

### The idea, plainly

You know how when you ask one person for directions and get lost, so you ask three more people and average it out? That's this. Instead of trusting your *one* phrasing of a question, you have the computer rewrite your question several different ways, search with each one, and then pool everything together.

Why does this help? Because **your single phrasing is a gamble.** You happened to use certain words. The book that answers you best might use slightly different words. By rephrasing, you throw a wider net and catch books your original wording would have swum right past.

### The steps, like a recipe

1. Take the user's question.
2. Ask the language model: *"Give me 4 other ways to ask this."*
3. Search the library once for each version. Now you have several little piles of books.
4. Mash the piles together, and let books that showed up in *many* piles rise to the top.
5. Hand that combined pile to the answer-writer.

### A worked example

**You ask:** *"How do I stop my machine learning model from overfitting?"*

The model rewrites it into:

- *"Techniques to prevent a neural network from memorizing its training data"*
- *"What is regularization in machine learning?"*
- *"My model does great on training data but terrible on new data — why?"*
- *"Explain dropout, weight decay, and early stopping"*

Look what happened. The *third* one phrases the symptom, not the cure — it'll find a troubleshooting article. The *fourth* one names specific tools — it'll find a technical doc your original question never would have touched. Each rewrite fishes in a slightly different spot. Pool them and you get a fuller, richer answer than any single phrasing could.

### The pooling step (the one bit of math)

When you mash the piles together, you reward books that ranked high across *several* of the rewrites. The popular recipe is **Reciprocal Rank Fusion**:

```
score(book) = sum over all queries of:  1 / (k + rank of book in that query)
```

`k` is just a steadying number, usually 60. A book sitting near the top of three different piles beats a book sitting at the top of only one. Democracy for documents.

```python
def reciprocal_rank_fusion(piles, k=60):
    scores = {}
    for ranked_list in piles:                # one pile per rephrased query
        for rank, doc_id in enumerate(ranked_list):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)
```

### Pros and cons

**Pros**
- Catches answers that hide behind different vocabulary.
- Covers a question that secretly has several parts.
- Simple to reason about — it's just "search more, then vote."

**Cons**
- You pay for several searches instead of one.
- One extra call to the language model to do the rewriting.
- If the rewrites all come out nearly identical, you spent money to learn nothing new.

### When to reach for it

Use Multi-Query when the question is **broad or many-sided**, or when you suspect the right document says the same thing in different words. It's your tool when **missing a relevant document is the thing you fear most.**

---

## Trick #2: HyDE — *"Make up a fake answer and chase its shadow"*

This one is delightfully sneaky, so pay attention.

### The idea, plainly

Remember the disease: questions and answers have different shapes, so searching *with a question* lands you in the wrong neighborhood. The obvious fix would be "search with the answer instead" — but you don't *have* the answer. That's literally why you're searching.

So HyDE does something that sounds insane the first time you hear it:

> **It tells the computer to make up a fake answer — and then searches using the fake answer.**

The fake answer is allowed to be *wrong*. You never show it to anyone. It exists for one reason only: a fake answer is *answer-shaped*, so its dot lands in the answer-neighborhood, right next to the *real* answers you're hunting for.

The full name is **Hypothetical Document Embeddings** — "hypothetical" because the document is made up, and "embeddings" because, as always, that just means "the dot we get from turning text into a location."

### Why a *wrong* answer still works — the crucial bit

Here's the thing most people miss, and it's beautiful. When you turn text into a dot, the dot mostly cares about **what words and topics appear**, not whether every fact is correct.

A fake answer about the blue sky will still say "sunlight," "scatter," "wavelength," "molecules" — even if it botches a number or invents a fake scientist. Those topic-words are exactly the words your *real* books use. So the fake answer's dot snuggles right up against the real documents, wrong facts and all. The errors don't matter because **you're not using the fake answer for facts — you're using it as bait.**

### The steps, like a recipe

1. Take the question.
2. Ask the model: *"Write a paragraph that answers this."* (Accuracy optional.)
3. Turn *that paragraph* into a dot — not the question.
4. Search the library near that dot. Pull out the **real** books.
5. Write the final answer using only those real books. Throw the fake one in the trash.

### A worked example

**You ask:** *"Why do leaves change color in autumn?"*

The model invents this paragraph:

> *"Leaves change color in autumn because chlorophyll breaks down as daylight shortens. This unmasks carotenoid pigments that give yellow and orange. Anthocyanins, made in cool weather, produce reds and purples. Temperature, light, and water all shape how vivid the display becomes."*

Maybe one claim is a little off. **Who cares.** That paragraph is now bursting with "chlorophyll," "carotenoid," "anthocyanin," "pigment" — the exact vocabulary sitting inside your real botany articles. Search with *its* dot and you land smack in the middle of the good books. Then you write the true answer from those true books, and the fake paragraph quietly disappears.

```python
def hyde_retrieve(query, llm, embedder, library, k=5):
    fake_answer = llm.generate(f"Write a detailed paragraph answering: {query}")
    probe = embedder.embed(fake_answer)        # embed the FAKE answer
    return library.similarity_search(probe, k=k)   # but retrieve REAL books
```

### Pros and cons

**Pros**
- Directly closes the question-vs-answer shape gap — searches *answer-to-answer*.
- Wonderful for short, bare questions against a library of essay-like prose.
- Only one extra search, unlike Multi-Query's several.

**Cons**
- If the model knows nothing about your weird, specialized field, it writes a *bad* fake answer and drags your search off into the weeds.
- Still costs one language-model call per question.
- Feels uncomfortable trusting a deliberate hallucination, even though it works.

### When to reach for it

Use HyDE when your questions are **short and blunt** but your library is full of **rich, answer-shaped prose**, and when the topic is **general enough** that the model can fake a plausible answer. It shines in open, everyday knowledge; it stumbles in narrow expert domains it has never seen.

---

## The two tricks, side by side

| | Multi-Query | HyDE |
|---|---|---|
| What the model writes | Several **versions of the question** | One **fake answer** |
| What you search with | Each question version | The fake answer |
| Direction of search | Question → books (many times) | Fake-answer → books (once) |
| The big win | Wider net, fewer misses | Right neighborhood, better aim |
| Extra cost | 1 model call + several searches | 1 model call + 1 search |
| Where it trips | Rewrites come out too similar | Model doesn't know the field |
| The metaphor | Ask five people for directions | Carve a decoy fish to lure the real ones |

---

## The Feynman takeaway

Both tricks admit the same humble truth: **the way a human asks is a lousy map to where the answer lives.** Multi-Query fixes it by asking in many voices until one of them rhymes with the answer. HyDE fixes it by faking the answer's shape and following its shadow home.

And here's the fun part — you can do both at once. Generate several rephrased questions, run HyDE on each, pool the results. Most thorough, most expensive. As always in engineering: you pay for what you catch.

> *"The first principle is that you must not fool yourself — and you are the easiest person to fool."* — That's exactly why HyDE works: it fools the *search*, on purpose, while being honest with *you*.
