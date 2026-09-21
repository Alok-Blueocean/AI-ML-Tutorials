# References — Neural Network Fundamentals

## Foundational Reading
- **3Blue1Brown — "Neural Networks" video series (Deep Learning chapters 1-4)**: the best visual intuition for forward pass, gradient descent, and backpropagation. https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZCJB-3pi (also indexed at youtube.com/c/3blue1brown). Verified: channel and series confirmed to exist via search.
- **Andrej Karpathy — "Neural Networks: Zero to Hero" (video course + code)**: builds backprop and an autograd engine (`micrograd`) from scratch in Python, then scales up to MLPs. https://github.com/karpathy/nn-zero-to-hero — verified, exists.
- **Andrej Karpathy — `micrograd`**: ~150-line autograd engine + tiny NN library, the clearest from-scratch backprop implementation available. https://github.com/karpathy/micrograd — verified via WebFetch, 17.6k stars, description confirmed.
- **CS231n (Stanford) — "Neural Networks" lecture notes**: rigorous, concise notes on backprop, activation functions, initialization, and optimizers, written for exactly this kind of interview prep. https://cs231n.github.io/ (course home: http://cs231n.stanford.edu/) — verified, exists.

## Book
- **Goodfellow, Bengio, Courville — "Deep Learning" (MIT Press, 2016)**, Chapters 6 (Deep Feedforward Networks) and 8 (Optimization). The standard reference text; free to read online at https://www.deeplearningbook.org/ — verified via WebFetch.

## Key Papers
- **Glorot & Bengio, 2010 — "Understanding the difficulty of training deep feedforward neural networks"** (introduces Xavier initialization). Findable via Google Scholar / PMLR (JMLR Workshop and Conference Proceedings, vol. 9). "(not URL-verified this session)."
- **He et al., 2015 — "Delving Deep into Rectifiers: Surpassing Human-Level Performance on ImageNet Classification"** (introduces He initialization, motivates ReLU-aware init). arXiv:1502.01852. "(not URL-verified this session)."
- **Kingma & Ba, 2014 — "Adam: A Method for Stochastic Optimization"**. https://arxiv.org/abs/1412.6980 — verified via WebFetch.
- **Srivastava et al., 2014 — "Dropout: A Simple Way to Prevent Neural Networks from Overfitting"**, JMLR 15. "(not URL-verified this session)."
- **Ioffe & Szegedy, 2015 — "Batch Normalization: Accelerating Deep Network Training by Reducing Internal Covariate Shift"**. https://arxiv.org/abs/1502.03167 — verified via WebFetch.

## Articles Worth Reading
- **"Understanding the Vanishing Gradient Problem"** and similar explainers on GeeksforGeeks — good quick refresher with diagrams, surfaces reliably under "vanishing exploding gradients deep learning" searches.
- **Sebastian Ruder — "An overview of gradient descent optimization algorithms"**: the most-cited blog explainer comparing SGD/momentum/RMSProp/Adam side by side, commonly linked from ML interview prep guides. https://ruder.io/optimizing-gradient-descent/ — verified via WebFetch.

## Interview Prep
- **Adaface — "65 Neural Networks Interview Questions"**: https://www.adaface.com/blog/neural-networks-interview-questions/ — verified via search, covers overfitting, activation choice, and initialization questions.
- **Aced (Exponent) — "Top 39 Deep Learning Interview Questions"**: https://www.tryexponent.com/blog/top-deep-learning-interview-questions — verified via search.

## Practice Platform
- PyTorch official tutorials — "Learning PyTorch with Examples" (autograd, `nn.Module`, optimizers): https://pytorch.org/tutorials/beginner/pytorch_with_examples.html "(not URL-verified this session, but this is a stable, well-known official PyTorch documentation path)."

