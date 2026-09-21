# Exercises — CNN and RNN Architectures

1. **(Easy)** By hand, compute the output spatial size of a 32x32 input after a 3x3 convolution with stride 1, padding 0. Repeat with "same" padding, and repeat again with stride 2.
2. **(Easy)** Implement a single 2D convolution operation (no library conv function) in NumPy — a nested loop version is fine — and verify it matches `torch.nn.functional.conv2d` on a small random input.
3. **(Easy)** Explain why two stacked 3x3 convolutions have a larger receptive field than one 3x3 convolution, and compute the receptive field size after 3 stacked 3x3 conv layers (stride 1).
4. **(Easy-Medium)** Implement max pooling and average pooling forward passes from scratch in NumPy on a 4D batch tensor.
5. **(Medium)** Load a pretrained ResNet-18 (`torchvision.models.resnet18(pretrained=True)`), freeze all layers except the final FC layer, replace it for a 5-class problem, and fine-tune on a small custom/Kaggle image dataset (e.g., "Intel Image Classification" or "Cats vs Dogs" subset).
6. **(Medium)** Derive why residual connections (`y = F(x) + x`) make the gradient `dL/dx = dL/dy · (1 + dF/dx)` — explain why the "+1" term is the key to preventing vanishing gradients in very deep nets.
7. **(Medium)** Implement a vanilla RNN cell from scratch (no `nn.RNN`) in PyTorch or NumPy, train it on a toy sequence task (e.g., predicting the next character in a short repeated string), and observe how far back it can "remember" before failing.
8. **(Medium)** Implement an LSTM cell's forward pass manually (compute all 3 gates and the cell state update) using raw tensor ops, and verify your hidden/cell state outputs match `torch.nn.LSTMCell` given the same weights.
9. **(Medium)** Compare parameter counts of an LSTM vs. GRU cell with the same hidden size analytically (write the formula), then verify with `sum(p.numel() for p in model.parameters())` in PyTorch.
10. **(Medium-Hard)** Train a unidirectional vs. bidirectional LSTM on a sequence labeling task (e.g., POS tagging on a small tagged corpus) and compare F1 scores — explain the result in terms of what context each architecture can see.
11. **(Hard)** Derive mathematically why vanilla RNN gradients vanish/explode over long sequences (show the product-of-Jacobians term in BPTT and its dependence on sequence length).
12. **(Hard)** Build a simple seq2seq encoder-decoder (2-layer GRU each) for a toy translation-like task (e.g., reversing a sequence of tokens, or converting date formats), and identify the failure mode as input length grows — connect this to why attention was added to seq2seq models.
13. **(Hard)** Given a fixed compute budget, design and justify a CNN architecture (depth, filter counts, use of skip connections, pooling strategy) for a 64x64 satellite-image binary classification task (deforestation vs. not) with 15,000 labeled images.
