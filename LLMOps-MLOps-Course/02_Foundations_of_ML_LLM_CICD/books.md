# Books — Module 02: Foundations of ML & LLM CI/CD

Books and specific chapters that deepen the theory behind this module: why ML/LLM
delivery needs extra gates beyond DevOps, and how to design build/test/deploy
pipelines with data and model validation baked in.

---

## 1. Designing Machine Learning Systems — Chip Huyen (O'Reilly, 2022)

- **What it teaches:** The most widely recommended senior-level ML systems book.
  Chapters on "Data Engineering Fundamentals," "Training Data," "Model Development
  and Offline Evaluation," and "Continual Learning and Test in Production" give the
  conceptual backbone for why a model pipeline needs schema/distribution/row-count
  checks (this module's Lesson 3) and why silent degradation, not crashes, is the
  default failure mode in ML (Lesson 1). Its discussion of "natural labels vs. delayed
  feedback" and CT (continuous training) pairs directly with the churn-retrain example
  in Lesson 2.
- **Difficulty:** Intermediate–Advanced
- **Estimated reading time:** ~10–12 hours for the whole book; ~2 hours for the two
  chapters most relevant to this module (Ch. 8 "Model Deployment and Prediction
  Service" and Ch. 9 "Continual Learning and Test in Production").
- **Why it matters for this module:** It is the standard reference senior MLOps
  interview loops draw vocabulary from (e.g. "train-serving skew," "concept drift vs.
  covariate drift," "shadow deployment") — the same vocabulary this module's transcripts
  use informally ("model trained on corrupted data silently," "recommendation quality
  dropped").
- **Companion repo:** `chiphuyen/dmls-book` on GitHub (chapter summaries and resources).

---

## 2. Machine Learning Design Patterns — Lakshmanan, Robinson & Munn (O'Reilly, 2021)

- **What it teaches:** A patterns catalog. Two patterns map almost one-to-one onto this
  module: **Workflow Pipeline** (Pattern 25 — containerize each pipeline stage so the
  whole train→validate→deploy sequence is portable and reproducible, exactly the
  cron-triggered GitHub Actions shape in Lesson 2) and **Continuous Model Evaluation**
  (Pattern 18 — monitor deployed model performance and retrain on a schedule or on
  threshold breach, matching the AUC-vs-baseline gate in Lesson 2/3).
- **Difficulty:** Intermediate
- **Estimated reading time:** ~45–60 minutes for the two relevant patterns; ~8 hours
  for the full book.
- **Why it matters for this module:** Gives precise, reusable vocabulary ("pattern
  name") for architecture decisions you'll otherwise have to describe from scratch in
  an interview or design doc.

---

## 3. Reliable Machine Learning: Applying SRE Principles to ML in Production — Chen, Murphy, Parisa, Sculley, Underwood (O'Reilly, 2022)

- **What it teaches:** Written by Google SRE and ML engineers; reframes "is the model
  healthy" as an SLO problem, not just an accuracy number. Chapters on "ML Training
  Pipelines," "Build and Validate Applications," "Quality and Performance Evaluation,"
  and "Defining and Measuring SLOs" are the most relevant — they formalize this
  module's claim that "a green deployment status does not always mean business
  success" into concrete SLO/error-budget language.
- **Difficulty:** Advanced
- **Estimated reading time:** ~6–8 hours full book; ~90 minutes for the four relevant
  chapters.
- **Why it matters for this module:** Bridges this module's "silent production
  failures" theme to the SRE discipline that Netflix/Google/Uber-scale teams actually
  use to decide when a quality regression counts as an incident.

---

## 4. Practical MLOps: Operationalizing Machine Learning Models — Noah Gift & Alfredo Deza (O'Reilly, 2021)

- **What it teaches:** A hands-on, tool-forward book. Chapter "Configuring Continuous
  Integration with GitHub Actions" (and the surrounding "DevOps and MLOps" / "An MLOps
  Hierarchy of Needs" chapters) is a near-literal walkthrough of the exact stack this
  module uses (GitHub Actions, Python-based validation scripts, promotion gates).
- **Difficulty:** Beginner–Intermediate
- **Estimated reading time:** ~1 hour for the CI chapter; ~7 hours full book.
- **Why it matters for this module:** Best "read this, then go write the YAML"
  companion — least abstract of the four books listed here.
- **Companion repo:** `paiml/practical-mlops-book` on GitHub (code samples).

---

## 5. Continuous Delivery for Machine Learning (CD4ML) — Danilo Sato, Arif Wider, Christoph Windheuser (ThoughtWorks / martinfowler.com, 2019)

This is a long-form article series, not a book, but it is treated as required reading
in most senior MLOps curricula and is short enough to read in one sitting — listed
here because of its weight, with the direct link filed in `references.md`. It
introduced the now-standard framing that ML systems vary along three axes
simultaneously — code, model, and data — which is the exact structure Lesson 1 of
this module uses to explain why ML Ops "manages software plus model delivery."

- **Difficulty:** Intermediate
- **Estimated reading time:** ~40–50 minutes
