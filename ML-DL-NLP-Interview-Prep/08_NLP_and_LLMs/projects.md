# NLP and LLMs — Projects

## Small: Document Q&A over a Personal Knowledge Base

Build a RAG pipeline over 20-50 PDFs or Markdown notes (e.g., your own study notes, or a set of public research papers on one topic). Chunk the text, embed with a sentence-transformer model, store in FAISS or Chroma, and wrap retrieval + generation behind a simple CLI or Streamlit UI that answers questions with citations back to source chunks. This proves you can build the standard retrieval-then-generation pipeline end to end, including the unglamorous parts (chunking strategy, prompt construction) that most tutorials skip.

## Medium: Domain-Adapted Support Assistant with PEFT

Take an open dataset of support tickets or FAQs (e.g., a public customer-support dataset, or scrape a product's public help center with permission) and build a two-part system: (1) a RAG layer that retrieves the relevant policy/FAQ, and (2) a LoRA-fine-tuned small open LLM that adapts tone/format to match the company's actual support responses. Evaluate hallucination rate on a held-out set of questions with no answer in the knowledge base (the model should say "I don't know"). This proves you understand both retrieval grounding and lightweight fine-tuning, and that you evaluate for hallucination rather than just eyeballing outputs.

## Large: Multi-Source Semantic Search Engine with Evaluation Harness

Build a semantic search service over a large, messy multi-format corpus (e.g., a mix of PDFs, HTML pages, and Slack/forum exports — 1,000+ documents). Include: a chunking pipeline tuned per document type, a hybrid retriever (BM25 keyword search + dense embedding search, combined via reciprocal rank fusion or a re-ranker), a vector DB deployed as a real service (not just in-memory FAISS), and an automated evaluation harness that scores retrieval precision/recall against a hand-labeled query set plus an LLM-as-judge faithfulness score for generated answers. This proves you can design production-grade retrieval (not just a notebook demo), and that you know how to measure quality quantitatively rather than by inspection — the single biggest gap between demo-level and production-level RAG work.
