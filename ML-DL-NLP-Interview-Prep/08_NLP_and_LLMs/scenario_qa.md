# NLP and LLMs — Scenario-Based Q&A

**Situation:** Your company's internal support chatbot, built on RAG over the employee handbook, confidently tells an employee they get unlimited PTO — the handbook says no such thing. What would you do and why?

Model answer: First, treat this as a grounding failure, not just a "bad model" issue — check whether the relevant handbook section was even retrieved (log the retrieved chunks for this query). If retrieval missed it, fix chunking/embedding or query rewriting. If it was retrieved but ignored, tighten the prompt to instruct the model to answer only from provided context and say "I don't know" otherwise, and consider a lower temperature. Add an automated faithfulness check that flags answers whose claims aren't traceable to retrieved text before they reach the user. Longer term, add this exact question to a regression eval set so it never silently regresses again.

---

**Situation:** You need to decide between fine-tuning a model and just improving the prompt for a new "summarize customer calls" feature, and leadership wants it shipped in a week. What would you do and why?

Model answer: Start with prompt engineering plus few-shot examples on the best available base model — it costs nothing to iterate and can be shipped same-week. Only escalate to fine-tuning (ideally PEFT/LoRA, not full fine-tuning) if prompting plateaus below the required quality bar after reasonable effort, since fine-tuning requires curated labeled data, training infrastructure, and evaluation cycles that don't fit a one-week timeline. Frame it to leadership as "ship the prompt-engineered version now, and treat fine-tuning as a fast-follow if quality metrics don't hit target after real usage data comes in."

---

**Situation:** A multilingual customer base means your tokenizer produces 3-4x more tokens per sentence for Japanese and Arabic input than for English, inflating cost and hitting context-window limits. What would you do and why?

Model answer: This is a tokenizer/vocabulary coverage problem, not a model-capability problem. Check whether the tokenizer (BPE/WordPiece/SentencePiece) was trained on a corpus that under-represents those languages — if so, switch to a model whose tokenizer vocabulary was built on a properly multilingual corpus, or retrain a SentencePiece tokenizer including target-language data. Also audit whether inputs are being needlessly duplicated (e.g., including full conversation history in every call) since that compounds the token-inflation cost regardless of language.

---

**Situation:** You've fine-tuned BERT for support-ticket classification and it performs well, but a stakeholder asks "why not just use GPT-4 with a prompt instead, no training needed?" What would you do and why?

Model answer: Explain the tradeoff concretely: an encoder-only fine-tuned classifier is cheaper per call, faster (single forward pass, no generation), and more consistent for a fixed label set, while a large generative model via prompting is more flexible (handles unseen categories, needs no training data) but costs more per call and adds latency/variance. Recommend keeping the fine-tuned BERT classifier for the stable, high-volume core categories, and reserving the general LLM for edge cases or when the label taxonomy is still evolving.

---

**Situation:** Your RAG system's vector database has grown to 50 million document chunks and query latency has become unacceptable (p95 > 2s). What would you do and why?

Model answer: First profile whether the bottleneck is the ANN search itself or the embedding/generation steps around it. If it's the vector index, move from brute-force/flat search to an approximate index (e.g., HNSW or IVF-based) that trades a small recall loss for large speed gains, and consider sharding the index. Also reduce candidate volume upstream with metadata filtering (e.g., restrict to relevant document category before vector search) and consider a smaller/faster embedding model for the query side if quality allows. Re-measure recall after any ANN change since approximate search can silently degrade answer quality.

---

**Situation:** Leadership wants to know, in numbers, whether your LLM feature is "hallucinating too much" before a wider rollout. What would you do and why?

Model answer: Build a labeled evaluation set that separates answerable-from-context questions from genuinely unanswerable ones, then measure two numbers: the faithfulness rate (fraction of claims in an answer that are supported by retrieved context) on answerable questions, and the correct-abstention rate (fraction of unanswerable questions where the model says it doesn't know) on unanswerable ones. Track both over time as a regression suite, and supplement with a sample of human-reviewed production transcripts weekly, since automated LLM-as-judge scoring is a proxy, not ground truth, and needs periodic calibration against human judgment.

---

**Situation:** You're asked to add semantic search to an e-commerce site that currently uses only keyword (BM25) search, and the search team is worried semantic search will hurt exact-match queries like SKU numbers or exact model names. What would you do and why?

Model answer: Don't replace BM25 — combine them. Use hybrid retrieval: run both BM25 and dense embedding search, then merge results (e.g., reciprocal rank fusion or a learned re-ranker). This preserves BM25's strength on exact tokens/SKUs/rare identifiers while gaining embedding search's strength on paraphrased or descriptive queries ("comfortable running shoes for flat feet"). Validate with an A/B test measuring both click-through rate and zero-result-rate, since the goal is measurably fewer "no results" pages, not just architectural elegance.

---

**Situation:** A team wants to fine-tune a 70B parameter open-source model on a single 80GB GPU for a niche legal-document task and asks if that's feasible. What would you do and why?

Model answer: Full fine-tuning of a 70B model needs far more memory than one 80GB GPU can hold (parameters, gradients, optimizer states). Recommend LoRA or QLoRA instead: freeze the base weights, quantize them to 4-bit to fit in memory, and train small low-rank adapter matrices on top — this is exactly the regime PEFT methods were designed for and commonly fits on a single high-memory GPU. Set expectations that this gets close to full fine-tuning quality for a narrow task, not necessarily matching it, and validate against a held-out legal-document test set before committing further budget.
