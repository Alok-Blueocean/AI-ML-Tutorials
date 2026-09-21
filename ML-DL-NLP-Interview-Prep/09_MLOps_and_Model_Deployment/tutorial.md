# MLOps and Model Deployment — Condensed Study Notes

## The ML Workflow Lifecycle

Business framing -> data collection -> EDA -> modeling -> evaluation -> deployment -> monitoring -> (feedback loop back to data/business framing).

Most real-world ML work and most interview time is spent on the steps around modeling, not modeling itself — data quality, evaluation design, and monitoring are where systems actually fail.

- Real-world example: a churn-prediction project spends 2 weeks on the model and 2 months on getting a clean, leakage-free label definition of "churned" (does a 29-day-inactive user count?) — the business framing step is where most of the real risk lives.

## Experiment Tracking and Reproducibility (MLflow, DVC)

- **MLflow** logs parameters, metrics, artifacts, and models per training run, and provides a model registry (staging/production versions with lineage). Answers "which exact code + data + hyperparameters produced the model currently in production?"

```
with mlflow.start_run():
    mlflow.log_param("lr", 0.01)
    mlflow.log_metric("val_auc", 0.87)
    mlflow.sklearn.log_model(model, "model")
# later: mlflow.register_model(run_uri, "fraud-model") -> versioned, promotable
```

- **DVC (Data Version Control)** does for datasets and pipelines what Git does for code: version large data/model files (stored in cloud storage, referenced via lightweight pointer files in Git) and define reproducible pipeline DAGs (`dvc repro` re-runs only the stages affected by a change).

- Real-world example: six months after a model ships, a bug report asks "why did the model predict this?" — MLflow's run history lets you pull up the exact training data version, hyperparameters, and code commit that produced that specific model artifact.

## Feature Stores

- A feature store centralizes feature computation and serving so that training and inference use **exactly the same feature definitions and values**, avoiding train/serve skew. It typically has an offline store (for training, batch) and an online store (low-latency, for real-time inference).

- Real-world example: a fraud model computes "transactions in the last 10 minutes" as a feature. Without a feature store, the batch training pipeline and the real-time serving pipeline might implement this window slightly differently (inclusive vs exclusive boundaries) — a feature store guarantees both paths compute it identically.

## Model Versioning and Registry

- Every deployed model should be traceable to a specific version: training data snapshot, code commit, hyperparameters, evaluation metrics, and approval status (staging/production/archived).

- Real-world example: a regulator asks a bank to explain why a specific loan was denied 8 months ago — without a model registry tying that prediction's timestamp to an exact model version, there is no way to reconstruct or defend the decision.

## CI/CD for ML Pipelines

- Standard CI/CD (lint, unit tests, build) plus ML-specific gates: data validation tests (schema, null rates, distribution sanity checks), model validation tests (does the new model beat the current production model on a fixed eval set?), and automated retraining triggers (continuous training, "CT").

- A useful maturity model (Google Cloud): Level 0 = fully manual; Level 1 = automated training pipeline with continuous training; Level 2 = full CI/CD automation of the pipeline itself.

- Real-world example: a CI pipeline blocks a model-update PR because the new model's precision on the "high-value customer" eval slice dropped 3 points even though overall accuracy improved — a slice-level gate caught a regression that an aggregate metric would have hidden.

## Data Drift vs Concept Drift

- **Data drift (covariate shift):** the distribution of input features changes, but the relationship between features and target is still the same. Example: a sudden shift in customer age distribution due to a new marketing campaign.

- **Concept drift:** the relationship between inputs and target itself changes. Example: "spam" patterns change because spammers adapt; the same email features now mean something different.

- Detection for data drift: statistical tests comparing production feature distributions to training distributions.

```
# e.g., population stability index (PSI) per feature bucket
PSI = sum( (prod_pct - train_pct) * ln(prod_pct / train_pct) )
# PSI < 0.1 stable, 0.1-0.25 moderate shift, > 0.25 significant shift
```

- Detection for concept drift: monitor live label-based performance metrics (when labels arrive) or proxy metrics, since concept drift often can't be seen from input distributions alone.

- Real-world example: during COVID-19, an e-commerce demand-forecasting model saw both — input drift (people bought different product mixes) and concept drift (past purchase patterns stopped predicting future ones the same way) — requiring both retraining and, temporarily, larger human override of the model's outputs.

## A/B Testing vs Shadow vs Canary Deployment

