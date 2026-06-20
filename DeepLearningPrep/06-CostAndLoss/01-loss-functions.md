# Loss and Cost Functions

## Problem It Solves
We need a number that measures HOW WRONG our predictions are. Without this, we can't know if training is helping. The loss function turns prediction errors into a single number we can minimize.

## Loss vs Cost — The Difference

```
Loss  = error for ONE example          L(ŷ, y)
Cost  = average loss for ALL examples  J = (1/m) × Σ L
```

In practice, people use these terms interchangeably — but technically the formula above is correct.

## The 5 Main Loss Functions

| Function | Problem Type | Formula | Output Activation |
|----------|--------------|---------|-------------------|
| **MSE** | Regression | `(ŷ - y)²` | Linear |
| **MAE** | Regression | `\|ŷ - y\|` | Linear |
| **Binary Cross-Entropy** | Binary classification | `-[y·log(ŷ) + (1-y)·log(1-ŷ)]` | Sigmoid |
| **Categorical Cross-Entropy** | Multi-class | `-Σ yᵢ·log(ŷᵢ)` | Softmax |
| **Hinge Loss** | SVM-style classification | `max(0, 1 - y·ŷ)` | Linear |

## 1. Mean Squared Error (MSE) — For Regression

```
Loss = (ŷ - y)²
Cost = (1/m) × Σ (ŷ - y)²

Use when: predicting continuous numbers
   - House price prediction
   - Temperature forecast
   - Stock price

Pros:
   ✅ Easy to compute, smooth
   ✅ Strongly penalizes large errors

Cons:
   ❌ Sensitive to outliers (one bad point dominates)
```

**Example:**
```
True y = 50 lakhs
Predicted ŷ = 45 lakhs

Loss = (45 - 50)² = 25 lakhs²
```

## 2. Binary Cross-Entropy — For Binary Classification

```
Loss = -[y·log(ŷ) + (1-y)·log(1-ŷ)]
Cost = -(1/m) × Σ [y·log(ŷ) + (1-y)·log(1-ŷ)]

Use when: 2-class classification
   - Cat vs Not-Cat
   - Spam vs Not-Spam
   - Tumor: Malignant vs Benign

Pros:
   ✅ Smooth, well-behaved gradients
   ✅ Strongly penalizes confident wrong predictions

Cons:
   ❌ Requires Sigmoid output activation
```

**Visual: Why it works**
```
True y = 1 (positive class):
   If ŷ = 0.99 (correct, confident)  → Loss = -log(0.99) ≈ 0.01  ✅
   If ŷ = 0.50 (uncertain)           → Loss = -log(0.50) ≈ 0.69
   If ŷ = 0.01 (wrong, confident)    → Loss = -log(0.01) ≈ 4.6   ❌

True y = 0 (negative class):
   If ŷ = 0.01 (correct, confident)  → Loss = -log(0.99) ≈ 0.01  ✅
   If ŷ = 0.99 (wrong, confident)    → Loss = -log(0.01) ≈ 4.6   ❌
```

The function PUNISHES confident wrong predictions heavily!

## 3. Categorical Cross-Entropy — For Multi-Class

```
Loss = -Σ yᵢ × log(ŷᵢ)
Cost = -(1/m) × ΣΣ y × log(ŷ)

Use when: 3+ class classification
   - Cat/Dog/Bird
   - Digits 0-9 (MNIST)
   - 1000 ImageNet classes
```

**Example with 3 classes (Cat, Dog, Bird):**
```
True (one-hot): [1, 0, 0]   (it's a Cat)
Predicted:      [0.7, 0.2, 0.1]

Loss = -(1×log(0.7) + 0×log(0.2) + 0×log(0.1))
     = -log(0.7)
     ≈ 0.357
```

## 4. Mean Absolute Error (MAE) — Robust Regression

