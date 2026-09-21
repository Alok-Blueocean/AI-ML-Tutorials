# RAG Systems Deep Dive — Scenario-Based Q&A

**Situation:** Your RAG demo worked great with a handful of test questions, but once real users started asking questions in their own words, answer quality dropped noticeably even though nothing about the model or the documents changed. What would you do and why?

Model answer: Suspect a retrieval problem before a generation problem — most "the AI gave a wrong answer" complaints in production RAG trace back to retrieval finding the wrong chunks, not the model reasoning badly over good context. Start by logging and inspecting the actual retrieved chunks for a sample of the new, more varied real-world questions, and check whether the phrasing gap between casual user questions and formally-written source documents is causing the vector search to miss relevant content. If so, multi-query retrieval — having the LLM generate a few rephrasings of the user's question before retrieving — is the most direct fix, since it casts a wider net that catches documents the original casual phrasing alone would miss.

---

**Situation:** A teammate proposes fixing a "wrong answer" complaint by switching to a more expensive, higher-quality LLM for generation. What would you check first, and why?

Model answer: Check the retrieved chunks for that exact question before spending money on a bigger model. If retrieval never surfaced the relevant information in the first place, no amount of generation-side model quality can fix the answer — the model can only work with what it's given, and a more expensive model reasoning well over the wrong context still produces a wrong answer. Evaluate retrieval on its own (did we find the right chunks for this question?) separately from evaluating the final answer, since it's much cheaper to debug and fix the two halves independently than to guess which one is broken.

---

**Situation:** Your documents are long product manuals, and you chunked them at a fixed 300 characters per chunk to keep things simple. Now searches for specific facts return chunks that cut off mid-sentence, and the model sometimes answers based on half a thought. What would you do and why?

Model answer: Move from fixed-size chunking to recursive/structure-aware chunking, which splits along natural boundaries first — sections, then paragraphs, then sentences — and only falls back to fixed-size splitting when a piece is still too large. This keeps each chunk a coherent unit of meaning instead of an arbitrary character-count slice that can cut a sentence or a table row in half. Also add some overlap between chunks regardless of strategy, so a fact that sits right at a natural boundary still has a reasonable chance of appearing whole in at least one chunk.

---

**Situation:** A user asks "what are the recurring themes across all of last quarter's 300 customer feedback submissions," and your standard top-k RAG setup gives a shallow answer that clearly only reflects 3-4 of the submissions. What would you do and why?

Model answer: Recognize this as a structural mismatch between the question and what plain top-k similarity retrieval can do, not a tuning problem — no single top-k retrieval, however well-optimized, will surface "all recurring themes" because no handful of chunks contains the full aggregate picture across 300 documents. Set expectations that plain RAG is built for localized fact lookup, not corpus-wide aggregation, and reach for a different pattern for this specific question type: a summarization/map-reduce pass across the full corpus, or a structure that pre-computes theme clusters so the question can be answered by traversing summaries rather than searching for the single most similar chunk.

---

**Situation:** Your e-commerce search team is worried that adding semantic (embedding) search will hurt exact-match queries like SKU numbers or specific model names that currently work fine with keyword search. What would you do and why?

Model answer: Don't replace keyword search — add hybrid search, running both vector and keyword (BM25) retrieval in parallel and combining the results, often with a fusion method like Reciprocal Rank Fusion. This preserves BM25's strength on exact tokens like SKUs and model names, which embeddings are weak at because they represent meaning rather than exact strings, while gaining the embedding search's strength on descriptive or paraphrased queries. Validate the change with a test set that specifically includes exact-match queries, so you can confirm precision on those didn't regress before rolling hybrid search out broadly.

---

**Situation:** You chunked documents small (roughly 150 characters) to get precise, highly targeted matches during retrieval, but now the model's answers feel thin and miss context a human would obviously have used from the surrounding paragraph. What would you do and why?

Model answer: This is the classic small-chunk tradeoff — small chunks are good at precise matching but lose the surrounding context needed to actually use the matched fact well. Use parent-child ("small-to-big") retrieval: keep searching over the small chunks for precise matching, but when one is retrieved, hand the model the larger parent section it belongs to instead of just the tiny snippet. This gets you both the precision of small-chunk search and enough surrounding context for the model to produce a complete, well-grounded answer.

---

**Situation:** Your RAG pipeline retrieves the top 5 chunks and stuffs all of them directly into the prompt, and you notice the context window filling up with a lot of tangential sentences alongside the actually-relevant ones, driving up token cost. What would you do and why?

Model answer: Add a context compression step between retrieval and generation — filter or summarize each retrieved chunk down to just the sentences relevant to the specific query before it goes into the prompt, either by having an LLM extract the relevant sentences or using a smaller model to summarize each chunk. This directly cuts token cost, reduces distracting noise the model has to wade through, and often improves answer quality as a side effect, since the model isn't forced to find the needle in a chunk full of irrelevant filler.

---

**Situation:** A stakeholder asks why the team is adding a re-ranking step when the vector search already returns results sorted by similarity score — isn't that already a ranking?

