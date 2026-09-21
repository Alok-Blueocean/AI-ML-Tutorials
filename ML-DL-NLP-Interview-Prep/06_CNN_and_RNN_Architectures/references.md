# References — CNN and RNN Architectures

## Courses / Notes
- **CS231n (Stanford) — Convolutional Neural Networks for Visual Recognition**: the canonical course for CNN architectures, receptive fields, and transfer learning. Notes: https://cs231n.github.io/ , course home: http://cs231n.stanford.edu/ — verified, exists.
- **colah's blog — "Understanding LSTM Networks"**: the most widely cited visual/intuitive explanation of LSTM gates and cell state. https://colah.github.io/posts/2015-08-Understanding-LSTMs/ — verified via WebFetch.

## Key Papers
- **He, Zhang, Ren, Sun, 2015 — "Deep Residual Learning for Image Recognition"** (ResNet). https://arxiv.org/abs/1512.03385 — verified via WebFetch (title, authors, abstract confirmed).
- **Simonyan & Zisserman, 2014 — "Very Deep Convolutional Networks for Large-Scale Image Recognition"** (VGG). https://arxiv.org/abs/1409.1556 — verified via search (arXiv id confirmed, paper also hosted at robots.ox.ac.uk/~vgg).
- **Hochreiter & Schmidhuber, 1997 — "Long Short-Term Memory"**, Neural Computation 9(8):1735-1780 (the original LSTM paper). Hosted at MIT Press: https://direct.mit.edu/neco/article/9/8/1735/6109/Long-Short-Term-Memory — verified via search (title/venue/authors/year confirmed); full text is paywalled but the abstract page resolves.
- **Cho et al., 2014 — "Learning Phrase Representations using RNN Encoder-Decoder for Statistical Machine Translation"** (introduces the GRU). https://arxiv.org/abs/1406.1078 — verified via search.
- **Sutskever, Vinyals, Le, 2014 — "Sequence to Sequence Learning with Neural Networks"** (the seq2seq paper). https://arxiv.org/abs/1409.3215 — verified via search.
- **Bahdanau, Cho, Bengio, 2014 — "Neural Machine Translation by Jointly Learning to Align and Translate"** (adds attention to seq2seq, direct precursor to Transformers). https://arxiv.org/abs/1409.0473 — verified via search.

## Official Docs / Hands-On
- **PyTorch official tutorial — "Transfer Learning for Computer Vision Tutorial"**: fine-tuning vs. feature-extraction with a pretrained ResNet, directly usable code. https://docs.pytorch.org/tutorials/beginner/transfer_learning_tutorial.html — verified via search.
- **PyTorch docs — `torch.nn.LSTM`, `torch.nn.GRU`, `torch.nn.Conv2d`**: official API reference for exact gate equations and parameter shapes. https://pytorch.org/docs/stable/nn.html "(not URL-verified this session, but this is the stable, canonical PyTorch nn documentation index)."

## GitHub Repos
- **KaimingHe/deep-residual-networks**: the original authors' Caffe implementation and model zoo for ResNet. https://github.com/kaiminghe/deep-residual-networks — verified via search results.
- **karpathy/char-rnn**: classic from-scratch character-level RNN/LSTM implementation, still widely used to teach BPTT and text generation. Findable at https://github.com/karpathy/char-rnn "(not URL-verified this session, but well-known, long-standing repo)."

## Interview Prep
- **Analytics Vidhya — "Top 25 Interview Questions on RNN"**: https://www.analyticsvidhya.com/blog/2023/05/top-interview-questions-for-rnn/ — verified via search, covers vanishing gradients, LSTM/GRU comparisons.
- **GeeksforGeeks — "RNN vs LSTM vs GRU vs Transformers"**: quick comparison table format frequently used in interview prep. https://www.geeksforgeeks.org/deep-learning/rnn-vs-lstm-vs-gru-vs-transformers/ — verified via search.
