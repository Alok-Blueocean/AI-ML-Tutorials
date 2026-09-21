# MLOps and Model Deployment — Projects

## Small: Trackable, Reproducible Training Pipeline

Take an existing model-training script (any tabular or text classification task) and wrap it with MLflow experiment tracking (params, metrics, artifacts) and DVC for data versioning, so that any past run's exact data + code + hyperparameters + resulting model can be reconstructed. Use a public dataset (e.g., a Kaggle tabular dataset) for simplicity. This proves you understand reproducibility as a first-class requirement, not an afterthought — the single most common gap between "notebook that worked once" and "system someone can trust."

## Medium: Containerized Model with CI/CD Gate and Drift Monitoring

Build a REST-served, Dockerized model (e.g., a churn or fraud classifier on a public dataset) with a CI pipeline that blocks deployment if the new model underperforms the current production model on a fixed holdout set, plus a lightweight Evidently-based monitoring job that runs against simulated "incoming" production data and flags drift. Deliberately inject a distribution shift into the simulated production data partway through and show the monitoring catching it. This proves you can operationalize the full loop: train, gate, deploy, and detect when things go wrong — not just deploy once and walk away.

## Large: End-to-End MLOps Platform with Progressive Rollout

Build a more complete system: a feature pipeline (batch + a simple online feature cache), a model registry with staged promotion (staging -> production), a serving layer supporting both shadow deployment and canary rollout (e.g., route 5% of simulated traffic to a new model version and compare metrics before promoting), and a monitoring dashboard tracking system health, prediction drift, and business-metric proxies. Use a realistic use case like fraud detection or recommendation on a public dataset (e.g., a Kaggle fraud dataset with a simulated transaction stream). This proves you can reason about and build the operational infrastructure around a model — the part of MLOps interviews that separates "can train a model" candidates from "can run a model in production" candidates.
