# Deep Learning Prep — Master Index

> A complete reference for deep neural network concepts.
> Following the 5-Layer Learning System: Cheat Sheets → Diagrams → Decision Trees → Real Examples → Active Recall.

---

## 📚 Folder Structure

```
DeepLearningPrep/
├── 01-Foundations/           Math basics: matrices, vectors, derivatives
├── 02-NeuralNetworks/        Architecture, layers, neurons, notation
├── 03-ActivationFunctions/   ReLU, Sigmoid, Tanh, Linear, Softmax
├── 04-ForwardPropagation/    The forward pass math
├── 05-BackPropagation/       Gradients & chain rule
├── 06-CostAndLoss/           MSE, Cross-Entropy
├── 07-Regularization/        L1, L2, preventing overfitting
├── 08-DataSplitting/         Train/Dev/Test, Bias/Variance
├── 09-ImplementationPatterns/Code patterns & vectorization
└── 10-CheatSheets/           One-page summaries
```

---

## 🎯 Learning Path (Recommended Order)

| # | Topic | File |
|---|-------|------|
| 1 | Matrix Shapes & Notation | `02-NeuralNetworks/01-notation-and-shapes.md` |
| 2 | Network Architecture | `02-NeuralNetworks/02-architecture.md` |
| 3 | Activation Functions | `03-ActivationFunctions/01-all-activations.md` |
| 4 | Why Non-Linear Activations | `03-ActivationFunctions/02-why-nonlinear.md` |
| 5 | Forward Propagation | `04-ForwardPropagation/01-forward-pass.md` |
| 6 | Vectorization | `04-ForwardPropagation/02-vectorization.md` |
| 7 | Loss & Cost Functions | `06-CostAndLoss/01-loss-functions.md` |
| 8 | Backpropagation | `05-BackPropagation/01-backward-pass.md` |
| 9 | Gradient Descent Update | `05-BackPropagation/02-weight-updates.md` |
| 10 | Train/Dev/Test Sets | `08-DataSplitting/01-data-splits.md` |
| 11 | Bias & Variance | `08-DataSplitting/02-bias-variance.md` |
| 12 | L1 & L2 Regularization | `07-Regularization/01-l1-l2-regularization.md` |
| 13 | Building DNN from Scratch | `09-ImplementationPatterns/01-build-dnn.md` |

---

## 🃏 Quick-Access Cheat Sheets

- **One-Page Master Reference**: `10-CheatSheets/01-master-cheat-sheet.md`
- **All Decision Trees**: `10-CheatSheets/02-decision-trees.md`
- **Shape Quick Reference**: `10-CheatSheets/03-shapes-reference.md`

---

## 📊 Visual Diagrams (Mermaid)

All diagrams use Mermaid for version control. Render in:
- GitHub (automatic)
- Obsidian (with Mermaid plugin)
- VSCode (with Markdown Preview Mermaid extension)
- Notion (paste as code block)

---

## 🎓 Real World Anchors

Tie every concept to a real system:

| Concept | Real System Example |
|---------|---------------------|
| Deep CNN | Image recognition (Google Photos, Tesla Autopilot) |
| LSTM/RNN | Language translation (Google Translate) |
| Transformer | ChatGPT, BERT, Claude |
| ReLU | All modern computer vision networks |
| Batch Normalization | ResNet, used in production at every major tech company |
| Dropout | Speech recognition (Siri, Alexa) |

---

## 💡 Daily Routine

```
Monday-Friday (1 hour/day):
  20 min → Read one topic + take notes
  20 min → Draw diagram + cheat sheet
  20 min → Review previous topics (active recall)

Weekend (2 hours):
  → Implement one concept in code from scratch
  → Explain a topic out loud (Feynman method)
```

---

## ✅ Mastery Checklist

For each topic, check if you can:

- [ ] Explain the concept in ONE sentence
- [ ] Draw the diagram from memory
- [ ] State the key formula
- [ ] Give a real-world example
- [ ] Name when to use it
- [ ] Name when NOT to use it
- [ ] State the trade-offs
