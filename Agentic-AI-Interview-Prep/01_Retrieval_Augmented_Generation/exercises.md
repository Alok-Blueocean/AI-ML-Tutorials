# RAG — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual questions.

1. **Build a minimal RAG pipeline.** Take 20 PDFs (research papers, or your own notes), chunk them with a fixed-size splitter (500 tokens, 50 overlap), embed with a sentence-transformer model, index with FAISS or Chroma, and answer 10 test questions by retrieving top-5 chunks and passing them to an LLM. Log the retrieved chunks for each answer.

2. **Conceptual: chunk size tradeoff.** Explain, without running anything, why doubling chunk size can both fix and cause failures in the same pipeline. Give one concrete example of each direction.

3. **Compare chunking strategies.** Re-run exercise 1's pipeline with three chunking strategies — fixed-size, recursive/structure-aware, and semantic (embedding-similarity based) — on the same 20 PDFs. Compare retrieval quality on the same 10 questions and explain the differences you observe.

4. **Add hybrid search.** Add a BM25 index alongside your vector index from exercise 1, and combine results using Reciprocal Rank Fusion. Find at least 2 queries where BM25 retrieves something the dense retriever missed (e.g., an exact term/code) and 2 where dense retrieval wins.

5. **Conceptual: bi-encoder vs cross-encoder.** Explain why a cross-encoder is more accurate than a bi-encoder for scoring query-document relevance, and why you can't just use a cross-encoder for the entire first-pass retrieval over a large corpus.

6. **Add a reranker.** Add a cross-encoder reranking step to your pipeline: retrieve top-30 candidates, rerank, keep top-5. Measure whether answer quality improves on your 10 test questions, and note the added latency.

7. **Evaluate with RAGAS.** Build a small evaluation dataset (question, ground-truth answer, ground-truth relevant chunk) for 15 questions against your corpus. Run RAGAS (or implement the four metrics — context precision, context recall, faithfulness, answer relevancy — yourself using an LLM judge) and report scores per question.

8. **Conceptual: diagnosing "hallucination" complaints.** Given a RAG system that users say is "hallucinating," describe how you would use context recall vs. faithfulness scores to distinguish between a retrieval-miss bug and a true generation-side hallucination, before touching any prompt.

9. **Implement HyDE.** Add a HyDE step to your pipeline: for each query, have the LLM generate a hypothetical answer, embed that instead of the raw query, and retrieve. Compare retrieval quality against plain query embedding on 10 vague/underspecified questions.

10. **Build a query-rewriting / multi-query step.** For a set of short, ambiguous user queries, have the LLM generate 3 rephrasings/sub-questions, retrieve for each, and merge/deduplicate results. Show at least 2 cases where this recovers a relevant chunk the original query missed.

11. **Parent-child (small-to-big) retrieval.** Modify your pipeline so that small chunks are used for matching but the larger parent section is what's passed to the LLM. Compare answer completeness against plain small-chunk retrieval on questions that need surrounding context.

12. **Simulate a stale-index failure.** Update one source document (e.g., change a policy value), but don't re-index. Show the system confidently giving the old answer. Then implement a simple re-indexing trigger and demonstrate the fix.

13. **Conceptual: GraphRAG vs vector RAG.** Describe a question type where flat vector RAG structurally cannot succeed regardless of chunking/reranking quality, and explain at a high level how a graph/community-based approach addresses it.

14. **Multi-hop retrieval.** Construct 5 questions that require two sequential retrieval steps (the second query depends on the first retrieval's result). Build a simple agentic loop that retrieves, checks sufficiency, and retrieves again if needed. Compare against single-shot retrieval on these 5 questions.

15. **End-to-end system critique.** Given a described production RAG chatbot answering questions from a 500-page manual with ~10% hallucination rate reported by users, propose a prioritized list of 5 concrete interventions (indexing/chunking, retrieval, reranking, prompting, evaluation) and justify the order using the metrics from exercise 7.
