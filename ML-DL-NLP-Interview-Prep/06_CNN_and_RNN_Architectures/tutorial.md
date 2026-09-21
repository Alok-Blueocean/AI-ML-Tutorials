# CNN and RNN Architectures

## The Convolution Operation

A convolution slides a small learnable filter (e.g., 3x3) across the input, computing a dot product at each position to produce a feature map. Unlike a fully connected layer, the same filter (weight-sharing) is reused across the whole image, drastically cutting parameters and encoding translation invariance — a cat's whisker looks like a whisker wherever it appears in the frame.

**Real-world grounding:** an X-ray defect detector doesn't need a separate set of weights to detect a crack in the top-left corner vs. bottom-right — the same edge-detecting filter should fire wherever the crack is, which is exactly what convolution weight-sharing gives for free.

## Filters, Stride, and Padding

- **Filters/kernels**: each filter learns to detect one pattern (edge, texture, later: object parts). A conv layer with 64 filters outputs 64 feature maps.
- **Stride**: how far the filter moves each step. Stride 2 halves spatial resolution, trading detail for compute/receptive-field growth.
- **Padding**: "same" padding (zero-pad the border) keeps output size equal to input size; "valid" (no padding) shrinks it. Padding matters most for deep networks — without it, spatial dimensions vanish after a handful of layers.

**Real-world grounding:** a real-time video model on an edge device deliberately uses stride-2 convolutions instead of pooling in early layers purely to cut compute, since stride can do the downsampling job pooling used to do, with fewer operations.

## Pooling

Max/average pooling downsamples feature maps (e.g., 2x2 max pool halves height and width), providing a small amount of translation invariance and reducing computation for deeper layers. Modern architectures (ResNet onward) often replace pooling with strided convolutions or use pooling sparingly, relying on it mainly at the very end (global average pooling before the classifier head).

**Real-world grounding:** global average pooling before the final classification layer (instead of flattening) is why modern CNNs can accept variable input image sizes at inference time.

## Receptive Fields

The receptive field is the region of the original input that a given neuron's activation "sees." It grows with depth: stacking two 3x3 convolutions gives a 5x5 effective receptive field with fewer parameters than one 5x5 conv, and adding stride/pooling grows it faster. Interviewers often probe this because architecture depth is partly a receptive-field engineering decision — you need a large enough receptive field to see the whole object of interest.

**Real-world grounding:** a satellite-imagery model detecting large structures (airports, stadiums) needs many stacked layers or dilated convolutions specifically to grow the receptive field to cover the whole object, not just texture detail.

## Classic CNN Architectures

- **LeNet (1998)**: the original conv+pool+FC pattern for digit recognition, ~60k parameters, proof of concept.
- **VGG (2014)**: uniform stacks of small 3x3 convs, very deep (16-19 layers) but ~138M parameters — showed depth helps but at heavy compute/memory cost.
- **ResNet (2015)**: introduces skip/residual connections — `output = F(x) + x` — so a layer only needs to learn a residual correction, not a full remapping. This solves the degradation problem where plain networks beyond ~20 layers got *worse* (not just overfit) as depth increased, because gradients could no longer propagate cleanly and identity mappings were hard to learn. ResNet made 50-152+ layer networks trainable and is still the backbone of choice for many production vision pipelines.

**Real-world grounding:** a 152-layer ResNet is why an industrial defect-detection pipeline can use a very deep, high-accuracy backbone at all — without skip connections, that depth would simply fail to train (gradients vanish or the network degrades below a shallower baseline).

## Transfer Learning / Fine-Tuning Pretrained CNNs

Pretrained ImageNet backbones (ResNet, EfficientNet) have already learned generic low/mid-level features (edges, textures, shapes) in early layers. For a new task with limited data: freeze early layers, replace the final classification head, train the head first, then optionally unfreeze the last few blocks with a small learning rate for fine-tuning. This is the default strategy whenever labeled data is scarce (< tens of thousands of images).

**Real-world grounding:** a retail company detecting shelf out-of-stocks with only 2,000 labeled photos fine-tunes a pretrained ResNet head rather than training from scratch — training from scratch on 2,000 images would badly overfit.

## RNNs and Backpropagation Through Time (BPTT)

An RNN applies the same weights at every timestep, maintaining a hidden state `h_t = f(W_x x_t + W_h h_{t-1} + b)`. BPTT unrolls the recurrence across time and backpropagates the loss through every timestep, meaning the same weight matrix receives gradient contributions from every step — effectively a very deep network (depth = sequence length) with shared weights.

**Real-world grounding:** BPTT is why training on very long sequences (e.g., a full customer session log of 500+ events) requires truncated BPTT — backpropagating through the full history is both memory-prohibitive and gradient-unstable.

## Vanishing Gradients Over Long Sequences

Because BPTT effectively multiplies the same weight matrix (and activation derivative) at every timestep, gradients shrink (or explode) exponentially with sequence length — an RNN struggles to learn dependencies more than ~10-20 steps back. This was the central motivation for LSTM/GRU.

**Real-world grounding:** a plain RNN predicting the next word in "The trophy doesn't fit in the suitcase because **it** is too [big/small]" fails to resolve "it" correctly over long contexts — exactly the long-range dependency problem LSTMs were built to fix.

## LSTM (Forget/Input/Output Gates)

LSTM adds a separate cell state `C_t` that flows through time with only minor, additive, gated modifications — no repeated matrix multiplication by the same weights at every step, which is why gradients survive much longer.

- **Forget gate**: `f_t = σ(W_f[h_{t-1}, x_t])` — decides what fraction of the old cell state to keep.
- **Input gate**: `i_t = σ(W_i[h_{t-1}, x_t])` — decides how much of the new candidate value to write in.
- **Output gate**: `o_t = σ(W_o[h_{t-1}, x_t])` — decides what part of the cell state to expose as the hidden state.

`C_t = f_t * C_{t-1} + i_t * tanh(candidate)`, `h_t = o_t * tanh(C_t)`.

**Real-world grounding:** an LSTM-based demand forecasting model relies on the forget gate to "drop" a promotional spike from three months ago while the cell state still carries the underlying seasonal trend forward.

## GRU

GRU merges the forget and input gates into a single "update gate" and drops the separate cell state, using only the hidden state. Fewer parameters (~25% less) than LSTM, trains faster, and performs comparably on many tasks — a common production choice when compute/latency matters more than squeezing out the last bit of accuracy.

**Real-world grounding:** a mobile keyboard's on-device next-word predictor uses a GRU rather than an LSTM specifically to fit within a tight memory and latency budget.

## Bidirectional RNNs

Runs two RNNs over the sequence — one forward, one backward — and concatenates their hidden states, so every position's representation has context from both past and future. Only usable when the full sequence is available upfront (not for real-time/streaming generation).

**Real-world grounding:** named entity recognition ("Washington" as a person vs. a place) benefits from seeing words *after* "Washington" too, which is why NER taggers commonly use BiLSTMs rather than unidirectional RNNs.

## Seq2Seq

An encoder RNN compresses an input sequence into a fixed-size context vector; a decoder RNN generates the output sequence conditioned on that vector (and, in later variants, on attention over all encoder states rather than one fixed vector). This became the backbone of machine translation before Transformers replaced the recurrence entirely.

**Real-world grounding:** early Google Translate (2016, pre-Transformer) used an 8-layer LSTM seq2seq encoder-decoder — the fixed-context-vector bottleneck for long sentences was exactly the problem that motivated adding attention, which in turn led directly to the Transformer.
