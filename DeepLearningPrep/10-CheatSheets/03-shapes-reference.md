# Matrix Shapes — Quick Reference

> Print this. Pin it to your wall. 90% of neural network bugs are shape errors.

---

## 🎯 The Golden Rules

```
For network with layers: n[0] → n[1] → n[2] → ... → n[L]
And m training examples:

┌─────────────────────────────────────────────┐
│  X (input)        = (n[0], m)               │
│  Y (true labels)  = (n[L], m)               │
│  ŷ (predictions)  = (n[L], m)               │
│                                             │
│  W[L]             = (n[L], n[L-1])          │
│  b[L]             = (n[L], 1)               │
│  Z[L]             = (n[L], m)               │
│  A[L]             = (n[L], m)               │
│                                             │
│  dW[L]            = (n[L], n[L-1])          │
│  db[L]            = (n[L], 1)               │
│  dZ[L]            = (n[L], m)               │
│  dA[L]            = (n[L], m)               │
└─────────────────────────────────────────────┘
```

---

## 📋 Example: 4-Layer Network

**Architecture:** 12288 → 20 → 7 → 5 → 1 (cat classifier with m=209 examples)

```
n[0]=12288,  n[1]=20,  n[2]=7,  n[3]=5,  n[4]=1,  m=209

┌────────┬────────────────┬─────────────────┐
│ Var    │ Shape          │ What it means   │
├────────┼────────────────┼─────────────────┤
│ X      │ (12288, 209)   │ Input data      │
├────────┼────────────────┼─────────────────┤
│ W[1]   │ (20, 12288)    │ Layer 1 weights │
│ b[1]   │ (20, 1)        │ Layer 1 biases  │
│ Z[1]   │ (20, 209)      │ Layer 1 linear  │
│ A[1]   │ (20, 209)      │ Layer 1 ReLU    │
├────────┼────────────────┼─────────────────┤
│ W[2]   │ (7, 20)        │ Layer 2 weights │
│ b[2]   │ (7, 1)         │ Layer 2 biases  │
│ Z[2]   │ (7, 209)       │ Layer 2 linear  │
│ A[2]   │ (7, 209)       │ Layer 2 ReLU    │
├────────┼────────────────┼─────────────────┤
│ W[3]   │ (5, 7)         │ Layer 3 weights │
│ b[3]   │ (5, 1)         │ Layer 3 biases  │
│ Z[3]   │ (5, 209)       │ Layer 3 linear  │
│ A[3]   │ (5, 209)       │ Layer 3 ReLU    │
├────────┼────────────────┼─────────────────┤
│ W[4]   │ (1, 5)         │ Output weights  │
│ b[4]   │ (1, 1)         │ Output bias     │
│ Z[4]   │ (1, 209)       │ Output linear   │
│ AL=ŷ   │ (1, 209)       │ Sigmoid output  │
├────────┼────────────────┼─────────────────┤
│ Y      │ (1, 209)       │ True labels     │
└────────┴────────────────┴─────────────────┘
```

---

## ✅ Matrix Multiplication Cheat Sheet

```
Rule: (A, B) · (B, C) = (A, C)
      ↑    ↑    ↑    ↑    ↑
   rows  cols rows cols
        ↑        ↑
        These must MATCH

Forward pass:
   W[L]  ·  A[L-1]  =  Z[L]
   (n_L, n_{L-1}) · (n_{L-1}, m) = (n_L, m) ✅

Backward pass:
   dZ[L]  ·  A[L-1].T  =  dW[L]
   (n_L, m) · (m, n_{L-1}) = (n_L, n_{L-1}) ✅
   
   W[L].T  ·  dZ[L]  =  dA[L-1]
   (n_{L-1}, n_L) · (n_L, m) = (n_{L-1}, m) ✅
```

---

## 🔢 Element-Wise vs Matrix Multiplication

```
ELEMENT-WISE (*):              MATRIX MULTIPLY (@ or np.dot):
   Same shape required            Inner dims must match
   
   (3, 4) * (3, 4) = (3, 4)       (3, 4) @ (4, 5) = (3, 5)
   
   USE FOR:                       USE FOR:
   • Activation derivative        • W · A in forward pass
     dA × g'(Z)                   • Computing dW
   • Adding bias (broadcasting)   • Computing dA_prev
```

---

## 🎨 Broadcasting Rules

```
NumPy broadcasts when shapes are COMPATIBLE:

(n, 1) + (n, m) → broadcasts to (n, m) + (n, m) = (n, m) ✅
(1, m) + (n, m) → broadcasts to (n, m) + (n, m) = (n, m) ✅
(n, 1) + (1, m) → broadcasts to (n, m) + (n, m) = (n, m) ✅
(3, 4) + (5, 4) → INCOMPATIBLE → ERROR ❌

Common case:
   Z = W · X + b
   (n_L, m) + (n_L, 1) → broadcasts ✅
```

---

## 🚨 Common Shape Bugs

### Bug 1: Wrong X orientation
```
❌ X = (samples, features)  ← natural data view
✅ X = (features, samples)  ← neural network convention

Fix: X = X.T  (transpose)
```

### Bug 2: Forgot keepdims in db
```python
❌ db = (1/m) * np.sum(dZ, axis=1)              # shape (n,)  — 1D!
✅ db = (1/m) * np.sum(dZ, axis=1, keepdims=True)  # shape (n, 1) ✅
```

### Bug 3: Wrong W dimensions
```python
❌ W1 = np.random.randn(n_x, n_h)  # WRONG order
✅ W1 = np.random.randn(n_h, n_x)  # rows = current, cols = prev
```

### Bug 4: Element-wise where matrix needed
```python
❌ Z = W * X + b      # element-wise, will fail if shapes differ
✅ Z = W @ X + b      # matrix multiplication
✅ Z = np.dot(W, X) + b
```

### Bug 5: Confused dZ with dA
```
dZ = derivative w.r.t. PRE-activation Z
dA = derivative w.r.t. POST-activation A

dZ = dA × g'(Z)   ← convert dA to dZ
```

---

## 🛠️ Shape Debugging Workflow

```
When you get a shape mismatch error:

1. Print all shapes:
   print("X:", X.shape)
   print("W:", W.shape)
   print("b:", b.shape)
   
2. Check matrix multiply rule:
   (A.shape[1] == B.shape[0]) for A @ B
   
3. Trace through forward pass step by step
   
4. Look for missing transpose (.T)
   
5. Check axis= and keepdims= in sum operations
```

---

## 💡 Mental Model

```
Think of W[L] as a "translator":
   Takes vectors from layer L-1 space (n_{L-1} dimensional)
   Outputs vectors in layer L space (n_L dimensional)
   
   Hence: W[L] has shape (output_size, input_size)
                       = (n[L], n[L-1])
   
This is the STANDARD math convention everywhere.
```

---

## 🎯 Quick Self-Test

Without looking, fill in:

```
Network: 5 → 10 → 8 → 3 → 1, m = 100 examples

X.shape    = (___, ___)
W[1].shape = (___, ___)
b[1].shape = (___, ___)
A[1].shape = (___, ___)
W[3].shape = (___, ___)
A[3].shape = (___, ___)
ŷ.shape    = (___, ___)
```

<details>
<summary>Answers (click to expand)</summary>

```
X.shape    = (5, 100)
W[1].shape = (10, 5)
b[1].shape = (10, 1)
A[1].shape = (10, 100)
W[3].shape = (3, 8)
A[3].shape = (3, 100)
ŷ.shape    = (1, 100)
```
</details>
