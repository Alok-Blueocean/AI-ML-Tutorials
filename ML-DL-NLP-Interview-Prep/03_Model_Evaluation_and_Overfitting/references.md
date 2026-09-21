# References — Model Evaluation and Overfitting

## Official Docs
- [scikit-learn: Model evaluation — quantifying prediction quality](https://scikit-learn.org/stable/modules/model_evaluation.html) — precision/recall/F1, ROC-AUC, average precision (PR-AUC), regression metrics, all with formulas and code. Verified.
- [scikit-learn: Cross-validation](https://scikit-learn.org/stable/modules/cross_validation.html) — KFold, StratifiedKFold, TimeSeriesSplit, and nested CV example link. Verified (checked earlier in this session).
- [Optuna documentation](https://optuna.readthedocs.io/) — Bayesian/TPE hyperparameter search, pruning callbacks. (not URL-verified this session; GitHub repo verified below)

## Key Papers
- Srivastava, N., Hinton, G., Krizhevsky, A., Sutskever, I., Salakhutdinov, R. (2014). *Dropout: A Simple Way to Prevent Neural Networks from Overfitting*. JMLR 15. [Official paper page](https://www.jmlr.org/papers/v15/srivastava14a.html). Verified via search — the canonical reference for why dropout works (implicit ensembling of "thinned" networks).
- Cawley, G. & Talbot, N. (2010). *On Over-fitting in Model Selection and Subsequent Selection Bias in Performance Evaluation*. JMLR. The classic paper motivating **nested cross-validation** to avoid optimistic bias from tuning and evaluating on the same folds. (not URL-verified this session)

## Articles / Blog Posts (interview-oriented)
- [MachineLearningMastery: ROC AUC vs Precision-Recall for Imbalanced Data](https://machinelearningmastery.com/roc-auc-vs-precision-recall-for-imbalanced-data/) by Iván Palomares Carrascosa — concrete fraud-detection example showing ROC-AUC 0.957 vs. PR-AUC 0.708 on the same model; exactly the kind of number interviewers expect you to know. Verified.
- [MachineLearningMastery: Failure of Classification Accuracy for Imbalanced Class Distributions](https://machinelearningmastery.com/failure-of-accuracy-for-imbalanced-class-distributions/) — the "accuracy paradox," directly relevant to the "98% accuracy but useless" interview question.
- [MachineLearningMastery: 8 Tactics to Combat Imbalanced Classes](https://machinelearningmastery.com/tactics-to-combat-imbalanced-classes-in-your-machine-learning-dataset/) — practical checklist (resampling, class weights, threshold moving, anomaly framing).
- [GitHub: Devinterview-io/bias-and-variance-interview-questions](https://github.com/Devinterview-io/bias-and-variance-interview-questions) — 45 bias/variance interview Q&A with code snippets. Verified (15 stars, content matches).

## Book
- James, Witten, Hastie, Tibshirani. *An Introduction to Statistical Learning*, Chapter 2 (bias-variance) and Chapter 5 (Resampling: cross-validation, bootstrap). Free PDF at [statlearning.com](https://www.statlearning.com/). Verified.
- Hastie, Tibshirani, Friedman. *The Elements of Statistical Learning*, Chapter 7 (Model Assessment and Selection) — deeper, more mathematical treatment of bias-variance and CV. (not URL-verified this session; freely available PDF is well-known via the authors' Stanford page)

## Course
- Andrew Ng, [Machine Learning Specialization (Coursera)](https://www.coursera.org/specializations/machine-learning-introduction) — "Advice for Applying Machine Learning" week covers bias/variance diagnosis and learning curves directly. Verified.

## GitHub Repos
- [optuna/optuna](https://github.com/optuna/optuna) — Bayesian hyperparameter optimization framework used in the nested-CV / Bayesian-search exercises. Verified (14.8k stars).
- scikit-learn source for `sklearn/model_selection/_search.py` (GridSearchCV/RandomizedSearchCV) and `_split.py` (TimeSeriesSplit, StratifiedKFold) on [github.com/scikit-learn/scikit-learn](https://github.com/scikit-learn/scikit-learn) — read the real implementation logic. (not URL-verified this session)
