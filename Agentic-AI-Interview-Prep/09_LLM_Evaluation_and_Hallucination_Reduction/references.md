# LLM Evaluation and Hallucination Reduction — References

All links below were fetched and content-verified against the claim this session, unless explicitly marked otherwise.

## Papers

- Zheng et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" (NeurIPS 2023) — the foundational LLM-as-judge study; names and measures position, verbosity, and self-enhancement bias; reports 80%+ judge-human agreement, comparable to human-human agreement. https://arxiv.org/abs/2306.05685
- Wang et al., "Large Language Models are not Fair Evaluators" (ACL 2024) — demonstrates position bias severe enough that reordering candidate responses alone let a weaker model "beat" a stronger one on the majority of tested queries; proposes calibration strategies. https://arxiv.org/abs/2305.17926
- Liu et al., "G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment" (EMNLP 2023) — the chain-of-thought-first, form-filling judge pattern; reports Spearman correlation of 0.514 with human judgment on summarization. https://arxiv.org/abs/2303.16634
- Verga et al., "Replacing Judges with Juries: Evaluating LLM Generations with a Panel of Diverse Models" (2024) — introduces PoLL (Panel of LLM evaluators); a panel of smaller, diverse-family judges outperforms one large judge, reduces intra-model bias, and costs 7x+ less. https://arxiv.org/abs/2404.18796
- Wang et al., "Self-Consistency Improves Chain of Thought Reasoning in Language Models" (ICLR 2023) — sampling multiple reasoning paths and majority-voting the answer, boosting accuracy on reasoning benchmarks (e.g., +17.9% on GSM8K) with no extra training. https://arxiv.org/abs/2203.11171
- Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (NeurIPS 2020) — the original RAG paper; grounding generation in retrieved non-parametric memory. https://arxiv.org/abs/2005.11401
- Gao et al., "RARR: Researching and Revising What Language Models Say, Using Language Models" (ACL 2023) — post-hoc retrieval-then-verify: automatically finds attribution for generated text and revises unsupported claims. https://arxiv.org/abs/2210.08726
- Dhuliawala et al., "Chain-of-Verification Reduces Hallucination in Large Language Models" (Meta, Findings of ACL 2024) — draft, generate independent verification questions, answer them separately, then revise; reduces hallucination across multiple generation tasks. https://arxiv.org/abs/2309.11495
- Kadavath et al., "Language Models (Mostly) Know What They Know" (Anthropic, 2022) — larger models are reasonably well-calibrated on multiple-choice framings and can self-evaluate "P(True)" for their own answers with useful, scaling accuracy. https://arxiv.org/abs/2207.05221
- "Are LLM Benchmarks Already Contaminated? A Systematic Review of Contamination Detection Methods" (GEM Workshop, 2026, Outstanding Paper) — review of 55 studies; every popular static benchmark shows some contamination, no single detection method is reliable across all settings, performance inflation estimated at roughly 6-40% depending on benchmark. https://aclanthology.org/2026.gem-main.50/

## Official Docs / Tooling

- Promptfoo, CI/CD Integration docs — quality gates, regression testing on prompt/model changes, GitHub Actions/GitLab CI/Jenkins integration. https://www.promptfoo.dev/docs/integrations/ci-cd/
- Promptfoo main repository (25k+ GitHub stars at check time; CLI and library for evaluating and red-teaming LLM apps). https://github.com/promptfoo/promptfoo
- DeepEval — pytest-style LLM evaluation framework; `GEval` (LLM-as-judge custom-criteria metric) and `DAGMetric` (deterministic decomposition of sub-checks); integrates with any CI/CD via `deepeval test run`. https://github.com/confident-ai/deepeval
- Ragas — RAG-specialized evaluation framework (faithfulness, answer relevancy, context precision/recall); official docs. https://docs.ragas.io/
- τ²-bench / τ³-bench (Sierra Research) — agent-evaluation benchmark for tool-using, policy-adherent conversational agents; useful reference point for golden-dataset/regression design in agentic systems. https://github.com/sierra-research/tau2-bench
- AgentBench (THUDM) — multi-environment LLM-agent benchmark; now includes an AgentBench FC (function-calling) variant added October 2025. https://github.com/THUDM/AgentBench

## Articles / Interview Prep

- Datadog, "Detecting hallucinations with LLM-as-a-judge: Prompt engineering and beyond." https://www.datadoghq.com/blog/ai/llm-hallucination-detection/
- Evidently AI, "LLM-as-a-judge: a complete guide to using LLMs for evaluations." https://www.evidentlyai.com/llm-guide/llm-as-a-judge
- Sanjay Kumar PhD, "LLM Evaluation Interview Questions and Answers" (Medium) — broad coverage of judge design, bias, and hallucination-detection interview patterns. https://skphd.medium.com/llm-evaluation-interview-questions-and-answers-8748c60da5c1 (not URL-verified this session beyond the search snippet — content matches the topic but was not independently fetched and read in full).

## Notes on What to Prioritize

Interview signal consistently centers on: naming and diagnosing the three classic judge biases (position, verbosity, self-preference) with a concrete test for each; the difference between statistical and practical significance when gating a prompt/model promotion; why a golden/regression eval set must be frozen and versioned; the distinction between RAG grounding (reduces but does not eliminate hallucination) and a separate faithfulness/fact-checking layer (catches cases where grounding was ignored); and knowing when ensembling judges or self-consistency sampling is worth the added cost versus when it's overkill. Be ready to whiteboard a CI-gated regression pipeline end to end (golden set -> judge -> delta check -> pass/block) and to reason about a "judge gives inconsistent verdicts" scenario as a diagnostic exercise, not just a definitions quiz.

## Corrections Applied This Session (worth flagging)

- τ-bench (tau-bench) has been superseded by τ²-bench, which — as of this session's research (mid-2026) — has itself been further superseded by τ³-bench, hosted at the same repository URL (`sierra-research/tau2-bench`). Cite τ³-bench as the current benchmark; the repo name did not change even though the benchmark version did.
- Ragas' GitHub organization has moved: the historical `explodinggradients/ragas` URL now redirects to `github.com/vibrantlabsai/ragas`, reflecting a company rebrand from Exploding Gradients to Vibrant Labs (Exploding Gradients Inc. remains the legal entity). Old bookmarks/links should be updated.
- Promptfoo was acquired by OpenAI (announced March 2026) and is being folded into OpenAI's Frontier enterprise-agent platform; OpenAI has committed to keeping Promptfoo's tools open-source. Worth knowing as background if asked "who maintains this" in an interview, though it does not change how the tool is used.
