# Projects — Model Evaluation and Overfitting

## Small: Metric Choice Audit on an Imbalanced Dataset
Using Kaggle's "Credit Card Fraud Detection" dataset (~0.17% positive class), train a single baseline classifier and report accuracy, ROC-AUC, PR-AUC, precision, recall, and F1 side by side. Write a short memo explaining which metric you'd actually report to a non-technical stakeholder and why the others would mislead them. Proves you understand that metric choice, not just model choice, is often the real interview differentiator.

## Medium: Time-Series Forecasting Without Leakage
Build a demand or sales forecasting model (e.g., Kaggle's "Store Sales - Time Series Forecasting" or "M5 Forecasting") and deliberately compare plain shuffled k-fold CV against walk-forward (TimeSeriesSplit) CV. Quantify the gap between the two CV estimates and explain, with a concrete example row, how shuffled CV leaked future information. Proves you can catch one of the most common and damaging mistakes in real-world forecasting projects.

## Large: Overfitting Diagnosis and Remediation Pipeline
Build a reusable diagnostic pipeline that, given any tabular dataset and model, automatically produces: learning curves, train/val/test gap analysis, nested-CV performance estimate vs. naive CV estimate, and a report recommending which item from the "overfitting toolkit" (regularization, more data, feature selection, ensembling, etc.) is most likely to help, based on the diagnosed symptom. Apply it to 3 different real datasets (one small/tabular, one high-cardinality categorical, one where you simulate a small neural net) from Kaggle. Proves genuine engineering maturity — building a tool that operationalizes the exact judgment interviewers are testing for in scenario questions.
