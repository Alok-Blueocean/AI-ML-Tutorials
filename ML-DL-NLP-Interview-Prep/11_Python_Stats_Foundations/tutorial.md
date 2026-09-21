# Python & Statistics Foundations

Brief primer only — this is prerequisite knowledge, not a separate deep-dive area. See `01_Regression_and_Regularization` onward for the topics that actually carry interview weight.

## NumPy / pandas fluency

- **Vectorization over loops.** `df['total'] = df['price'] * df['qty']` runs in C under the hood; a Python `for` loop over rows does not. A pricing team computing revenue across 50M order rows notices the difference immediately — vectorized pandas finishes in seconds, a row-wise loop can take hours.
- **groupby / merge / pivot** are the three operations you'll actually type daily: `df.groupby('region')['revenue'].sum()`, `pd.merge(orders, customers, on='customer_id', how='left')`, `df.pivot_table(index='month', columns='product', values='units')`.
- **Memory awareness.** Downcasting `int64` → `int32`/`category` dtypes can shrink a dataframe 3-5x — relevant the moment a dataset stops fitting comfortably in RAM.

## Probability & distributions

- **Normal distribution** underlies most classical statistical tests and the assumption behind linear regression's error term.
- **Binomial/Poisson** show up constantly in real systems: Poisson models "number of support tickets per hour"; binomial models "number of conversions out of N visitors."
- **Bayes' theorem** — the backbone of spam filters and medical-test reasoning: `P(disease | positive test) = P(positive | disease) * P(disease) / P(positive)`. A test that's 99% accurate can still mostly return false positives if the disease is rare (low prior) — this exact reasoning trips up almost everyone the first time they see it.

## Hypothesis testing

- **p-value**: probability of seeing data this extreme *if the null hypothesis were true* — not "probability the null is true." A/B test tooling (Optimizely, in-house experimentation platforms) reports this to decide whether a new checkout flow's lift is real or noise.
- **Confidence intervals** communicate uncertainty, not just a point estimate — "conversion lift is +2.1%, 95% CI [0.3%, 3.9%]" is a materially different claim than "+2.1%."
- **Type I vs Type II error**: in fraud detection, Type I (false positive) blocks a legitimate customer; Type II (false negative) lets fraud through. The threshold you pick trades one against the other — there's no error-free choice.

## Core theorems that keep coming up

- **Central Limit Theorem**: sample means tend toward a normal distribution as sample size grows, regardless of the underlying population's distribution — this is *why* confidence intervals and t-tests work even on non-normal raw data.
- **Bias-variance tradeoff**: underfitting (high bias, e.g. a straight line through curved data) vs overfitting (high variance, e.g. a tree that memorizes training noise). Nearly every model-selection decision in this course traces back to this one idea.
- **Correlation ≠ causation**: ice cream sales and drowning deaths correlate (both rise in summer) with no causal link between them. Only a randomized experiment, or a causal-inference technique (instrumental variables, difference-in-differences) when experiments aren't possible, can support a causal claim.

## Quick self-check

If you can explain *why* a 99%-accurate spam filter can still flag mostly legitimate email as spam (Bayes' theorem + base rates), and *why* k-fold cross-validation would be the wrong tool for a time-series forecasting problem (i.i.d. assumption violated — see `03_Model_Evaluation_and_Overfitting`), you have enough of this foundation to move on.
