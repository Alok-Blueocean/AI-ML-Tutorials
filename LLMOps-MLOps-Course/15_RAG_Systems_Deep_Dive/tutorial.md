# Module 15 — RAG Systems Deep Dive

## What This Topic Is

Retrieval-Augmented Generation (RAG) is the pattern of giving an LLM access to your own documents at answer time, instead of relying only on what it learned during training. A user asks a question, the system searches a knowledge base for relevant chunks of text, and those chunks are inserted into the prompt so the model can answer using real, current, company-specific information.

The basic RAG pipeline everyone starts with — split documents into chunks, embed them, store them in a vector database, retrieve the top matches for a query, stuff them into the prompt — works fine as a demo. This module is about the six techniques that turn that demo into something that holds up in production: **chunking strategies, parent-child retrieval, hybrid search, multi-query retrieval, context compression, and re-ranking.**

## Why It Matters

A RAG system is only as good as the context it hands the model. If retrieval finds the wrong chunks, no amount of prompt engineering or model quality fixes the answer — the model will confidently answer based on irrelevant or incomplete information. Most "the AI gave a wrong answer" complaints in production RAG systems trace back to a retrieval problem, not a generation problem. That's why teams spend most of their tuning effort on the retrieval side, and why this module exists as its own deep dive rather than a footnote under "vector databases."

Getting retrieval right also directly controls cost and latency: every chunk you retrieve is tokens you pay for and time the model spends reading before it can answer. Good retrieval means fewer, better chunks — cheaper and faster, not just more accurate.

## Main Concepts

### 1. Chunking Strategies

You can't embed and search an entire document as one block — it's too long, and it mixes too many topics for a single embedding to represent well. So documents get split into "chunks." How you split matters a lot:

- **Fixed-size chunking** — split every N characters or tokens, often with some overlap so you don't cut a sentence in half. Simple, but can slice a thought in two.
- **Recursive/structure-aware chunking** — split along natural boundaries first (sections, then paragraphs, then sentences), falling back to fixed-size only when a piece is still too big. This keeps chunks semantically coherent.
- **Semantic chunking** — group sentences by how similar their embeddings are, so a chunk boundary falls where the topic actually shifts, not at an arbitrary character count.

The tradeoff is always the same: chunks too small lose surrounding context; chunks too large dilute the embedding and waste tokens on irrelevant text. There's no universal "right" chunk size — it depends on your documents and how atomic the ideas in them are.

### 2. Parent-Child Retrieval

A neat way to get the best of both small and large chunks: embed and search over small chunks (good for precise matching), but when one is retrieved, hand the model its larger "parent" section instead of just the tiny snippet. The small chunk finds the needle; the parent chunk gives the model enough surrounding context to actually use it. This is sometimes called "small-to-big" retrieval.

### 3. Hybrid Search

Pure vector (embedding) search is great at matching meaning but sometimes misses exact keywords — like a product code, an error message, or a person's name — because embeddings care about semantic similarity, not exact strings. Keyword search (like BM25) is the opposite: great at exact matches, weak at paraphrases. Hybrid search runs both in parallel and combines the results (often with a weighted score or a fusion method like Reciprocal Rank Fusion), giving you both semantic recall and keyword precision.

### 4. Multi-Query Retrieval

A single user question is often phrased in a way that doesn't match how the answer is phrased in your documents. Multi-query retrieval asks the LLM itself to generate several rephrasings or sub-questions from the original query, runs retrieval for each one, and merges the results. This casts a wider net and catches relevant documents that the original phrasing alone would have missed.

### 5. Context Compression

Retrieved chunks often contain a mix of useful and irrelevant sentences. Context compression filters or summarizes retrieved content down to just what's relevant to the query *before* it goes into the prompt — either by having an LLM extract the relevant sentences, or by using a smaller model to summarize each chunk. This reduces token cost, cuts down on distracting noise, and often improves answer quality because the model isn't wading through filler to find the relevant part.

### 6. Re-ranking

Initial retrieval (vector or hybrid) is optimized for speed, so it casts a wide net — say, the top 20-50 candidate chunks — using a relatively cheap similarity score. A re-ranker is a more expensive but more accurate model (typically a cross-encoder) that looks at the query and each candidate chunk *together* and re-scores them for true relevance. You then keep only the top few after re-ranking. This two-stage approach (fast retrieval, then accurate re-ranking) is the standard way to get both speed and quality.

## A Simple Example

Here's roughly how these pieces fit together in code (using pseudocode close to common libraries like LangChain):

```python
# 1. Chunk documents with a structure-aware splitter
chunks = recursive_text_splitter(documents, chunk_size=500, chunk_overlap=50)

# 2. Store both keyword and vector indexes for hybrid search
vector_index = build_vector_index(chunks)
keyword_index = build_bm25_index(chunks)

def answer_question(user_query):
    # 3. Multi-query: ask the LLM for a couple of rephrasings
    queries = [user_query] + llm_generate_rephrasings(user_query, n=2)

    # 4. Hybrid retrieval for each query variant
    candidates = []
    for q in queries:
        candidates += vector_index.search(q, top_k=10)
        candidates += keyword_index.search(q, top_k=10)
    candidates = deduplicate(candidates)

    # 5. Re-rank candidates with a cross-encoder
    reranked = cross_encoder_rerank(user_query, candidates, top_k=5)

    # 6. Compress: keep only the relevant sentences from each chunk
    compressed_context = [extract_relevant_sentences(user_query, c) for c in reranked]

    # Send compact, high-quality context to the LLM
    return llm_generate_answer(user_query, compressed_context)
```

In production, parent-child retrieval would slot into step 4-5: you'd search over small child chunks but pass along the larger parent chunk once one is selected.

## Key Takeaways / Best Practices

- Start simple — fixed-size or recursive chunking with a basic vector search is a fine baseline. Add complexity (hybrid search, re-ranking, multi-query) only once you can measure that retrieval quality actually needs it.
- Chunk size is a tuning parameter, not a constant — test a few sizes against your real queries and documents rather than guessing.
- Use parent-child retrieval when your documents have natural hierarchy (sections within pages, articles within a manual) and precise snippets need surrounding context to be useful.
- Add hybrid search whenever exact terms (IDs, codes, names) matter to your users — pure semantic search alone will miss them.
- Multi-query retrieval helps most when user questions are phrased casually but your source documents are phrased formally (or vice versa).
- Re-ranking is one of the highest-value, lowest-effort upgrades to a RAG system — it's usually worth adding before you invest in a fancier embedding model.
- Compress context before it hits the prompt — it saves tokens and often improves accuracy by removing distracting irrelevant text.
- Always evaluate retrieval on its own (did we find the right chunks?) separately from evaluating the final generated answer — a great model can't fix bad retrieval, and it's much cheaper to debug the two halves independently.
