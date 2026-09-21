# Deep Learning MLOps at Scale — Condensed Study Notes

Scope note: `09_MLOps_and_Model_Deployment/` already covers drift detection, CI/CD gates, A/B/canary/shadow rollout, and generic serving patterns — that material is not repeated here. This folder is specifically about the tools and operational patterns unique to **deep learning at scale**: SageMaker as a managed DL platform, MLflow's DL-specific tracking/registry/deployment surface, GPU-aware serving choices, and running all of this across many teams without chaos.

## Amazon SageMaker as a Managed DL Platform

SageMaker's pitch for deep learning specifically: GPU training/serving infrastructure, distributed-training libraries, and a model lifecycle (registry, pipelines, monitoring) come pre-wired, so a DL team doesn't hand-roll cluster orchestration, container builds, and endpoint autoscaling from scratch.

### Managed training jobs and spot instances

- A SageMaker **Estimator** (PyTorch, TensorFlow, HuggingFace, or a custom container) launches ephemeral training instances, runs your script, writes model artifacts to S3, and tears the cluster down — you never SSH into a box.
- **Managed Spot Training** runs the same job on spare EC2 capacity for up to ~90% cost savings; the catch is Spot can be reclaimed with a 2-minute warning, so SageMaker requires checkpointing to S3 (`checkpoint_s3_uri`) so an interrupted job resumes rather than restarts from scratch.
- Real-world example: a vision team trains a ResNet variant for 18 hours nightly. Switching to spot instances with checkpointing every 500 steps cuts the monthly training bill by roughly 70%, at the cost of occasionally losing the last few minutes of progress when a Spot interruption lands.

### Built-in framework containers vs. bring-your-own-container (BYOC)

- SageMaker publishes maintained Deep Learning Containers for PyTorch, TensorFlow, and HuggingFace with CUDA/cuDNN/NCCL already tuned — you supply a training script and requirements, not a Dockerfile.
- BYOC is for anything that doesn't fit: a custom CUDA op, an unsupported framework version, or a research codebase with a bespoke dependency stack. It costs you the maintenance of that image (security patches, CUDA compatibility) going forward.
- Real-world example: a team fine-tuning a HuggingFace transformer uses the prebuilt HF container and ships in a day. A team running a custom JAX-based training loop with a patched XLA build has no prebuilt container and must maintain their own image indefinitely — a real, ongoing cost that "just use the managed container" glosses over.

### Distributed training libraries: data parallel vs. model parallel

- **SageMaker distributed data parallelism (SMDDP)**: each GPU holds a full copy of the model and processes a different data shard; gradients are synchronized (all-reduce) across GPUs each step. Use when the model fits on one GPU but training is too slow on one — this is the common case.
- **SageMaker distributed model parallelism (SMP)**: the model itself is partitioned across GPUs (layers or tensor shards), because it doesn't fit in one GPU's memory. Needed for large transformers where even one training example plus model weights exceeds a single GPU's VRAM.
- These are SageMaker's managed alternative to plain PyTorch DDP / Horovod / DeepSpeed — same underlying problem, but with AWS-network-optimized collective communication (AllReduce/AllGather) baked in, at the cost of being AWS-specific.
- Real-world example: a 7B-parameter language model doesn't fit on a single 40GB A100. The team uses SMP to shard the model across 8 GPUs on one node (tensor parallelism) combined with SMDDP data parallelism across multiple nodes — a hybrid approach neither pure data- nor pure model-parallelism alone would support.

### SageMaker Studio, Pipelines, and the Model Registry

