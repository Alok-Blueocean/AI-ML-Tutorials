# RAG — Projects

## Small: Cited Document Q&A over a Personal Knowledge Base

Build a RAG pipeline over 20-50 PDFs or Markdown notes (your own study notes, or a set of public papers on one topic). Chunk with a structure-aware splitter, embed with a sentence-transformer model, store in FAISS or Chroma, and wrap retrieval + generation behind a CLI or Streamlit UI that always cites which source chunk each claim came from. This proves you can build the standard pipeline end to end, including the unglamorous parts (chunking, prompt construction, citation formatting) most tutorials skip.

## Medium: Hybrid-Search RAG with a RAGAS Evaluation Harness

Take a moderately messy real corpus (a product's public documentation, a set of SEC filings, or a company wiki export) and build a hybrid retriever (BM25 + dense embeddings fused with RRF) plus a cross-encoder reranking stage. Build a 30-50 question evaluation set with ground-truth answers and relevant chunks, and run it through RAGAS (context precision, context recall, faithfulness, answer relevancy) after every pipeline change. This proves you evaluate retrieval and generation quality separately and quantitatively, rather than by inspection — the single biggest gap between demo-level and production-level RAG work.

## Medium: Stale-Index Detection and Auto-Refresh Pipeline

Build a RAG system over a document source that changes over time (a public changelog, a wiki, or a folder you edit yourself), with an automated re-indexing pipeline triggered on document change (file-hash diffing or a webhook), plus a monitoring check that flags when the index's last-refresh timestamp is older than a threshold relative to the source's last-modified time. This proves you think about RAG as a live system with a staleness failure mode, not a one-time notebook build.

## Large: Multi-Hop Agentic RAG with GraphRAG Comparison

Build a question-answering system over a corpus where many realistic questions require multi-hop reasoning (e.g., a dataset of interlinked Wikipedia articles, or a company's interconnected internal docs). Implement two approaches on the same corpus: (1) an agentic RAG loop that plans, retrieves, checks sufficiency, and retrieves again; and (2) a GraphRAG-style pipeline that extracts entities/relationships into a graph and answers via graph traversal or community summarization. Evaluate both on a held-out set of multi-hop and "holistic" (whole-corpus-theme) questions, and report where each approach wins. This proves you understand the limits of naive top-k retrieval and can reason about architecture choice, not just implement a single fixed pipeline.
