# Recommended GitHub Repositories — Experiment Tracking and MLflow Deep Dive

Star counts below were fetched live in July 2026 and are approximate as of that date — treat them as "order of magnitude / popularity tier" indicators, not exact figures to quote verbatim, since they change continuously.

## mlflow/mlflow
- **URL**: https://github.com/mlflow/mlflow
- **Purpose**: The MLflow core project itself — Tracking, Projects, Models, Model Registry, and (in the 3.x line) LLM/agent tracing and evaluation, all in one repository.
- **Popularity tier**: Very popular / widely adopted — ~27k stars, Apache-2.0 licensed, one of the de facto standard open-source MLOps tools.
- **Why it matters**: This is the ground truth for everything in this chapter — the actual source of `mlflow.log_metric()`, the Tracking Server, and the Registry. Worth cloning to read the `mlflow/tracking/` and `mlflow/store/model_registry/` source when you want to know exactly what a call does under the hood rather than trusting docs prose. The `examples/` directory in this repo also contains runnable end-to-end scripts for common frameworks.
- **Relation to this module**: Primary backbone — every code sample in this chapter (`mlflow.start_run()`, `log_param`, `log_artifact`, `register_model`) traces back to this codebase.

## mlflow/mlflow-example
- **URL**: https://github.com/mlflow/mlflow-example
- **Purpose**: A minimal, official reference implementation of an **MLflow Project** — shows the `MLproject` file format, a `conda.yaml`/environment spec, and an entry point in the smallest possible working example.
- **Popularity tier**: Smaller reference repo (official, maintained under the `mlflow` GitHub org, not a community fork)
- **Why it matters**: Most learners never actually package a project with an `MLproject` file — they only ever call the Tracking API from a notebook. This repo is the fastest way to see the Projects component (the "P" in MLflow's four components) as real files rather than documentation prose.
- **Relation to this module**: Directly supports the "MLflow Projects explained end to end" part of this chapter's scope.

## wandb/wandb
- **URL**: https://github.com/wandb/wandb
- **Purpose**: The official Weights & Biases Python client library — `wandb.init()`, `wandb.log()`, Sweeps (hyperparameter search), Artifacts, and the reporting API.
- **Popularity tier**: Very popular / widely adopted — ~11k stars, MIT licensed.
- **Why it matters**: Read this alongside `mlflow/mlflow` to compare API design philosophy directly — W&B's client is intentionally more "batteries-included" (automatic system-metrics capture, built-in visualization panels) versus MLflow's more minimal, self-hosted-first design. This contrast is the core of the MLflow-vs-W&B comparison table in this chapter.
- **Relation to this module**: Backs the "W&B parallel-coordinates views" content from the source transcripts — the panel code and config live in this repo's `wandb/apis/` and dashboard-facing code.

## wandb/examples
- **URL**: https://github.com/wandb/examples
- **Purpose**: A curated collection of example projects (PyTorch, TensorFlow/Keras, Hugging Face Transformers, PyTorch Lightning, XGBoost, scikit-learn) showing W&B integrated into real training loops.
- **Popularity tier**: Moderate — ~1.2k stars; an official companion repo rather than the core library.
- **Why it matters**: Gives you copy-pasteable, framework-specific starting points for the "compare experiments across model/prompt versions" exercises in this module, instead of writing W&B integration code from scratch.
- **Relation to this module**: Practical companion to the parallel-coordinates and run-comparison sections.

## comet-ml/opik
- **URL**: https://github.com/comet-ml/opik
- **Purpose**: Comet's open-source platform for debugging, evaluating, and monitoring LLM applications, RAG pipelines, and agentic workflows — tracing, automated evaluation, and production dashboards.
- **Popularity tier**: Very popular / widely adopted — 20k+ stars; notably, this is Comet's fastest-growing and most prominent open-source project in 2025–2026, reflecting the broader industry shift from pure "ML experiment tracking" toward "LLM/agent observability."
- **Why it matters**: When comparing Comet to MLflow and W&B in this chapter, note that Comet's center of gravity has shifted toward LLM observability (via Opik) even as its core experiment-tracking product continues. This is a useful "where is the industry going" data point for a mid-2026 course.
- **Relation to this module**: Supports the MLflow vs. W&B vs. Comet comparison table, specifically the "production LLMOps use" column.

## comet-ml/comet-examples
- **URL**: https://github.com/comet-ml/comet-examples
- **Purpose**: Official tutorials, notebooks, and sample pipelines demonstrating Comet's experiment-tracking SDK across frameworks.
- **Popularity tier**: Moderate — official companion/reference repo.
- **Why it matters**: The fastest way to see Comet's logging API side-by-side with MLflow's and W&B's for the same kind of training loop, useful when writing or reviewing the comparison table's code-style differences.
- **Relation to this module**: Reference material for the Comet column of the tool-comparison table.

## GokuMohandas/Made-With-ML
- **URL**: https://github.com/GokuMohandas/Made-With-ML
- **Purpose**: A full, production-mindset ML course-as-repository covering the whole lifecycle — design, develop, deploy, iterate — with an explicit, hands-on experiment-tracking module built on MLflow (setting up a tracking server, logging runs, viewing the Tracking UI, loading saved checkpoints back out for inference).
- **Popularity tier**: Extremely popular / widely adopted — ~49k stars, one of the most-cited "how to actually do MLOps" open-source teaching resources.
- **Why it matters**: Unlike the official MLflow docs, this repo shows MLflow used inside a complete, opinionated production-style project (including a Ray-based training loop with an `MLflowLoggerCallback`), which is a good "does this fit together in a real project" sanity check after working through the isolated code samples in this chapter.
- **Relation to this module**: Cross-reference for the "complete Python code for logging experiments" section — compare its patterns against this chapter's own code to see two independently reasonable ways to structure the same MLflow workflow.
