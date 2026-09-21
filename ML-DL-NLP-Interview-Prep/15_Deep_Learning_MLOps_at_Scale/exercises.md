# Deep Learning MLOps at Scale — Exercises

Ordered easy to hard. Mix of hands-on tasks and conceptual/design questions. Hands-on exercises that require an AWS account are marked; if you don't want to incur cost, at minimum write out the exact SDK code and explain what each call does.

1. **Track a DL training run in MLflow.** Train a small PyTorch Lightning model (or a Keras model) on any small dataset, enable `mlflow.pytorch.autolog()` (or `mlflow.tensorflow.autolog()`), and confirm params, metrics, and the model artifact land in the MLflow UI without a single manual `log_metric` call.

2. **Register and promote a model with aliases.** From the run in exercise 1, register the model under a name, then assign a `@champion` alias to it via the MLflow client API. Load the model back purely by `models:/<name>@champion` (not by run ID) to prove alias-based loading works.

3. **Serve an MLflow model locally.** Run `mlflow models serve -m runs:/<run_id>/model -p 5001` and send it a real inference request with `curl`/`requests`. Then run `mlflow deployments run-local` for the same model and compare the two.

4. **Conceptual: stages vs. aliases.** Explain why MLflow moved from Staging/Production/Archived stages to aliases + tags, and describe one promotion scenario (e.g., simultaneous champion/challenger canary) that aliases support and single-slot stages cannot.

5. **Log DL-specific artifacts beyond scalars.** Extend exercise 1 to log a learning-curve plot as an artifact and, if training on GPU, log GPU utilization/memory as a metric over the run. Explain why these matter more for DL debugging than they do for a simple sklearn model.

6. **Deploy a HuggingFace model to a SageMaker real-time endpoint.** (AWS account required.) Using the SageMaker Python SDK, deploy a small pretrained HuggingFace model to a real-time endpoint on a GPU instance, invoke it, and then delete the endpoint. If you don't want to run this live, write out the exact `HuggingFaceModel(...).deploy(...)` code and explain each parameter (instance type, instance count, role).

7. **Write a SageMaker Pipeline with a conditional registration step.** Design (and if possible run) a pipeline with a Processing step, a Training step, an evaluation Processing step, a Condition step comparing the new model's metric against a fixed threshold, and a RegisterModel step that only executes if the condition passes. Sketch the DAG even if you don't execute it.

8. **Configure SageMaker managed spot training.** Take any training script and configure it as a SageMaker Estimator with `use_spot_instances=True`, a `checkpoint_s3_uri`, and `max_wait` greater than `max_run`. Explain what happens if the Spot instance is reclaimed mid-training.

9. **Compare real-time, batch transform, and async inference for the same model.** For one trained model, write the SageMaker deployment code (or pseudocode) for all three serving modes, and state which payload size/latency/cost profile makes each one the right choice.

10. **Design a multi-model endpoint layout.** Given 50 small per-tenant classifiers with highly uneven traffic (a handful of tenants generate most of the traffic), design which models would live behind a shared multi-model endpoint vs. a dedicated endpoint, and justify the split using cold-start and TPS considerations.

11. **Benchmark dynamic batching.** Take a small PyTorch model, serve it two ways — a naive FastAPI endpoint handling one request at a time, and (if feasible) a Triton or `torch.compile`-based batched server — and measure throughput (requests/sec) under concurrent load. Quantify the improvement from batching alone.

12. **Compile a model with ONNX Runtime or TensorRT.** Export a trained model to ONNX and run inference with ONNX Runtime; measure latency against native framework inference. If you have GPU access, additionally try a TensorRT-compiled version and compare all three.

13. **Design a model-promotion approval workflow for 5 ML teams sharing one MLflow registry.** Specify: the naming convention enforced at registration, who can reassign a `@champion` alias, what evidence (metrics, approval record) is required before a reassignment, and how you'd detect and prevent name collisions between teams.

14. **Conceptual: SageMaker vs. self-managed Kubernetes for DL serving.** For a hypothetical large model (e.g., a 13B-parameter LLM) that needs GPU inference at scale, list the concrete factors that would push you toward a SageMaker-managed endpoint vs. a self-hosted Triton deployment on EKS, and state which factor you'd weight most heavily and why.

15. **End-to-end platform critique.** Given a described org where 6 DL teams each run their own always-on GPU SageMaker endpoints, use TorchServe by default without anyone having checked its maintenance status, and have no shared model registry, propose a prioritized set of changes (covering cost, governance, and serving-stack risk) and justify the order given limited engineering time.
