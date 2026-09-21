# Data Preprocessing & Feature Engineering

Brief primer — supporting skill for every modeling topic rather than a standalone interview deep-dive.

## Missing values

- **Mean/median/mode imputation** is the default, fast fix — median is safer than mean when a column has outliers (e.g. household income).
- **Indicator flags** (`was_missing_income = 1`) preserve the signal that a value was missing at all, which is sometimes informative on its own (e.g. a loan applicant who skipped the income field).
- **KNN imputation** fills a missing value using similar rows' values — more accurate, more expensive, easy to justify on a small-to-medium dataset.

## Outliers

- **IQR / z-score** for a quick statistical flag; **Isolation Forest** when outliers are multivariate and don't show up looking at any single column in isolation (e.g. a fraudulent transaction that looks normal on amount *and* on location, but not on the combination).

## Encoding categoricals

- **One-hot** for low-cardinality categories (e.g. `payment_method` with 4 values).
- **Target/mean encoding** for high-cardinality categories (e.g. `zip_code` with 30,000 values) — replace each category with the mean of the target for that category, but fit it only on training folds or you leak the target into your features.
- **Ordinal encoding** only when categories have a real order (`low/medium/high`) — using it on unordered categories silently tells a linear model there's a numeric relationship that doesn't exist.

## Scaling

- **Standardization (z-score)** for algorithms that assume Gaussian-ish inputs or use gradient descent (linear/logistic regression, neural nets, PCA).
- **Min-max normalization** for bounded-input needs (e.g. pixel values into a sigmoid-based layer).
- **Tree-based models (Random Forest, XGBoost) don't need scaling at all** — they split on per-feature thresholds, unaffected by monotonic transforms. This is a very common interview trick question.

## Feature construction

- **Date/time decomposition**: hour, day-of-week, is_weekend, is_holiday, plus cyclical sin/cos encodings of hour and day-of-year so December 31 and January 1 aren't treated as maximally far apart.
- **Lag / rolling-window features** for time series: a 7-day rolling average of sales is often more predictive than the raw daily number.
- **Binning / polynomial features** to let a linear model approximate non-linear relationships without switching model families.

## Imbalanced data

- Real fraud, churn, and disease-detection datasets are rarely balanced (often 1-5% positive class).
- Fix at the metric level first (precision/recall/PR-AUC, not accuracy), then consider **SMOTE** (synthetic minority oversampling), class-weighted loss, or threshold tuning — resampling the training set is a last resort, not the first lever.

## Data leakage — the single most interview-relevant idea here

Leakage means information from outside the legitimate training window sneaks into a feature. Classic real example: building a churn model and including "days since last support ticket" when a spike in support tickets is *itself* what triggers the cancellation flow — the feature is really encoding the outcome. Always ask: *would this value be known at prediction time, before the outcome happens?* If not, it's leakage.

## Quick self-check

Given a raw e-commerce transactions table, can you name (a) one leakage risk, (b) how you'd encode a `product_category` column with 500 values, and (c) whether a Random Forest on this data needs scaling? If yes, you're ready for `04_Ensemble_Methods_Bagging_Boosting`.
