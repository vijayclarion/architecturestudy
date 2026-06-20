# L1 and L2 Regularization

## Problem It Solves
Models with too many large weights MEMORIZE training data → high variance → poor generalization. Regularization adds a PENALTY for large weights, forcing the model to stay simple and generalize better.

## The Core Idea

```
ORIGINAL cost (no regularization):
   J = (1/m) × Σ L(ŷ, y)
       ↑
   just measures prediction error

REGULARIZED cost:
   J = (1/m) × Σ L(ŷ, y)  +  λ × (penalty for big weights)
       ↑                     ↑
   prediction error      "weight tax"

The model now has TWO things to minimize:
   1. Prediction error (be accurate)
   2. Weight size (stay simple)
   
Result: model accepts slightly worse accuracy
        in exchange for simpler weights → better generalization
```

## L2 Regularization (Most Common)

### Formula
```
J = (1/m) × Σ L(ŷ, y)  +  (λ/2m) × ||w||²₂

Where ||w||²₂ = w₁² + w₂² + ... + wₙ²
              = sum of all weights SQUARED
              = wᵀw
```

### Visual: What ||w||²₂ Means
```
w = [0.5, -0.3, 0.8, -0.2]

||w||²₂ = 0.5² + (-0.3)² + 0.8² + (-0.2)²
        = 0.25 + 0.09 + 0.64 + 0.04
        = 1.02

LARGER weights → LARGER penalty
SMALLER weights → SMALLER penalty
```

## L1 Regularization

### Formula
```
J = (1/m) × Σ L(ŷ, y)  +  (λ/2m) × ||w||₁

Where ||w||₁ = |w₁| + |w₂| + ... + |wₙ|
             = sum of ABSOLUTE values of weights
```

### Key Property
L1 produces SPARSE weights — many become EXACTLY zero. Useful for feature selection.

## L1 vs L2 — Visual Comparison

```mermaid
graph LR
    L2[L2: w² penalty] -->|effect| L2E[Small weights<br/>but rarely zero]
    L1[L1: |w| penalty] -->|effect| L1E[Many weights<br/>EXACTLY zero<br/>= sparse]
```

```
L2 weights example:
   w = [0.05, 0.01, 0.08, 0.02, 0.03, 0.03]
       ↑ all small, rarely zero

L1 weights example:
   w = [0.5, 0.0, 0.8, 0.0, 0.0, 0.3]
                ↑ many zeros!
```

## The Hyperparameter λ (Lambda)

```
λ controls regularization STRENGTH:

   λ = 0       → No regularization (original loss)
   λ = 0.001   → Very mild
   λ = 0.01    → Mild (typical)
   λ = 0.1     → Moderate
   λ = 1.0     → Strong
   λ = 100     → Very strong (weights forced to ~0)

You CHOOSE λ via dev set experimentation!
```

## Effect on Weight Updates

```
WITHOUT regularization:
   dW = (1/m) × dZ · A.T

WITH L2 regularization:
   dW = (1/m) × dZ · A.T  +  (λ/m) × W
                              ↑
                        extra penalty gradient

Weight update:
   W_new = W - α × dW
         = W × (1 - αλ/m)  -  α × normal_gradient
              ↑
        Less than 1 → W shrinks!
        Called "WEIGHT DECAY" ✅
```

## Why It Prevents Overfitting

```
Step-by-step intuition:

1. λ penalty discourages large weights
2. Model can't memorize specific examples (needs large weights)
3. Forced to find SIMPLER patterns
4. Simpler patterns GENERALIZE better
5. Dev/test error decreases ✅
```

## Decision Tree: Choose L1 or L2?

```
What's your goal?
│
├─ Just prevent overfitting?
│   → L2 (default choice, smoother gradients)
│
├─ Feature selection (want some weights = 0)?
│   → L1 (creates sparsity)
│
├─ Both? 
│   → Elastic Net (combination of L1 + L2)
│
└─ Working with neural networks?
    → L2 almost always (L1 is rare in deep learning)
```

