# GitHub Repositories — Prompt Lifecycle: Versioning and Statistical Evaluation

All repositories below were verified to exist (fetched directly) at the time of writing, mid-2026. Popularity is described in tiers rather than exact live star counts (which drift daily), except where a specific count was directly observed during research — those are marked "as observed" with an approximate figure, not a guaranteed live number.

---

## mlflow / mlflow

- **URL:** https://github.com/mlflow/mlflow
- **Purpose:** MLflow is the most widely adopted open-source ML lifecycle platform (tracking, models, registry). As of the MLflow 3.x GenAI release line, it ships a first-class **Prompt Registry** (`mlflow.genai.register_prompt`, `load_prompt`, `search_prompts`) with Git-style immutable versions, commit messages, alias-based promotion (`production`/`staging`), and lineage links from prompt versions to evaluation runs/traces.
- **Popularity tier:** Very popular / de-facto standard in the open-source MLOps ecosystem — one of the most widely deployed experiment-tracking and model-registry tools in the industry.
- **Relation to this module:** This is the single most direct real-world implementation of Part 1 of the module (semantic-versioned, immutable, metadata-rich prompt logging). The tutorial's "prompt registry design" section should be read as "here is what MLflow's real registry already gives you for free, and here is what you'd still need to build on top of it (delta gating, statistical promotion tests)."

---

## promptfoo / promptfoo

- **URL:** https://github.com/promptfoo/promptfoo
- **Purpose:** A CLI and library for evaluating, testing, comparing, and red-teaming LLM prompts and applications across providers (OpenAI, Anthropic, Azure, Bedrock, Ollama, etc.). Runs evaluations locally, supports side-by-side comparison matrices, and is explicitly designed to be wired into CI/CD as an automated gate ("automate checks in CI/CD... make decisions based on metrics, not gut feel").
- **Popularity tier:** Very popular — one of the most widely used open-source LLM-eval tools (observed ~24k GitHub stars during research, mid-2026; treat as an approximate point-in-time figure, not a guarantee).
- **Relation to this module:** The closest open-source analog to the `delta_check.py` pattern described in Part 2 — promptfoo's YAML-configured test suites plus its CI integration are essentially a productionized, reusable version of "run baseline and candidate prompt on the same fixed eval set, compare metrics, block on regression."

---

## confident-ai / deepeval

- **URL:** https://github.com/confident-ai/deepeval
- **Purpose:** An LLM evaluation framework built to feel like Pytest for LLM outputs — unit-testable metrics (answer relevancy, faithfulness/hallucination, G-Eval-style LLM-as-judge scoring, bias/toxicity, task completion) that run locally and plug into standard CI.
- **Popularity tier:** Very popular / widely adopted (observed ~17k GitHub stars during research, mid-2026, approximate).
- **Relation to this module:** Demonstrates how to turn the "multi-metric delta" idea from Part 2 into actual assertions in a test suite — i.e., how a team would literally write the automated gate that blocks promotion when correctness regresses or tone/format drifts, using metrics libraries instead of hand-rolled comparison code.

---

## openai / evals

- **URL:** https://github.com/openai/evals
- **Purpose:** OpenAI's framework and registry of evaluations for benchmarking LLMs and LLM-based systems, including a way to define custom, task-specific evals against a fixed dataset.
- **Popularity tier:** Very popular / foundational reference in the LLM-eval space (observed ~19k GitHub stars during research, mid-2026, approximate).
- **Relation to this module:** A canonical example of the "fixed, versioned evaluation dataset run against multiple model/prompt variants" pattern that underlies both Part 2 (delta tracking) and Part 3 (statistical promotion testing) of this module — useful as a reference for how a major AI lab structures eval-set definitions and scoring functions.

---

## growthbook / growthbook

- **URL:** https://github.com/growthbook/growthbook
- **Purpose:** Open-source feature-flagging and A/B-testing/experimentation platform with SDKs for many languages, warehouse-native architecture (queries your own BigQuery/Snowflake/Databricks data), and built-in statistical methods including sequential testing, Bayesian analysis, CUPED variance reduction, and sample-ratio-mismatch (SRM) checks.
- **Popularity tier:** Popular / actively used in production experimentation stacks (observed ~8k GitHub stars during research, mid-2026, approximate).
- **Relation to this module:** While GrowthBook isn't prompt-specific, it is a real, production-grade implementation of the general A/B-testing statistical machinery Part 3 teaches in miniature (paired significance testing, minimum-detectable-effect sizing, guardrails against false positives). Studying how a mature experimentation platform structures experiment definitions, metrics, and stopping rules is directly transferable to building an in-house prompt A/B testing framework.

---

## scipy / scipy (stats module) and statsmodels / statsmodels

- **URLs:** https://github.com/scipy/scipy · https://github.com/statsmodels/statsmodels
- **Purpose:** The two foundational Python statistics libraries used to actually implement the tests this module teaches: `scipy.stats.ttest_rel` (paired t-test — exactly what the transcript's `scipy.stats.ttest` example is doing) and `statsmodels.stats.power` (`TTestPower`, `TTestIndPower`, `tt_solve_power`) for power analysis / sample-size calculation.
- **Popularity tier:** Extremely popular / core scientific-Python infrastructure, used across virtually the entire Python data/ML ecosystem.
- **Relation to this module:** These are the actual libraries the module's code examples are built on — `delta_check.py` and the A/B test script both reduce, in practice, to a few lines of `scipy.stats` and (for sample-size planning) `statsmodels.stats.power`.

---

## langchain-ai / langsmith-cookbook (historical reference — archived)

- **URL:** https://github.com/langchain-ai/langsmith-cookbook
- **Purpose:** Previously contained worked examples of LangSmith Prompt Hub versioning workflows (committing, tagging, comparing prompt versions).
- **Popularity tier:** Was moderately popular as a cookbook reference; **this repository was archived (read-only) in February 2026** as LangChain consolidated cookbook content into the main `docs.langchain.com` site.
- **Relation to this module:** Useful only as a historical pointer — for current guidance on LangSmith's prompt versioning/rollback model (commits, tags-as-aliases, staging/production environments with rollback history), use the live documentation at https://docs.langchain.com/langsmith/manage-prompts instead of this archived repo.

---

## How these fit together

| Concern from the transcripts | Repo(s) that implement it |
|---|---|
| Immutable, semantic-versioned prompt log + metadata | `mlflow/mlflow` (Prompt Registry) |
| Fixed eval set, multi-metric delta report, CI gate | `promptfoo/promptfoo`, `confident-ai/deepeval`, `openai/evals` |
| Paired t-test / significance math | `scipy/scipy`, `statsmodels/statsmodels` |
| Full A/B experimentation platform patterns (sizing, guardrails, sequential testing) | `growthbook/growthbook` |
