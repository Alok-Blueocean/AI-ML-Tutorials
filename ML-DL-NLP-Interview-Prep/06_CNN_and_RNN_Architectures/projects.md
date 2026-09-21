# Projects — CNN and RNN Architectures

## Small: CNN From Scratch on CIFAR-10
Build a 4-6 layer CNN (conv + batch norm + ReLU + pooling blocks, ending in global average pooling and a softmax head) in PyTorch, trained from scratch on CIFAR-10 (60k 32x32 images, 10 classes, built into `torchvision.datasets`). Compare accuracy and training stability with/without batch norm and with/without a residual connection between two blocks. This proves you can implement and reason about the core CNN building blocks without leaning on a pretrained model.

## Medium: Transfer Learning for a Real Defect/Quality Dataset
Fine-tune a pretrained ResNet-18 or EfficientNet on a small real-world image dataset such as the Kaggle "Casting Product Image Data for Quality Inspection" (~7,000 images, defective vs. ok castings) or the MVTec Anomaly Detection dataset. Compare frozen-backbone-plus-head training vs. full fine-tuning vs. training from scratch, reporting accuracy and training time for each. This directly demonstrates the transfer learning workflow interviewers ask about most: "you have a small labeled image dataset — what do you do?"

## Medium-Large: Sequence Tagging with LSTM/GRU/BiLSTM Comparison
Build a named-entity-recognition or part-of-speech tagger on a real tagged corpus (e.g., the CoNLL-2003 NER dataset) using an embedding layer + LSTM, then repeat with GRU and BiLSTM variants, keeping everything else constant. Report F1 per architecture and explain the differences in terms of what each architecture can and cannot see. This proves practical fluency with sequence models beyond reciting gate equations.

## Large: Seq2Seq Machine Translation With and Without Attention
Implement an encoder-decoder GRU/LSTM seq2seq model for a real (small-scale) translation task, e.g., English-French or English-German sentence pairs from the Multi30k dataset (~30k sentence pairs, commonly used for teaching NMT). First train a vanilla fixed-context-vector seq2seq model, then add Bahdanau/Luong-style attention on top, and quantify the BLEU score improvement, especially on longer sentences. This project is the single best way to internalize *why* attention (and eventually the Transformer) was invented — you will directly observe the fixed-context-vector bottleneck degrade on longer sequences and watch attention fix it.