## Decision Tree: How Much Regularization?

```
Run model with λ = 0:
│
├─ Severe overfitting (train 1%, dev 30%)?
│   → Try λ = 0.1
│
├─ Mild overfitting (train 5%, dev 15%)?
│   → Try λ = 0.01
│
├─ No overfitting (train 5%, dev 6%)?
│   → λ = 0 is fine, focus on other issues
│
└─ Always: try several λ values, pick best on DEV set
```

## Trade-offs

| Regularization | Pros | Cons |
|----------------|------|------|
| **L2** | Smooth, easy to optimize, default | Doesn't create sparsity |
| **L1** | Creates sparsity (feature selection) | Non-smooth at 0, harder optimization |
| **None** | Best training accuracy | Often overfits |

## Bias-Variance Trade-off

```
Higher λ:
   ✅ Less variance (less overfitting)
   ❌ More bias (model more constrained)

Lower λ:
   ✅ Less bias (model has freedom)
   ❌ More variance (may overfit)

Find balance: smallest dev error
```

## Why Skip the Bias Term?

```
Regularization formula focuses on W, ignores b

Why?
   W has thousands/millions of parameters
   b has only n[L] parameters (one per neuron)
   
   Regularizing b: negligible effect
   Standard convention: skip b

It's mathematically equivalent (almost) and simpler!
```

## Real World Examples

| System | Regularization Used |
|--------|---------------------|
| Cat classifier (Coursera) | L2 (called weight_decay in PyTorch) |
| ImageNet ResNets | L2 with λ = 0.0001 |
| BERT | L2 + Dropout |
| Logistic Regression | L2 (in scikit-learn `Ridge`) |
| Compressed Sensing | L1 (sparsity essential) |

## Code Example

```python
import numpy as np

# L2 Regularized cost
def compute_cost_with_L2(AL, Y, parameters, lambd):
    m = Y.shape[1]
    L = len(parameters) // 2
    
    # Original cross-entropy
    cross_entropy = -(1/m) * np.sum(Y*np.log(AL) + (1-Y)*np.log(1-AL))
    
    # L2 regularization term: sum of squared weights across ALL layers
    L2_sum = 0
    for l in range(1, L+1):
        W = parameters[f'W{l}']
        L2_sum += np.sum(W ** 2)   # = ||W||²
    
    L2_term = (lambd / (2*m)) * L2_sum
    return cross_entropy + L2_term


# L2 Regularized weight update
def update_with_L2(parameters, grads, learning_rate, lambd, m):
    L = len(parameters) // 2
    for l in range(1, L+1):
        # Modified gradient = original + λ/m × W
        dW = grads[f'dW{l}'] + (lambd/m) * parameters[f'W{l}']
        parameters[f'W{l}'] -= learning_rate * dW
        parameters[f'b{l}'] -= learning_rate * grads[f'db{l}']
    return parameters
```

## Interview Soundbite
> "L2 regularization adds (λ/2m)·||W||² to the cost, penalizing large weights. This forces the model to find simpler solutions that generalize better. The result is 'weight decay' — weights shrink slightly every update. L1 creates sparsity (many zeros), L2 just keeps weights small."

## Other Forms of Regularization (Bonus)

| Method | How It Works |
|--------|-------------|
| **Dropout** | Randomly turn off neurons during training |
| **Early Stopping** | Stop training before overfitting begins |
| **Data Augmentation** | Generate more training data (rotate, crop images) |
| **Batch Normalization** | Slight regularization side-effect |

These often work BETTER than L1/L2 for deep networks.

## Common Mistakes

❌ Using regularization when bias is the real problem
✅ Fix bias first (bigger model), then add regularization

❌ Using huge λ values → underfitting
✅ Start small (0.01) and increase only if needed

❌ Regularizing biases (b) — wastes effort
✅ Only regularize weights (W)
