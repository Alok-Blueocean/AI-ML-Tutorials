# Projects — Classification Algorithms

## Small: Multi-Model Spam/Sentiment Classifier Bakeoff
Using a public text dataset (SMS Spam Collection or IMDB sentiment), build and compare Multinomial Naive Bayes, linear SVM, and logistic regression on TF-IDF features. Report accuracy, F1, training time, and inference latency. Proves you understand the practical tradeoffs (speed vs. accuracy vs. interpretability) between classic text-classification baselines that still beat deep learning on many small/latency-sensitive production tasks.

## Medium: Calibrated Risk Scoring for Loan Default
Using the Kaggle "Give Me Some Credit" or LendingClub loan dataset, train logistic regression, SVM, and a decision tree to predict default. Evaluate not just AUC but **calibration** (reliability diagrams, Brier score) since a risk score of "20% chance of default" must mean something real to a credit committee. Apply Platt scaling/isotonic regression to the SVM and show the calibration improvement. Proves you understand that discriminative power and calibrated probabilities are different requirements — a distinction many candidates miss.

## Large: Fraud Detection with Nonlinear Decision Boundaries and Explainability
Using a public fraud dataset (e.g., Kaggle's "Credit Card Fraud Detection," heavily imbalanced), build a pipeline comparing linear models, RBF-kernel SVM, and decision trees, tuned via grid search over C/gamma or max_depth. Handle severe class imbalance properly (class weights, PR-AUC as the primary metric, not accuracy). Add a k-NN-based "similar past fraud cases" lookup as an investigator-facing explainability tool. Proves end-to-end thinking on a real high-stakes imbalanced classification problem: model choice, boundary shape, calibration, and interpretability for non-ML stakeholders (fraud analysts).
