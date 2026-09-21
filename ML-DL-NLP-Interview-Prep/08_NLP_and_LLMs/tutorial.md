# NLP and LLMs — Condensed Study Notes

## Text Preprocessing: Tokenization, Stemming, Lemmatization

- **Tokenization** splits text into units (words, subwords, characters) a model can consume. Modern LLMs use subword tokenization (below), not whitespace splitting, because it handles rare/unseen words gracefully.

- **Stemming** chops word endings with crude rules (`running` -> `run`, but also `universe` -> `univers`). Fast, noisy.

- **Lemmatization** maps a word to its dictionary root using vocabulary + POS tagging (`better` -> `good`). Slower, more accurate.

- Real-world example: a legal-document search engine lemmatizes queries so "filing", "filed", and "files" all match the same indexed term, while a quick log-analytics grep pipeline uses stemming because speed matters more than precision.

## TF-IDF

- Term Frequency x Inverse Document Frequency scores a word by how often it appears in a document relative to how common it is across the corpus. Common words (`the`, `is`) get down-weighted; distinctive words get up-weighted.

```
tfidf(word, doc, corpus) = tf(word, doc) * log(N / df(word, corpus))
# tf  = count of word in doc (often normalized by doc length)
# df  = number of docs containing word
# N   = total number of docs
```

- Still used as a strong, cheap baseline for search/ranking and as a feature for classical ML classifiers before trying embeddings.

- Real-world example: a support-ticket triage system uses TF-IDF + logistic regression as the "boring baseline" that a fancy transformer model must beat before anyone approves the extra GPU cost.

## Word2Vec (CBOW vs Skip-gram) and GloVe

- **Word2Vec** learns dense vectors such that words in similar contexts get similar vectors ("king" - "man" + "woman" ~ "queen").

  - **CBOW** (Continuous Bag of Words): predicts the center word from surrounding context words. Faster, works better with frequent words.

  - **Skip-gram**: predicts surrounding context words from the center word. Slower, works better with rare words and smaller datasets.

- **GloVe** builds embeddings from global word co-occurrence statistics across the whole corpus (a count-based matrix factorization) instead of a sliding-window prediction task.

- Paper: Mikolov et al., "Efficient Estimation of Word Representations in Vector Space" (2013).

- Real-world example: an e-commerce search team in 2015 used Word2Vec-trained product-title embeddings to catch "sneakers" and "trainers" as related queries — something pure keyword match could not do.

## Why Static Embeddings Lost to Contextual Embeddings

- Word2Vec/GloVe give **one fixed vector per word**, regardless of context. "Bank" (river) and "bank" (finance) get the same vector.

- Contextual embeddings (ELMo, then BERT/GPT-style transformers) generate a **different vector per occurrence**, computed from the surrounding sentence via self-attention.

- Real-world example: a financial-news sentiment model using static embeddings misclassified "the stock crashed the party" style idioms; switching to BERT embeddings, which encode context, fixed a measurable chunk of these errors.

## BERT vs GPT

| | BERT | GPT |
|---|---|---|
| Architecture | Encoder-only (Transformer encoder stack) | Decoder-only (Transformer decoder stack) |
| Objective | Masked Language Modeling (MLM) + Next Sentence Prediction | Autoregressive next-token prediction |
| Attention | Bidirectional (sees full sentence, left and right) | Causal / masked (only sees tokens so far) |
| Best at | Understanding tasks: classification, NER, extractive QA | Generation tasks: chat, summarization, completion |

- BERT paper: Devlin et al., "BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding" (2018).

- GPT-3 paper: Brown et al., "Language Models are Few-Shot Learners" (2020).

- Real-world example: a company building a resume-parsing feature (extract structured fields from unstructured text) fine-tunes BERT because it is an understanding task; a company building a cover-letter generator uses a GPT-style model because it is a generation task.

## Encoder-only vs Decoder-only vs Encoder-Decoder

- **Encoder-only** (BERT, RoBERTa): produces contextual representations of input text. No native generation. Used for classification, embeddings, retrieval.

- **Decoder-only** (GPT, LLaMA, Claude, Mistral): generates text left-to-right. Dominant architecture for modern general-purpose LLMs because one architecture handles almost any task via prompting.

- **Encoder-decoder** (T5, BART, original Transformer, translation models): encoder reads the full input, decoder generates output conditioned on it. Natural fit for sequence-to-sequence tasks where input and output are distinct sequences.

- Real-world example: Google Translate-style systems favor encoder-decoder because "read a French sentence, write an English sentence" is exactly that shape; a general chatbot favors decoder-only because tasks are open-ended and mostly expressible as "continue this text."

## Subword Tokenization: BPE, WordPiece, SentencePiece

- Problem: word-level vocabularies can't cover every possible word (rare words, typos, new brands, other languages) and character-level is too slow/long-sequence.

- **BPE (Byte-Pair Encoding)**: iteratively merges the most frequent adjacent character/byte pairs into new tokens until a target vocab size is reached. Used by GPT-family models.