- **Studio** is the IDE: notebooks, experiment tracking, pipeline visualization, and endpoint management in one UI — the "where you click around" layer.
- **Pipelines** is orchestration: a DAG of steps (Processing → Training → Evaluation → a Condition step gating on a metric threshold → RegisterModel → CreateModel/Transform), defined via the Python SDK and re-runnable as a versioned, auditable workflow. This is SageMaker's answer to "CI/CD for ML" — a training run becomes a pipeline execution, not a notebook someone ran once.
- **Model Registry** catalogs trained models into Model Groups, tracks versions with lineage back to the training job/data, and holds an approval status (e.g., pending/approved/rejected) that gates deployment — this is what a `RegisterModel` pipeline step writes into.
- Real-world example: a fraud-detection pipeline runs nightly, and a Condition step blocks the RegisterModel step entirely if the new model's AUC on a fixed holdout is below the current production model's — a bad retrain never even reaches the registry, let alone an endpoint.

### Serving options: real-time, batch transform, asynchronous, multi-model

- **Real-time endpoints**: a persistent, autoscaling HTTPS endpoint for millisecond-latency, request/response traffic (payloads up to 6MB, processing under 60s). This is what backs an interactive product feature.
- **Batch transform**: SageMaker spins up instances on demand, scores an entire dataset sitting in S3, writes results back to S3, and tears the compute down — no endpoint stays running. Right fit for large, non-urgent scoring jobs.
- **Asynchronous inference**: sits between the two — you queue a request (payload up to 1GB, processing up to 1 hour), get back a ticket, and poll/get notified for the result. Crucially, it can autoscale the instance count **to zero** when idle, so you don't pay for an always-on GPU for a bursty, slow workload.
- **Multi-model endpoints (MME)**: many models share one fleet and one serving container; SageMaker dynamically loads/unloads models into memory based on traffic, with GPU-backed MME (via an NVIDIA Triton container) supporting large numbers of GPU models on shared instances. Tradeoff: infrequently used models pay a cold-start penalty when they get swapped back in.
- Real-world example: a platform team serves 200 small per-customer fine-tuned classifiers. Running 200 dedicated real-time endpoints would be prohibitively expensive and mostly idle GPU; a GPU-backed multi-model endpoint serves all 200 from a handful of shared instances, accepting occasional cold-start latency for rarely-hit customers.

### Autoscaling and Model Monitor

- Real-time and multi-model endpoints support target-tracking autoscaling on invocation count, latency, or GPU/CPU utilization — the same instance-count-up/down pattern as any web service, but GPU-cost-sensitive.
- **SageMaker Model Monitor** watches data quality (input drift vs. a training baseline), model quality (accuracy drift once ground truth arrives), bias drift, and feature-attribution drift, on a schedule, for real-time or batch-transform-served models. **Important caveat as of this session's docs check: AWS states Model Monitor is no longer open to new customers** (existing users can keep using it; no new features are planned). For a new project, budget for either an alternative approach (custom CloudWatch-based checks, SageMaker Clarify for bias/attribution specifically, or a third-party tool like Evidently — see `09_MLOps_and_Model_Deployment/`) rather than assuming Model Monitor is the default forward-looking answer.
- Real-world example: a team designing a new DL platform in 2026 initially planned around Model Monitor for drift alerts, then had to re-architect that layer around a custom Evidently-based job once they discovered the new-customer restriction during due diligence — a good illustration of why you verify current product status before basing an architecture on a specific managed service.

## Advantages of SageMaker vs. Self-Managed Infra — and the Honest Tradeoffs

**Where it wins:**
- No cluster/orchestration code to write for training or serving — Estimators, Pipelines, and endpoints replace a self-built Kubernetes + Kubeflow + Airflow stack.
- Spot training integration and checkpointing are a few config lines, not custom interruption-handling logic.
- Model Registry + Pipelines gives audit trail and approval gates "for free" instead of building a governance layer on top of raw infra.
- IAM-based access control and CloudTrail logging come with the AWS account boundary already.

