# GitHub Repositories — Module 11: LLM-as-Judge: Evaluator Design and Limits

---

### 1. [confident-ai/deepeval](https://github.com/confident-ai/deepeval)
- **Purpose:** "Pytest for LLMs" — an open-source LLM evaluation framework with 40+ ready-made metrics (G-Eval, hallucination, answer relevancy, faithfulness, task completion, conversational and agentic metrics), designed to be run as unit tests inside a normal CI pipeline via `deepeval test run`.
- **Popularity tier:** Very popular / widely adopted (tens of thousands of GitHub stars; verified at roughly 17k+ stars and 1.7k+ forks at last check, actively maintained by the Confident AI team).
- **Why it matters:** DeepEval's `GEval` metric is a direct, production-grade implementation of the chain-of-thought-first, explicit-rubric judge pattern this module teaches. Its `evaluation_steps` parameter is literally "write your rubric as an explicit list of reasoning steps" — the same principle behind the module's `judge_prompt.txt` example. Its `DAGMetric` (deterministic acyclic graph scoring) is a good real-world example of decomposing one fuzzy judgment into a tree of smaller, more reliable sub-judgments, which parallels the module's separated-scoring-dimensions approach.
- **How it relates to this module:** This is the primary framework to actually build and run the judges the module designs conceptually. Recommended as the "hands-on" repo to clone and experiment with while working through the judge_prompt.txt and composite-scoring material.

### 2. [explodinggradients/ragas](https://github.com/explodinggradients/ragas)
- **Purpose:** An evaluation and test-data-generation toolkit purpose-built for RAG (Retrieval-Augmented Generation) pipelines, combining LLM-based metrics (faithfulness, answer relevancy) with more traditional/statistical metrics (context precision, context recall) and synthetic test-set generation.
- **Popularity tier:** Very popular / widely adopted (verified at roughly 15k+ stars and 1.6k+ forks at last check; maintained by the Ragas/ExplodingGradients team, now referred to in places as VibrantLabs).
- **Why it matters:** Ragas is the clearest real-world example of *composite scoring done right* — it doesn't collapse RAG quality into one number, it reports separate named metrics (faithfulness, answer relevancy, context precision, context recall) that a team then combines with their own weights and thresholds, exactly mirroring the module's 0.5/0.3/0.2 weighted-score pattern and per-dimension thresholds.
- **How it relates to this module:** Use this repo as the reference implementation for "per-dimension thresholds" — read its metric definitions to see how a mature framework decides what counts as a separate evaluable dimension versus what gets rolled up.

