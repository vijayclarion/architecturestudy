# Python for Machine Learning — Essential Guide

> Everything you actually need to start ML. Each concept has a simple, runnable example.
> Focus: the 20% of Python you'll use 80% of the time in ML.

---

## Table of Contents

1. [Python Basics You Must Know](#1-python-basics-you-must-know)
2. [Lists, Dicts, and Comprehensions](#2-lists-dicts-and-comprehensions)
3. [Functions](#3-functions)
4. [NumPy — The Heart of ML](#4-numpy--the-heart-of-ml)
5. [Pandas — Data Handling](#5-pandas--data-handling)
6. [Matplotlib — Visualization](#6-matplotlib--visualization)
7. [Scikit-learn — Your First Model](#7-scikit-learn--your-first-model)
8. [Putting It All Together](#8-putting-it-all-together)

---

# 1. Python Basics You Must Know

## Variables and Types

```python
# Numbers
learning_rate = 0.01        # float
epochs = 100                # int

# Strings
model_name = "neural_net"

# Booleans
is_training = True

# Check the type
print(type(learning_rate))  # <class 'float'>
```

## The 4 Operations You'll Use Constantly

```python
a = 10
b = 3

print(a + b)    # 13  addition
print(a / b)    # 3.333  division (always float)
print(a // b)   # 3   integer division
print(a ** 2)   # 100  power (a squared)
print(a % b)    # 1   remainder (modulo)
```

## Conditionals

```python
accuracy = 0.92

if accuracy > 0.90:
    print("Great model!")
elif accuracy > 0.70:
    print("Decent model")
else:
    print("Needs work")

# Output: Great model!
```

## Loops

```python
# Loop a fixed number of times
for epoch in range(3):
    print(f"Epoch {epoch}")
# Epoch 0
# Epoch 1
# Epoch 2

# Loop over a collection
losses = [0.5, 0.3, 0.1]
for loss in losses:
    print(f"Loss: {loss}")
```

## f-strings (formatting output)

```python
epoch = 5
loss = 0.0234

# The clean way to build strings
print(f"Epoch {epoch}: loss = {loss:.2f}")
# Epoch 5: loss = 0.02

# :.2f means "show 2 decimal places"
```

---

# 2. Lists, Dicts, and Comprehensions

## Lists — ordered collections

```python
features = [5.1, 3.5, 1.4, 0.2]

# Access by index (starts at 0)
print(features[0])     # 5.1  first
print(features[-1])    # 0.2  last

# Slicing [start:end]  (end not included)
print(features[0:2])   # [5.1, 3.5]
print(features[2:])    # [1.4, 0.2]

# Add items
features.append(9.9)
print(len(features))   # 5
```

## Dictionaries — key-value pairs

```python
# Great for storing model config / results
config = {
    "learning_rate": 0.01,
    "epochs": 100,
    "layers": [64, 32, 1]
}

print(config["learning_rate"])   # 0.01
config["batch_size"] = 32        # add new key

# Loop over a dict
for key, value in config.items():
    print(f"{key}: {value}")
```

## List Comprehensions — ML's favorite shortcut

```python
# Instead of this:
squares = []
for x in range(5):
    squares.append(x ** 2)

# Write this (one line!):
squares = [x ** 2 for x in range(5)]
print(squares)   # [0, 1, 4, 9, 16]

# With a condition
evens = [x for x in range(10) if x % 2 == 0]
print(evens)     # [0, 2, 4, 6, 8]

# Common in ML: normalize a list
data = [10, 20, 30, 40]
normalized = [x / max(data) for x in data]
print(normalized)   # [0.25, 0.5, 0.75, 1.0]
```

---

# 3. Functions

```python
# Basic function
def predict(x, weight, bias):
    return x * weight + bias

result = predict(5, 2, 1)
print(result)   # 11

# Default arguments
def train(data, learning_rate=0.01, epochs=100):
    print(f"Training with lr={learning_rate}, epochs={epochs}")

train([1, 2, 3])                    # uses defaults
train([1, 2, 3], learning_rate=0.1) # override one
```

## A real ML function — Mean Squared Error

```python
def mean_squared_error(predictions, targets):
    total = 0
    for p, t in zip(predictions, targets):   # zip pairs them up
        total += (p - t) ** 2
    return total / len(predictions)

preds = [2.5, 0.0, 2.1]
true  = [3.0, -0.5, 2.0]
print(mean_squared_error(preds, true))   # 0.1700
```

---

# 4. NumPy — The Heart of ML

**NumPy is the most important library.** Almost all ML math is NumPy arrays.

```python
import numpy as np
```

## Creating Arrays

```python
# From a list
a = np.array([1, 2, 3])
print(a)          # [1 2 3]

# 2D array (matrix)
M = np.array([[1, 2, 3],
              [4, 5, 6]])
print(M.shape)    # (2, 3)  → 2 rows, 3 columns

# Useful starters
zeros = np.zeros((2, 3))      # 2x3 of zeros
ones  = np.ones((3,))         # [1. 1. 1.]
rng   = np.arange(0, 10, 2)   # [0 2 4 6 8]
grid  = np.linspace(0, 1, 5)  # [0. 0.25 0.5 0.75 1.]
```

## Why NumPy? Vectorization (this is the key idea)

```python
# SLOW — pure Python loop
python_list = [1, 2, 3, 4]
doubled = [x * 2 for x in python_list]

# FAST — NumPy does it all at once
arr = np.array([1, 2, 3, 4])
doubled = arr * 2
print(doubled)   # [2 4 6 8]

# Operations apply to EVERY element automatically:
print(arr + 10)   # [11 12 13 14]
print(arr ** 2)   # [1  4  9 16]
print(arr / 2)    # [0.5 1.  1.5 2. ]
```

> **Why it matters:** NumPy runs these operations in optimized C code, often 10–100x faster than Python loops. ML datasets are huge, so this speed is essential.

## Array Math (used everywhere in ML)

```python
a = np.array([1, 2, 3])
b = np.array([4, 5, 6])

# Element-wise
print(a + b)        # [5 7 9]
print(a * b)        # [4 10 18]

# Dot product (core of neural networks!)
print(np.dot(a, b)) # 32  → (1*4 + 2*5 + 3*6)

# Statistics
print(a.sum())      # 6
print(a.mean())     # 2.0
print(a.max())      # 3
print(a.std())      # 0.816...
```

## Matrix Multiplication (the basis of neural nets)

```python
# A neuron computes: output = inputs · weights + bias
inputs  = np.array([1.0, 2.0, 3.0])      # shape (3,)
weights = np.array([0.2, 0.8, -0.5])     # shape (3,)
bias    = 2.0

output = np.dot(inputs, weights) + bias
print(output)   # 2.3

# For matrices, use @ or np.matmul
A = np.array([[1, 2],
              [3, 4]])      # 2x2
B = np.array([[5, 6],
              [7, 8]])      # 2x2
print(A @ B)
# [[19 22]
#  [43 50]]
```

## Indexing and Slicing

```python
M = np.array([[1, 2, 3],
              [4, 5, 6],
              [7, 8, 9]])

print(M[0, 0])     # 1    row 0, col 0
print(M[1, :])     # [4 5 6]   entire row 1
print(M[:, 0])     # [1 4 7]   entire column 0
print(M[0:2, 1:3]) # [[2 3]
                   #  [5 6]]   sub-matrix

# Boolean masking (very common for filtering data)
data = np.array([1, 5, 2, 8, 3])
print(data[data > 3])   # [5 8]
```

## Reshaping (critical for ML data prep)

```python
a = np.arange(6)        # [0 1 2 3 4 5]
print(a.reshape(2, 3))  # [[0 1 2]
                        #  [3 4 5]]

# -1 means "figure it out automatically"
print(a.reshape(3, -1)) # [[0 1]
                        #  [2 3]
                        #  [4 5]]
```

---

# 5. Pandas — Data Handling

**Pandas loads and cleans your data** before you feed it to a model.

```python
import pandas as pd
```

## DataFrames — like a spreadsheet in code

```python
# Create from a dictionary
df = pd.DataFrame({
    "age":    [25, 32, 47, 51],
    "salary": [50000, 60000, 80000, 90000],
    "bought": [0, 0, 1, 1]
})

print(df)
#    age  salary  bought
# 0   25   50000       0
# 1   32   60000       0
# 2   47   80000       1
# 3   51   90000       1
```

## Loading Real Data

```python
# The most common first line in any ML project:
df = pd.read_csv("data.csv")

# Quick inspection
df.head()        # first 5 rows
df.shape         # (rows, columns)
df.info()        # column types & missing values
df.describe()    # statistics for each column
```

## Selecting Data

```python
# One column (returns a Series)
print(df["age"])

# Multiple columns
print(df[["age", "salary"]])

# Filter rows by condition
high_earners = df[df["salary"] > 70000]
print(high_earners)

# Combine conditions (use & for and, | for or)
result = df[(df["age"] > 30) & (df["bought"] == 1)]
```

## Common Cleaning Tasks

```python
# Check for missing values
print(df.isnull().sum())

# Fill missing values with the mean
df["age"] = df["age"].fillna(df["age"].mean())

# Drop rows with missing data
df = df.dropna()

# Create a new column
df["salary_k"] = df["salary"] / 1000
```

## Splitting Features and Target (essential ML step)

```python
# X = features (inputs), y = target (what we predict)
X = df[["age", "salary"]]    # the inputs
y = df["bought"]             # the answer

# Convert to NumPy for ML libraries
X = X.values
y = y.values
```

---

# 6. Matplotlib — Visualization

**See your data before modeling.** Catch problems early.

```python
import matplotlib.pyplot as plt
```

## Line Plot — track training loss

```python
epochs = [1, 2, 3, 4, 5]
loss   = [0.9, 0.6, 0.4, 0.25, 0.15]

plt.plot(epochs, loss)
plt.xlabel("Epoch")
plt.ylabel("Loss")
plt.title("Training Loss Over Time")
plt.show()
```

## Scatter Plot — see relationships

```python
age    = [25, 32, 47, 51, 62]
salary = [50, 60, 80, 90, 100]

plt.scatter(age, salary)
plt.xlabel("Age")
plt.ylabel("Salary (k)")
plt.title("Age vs Salary")
plt.show()
```

## Histogram — see distributions

```python
import numpy as np
data = np.random.randn(1000)   # 1000 random values

plt.hist(data, bins=30)
plt.title("Distribution of Values")
plt.show()
```

## Multiple Plots

```python
x = np.linspace(0, 10, 100)

plt.plot(x, np.sin(x), label="sin")
plt.plot(x, np.cos(x), label="cos")
plt.legend()      # show labels
plt.show()
```

---

# 7. Scikit-learn — Your First Model

**The standard library for classic ML.** Same pattern for every model.

```python
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error
```

## The Universal Scikit-learn Pattern

```python
# Every model follows these steps:
# 1. Split data → 2. Create model → 3. Fit → 4. Predict → 5. Evaluate
```

## Complete Example — Predicting House Prices

```python
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error, r2_score

# --- Fake data: house size (sq ft) → price (k) ---
X = np.array([[650], [800], [1200], [1500], [2000], [2500]])
y = np.array([150, 180, 250, 310, 400, 480])

# 1. SPLIT into train and test (80/20)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# 2. CREATE the model
model = LinearRegression()

# 3. FIT (train) on training data
model.fit(X_train, y_train)

# 4. PREDICT on test data
predictions = model.predict(X_test)

# 5. EVALUATE
mse = mean_squared_error(y_test, predictions)
print(f"MSE: {mse:.2f}")

# Predict a brand new house
new_house = np.array([[1800]])
print(f"Predicted price: {model.predict(new_house)[0]:.0f}k")

# Look at what the model learned
print(f"Slope (price per sqft): {model.coef_[0]:.3f}")
print(f"Intercept: {model.intercept_:.1f}")
```

## Classification Example — will a customer buy?

```python
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score

# Features: [age, salary_k]   Target: bought (0 or 1)
X = np.array([[25, 50], [32, 60], [47, 80], [51, 90], [62, 100]])
y = np.array([0, 0, 1, 1, 1])

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.4, random_state=42
)

model = LogisticRegression()
model.fit(X_train, y_train)

predictions = model.predict(X_test)
accuracy = accuracy_score(y_test, predictions)
print(f"Accuracy: {accuracy:.0%}")
```

## Feature Scaling (often needed before training)

```python
from sklearn.preprocessing import StandardScaler

# Many models work better when features are on similar scales
scaler = StandardScaler()

X_train_scaled = scaler.fit_transform(X_train)  # fit + transform train
X_test_scaled  = scaler.transform(X_test)       # only transform test

# Note: fit only on training data to avoid "data leakage"
```

---

# 8. Putting It All Together

A complete mini ML project using everything above:

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score

# --- 1. CREATE / LOAD DATA ---
df = pd.DataFrame({
    "age":    [22, 25, 47, 52, 46, 56, 23, 30, 40, 60],
    "salary": [25, 30, 80, 95, 78, 100, 28, 45, 70, 110],
    "bought": [0, 0, 1, 1, 1, 1, 0, 0, 1, 1]
})

# --- 2. EXPLORE ---
print(df.describe())

plt.scatter(df["age"], df["salary"], c=df["bought"])
plt.xlabel("Age"); plt.ylabel("Salary (k)")
plt.title("Customers (color = bought)")
plt.show()

# --- 3. PREPARE ---
X = df[["age", "salary"]].values
y = df["bought"].values

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42
)

# --- 4. SCALE ---
scaler = StandardScaler()
X_train = scaler.fit_transform(X_train)
X_test  = scaler.transform(X_test)

# --- 5. TRAIN ---
model = LogisticRegression()
model.fit(X_train, y_train)

# --- 6. EVALUATE ---
predictions = model.predict(X_test)
print(f"Accuracy: {accuracy_score(y_test, predictions):.0%}")

# --- 7. PREDICT NEW CUSTOMER ---
new = scaler.transform([[35, 55]])   # age 35, salary 55k
print(f"Will buy: {model.predict(new)[0]}")
print(f"Probability: {model.predict_proba(new)[0][1]:.0%}")
```

---

# Quick Reference Cheat Sheet

## The Imports You'll Always Use

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
```

## NumPy Essentials

| Task | Code |
|------|------|
| Create array | `np.array([1,2,3])` |
| Zeros / ones | `np.zeros((2,3))`, `np.ones(5)` |
| Range | `np.arange(0,10,2)` |
| Shape | `arr.shape` |
| Reshape | `arr.reshape(2,3)` |
| Dot product | `np.dot(a,b)` or `a @ b` |
| Mean / sum | `arr.mean()`, `arr.sum()` |
| Filtering | `arr[arr > 5]` |

## Pandas Essentials

| Task | Code |
|------|------|
| Load CSV | `pd.read_csv("file.csv")` |
| First rows | `df.head()` |
| Shape | `df.shape` |
| Stats | `df.describe()` |
| Select column | `df["col"]` |
| Filter rows | `df[df["col"] > 5]` |
| Missing values | `df.isnull().sum()` |
| Fill missing | `df.fillna(value)` |

## Scikit-learn Pattern

| Step | Code |
|------|------|
| Split | `train_test_split(X, y, test_size=0.2)` |
| Create | `model = LinearRegression()` |
| Train | `model.fit(X_train, y_train)` |
| Predict | `model.predict(X_test)` |
| Score | `accuracy_score(y_test, preds)` |

---

# What to Learn Next

1. **More models**: `DecisionTree`, `RandomForest`, `SVM`, `KNN` — all use the same fit/predict pattern
2. **Cross-validation**: `cross_val_score` for more reliable evaluation
3. **Pipelines**: `Pipeline` to chain scaling + model cleanly
4. **Deep learning**: PyTorch or TensorFlow when you outgrow scikit-learn
5. **Pandas deeper**: `groupby`, `merge`, `pivot_table` for real data wrangling

---

## Practice Tips

- **Type every example** — don't copy-paste. Muscle memory matters.
- **Break things on purpose** — change values, see what errors look like.
- **Start with NumPy + Pandas** — they're 80% of daily ML Python work.
- **One library at a time** — master NumPy before moving to scikit-learn.
