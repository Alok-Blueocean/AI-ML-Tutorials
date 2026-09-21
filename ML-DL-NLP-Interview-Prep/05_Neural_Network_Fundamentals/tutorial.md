# Neural Network Fundamentals

## Perceptron and the XOR Problem

A perceptron computes `y = step(w·x + b)` — a single linear boundary. It can learn AND/OR (linearly separable) but not XOR, because no single straight line separates XOR's classes. Minsky & Papert's 1969 critique of this limitation nearly killed neural net research for a decade. The fix is depth: stack two layers (hidden layer + output layer) so the network can combine two half-plane decisions into a non-linear boundary.

**Real-world grounding:** fraud detection where "high amount AND new device" is fraud but "high amount alone" or "new device alone" is not — this XOR-like interaction is exactly why a linear model (logistic regression) plateaus and a small MLP with one hidden layer clears the plateau.

## Forward Pass and Backpropagation (Chain Rule)

Forward pass: `z^[l] = W^[l] a^[l-1] + b^[l]`, `a^[l] = f(z^[l])`, ending in a loss. Backprop applies the chain rule from the loss backward: `dL/dW^[l] = dL/dz^[l] · a^[l-1]^T`, where `dL/dz^[l]` is propagated from layer `l+1` via `W^[l+1]^T`. It's just reverse-mode autodiff — the same machinery `torch.autograd` runs under the hood.

**Real-world grounding:** debugging a model that "isn't learning" almost always means inspecting where in this chain gradients die (e.g., saturated sigmoid at layer 2 of 20 zeroes out everything upstream).

```python
import numpy as np
def forward(X, W1, b1, W2, b2):
    z1 = X @ W1 + b1; a1 = np.tanh(z1)
    z2 = a1 @ W2 + b2; a2 = 1 / (1 + np.exp(-z2))  # sigmoid output
    return z1, a1, z2, a2

def backward(X, y, z1, a1, a2, W2):
    m = X.shape[0]
    dz2 = a2 - y                      # dL/dz2 for BCE + sigmoid
    dW2 = a1.T @ dz2 / m
    db2 = dz2.mean(0)
    da1 = dz2 @ W2.T
    dz1 = da1 * (1 - a1 ** 2)         # tanh'
    dW1 = X.T @ dz1 / m
    db1 = dz1.mean(0)
    return dW1, db1, dW2, db2
```

## Activation Functions and Vanishing/Exploding Gradients

- **Sigmoid** (`1/(1+e^-x)`): saturates at both ends, gradient → 0 for |x| large. Used only at output for binary probabilities now.
- **Tanh**: zero-centered sigmoid variant, still saturates.
- **ReLU** (`max(0,x)`): no saturation for x>0, cheap, but "dying ReLU" — a neuron stuck at x<0 forever gets zero gradient.
- **Leaky ReLU** (`max(0.01x, x)`): small negative slope keeps dead neurons alive.
- **GELU** (`x·Φ(x)`): smooth, probabilistic gating, default in transformers (BERT, GPT) because it slightly outperforms ReLU on large models.

Vanishing gradients: multiplying many sigmoid/tanh derivatives (<1) through 20+ layers shrinks the gradient toward zero at early layers — the classic reason plain deep MLPs/RNNs pre-2015 topped out around 5-10 layers. Exploding gradients: the opposite, from large weights or long unrolled recurrence, causing NaN losses — fixed with gradient clipping.

**Real-world grounding:** switching a 10-layer tabular fraud model's hidden activations from sigmoid to ReLU is often the single change that turns "loss stuck at 0.69 forever" into a model that actually trains.

## Loss Functions and Softmax

- Regression: MSE (`(y-ŷ)²`), MAE for outlier-robustness.
- Binary classification: binary cross-entropy on a sigmoid output.
- Multi-class: categorical cross-entropy on a softmax output — `softmax(z)_i = e^{z_i} / Σ_j e^{z_j}`, converting logits to a probability distribution that sums to 1.
- Cross-entropy's gradient w.r.t. logits simplifies beautifully to `(ŷ - y)`, which is why softmax + cross-entropy is always paired, never softmax + MSE.

**Real-world grounding:** a 1000-class product-category classifier uses softmax + categorical cross-entropy; a multi-label tagging system (an image can have multiple tags) instead uses independent sigmoids + binary cross-entropy per label — mixing these up is a common production bug.

