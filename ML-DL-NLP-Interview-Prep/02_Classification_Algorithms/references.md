# References — Classification Algorithms

## Official Docs
- [scikit-learn: Support Vector Machines](https://scikit-learn.org/stable/modules/svm.html) — kernels, C/gamma tuning guidance, multi-class strategy. Verified.
- [scikit-learn: Decision Trees](https://scikit-learn.org/stable/modules/tree.html) — Gini vs. entropy formulas and minimal cost-complexity (`ccp_alpha`) pruning. Verified.
- [scikit-learn: Naive Bayes](https://scikit-learn.org/stable/modules/naive_bayes.html) — Gaussian/Multinomial/Bernoulli/Complement/Categorical variants. Verified.
- [scikit-learn: Probability Calibration](https://scikit-learn.org/stable/modules/calibration.html) — Platt scaling (sigmoid), isotonic regression, reliability diagrams, Brier score. Verified.

## Key Papers
- Cortes, C. & Vapnik, V. (1995). *Support-Vector Networks*. Machine Learning, 20(3), 273–297. The original SVM paper, extending the separable-case idea to soft-margin (non-separable) classification. Confirmed to exist via search (Springer/ACM listing); (not URL-verified this session — access via [Springer](https://link.springer.com/article/10.1007/BF00994018)).
- Platt, J. (1999). *Probabilistic Outputs for Support Vector Machines and Comparisons to Regularized Likelihood Methods*. The paper behind "Platt scaling" used in `CalibratedClassifierCV(method="sigmoid")`. (not URL-verified this session)

## Articles / Blog Posts (interview-oriented)
- [GeeksforGeeks: Differentiate between SVM and Logistic Regression](https://www.geeksforgeeks.org/machine-learning/differentiate-between-support-vector-machine-and-logistic-regression/) — the exact comparison table interviewers ask for.
- [Analytics Vidhya: 30 Questions to Test Your Skills on Tree Based Models](https://www.analyticsvidhya.com/blog/2017/09/30-questions-test-tree-based-models/) — broad tree/ensemble quiz bank.
- [GeeksforGeeks: Gini Impurity and Entropy in Decision Tree](https://www.geeksforgeeks.org/machine-learning/gini-impurity-and-entropy-in-decision-tree-ml/) — side-by-side math and when results diverge.
- [Analytics Vidhya: 20 Questions to Test your Skills on Logistic Regression](https://www.analyticsvidhya.com/blog/2021/05/20-questions-to-test-your-skills-on-logistic-regression/) — commonly recycled logistic-regression interview bank.

## Video / Course
- StatQuest with Josh Starmer — [official video index](https://statquest.org/video_index.html): "Decision and Classification Trees," "Naive Bayes" / "Gaussian Naive Bayes," and the 4-part "Support Vector Machines" series (polynomial and RBF kernels). Verified — best short, visual explanations for interview-night review.
- Andrew Ng, [Machine Learning Specialization (Coursera)](https://www.coursera.org/specializations/machine-learning-introduction) — Course 1 covers logistic regression and decision boundaries from first principles. Verified.

## Book
- James, Witten, Hastie, Tibshirani. *An Introduction to Statistical Learning*, Chapter 4 (Classification: logistic regression, LDA, Naive Bayes) and Chapter 9 (Support Vector Machines). Free PDF at [statlearning.com](https://www.statlearning.com/). Verified.

## GitHub Repos
- [0xHadyy/LogisticLearn](https://github.com/0xHadyy/LogisticLearn) — Logistic regression from scratch in NumPy with L1/L2, cross-validation, grid search, and sklearn benchmarks; useful cross-check for a from-scratch exercise. (not URL-verified this session, but matched search result closely)
- scikit-learn source, `sklearn/svm/`, `sklearn/tree/`, `sklearn/naive_bayes.py` on [github.com/scikit-learn/scikit-learn](https://github.com/scikit-learn/scikit-learn) — read the real production implementations after building your own from scratch. (not URL-verified this session)
