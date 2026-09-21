# GitHub Repositories — Module 03: Versioning, Registries, and Rollback

## mlflow/mlflow
- **URL:** https://github.com/mlflow/mlflow
- **Purpose:** The official MLflow monorepo — Tracking, Model Registry, Projects, Model Serving, Evaluation, and (as of MLflow 3) the GenAI Prompt Registry and tracing stack, all in one open-source platform.
- **Popularity tier:** Very popular / one of the most widely adopted open-source MLOps projects (tens of thousands of GitHub stars; de facto standard registry in most non-cloud-locked MLOps stacks).
- **Why it matters:** This is the reference implementation for essentially every code example in this module — `mlflow.register_model`, `MlflowClient`, aliases, tags, and the new `mlflow.genai` prompt registry APIs all live here. Reading the `mlflow/store/model_registry` source is the fastest way to understand exactly what a "promotion" mutates under the hood (a row in a backing SQL/file store, not magic).
- **How it relates to this module:** Directly backs the MLflow registry walkthrough (registering → staging/aliasing → promoting → archiving a model) and the "MLflow vs W&B vs Comet" comparison table.

## DataTalksClub/mlops-zoomcamp
- **URL:** https://github.com/DataTalksClub/mlops-zoomcamp
- **Purpose:** Full open-source curriculum (notebooks, slides, homework, recorded lectures) for a free 9-week MLOps course; Module 2 (`02-experiment-tracking`) is a working, runnable MLflow registry example.
- **Popularity tier:** Very popular in the MLOps education space; widely cited/forked as a self-study curriculum.
- **Why it matters:** Gives you real, runnable notebooks (`model-registry.ipynb`) rather than slideware — good source of copy-adapt code for this module's MLflow walkthrough section, and its homework structure models the promotion-criteria mindset (does the new model beat the registered one on held-out metrics before promotion).
- **How it relates to this module:** Backs the registry walkthrough and the general "registry as waiting room" mental model used across all three source transcripts.

## iterative/dvc (DVC — Data Version Control)
- **URL:** https://github.com/iterative/dvc
- **Purpose:** Git-native version control for datasets, models, and ML pipelines — content-addressed storage with lightweight `.dvc` pointer files checked into Git, remote storage backends (S3/GCS/Azure/local), and pipeline DAG tracking (`dvc.yaml`/`dvc repro`).
- **Popularity tier:** Very popular / widely adopted; the standard open-source answer to "how do I version large data/model files without bloating Git."
- **Why it matters:** DVC is the most common way teams implement the "dataset" leg of the deployment triple (model + prompt + dataset) in practice — it lets you `git tag` a commit that pins an exact dataset version alongside the code and model version, which is precisely the reproducibility guarantee this module's versioning theory section asks for.
- **How it relates to this module:** Directly supports the semantic-versioning-for-datasets material in Transcript 1 and the lineage-tracking material in Transcript 3 (you can reconstruct "what dataset produced this exact model" from DVC + Git history).

## wandb/wandb
- **URL:** https://github.com/wandb/wandb
- **Purpose:** Client SDK and CLI for Weights & Biases — experiment tracking, Artifacts (versioned, content-addressed datasets/models with an automatic lineage DAG), and the W&B Model Registry / Registry (their newer unified artifact registry).
- **Popularity tier:** Very popular / widely adopted, especially in research-heavy and applied-research teams.
- **Why it matters:** W&B Artifacts automatically build a lineage graph between runs → datasets → models, which is the cleanest concrete example of "lineage tracking" from Transcript 3 — you can literally click backward from a production model artifact to the exact run, code version, and input datasets that produced it.
- **How it relates to this module:** Primary source for the MLflow vs. W&B comparison table, and a strong illustrative example for the lineage-based-restore section of the rollback runbook.

## comet-ml/comet-examples
- **URL:** https://github.com/comet-ml/comet-examples
- **Purpose:** Official collection of runnable example projects for Comet ML, including model registry registration/promotion workflows across frameworks (PyTorch, scikit-learn, Hugging Face, etc.).
- **Popularity tier:** Moderate adoption; smaller community than MLflow/W&B but a real, actively maintained production SaaS+self-host product used by enterprise ML teams.
- **Why it matters:** Comet's registry model (workspace-scoped registered models, explicit semantic-version strings, status/stage tags) is different enough from both MLflow and W&B to make a genuinely useful three-way comparison rather than two tools plus an afterthought.
- **How it relates to this module:** Backs the third column of the MLflow vs. W&B vs. Comet comparison table.

## GoogleCloudPlatform/vertex-ai-samples
- **URL:** https://github.com/GoogleCloudPlatform/vertex-ai-samples
- **Purpose:** Official Google Cloud sample repository, including notebooks for Vertex AI Model Registry — versioning, aliasing, and traffic-split rollout/rollback between model versions behind a single endpoint.
- **Popularity tier:** Popular / official (maintained by Google Cloud).
- **Why it matters:** Gives a managed-cloud counterpoint to the self-hosted MLflow/DVC story — shows how a cloud-native registry implements traffic splitting between model versions at the endpoint level, which is the mechanism that makes canary release and instant rollback possible without redeploying infrastructure.
- **How it relates to this module:** Directly supports the canary-release and rollback-script material in Transcript 3.
