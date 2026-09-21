# Ensemble Methods: Bagging and Boosting — Study Notes

## Bagging and Random Forest
**Bagging** (Bootstrap Aggregating): train many high-variance base learners (usually deep trees) independently, each on a bootstrap sample (sampling with replacement) of the training data, then average (regression) or vote (classification). Reduces variance because averaging uncorrelated errors cancels them out; bias stays roughly the same as a single tree.
- **Random Forest** adds one more decorrelation trick: at each split, only a random subset of features is considered (`max_features`), not all of them. This prevents every tree from making the same dominant-feature split, making trees less correlated with each other, which improves the variance-reduction benefit of averaging.
- **Out-of-bag (OOB) error**: each bootstrap sample leaves out ~37% of rows on average; those left-out rows form a free validation set per tree. Averaging each row's prediction over the trees that didn't see it gives an unbiased performance estimate **without needing a separate holdout set**.
- *Example*: a Random Forest for credit scoring trained on 500 features (many redundant, correlated bureau variables) — feature subsampling stops the model from being dominated by one or two proxy variables and produces a more robust, less overfit score.

## Boosting
Builds models **sequentially**, where each new model focuses on correcting the errors of the ensemble so far — reduces both bias and variance, but has more risk of overfitting than bagging if run too long or too aggressively.
- **AdaBoost**: reweights training examples after each round — misclassified points get higher weight so the next weak learner (often depth-1 "stumps") focuses on them. Final prediction is a weighted vote of all weak learners.
- **Gradient Boosting**: instead of reweighting samples, each new tree is fit to the **negative gradient (residual)** of the loss function with respect to current predictions — a general framework that works for any differentiable loss (not just exponential loss like AdaBoost).
- *Example*: AdaBoost's classic use case — face detection cascades (Viola-Jones) chain simple weak classifiers (rectangle features) that each catch what previous stages missed, run extremely fast per stage.

## XGBoost / LightGBM / CatBoost Internals
All three are gradient boosting implementations optimized for speed, regularization, and scale — but with different core tricks:

### XGBoost
- **Regularized objective**: `obj = Σ loss(y_i, ŷ_i) + Σ Ω(f_k)`, where `Ω(f) = γT + ½λΣw_j²` (T = number of leaves, w_j = leaf weights) — explicitly penalizes tree complexity, unlike classic gradient boosting.
- **Second-order gradients**: uses both the gradient (`g_i`) and Hessian (`h_i`, second derivative) of the loss via a Taylor expansion, giving a more accurate approximation of the optimal leaf weight and split gain than first-order-only methods — this is XGBoost's key theoretical contribution over earlier GBMs.
- Grows trees **level-wise** (depth-first, balanced) by default.
- *Example*: XGBoost's regularized, second-order objective is why it consistently won Kaggle tabular competitions circa 2015-2018 — it converges faster and generalizes better than plain gradient boosting on the same data.

### LightGBM
- **Histogram-based splitting**: buckets continuous features into discrete bins before finding splits — dramatically faster and lower memory than scanning every possible split point.
- **Leaf-wise (best-first) growth**: always splits the leaf with the highest loss reduction, regardless of depth — produces deeper, less balanced but often more accurate trees for the same leaf budget, at higher overfitting risk on small data (control with `num_leaves`, `min_data_in_leaf`).
- **GOSS (Gradient-based One-Side Sampling)**: keeps all high-gradient (under-trained, informative) samples but only randomly samples low-gradient ones, speeding up training with minimal accuracy loss.
- **EFB (Exclusive Feature Bundling)**: bundles mutually-exclusive sparse features (e.g., one-hot columns) into one feature, reducing dimensionality without losing information.
- *Example*: LightGBM is the default choice for click-through-rate models with hundreds of millions of rows, where XGBoost's histogram is still slower and memory-heavier at that scale.

### CatBoost
- **Ordered boosting**: computes gradients using a model trained only on data *before* the current point in a random permutation, avoiding the "target leakage" that comes from using a data point's own label (via other features) to compute its own gradient.
- **Native categorical handling**: converts categoricals to numeric via **ordered target statistics** — similar spirit to target encoding but computed causally (only using preceding rows in a random permutation) to avoid leakage, so you skip manual one-hot/target-encoding entirely.
- *Example*: CatBoost is popular for e-commerce/ad datasets with many raw categorical columns (brand, category, city) where manual encoding is tedious and leak-prone — CatBoost handles it out of the box with less overfitting than naive target encoding.

### Level-wise vs Leaf-wise, side by side
| | XGBoost (default) | LightGBM (default) |
|---|---|---|
| Growth | Level-wise (depth-first, balanced) | Leaf-wise (best-first, unbalanced) |
| Speed on large data | Slower (histogram approx available too) | Faster |
| Overfitting risk | Lower per split | Higher — needs `num_leaves`/`min_data_in_leaf` tuning |

## Stacking / Blending
- **Stacking**: train several diverse base models (level-0), then train a meta-model (level-1, often simple logistic/linear regression) on the base models' **out-of-fold predictions** to learn how to best combine them. Must use out-of-fold predictions for training the meta-model, or you leak information and overfit.
- **Blending**: simpler variant — hold out a single validation set, train base models on the rest, then fit the meta-model on base models' predictions on that one holdout set. Less data-efficient than stacking but simpler and faster to implement (popularized in Kaggle competitions like the Netflix Prize).
- *Example*: a Kaggle-winning ensemble stacking XGBoost + LightGBM + a neural net + a linear model — none individually best, but the stacked combination generalizes better because their errors are only weakly correlated.

## Feature Importance
- **Gain-based (split-gain)**: default in XGBoost/LightGBM — total loss reduction attributable to splits on a feature. Biased toward high-cardinality features that get more split opportunities.
- **Permutation importance**: shuffle one feature's values, measure the drop in model performance — model-agnostic, reflects real predictive contribution on held-out data, more trustworthy than gain but more expensive to compute.
- **SHAP (SHapley Additive exPlanations)**: game-theoretic, distributes a prediction's deviation from the average fairly among features, gives both global importance and per-prediction, per-feature attribution (with direction: pushed prediction up or down).
- *Example*: a churn model's gain-importance ranked `customer_id`-derived hash features highly (an artifact of high cardinality and overfitting), but permutation importance and SHAP correctly showed they contributed almost nothing to genuine out-of-sample predictive power — gain importance alone would have misled the team into keeping a spurious feature.

## Quick Reference
| Method | Reduces | Trains | Key hyperparameters |
|---|---|---|---|
| Bagging / Random Forest | Variance | Parallel | n_estimators, max_features, max_depth |
| AdaBoost | Bias (mainly) | Sequential | n_estimators, learning_rate |
| Gradient Boosting / XGBoost | Bias & variance | Sequential | n_estimators, learning_rate, max_depth, reg_alpha/lambda |
| LightGBM | Bias & variance | Sequential | num_leaves, min_data_in_leaf, learning_rate |
| Stacking | Bias & variance (via diversity) | Parallel base + sequential meta | choice of base models, meta-model |
