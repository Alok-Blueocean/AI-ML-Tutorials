# LLM Evaluation and Hallucination Reduction — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual/design questions.

1. **Write a weak vs. strong judge prompt.** Write a naive "rate this 1-5" judge prompt and a strong version with an explicit rubric, chain-of-thought-first instruction, and JSON output contract. Run both against the same 10 candidate answers and compare how often their scores diverge from your own manual rating.

2. **Measure position bias.** Build a pairwise LLM-as-judge prompt that picks the better of two answers (A vs. B). Run it on 20 example pairs, then re-run with A and B swapped. Report how many verdicts flip purely from reordering, and propose a concrete mitigation.

3. **Measure verbosity bias.** Take 10 correct, concise answers and produce a padded, longer version of each that adds no new correct information. Run both through an LLM judge and report how often the padded version scores higher despite adding no value.

4. **Set up an LLM-as-judge regression test.** Build a golden dataset of 20 examples for a task of your choice (e.g., support Q&A). Write two prompt versions (baseline and candidate) and a CI-style script that runs both against the judge, computes a delta, and exits non-zero if the candidate regresses correctness by more than 2%.

5. **Conceptual: composite vs. per-dimension gating.** Given a composite score formula `0.5*correctness + 0.3*relevance + 0.2*format`, construct a concrete example where the composite score passes a 4.0 threshold but one dimension is dangerously low. Explain what per-dimension threshold rule would have caught it.

6. **Calibrate a judge against human raters.** Hand-label 30 examples yourself (or with a colleague) on a 1-5 scale, then score the same 30 with an LLM judge. Compute Spearman's rho between the two score series and decide, using the ρ ≥ 0.75 convention, whether this judge is safe to use as a CI gate.

7. **Implement self-consistency voting.** For a set of 20 reasoning questions (math word problems or logic puzzles), sample 5 answers per question at temperature 0.7, take the majority vote, and compare accuracy against a single greedy-decoded answer per question. Report the accuracy lift and the added cost/latency.

8. **Build a retrieval-then-verify pipeline.** Take 10 outputs from an LLM answering questions from a small document set. For each output, extract the individual factual claims, independently verify each claim against the source documents, and flag any unsupported claim. Report the hallucination rate before and after the verification pass removes flagged claims.

9. **Conceptual: task-specific metric vs. general benchmark.** Pick a public benchmark (e.g., MMLU) and a hypothetical production task (e.g., internal HR-policy Q&A). Explain concretely why a model's MMLU rank would be a poor proxy for how well it performs on the HR task, and design a 10-question task-specific eval set that would actually measure it.

10. **Design a structured-output guardrail.** Define a Pydantic (or JSON Schema) contract for a support-agent response that must include an answer, a confidence score, and a list of source chunk IDs. Write a validation function that rejects malformed output and a bounded-retry policy (e.g., 2 retries then escalate to human) for when validation keeps failing.

11. **Build a faithfulness fact-checking layer.** For a RAG pipeline of your choosing, implement a faithfulness check that breaks a generated answer into individual claims and scores whether each claim is supported by the retrieved context. Run it against 15 generated answers and report the fraction of unsupported claims caught.

12. **Conceptual: benchmark contamination.** Explain, in your own words, two different ways benchmark contamination can occur (during pretraining vs. during instruction tuning), and propose one detection method and one mitigation your team could realistically apply without academic-scale tooling.

13. **Design a tiered CI evaluation strategy.** For a customer-facing LLM feature, design a 3-tier evaluation strategy (PR-triggered, push-to-main-triggered, weekly-scheduled) specifying sample size, checks run, and approximate cost per tier. Justify why a single-tier "always run everything" strategy would be worse.

14. **Diagnose an inconsistent judge.** You run the same LLM-as-judge scoring pass on the same 50 examples twice (same prompt, same model, temperature 0) and get meaningfully different aggregate scores both times. List the 4 most likely root causes, in priority order, and the first diagnostic step for each.

15. **End-to-end critique.** Given a described production RAG chatbot that passes its CI regression gate every time but users report a rising hallucination rate over the past month, propose a prioritized list of 5 concrete interventions spanning golden-dataset freshness, judge calibration, guardrails, and monitoring — and justify the order.
