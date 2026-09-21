# Summarization — Projects

## Small: Multi-Length, Cited Article Summarizer

Build a tool that takes a news article or blog post URL and generates summaries at three controllable lengths (one sentence, one paragraph, bullet-point brief), with each generated claim tagged back to the source sentence it came from. Evaluate on 15-20 articles using both ROUGE/BERTScore (for a cheap sanity baseline) and an LLM-as-judge faithfulness score. This proves you understand length control, citation/traceability, and the gap between reference-based and LLM-judge evaluation.

## Medium: Meeting-Transcript Summarizer with Query Focus

Build a summarizer over meeting transcripts (use a public meeting-transcript dataset, or record and transcribe your own team meetings with consent) that produces different query-focused summaries on demand — action items, decisions made, risks/blockers, financial figures mentioned — from the same transcript. Build a small hallucination-detection gate (NLI or QA-based factuality check) that flags any generated claim not traceable to the transcript before it's shown to the user. This proves you understand query-focused summarization as a distinct capability and treat faithfulness as something to actively gate, not hope for.

## Large: Long-Document Compliance Summarizer with Map-Reduce/Refine Comparison

Build a summarizer over long regulatory or legal documents (public SEC filings, public court opinions, or public regulatory guidance documents — all real, freely available sources exceeding typical context windows) that implements both map-reduce and refine strategies and lets a user compare outputs from both on the same document. Build an evaluation harness scoring both strategies on faithfulness (LLM-as-judge plus an NLI-based factuality checker) and on cross-section coherence (does the summary correctly connect an obligation mentioned early with a related clause mentioned much later). This proves you understand the real tradeoffs between long-context summarization strategies with actual measurement, not just a textbook description of "map-reduce vs. refine."
