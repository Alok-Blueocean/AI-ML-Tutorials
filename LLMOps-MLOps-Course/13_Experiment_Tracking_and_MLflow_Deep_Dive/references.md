# References — Experiment Tracking and MLflow Deep Dive

Official documentation, engineering blog posts, and comparison write-ups verified via live search/fetch in July 2026. Each entry notes what it teaches and roughly how long it takes to work through.

## MLflow — Official Documentation

| Resource | What it teaches | Difficulty | Reading time |
|---|---|---|---|
| [MLflow Docs — Home](https://mlflow.org/docs/latest/) | Top-level map of the whole platform: the two big halves are now "LLMs & Agents" (tracing, prompt management, GenAI evaluation) and "Machine Learning" (tracking, model packaging, registry, deployment). Confirms MLflow has repositioned itself as an "AI engineering platform," not just an ML experiment tool. | Beginner | 15 min |
| [MLflow Tracking — Concepts](https://mlflow.org/docs/latest/ml/tracking/) | The core mental model: **Runs** (a single execution that records params/metrics/artifacts/timestamps), **Experiments** (logical grouping of runs), and **Models** (artifacts produced by a run). Covers manual logging vs. `autolog()` for scikit-learn/PyTorch/etc., and the three tracking-server deployment modes (local file store, local DB backend, remote tracking server for team use). Also documents MLflow 3's newer `models:/<model_id>` URI scheme and per-run model checkpoints — useful for noting how the API has evolved since 2024-era tutorials. | Beginner–Intermediate | 30–40 min |
| [MLflow Projects](https://mlflow.org/docs/latest/ml/projects/) | The packaging component: `MLproject` file format, entry points, parameters, and the four supported environment managers (virtualenv, conda, Docker, system). Explains convention-based projects (any directory with `.py`/`.sh` files) vs. explicit `MLproject` files, plus remote execution on Databricks/Kubernetes and multi-step pipelines. This is the piece most tutorials skip — read it to actually understand reproducible run packaging, not just tracking. | Intermediate | 30 min |
| [MLflow Models](https://mlflow.org/docs/latest/ml/model/) | The `MLmodel` file format and the "flavors" concept (the interoperability trick that lets any deployment tool load a model regardless of source library). Covers model signatures (input/output schema, inferred automatically from an input example), auto-captured `conda.yaml`/`requirements.txt`, and the 20+ built-in flavors (sklearn, PyTorch, TensorFlow, XGBoost, LLM flavors, generic `pyfunc`). | Intermediate | 25 min |
| [ML Model Registry — Concepts](https://mlflow.org/docs/latest/ml/model-registry/) | The governance layer: registering model versions, adding tags/descriptions, and moving models between lifecycle states. Both a UI and API are documented. | Intermediate | 20 min |
| [Model Registry Tutorial](https://mlflow.org/docs/latest/ml/model-registry/tutorial/) | Hands-on: `log_model(registered_model_name=...)` and `mlflow.register_model()`, then organizing versions with **tags** and **aliases** (e.g. an alias like `champion` you reassign instead of hard-coding a version number in production code) via `client.set_registered_model_alias()` / `get_model_version_by_alias()`. **Important mid-2026 note**: this tutorial no longer teaches the old `Staging`/`Production`/`Archived` stage transitions from 2023-era MLflow docs — aliases + tags are now the recommended pattern. If you learned MLflow registry workflow from an older course, this is the update to internalize. | Intermediate | 30 min |
| [Databricks: Track model development using MLflow](https://docs.databricks.com/aws/en/mlflow/tracking) | How the same open-source Tracking API behaves inside a managed Databricks workspace — notebook/run linkage, dataset lineage, and autologging defaults. Useful if your production stack runs on Databricks rather than self-hosted MLflow. | Intermediate | 20 min |
| [Databricks: Organize training runs with MLflow experiments](https://docs.databricks.com/aws/en/mlflow/experiments) | Workspace-level experiment organization patterns (naming, permissions, notebook-scoped vs. workspace experiments) — directly relevant to the tagging-strategy section of this module. | Beginner–Intermediate | 15 min |
| [Databricks: Log, load, and register MLflow models](https://docs.databricks.com/aws/en/mlflow/models) | End-to-end walk-through of logging, loading, and registering combined in one managed environment — good as a second pass after the pure open-source docs above. | Intermediate | 20 min |

## Weights & Biases — Official Documentation

| Resource | What it teaches | Difficulty | Reading time |
|---|---|---|---|
| [W&B Docs: Parallel coordinates panel](https://docs.wandb.ai/models/app/features/panels/parallel-coordinates) | Directly backs the "parallel-coordinates view" content from this module's source transcripts. Explains the panel's axes (hyperparameters from `wandb.Run.config`, metrics from `wandb.Run.log()`), one line per run, hover tooltips, log-scale axes, and custom gradients — this is the exact visual tool used to compare dozens of prompt/model-version experiments at a glance. | Beginner–Intermediate | 15 min |
| [W&B: Experiment Tracking product page](https://wandb.ai/site/experiment-tracking/) | High-level pitch and feature tour: what W&B captures automatically (code, config, metrics, system resource usage, GPU utilization) and how it complements manual `wandb.log()` calls. Good as a 10-minute orientation before the deeper docs. | Beginner | 10 min |
| [W&B example report: Parallel Coordinates on a sweep](https://wandb.ai/example-team/sweep-demo/reports/Parallel-Coordinates--Vmlldzo5MTQ4Nw) | A live, real W&B Report showing a parallel-coordinates chart driven by an actual hyperparameter sweep — read this alongside the docs above to see the panel applied to real run data rather than just described. | Beginner–Intermediate | 10 min |

## Comet ML — Official Documentation

| Resource | What it teaches | Difficulty | Reading time |
|---|---|---|---|
| [Comet Docs (v2)](https://www.comet.com/docs/v2/) | Comet's full documentation hub: experiment management/tracking, model registry and versioning, production monitoring, artifact management, the Comet Optimizer for hyperparameter search, and SDKs for Python/Java/JS/R. Also documents **Opik**, Comet's open-source LLM evaluation/observability platform — relevant background for later LLMOps-focused modules in this course. Use this as the third leg of the MLflow/W&B/Comet comparison table. | Intermediate | 25–30 min |

## Comparative / Analyst Reading (secondary sources — use for framing, not as ground truth)

| Resource | What it teaches | Difficulty | Reading time |
|---|---|---|---|
| [ZenML Blog: MLflow vs Weights & Biases vs ZenML](https://www.zenml.io/blog/mlflow-vs-weights-and-biases) | A vendor-adjacent but reasonably even-handed comparison of tracking-tool philosophies: MLflow's open-source/self-hosted model vs. W&B's managed collaboration-first platform. Cross-check claims against the official docs above rather than taking pricing/feature claims at face value — comparison blogs from tool vendors have an angle. | Intermediate | 15 min |
| [ZenML Blog: We Tested 9 MLflow Alternatives for MLOps](https://www.zenml.io/blog/mlflow-alternatives) | Broader landscape scan (Neptune, Comet, ClearML, Sacred, etc.) — useful for knowing what else exists beyond the three tools this module compares in depth. | Intermediate | 15 min |
| [neptune.ai Blog: Best MLflow Alternatives](https://neptune.ai/blog/best-mlflow-alternatives) | Same genre from a competing vendor (Neptune) — read both ZenML's and Neptune's takes side by side to triangulate a vendor-neutral view. | Intermediate | 15 min |

## Notes on currency (mid-2026)

- MLflow's PyPI page shows **MLflow 3.14.0**, released June 17, 2026 (`pip install mlflow`). MLflow crossed the "3.x" line in 2024–2025 with a significant repositioning toward GenAI/agent tracing on top of the original ML tracking core — expect the docs and UI to look noticeably different from a 2023-era MLflow 2.x tutorial.
- The Model Registry has moved from `Staging`/`Production`/`Archived` **stage** transitions (the pattern most 2023–2024 tutorials teach) to **tags + aliases** as the recommended promotion mechanism. Teach the alias pattern as current best practice, but mention stages as "what you'll still see in older codebases and blog posts."
- Weights & Biases was **acquired by CoreWeave in March 2025**; it now sits inside CoreWeave's AI-cloud stack alongside GPU infrastructure rather than operating as a fully independent company — worth mentioning when discussing vendor risk/lock-in trade-offs for a production MLOps platform choice.
