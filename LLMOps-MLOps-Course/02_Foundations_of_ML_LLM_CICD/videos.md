# Videos — Module 02: Foundations of ML & LLM CI/CD

Curated, verified videos that complement the three source lessons for this module:
(1) how ML/LLM CI/CD differs from DevOps and the three extra gates, (2) build/test/deploy
pipeline mechanics with trigger-driven retraining, (3) data/model validation gates
(Pandera, SciPy/PSI, MLflow) inside CI/CD.

Difficulty legend: Beginner = no prior MLOps exposure needed. Intermediate = comfortable
with Python + basic CI/CD. Advanced = assumes production ML systems experience.

---

## 1. MLOps: Continuous Delivery and Automation Pipelines in Machine Learning (Google Cloud Tech)

- **Creator/Channel:** Google Cloud Tech (YouTube)
- **Difficulty:** Advanced
- **Why it's worth watching:** This is the video treatment of Google's canonical MLOps
  whitepaper — the same document that popularized the CI (validate data + models,
  not just code) / CD (deploy a whole training pipeline, not just a binary) / CT
  (continuous training) framing used throughout this module. It is the closest thing
  the industry has to an "official" definition of why ML CI/CD needs extra gates, and
  it directly backs up transcript 1's claim that DevOps is code-centric while ML Ops
  must also version data and models.
- **Complements:** Lesson 1 (ML/LLM CI/CD architecture vs. DevOps, the three extra gates).
- **Rating:** ★★★★★

## 2. MLOps Coffee Sessions — "Continuous Delivery and Automation Pipelines in ML" (Parts 1–3)

- **Creator/Channel:** MLOps Community (YouTube)
- **Difficulty:** Advanced
- **Why it's worth watching:** A three-part practitioner roundtable that dissects the
  same Google whitepaper section by section, arguing about where the theory breaks
  down in real production teams (e.g., how strict a data-schema gate should really be,
  what "continuous training" costs at scale). Useful as a counterweight to the
  polished vendor narrative — it surfaces the operational friction that a course
  transcript can't.
- **Complements:** Lesson 1 (silent production failures, why green deploys can still
  mean business failure) and Lesson 3 (validation gate design tradeoffs).
- **Rating:** ★★★★☆

## 3. MLOps Tutorial #1: Continuous Integration (CI/CD) for ML Pipelines with GitHub Actions

- **Creator/Channel:** Iterative.ai / DVC (YouTube)
- **Difficulty:** Beginner–Intermediate
- **Why it's worth watching:** A hands-on walkthrough building an actual GitHub Actions
  workflow that trains a model, runs metrics checks, and posts results back to a pull
  request using CML (Continuous Machine Learning). This is the most direct video
  analog of transcript 2's churn-model GitHub Actions example — same trigger-driven,
  gate-then-promote shape, but you can pause and read the YAML yourself.
- **Complements:** Lesson 2 (trigger-driven pipelines, GitHub Actions mechanics, fail-fast).
- **Rating:** ★★★★★

## 4. MLOps Tutorial #3: Track ML Models with Git & GitHub Actions

- **Creator/Channel:** Iterative.ai / DVC (YouTube)
- **Difficulty:** Intermediate
- **Why it's worth watching:** Follows on from #3 above to show how model artifacts and
  metrics get versioned alongside code so that a pipeline rerun is reproducible — a
  direct illustration of the idempotency principle emphasized at the end of transcript 2
  ("same 30-day input produces the same model hash").
- **Complements:** Lesson 2 (idempotency, reproducibility, promotion as artifact).
- **Rating:** ★★★★☆

## 5. MLOps Course — Build Machine Learning Production-Grade Projects

- **Creator/Channel:** freeCodeCamp.org (YouTube), instructed by Ayush Singh
- **Approx. Duration:** ~3 hours
- **Difficulty:** Beginner–Intermediate
- **Why it's worth watching:** A full, free, single-sitting course that builds an
  end-to-end pipeline (data ingestion → validation → training → evaluation → deployment)
  using ZenML and MLflow, then wires it into CI. It's a good "watch the whole thing
  once" complement after reading the module theory, because it shows the same three
  gates (data validation, model evaluation, deployment gate) implemented in a slightly
  different but transferable toolchain than the Pandera/GitHub Actions stack used in
  this module's lessons.
- **Complements:** Lessons 2 and 3 (full pipeline mechanics + validation gates end to end).
- **Rating:** ★★★★☆

---

### A note on currency (as of mid-2026)

DeepLearning.AI's original "Machine Learning Engineering for Production (MLOps)"
Coursera specialization (Andrew Ng / Robert Crowe) closed to new enrollment; its
public GitHub notes repositories (e.g. `kennethleungty/MLOps-Specialization-Notes`)
still capture the curriculum and remain a solid free substitute for the lecture
content on data validation (TFDV) and model analysis (TFMA), even though you can
no longer take the graded course directly. Prefer the Google Cloud Tech video above
plus the Made With ML written course (see `github.md`/`references.md`) for
up-to-date, actively maintained equivalents.
