# Summarization — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual questions.

1. **Extractive vs abstractive, by hand.** Take 5 news articles. Manually produce an extractive summary (pick and stitch 2-3 sentences verbatim) and an abstractive summary (paraphrase in your own words) for each. Compare readability and information density.

2. **Basic abstractive summarizer.** Use an LLM to abstractively summarize the same 5 articles at a fixed target length (e.g., 3 sentences). Score them with ROUGE-1/2/L against a reference summary (write your own reference if none exists) and note any cases where a summary you judge as "good" scores surprisingly low on ROUGE.

3. **Conceptual: why ROUGE misleads on paraphrase.** Using one of your ROUGE results from exercise 2, explain in your own words exactly why n-gram overlap penalizes a well-written paraphrase, and what BERTScore would likely do differently on the same example.

4. **Add BERTScore.** Score the same 5 summaries with BERTScore and compare rankings against ROUGE. Find at least one case where the two metrics disagree on which summary is better, and manually judge which metric's ranking you agree with.

5. **Build an LLM-as-judge evaluator.** Write a G-Eval-style prompt that scores a summary against its source document on faithfulness, coherence, and relevance (1-5 scale each, with justification). Run it on your 5 summaries and compare against your own human judgment.

6. **Multi-document summarization.** Take 8-10 product reviews for the same item (some positive, some negative, some contradictory) and produce a single summary that fairly represents the range of opinions, not just the majority view. Manually check whether any minority-but-important complaint got dropped.

7. **Build a map-reduce summarizer.** Take a document too long for a single prompt (split it into 8-10 chunks). Implement map-reduce: summarize each chunk, then summarize the chunk-summaries into a final summary. Identify one place where information connecting two non-adjacent chunks was lost.

8. **Build a refine summarizer.** Implement the refine strategy on the same document from exercise 7 (sequentially update a running summary one chunk at a time). Compare the final summary against your map-reduce result for narrative coherence and note which chunks seem over- or under-represented (recency bias check).

9. **Conceptual: map-reduce vs refine tradeoff.** Given a specific document type (choose one: a multi-year legal case history, a single 50-page technical report, a year of weekly status updates), argue which strategy you'd pick and why, considering both quality and cost/latency.

10. **Length-controlled summarization.** Prompt for summaries at 3 different target lengths (1 sentence, 1 paragraph, 1 page) of the same source. Measure how often the model actually respects the requested length without an enforcement step, and add a retry/truncation mechanism for violations.

11. **Query-focused summarization.** Take one long meeting transcript or long document and generate 3 different summaries of it, each focused on a different stated angle (e.g., action items, risks, financial figures). Verify the three summaries are meaningfully different, not near-duplicates with different headers.

12. **Hallucination detection.** Design a test set of 15 source-summary pairs, where 10 summaries are faithful and 5 have a deliberately injected fabricated fact (a wrong number, an invented claim, a misattributed quote). Build an NLI-based or QA-based faithfulness checker and measure how many of the 5 hallucinations it catches.

13. **Intrinsic vs extrinsic hallucination classification.** For a set of summarization outputs your pipeline produced (from any earlier exercise), find or construct 3 examples of intrinsic hallucination (a source fact stated incorrectly) and 3 of extrinsic hallucination (a fact with no basis in the source at all). Propose a different mitigation for each category.

14. **Long-output degradation check.** Generate a very long single-pass summary (push the model to produce as long an output as it will) of a long source document, and specifically check whether hallucination rate or factual drift increases toward the end of the output compared to the beginning.

15. **End-to-end system critique.** Given a described production meeting-summarization tool used by hundreds of teams, where users occasionally report "it said something that was never discussed," propose a prioritized list of 5 concrete interventions (extraction step, chunking strategy, evaluation, faithfulness gating, UX/transparency) and justify the order.
