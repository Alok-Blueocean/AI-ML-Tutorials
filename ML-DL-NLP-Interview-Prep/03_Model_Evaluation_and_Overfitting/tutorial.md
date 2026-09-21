# Model Evaluation and Overfitting — Study Notes

## Bias-Variance Tradeoff
Total error decomposes as `Bias² + Variance + Irreducible Error`.
- **High bias (underfitting)**: model too simple to capture the pattern — poor on both train and test.
- **High variance (overfitting)**: model too flexible, memorizes training noise — great on train, poor on test.
- Increasing model complexity generally decreases bias but increases variance — you can't minimize both without more data or better regularization.
- *Example*: predicting demand with a single average (high bias, ignores seasonality) vs. a 50-layer neural net on 200 rows of sales data (high variance, memorizes noise). The sweet spot might be a simple gradient-boosted model with 5-10 well-chosen features.

## Train/Val/Test Splits
- **Train**: fit parameters. **Validation**: tune hyperparameters and pick between models. **Test**: final, untouched estimate of real-world performance — touch it once.
- Leaking test data into any tuning decision ("peeking") makes your reported metric optimistic and is one of the most common real-world causes of "it worked in the notebook, failed in production."
- *Example*: a Kaggle competition's public vs. private leaderboard split exists specifically to catch teams that overfit to the public validation set through repeated submissions.

## Cross-Validation
- **k-fold CV**: split data into k folds, train on k-1, validate on the held-out fold, rotate, average the metric. Reduces variance in the performance estimate vs. a single train/val split — important with small datasets.
- **Stratified k-fold**: preserves class proportions in each fold — essential for imbalanced classification (without it, a fold might contain zero minority-class examples).
- **Time-series CV (walk-forward / rolling-origin)**: never validate on data that precedes training data in time. Fold 1 trains on months 1-3, validates on month 4; fold 2 trains on months 1-4, validates on month 5, etc. Standard k-fold shuffling leaks future information into training — a common, damaging mistake in forecasting interviews.
- **Nested CV**: an outer loop estimates generalization performance; an inner loop (within each outer training fold) tunes hyperparameters. Prevents the "used the same fold to tune AND report performance" leakage that inflates reported metrics when doing plain grid search + single test set.
- *Example*: a retail demand-forecasting model validated with ordinary k-fold looked great offline (CV R²=0.91) but failed in production — because shuffled folds let the model "see" post-holiday data while training on pre-holiday data. Walk-forward CV would have caught this before ship.

## Classification Metrics
- **Confusion matrix**: TP, FP, FN, TN — the source of truth every other metric derives from.
- **Precision** = TP/(TP+FP) — "of what I flagged positive, how much was right." Matters when false positives are costly (e.g., blocking legitimate transactions).
- **Recall** = TP/(TP+FN) — "of all actual positives, how many did I catch." Matters when false negatives are costly (e.g., missing a cancer diagnosis or fraud case).
- **F1** = harmonic mean of precision/recall — use when you need one number balancing both, especially under class imbalance.
- **ROC-AUC**: probability a random positive is ranked above a random negative. Threshold-independent, but **can look artificially good on imbalanced data** because it's insensitive to class ratio.
- **PR-AUC**: precision-recall curve area. More informative than ROC-AUC on imbalanced data because it's sensitive to false positives on the rare class; the random baseline for PR-AUC equals the positive-class prevalence, not a fixed 0.5.
- *Example*: for a 1%-prevalence fraud dataset, a model can post ROC-AUC=0.95 (looks great) while PR-AUC is only 0.30 (mediocre) — report PR-AUC to leadership, not ROC-AUC, or you'll oversell the model.

## Regression Metrics
- **RMSE**: penalizes large errors heavily (squared) — sensitive to outliers.
- **MAE**: linear penalty, more robust to outliers, more interpretable ("average error in dollars").
- **MAPE**: percentage error — intuitive for stakeholders ("off by 8% on average") but breaks down / blows up when actual values are near zero.
- *Example*: forecasting SaaS revenue, MAPE is popular with finance ("we're within 5%"), but if some months have near-zero revenue (new product line), MAPE becomes unstable — use MAE or a symmetric variant (sMAPE) instead.

## Learning Curves
Plot train and validation error vs. training-set size (or vs. epochs for iterative models).
- **Gap between train and val error that isn't closing** → high variance (overfitting) → more data or regularization helps.
- **Both curves converge to a high error plateau** → high bias (underfitting) → need a more expressive model or better features; more data won't help much.
- *Example*: if validation loss is still dropping when you cut off training at 10 epochs "to save GPU time," the learning curve tells you you're leaving accuracy on the table — not overfitting yet.

## Hyperparameter Tuning
- **Grid search**: exhaustive over a specified parameter grid — reliable but expensive; wasteful when many hyperparameters barely matter.
- **Random search**: samples the space randomly — often finds a comparably good result far faster because typically only a few hyperparameters actually matter, and random search covers a wider range of those.
- **Bayesian optimization** (e.g., Optuna, Hyperopt): builds a surrogate model of the objective and picks the next point to try based on expected improvement — efficient for expensive-to-train models (e.g., deep nets, large boosted ensembles).
- Always tune on a validation set / inner CV loop, never on the test set.

## The Full Toolkit for Reducing Overfitting
This is the single most commonly asked practical ML interview topic. Know all of these cold, and know which to reach for first given the situation:

1. **Get more data** — the single most effective fix when feasible; more data directly reduces variance.
2. **Data augmentation** — synthetically expand training data (image flips/crops/rotations; text back-translation/synonym swap; SMOTE for tabular minority classes) when collecting more real data is expensive.
3. **Regularization (L1/L2/ElasticNet)** — penalize large weights; L1 also does feature selection.
4. **Dropout** (neural nets) — randomly zero activations during training, forcing redundant, more robust representations; acts like an implicit ensemble.
5. **Early stopping** — halt training when validation loss stops improving, even though training loss keeps dropping.
6. **Simplify the model** — fewer layers/trees/depth/parameters; matches capacity to the amount of data available.
7. **Cross-validation** — doesn't reduce overfitting directly but reliably *detects* it and guides hyperparameter choices that do.
8. **Ensembling** (bagging especially) — averaging multiple high-variance models cancels out uncorrelated errors, reducing variance without much bias cost.
9. **Feature selection / dimensionality reduction** — remove noisy or redundant features that give the model room to memorize noise.
10. **Pruning** (trees) — remove branches that don't generalize; the tree-specific analogue of regularization.
11. **Batch normalization** (neural nets) — stabilizes training and has a mild regularizing side-effect.
12. **Label smoothing / noise injection** — prevents the model from becoming overconfident on training labels.

## Scenario Heuristics (map symptom → most likely first fix)
| Symptom | First things to check/try |
|---|---|
| Perfect train, poor test, small tabular dataset | Simpler model, regularization, cross-validation, more data if possible |
| Deep net: train loss ↓, val loss ↑ after epoch N | Early stopping, dropout, data augmentation, weight decay |
| Great CV score, poor production performance | Check for train/test leakage, distribution shift, wrong CV strategy (e.g., time series shuffled) |
| High-cardinality categorical feature added → overfitting spikes | Target encoding with smoothing/out-of-fold encoding, hashing trick, embeddings, or drop rare categories into "other" |
