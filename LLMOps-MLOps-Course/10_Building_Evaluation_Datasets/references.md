# References — Building Evaluation Datasets

Official documentation, papers, and engineering blog posts, organized by the three source-transcript topics plus the module's expansion areas (pipeline design, DeepEval/Ragas-compatible formats, dataset versioning).

---

## 1. Official Documentation

### DeepEval — Datasets & Synthesizer
- **Datasets:** https://deepeval.com/docs/evaluation-datasets — the `Golden`/`EvaluationDataset` schema, loading from CSV/JSON/JSONL, `push()`/`pull()` to a hosted registry.
- **Synthesizer (synthetic data generation):** https://deepeval.com/guides/guides-using-synthesizer and https://deepeval.com/docs/golden-synthesizer — generating goldens from documents/contexts or from existing goldens; directly relevant to the "synthetic" category in the edge-case sampling taxonomy.
- **Confident AI platform concepts:** https://documentation.confident-ai.com/concepts/datasets — dataset lifecycle in the hosted product (useful even if you self-host, as a model of what a mature dataset-management UI tracks: versions, alias, last-updated, row counts by category).
- **Difficulty:** Beginner–Intermediate. **Reading time:** ~30–45 minutes across the three pages.

### Ragas — Test Data Generation
- **Concepts — Test Data Generation:** https://docs.ragas.io/en/stable/concepts/test_data_generation/
- **Getting started — RAG testset generation:** https://docs.ragas.io/en/stable/getstarted/rag_testset_generation/
- **API reference — Generation:** https://docs.ragas.io/en/latest/references/generate/
- **What it teaches:** The evolutionary ("Evol-Instruct"-style) approach to synthesizing questions with varying reasoning/condition/multi-context complexity from a document set, and the question/contexts/ground_truth schema that most RAG-eval tooling has converged on.
- **Difficulty:** Intermediate. **Reading time:** ~45 minutes.

### DVC — Data Versioning
- **Tutorial — Versioning Data and Models:** https://doc.dvc.org/example-scenarios/versioning-data-and-models/tutorial — the canonical `dvc add` → `git commit` → `git tag` → `git checkout && dvc checkout` workflow this module's Part 3 adapts for evaluation datasets.
- **User Guide:** https://doc.dvc.org/user-guide
- **Get Started:** https://doc.dvc.org/start
- **Difficulty:** Beginner–Intermediate. **Reading time:** ~30 minutes for the core tutorial.

### LangSmith — Evaluation & Datasets
- **Evaluation concepts:** https://docs.langchain.com/langsmith/evaluation — datasets as collections of "examples" (input/reference-output pairs), building datasets from production traces and annotation queues, and running offline evaluations against a frozen dataset version before deploying a change.
- **Difficulty:** Intermediate. **Reading time:** ~20–30 minutes.
- **Why it matters:** A second, commercially-backed reference implementation (alongside DeepEval/Ragas) of "dataset built from production traces + annotation queue for human labeling," useful for cross-checking your own pipeline design against how a widely used platform models the same problem.

### OpenAI Evals — Building Custom Evals
- **Getting started:** https://developers.openai.com/cookbook/examples/evaluation/getting_started_with_openai_evals
- **Repository docs:** https://github.com/openai/evals (see `docs/` folder for "build-eval," "custom-eval," and "completion-fn-protocol" guides)
- **Difficulty:** Intermediate. **Reading time:** ~30 minutes.

---

## 2. Papers & Formal References

### Inter-annotator agreement (gold-labeling with Cohen's kappa)
- **Cohen, J. (1960). "A Coefficient of Agreement for Nominal Scales."** *Educational and Psychological Measurement.* The original definition of the kappa statistic used to justify the "two domain experts, kappa ≥ 0.7" gold-labeling requirement in Part 1.
- **Landis, J.R. & Koch, G.G. (1977). "The Measurement of Observer Agreement for Categorical Data."** *Biometrics.* Source of the widely cited interpretation bands (0.61–0.80 = "substantial," 0.81–1.00 = "almost perfect") used to justify why 0.7 is a reasonable production bar rather than an arbitrary number.
- **Accessible explainer:** "Inter-Annotator Agreement: An Introduction to Cohen's Kappa Statistic" (Surge AI, Medium) — https://surge-ai.medium.com/inter-annotator-agreement-an-introduction-to-cohens-kappa-statistic-dcc15ffa5ac4 — plain-language walkthrough of the formula (κ = (p_o − p_e)/(1 − p_e)) and the "kappa paradox" (kappa can look low even with high raw agreement when label distributions are highly imbalanced) — an important caveat when your gold set is intentionally skewed toward rare edge cases.
- **Difficulty:** Beginner (explainer) / Intermediate (original papers). **Reading time:** ~15 minutes (explainer), ~20 minutes each (original papers).

### Bias, contamination, and dataset quality
- **Sambasivan, N. et al. (2021). "'Everyone Wants to Do the Model Work, Not the Data Work': Data Cascades in High-Stakes AI."** *CHI 2021.* Documents how downstream failures in high-stakes ML systems repeatedly trace back to undervalued, poorly audited data work — the research basis for why this module treats the pre-freeze bias audit as non-optional rather than a nice-to-have.
- **Gebru, T. et al. (2018/2021). "Datasheets for Datasets."** *Communications of the ACM.* Proposes documenting a dataset's collection process, composition, and known biases — a good template to adapt for documenting your gold set's provenance (which production window it was sampled from, category distribution, known gaps) alongside its DVC version tag.
- **"A Taxonomy for Data Contamination in Large Language Models"** — https://arxiv.org/pdf/2407.08716 — categorizes the ways benchmark/eval data leaks into training data, directly relevant to why a frozen, versioned gold set must be kept out of any fine-tuning or RAG-indexing pipeline.
- **"LLM Benchmark Datasets Should Be Contamination-Resistant"** — https://arxiv.org/html/2605.19999 — a 2026 position paper arguing for practices (e.g., periodic refresh, canary strings, held-back subsets) that map directly onto this module's dataset-versioning and freeze/refresh discipline.
- **Difficulty:** Intermediate–Advanced (papers), Beginner (blog summaries). **Reading time:** ~30–40 minutes per paper.

