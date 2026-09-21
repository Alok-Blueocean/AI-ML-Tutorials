# References — Regression and Regularization

## Official Docs
- [scikit-learn: Linear Models (OLS, Ridge, Lasso, Elastic-Net)](https://scikit-learn.org/stable/modules/linear_model.html) — math formulations, solvers, and code for every regression variant in this topic. Verified.
- [scikit-learn: Cross-validation (RidgeCV/LassoCV usage)](https://scikit-learn.org/stable/modules/cross_validation.html) — for tuning alpha. Verified.
- [statsmodels: Variance Inflation Factor](https://www.statsmodels.org/stable/generated/statsmodels.stats.outliers_influence.variance_inflation_factor.html) — VIF implementation used in exercises. (not URL-verified this session)

## Key Papers
- Tibshirani, R. (1996). *Regression Shrinkage and Selection via the Lasso*. Journal of the Royal Statistical Society, Series B. The original Lasso paper. (not URL-verified this session — a stable copy is on the author's Stanford page, search "Tibshirani 1996 Lasso JRSS")
- Zou, H. & Hastie, T. (2005). *Regularization and Variable Selection via the Elastic Net*. JRSS-B. Introduces ElasticNet and formally explains the "grouping effect" that fixes Lasso's instability under correlated features. (not URL-verified this session)

## Articles / Blog Posts (interview-oriented)
- [Ridge & Lasso Regression — 30+ Interview Questions (with Follow-ups), Medium](https://medium.com/@amitmaurya5033/ridge-lasso-regression-30-interview-questions-with-follow-ups-8a13d23e0ce2) — large bank of exactly the kind of follow-up questions interviewers ask.
- [Lasso vs Ridge vs Elastic Net, GeeksforGeeks](https://www.geeksforgeeks.org/machine-learning/lasso-vs-ridge-vs-elastic-net-ml/) — quick side-by-side comparison table, good for last-minute review.
- [Lasso regression: implementation of coordinate descent (Xavier Bourret Sicotte)](https://xavierbourretsicotte.github.io/lasso_implementation.html) — walks through the exact soft-thresholding coordinate-descent derivation used in Exercise 6.
- [Statistical Horizons: When Can You Safely Ignore Multicollinearity?](https://statisticalhorizons.com/multicollinearity/) — a more nuanced, contrarian take on VIF thresholds worth knowing for interviews that probe beyond "VIF > 10 is bad."

## Book
- James, Witten, Hastie, Tibshirani. *An Introduction to Statistical Learning* (2nd ed. / Python ed.), Chapter 3 (Linear Regression) and Chapter 6 (Regularization: Ridge, Lasso). Free PDF at [statlearning.com](https://www.statlearning.com/) — confirmed official site offering free PDFs of the R and Python editions. Verified.

## GitHub Repos
- [AjNavneet/Regression-Models-Numpy-Lasso-Ridge-DT](https://github.com/AjNavneet/Regression-Models-Numpy-Lasso-Ridge-DT) — Ridge/Lasso/Decision-Tree regression implemented from scratch in NumPy; good cross-check for Exercise 4 and 6. Verified (small repo, 1 star, but content matches).
- scikit-learn source for `sklearn/linear_model/_ridge.py` and `_coordinate_descent.py` on [github.com/scikit-learn/scikit-learn](https://github.com/scikit-learn/scikit-learn) — read the actual production implementation after doing your own from scratch. (not URL-verified this session)

## Course
- Andrew Ng, [Machine Learning Specialization (Coursera / DeepLearning.AI + Stanford Online)](https://www.coursera.org/specializations/machine-learning-introduction) — Course 1 covers linear/logistic regression and regularization intuition clearly. Verified.
