# Formula & Key Numbers Reference

> Every formula and number worth memorising — in one place.

---

## Core Formulas

### Attention

```
Attention(Q, K, V) = softmax( QKᵀ / √d_k ) · V

Where:
  Q = query matrix   (seq_len × d_k)
  K = key matrix     (seq_len × d_k)
  V = value matrix   (seq_len × d_v)
  d_k = key/query dimension per head

Step breakdown:
  1. QKᵀ          = (seq_len × seq_len)  raw scores
  2. / √d_k       = scale to prevent saturation
  3. softmax(.)   = attention weights, each row sums to 1
  4. × V          = weighted mix of values
```

### Multi-Head Attention

```
MultiHead(Q,K,V) = Concat(head₁, ..., headₕ) · Wₒ

headᵢ = Attention(Q·Wᵢq, K·Wᵢk, V·Wᵢv)

Dimensions:
  d_head = d_model / H
  Each Wᵢq, Wᵢk, Wᵢv: (d_model × d_head)
  Wₒ: (H × d_head × d_model) = (d_model × d_model)
```

### Feed-Forward Network

```
FFN(x) = max(0, x·W₁ + b₁) · W₂ + b₂

  W₁: (d_model × d_ffn)     d_ffn = 4 × d_model (typically)
  W₂: (d_ffn × d_model)

Modern variant with GELU:
  FFN(x) = GELU(x·W₁ + b₁) · W₂ + b₂
```

### Layer Normalisation

```
LayerNorm(x) = (x - μ) / σ · γ + β

  μ = mean across d_model dimensions
  σ = std across d_model dimensions
  γ, β = learned scale and shift (size d_model each)
```

### Residual Connection

```
output = x + SubLayer(LayerNorm(x))

Full transformer block:
  x = x + Attention(LayerNorm(x))
  x = x + FFN(LayerNorm(x))
```

### Sampling / Temperature

```
p(tokenᵢ) = exp(logitᵢ / T) / Σⱼ exp(logitⱼ / T)

T → 0:  argmax (greedy)
T = 1:  unchanged softmax
T → ∞:  uniform distribution
```

### Sinusoidal Positional Encoding

```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))

pos = position in sequence (0-indexed)
i   = dimension index
```

---

## Key Architecture Numbers

### Model Sizes

```
┌──────────────────────────────────────────────────────────┐
│  MODEL      LAYERS  D_MODEL  D_FFN    HEADS  PARAMS      │
├──────────────────────────────────────────────────────────┤
│  GPT-2 sm   12       768     3,072    12     117M        │
│  GPT-2 lg   36      1,280    5,120    20     774M        │
│  GPT-2 XL   48      1,600    6,400    25     1.5B        │
│  GPT-3      96     12,288   49,152    96     175B        │
│  LLaMA-7B   32      4,096   11,008    32     7B          │
│  LLaMA-13B  40      5,120   13,824    40     13B         │
│  LLaMA-70B  80      8,192   28,672    64     70B         │
│  Mistral-7B 32      4,096   14,336    32     7B          │
└──────────────────────────────────────────────────────────┘
```

### Vocabulary Sizes

```
GPT-2/3/4 (cl100k):   50,257 to ~100,000 tokens
LLaMA 1/2:            32,000 tokens
LLaMA 3:              128,256 tokens
Mistral:              32,000 tokens
```

### Context Windows

```
GPT-2:               1,024 tokens
GPT-3:               2,048 tokens
GPT-3.5-turbo:       4,096 → 16,384 tokens
GPT-4:               8,192 → 128,000 tokens
Claude 2:            100,000 tokens
Claude 3:            200,000 tokens
LLaMA-2:             4,096 tokens (fine-tuned: 32K+)
Gemini 1.5 Pro:      1,000,000 tokens
```

---

## Complexity Analysis

```
Operation                 Complexity
─────────────────────────────────────────────
Attention computation     O(n² · d)
Attention memory          O(n² + n·d)
FFN computation           O(n · d²)
Embedding lookup          O(1) per token
Full forward pass         O(n² + n·d²)

Where:
  n = sequence length
  d = model dimension

Consequence:
  2× longer context = 4× more compute for attention
  This is why long contexts are expensive!
```

---

## Parameter Count Estimation

```
One transformer block:
  Attention:
    Wq, Wk, Wv each: d_model × d_model
    Wo:              d_model × d_model
    Total: 4 × d_model²

  FFN:
    W1: d_model × d_ffn  = d_model × 4d_model = 4 × d_model²
    W2: d_ffn × d_model  = 4 × d_model²

  Layer Norms: 4 × d_model (negligible)

  One block ≈ 8 × d_model² params

Total model params:
  Embedding: vocab × d_model
  N blocks:  N × 8 × d_model²
  Output:    d_model × vocab (often tied with embedding)

GPT-3 check:
  96 × 8 × 12288² ≈ 96 × 8 × 150M ≈ 115B
  + embeddings + output ≈ 175B total ✓
```

---

## KV Cache Memory

```
KV Cache size = 2 (K and V) × N_layers × seq_len × d_model × bytes

GPT-3 at 4K context:
  2 × 96 × 4096 × 12288 × 2 bytes = ~9.7 GB

GPT-3 at 32K context:
  2 × 96 × 32768 × 12288 × 2 bytes = ~77 GB

→ Long contexts need enormous memory even for inference
→ Why 200K context Claude needs serious hardware
```

---

## Tokenisation Ratios

```
English prose:        ~0.75 tokens/word
Code:                 ~1.5-2 tokens/word (more symbols)
Chinese/Japanese:     ~1-2 chars/token (more efficient)
Random characters:    ~1 token/char (worst case)

1 page (~500 words) ≈ 375 tokens
1 novel (~90K words) ≈ 67,500 tokens
GPT-4 128K context  ≈ ~170K English words ≈ 340 pages
```

---

## API Cost Quick Reference (approximate 2024)

```
Model              Input $/1K tokens   Output $/1K tokens
──────────────────────────────────────────────────────
GPT-4o             $0.005              $0.015
GPT-4o-mini        $0.00015            $0.0006
GPT-3.5-turbo      $0.0005             $0.0015
Claude 3 Sonnet    $0.003              $0.015
Claude 3 Haiku     $0.00025            $0.00125

Rule of thumb:
  1,000-word response ≈ 750 tokens
  At GPT-4o pricing: ~$0.01 per response
  At mini pricing:   ~$0.001 per response (10× cheaper)
```
