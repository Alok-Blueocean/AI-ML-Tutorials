# Videos — Module 03: Versioning, Registries, and Rollback

Curated, verified videos that complement the three source transcripts (semantic
versioning & the deployment triple / registry comparison & promotion criteria /
rollback & lineage). Ranked by how directly each one supports the module content
you should build hands-on skill from.

---

## ★★★★★ Learn MLOps with MLflow and Databricks – Full Course for Machine Learning Engineers
- **Creator/Channel:** freeCodeCamp.org (YouTube)
- **URL:** https://www.youtube.com/watch?v=tVskbekONlw
- **Duration:** ~5 hours (full course)
- **Difficulty:** Beginner → Intermediate
- **Why it's worth watching:** This is the single best long-form, free video resource for this module. It walks through experiment tracking, model logging, the Model Registry lifecycle (register → stage/alias → promote → archive), and — importantly for 2026 — it also covers MLflow's newer GenAI surface: the **Prompt Registry** for versioning prompt templates and LLM-as-a-judge evaluation gates before promotion. That maps almost one-to-one onto the "deployment triple" (model + prompt + dataset) idea from the source transcripts.
- **Complements:** Transcript 1 (versioning + deployment triple + MLflow promotion) and Transcript 2 (registry mechanics, promotion criteria).

## ★★★★★ MLOps Zoomcamp — Module 2: Experiment Tracking and Model Management
- **Creator/Channel:** DataTalksClub (YouTube playlist)
- **URL:** https://www.youtube.com/playlist?list=PL3MmuxUbc_hIUISrluw_A7wDSmfOhErJK
- **Companion notebook:** https://github.com/DataTalksClub/mlops-zoomcamp/blob/main/02-experiment-tracking/model-registry.ipynb
- **Difficulty:** Intermediate
- **Why it's worth watching:** A free, structured, project-based module taught with real code, not slides. It explicitly frames the registry as a "waiting room" for models awaiting promotion to staging/production/archive — the exact mental model the source transcripts use. The accompanying notebook is a working, runnable MLflow registry walkthrough you can adapt directly into the module's code examples.
- **Complements:** Transcript 1 & 2 — MLflow registry walkthrough, promotion mechanics.

## ★★★★☆ MLflow on Databricks End-to-End Tutorial | Experiments, Registry, Serving, Nested Runs
- **Creator/Channel:** YouTube (Databricks-ecosystem tutorial channel)
- **URL:** https://www.youtube.com/watch?v=9AenofD8GZ8
- **Difficulty:** Intermediate
- **Why it's worth watching:** Covers the full arc from experiment run → registered model → alias-based promotion → serving endpoint, in one continuous walkthrough. Useful for seeing how registry promotion connects downstream to actual serving infrastructure, which is exactly where rollback decisions get executed in production.
- **Complements:** Transcript 2 (registry choice/promotion) and Transcript 3 (rollback in a serving context).

## ★★★★☆ MLflow & Databricks Model Serving Explained | MLOps Concepts & Deployment
- **URL:** https://www.youtube.com/watch?v=UmHISXgPhGk
- **Difficulty:** Intermediate
- **Why it's worth watching:** A more conceptual (less click-through-the-UI) explanation of how registry state (aliases like `@champion`/`@challenger`) drives what a serving layer actually loads — useful for understanding *why* alias-based promotion enables safe canary/rollback patterns instead of the old fixed-stage model.
- **Complements:** Transcript 2 (promotion criteria) and Transcript 3 (canary releases, rollback).

## ★★★☆☆ Weights & Biases: Quick Start Tutorial (Log Metrics & Models Fast)
- **Creator/Channel:** YouTube, "Master MLOps" style walkthrough
- **URL:** https://www.youtube.com/watch?v=iq8NFthBffM
- **Difficulty:** Beginner
- **Why it's worth watching:** Fastest way to see the W&B Artifacts + Model Registry mental model in action (lineage graph between runs, datasets, and model artifacts) so you can contrast it directly against MLflow's registry when building the comparison table for this module.
- **Complements:** Transcript 2 (MLflow vs W&B vs custom registry comparison).

## ★★★☆☆ Productionalizing Models through CI/CD Design with MLflow
- **URL:** https://m.youtube.com/watch?v=wpFDsvXORuU
- **Difficulty:** Advanced
- **Why it's worth watching:** Focuses on wiring registry promotion into a CI/CD pipeline — i.e., turning "promote to production" from a manual UI click into a gated pipeline step (regression tests, accuracy/latency thresholds), which is precisely the `release.yaml` pattern and promotion-criteria material in the source transcripts.
- **Complements:** Transcript 1 (release.yaml pattern) and Transcript 2 (promotion criteria/thresholds).

---

### Watching order recommendation
1. DataTalksClub Module 2 (concepts + registry mental model, free, structured)
2. freeCodeCamp full course (breadth: registry + prompt registry + LLMOps)
3. Databricks end-to-end tutorial (see it wired into serving)
4. W&B quick start (contrast a second registry implementation)
5. CI/CD with MLflow (see promotion criteria enforced by a pipeline, not a human)