```
vocab = set(all characters)
repeat until vocab_size reached:
    pair = most_frequent_adjacent_pair(corpus, vocab)
    vocab.add(merge(pair))
    replace all occurrences of pair in corpus with merged token
```

- **WordPiece**: similar to BPE but merges pairs that maximize the likelihood of the training data (used by BERT).

- **SentencePiece**: a tokenizer framework that treats text as a raw stream (including spaces) so it works without pre-tokenized/whitespace-segmented input — critical for languages like Japanese/Chinese with no spaces. Can implement BPE or unigram-LM tokenization internally.

- Real-world example: a multilingual LLM serving Japanese and English users needs SentencePiece-style tokenization because whitespace-based pretokenization simply doesn't exist in Japanese.

## Fine-Tuning vs Prompt Engineering vs Parameter-Efficient Fine-Tuning (PEFT/LoRA)

- **Full fine-tuning**: update all model weights on task-specific data. Best final quality, most expensive (compute, storage per model copy), risk of catastrophic forgetting.

- **Prompt engineering**: change only the input (instructions, few-shot examples, structure) — zero training cost, fastest to iterate, but capped by what the base model already knows/can do, and brittle to prompt wording.

- **PEFT (e.g., LoRA, adapters, prefix-tuning)**: freeze the base model, train a small number of extra parameters. Gets close to full fine-tuning quality at a fraction of the trainable parameters and storage.

```
# LoRA idea: freeze W0 (pretrained), learn a low-rank update
h = W0 @ x + (B @ A) @ x         # B: d x r, A: r x k,  r << min(d, k)
# only B and A are trained; W0 stays frozen and unchanged
```

- LoRA paper: Hu et al., "LoRA: Low-Rank Adaptation of Large Language Models" (2021).

- Real-world example: a SaaS company serving 200 enterprise clients uses one base LLM plus 200 small LoRA adapters (one per client's tone/domain) instead of hosting 200 full fine-tuned models — this is the difference between feasible and impossible on their GPU budget.

## Retrieval-Augmented Generation (RAG)

- Combines a retriever (vector search over a document store) with a generator (LLM) so the model answers using retrieved, up-to-date, or proprietary text instead of relying only on what it memorized during pretraining.

```
query -> embed(query) -> vector_search(index, k=5) -> top_k_chunks
prompt = build_prompt(query, top_k_chunks)
answer = llm.generate(prompt)   # answer grounded in top_k_chunks, ideally with citations
```

- Paper: Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (2020).

- Real-world example: a support-ticket assistant answers questions about a refund policy that changed yesterday, without any retraining, because the new policy PDF was just re-indexed into the vector store.

## Hallucination: Evaluation and Mitigation

- Hallucination = the model states something fluent and confident but factually wrong or unsupported by its sources.

- Causes: gaps in training data, next-token objective rewards plausible-sounding text over "I don't know", and (in RAG) irrelevant or missing retrieved context that the model still tries to answer from.

- Evaluation: human review against source-of-truth; automated faithfulness/groundedness scoring (does the answer's claims trace back to retrieved context); LLM-as-judge scoring; benchmark suites (e.g., TruthfulQA-style probes).

- Mitigation: RAG with citations, instructing the model to say "I don't know" when unsupported, lowering temperature, constrained decoding/structured output, RLHF/DPO fine-tuning against hallucinated responses, and post-hoc fact-checking passes.

- Real-world example: a medical-info chatbot adds a hard rule — never answer dosage questions unless the answer is directly quoted from a retrieved, approved source document — because a fluent but wrong dosage is a liability, not just an inconvenience.

## Embeddings for Semantic Search and Vector Databases

- An embedding model maps text (or images) to a dense vector such that semantically similar inputs land close together in vector space (cosine similarity / dot product / L2 distance).

- A vector database (FAISS, Pinecone, Weaviate, Milvus, pgvector) indexes these vectors for fast approximate nearest-neighbor (ANN) search at scale (millions-billions of vectors) instead of brute-force comparison.

- This is the retrieval half of RAG, and also powers semantic search, deduplication, recommendation, and clustering.

- Real-world example: an internal knowledge-base search at a company returns the right onboarding doc even when the employee searches "how do I get a laptop" instead of the doc's literal title "IT Equipment Request Process," because the query and doc embeddings are close in vector space despite sharing almost no exact words.

## Quick Gotchas Worth Naming in an Interview

- Static embeddings cannot disambiguate polysemous words; contextual embeddings can, at the cost of needing a full forward pass per occurrence.

- A larger RAG chunk size improves context but increases the chance of irrelevant text diluting the prompt; a smaller chunk size improves precision but risks losing surrounding context — this is a tunable tradeoff, not a fixed rule.

- LoRA reduces trainable parameters, not inference latency by itself — at serving time the adapter is typically merged back into the base weights (or applied as a small extra matrix multiply), so inference cost is close to the base model's.

- Encoder-only models are not obsolete just because decoder-only LLMs are popular — for pure classification/retrieval/embedding tasks at scale, a fine-tuned encoder is usually cheaper and faster than prompting a large generative model.
