# Projects — Attention and Transformers

## Small: Scaled Dot-Product and Multi-Head Attention From Scratch
Implement scaled dot-product attention and multi-head attention purely in PyTorch tensor ops (no `nn.MultiheadAttention`), including causal masking. Validate correctness by matching output shapes and (with matched weights) numerical outputs against `torch.nn.MultiheadAttention`. This is a near-universal take-home/whiteboard exercise and proves you understand the mechanism at the tensor level, not just the diagram.

## Medium: Text Classification — Transformer Encoder vs. LSTM Baseline
Build a small Transformer encoder (2-4 layers, following your own implementation or "The Annotated Transformer") for a text classification task such as IMDB sentiment or AG News, and compare accuracy, training time, and convergence speed against an LSTM baseline on the same data and tokenization. This demonstrates the practical parallelization and accuracy tradeoffs discussed in the tutorial, grounded in a measurable result rather than theory alone.

## Medium-Large: Fine-Tune a Pretrained Transformer for a Domain-Specific Task
Fine-tune `distilbert-base-uncased` or `bert-base-uncased` (via Hugging Face `transformers`) on a real, moderately-sized labeled dataset outside its original pretraining distribution — e.g., support-ticket categorization, legal-clause classification, or a Kaggle NLP competition dataset. Report accuracy/F1 before and after fine-tuning, and inspect a few attention-head visualizations (e.g., via `bertviz`) on misclassified examples. This proves practical fine-tuning competence and the ability to diagnose model behavior via attention, both common interview and take-home asks.

## Large: Sequence-to-Sequence Machine Translation With a From-Scratch Transformer
Implement a full encoder-decoder Transformer (masked decoder self-attention + encoder-decoder cross-attention, following the Harvard NLP "Annotated Transformer") and train it on a real small-scale translation dataset such as Multi30k (English-German, ~30k sentence pairs). Compare BLEU score and training time against a GRU/LSTM-based seq2seq-with-attention baseline built in the CNN/RNN topic's projects. This is the capstone project for this topic: it forces you to implement every mechanism covered in the tutorial (Q/K/V, scaling, multi-head, positional encoding, residual+layernorm, masking, cross-attention) end to end and produces a directly comparable, quantified result against the RNN-era approach it replaced.