Model answer: Explain the two-stage tradeoff: initial retrieval (vector or hybrid) is optimized for speed, so it uses a relatively cheap similarity score to cast a wide net over dozens of candidates, but cheap similarity scoring is not the same as true relevance. A re-ranker, typically a cross-encoder, looks at the query and each candidate chunk together and re-scores them for actual relevance — something the fast first-pass retrieval structurally can't do at scale. Keeping only the top few results after re-ranking gets you both the speed of a wide, cheap first pass and the accuracy of an expensive, precise second pass, which is why re-ranking is often the single highest-value, lowest-effort upgrade to a RAG system.

---

**Situation:** Your team wants to make big improvements to RAG quality and someone proposes jumping straight to semantic chunking, parent-child retrieval, hybrid search, multi-query retrieval, context compression, and re-ranking all at once, in the first sprint. What would you push back on?

Model answer: Push back on adding all six techniques before measuring which one the system actually needs — start simple with fixed-size or recursive chunking and basic vector search as a baseline, and add complexity only once you can measure that retrieval quality actually needs it. Without a baseline and a retrieval-quality measurement in between changes, if quality improves after shipping all six at once, you won't know which change mattered, and if something regresses, you won't know which change caused it. Recommend adding techniques one at a time, in the order the team's actual failure pattern suggests (e.g., re-ranking first, since it's usually the highest-value, lowest-effort win), measuring retrieval quality after each.

---

**Situation:** A document management migration reformats your policy PDFs — new headers, reflowed paragraphs — with no code changes on your side, and retrieval quality on policy questions drops sharply overnight. What would you do and why?

Model answer: Recognize this as evidence that chunking is tied to document structure — a format change silently reshuffled paragraph and section boundaries the chunker relied on, producing chunks that no longer align with coherent ideas the way they did before. Diff the chunk outputs before and after the migration for a few known documents to confirm this. Make the chunker more robust to structural variance by leaning on recursive/structure-aware splitting rather than pure fixed-size-by-character-offset, and add a canary check that re-chunks a fixed reference document on a schedule and flags unexpected changes in chunk count or boundaries whenever the ingestion pipeline runs.

---

**Situation:** Your multi-query retrieval step asks the LLM to generate three rephrasings of every user question before retrieving, and someone notices this has roughly quadrupled retrieval latency and cost for every single query, including simple ones that were already answered correctly before. What would you do and why?

Model answer: Point out that multi-query retrieval helps most specifically when casual user phrasing diverges from formal document phrasing, and applying it unconditionally to every query — including ones the original single-query retrieval already handled fine — pays that 4x cost for no benefit on a large fraction of traffic. Consider gating multi-query generation behind a signal that the first-pass retrieval looks weak (e.g., low top-result similarity scores) so the expensive rephrasing path only triggers when it's likely to actually help, rather than running it unconditionally on every request.

---

**Situation:** You've added re-ranking to your pipeline, and offline evaluation shows a clear relevance improvement, but a stakeholder is asking whether the added latency from the cross-encoder step is worth it for a live chat interface with tight response-time expectations.

Model answer: Quantify both sides of the tradeoff explicitly rather than defending re-ranking on principle — measure the actual latency added by the cross-encoder step and weigh it against the measured relevance/quality lift, and check whether the product's latency budget can absorb it (a live chat UI has much less room than an async report-generation feature). If latency is the binding constraint, consider re-ranking only when the initial retrieval's top candidates have closely clustered similarity scores (a signal the fast first pass is genuinely uncertain), using a smaller/distilled cross-encoder, or reducing how many candidates get sent to the reranker, rather than dropping the accuracy benefit of re-ranking entirely.

---

**Situation:** A new engineer on the team asks why RAG quality evaluation should be split into "did we retrieve the right chunks" and "was the final answer good," instead of just checking whether the final answer is correct.

Model answer: Explain that a RAG system can fail in two structurally different places, and a single end-to-end correctness check can't tell you which one happened. If retrieval never surfaced the relevant chunk, no generation-side fix will help — the model literally never saw the needed information. If retrieval found the right chunk but the model still answered incorrectly, that's a generation/grounding problem, and the fix is entirely different (prompt tightening, stricter grounding instructions). Evaluating the two halves separately turns "the answer was wrong, no idea why" into a specific, actionable diagnosis, and it's much cheaper to fix the right half of the pipeline than to guess.

---

**Situation:** Your knowledge base has grown from a few hundred documents to tens of thousands, and query latency has crept up noticeably even though you haven't changed the retrieval pipeline's logic at all. What would you check, and what would you consider changing?

Model answer: Check whether the vector index itself has become the bottleneck now that the corpus is much larger — a brute-force/flat similarity search that was fast enough at a few hundred documents can become the dominant cost at tens of thousands or more. Consider moving to an approximate nearest-neighbor index (like HNSW or IVF-based indexing) to keep search fast at scale, and where possible, narrow the search space upstream with metadata filtering (restricting to the relevant category or source before running vector search) rather than searching the entire growing corpus on every query. Re-measure retrieval quality after any approximate-index change, since approximate search trades a small amount of recall for speed, and that tradeoff should be measured rather than assumed acceptable.
