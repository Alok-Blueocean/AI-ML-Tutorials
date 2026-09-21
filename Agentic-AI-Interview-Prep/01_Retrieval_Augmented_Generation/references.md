# RAG — References

All links below were fetched and content-verified this session (2026-09-18) unless explicitly marked otherwise.

## Papers

- Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (NeurIPS 2020) — the original RAG paper. https://arxiv.org/abs/2005.11401
- Gao, Ma, Lin, Callan, "Precise Zero-Shot Dense Retrieval without Relevance Labels" (ACL 2023) — the HyDE paper. https://arxiv.org/abs/2212.10496 (code: https://github.com/texttron/hyde)
- Edge et al., GraphRAG paper (Microsoft Research), background for the GraphRAG project. https://arxiv.org/pdf/2404.16130

## Frameworks and Docs

- RAGAS — RAG evaluation framework (context precision/recall, faithfulness, answer relevancy). Actively maintained, ~15.8k GitHub stars at check time. https://github.com/explodinggradients/ragas and docs at https://docs.ragas.io
- DeepEval — a fast-rising alternative/complement to RAGAS as of 2026: pytest-native, broader metric library (agents, multi-turn, safety), commonly used alongside RAGAS rather than instead of it (RAGAS for dataset-level tuning, DeepEval for CI regression gates). (not URL-verified this session — flagged by research pass as a real, current tool; verify `github.com/confident-ai/deepeval` before citing a specific star count).
- Microsoft GraphRAG — modular graph-based RAG system, actively maintained. https://github.com/microsoft/graphrag and docs at https://microsoft.github.io/graphrag/
- LlamaIndex — official RAG/indexing docs. Note: URL structure moved in 2025-2026 from `docs.llamaindex.ai` to the current site. https://developers.llamaindex.ai/python/framework/understanding/rag/
- LangChain — official RAG docs. Note: URL structure moved to a unified docs site and the framing has shifted toward agentic/"Deep Agents" retrieval patterns rather than a single simple retrieval chain. https://docs.langchain.com/oss/python/langchain/rag
- Sentence-Transformers, "Retrieve & Rerank" (bi-encoder + cross-encoder two-stage pattern). https://sbert.net/examples/sentence_transformer/applications/retrieve_rerank/README.html
- Cohere Rerank API docs. https://docs.cohere.com/docs/rerank-overview (model versions cycle quickly — confirm the current model name, e.g. rerank-v3.5, before an interview since older versions get marked deprecated on short notice)
- BAAI bge-reranker-v2-m3 — open multilingual cross-encoder reranker model card. https://huggingface.co/BAAI/bge-reranker-v2-m3
- Elasticsearch RRF (Reciprocal Rank Fusion) retriever docs — official reference for the fusion formula used in hybrid search. https://www.elastic.co/docs/reference/elasticsearch/rest-apis/retrievers/rrf-retriever

## Articles / Interview Prep

- Analytics Vidhya, "RAG Interview: 40 Questions to Go from Beginner to Advanced" — tiered coverage of chunking, BM25 vs dense vs hybrid, retrieval metrics (Recall@k, Precision@k, MRR, nDCG), reranking, agentic RAG, and production failure modes. https://www.analyticsvidhya.com/blog/2026/02/rag-interview-questions-and-answers/
- ActiveWizards, "The RAG Failure Taxonomy: 12 Ways Production Retrieval Pipelines Break" — source for several failure-mode examples used in this kit's tutorial and scenario files (stale index, chunk-boundary information loss, retrieval-generation mismatch). https://activewizards.com/blog/the-rag-failure-taxonomy-12-ways-production-retrieval-pipelines-break/
- "Chunking Strategies in RAG Systems: Insights from 80+ GenAI Interviews" (levelup.gitconnected.com). (not URL-verified this session — surfaced via search only, not independently fetched; check before citing.)

## Notes on What to Prioritize

Interview signal consistently points to: the two-stage retrieve-then-rerank pattern, chunking strategy tradeoffs (and being able to name a concrete chunk-boundary failure), hybrid search and why RRF/fusion matters more than "just add BM25," and being able to explain the difference between a retrieval failure and a generation failure using context recall vs. faithfulness rather than "the model hallucinated." Know RAGAS metric names cold, and be ready to discuss GraphRAG/agentic RAG for "holistic" or multi-hop questions that plain top-k retrieval structurally cannot answer. A notable 2025-2026 shift: both LangChain and LlamaIndex have reframed their RAG documentation around agentic patterns rather than a single fixed retrieval chain — reflects the field's move toward retrieval-as-a-tool-within-an-agent-loop rather than a standalone pipeline.
