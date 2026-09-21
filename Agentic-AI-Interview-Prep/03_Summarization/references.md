# Summarization — References

All links below were fetched and content-verified this session (2026-09-18) unless explicitly marked otherwise.

## Papers

- Lin, C-Y., "ROUGE: A Package for Automatic Evaluation of Summaries" (ACL Workshop, 2004) — defines ROUGE-N/L/W/S. https://aclanthology.org/W04-1013/
- Zhang, Kishore, Wu, Weinberger, Artzi, "BERTScore: Evaluating Text Generation with BERT" (ICLR 2020). https://arxiv.org/abs/1904.09675 (code: github.com/Tiiiger/bert_score)
- Maynez, Narayan, Bohnet, McDonald, "On Faithfulness and Factuality in Abstractive Summarization" (ACL 2020) — the seminal large-scale human-evaluation study on hallucination types (intrinsic vs. extrinsic) in neural summarizers; found hallucination present across essentially all systems tested, including strong pretrained models. https://arxiv.org/abs/2005.00661
- Liu, Iter, Xu, Wang, Xu, Zhu, "G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment" (EMNLP 2023) — the widely-cited LLM-as-judge evaluation approach; reports substantially better correlation with human judgment on summarization than prior reference-based metrics. https://arxiv.org/abs/2303.16634
- Zhong et al., "QMSum: A New Benchmark for Query-based Multi-domain Meeting Summarization" (NAACL 2021) — 1,808 query-summary pairs across 232 meetings, the standard reference point for query-focused summarization. https://arxiv.org/abs/2104.05938
- "Hallucinate at the Last in Long Response Generation: A Case Study on Long Document Summarization" (2025) — documents that hallucination rate increases specifically toward the end of long generated summaries, relevant to long-context/single-pass summarization risk. (not URL-verified this session — surfaced via search only; arXiv id 2505.15291, re-check before citing verbatim)
- "Summarization is Not Dead Yet" (2026) — multi-track evaluation across 5 datasets and 5 SOTA LLMs finding human references still win on informativeness/faithfulness while LLMs win on fluency/coherence, with notable stylistic homogenization across models; a useful counterpoint to older "summarization is solved" claims. (not URL-verified this session — surfaced via search only; arXiv id 2606.08000, re-check before citing verbatim)

## Practitioner Resources

- Eugene Yan, "Evaluation & Hallucination Detection for Abstractive Summaries" — substantive practitioner writeup covering reference-based (ROUGE/BERTScore/MoverScore), NLI-based (SummaC), QA-based (QuestEval/QAFactEval), and LLM-judge (G-Eval) approaches to summary evaluation. https://eugeneyan.com/writing/abstractive/

## Framework Notes

- LangChain's map-reduce/refine summarization chain pattern (`load_summarize_chain` with `chain_type="map_reduce"|"refine"|"stuff"`) is a widely referenced implementation pattern, but as of this session LangChain's own docs have moved away from a dedicated summarization tutorial toward a RAG/agentic-delegation framing (a "retrieve, offload, delegate" pattern using parallel subagents rather than manual map-reduce chains) — treat the map-reduce/refine chain-type API as the conceptual pattern to know, not assume a stable current first-party tutorial URL exists for it. (not URL-verified this session for a current official tutorial link; the concept and API names were confirmed via multiple secondary sources)

## Notes on What to Prioritize

There isn't one canonical "text summarization system design interview" article specific to LLMs — most generic hits are older, non-LLM system-design framings. Build interview readiness from the metric papers (know what ROUGE/BERTScore actually measure and why they under-serve abstractive summaries), the Maynez faithfulness paper (the reference point for "hallucination in summarization" as a named, studied phenomenon with an intrinsic/extrinsic taxonomy), and G-Eval (the reference point for "why we use LLM-as-judge now"). Be ready to discuss the nuanced, not absolute, relationship between growing context windows and the continued relevance of map-reduce/refine strategies — this is a good "shows depth" question in a senior-level interview.
