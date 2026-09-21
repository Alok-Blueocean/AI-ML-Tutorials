# Attention and Transformers

## Self-Attention: Queries, Keys, Values

Every token produces three learned projections of its embedding: a **query** (what am I looking for?), a **key** (what do I offer?), and a **value** (what information do I carry?). Attention output for a token is a weighted sum of all tokens' values, where the weight comes from how well that token's query matches each other token's key (`Q·K^T`). This lets every position directly access information from every other position in one step, regardless of distance — unlike an RNN, which has to relay information step by step.

**Real-world grounding:** in the sentence "The animal didn't cross the street because **it** was too tired," resolving "it" requires directly connecting it to "animal" many tokens away — self-attention lets "it"'s query directly match "animal"'s key in a single hop, which is exactly the long-range dependency problem RNNs struggled with.

## Scaled Dot-Product Attention and Why the Scaling Factor Matters

`Attention(Q,K,V) = softmax(QK^T / √d_k) V`. Without the `√d_k` divisor, dot products grow in magnitude as the key dimension `d_k` increases (variance of the dot product scales with `d_k` for random Q, K), pushing softmax inputs into a regime where the max value dominates almost completely — the softmax becomes near one-hot, gradients through it vanish, and training stalls. Dividing by `√d_k` renormalizes the variance back to ~1 regardless of dimensionality, keeping softmax in a well-behaved gradient regime.

**Real-world grounding:** this is one of the single most common "derive it" interview questions for Transformer roles — being unable to explain the scaling factor is a common way candidates lose credibility on an otherwise strong résumé.

```python
import torch, torch.nn.functional as F

def scaled_dot_product_attention(Q, K, V, mask=None):
    d_k = Q.size(-1)
    scores = Q @ K.transpose(-2, -1) / d_k ** 0.5
    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))
    weights = F.softmax(scores, dim=-1)
    return weights @ V, weights
```

## Multi-Head Attention

Instead of one attention computation over the full embedding dimension, split into `h` heads, each operating on a smaller `d_k = d_model / h` subspace, run scaled dot-product attention independently per head, then concatenate and project back to `d_model`. Each head can specialize — e.g., one head tracking syntactic dependencies, another tracking coreference — which a single attention computation over the full dimension cannot do as cleanly.

**Real-world grounding:** in BERT, visualizing attention heads shows some heads consistently attend to the previous token (local syntax), others to the sentence's subject regardless of position (long-range dependency) — this specialization is the empirical payoff of using multiple heads instead of one.

## Positional Encoding

Self-attention has no inherent notion of order — `Attention(Q,K,V)` is permutation-invariant, since swapping two input rows just swaps two output rows identically. Transformers inject position information explicitly by adding a positional encoding vector to each token embedding before the first layer. The original paper uses fixed sinusoidal functions of different frequencies (`sin`/`cos` at each dimension), so relative positions can be represented as a linear function of each other; many modern models use learned or rotary (RoPE) positional embeddings instead.

**Real-world grounding:** without positional encoding, a Transformer reading "dog bites man" and "man bites dog" would produce nearly identical representations for both sentences (same set of tokens, same attention math) — positional encoding is what lets order-dependent meaning survive.

## Transformer Encoder/Decoder Blocks

- **Encoder block**: multi-head self-attention → add & layer norm → feed-forward network (2-layer MLP with a nonlinearity) → add & layer norm. Stacked N times (6 in the original paper).
- **Decoder block**: adds a *masked* self-attention (each position can only attend to earlier positions, preserving autoregressive generation) plus a cross-attention layer where queries come from the decoder and keys/values come from the encoder output — this is how the decoder "looks up" relevant source information while generating each output token.

**Real-world grounding:** in machine translation, decoder cross-attention is literally the mechanism that lets the model look back at the right source word (e.g., align "chat" in French to "cat" in English) while generating each target token — a direct, learned replacement for the fixed-context-vector bottleneck of vanilla seq2seq.

## Layer Norm and Residual Connections

Every sub-layer (attention, feed-forward) is wrapped as `LayerNorm(x + Sublayer(x))` — a residual connection plus layer normalization. The residual connection lets gradients flow directly through the stack even as depth grows to dozens of layers (same idea as ResNet, applied to Transformers). Layer norm (normalizing across the feature dimension per token, not across the batch like batch norm) stabilizes training regardless of batch size or sequence length — a good fit for NLP where batch statistics vary a lot with padding.

**Real-world grounding:** GPT-3/GPT-4-scale models (96+ transformer blocks) are only trainable at all because of this residual+layernorm pattern — remove it and gradient flow collapses well before you reach that depth, mirroring exactly why ResNet needed skip connections for very deep CNNs.

## Why Transformers Parallelize Better Than RNNs — And at What Cost

RNNs process a sequence strictly step by step (`h_t` depends on `h_{t-1}`), so training cannot parallelize across the time dimension — an `n`-length sequence takes `O(n)` sequential operations. Self-attention computes all pairwise token interactions at once — `O(1)` sequential operations, `O(n²)` parallel work — so on a GPU with enough memory, the entire sequence is processed in one shot, drastically speeding up training. The cost is that attention's memory and compute scale as `O(n²)` in sequence length, which is why very long contexts (documents, whole codebases) require specialized techniques (sparse attention, sliding-window attention, linear attention, FlashAttention-style memory-efficient kernels) rather than naive full attention.

**Real-world grounding:** this exact tradeoff — RNN's poor parallelization vs. Transformer's O(n²) memory wall — is why long-context LLMs (100k+ tokens) rely on engineering tricks like FlashAttention and sliding-window/sparse attention patterns rather than plain scaled dot-product attention over the full context.