- **A/B test:** split live traffic between old and new model, compare business/ML metrics with statistical rigor. Answers "is the new model actually better for users," not just "does it score higher offline."

- **Shadow deployment:** new model runs in parallel on live traffic but its predictions are logged, not shown to users. Zero user risk; validates the new model behaves sanely on real traffic before it can affect anyone.

- **Canary deployment:** roll the new model out to a small percentage of real traffic (that does see its predictions), monitor closely, then ramp up if healthy. Limits blast radius of a bad model.

- Real-world example: before replacing a recommendation model, a team shadow-deploys it for a week to catch crashes/latency issues with zero user impact, then canaries to 5% of traffic to check real engagement metrics, then A/B tests at 50/50 to get a statistically confident lift number before full rollout.

## Serving Patterns: REST / Batch / Streaming

- **REST (online) serving:** low-latency, request/response, one prediction at a time (or small batches) — e.g., fraud check on a live transaction.

- **Batch serving:** score a large dataset on a schedule and store results — e.g., nightly churn-risk scores for every customer, no latency pressure.

- **Streaming serving:** continuously score events from a stream (Kafka, etc.) as they arrive — e.g., real-time anomaly detection on sensor data.

- Real-world example: a recommendation system precomputes "you might also like" lists in a nightly batch job for most users (cheap), but recomputes them online in real time for users currently browsing, where freshness of the last few clicks matters.

## Containerization and Scaling Inference

- Docker packages the model, dependencies, and serving code into a reproducible unit, deployed via Kubernetes (or managed equivalents) for horizontal scaling, rolling updates, and resource isolation.

- Scaling inference: autoscaling on request volume/GPU utilization, batching concurrent requests to improve GPU throughput, and separating CPU-bound preprocessing from GPU-bound inference.

- Real-world example: a computer-vision model container autoscales from 2 to 40 pods during a retailer's flash sale, then scales back down overnight — without containerization this would require manually provisioning and configuring servers under time pressure.

## Model Optimization for Serving: Quantization, Pruning, Distillation, ONNX

- **Quantization:** reduce numeric precision of weights (FP32 -> INT8/FP16/4-bit) to shrink memory and speed up inference, with a small accuracy cost.

- **Pruning:** remove low-importance weights/neurons/connections, shrinking the model.

- **Distillation:** train a smaller "student" model to mimic a larger "teacher" model's outputs, retaining most of the quality at a fraction of the size.

- **ONNX:** a standard model interchange format that lets you export a model trained in one framework (PyTorch, TensorFlow) and run it in an optimized, framework-agnostic runtime for faster inference.

- Real-world example: a mobile keyboard's next-word-prediction model is distilled and quantized to run on-device in milliseconds, whereas the full-size research model would be far too slow and memory-heavy for a phone.

## Monitoring and Observability for ML Services

- Three layers: (1) system health (latency, error rate, throughput — standard SRE metrics), (2) data/model health (input drift, prediction distribution shift, feature null rates), (3) business/outcome metrics (actual accuracy once labels arrive, downstream KPIs like conversion or fraud loss).

- Real-world example: a model's request latency and error rate look perfectly healthy on a dashboard while its prediction distribution has quietly shifted (e.g., suddenly predicting "not fraud" for 99% of transactions) — only layer-2 monitoring catches this, which is why uptime dashboards alone are not enough for ML services.

## Rollback Strategy

- Always keep the previous production model version deployable within minutes (registry + versioned artifacts make this possible), define automatic rollback triggers (error rate spike, latency spike, drastic prediction-distribution shift), and rehearse rollback the same way you'd rehearse any production incident response.

- Real-world example: a newly deployed pricing model starts recommending near-zero prices due to a feature pipeline bug; an automatic rollback trigger on "prediction distribution outside historical bounds" reverts to the last known-good model within minutes, before meaningful revenue is lost.

## Quick Gotchas Worth Naming in an Interview

- "Accuracy" is rarely the right production monitoring metric on its own — labels are often delayed or missing, so monitoring leans on proxies (drift, prediction distribution) between the times true accuracy can actually be computed.

- A model can pass every CI gate and still fail in production if the CI eval set itself has gone stale relative to current real-world traffic — periodically refresh eval sets, don't treat them as permanent.

- Canary and shadow deployment solve different problems: shadow tells you "does this crash or behave insanely," canary tells you "do real users actually respond better to this" — most serious rollouts use both, in that order.

- Quantization/pruning/distillation are not mutually exclusive — production systems often combine distillation (smaller architecture) with quantization (lower precision) for compounding gains.
