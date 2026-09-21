# Videos and Courses — Prompt Lifecycle: Versioning and Statistical Evaluation

Curated external video/course resources. Each entry was checked to exist via search/fetch at the time of writing (mid-2026). Where an exact runtime wasn't independently confirmable, it is omitted rather than guessed.

---

## ★★★★★ Evaluating and Debugging Generative AI (DeepLearning.AI)

- **Creator/Channel:** DeepLearning.AI, taught by **Carey Phelps** (Founding Product Manager, Weights & Biases)
- **Duration:** ~1 hour (7 video lessons + 5 code examples + 1 graded assignment)
- **Difficulty:** Beginner → Intermediate
- **Why it's worth watching:** This is the closest official-course match to this module's core theme: it walks through instrumenting notebooks to log configs, metrics, datasets, model versions, and **traces of prompts over time** — exactly the "immutable prompt log with metadata" pattern the transcripts describe, but shown live inside a real tool (W&B) instead of just as a slide concept.
- **Complements:** Part 1 (Logging Prompt Variants — semantic versioning, metadata, immutability).
- **Link:** https://www.deeplearning.ai/short-courses/evaluating-debugging-generative-ai/

---

## ★★★★★ Automated Testing for LLMOps (DeepLearning.AI)

- **Creator/Channel:** DeepLearning.AI, taught by **Rob Zuber** (CTO, CircleCI)
- **Duration:** ~1 hour 12 minutes (6 video lessons + 4 code examples + 1 graded assignment)
- **Difficulty:** Intermediate
- **Why it's worth watching:** Directly builds the "automate the comparison workflow" idea from the transcripts — rules-based tests, model-graded ("LLM-as-judge") tests, and wiring both into a CI pipeline that blocks a release when a check fails. This is the industrial version of the `delta_check.py` pattern: instead of one script, a full pipeline stage that gates merges/deploys.
- **Complements:** Part 2 (Tracking Prompt/Response Deltas — automated gates, delta_check.py, auto-promote/auto-block).
- **Link:** https://www.deeplearning.ai/short-courses/automated-testing-llmops/

---

## ★★★★★ A/B Testing (Udacity, Google)

- **Creator/Channel:** Udacity, taught by **Carrie Grimes, Caroline Buckey, Diane Tang** (Google)
- **Difficulty:** Beginner (no prior stats required, though it moves fast into real content)
- **Cost:** Free
- **Why it's worth watching:** This is the canonical, industry-standard treatment of A/B testing methodology — choosing metrics, sizing experiments, designing a fair comparison, and analyzing/interpreting results (including the difference between statistical and practical significance). Everything the third transcript compresses into a few slides (fixed eval set, n≥200, paired test, p<0.05 + practical-significance threshold) is this course's entire syllabus, applied generally to online experiments rather than specifically to prompts — the skill transfers directly.
- **Complements:** Part 3 (Statistical Testing for Prompt Promotion — significance vs. practical significance, sample sizing, A/B test design).
- **Link:** https://www.udacity.com/course/ab-testing--ud257
- **Rating (per Udacity):** 4.7/5 (61 reviews)

---

## ★★★★☆ StatQuest: p-values, t-tests, and Power Analysis (Josh Starmer)

- **Creator/Channel:** StatQuest with Josh Starmer (YouTube) — widely used by ML engineers to build statistics intuition
- **Difficulty:** Beginner
- **Why it's worth watching:** Josh Starmer's videos on p-values, hypothesis testing, and statistical power are the standard "learn this concept in plain English with a picture" resource that most senior engineers cite when they had to relearn stats for evaluation work. Watch the p-value video and the power-analysis video back to back before touching `scipy.stats.ttest_rel` or `statsmodels.stats.power` — they build the intuition for *why* n≥200 and p<0.05 appear in the transcript's promotion rule, rather than treating them as magic numbers.
- **Complements:** Part 3 (why paired t-tests, why sample size matters, what a p-value actually means).
- **Note:** Search "StatQuest p-value" and "StatQuest power analysis" on the StatQuest channel — exact video titles have been slightly renamed over the years, but the channel and topic coverage are stable and authoritative.

---

## ★★★★☆ MLflow Prompt Registry — official product walkthrough

- **Creator/Channel:** Databricks / MLflow project (official docs site includes embedded walkthroughs and the MLflow YouTube channel covers GenAI releases)
- **Difficulty:** Intermediate
- **Why it's worth watching:** Shows the concrete mechanics of `register_prompt`, immutable version numbers, alias-based promotion (`production`/`staging`), and rollback-by-repointing-alias — i.e., a production-grade implementation of everything Part 1 of the transcripts describes conceptually. Use this to ground the "prompt registry design" section of the tutorial in a real, currently-maintained tool rather than a hypothetical.
- **Complements:** Part 1 (registry design, immutability guarantees, rollback).
- **Link:** https://mlflow.org/docs/latest/genai/prompt-version-mgmt/prompt-registry/

---

## How to use this list

1. Watch the DeepLearning.AI **Evaluating and Debugging Generative AI** short course first — it's the fastest path to seeing prompt/version/metric logging in a real notebook.
2. Then **Automated Testing for LLMOps** to see the delta-check idea become a CI gate.
3. Read/skim the **MLflow Prompt Registry** docs (video or text) to see a real immutable registry with aliases.
4. Do the StatQuest p-value + power-analysis videos if paired t-tests and sample-size formulas feel like a black box.
5. Finish with the Google **A/B Testing** Udacity course for the full methodology of designing a fair, adequately powered experiment — this is the deepest, most rigorous resource on this list and pays off well beyond prompt evaluation.