**Where the honest tradeoffs are:**
- **Cost at scale**: SageMaker's managed convenience carries a premium over raw EC2/EKS, and it compounds across many always-on real-time endpoints — this is precisely why endpoint sprawl (below) becomes a real line-item problem in a large org.
- **Less flexibility than raw Kubernetes**: a team wanting a custom scheduler, exotic multi-tenant GPU sharing, or a serving stack SageMaker doesn't natively support (say, a bespoke Triton ensemble topology) will fight the platform's opinions.
- **Vendor lock-in**: Pipelines definitions, Model Registry metadata, and SageMaker-specific SDK calls are not portable to GCP Vertex AI or a self-hosted stack without rework — a real switching cost if AWS pricing or roadmap changes.
- **Product churn risk**: the Model Monitor new-customer restriction above is a concrete example — a managed service you architect around today may quietly stop being AWS's forward investment.

The realistic framing for an interview: SageMaker is a strong default for a team that wants to move fast and doesn't need bespoke infra control; a platform team running GPU infrastructure for dozens of ML teams under tight cost pressure will often end up on a hybrid — SageMaker (or Vertex AI) for training/registry, and a self-hosted Kubernetes + Triton/KServe layer for serving where cost or control matters most.

## MLflow for Deep Learning

MLflow is cloud-agnostic — its tracking server, registry, and deployment tooling work the same whether you train on a laptop, on-prem GPUs, or SageMaker, which is why teams often run MLflow tracking independently of whichever cloud does the actual training.

### DL-specific autologging

- `mlflow.pytorch.autolog()` — as of current MLflow docs, autologging integrates with **PyTorch Lightning** specifically (calling `Trainer.fit()` triggers automatic logging of metrics, params, and the model); a raw PyTorch training loop without Lightning needs manual `mlflow.log_metric`/`log_param`/`mlflow.pytorch.log_model` calls instead of relying on autolog.
- `mlflow.tensorflow.autolog()` covers `tf.keras` and `tf.estimator` (TensorFlow >= 2.3) and logs metrics per epoch automatically.
- Beyond scalar metrics, DL runs benefit from explicitly logging checkpoints as artifacts (so you can resume or roll back to an intermediate epoch), learning-curve plots, and system metrics (GPU utilization/memory) — MLflow supports logging system metrics alongside training metrics for exactly this reason.

```python
import mlflow

mlflow.pytorch.autolog()  # PyTorch Lightning path
with mlflow.start_run():
    trainer.fit(model, datamodule)
    mlflow.log_artifact("training_curves.png")
    mlflow.pytorch.log_model(model, "model", registered_model_name="fraud-cnn")
```

- Real-world example: a research engineer debugging a training instability six weeks later pulls up the run in MLflow, sees GPU memory utilization spiked right before the loss diverged, and correlates it to a batch-size change in that run's logged hyperparameters — without those two things logged together, that root cause is very hard to reconstruct after the fact.

### Model Registry: promotion and the shift from stages to aliases

- The registry versions every model registered under a name, and historically MLflow used four fixed **stages** (None → Staging → Production → Archived) to represent promotion state.
- Current MLflow guidance has moved to **aliases and tags** instead of stages: you assign a mutable alias like `@champion` to a specific model version, target `models:/fraud-cnn@champion` from serving code, and re-point the alias to promote a new version — with the advantage (over stages) that multiple aliases can point at different versions simultaneously (e.g., `@champion` in production and `@challenger` in a canary), which a single-slot "Production" stage couldn't represent.
- Real-world example: a team runs a shadow/canary test by tagging the incumbent model `@champion` and the new candidate `@challenger` on the same registered model name — serving code checks both aliases, routes 95% of traffic to `@champion` and 5% to `@challenger`, and promotion is just reassigning which version `@champion` points to once the challenger wins.

### Serving an MLflow-logged model

