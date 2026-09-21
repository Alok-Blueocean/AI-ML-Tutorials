# References — Module 03: Versioning, Registries, and Rollback

Official docs, papers, and engineering blogs, ordered roughly by how central each is to
the module's core topics: versioning theory, MLflow registry mechanics, registry
comparison, and rollback/lineage.

---

## Official Documentation

### ML Model Registry — MLflow AI Platform docs
- **URL:** https://mlflow.org/docs/latest/ml/model-registry/
- **Teaches:** The core registry data model — registered models, model versions, aliases, tags, and annotations — and the full lifecycle from registering through archiving.
- **Difficulty:** Intermediate. **Reading time:** ~30–40 minutes.

### Model Registry Workflows — MLflow AI Platform docs
- **URL:** https://mlflow.org/docs/latest/ml/model-registry/workflow/
- **Teaches:** Concrete promotion workflows using the Python client API (`transition_model_version_stage`, and the current alias-based equivalents), including UI- and API-driven promotion paths.
- **Difficulty:** Intermediate. **Reading time:** ~20 minutes.

### Model Registry Tutorials — MLflow AI Platform docs
- **URL:** https://mlflow.org/docs/latest/ml/model-registry/tutorial/
- **Teaches:** End-to-end hands-on tutorial: register a model, inspect its structure, load a specific version. This is the primary source to adapt for the module's "real Python code" walkthrough.
- **Difficulty:** Beginner–Intermediate. **Reading time:** ~25 minutes.

### MLflow Prompt Registry — MLflow AI Platform docs
- **URL:** https://mlflow.org/docs/latest/genai/prompt-registry/
- **Teaches:** MLflow 3's native prompt versioning and registry system — versioning prompt templates with diffs/commit messages, environment aliases for staged prompt deployment, and linking prompt versions to the app/model versions that use them. This is the direct, current-generation implementation of the "prompt" leg of the deployment triple.
- **Difficulty:** Intermediate. **Reading time:** ~20–25 minutes.
- **2026 relevance note:** This did not exist in earlier MLflow releases; MLflow 3 (mid-2020s release) is described by the MLflow team as the platform's biggest evolution to date, specifically to bring GenAI/prompt-versioning capability natively into the same registry used for classic ML models. If a source material predates MLflow 3, its "how do you version prompts" answer ("bolt on your own store") is now outdated — MLflow ships a first-class answer.

### Model Registry (Registry overview) — Weights & Biases docs
- **URL:** https://docs.wandb.ai/models/registry
- **Teaches:** W&B's Registry concept — an organization-wide, curated hub of Artifact versions (models and datasets) with governance/permissions, distinct from a single team's Artifacts store.
- **Difficulty:** Intermediate. **Reading time:** ~20 minutes.

### Model Registry — Comet Docs (Quickstart + Using Comet's Model Registry)
- **URLs:** https://www.comet.com/docs/v2/guides/model-registry/quickstart/ and https://www.comet.com/docs/v2/guides/model-registry/using-model-registry/
- **Teaches:** Comet's workspace-scoped registered models, explicit semantic-version strings (e.g., `1.0.0`, `2.1.5-alpha`), and status/stage tag management via the Python SDK.
- **Difficulty:** Beginner–Intermediate. **Reading time:** ~20 minutes combined.

### Data Version Control (DVC) Documentation
- **URL:** https://dvc.org/doc
- **Teaches:** Git-native data/model versioning, pipeline DAGs (`dvc.yaml`), and remote storage; the standard mechanism for pinning the "dataset" element of a deployment triple to an exact, reproducible version.
- **Difficulty:** Intermediate. **Reading time:** ~40–60 minutes for the core concepts pages.

---

## Papers

### Hidden Technical Debt in Machine Learning Systems
- **Authors:** D. Sculley, Gary Holt, Eugene Davydov, et al. (Google) — NeurIPS 2015
- **URL:** https://papers.nips.cc/paper/5656-hidden-technical-debt-in-machine-learning-systems
- **Teaches:** The theoretical foundation for why ML systems need aggressive versioning discipline — entanglement (CACE: Changing Anything Changes Everything), undeclared consumers, unstable data dependencies, and configuration debt.
- **Difficulty:** Intermediate. **Reading time:** ~30–45 minutes.

---

## Engineering Blogs

