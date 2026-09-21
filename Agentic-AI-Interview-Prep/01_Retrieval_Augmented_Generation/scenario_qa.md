# RAG — Scenario-Based Q&A

**Situation:** A support chatbot built on RAG over the employee handbook confidently tells an employee they have unlimited PTO — the handbook says no such thing. What would you do and why?

Model answer: Treat it as a grounding/retrieval failure first, not "the model is broken." Log and inspect the retrieved chunks for this exact query. If the relevant PTO clause was never retrieved, the bug is in chunking/embedding/query formulation (check whether the clause and its exception were split across a chunk boundary). If it was retrieved but the model ignored it, tighten the prompt's grounding instruction ("answer only from the provided context, say you don't know otherwise"), lower temperature, and add a faithfulness check to the eval pipeline. Add this exact question to a permanent regression set so it can't silently regress again.

---

**Situation:** After a routine document-management migration, a policy PDF was reformatted (new headers, reflowed paragraphs), and your RAG system's retrieval quality on policy questions dropped sharply overnight with no code changes. What would you do and why?

Model answer: This is a symptom of chunking being tied to document structure — a format change silently reshuffled paragraph/section boundaries that your chunker relied on, producing chunks that no longer align with coherent ideas (or duplicate/split content differently than before). First diff the chunk outputs before and after the migration for a few known documents to confirm this theory. Fix by making the chunker more robust to structural variance (recursive/structure-aware splitting rather than pure fixed-size-by-character-offset) and by adding a canary test that re-chunks a fixed reference document and flags unexpected changes in chunk count/boundaries whenever the ingestion pipeline runs.

---

**Situation:** Your vector index has grown to 50 million chunks and p95 query latency has crossed 2 seconds, and the business wants this fixed without a quality regression. What would you do and why?

Model answer: Profile first — determine whether the bottleneck is the ANN search itself, the embedding call, or the reranking/generation step. If it's the index, move from a flat/brute-force index to an approximate index (HNSW or IVF-based), and consider sharding. Reduce candidate volume upstream with metadata filtering (restrict to the relevant document category/tenant before vector search) rather than searching the full 50M every time. Re-measure recall after any ANN change, since approximate search trades a small, sometimes silent, recall loss for speed — that tradeoff needs to be measured, not assumed acceptable.

---

**Situation:** Leadership asks, in hard numbers, whether the RAG assistant is "hallucinating too much" before a wider rollout. What would you do and why?

Model answer: Build a labeled evaluation set separating answerable-from-context questions from genuinely unanswerable ones. Measure faithfulness (fraction of claims in the answer supported by retrieved context) on the answerable set, and correct-abstention rate (the model saying "I don't know" appropriately) on the unanswerable set. Track both as a regression suite over time, and supplement with periodic human review of sampled production transcripts, because LLM-as-judge scoring (as used in RAGAS-style faithfulness scoring) is a useful proxy but needs periodic calibration against human judgment, not blind trust.

---

**Situation:** The search team is worried that adding semantic (embedding) search to an e-commerce site currently running only BM25 keyword search will hurt exact-match queries like SKU numbers or exact model names. What would you do and why?

Model answer: Don't replace BM25 — combine them with hybrid search. Run both retrievers and fuse results (Reciprocal Rank Fusion, or a trained reranker over the union of both candidate sets). This keeps BM25's strength on exact tokens/SKUs while gaining dense retrieval's strength on descriptive/paraphrased queries. Validate with an A/B test measuring click-through rate and zero-result rate specifically, since the actual goal is fewer "no results" pages without regressing exact-match precision — not architectural elegance for its own sake.

---

**Situation:** A user asks a "holistic" question — "what are the recurring themes across all 300 customer feedback submissions this quarter?" — and your standard top-k vector RAG gives a shallow, incomplete answer that only reflects 3-4 of the submissions. What would you do and why?

Model answer: Recognize this as a structural limitation of top-k similarity retrieval, not a prompting problem — no single chunk contains "all recurring themes," so no top-k retrieval, however well-tuned, will surface a complete answer. Reach for a summarization/map-reduce pass over the full corpus for aggregate questions, or a GraphRAG-style approach that builds entity/theme clusters ahead of time so the question can be answered by traversing precomputed summaries rather than searching for the single most similar chunk. Set the expectation up front that plain RAG is optimized for localized fact lookup, not corpus-wide aggregation.

---

**Situation:** Your team ships a RAG chatbot that performs well in testing, but two weeks after launch the embeddings team silently upgrades the embedding model in a shared internal service to improve benchmark scores, and your retrieval quality collapses. What would you do and why?

Model answer: Diagnose first: an embedding model swap without re-indexing means query vectors and document vectors now come from incompatible spaces, so similarity scores become close to meaningless even though nothing about your code changed. Immediately re-index the full corpus with the new model version, or pin your service to a specific embedding model version until you can coordinate a planned migration. Long term, treat the embedding model as a versioned dependency with an explicit contract (version pin, deprecation notice, coordinated re-index window) rather than a transparent shared service that can change under you.

---

**Situation:** A cross-encoder reranker you added improved offline relevance metrics but a stakeholder is asking why response latency went up by 300ms and whether it's worth it. What would you do and why?

Model answer: Quantify the tradeoff concretely rather than defending the change on principle: report the measured lift in context precision/faithfulness (or downstream task success) against the added latency, and check whether the latency budget actually matters for this use case (a batch/async report-generation feature can absorb 300ms easily; a live chat UI may not). If latency is the binding constraint, consider reranking only when the initial retrieval's top results are ambiguous (e.g., close similarity scores), using a smaller/distilled cross-encoder, or reducing the number of candidates sent to the reranker, rather than dropping reranking outright.

---

**Situation:** An engineer proposes retrieving the top-20 chunks instead of top-5 "to be safe" and stuffing all of them into the prompt, reasoning that more context can only help. What would you do and why?

Model answer: Push back with the "lost in the middle" effect and the noise-dilution argument: irrelevant chunks increase token cost and latency, and empirically make models more likely to miss or misweight the actually relevant chunk buried among them, in addition to increasing hallucination surface area. Recommend keeping retrieval wide (e.g., top-30) but reranking down to a small, high-precision set (e.g., top-5) before generation, and validate the choice of cutoff against context precision/recall on a labeled eval set rather than intuition about "more is safer."
