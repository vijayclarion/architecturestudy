# Transformer LLM — Master Cheat Sheet

> **Print this. Pin it up. If you can explain every row from memory, you understand LLMs.**

---

## The 7-Step Pipeline (Memorise This)

```
┌──────────────────────────────────────────────────────────┐
│  1. TOKENIZE    Text → integer IDs via BPE               │
│  2. EMBED       IDs → learned vectors (d_model dims)     │
│  3. POSITION    Add position signal to each vector       │
│  4. ATTEND      N × (LayerNorm + MHA + Residual +        │
│                      LayerNorm + FFN + Residual)         │
│  5. PROJECT     Final vector → vocab-size logits         │
│  6. SAMPLE      Temperature + Top-p → next token ID      │
│  7. LOOP        Append token, repeat until EOS           │
└──────────────────────────────────────────────────────────┘
```

---

## Core Formula Reference

```
Attention(Q,K,V) = softmax( QKᵀ / √d_k ) · V

FFN(x) = max(0, xW₁ + b₁) W₂ + b₂

p(token) = softmax(logits / T)

transformer_block(x) = x + FFN(LN(x + Attn(LN(x))))
                                   ↑                ↑
                            residuals everywhere
```

---

## Key Components

| Component | What it does | Key property |
|-----------|-------------|-------------|
| Tokenizer | Text → IDs | BPE, ~50-100K vocab |
| Embedding | IDs → vectors | Similar tokens → nearby vectors |
| Positional enc. | Inject order | RoPE (modern), sinusoidal (original) |
| Layer Norm | Stabilise activations | mean=0, std=1 per token |
| Self-Attention | Token-to-token mixing | Q·Kᵀ / √d scaled, softmax |
| Multi-Head Attn | Parallel patterns | H heads, each d_model/H dim |
| Residual | Gradient highway | output = x + sublayer(x) |
| FFN | Per-token transform | 4× wider, where facts live |
| Output head | Project to vocab | Linear, then softmax |
| Sampling | Pick next token | Temperature, top-p, greedy |
| KV Cache | Avoid recomputation | O(n) not O(n²) per step |

---

## The One-Page Architecture

```
Prompt text
    ↓
[ TOKENIZER ]  →  "France" maps to ID 9822
    ↓
[ EMBEDDING ]  →  ID 9822 → 4096-dim vector (learned)
    ↓
[ POSITION ]   →  + positional signal (RoPE/sinusoidal)
    ↓
┌──────────────────────────────────────┐
│  TRANSFORMER BLOCK × N               │
│                                      │
│  LayerNorm → Multi-Head Attention    │
│            → + x (residual)          │
│  LayerNorm → FFN                     │
│            → + x (residual)          │
└──────────────────────────────────────┘
    ↓
[ OUTPUT HEAD ]  →  4096-dim → 100K-dim logits
    ↓
[ SAMPLING ]     →  softmax(logits/T) → pick token
    ↓
[ LOOP ]         →  append token → repeat
    ↓
Response: "The capital of France is Paris."
```

---

## Numbers to Know

```
┌─────────────────────────────────────────────────────┐
│  MODEL          LAYERS  D_MODEL  HEADS  PARAMS      │
├─────────────────────────────────────────────────────┤
│  GPT-2 small    12       768      12    117M        │
│  GPT-2 XL       48      1600      25    1.5B        │
│  GPT-3          96     12288      96    175B        │
│  LLaMA-7B       32      4096      32    7B          │
│  LLaMA-70B      80      8192      64    70B         │
│  GPT-4 (est.)  ~120   ~12288    ~96   ~1.8T        │
├─────────────────────────────────────────────────────┤
│  English words ≈ 0.75 tokens                        │
│  Cost: ~$0.01-0.06 per 1K tokens (GPT-4 range)     │
│  Attention: O(n²) in sequence length                │
└─────────────────────────────────────────────────────┘
```

---

## Attention vs FFN — Quick Contrast

```
ATTENTION                       FFN
────────────────────────────────────────────────────
Mixes info ACROSS tokens       Transforms each token INDEPENDENTLY
Detects relationships          Applies learned transformations
"France" links to "capital"    "This vector means country → add capital"
O(n²) in seq length            O(n) in seq length
Where context lives            Where facts are stored
```

---

## Sampling Quick Reference

```
T = 0.0     Greedy: always top token. Use: code, facts
T = 0.7     Balanced (ChatGPT default). Use: general chat
T = 1.0     Neutral: distribution unchanged
T = 1.5+    Creative/noisy. Use: brainstorming, stories
Top-p=0.9   Keep tokens summing to 90% probability
Top-k=50    Keep top 50 tokens only
```

---

## Decision Tree: Debugging LLM Behaviour

```
Model repeats itself?
  → Repetition penalty | Lower temperature | Better prompt

Model hallucinating facts?
  → Lower temperature | Add RAG | Better system prompt

Model ignores instructions?
  → Fine-tuning (RLHF) needed | Better prompt engineering

Model runs out of context?
  → Chunking | Summarisation | Longer-context model

Model too slow?
  → KV cache | Quantisation | Smaller model | Speculative decoding

Response cuts off?
  → Increase max_tokens

Answer varies too much?
  → Lower temperature | Set seed for reproducibility
```

---

## Real World Anchors

| Concept | Real System | Why it matters |
|---------|------------|----------------|
| BPE Tokenizer | GPT-2/3/4 (tiktoken) | Vocabulary = 100K tokens |
| Rotary PE (RoPE) | LLaMA, Mistral | Long context extrapolation |
| Multi-head attention | All LLMs | Parallel pattern detection |
| KV Cache | All production LLMs | Makes inference feasible |
| Temperature=0 | GitHub Copilot | Deterministic code completion |
| Streaming tokens | ChatGPT UI | Each token sent immediately |
| RLHF on top | ChatGPT, Claude | Base model → helpful model |

---

## Interview Soundbite (The One Paragraph)

> "An LLM tokenizes input into integer IDs, looks up learned embedding vectors, adds positional encodings, then passes the sequence through N transformer blocks. Each block runs multi-head self-attention — where every token attends to every other via scaled dot-product QKV — followed by a per-token FFN, with residual connections and layer norm throughout. The final vector projects to vocabulary-size logits; softmax with temperature converts these to probabilities; sampling picks the next token. The token is appended and the forward pass repeats until EOS — this autoregressive loop is how the full response is generated."