### Edge-case, adversarial, and stratified sampling design
- **"When Generic Prompt Improvements Hurt: Evaluation-Driven Iteration for LLM Applications"** — https://arxiv.org/html/2601.22025 — a 2026 paper on evaluation-driven iteration that explicitly discusses the risk of over-indexing on easy/common cases in a test set and proposes a four-bucket design (representative production sample, adversarial-input library, deliberately constructed edge cases, replayed past failures) — almost a direct academic formalization of this module's failure-driven/escalation/statistical/adversarial/synthetic taxonomy from Part 2.
- **"LLM Eval Golden Set Design: A 2026 Engineering Guide"** (FutureAGI) — https://futureagi.com/blog/llm-eval-golden-set-design-2026/ — practitioner-oriented 2026 guide covering golden-set sizing, stratification, and freeze discipline; useful as a cross-check against this module's own recommendations since it is contemporaneous (mid-2026).
- **"LLM Red Teaming: The Complete Step-by-Step Guide"** (Confident AI) — https://www.confident-ai.com/blog/red-teaming-llms-a-step-by-step-guide — manual vs. automated adversarial-testing trade-offs; the manual approach is described as better at surfacing subtle, nuanced edge-case failures, while automated attack simulation buys scale/repeatability — useful framing for deciding how to spend your ~20% edge-case budget between hand-crafted and generated adversarial examples.
- **Difficulty:** Intermediate–Advanced. **Reading time:** ~30–45 minutes each.

---

## 3. Engineering Blog Posts (production log → eval dataset pipelines)

- **Braintrust — "What is LLM evaluation? A practical guide to evals, metrics, and regression testing"** — https://www.braintrust.dev/articles/llm-evaluation-guide — describes closing the production-incident-to-regression-test loop ("adding that case to a dataset takes one click") and automated failure-pattern mining from logs; a good industry reference point for the "production log → gold set" pipeline this module builds from first principles.
- **Datadog — "Building an LLM evaluation framework: best practices"** — https://www.datadoghq.com/blog/llm-evaluation-framework-best-practices/ — describes a pipeline that logs live LLM requests (prompts, responses, session ID, user feedback) and ingests them into a scoring service; a concrete architecture reference for the "production-log" ingestion stage of this module's pipeline.
- **Label Studio — "How to build an agent evaluation dataset"** — https://labelstud.io/learningcenter/how-to-build-an-agent-evaluation-dataset/ — practitioner note that ~20–30 carefully chosen real production logs often surface more of the long tail of behavior than a much larger volume of synthetic data; a useful counterweight when deciding the synthetic-vs-real-log ratio in your ~20% edge-case allocation.
- **Latitude — "How to Build Automated LLM Evaluation Pipelines"** — https://latitude.so/blog/how-to-build-automated-llm-evaluation-pipelines — reinforces starting from a sample of actual production requests/responses rather than hand-authoring an eval set from scratch.
- **Difficulty:** Beginner–Intermediate. **Reading time:** ~10–15 minutes each.

---

## 4. Practitioner Reference / FAQ (living documents)

- **Hamel Husain — "LLM Evals: Everything You Need to Know"** — https://hamel.dev/blog/posts/evals-faq/ — the single most detailed, continuously updated public reference on this exact module's subject matter. Concrete numeric guidance echoed in this module: review at least ~100 traces as a starting point for error analysis, treat "no new failure category found in the last 20 traces reviewed" as a saturation-based stopping signal, use 5 complementary sampling strategies (random, clustering, statistical/extreme-value, classifier-flagged, feedback-based) rather than one, and prefer a single expert "benevolent dictator" labeler over multi-rater kappa reconciliation when the domain allows it — a useful counterpoint to worth discussing against this module's two-expert-plus-kappa requirement (both are valid; the right choice depends on how contested your label definitions are).
- **Hamel Husain & Shreya Shankar — "AI Evals for Engineers & PMs" (Maven course)** — https://maven.com/parlance-labs/evals — the most field-tested current curriculum on this exact topic; referenced here as a paid, non-video resource worth knowing about even though it isn't excerpted directly in this module.
- **Difficulty:** Intermediate. **Reading time:** the FAQ is long-form and best used as a reference, not read linearly; budget 1–2 hours to read fully.

---

## Version/tooling notes as of mid-2026
- DeepEval and Ragas remain the two dominant open-source LLM-evaluation frameworks; both have converged on broadly similar dataset schemas (input/query, expected/ground-truth output, retrieved context), which is why this module recommends designing your gold_set export to satisfy both simultaneously rather than picking one.
- DVC remains the standard lightweight choice for versioning evaluation datasets as of mid-2026 for teams not already on a full feature-store/lakehouse platform; nothing has displaced its `add`/`push`/`checkout` workflow as the default pattern for "Git for data" at small-to-mid scale.
- Red-teaming/vulnerability-scanning features (adversarial payload libraries, OWASP LLM Top-10 coverage) have moved from being a niche add-on to a standard, expected feature of eval tooling (present in DeepEval, Promptfoo, and Giskard alike) — worth reflecting in how much weight the "adversarial" category gets in your own sampling taxonomy compared to earlier (2023–2024) guidance that treated it as optional.