## Weight Initialization (Xavier/He)

Initializing all weights to zero makes every neuron in a layer compute the same gradient (symmetry problem) — the network never differentiates. Random init breaks symmetry, but scale matters:
- **Xavier/Glorot**: `Var(W) = 1/n_in` (or `2/(n_in+n_out)`), designed for tanh/sigmoid to keep activation variance stable across layers.
- **He initialization**: `Var(W) = 2/n_in`, designed for ReLU (since ReLU zeroes half the inputs, you need double the variance to compensate).

**Real-world grounding:** using Xavier init with ReLU activations in a 50-layer net is a subtle bug that manifests as slower convergence, not a crash — this is why PyTorch's `nn.Linear` defaults and `kaiming_normal_` matter in practice.

## Optimizers

- **SGD + momentum**: `v = βv + ∇L`, `w -= ηv`. Momentum smooths noisy gradients and helps escape shallow local minima/saddle points.
- **RMSProp**: divides the learning rate by a running average of squared gradients per-parameter — adapts step size, good for non-stationary objectives (RNNs).
- **Adam**: combines momentum (1st moment) + RMSProp (2nd moment) with bias correction. Default choice for most deep learning today because it "just works" with minimal tuning.

**Real-world grounding:** Adam with default `lr=1e-3` is the standard starting point for a new architecture; SGD+momentum with a tuned learning-rate schedule is still preferred for final-accuracy image classification (ResNet on ImageNet) because it generalizes marginally better.

## Regularization

- **Dropout**: randomly zero a fraction `p` of activations each forward pass during training, forcing redundant representations (an ensemble-of-subnetworks effect). Disabled at inference (weights scaled by `1-p`, or inverted dropout scales during training).
- **Batch normalization**: normalizes layer inputs to zero mean/unit variance per mini-batch, then learns a scale/shift. Stabilizes and accelerates training, has a mild regularizing side effect from batch noise.
- **Early stopping**: stop training when validation loss stops improving — the cheapest regularizer, needs no code change beyond monitoring.
- **Weight decay (L2)**: adds `λ‖W‖²` to the loss, penalizing large weights and encouraging simpler decision boundaries. In Adam, prefer decoupled weight decay (AdamW) — naive L2 in Adam interacts badly with the adaptive learning rate.

**Real-world grounding:** a churn-prediction MLP that scores 99% train / 78% validation accuracy is fixed by adding dropout(0.3-0.5) between dense layers plus early stopping, before reaching for a bigger model.

## Universal Approximation Theorem

A feedforward network with a single hidden layer of enough (possibly huge) width can approximate any continuous function on a compact domain to arbitrary precision. In practice this is not an argument for wide-shallow networks — it says nothing about *learnability* or *sample efficiency*. Depth is what makes learning practical: deep networks can represent the same functions with exponentially fewer parameters than a single very wide layer, and gradient descent finds good solutions in deep architectures far more reliably.

**Real-world grounding:** this theorem is why "can a neural net learn this function in principle" is never the real question in an interview — the real question is architecture, depth, and optimization dynamics, which is what the rest of this course covers.

## Designing a Neural Network Architecture — a Practical Method

Interviewers care less about "what layers did you pick" and more about the *process*. A working method:

1. **Start with the simplest model that could plausibly work.** For tabular data, that's often a gradient-boosted tree, not a neural net — trees handle mixed feature types, need no scaling, and win on small-to-medium row counts with heavy tuning cost savings (see `04_Ensemble_Methods_Bagging_Boosting`). Reach for a neural net when: the data is unstructured (image/text/audio), you need learned representations shared across tasks, the dataset is large enough (roughly 10k+ rows per class) to justify it, or you specifically need end-to-end differentiability (e.g. joint feature learning + prediction).
2. **Size the input layer from the data, not intuition.** Tabular: one input unit per feature (after encoding). Images: flattened pixel count for an MLP, or just the `(H, W, C)` tensor shape for a CNN's first conv layer. Text: vocabulary size (for one-hot/embedding lookup) or embedding dimension if using pretrained embeddings.
3. **Pick depth/width with the capacity-then-regularize loop:** start narrow and shallow (e.g. 1-2 hidden layers, width between input and output size). Train. If the model *underfits* (train loss stays high), add capacity — one more layer or wider layers — and retrain. If the model *overfits* (train/val gap widens), stop adding capacity and add regularization (dropout, weight decay, early stopping) instead. Never do both at once; you can't tell which change caused what.
4. **Size and pick the output layer strictly from the task** — this is the one place where there's no ambiguity, listed below.

