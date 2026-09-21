# GitHub Repositories — Module 02: Foundations of ML & LLM CI/CD

Real, verifiable repositories that show working implementations of the concepts in
this module: trigger-driven pipelines, data validation gates, model evaluation gates,
and GitHub Actions-based retraining.

---

## 1. `GokuMohandas/mlops-course` (Made With ML)

- **Purpose:** A full project-based MLOps curriculum (also published as a course at
  madewithml.com) that builds a real ML application end to end — including a
  dedicated CI/CD section (`madewithml.com/courses/mlops/cicd/`) that wires GitHub
  Actions workflows to run tests, validate data, and retrain/deploy on pull requests
  and pushes.
- **Popularity tier:** Very popular / widely adopted — one of the most-cited
  open-source MLOps teaching repos, with an active community (tens of thousands of
  developers have gone through the material).
- **Why it matters:** It is the single best "see the whole thing wired together" repo
  for this module — every gate this module discusses in the abstract (schema/data
  checks, evaluation-before-promotion, CI-triggered retrain) appears as actual,
  runnable workflow files.
- **Relation to this module:** Direct implementation reference for Lessons 2 and 3.

---

## 2. `iterative/cml` (CML — Continuous Machine Learning)

- **Purpose:** A library of CI/CD "actions" (works with GitHub Actions and GitLab CI)
  purpose-built for ML: provisioning training runners (including GPUs from cloud
  providers), pulling versioned data via DVC, training, and posting a human-readable
  report (metrics, plots, diffs) back onto the pull request.
- **Popularity tier:** Very popular / widely adopted — the reference open-source
  tool for "CI/CD for ML" as a distinct category, maintained by Iterative.ai (the DVC
  company).
- **Why it matters:** Demonstrates the "treat output as promotional artifact" and
  "fail fast" principles from Lesson 2 as literal CI report comments — a PR either
  shows a passing metrics diff or it doesn't, which is exactly the gate-then-promote
  discipline the transcripts describe.
- **Relation to this module:** Concrete, alternative tooling for the same
  trigger-driven, gated pipeline pattern taught with raw GitHub Actions YAML in
  Lesson 2.

---

## 3. `unionai-oss/pandera`

- **Purpose:** The schema-validation library named explicitly in this module's source
  transcripts (Lesson 3) for validating pandas/polars/pyspark/ibis DataFrames — column
  types, nullability, ranges, and custom checks, with support for emitting structured,
  machine-readable failure output.
- **Popularity tier:** Very popular / widely adopted — a standard dependency in
  production ML data-validation stacks, actively maintained (2026 releases added a
  Narwhals-powered lazy validation backend spanning pandas/polars/pyspark/ibis).
- **Why it matters:** This is the actual tool behind the `validate_data.py` /
  "Pandera schema + `sys.exit(1)` on failure" pattern this module teaches as the
  hard gate before training.
- **Relation to this module:** Direct tool reference for Lesson 3's schema-validation
  gate.

---

## 4. `evidentlyai/evidently`

- **Purpose:** An open-source ML/LLM observability library with 100+ built-in metrics,
  including data drift detection (PSI, Kolmogorov–Smirnov, Jensen–Shannon, and more),
  packaged as "Test Suites" that can be dropped straight into a CI pipeline as
  pass/fail checks.
- **Popularity tier:** Very popular / widely adopted — commonly cited as the leading
  open-source tool in the ML-monitoring/data-drift category, with millions of
  downloads.
- **Why it matters:** Where Lesson 3 mentions "SciPy or PSI" for distribution
  validation, Evidently is the production-grade library that packages exactly those
  statistical tests (PSI thresholds like <0.1 stable / 0.1–0.25 slight drift / >0.25
  significant drift) into a reusable, CI-friendly test suite instead of hand-rolled
  SciPy calls.
- **Relation to this module:** Extends Lesson 3's distribution-check gate from a
  teaching-sized script into a tool a real team would actually adopt.

---

## 5. `mlflow/mlflow`

- **Purpose:** Experiment tracking, model registry, and (as of the 2.x/3.x line)
  built-in model evaluation APIs (`mlflow.evaluate`) — the tool named explicitly in
  Lesson 3 for the "accuracy vs. baseline × 0.99" model-evaluation gate and in
  Lesson 2 for auto-registering a model with run ID, dataset hash, and timestamp.
- **Popularity tier:** Very popular / widely adopted — the de facto standard open-source
  model registry and experiment tracker across the industry; actively developed
  (release cadence through mid-2026 running in the 3.1x line).
- **Why it matters:** Shows exactly how "promote only if the new model beats the
  registered baseline" gates are implemented against a real model registry API,
  including stage transitions that can require CI-gated approval before a model
  becomes the production alias.
- **Relation to this module:** Direct tool reference for the model-evaluation gate
  and the "MLflow registry with run ID / dataset hash" step described in Lesson 2.

---

## 6. `machine-learning-apps/actions-ml-cicd`

- **Purpose:** A curated collection of pre-built GitHub Actions specifically for
  MLOps use cases (data checks, model comparison reports, etc.), meant to be composed
  into a workflow YAML rather than written from scratch.
- **Popularity tier:** Moderately popular — a smaller, more niche repo than the others
  above, but useful as a source of copy-pasteable Action building blocks.
- **Why it matters:** Good reference when implementing this module's end-to-end
  GitHub Actions example — shows idiomatic ways to structure `jobs:`/`steps:` for
  ML-specific gates rather than reusing generic software-CI actions.
- **Relation to this module:** Implementation aid for Lesson 2's GitHub Actions
  pipeline mechanics.

---

## 7. `chiphuyen/dmls-book`

- **Purpose:** Official companion repository (chapter summaries, code pointers, and
  discussion resources) for *Designing Machine Learning Systems*.
- **Popularity tier:** Very popular / widely adopted among ML engineering readers.
- **Why it matters:** A fast, free way to review the book's CI/CD-and-testing-adjacent
  chapters (data validation, continual learning, test-in-production) without needing
  the physical book on hand.
- **Relation to this module:** Companion to the `books.md` entry above.
