# Classification Algorithms — Study Notes

## Logistic Regression (Sigmoid / Log-Odds)
Models `log(p/(1-p)) = Xw + b`, i.e., a linear model on the **log-odds**, squashed through the sigmoid `σ(z) = 1/(1+e^-z)` to get a probability in [0,1]. Trained by maximizing log-likelihood (equivalent to minimizing log-loss/cross-entropy).
- Coefficients are directly interpretable: `e^w_i` = the multiplicative change in odds per unit increase in feature `i`.
- *Example*: a bank's credit-default model reports "each additional missed payment multiplies default odds by 2.3x" — a sentence a risk committee can act on directly, unlike a black-box score.

## Decision Boundaries
Logistic regression, linear SVM, and Naive Bayes (with Gaussian, equal-variance assumptions) all produce **linear** decision boundaries. Decision trees produce **axis-aligned rectangular** boundaries. k-NN and RBF-kernel SVM produce **flexible, nonlinear** boundaries that hug the data.
- *Example*: fraud rings often use non-linear combinations of behavior (e.g., "small amount BUT unusual hour BUT new device") — a linear model misses this interaction unless you engineer it explicitly; a tree-based model finds it automatically.

## k-Nearest Neighbors (k-NN)
Lazy learner: no training phase, just stores data and predicts by majority vote (or average) among the `k` closest points at inference time.
- Sensitive to feature scale (always standardize) and the curse of dimensionality (distance becomes less meaningful in high dimensions).
- Small `k` → low bias/high variance (overfits to noise); large `k` → high bias/low variance (oversmooths).
- *Example*: a "similar products" recommendation widget on an e-commerce site is essentially k-NN in embedding space — no training needed, just fast nearest-neighbor lookup (often via approximate methods like FAISS/HNSW at scale).

## Support Vector Machines (SVM)
Finds the hyperplane that maximizes the **margin** between classes, using only the closest points (**support vectors**) — robust to points far from the boundary.
- **Kernel trick**: implicitly maps data into a higher-dimensional space (via a kernel function like RBF or polynomial) without ever computing the transformation explicitly, so non-linearly-separable data becomes separable.
- **C** (regularization/inverse penalty on misclassification): low C = wider margin, tolerates more misclassification (more bias, less variance); high C = narrow margin, fits training data tightly (less bias, more variance — risk of overfitting).
- **gamma** (RBF kernel): controls how far a single training point's influence reaches. High gamma = influence is very local → wiggly, overfit boundary; low gamma = smoother, more global boundary.
- *Example*: text classification (spam vs. not-spam) with linear SVM on TF-IDF features — high-dimensional, sparse data where linear SVMs are a strong, fast, well-calibrated-enough baseline before reaching for deep learning.

## Naive Bayes
Applies Bayes' theorem with a (naive) assumption that features are conditionally independent given the class. Despite the unrealistic independence assumption, it's fast, needs little data, and works surprisingly well for text.
- Variants: Gaussian (continuous features), Multinomial (word counts), Bernoulli (binary presence/absence).
- *Example*: a classic spam filter using word-frequency features — even though "free" and "money" aren't truly independent given spam, Multinomial Naive Bayes is fast enough to run inline on every incoming email and performs competitively.

## Decision Trees: Gini vs Entropy, Pruning
- **Gini impurity**: `1 - Σp_i²` — measures probability of misclassifying a randomly chosen element if labeled by class distribution. Cheaper to compute (no log).
- **Entropy / Information Gain**: `-Σp_i log2(p_i)` — from information theory; sometimes yields marginally more balanced trees. In practice, Gini vs. entropy rarely changes results much.
- Trees overfit easily (a fully grown tree can memorize training data). **Pruning** controls this:
  - *Pre-pruning*: `max_depth`, `min_samples_leaf`, `min_samples_split`, `max_leaf_nodes` set before/during training.
  - *Post-pruning*: grow a full tree, then use cost-complexity pruning (`ccp_alpha` in sklearn) to remove branches that don't improve validation performance.
- *Example*: a loan-approval tree that's unpruned might create a rule like "approve if income=$52,341 AND age=34 AND zip=10001" — an artifact of one training row, not a real pattern. Pruning collapses this back to sensible, generalizable rules.

## Multi-Class Strategies
- **One-vs-Rest (OvR)**: train one binary classifier per class vs. all others; predict the class with highest confidence. Simple, works with any binary classifier, but classifiers are trained on imbalanced setups.
- **One-vs-One (OvO)**: train a classifier for every pair of classes (`k(k-1)/2` classifiers); vote. More classifiers but each is trained on a more balanced, easier sub-problem. SVM in sklearn defaults to OvO.
- **Native multi-class**: softmax regression, decision trees, and boosting handle multi-class directly without decomposition.
- *Example*: a customer support ticket router with 50 categories — OvR would need 50 classifiers, each learning "category X vs. everything else" which gets imbalanced fast; a native softmax or gradient-boosted multi-class model is usually cleaner at this scale.

## Probability Calibration
A model's raw output isn't always a trustworthy probability. SVM decision scores and margin-based scores are notoriously poorly calibrated; even good discriminators (high AUC) can be miscalibrated (e.g., always predicting 0.9 for anything above the boundary).
- **Platt scaling**: fits a logistic regression on top of the raw scores.
- **Isotonic regression**: fits a non-parametric, monotonic mapping — more flexible, needs more data.
- Check calibration with a **reliability diagram** (predicted probability bucket vs. observed frequency) and **Brier score**.
- *Example*: a medical risk model outputting "73% chance of readmission" must actually mean ~73% of such patients get readmitted, not just rank patients correctly — clinicians and dosing/resource decisions depend on the number itself, not just the ranking, so calibration is a hard requirement before deployment, not a nice-to-have.

## Quick Reference
| Algorithm | Boundary shape | Needs scaling? | Handles high-dim sparse data well? |
|---|---|---|---|
| Logistic Regression | Linear | Yes | Yes |
| k-NN | Arbitrary (local) | Yes (critical) | No (curse of dimensionality) |
| SVM (linear) | Linear | Yes | Yes |
| SVM (RBF) | Nonlinear | Yes | Moderate |
| Naive Bayes | Linear-ish | No | Yes (text) |
| Decision Tree | Axis-aligned steps | No | Poor alone, great as ensemble base learner |
