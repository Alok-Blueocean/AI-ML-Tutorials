# Exercises — Regression and Regularization

Ordered easy → hard. Mix of coding and conceptual/derivation tasks.

1. **(Easy, coding)** Fit `sklearn.linear_model.LinearRegression` on the Boston/California housing dataset (or any tabular dataset). Print coefficients and R². Interpret two coefficients in plain business language.
2. **(Easy, conceptual)** List the 5 OLS assumptions and, for each, name one diagnostic plot or test that would reveal a violation.
3. **(Easy, coding)** Compute VIF for every feature in a dataset with `statsmodels.stats.outliers_influence.variance_inflation_factor`. Identify which features you'd drop or combine.
4. **(Medium, coding)** Implement Ridge regression's **closed-form solution** in NumPy: `w = (XᵀX + αI)⁻¹Xᵀy`. Compare coefficients and predictions against `sklearn.linear_model.Ridge` for 3 values of alpha.
5. **(Medium, conceptual)** Derive why Lasso induces sparsity **geometrically** (constraint region diamond vs. sphere intersecting elliptical loss contours). Sketch it in words: why does the diamond's corner get hit more often than the sphere's boundary?
6. **(Medium, coding)** Implement Lasso via **coordinate descent** from scratch (soft-thresholding update per coefficient). Verify it matches `sklearn.linear_model.Lasso` on a small synthetic dataset.
7. **(Medium, coding)** Generate synthetic data with 3 clusters of highly correlated features (e.g., duplicate a feature with small noise 5x). Fit OLS, Ridge, Lasso, and ElasticNet. Compare coefficient stability across 10 bootstrap resamples — which method gives the most stable coefficients?
8. **(Medium, conceptual)** Explain why you must standardize features before applying L1/L2 penalties, but not before plain OLS.
9. **(Medium, coding)** Use `RidgeCV`, `LassoCV`, and `ElasticNetCV` with 5-fold CV on the same dataset. Plot the regularization path (coefficients vs. alpha) for Lasso using `lasso_path`.
10. **(Hard, coding)** Build a polynomial regression pipeline (`PolynomialFeatures` + `Ridge`) and plot train/validation error vs. polynomial degree (1 to 10). Identify the degree where overfitting begins and explain using the bias-variance lens.
11. **(Hard, conceptual)** Derive the Ridge closed-form solution from the regularized loss `||y - Xw||² + α||w||²` by taking the gradient and setting it to zero.
12. **(Hard, coding)** Simulate heteroscedastic data (error variance scales with `x`). Fit OLS, plot residuals vs. fitted values to confirm the funnel shape, then fix it with a log-transform or weighted least squares and re-check the residual plot.
13. **(Hard, conceptual)** Explain, with an example, a scenario where Adjusted R² and cross-validated R² would disagree about which of two models is better, and why cross-validation is the more trustworthy signal in production.
14. **(Hard, coding)** Implement gradient descent (batch, mini-batch, and SGD) for linear regression from scratch and plot their loss curves on the same dataset. Discuss convergence speed and noise tradeoffs.
15. **(Hard, conceptual)** A colleague says "our Lasso model dropped `marketing_channel_7` entirely, so it has zero business value — let's stop spending there." What's wrong with that conclusion, and how would you validate it before acting?
