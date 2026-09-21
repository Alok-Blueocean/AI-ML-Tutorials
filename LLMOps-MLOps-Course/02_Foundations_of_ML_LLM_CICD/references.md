# References — Module 02: Foundations of ML & LLM CI/CD

Official documentation, papers/articles, and engineering blog posts backing this
module's claims about ML/LLM CI/CD architecture, pipeline mechanics, and validation
gates.

---

## Official documentation

### Pandera
- **Link:** https://pandera.readthedocs.io/
- **Teaches:** DataFrame schema validation — structure (required columns), types, and
  value constraints (`Check` objects); `lazy=True` to collect all failures at once
  instead of stopping at the first error; supported backends (pandas, polars,
  pyspark, ibis) via a Narwhals-powered engine as of the 2026 release line.
- **Relevant pages:** [DataFrame Schemas](https://pandera.readthedocs.io/en/stable/dataframe_schemas.html),
  [DataFrame Models](https://pandera.readthedocs.io/en/stable/dataframe_models.html),
  [Validating with Checks](https://pandera.readthedocs.io/en/stable/checks.html),
  [Data Type Validation](https://pandera.readthedocs.io/en/stable/dtype_validation.html)
- **Difficulty:** Beginner–Intermediate | **Reading time:** ~30–45 min for the core pages.

### MLflow — Model Registry & Evaluation
- **Link:** https://mlflow.org/docs/latest/ml/model-registry/ and
  https://mlflow.org/docs/latest/ml/model-registry/workflow/
- **Teaches:** Centralized model store with versioning, lineage, aliasing, and
  metadata tagging; how CI/CD stages can request/approve stage transitions so
  promotion is gated by a test suite rather than a manual UI click.
- **Also see:** https://mlflow.org/articles/mlops-pipeline-automation-best-practices-in-2026/
  — MLflow's own 2026 guidance article on wiring evaluation gates directly into CI/CD,
  explicitly framing "model evaluation gates before deployment" as one of the
  highest-impact practices for preventing production incidents in automated ML systems.
- **Difficulty:** Intermediate | **Reading time:** ~45–60 min.

### Google Cloud — MLOps: Continuous delivery and automation pipelines in machine learning
- **Link:** https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning
- **Teaches:** The canonical CI/CD/CT (continuous integration / continuous delivery /
  continuous training) framework for ML systems — CI extended to include testing and
  validating data, data schemas, and models (not just code); CD extended to deploying
  a whole training pipeline that itself outputs a prediction service; CT as the
  ML-specific addition of automatically retraining and serving models. Also covers
  "experimental-operational symmetry" (the same pipeline code must run in dev and
  prod) — a good rebuttal to naively porting DevOps pipelines to ML.
- **Difficulty:** Advanced | **Reading time:** ~45–60 min.

### GitHub Actions — official docs
- **Link:** https://docs.github.com/en/actions
- **Teaches:** Workflow syntax (`on:`, `jobs:`, `steps:`), the `schedule:` (cron)
  trigger used for the daily 3 a.m. churn-retrain example in this module, job
  dependencies (`needs:`), and exit-code-based step failure propagation — the exact
  mechanism `validate_data.py`'s `sys.exit(1)` relies on to block a workflow.
- **Difficulty:** Beginner | **Reading time:** ~30 min for triggers + jobs pages.

### SciPy — `scipy.stats.ks_2samp`
- **Link:** https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.ks_2samp.html
- **Teaches:** The two-sample Kolmogorov–Smirnov test used for distribution-shift
  (drift) checks — returns a test statistic and p-value; a low p-value (conventionally
  <0.05) is read as evidence the two samples come from different distributions,
  i.e. drift.
- **Difficulty:** Intermediate | **Reading time:** ~15–20 min.

### Evidently AI — documentation
- **Link:** https://docs.evidentlyai.com/
- **Teaches:** How to build "Test Suites" (pass/fail data and model quality checks
  meant for CI/CD) on top of Reports, including pre-built drift detectors (PSI, KS,
  Jensen–Shannon, chi-square) so teams don't hand-roll the distribution-check gate
  from raw SciPy calls.
- **Difficulty:** Intermediate | **Reading time:** ~30–40 min.

---

## Papers / long-form articles

### Continuous Delivery for Machine Learning (CD4ML)
- **Authors:** Danilo Sato, Arif Wider, Christoph Windheuser (ThoughtWorks)
- **Link:** https://martinfowler.com/articles/cd4ml.html
- **Teaches:** ML applications vary along three independent axes — code, model, and
  data — and CD4ML is the discipline of applying Continuous Delivery practices across
  all three simultaneously so a cross-functional team can release small, safe,
  reproducible increments at any time.
- **Difficulty:** Intermediate | **Reading time:** ~40–50 min.

### MLOps: Continuous Delivery and Automation Pipelines in Machine Learning (Google whitepaper)
- **Link:** https://cloud.google.com/resources/mlops-whitepaper (landing page) /
  full architecture doc listed above.
- **Teaches:** Same CI/CD/CT framework as above, in whitepaper form; widely cited as
  one of the most influential documents in the MLOps field and the likely origin of
  the "three extra gates" framing used throughout this module.
- **Difficulty:** Advanced | **Reading time:** ~50–60 min.

### MLOps: Continuous Delivery for Machine Learning on AWS
- **Link:** https://d1.awsstatic.com/whitepapers/mlops-continuous-delivery-machine-learning-on-aws.pdf
- **Teaches:** AWS's own reference architecture for ML CI/CD/CT, useful as a
  cross-cloud comparison point against the Google whitepaper above (same principles,
  different managed-service mapping — SageMaker Pipelines / CodePipeline instead of
  Vertex Pipelines / Cloud Build).
- **Difficulty:** Advanced | **Reading time:** ~40 min.

---

## Engineering blog posts (how real companies structure this)

### Uber — Meet Michelangelo: Uber's Machine Learning Platform
- **Link:** https://www.uber.com/en-US/blog/michelangelo-machine-learning-platform/
- **Teaches:** How Uber built an internal ML-as-a-service platform covering the full
  workflow (manage data → train → evaluate → deploy → predict → monitor) so that
  individual product teams don't each reinvent a validation/evaluation gate. Useful
  primary source for reasoning about how a company like Uber would structure the
  "three extra gates" as shared platform infrastructure rather than per-team scripts.
- **Difficulty:** Intermediate | **Reading time:** ~20–25 min.

### Netflix TechBlog — Open-Sourcing Metaflow, a Human-Centric Framework for Data Science
- **Link:** https://netflixtechblog.com/open-sourcing-metaflow-a-human-centric-framework-for-data-science-fa72e04a5d9
- **Teaches:** Why Netflix's ML infrastructure team found that data scientists' biggest
  obstacle to production wasn't compute or data scale but ordinary software-engineering
  friction — versioning, dependency management, and reliably chaining pipeline steps —
  which is exactly the gap this module's "three extra gates" close. Useful grounding
  for the module's illustrative Netflix recommendation-pipeline scenario: a real
  Netflix-style pipeline would express each gate (data validation, training,
  evaluation) as a distinct, independently retriable Metaflow/orchestration step.
- **Difficulty:** Intermediate | **Reading time:** ~15 min.

### Netflix TechBlog — Scaling Media Machine Learning at Netflix
- **Link:** https://netflixtechblog.com/scaling-media-machine-learning-at-netflix-f19b400243
- **Teaches:** A concrete example of an entire ML pipeline (Match Cutting) encapsulated
  as a single Metaflow flow with each step mapped to a pipeline stage for resource
  control — a real-world instance of the "trigger-driven, staged, gated pipeline"
  principle from Lesson 2.
- **Difficulty:** Intermediate–Advanced | **Reading time:** ~20 min.

### Great Expectations — Continuous Integration for your data with GitHub Actions
- **Link:** https://greatexpectations.io/blog/github-actions/
- **Teaches:** A worked example of wiring a data-validation framework (an alternative
  to Pandera, with a similar "assert expectations, fail the build" model) directly
  into a GitHub Actions workflow — useful as a second implementation pattern for
  Lesson 3's schema/row-count validation gate.
- **Difficulty:** Beginner–Intermediate | **Reading time:** ~15–20 min.

---

## Currency notes (mid-2026)

- **MLflow** is on the 3.1x release line (e.g. 3.14.0, June 2026); the model registry
  and `mlflow.evaluate` APIs referenced above are stable and this is the version to
  install for new pipelines (`pip install mlflow`, requires Python ≥3.10).
- **Pandera** has moved past the pandas-only validator implied by older tutorials: as
  of its 2026 releases it ships a Narwhals-powered backend giving lazy validation
  across pandas, polars, pyspark, and ibis — worth mentioning to learners who only
  know the pandas-only version from 2023–2024-era blog posts.
- **DeepLearning.AI's "Machine Learning Engineering for Production (MLOps)"**
  Coursera specialization closed to new enrollment; treat it as historical/background
  reading via community notes repos rather than a course to actively assign.