### 3. [nlpyang/geval](https://github.com/nlpyang/geval)
- **Purpose:** The original reference implementation accompanying the "G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment" paper (Liu et al., EMNLP 2023) — includes the exact CoT-based evaluation prompts, the scoring script (`gpt4_eval.py`), and a meta-evaluation script that checks the judge's scores against human annotations on the SummEval benchmark.
- **Popularity tier:** Niche / research-reference repo (a research-paper companion repo rather than a maintained product, but it's the canonical source for the technique this module builds on).
- **Why it matters:** This is the actual origin of the "chain-of-thought-first, form-filling" judge prompt pattern the module teaches. Reading the raw prompt templates here shows exactly how the paper's authors structured a CoT judge prompt before frameworks like DeepEval or Ragas abstracted it into a metric object.
- **How it relates to this module:** Read this before or alongside `judge_prompt.txt` in this module — it's the "primary source" the module's evaluator-prompt-design section is ultimately descended from.

### 4. [openai/evals](https://github.com/openai/evals)
- **Purpose:** OpenAI's framework and registry for evaluating LLMs and LLM-based systems, including "model-graded" eval templates — i.e., using one model to grade another model's output, with built-in templates for common grading patterns.
- **Popularity tier:** Very popular / widely adopted (verified at roughly 19k+ stars at last check).
- **Why it matters:** The model-graded eval templates are a useful second reference point (alongside DeepEval and Ragas) for how a major AI lab operationalizes LLM-as-judge at scale, including the registry pattern for versioning eval definitions — directly relevant to this course's earlier modules on versioning combined with this module's evaluator design.
- **How it relates to this module:** Good comparison repo to see how "the same idea" (an LLM grading another LLM) gets implemented slightly differently across DeepEval, Ragas, and OpenAI's own tooling — useful for the module's "how do DeepEval, Ragas, and G-Eval differ" comparative deep dive.

### 5. [Arize-ai/phoenix](https://github.com/Arize-ai/phoenix)
- **Purpose:** Open-source AI observability and evaluation platform — OpenTelemetry-based tracing for LLM apps, plus built-in LLM-powered evaluators for response quality, retrieval relevance, hallucination, and more, with a UI for inspecting individual judge decisions.
- **Popularity tier:** Very popular / widely adopted (verified at roughly 10k+ stars at last check; backed by Arize AI, who also co-produced the DeepLearning.AI "Evaluating AI Agents" course listed in `videos.md`).
- **Why it matters:** Phoenix is a good example of pairing LLM-as-judge scoring with tracing/observability — showing *why* a judge flagged a low score by letting you inspect the underlying trace, which is the practical infrastructure that a human-escalation pattern like the module's `escalator.py` needs to sit on top of.
- **How it relates to this module:** Bridges this module with the observability/monitoring modules elsewhere in the course — worth exploring for how "review triggers" get surfaced to a human in an actual UI rather than just logged.

### 6. [promptfoo/promptfoo](https://github.com/promptfoo/promptfoo)
- **Purpose:** A CLI/library for evaluating and red-teaming LLM applications and prompts — side-by-side model/prompt comparison, custom "assertions" including LLM-graded assertions, and CI/CD integration.
- **Popularity tier:** Very popular / widely adopted (verified at roughly 23k+ stars at last check; the project was acquired by / is now part of OpenAI while remaining MIT-licensed and open source).
- **Why it matters:** Promptfoo's `llm-rubric` assertion type is a lightweight, config-file-driven way to define exactly the kind of rubric-based judge this module describes, without adopting a full framework — useful as a "simplest possible implementation" reference alongside the heavier DeepEval/Ragas frameworks.
- **How it relates to this module:** A good repo to point at when a team wants the composite-scoring and rubric ideas from this module without adopting DeepEval or Ragas wholesale.

### 7. [haizelabs/verdict](https://arxiv.org/pdf/2502.18018) (Verdict library; paper available on arXiv, code linked from the paper)
- **Purpose:** A library for "scaling judge-time compute" — composing modular reasoning units (verification, debate, aggregation) to build more reliable judges out of ensembles of smaller reasoning steps, rather than relying on one large judge call.
- **Popularity tier:** Emerging / research-adjacent (newer project, not yet at the adoption scale of DeepEval or Ragas, but directly relevant to the ensembling angle of this module).
- **Why it matters:** This is the most directly relevant resource for the "ensembling multiple judges to reduce single-judge bias" expansion of this module — it formalizes the intuition behind panel-of-judges/jury approaches into a reusable library pattern (verify → debate → aggregate) rather than just averaging N independent judge calls.
- **How it relates to this module:** Read alongside the PoLL paper (see `references.md`) as the two main technical anchors for the ensembling section of this module.

---

## How to use these together

A realistic production stack for the pattern this module teaches typically looks like:

```
                 +-------------------+
   RAG-specific  |      Ragas        |  faithfulness, answer relevancy,
   dimensions    |                   |  context precision/recall
                 +-------------------+
                          |
                          v
                 +-------------------+
   general judge |     DeepEval      |  G-Eval / DAGMetric composite score,
   dimensions     |    (or Verdict)   |  per-dimension thresholds, CI gate
                 +-------------------+
                          |
                          v
                 +-------------------+
   observability |   Arize Phoenix   |  trace inspection for flagged/
   & escalation  |  (or a homegrown  |  low-confidence cases routed to
                 |   escalator.py)   |  human review
                 +-------------------+
```

Ragas and DeepEval are not competitors so much as complementary layers in many real teams' stacks — Ragas for RAG-specific grounding metrics, DeepEval (or a promptfoo `llm-rubric`) for the broader composite/CI-gate layer, and an observability tool like Phoenix for the human-escalation loop.
