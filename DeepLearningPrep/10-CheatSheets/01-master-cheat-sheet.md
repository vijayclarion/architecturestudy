# Deep Learning Master Cheat Sheet

> **The ONE page that has everything.** Print this and stick it on your wall.

---

## 🏗️ Network Architecture

```
INPUT → [LINEAR → RELU] × (L-1) → LINEAR → SIGMOID → OUTPUT
```

| Layer Type | Activation | When |
|------------|-----------|------|
| Input | None | Just data |
| Hidden | **ReLU** | Default |
| Output (binary) | Sigmoid | Yes/No |
| Output (multi-class) | Softmax | A/B/C |
| Output (regression) | Linear | Real number |

---

## 📐 Shape Cheat Sheet

```
For layer L with m examples:
   W[L] = (n[L], n[L-1])      ← weights
   b[L] = (n[L], 1)           ← bias
   Z[L] = (n[L], m)           ← linear output
   A[L] = (n[L], m)           ← activation output
   X    = (n[0], m)           ← input
```

**Golden Rule:** rows = current layer, cols = previous layer

---

## ➡️ Forward Propagation

```
For each layer L = 1, 2, ..., last:
   Z[L] = W[L] · A[L-1] + b[L]
   A[L] = g(Z[L])
   
   Save cache: (A[L-1], W[L], b[L], Z[L])
```

---

## ⬅️ Backward Propagation

```
Start at output:
   dZ[L] = ŷ - y                    (for sigmoid+CE or linear+MSE)

For each layer (going backwards):
   dW[L] = (1/m) × dZ[L] · A[L-1].T
   db[L] = (1/m) × sum(dZ[L], axis=1, keepdims=True)
   dA[L-1] = W[L].T · dZ[L]
   dZ[L-1] = dA[L-1] × g'(Z[L-1])
```

### Activation Derivatives
```
ReLU'(z)    = 1 if z>0, else 0
Sigmoid'(z) = sigmoid(z) × (1 - sigmoid(z))
Tanh'(z)    = 1 - tanh(z)²
Linear'(z)  = 1
```

---

## 🔄 Update Rule

```
W = W - α × dW
b = b - α × db

α = learning rate (try: 0.01, 0.001, 0.0001)
```

---

## 📊 Loss Functions

| Problem | Loss | Output Activation |
|---------|------|-------------------|
| Binary classify | Binary CE: `-[y·log(ŷ) + (1-y)·log(1-ŷ)]` | Sigmoid |
| Multi-class | Categorical CE: `-Σy·log(ŷ)` | Softmax |
| Regression | MSE: `(ŷ - y)²` | Linear |

---

## 🛡️ Regularization

### L2 (default for deep learning)
```
J = original_cost + (λ/2m) × ||W||²

Effect: weight decay (weights shrink each update)
```

### L1 (creates sparsity)
```
J = original_cost + (λ/2m) × ||W||₁

Effect: many weights become exactly 0
```

### λ Tuning
```
λ = 0      → No regularization (overfit)
λ = 0.01   → Default mild
λ = 0.1    → Strong
λ = 100    → Too strong (underfit)
```

---

## 🎯 Bias vs Variance Diagnosis

| Errors | Diagnosis | Fix |
|--------|-----------|-----|
| Training HIGH, Dev close | **High Bias** | Bigger model, train longer |
| Training LOW, Dev MUCH higher | **High Variance** | More data, regularize, smaller model |
| Both HIGH | **Both** | Fix bias first |
| Both LOW | **Perfect** | Ship it! 🚀 |

```
BIAS     = Training error - Human-level error
VARIANCE = Dev error - Training error
```

---

## 📦 Data Splitting

```
Small data (<10K):  60/20/20 (train/dev/test)
Medium (10K-1M):    80/10/10 or 90/5/5
Big data (>1M):     98/1/1
```

**Rules:**
- Train → update weights
- Dev → tune hyperparameters
- Test → final eval (touch ONCE)
- All from SAME distribution!

---

## 🏛️ The 10 Core Functions

```
1. initialize_parameters / initialize_parameters_deep
2. linear_forward          Z = W·A + b
3. linear_activation_forward   + apply activation
4. L_model_forward         loop all layers forward
5. compute_cost            measure error
6. linear_backward         compute dW, db, dA_prev
7. linear_activation_backward  + activation derivative
8. L_model_backward        loop all layers backward
9. update_parameters       W -= α·dW
10. predict                use trained model
```

---

## 🎛️ Hyperparameters Quick Reference

| Hyperparameter | Default | Tune Range |
|----------------|---------|------------|
| Learning rate (α) | 0.01 | 0.0001 to 0.1 |
| Hidden layers | 2-4 | 1 to many |
| Neurons per layer | 64-128 | 16 to 1024 |
| Batch size | 64 | 32, 64, 128, 256 |
| λ (regularization) | 0.01 | 0 to 1 |
| Epochs | 100-1000 | Watch dev error |
| Optimizer | Adam | SGD, Adam, RMSProp |

---

## 🐛 Common Bugs

| Bug | Fix |
|-----|-----|
| Shape mismatch | Check W·X dims: (n[L], n[L-1]) · (n[L-1], m) |
| Forgot `keepdims=True` | `db = np.sum(dZ, axis=1, keepdims=True)` |
| Loss = NaN | Lower learning rate, normalize data |
| Loss not decreasing | Check learning rate, gradient flow |
| Train good, test bad | Add regularization, more data |
| Train can't learn | Bigger model, lower learning rate, more epochs |

---

## ⚡ Vectorization Reminder

```python
# WRONG (slow loop)
for i in range(m):
    z[i] = W @ X[:, i:i+1] + b

# RIGHT (vectorized)
Z = W @ X + b   # 10-1000x faster!
```

---

## 🎓 Key Insights

1. **Without non-linear activations**, deep networks collapse to one layer
2. **Caches** during forward pass enable efficient backward pass
3. **Backprop** is just the chain rule applied systematically
4. **Initialize weights ×0.01** to prevent saturation
5. **Match output activation to loss**: Sigmoid+BCE, Softmax+CCE, Linear+MSE
6. **More data > more parameters** for fixing variance
7. **Bigger model > more epochs** for fixing bias

---

## 📚 Real World Anchors

| Concept | Real System |
|---------|-------------|
| Cross-Entropy + Softmax | ChatGPT, image classifiers |
| ReLU | All modern CNNs |
| L2 Regularization | Default in PyTorch (`weight_decay`) |
| Adam Optimizer | BERT, GPT, almost all modern training |
| Train/Dev/Test split | ImageNet, Kaggle competitions |

---

## 🏆 Decision Trees Combined

```
┌─ Which Activation? ──────────────────┐
│  Hidden → ReLU                       │
│  Binary output → Sigmoid             │
│  Multi-class output → Softmax        │
│  Regression output → Linear          │
└──────────────────────────────────────┘

┌─ Which Loss? ────────────────────────┐
│  Binary → BCE + Sigmoid              │
│  Multi-class → CCE + Softmax         │
│  Regression → MSE + Linear           │
└──────────────────────────────────────┘

┌─ Which Optimizer? ───────────────────┐
│  Learning → Plain GD                 │
│  Default → Adam                      │
│  Vision contests → SGD + Momentum    │
└──────────────────────────────────────┘

┌─ Diagnose Problem ───────────────────┐
│  High train error → bias → bigger    │
│  Big train-dev gap → variance →      │
│       regularize + more data         │
└──────────────────────────────────────┘
```
