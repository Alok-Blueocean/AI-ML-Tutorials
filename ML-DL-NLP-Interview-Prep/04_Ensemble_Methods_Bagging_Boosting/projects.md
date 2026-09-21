# Projects — Ensemble Methods: Bagging and Boosting

## Small: Random Forest vs. Gradient Boosting Bakeoff with OOB Validation
Using the Kaggle "Titanic" or "Adult Income" dataset, train a Random Forest (using OOB score for validation, no separate holdout needed) alongside an XGBoost model (using standard train/val split). Compare accuracy, training time, and how each ranks features by importance (gain vs. permutation importance). Proves you understand the practical bagging-vs-boosting tradeoff and OOB's role as a "free" validation mechanism.

## Medium: Leakage-Free High-Cardinality Categorical Pipeline
Using a dataset with several high-cardinality categorical columns (e.g., Kaggle's "Amazon Employee Access Challenge" or a real estate dataset with city/neighborhood columns), build three pipelines: (1) XGBoost with one-hot encoding, (2) XGBoost with naive target encoding, (3) CatBoost with native categorical handling. Measure train/test AUC gap for each and show which pipeline avoids the target-leakage overfitting trap. Proves hands-on understanding of ordered boosting's practical value, not just its theory.

## Large: Stacked Ensemble with SHAP-Driven Feature Audit for a Kaggle Competition
Pick an active or past Kaggle tabular competition (e.g., a churn, credit-risk, or click-through-rate dataset). Build individual XGBoost, LightGBM, and CatBoost models, tune each with Bayesian optimization (Optuna), then stack them with a logistic regression meta-model trained on out-of-fold predictions. Run a SHAP analysis on the best individual model to identify and justify removing any suspicious high-cardinality or leakage-prone features before finalizing. Proves full competitive-ML-engineer competency: model diversity, proper stacking (no leakage), and explainability-driven feature vetting.