- `mlflow models serve -m runs:/<run_id>/model` spins up a local REST server (via the `pyfunc` flavor) for quick validation — good enough for testing the exact artifact that will eventually be deployed.
- `mlflow deployments create -t sagemaker -m runs:/<run_id>/model -C region_name=... -C instance-type=ml.g5.xlarge` deploys straight to a SageMaker real-time endpoint: MLflow builds a Docker image, pushes it to ECR, uploads the model to S3, and creates the endpoint — the same MLflow-logged artifact, no manual Dockerfile.
- `mlflow deployments run-local` runs the exact SageMaker-bound container locally first, which is the sane way to catch container-level issues before paying for a real endpoint.
- `mlflow deployments` also targets Kubernetes (via KServe/Seldon-style targets) — the same registered model can be deployed to SageMaker or to a self-hosted cluster from the same registry entry, which is the core of "MLflow + SageMaker together" rather than "MLflow instead of SageMaker."
- Real-world example: a platform team standardizes on MLflow as the single source of truth for "what model is version 14 of `fraud-cnn`," while deployment targets vary by team — one team's inference workload goes to a SageMaker endpoint, another's goes to an in-house Kubernetes cluster with Triton — both pulling from the same registry entry so there's one place to answer "what's actually running."

## DL-Specific Model Serving Considerations

Deep learning inference has cost and latency characteristics classical ML serving doesn't: GPUs are expensive and often underutilized at low request rates, and a naive one-request-at-a-time Flask/FastAPI wrapper leaves most of a GPU's throughput on the table.