| Task | Output units | Output activation | Loss |
|---|---|---|---|
| Binary classification | 1 | sigmoid | binary cross-entropy |
| Multi-class (single label) | N classes | softmax | categorical / sparse categorical cross-entropy |
| Multi-label (independent tags) | N labels | sigmoid (per unit, independent) | binary cross-entropy (per label, summed/averaged) |
| Regression (unbounded) | 1 | linear/identity | MSE / MAE / Huber |
| Regression (positive-only, e.g. wait time) | 1 | softplus / exp | MSE on log-target, or Poisson loss |
| Embedding / similarity learning | D (embedding dim) | linear (often L2-normalized) | triplet / contrastive loss |

**Real-world grounding:** a candidate who says "I'd add layers until validation accuracy stops improving, then regularize" demonstrates the actual engineering loop; a candidate who names an arbitrary architecture ("ResNet-50, 4 dense layers, dropout 0.5") without justifying it against the data volume/task usually hasn't done this in production.

## Flatten and Reshaping Layers

`Flatten` collapses a multi-dimensional tensor into a 1D vector per sample — e.g. a CNN's final feature map of shape `(H, W, C)` (say `7×7×512`) becomes a single vector of length `H·W·C` (25,088) so a `Dense` layer can consume it. It has no learnable parameters; it's purely a reshape.

**Why it's a common bug source:** if an earlier conv/pooling layer's stride or padding changes, the flattened size changes too, and the first `Dense` layer's expected input size silently mismatches — this is the single most common "shape mismatch" error when porting a CNN between input resolutions.

**The modern alternative — Global Average Pooling (GAP):** instead of flattening `(H, W, C)` into `H·W·C` values, GAP averages each of the `C` channels over the full `H×W` spatial extent, producing a `C`-length vector. Consequences:
- Far fewer parameters into the next `Dense` layer (`C` inputs instead of `H·W·C`) — for the 7×7×512 example, 512 vs 25,088, a ~49x reduction, which sharply cuts overfitting risk in the classifier head.
- Works with *any* input spatial size (no fixed `H, W` baked into the flattened length), so the same trained backbone accepts variable image resolutions.
- Loses fine-grained spatial position information — acceptable for whole-image classification, less so for tasks needing spatial layout (segmentation, detection use different heads).

**Real-world grounding:** ResNet, Inception, and most modern CNN classifiers use `GlobalAveragePooling2D` immediately before the final `Dense(num_classes)` layer, not `Flatten` — this is why ResNet's classifier head has orders of magnitude fewer parameters than the AlexNet/VGG-era `Flatten → Dense(4096) → Dense(4096)` pattern, and why ResNet generalizes better with less overfitting for comparable depth.

```python
# Keras
x = layers.GlobalAveragePooling2D()(feature_map)   # (H,W,C) -> (C,)
# vs.
x = layers.Flatten()(feature_map)                  # (H,W,C) -> (H*W*C,)
```

## Regularization — Where and How to Actually Add It

Knowing regularizers exist isn't enough — placement and magnitude matter.

**L1/L2 weight decay** — applies to any layer with learnable weights (Dense, Conv). Penalizes weight magnitude in the loss (`L2: λΣw²`, `L1: λΣ|w|`, which also induces sparsity/feature selection). Typical `λ`: `1e-4` to `1e-2`; start at `1e-4` and increase if still overfitting.
```python
# Keras: per-layer
layers.Dense(64, kernel_regularizer=keras.regularizers.l2(1e-4))
# PyTorch: usually global, via the optimizer (AdamW decouples it correctly)
torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-4)
```

