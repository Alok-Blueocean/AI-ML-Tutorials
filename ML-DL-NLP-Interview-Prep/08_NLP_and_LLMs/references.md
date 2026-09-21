# NLP and LLMs — References

All links below were checked this session (fetched and content verified against the claim) unless explicitly marked otherwise.

## Papers

- Mikolov et al., "Efficient Estimation of Word Representations in Vector Space" (2013) — the Word2Vec paper (CBOW + skip-gram). https://arxiv.org/abs/1301.3781
- Devlin et al., "BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding" (2018). https://arxiv.org/abs/1810.04805
- Brown et al., "Language Models are Few-Shot Learners" (2020) — the GPT-3 paper. https://arxiv.org/abs/2005.14165
- Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (2020) — the original RAG paper. https://arxiv.org/abs/2005.11401
- Hu et al., "LoRA: Low-Rank Adaptation of Large Language Models" (2021). https://arxiv.org/abs/2106.09685
- Vaswani et al., "Attention Is All You Need" (2017) — the Transformer paper; background for everything above. (not URL-verified this session — well-known arXiv id 1706.03762, use with caution and re-check before sharing).

## Official Docs / Courses

- Hugging Face LLM/NLP Course (free, official) — covers tokenizers, transformers, fine-tuning, and a semantic-search-with-FAISS chapter. https://huggingface.co/learn/llm-course/en/chapter1/1
- Hugging Face Course source repository (GitHub). https://github.com/huggingface/course

## GitHub Repositories

- `huggingface/course` — official course notebooks/materials for the Hugging Face ecosystem (Transformers, Tokenizers, Datasets). https://github.com/huggingface/course
- Example FAISS + sentence-transformers semantic search build (good template for the RAG/semantic-search exercises): `kstathou/vector_engine`, "Build a semantic search engine with Transformers and Faiss." https://github.com/kstathou/vector_engine (not URL-verified this session — surfaced via search, smaller/less-established repo; check star count and last-commit date before relying on it, or substitute the official Hugging Face FAISS chapter above, which is verified).

## Articles / Interview Prep

- DataCamp, "Top LLM Interview Questions and Answers" — broad coverage of transformer architecture, fine-tuning (LoRA/QLoRA/PEFT), RAG, prompt engineering, hallucination. https://www.datacamp.com/blog/llm-interview-questions
- Devinterview-io, "LLMs interview questions and answers" (GitHub, community-maintained, actively updated). https://github.com/Devinterview-io/llms-interview-questions
- Analytics Vidhya, "RAG Interview: Questions to Go from Beginner to Advanced." https://www.analyticsvidhya.com/blog/2026/02/rag-interview-questions-and-answers/

## Notes on What to Prioritize

Interview signal consistently points to: transformer architecture differences (encoder vs decoder, bidirectional vs causal attention), tokenization (why subword, not word-level), fine-tuning vs PEFT vs prompting tradeoffs, and RAG end-to-end (including failure modes and hallucination mitigation). Prioritize being able to whiteboard the RAG pipeline and explain LoRA's low-rank decomposition intuition — both come up repeatedly in applied-AI/LLM engineer interviews.
