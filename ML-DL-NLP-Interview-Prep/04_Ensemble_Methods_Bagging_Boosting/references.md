# References — Ensemble Methods: Bagging and Boosting

## Official Docs
- [scikit-learn: Ensembles — Gradient boosting, random forests, bagging, voting, stacking](https://scikit-learn.org/stable/modules/ensemble.html) — covers all classic ensemble methods with math and code in one page. Verified.
- [XGBoost: Introduction to Boosted Trees](https://xgboost.readthedocs.io/en/stable/tutorials/model.html) — derives the regularized objective and the second-order (gradient+Hessian) structure-score formula directly. Verified.
- [LightGBM: Features](https://lightgbm.readthedocs.io/en/latest/Features.html) — histogram-based splitting and leaf-wise tree growth explained (note: this specific page doesn't detail GOSS/EFB — see the paper below for those). Verified.
- [CatBoost: Transforming categorical features to numerical features](https://catboost.ai/docs/en/concepts/algorithm-main-stages_cat-to-numberic) — explains ordered target statistics (the CTR formula) used for native categorical handling. Verified.

## Key Papers
- Chen, T. & Guestrin, C. (2016). *XGBoost: A Scalable Tree Boosting System*. [arXiv:1603.02754](https://arxiv.org/abs/1603.02754). Verified — the regularized objective and sparsity-aware split-finding described in the tutorial come straight from this paper.
- Ke, G. et al. (2017). *LightGBM: A Highly Efficient Gradient Boosting Decision Tree*. NeurIPS. Introduces GOSS and Exclusive Feature Bundling (EFB). (not URL-verified this session — search "LightGBM NeurIPS 2017 Ke" for the NeurIPS proceedings PDF)
- Prokhorenkova, L., Gusev, G., Vorobev, A., Dorogush, A.V., Gulin, A. (2017/2019). *CatBoost: Unbiased Boosting with Categorical Features*. [arXiv:1706.09516](https://arxiv.org/abs/1706.09516). Verified — introduces ordered boosting to eliminate target leakage/prediction shift.
- Lundberg, S. & Lee, S-I. (2017). *A Unified Approach to Interpreting Model Predictions*. NeurIPS. The SHAP paper. (not URL-verified this session)

## Articles / Blog Posts (interview-oriented)
- [GeeksforGeeks: GradientBoosting vs AdaBoost vs XGBoost vs CatBoost vs LightGBM](https://www.geeksforgeeks.org/machine-learning/gradientboosting-vs-adaboost-vs-xgboost-vs-catboost-vs-lightgbm/) — the exact side-by-side comparison table interviewers expect.
- [Analytics Vidhya: 30 Questions to Test a Data Scientist on Tree Based Models](https://www.analyticsvidhya.com/blog/2017/09/30-questions-test-tree-based-models/) — broad quiz covering bagging/boosting fundamentals.
- [GitHub: Devinterview-io/random-forest-interview-questions](https://github.com/Devinterview-io/random-forest-interview-questions) — 50 Random Forest interview Q&A with code. Verified (20 stars).
- [GitHub: Devinterview-io/light-gbm-interview-questions](https://github.com/Devinterview-io/light-gbm-interview-questions) — 45 LightGBM interview Q&A covering histogram methods, leaf-wise vs depth-wise growth, overfitting controls. Verified (10 stars).

## Book
- James, Witten, Hastie, Tibshirani. *An Introduction to Statistical Learning*, Chapter 8 (Tree-Based Methods: bagging, random forests, boosting). Free PDF at [statlearning.com](https://www.statlearning.com/). Verified.

## GitHub Repos (official implementations)
- [dmlc/xgboost](https://github.com/dmlc/xgboost) — official XGBoost source. Verified (28.8k stars).
- [lightgbm-org/LightGBM](https://github.com/lightgbm-org/LightGBM) — official LightGBM source (moved from `microsoft/LightGBM` in 2026; same maintainers). Verified (18.8k stars).
- [catboost/catboost](https://github.com/catboost/catboost) — official CatBoost source. Verified (9.1k stars).
- [shap/shap](https://github.com/shap/shap) — official SHAP library (works with XGBoost/LightGBM/CatBoost/sklearn/PyTorch/TF). Verified (25.8k stars).

## Course
- Andrew Ng, [Machine Learning Specialization (Coursera)](https://www.coursera.org/specializations/machine-learning-introduction) — Course 2 covers decision trees and ensembling (random forests, boosting) intuition. Verified.
