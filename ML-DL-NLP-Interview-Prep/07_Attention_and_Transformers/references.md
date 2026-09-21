# References — Attention and Transformers

## Key Papers
- **Vaswani et al., 2017 — "Attention Is All You Need"** (the original Transformer paper). https://arxiv.org/abs/1706.03762 — verified via WebFetch (title, authors, abstract confirmed).
- **Devlin et al., 2018 — "BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding"**. https://arxiv.org/abs/1810.04805 — verified via search.
- **Bahdanau, Cho, Bengio, 2014 — "Neural Machine Translation by Jointly Learning to Align and Translate"** (the original attention mechanism, pre-Transformer). https://arxiv.org/abs/1409.0473 — verified via search.
- **Dao et al., 2022 — "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness"** (why O(n²) attention memory can be engineered around without approximation). https://arxiv.org/abs/2205.14135 — verified via search.

## Visual / Intuitive Explainers
- **Jay Alammar — "The Illustrated Transformer"**: the standard visual walkthrough of Q/K/V, multi-head attention, and encoder/decoder blocks; used in Stanford/Harvard/MIT course materials. https://jalammar.github.io/illustrated-transformer/ — verified via WebFetch.
- **Jay Alammar — "The Illustrated GPT-2"**: extends the same visual approach to decoder-only, autoregressive Transformers. https://jalammar.github.io/illustrated-gpt2/ — verified via search (confirmed present on jalammar.github.io).

## Course
- **Hugging Face — "LLM Course" (formerly the "NLP Course"), Chapter 1**: "How do Transformers work?" and "Transformer Architectures" sections cover attention, encoder/decoder/encoder-decoder variants, hands-on with the `transformers` library. https://huggingface.co/learn/llm-course/en/chapter1/1 — verified via search.
- **CS231n / CS224n (Stanford)** — CS224n (NLP with Deep Learning) covers attention and Transformers in depth; course materials at https://web.stanford.edu/class/cs224n/ "(not URL-verified this session, but this is the long-standing, stable official CS224n course URL)."

## Hands-On Implementation
- **Harvard NLP — "The Annotated Transformer"**: a line-by-line PyTorch implementation of the original Transformer paper, the standard "build it yourself" reference. https://github.com/harvardnlp/annotated-transformer — verified via WebFetch (7.5k stars, MIT license, confirmed description). Blog version: http://nlp.seas.harvard.edu/annotated-transformer/.
- **labml.ai — `annotated_deep_learning_paper_implementations`**: side-by-side annotated PyTorch implementations of the original Transformer plus many variants (Transformer-XL, Switch, ViT). https://github.com/labmlai/annotated_deep_learning_paper_implementations — verified via search.
- **Dao-AILab — `flash-attention`**: the official FlashAttention implementation, useful to see how real systems handle the O(n²) memory cost in practice. https://github.com/Dao-AILab/flash-attention — verified via search.

## Interview Prep
- **DataInterview — "Top 32 LLMs & Transformers Interview Questions"**: https://www.datainterview.com/blog/llms-and-transformers-interview-questions — verified via search, covers scaled dot-product scaling, parallelization, and architecture comparison questions.
- **Analytics Vidhya — "Top 16 Interview Questions on Transformer"**: https://www.analyticsvidhya.com/blog/2022/11/top-6-interview-questions-on-transformer/ — verified via search.