**Dropout** — applies after the activation of Dense (or Conv) layers, *not* on the raw input layer (that destroys information before the network sees it) and *not* immediately before the final output layer that produces a probability/logit (that adds noise directly to the prediction you're trying to calibrate). Typical rate: 0.2-0.3 for conv layers, 0.3-0.5 for dense layers late in the network. Practical placement in a 4-layer MLP: `Input → Dense → BatchNorm → ReLU → Dropout(0.3) → Dense → BatchNorm → ReLU → Dropout(0.3) → Dense(output)`.
```python
# PyTorch block, in order
nn.Sequential(nn.Linear(256, 128), nn.BatchNorm1d(128), nn.ReLU(), nn.Dropout(0.3))
```

**Batch Normalization** — applies to Dense/Conv layer outputs, before the nonlinearity in the original paper's design (`Linear → BatchNorm → ReLU`), though many production architectures (e.g. later ResNet variants) show `Linear → ReLU → BatchNorm` works comparably. The practical default is BN-before-activation (matches the original paper and most framework examples); don't spend much tuning budget on this choice — it rarely swings accuracy more than a fraction of a point either way. BatchNorm and Dropout together needs care: putting Dropout directly before BatchNorm can create a train/inference variance mismatch ("variance shift") — if using both, put Dropout after BatchNorm+activation, or prefer only one of them per block.

**Real-world grounding:** a common interview trap is a network with `Dropout` on the input layer (destroys raw features) or right before a sigmoid output (randomly zeroing pre-sigmoid logits injects noise straight into the probability estimate) — both are placement bugs, not "regularization is bad" bugs.

## Hyperparameters and Their Role

| Hyperparameter | What it controls | First-guess rule of thumb |
|---|---|---|
| Learning rate | Step size of each weight update | `1e-3` for Adam, `1e-2` to `1e-1` for SGD+momentum; too high → loss oscillates/diverges/NaNs, too low → loss decreases but painfully slowly and may get stuck in a sharp local minimum |
| Batch size | Gradient noise / GPU utilization / memory | 32-256 for most problems; larger batches need a proportionally higher learning rate (linear scaling rule: doubling batch size → roughly double LR, with warmup) but tend to generalize slightly worse (large-batch generalization gap) than smaller, noisier batches |
| Epochs | How many passes over the data | Set high (e.g. 100+) and rely on early stopping rather than guessing a fixed number |
| Number of layers / width | Model capacity | Start shallow/narrow, grow only if underfitting (see capacity-then-regularize loop above) |
| Optimizer | How gradients update weights | Adam/AdamW by default; SGD+momentum with a schedule for final squeezed-out accuracy on well-studied problems (e.g. ImageNet) |
| Weight initialization | Starting variance of activations/gradients | He init with ReLU family, Xavier/Glorot with tanh/sigmoid |
| Dropout rate | Fraction of units zeroed for regularization | 0.2-0.3 typical, up to 0.5 for heavily overfitting dense layers |
| Weight decay strength | L2 penalty on weight magnitude | `1e-4` to `1e-2`, start at `1e-4` |
| Gradient clipping threshold | Caps gradient norm to prevent explosions | Norm clip at 1.0-5.0, standard for RNNs/Transformers and any deep/recurrent net |

**Real-world grounding:** the single highest-leverage hyperparameter to tune first is almost always learning rate — a wrong LR can make every other choice (architecture, regularization) look broken when it isn't. Tools like Keras Tuner or Optuna automate this search once you've bounded reasonable ranges (see `references.md`).

## Loss Functions — a Working Catalog

| Loss | Task | One real use case |
|---|---|---|
| MSE | Regression | House price prediction with roughly normal, non-outlier-heavy errors |
| MAE | Regression, outlier-robust | Delivery-time estimation where a few extreme delays shouldn't dominate the gradient |
| Huber | Regression, outlier-heavy targets | Sensor/financial data with occasional large spikes — behaves like MSE near zero (smooth gradient) and like MAE for large errors (bounded gradient), so outliers don't dominate training the way plain MSE's squared term would |
| Binary cross-entropy | Binary classification | Fraud/not-fraud scoring with a sigmoid output |
| Categorical cross-entropy | Multi-class, one-hot labels | 1000-class image classifier with labels as one-hot vectors |
| Sparse categorical cross-entropy | Multi-class, integer labels | Same task, but labels stored as class indices (`3`) instead of one-hot (`[0,0,0,1,...]`) — purely a label-encoding convenience, mathematically identical loss |
| Focal loss | Extreme class imbalance | Object detection (RetinaNet) where background patches outnumber real objects ~1000:1 — down-weights easy, well-classified negatives so the loss isn't swamped by them, forcing the model to focus on hard/rare positives |
| KL divergence | Distribution matching | VAE latent regularization (matching the encoder's distribution to a prior), or knowledge distillation (student mimics teacher's softened output distribution) |
| Triplet / contrastive loss | Embedding / similarity learning | Face verification — pulls same-identity embeddings together and pushes different-identity embeddings apart, with no fixed class count |
| Hinge loss | Margin-based classification | SVM-style binary classifiers, or margin-focused output layers where you care about a confident decision boundary more than a calibrated probability |

**Practical note:** categorical vs. sparse categorical cross-entropy is a pure implementation/label-format choice, not a modeling choice — pick whichever matches how your labels are already encoded and never one-hot-encode labels purely to "use categorical cross-entropy" when sparse is available.

## Activation Functions — a Working Catalog

| Activation | Hidden layer? | Output layer? | Failure mode | Default in |
|---|---|---|---|---|
| Sigmoid | Rare (saturates) | Yes — binary probability | Vanishing gradient (saturates both ends) | Binary classifier output |
| Tanh | Occasional (RNN gates) | No | Vanishing gradient, milder than sigmoid (zero-centered) | LSTM/GRU gate internals |
| ReLU | Yes — default for CNNs/MLPs | No | Dying ReLU (stuck at 0, zero gradient forever) | CNNs (ResNet, VGG) |
| Leaky ReLU / PReLU | Yes, when dying ReLU is observed | No | Avoids dying ReLU via small negative slope; PReLU learns the slope | GANs (avoids sparse gradients) |
| ELU | Yes, alternative to Leaky ReLU | No | Smooth negative saturation reduces bias shift, costlier to compute (exp) | Occasional CNN variants |
| GELU | Yes — default in Transformers | No | No dying-unit problem, smooth everywhere, costlier than ReLU | BERT, GPT, ViT |
| Swish / SiLU | Yes, alternative to GELU | No | Similar profile to GELU, smooth and non-monotonic near 0 | EfficientNet |
| Softmax | No | Yes — multi-class probability distribution | N/A (not used in hidden layers — collapses information by construction) | Multi-class classifier output |

**Real-world grounding:** GELU's dominance in Transformers (BERT, GPT) over ReLU is a case where a fractionally more expensive activation function's smoothness measurably improves optimization at very large parameter/data scale — a good illustration that "best activation" is somewhat scale- and architecture-dependent, not universal.

## Decision Cheat-Sheet: What to Use for Which Use Case

| Use case | Output layer + activation | Loss function | Key architecture/regularization note |
|---|---|---|---|
| Tabular binary classification | 1 unit, sigmoid | Binary cross-entropy (class-weighted or focal if imbalanced) | 2-4 hidden layers, dropout 0.2-0.3, batch norm; benchmark against gradient-boosted trees |
| Tabular regression (skewed/heavy-tailed target) | 1 unit, linear (often on log-transformed target) | Huber, or MSE on `log1p(y)` | Watch for outlier-driven gradient spikes; Huber or log-transform tames them |
| Multi-class image classification (small dataset) | N units, softmax | Categorical / sparse categorical cross-entropy | Transfer learning + GAP before the head, heavy augmentation, dropout in the head only |
| Multi-label classification | N units, independent sigmoid | Binary cross-entropy per label | Never use softmax here — it forces labels to compete for one unit of probability mass |
| Imbalanced classification (rare positive class) | 1 (or N) units, sigmoid/softmax | Focal loss or class-weighted cross-entropy | Evaluate with PR-AUC/F1, not accuracy; consider resampling in addition to loss reweighting |
| Sequence/text classification | 1 or N units, sigmoid/softmax | BCE / (sparse) categorical cross-entropy | Dropout on embeddings and recurrent/attention outputs; gradient clipping if RNN-based |
| Embedding/similarity learning | D-dim linear (often L2-normalized) | Triplet or contrastive loss | No softmax head; needs a mining strategy for hard negatives during training |
| Count-data regression (e.g. demand forecasting) | 1 unit, exp/softplus (ensures positivity) | Poisson loss (or MSE on log-target) | Poisson loss matches the discrete, non-negative, often over-dispersed nature of count data better than plain MSE |
