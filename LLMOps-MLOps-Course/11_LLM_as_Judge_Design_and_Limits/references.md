# References — Module 11: LLM-as-Judge: Evaluator Design and Limits

Official documentation, research papers, and engineering blog posts, organized by the module's four sub-topics.

---

## A. Foundational papers on LLM-as-judge

### 1. "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" — Zheng et al., NeurIPS 2023 Datasets & Benchmarks Track
- **Link:** https://arxiv.org/abs/2306.05685
- **What it teaches:** The paper that established LLM-as-judge as a credible practice. Introduces MT-Bench (80 multi-turn questions) and Chatbot Arena (crowdsourced human preference platform) as calibration benchmarks, and formally names and measures position bias, verbosity bias, and self-enhancement bias — the same bias taxonomy this module's "limits of automated scoring" section covers (length bias, self-preference bias, etc.). Reports that strong LLM judges (GPT-4 at the time) can reach roughly 80%+ agreement with human preferences — comparable to human-human agreement.
- **Difficulty:** Intermediate (research paper, but clearly written).
- **Reading time:** 30-45 minutes.

### 2. "G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment" — Liu et al., EMNLP 2023
- **Link:** https://arxiv.org/abs/2303.16634 (also on ACL Anthology: https://aclanthology.org/2023.emnlp-main.153/)
- **What it teaches:** The origin of the "chain-of-thought-first, form-filling" judge prompting pattern this module's `judge_prompt.txt` is built on. Describes a three-part method: (1) a prompt defining the task and criteria, (2) an explicit chain-of-thought of evaluation steps, (3) a scoring function that uses output token probabilities to produce a continuous score rather than a single discrete label. Reports Spearman correlation with human judgment around 0.51 on summarization — a concrete, citable number for why CoT-based judging outperforms naive "just ask for a score" prompting.
- **Difficulty:** Intermediate.
- **Reading time:** 25-35 minutes.

### 3. "From Generation to Judgment: Opportunities and Challenges of LLM-as-a-Judge" — Li et al., EMNLP 2025 (survey)
- **Link:** https://llm-as-a-judge.github.io/ (survey hub with paper list, code, and slides)
- **What it teaches:** A comprehensive, up-to-date survey organizing the LLM-as-judge literature along three axes — what to judge, how to judge, and where to judge — plus benchmarks and known failure modes. Good as a single jumping-off point into the wider 2024-2026 research literature this module draws from.
- **Difficulty:** Advanced (survey paper, dense but well-organized).
- **Reading time:** 1-2 hours for a full read; usable as a reference/index in 10-15 minutes.

---

## B. Bias taxonomy and calibration

### 4. Length/verbosity bias findings
- **Link:** referenced within Zheng et al. 2023 above, and explored further in later 2024-2025 papers on self-preference and rubric-based evaluation (e.g., arXiv:2604.06996, "Self-Preference Bias in Rubric-Based Evaluation of Large Language Models" and arXiv:2604.22891, "Quantifying and Mitigating Self-Preference Bias of LLM Judges").
- **What it teaches:** Quantifies that judge models systematically favor longer, more verbose responses even when the extra length adds no substantive value, and that self-preference bias is linked to the judge assigning lower perplexity (and therefore a higher score) to text that resembles its own output distribution — a concrete mechanistic explanation for why a judge model tends to score its own family's outputs more favorably. Directly informs the module's length-bias and self-preference-bias sections.
- **Difficulty:** Advanced.
- **Reading time:** 20-30 minutes per paper.

### 5. "How to Calibrate Your LLM Judge With Human Annotations" — Galileo
- **Link:** https://galileo.ai/blog/calibrate-llm-judge-human-annotations
- **What it teaches:** A concrete, five-step operational framework for ongoing judge calibration: stratified sampling of cases for human review (by confidence band, category, recency, and disagreement pattern), structured SME scoring without seeing the judge's verdict, converting high-confidence disagreements into rubric anchors, measuring agreement with Cohen's kappa (target ≥ 0.60) rather than raw agreement rate, and a decision rule for when to re-calibrate versus rebuild the judge entirely (recommends rebuilding once expert disagreement exceeds roughly 15% on core criteria).
- **Difficulty:** Intermediate.
- **Reading time:** 15-20 minutes.

