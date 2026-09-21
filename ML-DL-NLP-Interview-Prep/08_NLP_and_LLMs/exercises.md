# NLP and LLMs — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual questions.

1. **Tokenize and normalize.** Take 5 sentences, tokenize them, then produce both the stemmed and lemmatized versions using a library (e.g., NLTK or spaCy). Write down 3 cases where stemming and lemmatization disagree, and explain which output is "more correct" for a search use case.

2. **Build a TF-IDF baseline classifier.** Using a small labeled text dataset (e.g., SMS spam, or 20 Newsgroups subset), build a TF-IDF + logistic regression classifier. Report accuracy/F1. This is your baseline for exercise 6.

3. **Conceptual: CBOW vs skip-gram.** Explain why skip-gram tends to perform better than CBOW on rare words, using the structure of the training objective (not just "it's known to be better").

4. **Train tiny word embeddings.** Train a Word2Vec model (gensim is fine) on a modest corpus (e.g., a few thousand Wikipedia articles or a book corpus). Query nearest neighbors for 5 words and sanity-check whether they're semantically reasonable.

5. **Conceptual: static vs contextual.** Give a concrete sentence pair where a static embedding model would assign the same vector to a word with two different meanings, and explain in one paragraph how a transformer's self-attention resolves it.

6. **Subword tokenizer from scratch (simplified).** Implement a basic BPE tokenizer (merge most frequent adjacent pairs) on a small text corpus and show the merge steps for the first 10 merges. Compare the resulting vocabulary size and average tokens-per-word against whitespace tokenization.

7. **Fine-tune a small encoder.** Fine-tune a pretrained BERT-family model (e.g., `distilbert-base-uncased`) on the same classification task from exercise 2. Compare metrics against your TF-IDF baseline and discuss the tradeoff in latency/cost vs accuracy gain.

8. **Conceptual: architecture choice.** For each of the following tasks, state whether you'd pick encoder-only, decoder-only, or encoder-decoder, and justify in 2 sentences: (a) sentiment classification, (b) machine translation, (c) open-ended chatbot, (d) extracting named entities from contracts.

9. **Parameter-efficient fine-tuning.** Fine-tune a small open LLM (e.g., a 1-3B parameter model) using LoRA on a narrow task (e.g., customer-support tone adaptation, or converting formal text to casual text). Report trainable parameter count vs total parameter count, and compare output quality against zero-shot prompting of the same base model.

10. **Build a small RAG pipeline.** Build a RAG pipeline over ~20 PDFs (e.g., your company's internal docs, or a set of research papers): chunk the text, embed chunks with a sentence-embedding model, index with FAISS, and answer 10 test questions by retrieving top-k chunks and passing them to an LLM. Report which questions failed and why (bad chunking, bad retrieval, or bad generation).

11. **Conceptual: RAG failure modes.** For your pipeline in exercise 10, classify each failure into one of: retrieval miss (right chunk never retrieved), context stuffing failure (right chunk retrieved but LLM ignored it), or hallucination (LLM invented an answer not present in any chunk). Propose one concrete fix per category.

12. **Hallucination evaluation.** Design a small evaluation set of 15 questions for your RAG pipeline where 5 have clear answers in the docs, 5 have partial/ambiguous answers, and 5 have no answer in the docs at all. Measure how often the system correctly says "I don't know" for the last 5.

13. **Chunking strategy comparison.** Re-run your RAG pipeline from exercise 10 with three different chunking strategies (fixed-size, sentence-based, semantic/paragraph-based) and compare retrieval quality. Explain why chunk size affects both recall and hallucination risk.

14. **Conceptual: embeddings vs keyword search.** Explain a scenario where pure keyword search (BM25) would outperform semantic embedding search, and one where the reverse is true. What does a production search system usually do to get the best of both?

15. **End-to-end system critique.** Given a described production RAG chatbot that answers customer questions from a 500-page manual and is hallucinating ~10% of the time, propose a prioritized list of 5 concrete interventions (data, retrieval, prompting, model, evaluation) and justify the order.
