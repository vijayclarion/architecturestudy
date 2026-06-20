# Decision Trees — LLM Architecture & Behaviour

> Memorise these frameworks. They are what interviewers want to hear — your thinking process, not just facts.

---

## Tree 1: Why Is the Model Behaving This Way?

```
┌──────────────────────────────────────────────────────────┐
│  MODEL BEHAVIOUR DIAGNOSIS                               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Repeating itself in loops?                             │
│    → Temperature too low OR prompt not constraining     │
│    → Fix: repetition penalty | higher T | better prompt │
│                                                          │
│  Making up facts (hallucinating)?                       │
│    → Model doesn't know → fills with plausible text     │
│    → Fix: RAG (retrieval) | lower T | explicit "I don't │
│           know" instruction in system prompt            │
│                                                          │
│  Ignoring instructions?                                 │
│    → Base model issue (not instruction-tuned) OR        │
│       prompt too weak                                   │
│    → Fix: use instruction-tuned model (ChatGPT vs GPT) │
│           | stronger system prompt                      │
│                                                          │
│  Response cuts off mid-sentence?                        │
│    → max_tokens reached                                 │
│    → Fix: increase max_tokens parameter                 │
│                                                          │
│  Inconsistent answers to same question?                 │
│    → Temperature > 0 causes variation                   │
│    → Fix: temperature=0 + seed for reproducibility     │
│                                                          │
│  Slow first response, then fast?                        │
│    → Cold start: model loading / KV cache warming       │
│    → Fix: keep-alive pings | warm pool                  │
└──────────────────────────────────────────────────────────┘
```

---

## Tree 2: Which Temperature Should I Use?

```
┌──────────────────────────────────────────────────────────┐
│  TEMPERATURE SELECTION                                   │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Task requires exact/correct output?                    │
│    YES → T = 0.0 (greedy)                               │
│           Use for: code generation, SQL, maths          │
│                                                          │
│  Task is conversational / general?                      │
│    → T = 0.7 (balanced default)                         │
│       Use for: chat, Q&A, summarisation                 │
│                                                          │
│  Task requires creativity / variety?                    │
│    → T = 0.9 - 1.2                                      │
│       Use for: story writing, brainstorming, poetry     │
│                                                          │
│  Task involves exploration / multiple options?          │
│    → T = 1.0, top-p = 0.95                             │
│       Run multiple times to get varied outputs          │
│                                                          │
│  T > 1.5 — almost never useful in production           │
│    Output quality degrades quickly                      │
└──────────────────────────────────────────────────────────┘
```

---

## Tree 3: Which Model Size for My Use Case?

```
┌──────────────────────────────────────────────────────────┐
│  MODEL SIZE SELECTION                                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Need best quality, cost not a concern?                  │
│    → GPT-4 / Claude 3 Opus / Gemini Ultra               │
│                                                          │
│  Balanced quality vs cost?                              │
│    → GPT-3.5-turbo / Claude 3 Sonnet                    │
│       Good for 80% of tasks at ~10× lower cost          │
│                                                          │
│  High volume, latency-sensitive?                        │
│    → GPT-4o-mini / Claude 3 Haiku                       │
│       Fast, cheap, good for simple tasks                │
│                                                          │
│  Data privacy / on-premise required?                    │
│    → LLaMA-3 70B / Mistral / Mixtral                    │
│       Open weights, run locally                         │
│                                                          │
│  Edge device / mobile?                                  │
│    → LLaMA-3 8B quantised / Phi-3-mini                  │
│       4-bit quantisation, fits in <8GB RAM              │
└──────────────────────────────────────────────────────────┘
```

---

## Tree 4: Why Is Inference Slow?

```
┌──────────────────────────────────────────────────────────┐
│  INFERENCE LATENCY DIAGNOSIS                            │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Is it TTFT (time to first token)?                      │
│    → Prefill phase: processing your entire prompt        │
│    → Fix: shorter prompts | caching | prefix caching    │
│                                                          │
│  Is it token generation speed?                          │
│    → Memory bandwidth bound on most hardware            │
│    → Fix: quantisation (4-bit) | better GPU             │
│                                                          │
│  Gets slower as response gets longer?                   │
│    → KV Cache grows → more memory reads per step        │
│    → Fix: limit max_tokens | use sliding window attn    │
│                                                          │
│  Very long context (>32K tokens)?                       │
│    → Attention is O(n²) in context length               │
│    → Fix: sparse attention models | Flash Attention     │
│                                                          │
│  Running on CPU?                                        │
│    → Always 10-100× slower than GPU                     │
│    → Fix: GPU inference / use an API                    │
└──────────────────────────────────────────────────────────┘
```

---

## Tree 5: Attention — When Does It Break Down?

```
┌──────────────────────────────────────────────────────────┐
│  ATTENTION FAILURE MODES                                 │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  "Lost in the middle" phenomenon?                       │
│    → Attention concentrates on start/end of context     │
│    → Key info buried in middle gets underweighted       │
│    → Fix: put critical info at start or end of prompt   │
│                                                          │
│  Context window exceeded?                               │
│    → Position encodings not trained beyond this         │
│    → Model sees garbage / truncates                     │
│    → Fix: use longer-context model | chunking | RAG     │
│                                                          │
│  Model ignores early prompt as context grows?           │
│    → Attention dilutes over longer sequences            │
│    → Fix: re-inject key instructions periodically       │
│                                                          │
│  Slow at very long context?                             │
│    → O(n²) attention: 100K context = 10B entries        │
│    → Fix: Flash Attention, sparse attention, RAG        │
└──────────────────────────────────────────────────────────┘
```

---

## Tree 6: What Layer Depth Does This Concept Live In?

```
┌──────────────────────────────────────────────────────────┐
│  LAYER DEPTH AND CONCEPT TYPE                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Is it about spelling, grammar, word form?              │
│    → Early layers (1-20%)                               │
│    → POS tags, morphology, local syntax                 │
│                                                          │
│  Is it about facts, entities, world knowledge?          │
│    → Middle layers (20-70%)                             │
│    → "France capital → Paris" lives in FFN weights      │
│                                                          │
│  Is it about instruction-following, task reasoning?     │
│    → Late layers (70-100%)                              │
│    → "User is asking X, appropriate response is Y"      │
│                                                          │
│  Implications:                                          │
│    Fine-tuning early layers → changes linguistic style  │
│    Fine-tuning late layers → changes task behaviour     │
│    LoRA on all layers → balanced general fine-tuning    │
└──────────────────────────────────────────────────────────┘
```

---

## Tree 7: RAG vs Fine-Tuning — Which to Use?

```
┌──────────────────────────────────────────────────────────┐
│  RAG vs FINE-TUNING                                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Data changes frequently?                               │
│    YES → RAG (retrieval updates without retraining)     │
│    NO  → continue                                       │
│                                                          │
│  Need to verify source of information?                  │
│    YES → RAG (can return citations)                     │
│    NO  → continue                                       │
│                                                          │
│  Need specific response FORMAT / persona?               │
│    YES → Fine-tuning (teaches style, not just facts)   │
│    NO  → continue                                       │
│                                                          │
│  Need to inject large amounts of domain knowledge?      │
│    YES → Fine-tuning (bakes knowledge into weights)     │
│           OR RAG (if data > context window limit)       │
│                                                          │
│  Quick experiment, limited budget?                      │
│    → RAG first (no GPU cost, immediate)                 │
│       Fine-tune later if performance insufficient       │
└──────────────────────────────────────────────────────────┘
```
