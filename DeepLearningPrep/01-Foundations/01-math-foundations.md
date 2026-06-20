# Math Foundations for Deep Learning

## Problem It Solves
Deep learning sits on a base of linear algebra and calculus. You don't need a PhD, but you need these specific concepts to understand what's happening under the hood.

## Essential Concepts

### 1. Vectors and Matrices

```
VECTOR: a list of numbers (column or row)
   x = [1, 2, 3]   ← 1D
   
   In neural networks: each example or layer activation is a vector

MATRIX: a 2D grid of numbers
   W = [[1, 2], [3, 4]]  ← 2D
   
   In neural networks: weights of a layer
```

### 2. Matrix Multiplication

```
A · B works when: columns of A == rows of B

(m, k) · (k, n) = (m, n)

Example:
   A = (3, 4)
   B = (4, 5)
   A · B = (3, 5) ✅

In Python:
   np.dot(A, B)   or   A @ B
```

### 3. Transpose

```
A.T flips rows and columns

A shape = (3, 4)
A.T shape = (4, 3)

A = [1  2  3  4]      A.T = [1]
    [5  6  7  8]            [2]
    [9 10 11 12]            [3]
                            [4]
                            [...]
```

### 4. Element-Wise Operations

```
SAME shape needed:
   [1, 2, 3] * [4, 5, 6] = [4, 10, 18]
   [1, 2, 3] + [4, 5, 6] = [5, 7, 9]

In Python:
   A * B   (element-wise multiplication)
   A + B   (element-wise addition)
```

### 5. Broadcasting

```
NumPy auto-expands smaller arrays:

(3, 4) + (4,)  → broadcasts to (3, 4) + (3, 4)
(3, 1) + (3, 4) → broadcasts to (3, 4) + (3, 4)

CRUCIAL for bias addition:
   Z = W·X + b
   (n, m) + (n, 1) ← b broadcasts to (n, m)
```

---

## Calculus (Just the Essentials)

### 1. Derivatives

```
Derivative = rate of change

For f(x) = x², the derivative is f'(x) = 2x

WHY DO WE CARE?
   In neural networks, we need to know:
   "If I change weight W slightly, how much does loss change?"
   That's exactly what a derivative measures!
```

### 2. Chain Rule (THE KEY)

```
If y = f(g(x)), then:
   dy/dx = (dy/dg) × (dg/dx)

VISUAL:
   x → g(x) → f(g(x)) = y
   
   To find dy/dx, multiply derivatives along the chain:
   x → [×dg/dx] → g(x) → [×dy/dg] → y

This is the BASIS of backpropagation!
```

### 3. Partial Derivatives

```
When function has multiple variables:
   f(x, y) = x² + 3y

   ∂f/∂x = 2x    (treat y as constant)
   ∂f/∂y = 3     (treat x as constant)

In neural networks:
   Loss depends on ALL weights
   Take partial derivative w.r.t. each weight
   This gives the GRADIENT
```

### 4. Gradient

```
Gradient = vector of ALL partial derivatives

For function f(w₁, w₂, w₃):
   gradient = [∂f/∂w₁, ∂f/∂w₂, ∂f/∂w₃]
            = dW
            
This tells us:
   - Direction of steepest INCREASE
   - We go OPPOSITE direction to MINIMIZE loss
```

---

## Statistics (Bare Minimum)

### Mean (Average)
```
mean = (sum of values) / count

For [2, 4, 6]:
   mean = (2 + 4 + 6) / 3 = 4
   
In Python: np.mean(array)
```

### Variance & Standard Deviation
```
Variance = how spread out the data is
Standard deviation (σ) = sqrt(variance)

USE IN ML:
   Normalizing data: x_norm = (x - mean) / std
```

### Probability
```
For binary classification:
   sigmoid(z) outputs a probability
   0 ≤ p ≤ 1

   p = 0.9 means "90% chance positive class"
```

---

## Mental Models

### Matrix as Transformation
```
W·x = "transform vector x by W"
   x is in one space
   W·x is in another space
   
Neural networks are series of transformations:
   x → W1·x → W2·(W1·x) → ...
```

### Gradient as Slope
```
On a hill:
   Gradient = slope direction (steepest uphill)
   -Gradient = downhill direction
   
We use -Gradient to descend the LOSS hill
   = "Gradient Descent"
```

### Function as Recipe
```
Activation function g(z) is a recipe:
   Input: z (any number)
   Process: apply g (ReLU, Sigmoid, etc.)
   Output: a (transformed number)
```

---

## Key Operations in Practice

### Computing a Linear Layer
```python
# z = W·a + b
z = np.dot(W, a) + b
# OR
z = W @ a + b
```

### Computing Cost
```python
# Mean of losses
cost = np.mean(losses)
# OR
cost = (1/m) * np.sum(losses)
```

### Computing Gradient
```python
# Average gradient across batch
dW = (1/m) * np.dot(dZ, A_prev.T)
db = (1/m) * np.sum(dZ, axis=1, keepdims=True)
```

---

## When You Don't Need More Math

```
DON'T need:
   ❌ Real analysis
   ❌ Measure theory
   ❌ Tensor calculus
   ❌ Stochastic differential equations

DO need:
   ✅ Matrix multiplication
   ✅ Chain rule
   ✅ Partial derivatives (intuition)
   ✅ Basic probability
   ✅ Vector operations
```

---

## Interview Soundbite
> "Deep learning is built on linear algebra (matrix multiplication for forward pass) and calculus (chain rule for backpropagation). You don't need advanced math — just solid understanding of matrix dimensions, gradients, and basic derivatives."

## Practice Examples

```python
import numpy as np

# Matrix multiplication
W = np.array([[1, 2], [3, 4]])    # shape (2, 2)
x = np.array([[5], [6]])           # shape (2, 1)
z = W @ x                          # shape (2, 1)
# z = [[17], [39]]
#  Row 1: 1*5 + 2*6 = 17
#  Row 2: 3*5 + 4*6 = 39

# Element-wise multiplication
a = np.array([1, 2, 3])
b = np.array([4, 5, 6])
c = a * b                          # [4, 10, 18]

# Transpose
A = np.array([[1, 2, 3], [4, 5, 6]])  # shape (2, 3)
print(A.T)                              # shape (3, 2)

# Sum along axis
M = np.array([[1, 2], [3, 4]])
np.sum(M, axis=0)  # sum DOWN columns:   [4, 6]
np.sum(M, axis=1)  # sum ACROSS rows:    [3, 7]
```