### 6. "LLM Evals: Everything You Need to Know" (Evals FAQ) — Hamel Husain & Shreya Shankar
- **Link:** https://hamel.dev/blog/posts/evals-faq/
- **What it teaches:** The community's most cited practical reference on building and calibrating LLM judges: error analysis as the primary activity, binary pass/fail over Likert scales, the "100+ labeled examples and weekly maintenance" heuristic for a working judge, and the "benevolent dictator" pattern for keeping calibration decisions consistent across a small number of domain experts rather than diffusing it across many annotators. See also `books.md` in this module, where this is treated as a book-chapter-equivalent resource.
- **Difficulty:** Intermediate to Advanced.
- **Reading time:** 25-30 minutes.

### 7. Evidently AI — LLM-as-a-judge guide
- **Link:** https://www.evidentlyai.com/llm-guide/llm-as-a-judge
- **What it teaches:** A practitioner-oriented walkthrough of rubric design (binary/low-precision scoring, splitting complex criteria into separate evaluators), the standard bias taxonomy (position, verbosity, self-enhancement), and mitigation tactics like swapping response order and preferring direct scoring over pairwise comparison where possible.
- **Difficulty:** Intermediate to Advanced.
- **Reading time:** 25-35 minutes.

---

## C. Ensembling judges / panels of judges

### 8. "Replacing Judges with Juries: Evaluating LLM Generations with a Panel of Diverse Models" (PoLL) — Verga et al., 2024
- **Link:** https://arxiv.org/abs/2404.18796
- **What it teaches:** Proposes replacing a single large judge (e.g., GPT-4) with a Panel of LLM evaluators (PoLL) composed of a larger number of smaller, diverse models. Shows the panel approach outperforms a single large judge, exhibits less intra-model bias (because the panel spans disjoint model families), and costs over 7x less — the key citable result behind this module's "ensembling multiple judges" expansion.
- **Difficulty:** Intermediate.
- **Reading time:** 20-30 minutes.

### 9. "Verdict: A Library for Scaling Judge-Time Compute" — Kalra & Tang (Haize Labs), 2025
- **Link:** https://arxiv.org/abs/2502.18018
- **What it teaches:** Formalizes ensembling and multi-step reasoning (verification, debate, aggregation) as composable "reasoning units" that can be assembled into more reliable judge pipelines, showing this can match much larger fine-tuned judges without a proportional increase in model size. The natural "how do I actually implement an ensemble" companion to the PoLL paper's "why ensembling works" result.
- **Difficulty:** Advanced.
- **Reading time:** 25-35 minutes.

---

## D. Framework official documentation (DeepEval, Ragas, G-Eval)