### Raising the Bar on ML Model Deployment Safety — Uber Engineering
- **URL:** https://www.uber.com/us/en/blog/raising-the-bar-on-ml-model-deployment-safety/
- **Teaches:** How Uber's Michelangelo platform layers automated safety checks (regression thresholds, canary comparisons) on top of its model registry before allowing a promotion to production traffic — a real-world instance of the "promotion criteria" concept from the source transcripts.
- **Difficulty:** Intermediate. **Reading time:** ~15 minutes.

### From Predictive to Generative — How Michelangelo Accelerates Uber's AI Journey — Uber Engineering
- **URL:** https://www.uber.com/us/en/blog/from-predictive-to-generative-ai/
- **Teaches:** How a large, mature internal ML platform (Michelangelo, including its "Gallery" model/ML-metadata registry) evolved to also register and govern GenAI artifacts — relevant to how registries are converging on tracking models, prompts, and datasets together.
- **Difficulty:** Intermediate. **Reading time:** ~15–20 minutes.

### Netflix Introduces "Model Lifecycle Graph" to Scale Enterprise Machine Learning — InfoQ (reporting on Netflix engineering work)
- **URL:** https://www.infoq.com/news/2026/05/netflix-ml-graph/
- **Teaches:** A 2026-era look at how Netflix represents model lineage as an explicit graph (model ⟷ training data ⟷ code ⟷ deployment) to support governance and rollback/lineage-based restore at scale — a strong, current real-world analog to the "full-tuple rollback" and "lineage-based restore" material in Transcript 3.
- **Difficulty:** Intermediate. **Reading time:** ~10 minutes.

### MLflow Model Registry: Workflows, Benefits & Challenges — lakeFS blog
- **URL:** https://lakefs.io/blog/mlflow-model-registry/
- **Teaches:** A vendor-neutral (lakeFS is itself a data-versioning company) walkthrough of MLflow registry workflows plus a candid discussion of its limitations — good counterbalance before you commit to "MLflow is always the right choice" in the registry-comparison section.
- **Difficulty:** Intermediate. **Reading time:** ~15 minutes.

### Machine Learning Model Versioning: Top Tools & Best Practices — lakeFS blog
- **URL:** https://lakefs.io/blog/model-versioning/
- **Teaches:** Survey-style comparison of model/data versioning tooling approaches (DVC, Git LFS, lakeFS, registries) — useful as background reading before writing the "how do real companies structure registries" section.
- **Difficulty:** Beginner–Intermediate. **Reading time:** ~15 minutes.

### Canary Deployment for AI Models: A 2026 Guide — MLflow
- **URL:** https://mlflow.org/articles/what-is-canary-deployment-ai/
- **Teaches:** Current (2026) framing of canary deployment specifically for AI/ML models — traffic-slice rollout, comparison metrics between canary and stable populations, and rollback triggers, aligned with the module's canary-release content.
- **Difficulty:** Intermediate. **Reading time:** ~10–15 minutes.

### Top 5 Prompt Versioning Platforms in 2026 — getmaxim.ai
- **URL:** https://www.getmaxim.ai/articles/top-5-prompt-versioning-platforms-in-2026/
- **Teaches:** Current landscape of dedicated prompt-versioning/registry tools (PromptLayer, LangSmith Prompt Hub, and others) as of 2026 — useful context for why prompt versioning is now treated as a first-class registry problem distinct from (but connected to) model versioning.
- **Difficulty:** Beginner. **Reading time:** ~10 minutes.

---

## Notable mid-2026 tooling shifts to flag explicitly in the chapter text

- **MLflow model "Stages" (None → Staging → Production → Archived) have been deprecated since MLflow 2.9** in favor of **aliases** (e.g., `@champion`, `@challenger`) and freeform **tags**. Any pre-2024 tutorial or transcript that talks about "transitioning a model to the Production stage" as the state-of-the-art pattern is describing a deprecated mechanism — the modern equivalent is assigning/reassigning an alias, which also supports multiple simultaneous aliases per model (enabling clean canary/champion-challenger setups that fixed stages could not express).
- **MLflow 3 added a native GenAI Prompt Registry**, closing what used to be a gap that teams filled with third-party tools (PromptLayer, LangSmith Prompt Hub) or ad hoc solutions. This directly upgrades the "deployment triple" story: model registry and prompt registry can now live in the same platform with linked versions, rather than being two disconnected systems that have to be manually kept in sync.
