# Projects — Regression and Regularization

## Small: Interpretable House Price Model
Build a regression model on the Ames Housing dataset (or Kaggle's "House Prices - Advanced Regression Techniques") that predicts sale price from ~80 features. Diagnose multicollinearity with VIF, fix it, then compare plain OLS vs. Ridge vs. Lasso, reporting which features Lasso zeroes out and why that's defensible to a stakeholder. Proves you can take a real messy tabular dataset from raw features to a defensible, interpretable model.

## Medium: Marketing Mix / Media Spend Attribution
Using a realistic marketing dataset (e.g., a simulated multi-channel spend dataset, or Kaggle's "Advertising" dataset extended with synthetic correlated channels), build an ElasticNet model that attributes revenue to ad channels despite heavy channel-to-channel correlation (TV vs. digital retargeting often move together). Validate coefficient stability via bootstrap resampling and present a "safe-to-cut" channel list with confidence caveats. Proves you understand regularization's real business use — media mix modeling is a widely used, high-value application at ad-tech and CPG companies.

## Large: End-to-End Regularized Pricing Engine with Drift Monitoring
Build a full pipeline (ingestion → feature engineering → polynomial + regularized regression → API) that predicts a continuous business metric (e.g., ride-hailing fare, insurance premium, or SaaS churn-adjusted LTV) using a public dataset like NYC TLC trip data or Kaggle's insurance datasets. Include: automatic alpha tuning via nested CV, residual diagnostics dashboards, and a drift check that recomputes VIF/coefficient stability weekly and flags when retraining is needed. Proves production-level MLOps thinking layered on top of a classical model — the kind of "boring but reliable" system most companies actually run in production.