### 10. DeepEval official documentation
- **Link:** https://deepeval.com/docs/introduction and https://deepeval.com/docs/getting-started
- **What it teaches:** How to install DeepEval, define test cases, and run metrics (including `GEval`, DeepEval's implementation of the G-Eval CoT pattern) via a Pytest-style `deepeval test run` command, plus how to wire evaluation into CI/CD and optionally sync to the Confident AI cloud platform for regression tracking.
- **Difficulty:** Beginner to Intermediate.
- **Reading time:** 20-30 minutes for the getting-started path; the full metrics reference is a longer, ongoing reference document.

### 11. DeepEval — G-Eval metric page
- **Link:** https://deepeval.com/docs/metrics-llm-evals
- **What it teaches:** The concrete API for configuring a G-Eval-style judge: `criteria`, `evaluation_params`, optional explicit `evaluation_steps` (for reproducibility — the practical version of "write your rubric explicitly" from this module), `rubric` for constraining score ranges, and `strict_mode` for forcing binary pass/fail output. Also links onward to `DAGMetric` for deterministic, rule-based decomposition of a judgment into a graph of smaller checks.
- **Difficulty:** Intermediate.
- **Reading time:** 15-20 minutes.

### 12. Ragas official documentation
- **Link:** https://docs.ragas.io/en/stable/
- **What it teaches:** Framed explicitly as moving RAG evaluation "from vibe checks to systematic evaluation loops." Covers core concepts (experiments, metrics, datasets), how-to guides for LLM adapters, and metric references for faithfulness, answer relevancy, context precision, and context recall, plus synthetic test-set generation and integrations with LangChain/LlamaIndex.
- **Difficulty:** Beginner to Intermediate.
- **Reading time:** 20-30 minutes for the quickstart; longer as an ongoing reference.

### 13. OpenAI Evals — official repository documentation
- **Link:** https://github.com/openai/evals
- **What it teaches:** OpenAI's own registry-and-template system for evals, including "model-graded" eval templates — i.e., structured patterns for using one model to grade another's output — plus guidance on building custom evals and integrating with logging tools like Weights & Biases.
- **Difficulty:** Intermediate.
- **Reading time:** 20-30 minutes.

### 14. DeepEval blog — framework comparison posts
- **Link:** https://deepeval.com/blog/deepeval-vs-ragas and https://deepeval.com/blog/top-5-llm-evaluation-frameworks
- **What it teaches:** A maintainer's-eye (so, read with awareness of vendor bias) comparison of DeepEval against Ragas and other frameworks: DeepEval treats evaluation like unit testing (pass/fail test cases in CI), while Ragas treats it like measurement (continuous scores per RAG-specific dimension). Useful as a structured starting point for the module's "how do DeepEval, Ragas, and G-Eval differ" comparison, to be cross-checked against the Ragas and DeepEval docs directly for balance.
- **Difficulty:** Beginner to Intermediate.
- **Reading time:** 10-15 minutes.

---

## Quick-reference summary table

| # | Resource | Topic | Difficulty | Time |
|---|----------|-------|------------|------|
| 1 | Zheng et al. 2023 (MT-Bench) | Foundational LLM-as-judge paper, bias taxonomy | Intermediate | 30-45 min |
| 2 | Liu et al. 2023 (G-Eval) | CoT-based judge prompting, origin of the pattern | Intermediate | 25-35 min |
| 3 | LLM-as-a-Judge survey (EMNLP 2025) | Comprehensive current-state survey | Advanced | 1-2 hr |
| 4 | Self-preference bias papers (2025-2026) | Mechanism behind self-preference bias | Advanced | 20-30 min |
| 5 | Galileo calibration blog | Operational calibration framework, Cohen's kappa | Intermediate | 15-20 min |
| 6 | Hamel Husain / Shreya Shankar Evals FAQ | Practical judge-building + calibration | Intermediate-Advanced | 25-30 min |
| 7 | Evidently AI LLM-as-judge guide | Rubric design + bias + mitigation | Intermediate-Advanced | 25-35 min |
| 8 | Verga et al. 2024 (PoLL) | Panel-of-judges ensembling | Intermediate | 20-30 min |
| 9 | Verdict whitepaper (2025) | Judge-time compute scaling, ensembling library | Advanced | 25-35 min |
| 10-11 | DeepEval docs | Framework: unit-test style LLM eval, G-Eval metric | Beginner-Intermediate | 20-30 min |
| 12 | Ragas docs | Framework: RAG-specific composite metrics | Beginner-Intermediate | 20-30 min |
| 13 | OpenAI Evals repo docs | Framework: model-graded eval registry | Intermediate | 20-30 min |
| 14 | DeepEval comparison blog posts | DeepEval vs Ragas vs others | Beginner-Intermediate | 10-15 min |
