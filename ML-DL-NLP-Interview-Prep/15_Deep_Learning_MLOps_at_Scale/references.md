# Deep Learning MLOps at Scale — References

All links below were fetched and content-verified against the claim this session unless explicitly marked otherwise. Product status notes (e.g., "no longer open to new customers," "archived") reflect what the fetched page/repo stated at the time of this check — reverify before relying on them, since these statuses can change.

## Amazon SageMaker — Official Docs

- Distributed training overview (data parallel vs. model parallel concepts, cluster/GPU terminology). https://docs.aws.amazon.com/sagemaker/latest/dg/distributed-training.html
- Introduction to the SageMaker AI distributed data parallelism (SMDDP) library. https://docs.aws.amazon.com/sagemaker/latest/dg/data-parallel-intro.html
- SageMaker Pipelines overview (DAG structure, step types, RegisterModel/CreateModel/Transform steps). https://docs.aws.amazon.com/sagemaker/latest/dg/pipelines-overview.html
- Model Registry (catalog models, model groups/versions, approval status, CI/CD deployment). https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html
- Data and model quality monitoring with SageMaker Model Monitor — **note: this page states Model Monitor is no longer open to new customers as of this check; existing customers can continue using it, no new features planned.** https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor.html
- Multi-model endpoints (shared serving container, GPU support via NVIDIA Triton container, dynamic load/unload). https://docs.aws.amazon.com/sagemaker/latest/dg/multi-model-endpoints.html
- Deployment guardrails (blue/green with all-at-once/canary/linear traffic shifting, rolling deployments, CloudWatch-alarm auto-rollback). https://docs.aws.amazon.com/sagemaker/latest/dg/deployment-guardrails.html
- Asynchronous inference (queued requests, payloads up to 1GB, processing up to 1 hour, scale-to-zero). https://docs.aws.amazon.com/sagemaker/latest/dg/async-inference.html
- Batch transform for inference. https://docs.aws.amazon.com/sagemaker/latest/dg/batch-transform.html
- Managed Spot Training (up to ~90% training cost savings, checkpointing for fault tolerance). https://docs.aws.amazon.com/sagemaker/latest/dg/model-managed-spot-training.html
- Docker containers for training and deploying models (built-in framework containers vs. bring-your-own-container). https://docs.aws.amazon.com/sagemaker/latest/dg/docker-containers.html

## MLflow — Official Docs

- MLflow Model Registry (current guidance: aliases and tags, e.g. `@champion`, in place of the legacy Staging/Production/Archived stages). https://mlflow.org/docs/latest/ml/model-registry/
- MLflow PyTorch integration (autologging via `mlflow.pytorch.autolog()`, currently tied to PyTorch Lightning; `log_model` with signatures). https://mlflow.org/docs/latest/ml/deep-learning/pytorch/
- Automatic logging with MLflow Tracking (`autolog()` framework coverage, including `mlflow.tensorflow.autolog()` for `tf.keras`/`tf.estimator`, TensorFlow >= 2.3). https://mlflow.org/docs/latest/ml/tracking/autolog/
- Deploy an MLflow Model to Amazon SageMaker (`mlflow deployments create -t sagemaker`, `mlflow sagemaker build-and-push-container`, `mlflow deployments run-local` for pre-deployment testing). https://mlflow.org/docs/latest/ml/deployment/deploy-model-to-sagemaker/

## Serving Frameworks — Official Repos/Docs

- NVIDIA Triton Inference Server (dynamic batching, concurrent multi-model execution, multi-framework: TensorRT/PyTorch/ONNX/OpenVINO). https://github.com/triton-inference-server/server
- TorchServe — **note: repository is archived as of August 7, 2025; README states the project is no longer actively maintained** (still used as SageMaker's default PyTorch inference container as of this check). https://github.com/pytorch/serve
- TensorFlow Serving (versioned model serving for TensorFlow, actively maintained). https://github.com/tensorflow/serving
- NVIDIA TensorRT documentation (model compilation/quantization for GPU inference latency). https://docs.nvidia.com/deeplearning/tensorrt
- ONNX Runtime (cross-platform, hardware-accelerated inference runtime). https://onnxruntime.ai/

## AWS Governance / Scaling Case Studies

- AWS Machine Learning Blog, "Governing the ML lifecycle at scale, Part 1: A framework for architecting ML workloads using Amazon SageMaker" — multi-account team isolation, centralized model registry/approval workflows, tag-based cost visibility, tag-based access control for large orgs. https://aws.amazon.com/blogs/machine-learning/governing-the-ml-lifecycle-at-scale-part-1-a-framework-for-architecting-ml-workloads-using-amazon-sagemaker/

## GitHub Repositories

- `aws/amazon-sagemaker-examples` — official AWS-maintained example notebooks covering training, model customization, evaluation, inference (real-time/serverless/batch), and MLOps (pipelines, lineage, governance). https://github.com/aws/amazon-sagemaker-examples
- `mlflow/mlflow` — official MLflow repository (tracking, model registry, deployment, and the broader LLM/agent observability surface added in recent versions). https://github.com/mlflow/mlflow

## Notes on What to Prioritize

Interview questions in this area consistently center on: choosing the right SageMaker inference mode for a given latency/payload/cost profile, explaining SageMaker's distributed training libraries (data vs. model parallel) and when each is needed, MLflow's registry promotion mechanism (know that aliases have superseded stages as current guidance), and — the differentiator for senior candidates — governance and cost patterns for running DL infra across many teams (naming conventions, endpoint sprawl, FinOps visibility, rollback ownership). Be ready to name a concrete tool for each layer (SageMaker Pipelines, MLflow registry, Triton, TensorRT/ONNX Runtime) and explain the actual problem it solves, not just recite the name — and be ready to name the honest tradeoff of the managed option (cost, lock-in, product churn like the Model Monitor status above) rather than only listing advantages.
