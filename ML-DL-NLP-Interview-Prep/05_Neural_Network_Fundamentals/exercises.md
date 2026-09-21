# Exercises — Neural Network Fundamentals

1. **(Easy)** By hand, show why a single perceptron cannot solve XOR. Draw the 4 points in 2D and show no line separates them.
2. **(Easy)** Implement `sigmoid`, `tanh`, `relu`, `leaky_relu`, and their derivatives in NumPy. Plot all five functions and their gradients on `[-5, 5]`.
3. **(Easy)** Implement softmax in NumPy in a numerically stable way (subtract the max logit before exponentiating). Explain why the naive version overflows.
4. **(Easy-Medium)** Implement a 2-layer neural net's forward and backward pass in NumPy for binary classification (see the code sketch in `tutorial.md`), train it on a synthetic 2D XOR-like dataset, and verify it converges where a linear model fails.
5. **(Medium)** Derive the gradient of binary cross-entropy loss with respect to the pre-sigmoid logit `z`, and show it simplifies to `(ŷ - y)`.
6. **(Medium)** Implement He initialization and Xavier initialization from scratch (no `torch.nn.init`). Train the same 10-layer MLP with each and compare training loss curves over the first 100 steps.
7. **(Medium)** Implement dropout from scratch (inverted dropout) as a NumPy function usable in both train and eval mode. Verify that the expected activation value is unchanged between modes.
8. **(Medium)** Implement batch normalization forward and backward passes from scratch (forward is enough if time-limited); verify against `torch.nn.BatchNorm1d` output on the same input.
9. **(Medium)** Train the same small MLP on a noisy dataset with (a) no regularization, (b) L2 weight decay, (c) dropout, (d) early stopping. Plot train vs. validation loss for all four and explain the differences.
10. **(Medium-Hard)** Implement SGD with momentum, RMSProp, and Adam from scratch as plain Python update rules (given gradients each step). Run all three on the Rosenbrock function and compare convergence paths.
11. **(Hard)** Derive why Adam's raw moment estimates need bias correction in early steps (hint: consider `E[m_t]` when `m_0 = 0` and the true gradient is constant).
12. **(Hard)** Explain, with a small worked numerical example, how a 20-layer network of sigmoid activations makes gradients vanish during backprop. Compute the approximate gradient magnitude at layer 1 assuming each layer's local gradient averages 0.2.
13. **(Hard)** Given the Universal Approximation Theorem holds for a single wide hidden layer, explain concretely (with parameter-count reasoning) why a deep narrow network is usually preferred over a shallow wide one for a task like MNIST digit classification.
14. **(Hard)** Design (on paper, no code) a feedforward network for a 40-feature tabular fraud dataset with heavy class imbalance (0.5% fraud). Specify layer sizes, activations, output layer, loss function, and one regularization strategy, and justify each choice.

## Design-Decision Exercises (Scenario-Based)

15. **(Medium)** You're given a dataset of 10,000 product images across 200 fine-grained categories (single label per image). Design the output layer, activation, and loss function, and justify a choice of Flatten vs. Global Average Pooling before the classifier head, given the dataset size relative to the class count.

16. **(Medium)** A dataset of news articles must be tagged with zero or more of 15 topic labels per article (multi-label). Specify the output layer size, activation, and loss function. Explain concretely why using softmax + categorical cross-entropy here (a common mistake) would break, and what metric you'd monitor instead of plain accuracy.

17. **(Medium)** You must predict insurance claim amounts, and the target is heavily right-skewed with a small number of very large claims (potential outliers, not data errors). Compare MSE, MAE, and Huber loss for this target, pick one, and justify it. Would a log-transform of the target change your answer?

18. **(Medium)** A binary classifier for rare-disease screening has a 1:500 positive:negative ratio. Specify the output layer/activation, compare plain binary cross-entropy vs. class-weighted BCE vs. focal loss for this scenario, and state which evaluation metric you'd report to a non-technical stakeholder instead of accuracy.

19. **(Easy-Medium)** Implement a `Dense → BatchNorm → ReLU → Dropout` block in PyTorch as a reusable `nn.Module`, and in a comment explain why this ordering (and not `Dense → Dropout → BatchNorm → ReLU`, or `Dense → ReLU → Dropout → BatchNorm`) is the recommended default.

20. **(Easy-Medium)** Implement in Keras a small CNN feature extractor followed by (a) `Flatten → Dense(256) → Dense(num_classes)` and (b) `GlobalAveragePooling2D → Dense(num_classes)`. Print and compare the total parameter count of both classifier heads for a `(7,7,512)` feature map and `num_classes=20`.

21. **(Medium)** You're building a face-verification system (not classification — the set of identities at inference time is unbounded and unseen during training). Explain why a softmax classifier over training identities is the wrong architecture, and design the output layer and loss function you'd use instead.

22. **(Medium)** Given 40 input features for a tabular MLP, justify a starting hidden-layer width (e.g., 32 vs. 128 vs. 256 units) using the capacity-then-regularize loop described in `tutorial.md`. What experiment would tell you if 32 units is too few?

23. **(Medium-Hard)** A colleague trains a regression model to predict daily product demand (a non-negative count, often 0-5, occasionally 100+ during promotions) using plain MSE with a linear output. Diagnose what's likely wrong with this setup and redesign the output activation and loss function.

24. **(Medium-Hard)** Given a 4-layer MLP (`Input → Dense → Dense → Dense → Output`), specify exactly which layers get Dropout and which don't, and justify why Dropout should not sit directly on the input layer or directly before a sigmoid/softmax output.

25. **(Hard)** You must choose between BatchNorm-before-activation and BatchNorm-after-activation for a new CNN architecture with a tight one-day tuning budget. Design a minimal experiment (what you'd log, how many runs, what you'd conclude from a null result) to decide, rather than picking based on the original paper alone.

26. **(Hard)** A multi-class classifier (50 classes) trained with sparse categorical cross-entropy has near-100% train accuracy on 500 total training images. List, in priority order, the three architecture/regularization changes you'd try first, and for each, state what learning-curve signature would confirm it worked.

27. **(Hard)** Implement a small hyperparameter search (using Keras Tuner or Optuna, or a hand-rolled grid/random search if neither is available) over learning rate, dropout rate, and number of hidden units for a tabular binary classifier. Report the best configuration and the validation-metric sensitivity to each hyperparameter individually.
