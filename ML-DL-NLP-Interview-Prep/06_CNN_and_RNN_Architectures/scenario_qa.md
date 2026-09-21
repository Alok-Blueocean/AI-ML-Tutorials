# Scenario-Based Q&A — CNN and RNN Architectures

## 1. Small image dataset for a new visual inspection task
**Situation:** You're asked to build an image classifier for a manufacturing defect-detection task but only have 1,500 labeled images across 4 defect classes.

**What would you do and why:** Train from scratch is a non-starter at this data volume — a deep CNN would badly overfit. I'd use transfer learning: take a backbone (ResNet-18/34 or EfficientNet-B0) pretrained on ImageNet, freeze the convolutional base, replace the classification head for 4 classes, and train just the head first with heavy data augmentation (rotation, flips, brightness jitter appropriate to the factory camera setup). Once the head converges, I'd unfreeze the last block or two and fine-tune with a low learning rate (~1e-5) to adapt higher-level features to the specific defect textures. I'd validate with a stratified split and track per-class recall since defect classes are usually imbalanced (few defective samples vs. many "ok" samples).

## 2. Very deep network trains worse than a shallower one (not overfitting)
**Situation:** You train a 56-layer plain CNN (no skip connections) and it performs *worse* on both training and validation accuracy than an 18-layer version of the same architecture.

**What would you do and why:** This is the "degradation problem" ResNet was built to solve — it isn't overfitting (train accuracy is also worse), it's an optimization problem: gradients struggle to propagate through 56 plain layers, and the network fails to even learn the identity mapping a deeper net should trivially be able to represent. The fix is architectural, not data-related: add residual/skip connections every 2-3 conv layers so each block only has to learn a residual correction. I'd also add batch normalization after each conv if it's missing, since it further stabilizes gradient flow at that depth.

## 3. RNN forgets context from many steps ago
**Situation:** A sentiment classifier built on a vanilla RNN performs well on short reviews but fails on long reviews where the sentiment-bearing clause appears in the first sentence and is contradicted or clarified 40 words later.

**What would you do and why:** This is the vanishing-gradient long-range dependency problem inherent to vanilla RNNs. I'd switch to an LSTM or GRU, whose gating mechanism (particularly the forget gate and additive cell-state update) lets relevant information persist across many timesteps without being repeatedly squashed by matrix multiplication and activation saturation. If the review length is very long (100+ tokens) and directionality doesn't matter (the whole review is available at inference time), I'd also make it bidirectional so early-sentence context and later-sentence context both inform every position's representation. If accuracy still plateaus, I'd consider that this is exactly the kind of long-range dependency problem attention/Transformer-based models solve even better.

## 4. Choosing CNN vs RNN vs Transformer for a new sequence task
**Situation:** You're given a time-series of sensor readings (1 reading per second, sequences of ~500 steps) for equipment failure prediction, and asked to justify an architecture choice.

**What would you do and why:** I'd first try a simpler, cheaper baseline: a 1D CNN with dilated convolutions (or a small gradient-boosted-tree model on windowed features) since 500-step sequences with fairly local/periodic patterns often don't need full recurrence. If the failure signature depends on long-range temporal dependencies (a slow drift building over hundreds of steps), an LSTM/GRU is the next step up. I'd reach for a Transformer-based sequence model only if I have enough data to justify it and need to model very long-range or irregular dependencies, since at 500 steps attention's O(n²) cost is still manageable and its parallel training makes iteration faster than an RNN — but for equipment telemetry with modest data volume, I'd start simple and only add architectural complexity if the simpler baseline demonstrably underfits the pattern.

## 5. Fine-tuning destabilizes a pretrained model ("catastrophic forgetting")
**Situation:** After unfreezing the entire pretrained ResNet backbone and fine-tuning at the same learning rate used for the new head, validation accuracy drops well below the frozen-backbone baseline.

**What would you do and why:** This is a classic fine-tuning mistake — using too high a learning rate on pretrained weights destroys the useful features learned from ImageNet before the new head has a chance to guide gradients sensibly. I'd fix it with a much smaller learning rate for the backbone (often 10-100x smaller than the head's), potentially with per-layer/discriminative learning rates (deeper/earlier layers even smaller), and only unfreeze progressively (train the head first, then unfreeze block by block) rather than the whole network at once from the start.

## 6. Receptive field too small for the object of interest
**Situation:** A CNN for detecting large structural cracks in bridge-inspection images performs well on small cracks but consistently misses long cracks spanning much of the image.

**What would you do and why:** This points to an insufficient receptive field — the network's neurons never "see" enough of the image at once to recognize a crack that spans a large fraction of it. I'd increase receptive field size via deeper stacking of small convolutions, adding strided convolutions/pooling earlier to grow the field faster, or using dilated (atrous) convolutions to expand receptive field without losing resolution. I'd verify by computing the theoretical receptive field size of the current architecture and comparing it to the typical crack length in pixels.

## 7. GRU vs. LSTM choice under a latency constraint
**Situation:** You need a sequence model for on-device (mobile) autocomplete with a strict memory and latency budget, and a teammate insists on using LSTM because "it's more powerful."

**What would you do and why:** I'd push back with data, not opinion: GRU has ~25% fewer parameters than an LSTM at the same hidden size (it merges the forget/input gates and drops the separate cell state) and is empirically comparable in accuracy on many sequence tasks, particularly shorter ones typical of autocomplete. Given the hard on-device latency/memory budget, I'd benchmark both on the actual target hardware and let measured accuracy-vs-latency tradeoff decide, rather than assuming LSTM's extra capacity is worth the cost — in most autocomplete-scale deployments, GRU wins on this tradeoff.

## 8. Debugging exploding gradients in an RNN
**Situation:** Your RNN/LSTM training loss becomes `NaN` after a few hundred steps.

**What would you do and why:** This is the classic exploding-gradient signature from BPTT over a long sequence. I'd add gradient clipping (clip the global gradient norm to a fixed threshold, e.g., 1.0-5.0) — the standard, near-universal fix for this in recurrent models. I'd also check the learning rate isn't too high, verify inputs are normalized (unnormalized large-magnitude inputs compound the problem), and consider truncated BPTT if sequence length is very long, to reduce how many timesteps gradients must flow through.