- **GPU utilization and dynamic batching**: a GPU processes a batch of 32 images barely slower than a batch of 1 — a serving layer that queues concurrent requests for a few milliseconds and batches them before the forward pass can multiply throughput per GPU-hour. This is the single biggest lever for DL inference cost, and it's exactly the feature a bespoke FastAPI service usually lacks out of the box.
- **NVIDIA Triton Inference Server**: purpose-built for this — dynamic batching, concurrent execution of multiple models/instances on one GPU, and multi-framework support (TensorRT, PyTorch, ONNX, OpenVINO) behind one HTTP/gRPC server. Pick it when you need GPU-efficient serving across mixed frameworks, or when you're building a self-hosted alternative to SageMaker MME.
- **TorchServe**: the historical default for serving PyTorch models (multi-model management, REST/gRPC, used as the default in SageMaker's PyTorch inference containers). **As of this session's check, the `pytorch/serve` GitHub repo has been archived (no longer actively maintained as of August 2025)** — it still works and is still wired into some managed platforms, but it's no longer a safe long-term bet for a new serving stack; new projects should default to Triton, TensorFlow Serving (for TF models), or a managed SageMaker/Vertex endpoint instead.
- **TensorFlow Serving**: the TensorFlow-native equivalent (versioned model lookup, gRPC/REST), actively maintained, and the natural choice when your models are exclusively TensorFlow/Keras and you don't need Triton's multi-framework generality.
- **A bespoke FastAPI/Flask service**: still reasonable for low-traffic or CPU-only DL models, or where custom pre/post-processing logic dominates the request path and a general-purpose server adds more complexity than it saves. It's the wrong choice once GPU throughput or dynamic batching actually matters.
- **Quantization/compilation for latency**: TensorRT compiles a trained model into a GPU-optimized inference engine (layer fusion, FP16/INT8/FP8 precision), and ONNX Runtime provides a framework-agnostic path to run an exported model with hardware-specific execution providers. Both trade a one-time compilation/export step for meaningfully lower per-request latency and cost.
- Real-world example: a team serving a BERT-based classifier on a bespoke FastAPI+PyTorch service at 40ms p50 latency and needing 12 GPU instances to hit their SLA at peak load rewrites the serving layer on Triton with dynamic batching and TensorRT-compiled weights, and meets the same SLA with 4 instances — the win came almost entirely from batching and compilation, not from a bigger GPU.

## Operating This at Scale Across a Large Team/Org

The failure mode at scale isn't usually "the model is wrong" — it's "nobody can tell which model is running where, who approved it, or what it's costing."

- **Model registry governance**: enforce a naming convention (e.g., `<team>-<usecase>-<architecture>`) at the point of registration, not after the fact, and require a defined set of approvers (not "whoever has registry write access") before an alias like `@champion` can move. Without this, five teams sharing one MLflow or SageMaker registry inevitably produce collisions like two different `fraud-model` entries meaning different things.
- **Reproducibility**: pin the *combination* of container image digest, code commit, and config together as one versioned unit — logging just the hyperparameters (and not the exact base image/dependency versions) is a common way "the registered model" and "what was actually trained" quietly diverge.
- **Cost visibility / FinOps for GPU spend**: tag every training job and endpoint by team/project and pull that into a shared cost dashboard; GPU spend is invisible per-team until someone builds this, and it's the single fastest way to find endpoint sprawl and idle spot-eligible training jobs still running on-demand.
- **Access control and audit trails**: tie registry/endpoint actions to IAM identities (or MLflow's auth layer) so "who promoted this model to production" is a query, not an investigation — this matters as much for regulated-industry audits as for incident response.
- **Avoiding endpoint sprawl**: the natural failure mode is five teams each running their own always-on GPU real-time endpoint at low utilization. Push toward shared multi-model endpoints or a shared Triton cluster for low-traffic models, and reserve dedicated endpoints for workloads that actually need isolation or have distinct latency/throughput SLAs.
- **Shared feature/embedding stores**: for DL specifically, this often means a shared embedding store (e.g., precomputed user/item/document embeddings) so multiple teams' models don't each recompute the same expensive embedding pass — the DL analogue of the feature-store train/serve-skew problem covered in `09_MLOps_and_Model_Deployment/`.
- **On-call ownership**: every deployed DL service needs a named owning team with a paging path — "the platform team owns it" tends to mean nobody actually responds at 2am when a specific model's endpoint degrades.
- **Rollback strategy**: SageMaker deployment guardrails (blue/green with canary or linear traffic shifting, plus CloudWatch-alarm-triggered auto-rollback) give infrastructure-level rollback for real-time/async endpoints; MLflow alias reassignment gives registry-level rollback (`@champion` points back at the previous version) that works regardless of serving target. A mature org uses both: alias/version rollback to know instantly which model *should* be serving, and endpoint-level traffic rollback to actually stop bad traffic from reaching users.
- Real-world example: an org running 5 ML teams on one shared MLflow registry mandates that every registered model name is prefixed by team, every `@champion` reassignment requires a second approver's sign-off recorded as a tag, and every training job logs its exact container image digest as a run parameter — none of these are exotic, but together they're the difference between "we can reconstruct and roll back any incident in minutes" and "we're not sure which model has been serving production for the last two weeks."

## Quick Gotchas Worth Naming in an Interview

- SageMaker Model Monitor is no longer open to new customers (as of this session's docs check) — don't casually cite it as "the" SageMaker drift-monitoring answer for a brand-new architecture without noting that.
- TorchServe is archived/unmaintained as of August 2025 — it's still functionally present (including as SageMaker's default PyTorch serving container), but naming it as your forward-looking serving choice without flagging that is a gap an interviewer may probe.
- MLflow has moved from stages (Staging/Production/Archived) to aliases + tags as the recommended promotion mechanism — stages still exist for backward compatibility, but aliases are the current guidance and support patterns (multiple simultaneous pointers) stages can't.
- Real-time endpoints, batch transform, and async inference are not interchangeable defaults — the decision hinges on payload size, processing time, and whether you can tolerate scale-to-zero cold starts, not just "online vs. offline."
- Dynamic batching and compilation (TensorRT/ONNX Runtime) typically cut DL inference cost far more than picking a bigger/cheaper instance type — instance selection is a secondary lever compared to using the GPU efficiently in the first place.
- "SageMaker vs. Kubernetes" is rarely all-or-nothing at scale — many large orgs run SageMaker for training/registry governance and a self-hosted Triton/KServe layer for cost-sensitive serving, picking per workload rather than standardizing on one everywhere.
