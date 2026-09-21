# Regression and Regularization — Study Notes

## Linear & Multiple Regression
Linear regression fits `y = Xw + b` by minimizing squared error. Multiple regression just means more than one predictor column in `X`.
- **Real-world example**: A SaaS company predicts monthly recurring revenue (MRR) growth from `#sales reps`, `ad spend`, `avg deal size`. Coefficients tell finance which lever moves MRR most per dollar/hour invested.

## OLS Assumptions (and what breaks when they fail)
1. **Linearity** — the true relationship between X and y is linear in the parameters. Violation → systematic residual patterns (e.g., a curve), fixed with polynomial/log terms or a different model.
2. **Homoscedasticity** — residual variance is constant across fitted values. Violation (heteroscedasticity) → standard errors are wrong, so p-values/CIs are unreliable, even though point estimates stay unbiased.
   - *Example*: predicting house prices — error variance grows with price (a $50k miss is trivial for a $2M mansion, huge for a $100k condo). Fix: log-transform price, or use weighted least squares.
3. **No multicollinearity** — predictors aren't strongly linearly dependent. Violation → unstable, sign-flipping coefficients even though overall fit (R²) looks fine.
4. **Normally distributed residuals** — mainly matters for valid confidence intervals/hypothesis tests, not for point predictions. Check with a Q-Q plot.
5. **Independence of errors** — critical for time series (autocorrelated residuals violate this; check with Durbin-Watson).

## Gradient Descent Variants
- **Batch GD**: uses the whole dataset per step — stable but slow/memory-heavy on large data.
- **Stochastic GD (SGD)**: one sample per step — noisy but escapes shallow local minima, needed for streaming/online learning (e.g., updating a click-through-rate model in real time as new impressions arrive).
- **Mini-batch GD**: the practical default (32–512 samples) — balances stability and speed; what virtually all deep learning frameworks use.
- **Momentum / Adam**: accumulate gradient history to accelerate through flat regions and dampen oscillation in ravines — standard in modern training loops.
- Closed-form OLS (`w = (XᵀX)⁻¹Xᵀy`) is fine for small/medium, well-conditioned `X`; gradient descent (or ridge's closed form) is preferred when `XᵀX` is huge, singular, or ill-conditioned.

## Why Regularization Curbs Overfitting
All three add a penalty on coefficient size to the loss, trading a bit of bias for a large reduction in variance — critical when `p` (features) is large relative to `n` (rows) or features are correlated.

### Ridge (L2): `loss + α·Σw²`
Shrinks all coefficients smoothly toward zero but never exactly to zero. Handles multicollinearity well because it spreads weight across correlated predictors instead of arbitrarily picking one.
- *Example*: genomics with thousands of correlated gene-expression features and few samples — ridge stabilizes coefficient estimates that OLS can't even compute reliably.

### Lasso (L1): `loss + α·Σ|w|`
The L1 penalty has corners at the axes in coefficient space; the elliptical loss contours typically first touch the constraint region at a corner, which zeroes out coefficients — this is why L1 induces **sparsity** while L2 (a smooth sphere) does not.
- *Example*: a marketing-mix model with 200 correlated spend channels (TV, radio, 50 digital sub-channels) — Lasso automatically zeroes out redundant channels, leaving an interpretable shortlist for budget allocation.

### ElasticNet: `loss + α[ρ·Σ|w| + (1-ρ)/2·Σw²]`
Combines both: gets Lasso's sparsity while keeping Ridge's stability when features are highly correlated (Lasso alone tends to arbitrarily pick one of a correlated group and zero the rest, which is unstable across resamples).
- *Example*: genomic or text (TF-IDF) data with clusters of correlated features where you want sparsity but also want correlated features to survive or drop together.

### Practical notes
- Always **standardize features** before L1/L2 — penalty magnitude is scale-dependent.
- Tune `α` (and `l1_ratio` for ElasticNet) via cross-validation (`RidgeCV`, `LassoCV`, `ElasticNetCV`).
- Ridge has a closed-form solution: `w = (XᵀX + αI)⁻¹Xᵀy`. Lasso does not (non-differentiable at 0) — solved via coordinate descent or LARS.

## Multicollinearity and VIF
**Variance Inflation Factor**: `VIF_i = 1 / (1 - R_i²)`, where `R_i²` comes from regressing feature `i` on all other features.
- VIF = 1: no correlation. VIF 5–10: moderate-to-high concern. VIF > 10: strong multicollinearity — coefficient signs/magnitudes become unreliable.
- **Fixes**: drop/combine correlated features, use PCA, or switch to Ridge/ElasticNet which don't require this assumption to produce stable predictions.
- *Example*: a credit risk model with `income`, `credit_limit`, and `avg_monthly_spend` — all three move together; VIF flags this before a coefficient's sign flips sign in production and confuses the loan-approval policy team.

## Residual Diagnostics
- **Residuals vs. fitted plot**: look for random scatter around 0. A funnel shape = heteroscedasticity; a curve = missing nonlinearity.
- **Q-Q plot**: checks normality of residuals.
- **Leverage / Cook's distance**: flags influential outliers that disproportionately pull the fit line.
- *Example*: in a real-estate pricing model, one $50M mansion in a dataset of $200-500k homes can single-handedly rotate the regression line — Cook's distance catches it before it ships.

## R² vs Adjusted R²
- **R²** = fraction of variance explained; it **always increases (or stays flat)** as you add features, even useless ones — misleading for model comparison.
- **Adjusted R²** penalizes for the number of predictors: `1 - (1-R²)(n-1)/(n-p-1)`. Use it (or cross-validated R²) when comparing models with different feature counts.
- *Example*: adding "zip code as 500 dummy variables" to a housing model may bump raw R² a little just from overfitting noise — adjusted R² (or held-out RMSE) will expose that it didn't actually help.

## Polynomial Regression
Adds `x², x³, ...` terms to fit curvature while remaining a *linear* model in the (expanded) parameters — solved with the same OLS/regularization machinery.
- Risk: high-degree polynomials overfit badly and extrapolate wildly outside the training range. Almost always pair polynomial features with Ridge/Lasso.
- *Example*: modeling the relationship between ad spend and revenue with diminishing returns — a degree-2 term captures saturation; degree-8 would memorize noise and blow up predictions for spend levels never seen in training data.

## Quick Reference: When to Reach for What
| Situation | Choice |
|---|---|
| Many correlated features, want to keep all (shrink) | Ridge |
| Want automatic feature selection / sparse model | Lasso |
| Many correlated features AND want sparsity | ElasticNet |
| Curvature but no feature explosion | Polynomial + Ridge |
| p >> n (more features than rows) | Lasso/ElasticNet, or dimensionality reduction first |
