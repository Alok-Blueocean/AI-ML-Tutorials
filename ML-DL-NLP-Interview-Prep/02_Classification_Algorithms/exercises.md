# Exercises — Classification Algorithms

1. **(Easy, coding)** Train `LogisticRegression` on a binary dataset (e.g., Titanic or breast cancer). Convert 2 coefficients to odds ratios and write one sentence interpreting each.
2. **(Easy, conceptual)** Explain why k-NN requires feature scaling but a decision tree does not.
3. **(Easy, coding)** Train k-NN with `k` in {1, 5, 15, 50}. Plot train vs. validation accuracy for each `k` and identify the point of overfitting/underfitting.
4. **(Medium, coding)** Fit an SVM with an RBF kernel on a 2D synthetic non-linearly-separable dataset (e.g., `make_moons`). Plot the decision boundary for 3 combinations of `C` and `gamma` and describe the visual difference.
5. **(Medium, conceptual)** Derive/explain the kernel trick: why does replacing the dot product `x·x'` with a kernel function `K(x,x')` let SVM learn nonlinear boundaries without explicitly computing the higher-dimensional mapping?
6. **(Medium, coding)** Implement Gaussian Naive Bayes from scratch in NumPy (estimate per-class mean/variance, apply Bayes' rule) and compare predictions to `sklearn.naive_bayes.GaussianNB`.
7. **(Medium, coding)** Train a decision tree with `criterion='gini'` and `criterion='entropy'` on the same dataset. Compare resulting tree structure and accuracy — are they meaningfully different?
8. **(Medium, coding)** Grow an unpruned decision tree (`max_depth=None`) and a pruned one (tune `ccp_alpha` via cross-validation). Compare train vs. test accuracy for both.
9. **(Medium, conceptual)** Explain the difference between One-vs-Rest and One-vs-One multi-class strategies and when you'd prefer each, including their computational cost tradeoffs for a 20-class problem.
10. **(Hard, coding)** Train an SVM classifier, then check calibration with a reliability diagram (`sklearn.calibration.calibration_curve`). Apply Platt scaling (`CalibratedClassifierCV`) and re-plot — quantify the improvement with Brier score.
11. **(Hard, coding)** Build a spam classifier with Multinomial Naive Bayes on raw text (bag-of-words/TF-IDF). Compare accuracy and training time against a linear SVM on the same features.
12. **(Hard, conceptual)** A dataset has 2 classes separable except for 3 mislabeled outlier points deep inside the "wrong" class region. Explain how logistic regression vs. hard-margin vs. soft-margin SVM would each handle these outliers differently.
13. **(Hard, coding)** Implement logistic regression's gradient descent update from scratch (log-loss gradient) in NumPy, and verify convergence to the same coefficients as `sklearn.linear_model.LogisticRegression` (with no regularization, `penalty=None`).
14. **(Hard, conceptual)** Explain why a model with 99% test accuracy can still be poorly calibrated, and describe a real-world case (e.g., pricing, medical risk) where calibration matters more than raw discriminative accuracy.