```
Loss = |ŷ - y|
Cost = (1/m) × Σ |ŷ - y|

Use when: outliers in regression data
Pros:
   ✅ Less sensitive to outliers than MSE
Cons:
   ❌ Not smooth at 0 (derivative undefined)
```

## Decision Tree: Which Loss to Use?

```
What are you predicting?
│
├─ Continuous number (price, temperature)
│   ├─ Outliers present? → MAE
│   └─ No outliers → MSE (default)
│
├─ Binary (yes/no, cat/not-cat)
│   → Binary Cross-Entropy + Sigmoid output
│
├─ Multi-class (cat/dog/bird)
│   → Categorical Cross-Entropy + Softmax output
│
└─ Margin-based (SVM-style)
    → Hinge Loss
```

## Loss + Activation Pairing (Important!)

```
┌─────────────────────────────────────────────────────────┐
│  Output Activation  →  Loss Function                    │
├─────────────────────────────────────────────────────────┤
│  Linear              →  MSE (regression)                │
│  Sigmoid             →  Binary Cross-Entropy            │
│  Softmax             →  Categorical Cross-Entropy       │
└─────────────────────────────────────────────────────────┘

Pairing them correctly gives clean gradients!
dZ = ŷ - y  in all 3 cases ← elegant!
```

## Why This Pairing? (Mathematical Beauty)

When Sigmoid + Binary Cross-Entropy are used together:
```
dL/dZ = ŷ - y    (super simple!)
```

If you used Sigmoid + MSE:
```
dL/dZ = (ŷ - y) · ŷ · (1 - ŷ)    (ugly + vanishing gradient!)
```

That's why we ALWAYS pair them correctly.

## Trade-offs

| Loss | Strengths | Weaknesses |
|------|-----------|-----------|
| MSE | Smooth, easy gradients | Sensitive to outliers |
| MAE | Robust to outliers | Non-smooth at 0 |
| Binary CE | Probabilistic, clean gradients | Only for binary |
| Categorical CE | Standard for multi-class | Requires one-hot encoding |
| Hinge | Margin-based | Not probabilistic |

## Real World Examples

| Application | Loss Used |
|-------------|-----------|
| ChatGPT (next token prediction) | Categorical Cross-Entropy |
| Image classification (ImageNet) | Categorical Cross-Entropy |
| Object detection (YOLO) | Combined: CE + MSE for boxes |
| Stable Diffusion | MSE on noise prediction |
| Cat vs Not-Cat | Binary Cross-Entropy |
| Stock price prediction | MSE |

## Code Snippets

```python
import numpy as np

# MSE
def mse_loss(y_pred, y_true):
    return np.mean((y_pred - y_true) ** 2)

# Binary Cross-Entropy
def binary_cross_entropy(y_pred, y_true):
    eps = 1e-15  # prevent log(0)
    y_pred = np.clip(y_pred, eps, 1 - eps)
    return -np.mean(y_true * np.log(y_pred) + (1-y_true) * np.log(1-y_pred))

# Categorical Cross-Entropy
def categorical_cross_entropy(y_pred, y_true):
    eps = 1e-15
    y_pred = np.clip(y_pred, eps, 1 - eps)
    return -np.mean(np.sum(y_true * np.log(y_pred), axis=1))
```

## Interview Soundbite
> "Choose your loss to match your output activation: Sigmoid pairs with Binary Cross-Entropy, Softmax with Categorical Cross-Entropy, Linear with MSE. This pairing gives mathematically clean gradients — specifically dZ = ŷ - y — which is one reason these combinations dominate."

## Common Bugs

❌ **Bug:** Using MSE with Sigmoid output → vanishing gradients
✅ **Fix:** Pair Sigmoid with Binary Cross-Entropy

❌ **Bug:** Forgetting epsilon clipping → `log(0) = -inf`
✅ **Fix:** `y_pred = np.clip(y_pred, 1e-15, 1 - 1e-15)`

❌ **Bug:** Using Binary CE for multi-class problems
✅ **Fix:** Use Categorical CE with Softmax
