# Summarization — Scenario-Based Q&A

**Situation:** Your abstractive summarizer states a specific quarterly revenue figure in its summary of a financial news article, but the number is actually wrong — it belongs to a different company mentioned elsewhere in the same article. What would you do and why?

Model answer: This is a hallucination, specifically an intrinsic one — the fact exists in the source but got misattributed to the wrong entity, rather than invented from nothing. Add an automated post-generation faithfulness gate that checks every stated number/entity pairing against the source (an NLI-based or QA-based factuality check, or an LLM-as-judge prompt specifically asked to verify each claim's entity attribution) before the summary is shown to a user, especially for any domain where a wrong number has financial or compliance consequences. Also test whether shortening the summarization prompt's instructions to explicitly require entity-grounded claims ("attribute every number to the specific company it came from") reduces this failure rate, and add this exact article as a permanent regression case.

---

**Situation:** A fine-tuned summarization model shows an improved ROUGE score over the previous version in offline evaluation, but human reviewers rate its live outputs as worse after rollout. What would you do and why?

Model answer: Trust the human/LLM-judge signal over ROUGE here — ROUGE rewards n-gram overlap with reference summaries, so a model can learn to copy more reference-like phrasing (boosting ROUGE) while actually degrading real coherence or faithfulness in ways ROUGE can't detect. Roll back or hold the new model, and re-evaluate using an LLM-as-judge rubric (faithfulness, coherence, relevance) plus a sample of human review, using ROUGE going forward only as a cheap regression sanity check, not the deciding metric for a ship/no-ship call.

---

**Situation:** You're asked to summarize a 400-page regulatory filing that far exceeds your model's context window, and the business needs both speed and narrative coherence connecting obligations mentioned in different sections. What would you do and why?

Model answer: Recognize the tension directly: map-reduce (summarize each chunk independently, then summarize the chunk-summaries) is fast and parallelizable but can lose connections between non-adjacent chunks — exactly the risk here, since an obligation stated in section 3 and its exception in section 40 could end up in separate, disconnected chunk summaries. Refine (sequentially update a running summary chunk by chunk) preserves more cross-chunk continuity at the cost of being sequential and prone to recency bias. For a compliance document where cross-section connections matter, lean toward refine or a hybrid (map-reduce first pass for speed, then a refine pass focused specifically on cross-referencing obligations), and validate with an evaluation set specifically testing whether known cross-section connections survive in the final summary.

---

**Situation:** A sales team and an engineering team both use the same meeting-summarization tool, but both complain the generic summary is "full of stuff we don't care about" and misses what they actually need. What would you do and why?

Model answer: This is a signal that generic summarization is the wrong tool — what's needed is query-focused summarization, generating a different summary from the same transcript depending on the requesting team's angle (customer commitments and objections for sales, technical decisions and blockers for engineering). Implement this as a first-class capability (a focus parameter that materially changes what's extracted and emphasized, not just a cosmetic prompt tweak), and validate with an evaluation set that checks the two summaries of the same transcript are meaningfully different and each responsive to its stated focus.

---

**Situation:** Your document summarizer is asked to compress a document to "one paragraph," but manual review shows it frequently produces 2-3 paragraphs or drifts significantly over a token budget. What would you do and why?

Model answer: Treat length control as something to enforce, not just request — models reliably drift from a requested length without a validation/retry step. Add a post-generation length check, and on violation either automatically retry with a more explicit constraint (an exact word/sentence count rather than a vague "one paragraph"), or truncate/re-summarize the over-length output. If using an API with a max-output-token parameter, use it as a hard backstop, but don't rely on it alone since it can produce an abrupt mid-sentence cutoff rather than a well-formed shorter summary.

---

**Situation:** Leadership asks whether your summarization feature is "making things up" before a wider rollout, and wants a number, not a vibe check.

Model answer: Build a labeled evaluation set of source-summary pairs, some faithful and some with deliberately injected fabricated facts, and measure both a faithfulness score (via an NLI/QA-based factuality checker and/or LLM-as-judge) and a human-reviewed sample rate on live production outputs. Track this as a regression metric over time rather than a one-time check, since model updates, prompt changes, or shifts in typical input document length (longer inputs correlate with more end-of-summary hallucination in long single-pass generation) can silently move this number. Report the faithfulness rate with its measurement methodology explicitly stated, since an LLM-as-judge score is a calibrated proxy, not ground truth.

---

**Situation:** You're evaluating two candidate summarization models for a legal-document use case, and one scores higher on ROUGE while a small human review panel prefers the other model's outputs. Which do you ship and why?

Model answer: Ship the model the human panel preferred, and treat this exact ROUGE-vs-human disagreement as expected, not confusing — ROUGE measures n-gram overlap with a reference summary, which correlates poorly with human judgment specifically for abstractive, paraphrase-heavy summaries. For a legal-document use case where faithfulness and correctness matter far more than surface-level wording match, weight human review (and a calibrated LLM-as-judge faithfulness score, validated periodically against human judgment) far above ROUGE, and reserve ROUGE for tracking whether a future model update drifts unexpectedly, not for the ship decision itself.

---

**Situation:** A long-context model with a huge context window lets you skip chunking entirely for most documents now, and someone proposes retiring your map-reduce/refine summarization pipeline altogether. What would you do and why?

Model answer: Partially agree, but don't retire the pipeline outright. A large context window does remove the *need* for chunking on moderate-length single documents, simplifying a large fraction of use cases. But truly massive multi-document corpora (multi-year transcript archives, large legal discovery sets) can still exceed even very large windows, and there's evidence that long single-pass generation itself degrades in faithfulness toward the end of very long outputs — a distinct risk that chunking doesn't have. Recommend keeping a map-reduce/refine (or an agent-orchestrated delegation) path available for the genuinely oversized or highest-stakes cases, while defaulting to single-pass summarization for documents that comfortably fit, and measuring faithfulness by output length to confirm whether the long-output degradation risk shows up in your specific use case before removing the safety net entirely.

---

**Situation:** A customer-facing summarization feature was tested extensively on well-formatted English-language documents, but in production it started receiving scanned, OCR'd documents with garbled text and occasional gibberish, and the summarizer began confidently "cleaning up" nonsense into plausible-sounding but fabricated sentences. What would you do and why?

Model answer: This is a hallucination triggered by low-quality input rather than a model-quality problem in isolation — when the source itself is noisy, an abstractive model tends to smooth over gaps with plausible-sounding invented content instead of flagging uncertainty. Add an input-quality gate upstream (OCR confidence scoring, garbled-text detection) that either declines to summarize below a quality threshold, flags the specific low-confidence sections to the user, or routes low-quality input through an extractive-leaning or more conservative prompting strategy that explicitly instructs the model to omit anything it cannot clearly read rather than "fill in" ambiguous text. Add this OCR-noise case explicitly to your evaluation set now that you know it's a real production input distribution.
