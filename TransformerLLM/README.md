# How Transformer LLMs Work — Complete Study Notes

> Based on the Format.md 5-Layer Learning System.
> Covers: Tokenization → Embeddings → Attention → Transformer Layers → Sampling → Generation Loop.

---

## The One-Line Answer

> **An LLM is a next-token predictor. Everything else — attention, layers, embeddings — exists purely to make that single prediction as accurate as possible.**

---

## Learning Path (Recommended Order)

| # | Topic | File | Time |
|---|-------|------|------|
| 1 | Big Picture: Prompt → Answer | `01-Overview/01-big-picture.md` | 10 min |
| 2 | Tokenization | `02-Tokenization/01-tokenization.md` | 15 min |
| 3 | Embeddings | `03-Embeddings/01-embeddings.md` | 15 min |
| 4 | Self-Attention (the core) | `04-Attention/01-self-attention.md` | 25 min |
| 5 | Multi-Head Attention | `04-Attention/02-multi-head-attention.md` | 15 min |
| 6 | Transformer Layer Anatomy | `05-TransformerLayers/01-layer-anatomy.md` | 20 min |
| 7 | Positional Encoding | `05-TransformerLayers/02-positional-encoding.md` | 10 min |
| 8 | Sampling & Temperature | `06-Sampling/01-sampling.md` | 15 min |
| 9 | The Generation Loop | `07-GenerationLoop/01-generation-loop.md` | 15 min |

---

## Quick Reference — All in One Place

- **Master Cheat Sheet**: `08-CheatSheets/01-master-cheat-sheet.md`
- **All Decision Trees**: `08-CheatSheets/02-decision-trees.md`
- **Formula Reference**: `08-CheatSheets/03-formula-reference.md`

---

## Folder Structure

```
TransformerLLM/
├── README.md                          ← You are here
├── 01-Overview/                       ← Start here. The 7-step pipeline.
├── 02-Tokenization/                   ← Text → token IDs
├── 03-Embeddings/                     ← Token IDs → vectors
├── 04-Attention/                      ← The heart of transformers
├── 05-TransformerLayers/              ← Full layer anatomy
├── 06-Sampling/                       ← How the next token is chosen
├── 07-GenerationLoop/                 ← How full responses are built
└── 08-CheatSheets/                    ← Print these and pin them up
```

---

## Real World Anchors

| Concept | Real System | Why |
|---------|------------|-----|
| Transformer architecture | GPT-4, Claude, Gemini | All modern LLMs use it |
| BPE Tokenizer | GPT-2/3/4 (`tiktoken`) | ~50K token vocabulary |
| Rotary Positional Encoding | LLaMA, Mistral | Better length generalisation |
| Multi-head attention | All LLMs | Parallel pattern detection |
| Temperature sampling | ChatGPT API | `temperature=0` for coding, `1.0` for creative |
| KV Cache | Every production LLM | Makes generation fast |

---

## Daily Review Routine

```
Monday-Friday (1 hour/day):
  20 min → Read one topic file
  20 min → Draw the diagram from memory (no notes!)
  20 min → Explain the concept out loud (Feynman method)

Weekend:
  → Draw the full 7-step pipeline from memory
  → Answer: "What does attention actually do?" without looking
```

---

## Mastery Checklist

For each topic, check:
- [ ] Can explain it in one sentence
- [ ] Can draw the diagram from memory
- [ ] Can state the key formula
- [ ] Know one real-world system that uses it
- [ ] Know the trade-off or cost