## Architecture Design, Hyperparameters, Regularization, Losses & Activations
- **Andrej Karpathy — "A Recipe for Training Neural Networks" (2019)**: the canonical practical guide to structuring the training process itself — start simple, overfit a single batch first, then regularize; widely cited as the best "how do I actually get a network to train" reference. https://karpathy.github.io/2019/04/25/recipe/ — verified via WebFetch (title and opening line confirmed).
- **Machine Learning Mastery — "How to Choose Loss Functions When Training Deep Learning Neural Networks"**: a direct, practical walkthrough of matching loss functions (MSE/MSLE/MAE for regression; BCE/hinge for binary; categorical cross-entropy/sparse/KL-divergence for multi-class) to output layer configuration. https://machinelearningmastery.com/how-to-choose-loss-functions-when-training-deep-learning-neural-networks/ — verified via WebFetch.
- **Machine Learning Mastery — "5 Useful Loss Functions"**: shorter companion piece covering BCE, Hinge, MSE, MAE, Huber with code. https://machinelearningmastery.com/5-useful-loss-functions/ "(not URL-verified this session; same domain and author series as the verified link above)."
- **Keras official docs — Losses API**: the authoritative reference for every built-in loss (probabilistic: BinaryCrossentropy, CategoricalCrossentropy, SparseCategoricalCrossentropy, KLDivergence, Poisson; regression: MSE, MAE, Huber, LogCosh; hinge family). https://keras.io/api/losses/ — verified via WebFetch, contents confirmed.
- **Keras official docs — Layers API** (for Flatten, GlobalAveragePooling2D, Dropout, BatchNormalization, Dense): https://keras.io/api/layers/ "(not URL-verified this session, but this is the stable top-level Keras layers documentation index)."
- **PyTorch official docs — `torch.nn` loss functions** (L1Loss, MSELoss, SmoothL1Loss/HuberLoss, CrossEntropyLoss, BCEWithLogitsLoss, TripletMarginLoss, CosineEmbeddingLoss): https://pytorch.org/docs/stable/nn.html#loss-functions "(not URL-verified this session, but this is the stable official PyTorch documentation path)."
- **CS231n (Stanford) — "Neural Networks" notes, Part 1-3** (already listed above under Foundational Reading): also directly covers architecture sizing (number of layers/neurons) and practical hyperparameter guidance — re-read Part 1 specifically for the "how many layers and how large" section. https://cs231n.github.io/neural-networks-1/ — same verified domain as the top-level cs231n.github.io entry above.
- **fast.ai — "Practical Deep Learning for Coders"**: free course + book, top-down approach that spends significant time on choosing architectures, loss functions, and regularization for real datasets (images, tabular, text) rather than only theory. https://course.fast.ai/ — verified via search, official fast.ai site.

## Key Papers (Architecture/Loss/Activation Choices)
- **Hendrycks & Gimpel, 2016 — "Gaussian Error Linear Units (GELUs)"**, arXiv:1606.08415, the paper defining GELU (`x·Φ(x)`), now the default activation in BERT/GPT/ViT. https://arxiv.org/abs/1606.08415 — verified via search (arXiv ID and abstract confirmed).
- **Lin, Goyal, Girshick, He, Dollár, 2017 — "Focal Loss for Dense Object Detection"** (RetinaNet paper), arXiv:1708.02002, introduces focal loss for extreme foreground/background class imbalance. https://arxiv.org/abs/1708.02002 — verified via search (arXiv ID, authors, and abstract confirmed).
- Srivastava et al. 2014 (Dropout) and Ioffe & Szegedy 2015 (Batch Normalization) are already listed above under Key Papers — both are directly relevant to the regularization-placement material in `tutorial.md`.

## Hyperparameter Search Tools
- **KerasTuner — official documentation**: define-by-run hyperparameter search (`hp.Int`, `hp.Choice`, etc.) integrated directly with Keras models; a practical tool for automating the learning-rate/dropout/width search described in `tutorial.md`. https://keras.io/keras_tuner/ — verified via search, official Keras documentation domain.
- **Optuna — official site and docs**: framework-agnostic hyperparameter optimization with define-by-run search spaces, pruning of unpromising trials, and support for any PyTorch/Keras/sklearn model. https://optuna.org/ (docs: https://optuna.readthedocs.io/) — verified via WebFetch, official homepage confirmed.
