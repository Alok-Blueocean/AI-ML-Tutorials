# GitHub Repositories — Module 06: Kubernetes for ML and LLM Systems

All repositories below were verified to exist via search. Popularity is described in tiers rather than fabricated exact star counts (check the repo pages directly for current live numbers).

---

## Core Kubernetes / GPU Scheduling

### kubernetes/kubernetes
- **URL:** https://github.com/kubernetes/kubernetes
- **Purpose:** The Kubernetes project itself — API server, scheduler, controller-manager, kubelet source.
- **Popularity tier:** One of the most popular repositories on all of GitHub; foundational infrastructure project.
- **Why it matters:** Ground truth for every manifest field this chapter uses. When documentation is ambiguous, the `pkg/apis` type definitions and `staging/src/k8s.io/api` directory are the actual source of truth for what a Deployment/Service/HPA spec accepts.
- **Relation to module:** Reference repo, not something you deploy directly — useful for verifying exact API fields (e.g., HPA v2 `behavior` stanza, container `resources.limits`).

### NVIDIA/k8s-device-plugin
- **URL:** https://github.com/NVIDIA/k8s-device-plugin
- **Purpose:** NVIDIA's official Kubernetes device plugin — the DaemonSet that discovers GPUs on each node and registers them with the kubelet as the `nvidia.com/gpu` extended resource, which is what lets pods request GPUs via `resources.limits."nvidia.com/gpu"`.
- **Popularity tier:** Very popular / the standard, officially maintained GPU-scheduling component for Kubernetes clusters running NVIDIA hardware.
- **Why it matters:** This is the literal implementation behind the "GPU scheduling (nvidia device plugin)" section of the chapter. Its README shows the DaemonSet manifest and the exact resource-request syntax pods must use.
- **Relation to module:** Directly implements the GPU scheduling mechanism this chapter teaches; the chapter's example manifests mirror this repo's documented usage pattern.

### NVIDIA/gpu-operator
- **URL:** https://github.com/nvidia/gpu-operator
- **Purpose:** A Kubernetes Operator that automates installing and managing the full NVIDIA software stack on GPU nodes — drivers, container toolkit, device plugin, DCGM exporter for metrics, and node feature discovery — as a single Helm-installable unit.
- **Popularity tier:** Very popular in the cloud-native AI infrastructure space; the standard production path recommended by NVIDIA over hand-installing the bare device plugin.
- **Why it matters:** Represents the production-grade evolution of "just install the device plugin" — most real clusters use the GPU Operator, not the raw device plugin DaemonSet, because it also handles driver lifecycle and time-slicing/MIG configuration.
- **Relation to module:** Extends the GPU scheduling section with the operator-pattern approach; referenced as the recommended production alternative.

### kubernetes/autoscaler
- **URL:** https://github.com/kubernetes/autoscaler
- **Purpose:** Home of both the Cluster Autoscaler (adds/removes nodes) and the Vertical Pod Autoscaler (VPA — adjusts pod resource requests/limits based on observed usage).
- **Popularity tier:** Very popular / official Kubernetes SIG-managed project.
- **Why it matters:** VPA is the official implementation this chapter's "resource requests/limits and ... autoscaling (HPA/VPA/KEDA)" section refers to. Its README explains the three-component architecture (recommender, updater, admission-controller) that engineers are expected to know at senior level.
- **Relation to module:** Direct source for the VPA portion of the autoscaling section.

---

## Autoscaling: KEDA

### kedacore/keda
- **URL:** https://github.com/kedacore/keda
- **Purpose:** Kubernetes Event-Driven Autoscaling — extends the Kubernetes HPA to scale Deployments (and scale to/from zero) based on external event sources: queue depth (RabbitMQ, SQS, Kafka lag), Prometheus metrics, cron schedules, and 60+ other built-in scalers.
- **Popularity tier:** Very popular / widely adopted; a CNCF graduated project.
- **Why it matters:** This is the exact tool the module scope calls out for "KEDA for queue-depth-based scaling." The chapter's KEDA `ScaledObject` manifest examples are directly modeled on this repo's documented CRD schema.
- **Relation to module:** Core reference implementation for the KEDA portion of the autoscaling section — read the `ScaledObject` and scaler-specific docs (e.g., the RabbitMQ or Kafka scaler) before writing production ScaledObjects.

---

## Model Serving: KServe and Kubeflow

### kserve/kserve
- **URL:** https://github.com/kserve/kserve
- **Purpose:** "Standardized Distributed Generative and Predictive AI Inference Platform for Scalable, Multi-Framework Deployment on Kubernetes." Provides the `InferenceService` CRD and controller that turns a single YAML manifest into a Deployment + Service + (optionally) HPA/Knative autoscaling, supporting TensorFlow Serving, Triton, Hugging Face, XGBoost, LightGBM, and custom runtimes.
- **Popularity tier:** Very popular / widely adopted; a CNCF incubating project as of late 2025, with adoption at organizations including Bloomberg and IBM.
- **Why it matters:** This is the primary hands-on tool for the "Kubeflow and KServe for model serving" section. The `InferenceService` manifest example in this chapter is directly derived from this repo's own getting-started documentation.
- **Relation to module:** Central repo for the KServe section — the "Deploy Your First Predictive Model with InferenceService" guide in its docs site is the canonical first tutorial to pair with this chapter.

### kubeflow/kubeflow
- **URL:** https://github.com/kubeflow/kubeflow
- **Purpose:** "Machine Learning Toolkit for Kubernetes" — the umbrella project tying together Notebooks, Pipelines, Katib (hyperparameter tuning/AutoML), Kubeflow Trainer (distributed training), and KServe into one platform for the ML lifecycle on Kubernetes.
- **Popularity tier:** Very popular / widely adopted; long-standing CNCF incubating project, now repositioned (1.11 release, Dec 2025) as the "Kubeflow AI Reference Platform" with a sharpened focus on generative AI and LLM fine-tuning at scale.
- **Why it matters:** Gives the platform-level context for where KServe sits — most production deployments don't run KServe standalone, they run it as the serving layer of a broader Kubeflow (or Kubeflow-like) platform alongside Pipelines and Katib.
- **Relation to module:** Direct reference for the Kubeflow section; useful for readers who want to see the full pipeline-to-serving lifecycle, not just the serving piece.

### kubeflow/examples
- **URL:** https://github.com/kubeflow/examples
- **Purpose:** Extended, runnable tutorials for Kubeflow components, including a KServe example that deploys a scikit-learn model trained on the iris dataset via a first `InferenceService`.
- **Popularity tier:** Moderately popular; the standard "extended examples" companion repo for Kubeflow.
- **Why it matters:** Gives a complete, runnable end-to-end example (training artifact → InferenceService → prediction request) rather than an isolated YAML snippet, which is valuable for readers doing the hands-on exercises in this chapter.
- **Relation to module:** Practical companion repo for the KServe hands-on exercises.

---

## Progressive Delivery: Canary / Blue-Green

### argoproj/argo-rollouts
- **URL:** https://github.com/argoproj/argo-rollouts
- **Purpose:** A Kubernetes controller and CRD set (`Rollout`) providing canary, blue-green, canary analysis (metric-driven promotion via Prometheus/Datadog/CloudWatch), and experimentation on top of ordinary Deployments — integrates with Istio, Linkerd, NGINX Ingress, and AWS ALB for traffic shifting.
- **Popularity tier:** Very popular / the de facto standard progressive-delivery controller in the Argo/CNCF ecosystem.
- **Why it matters:** This is the concrete tool behind the "canary/blue-green deployments on K8s" section — the `Rollout` CRD mirrors the `Deployment` spec but adds a `strategy` block, which is exactly the diff the chapter walks through.
- **Relation to module:** Primary reference implementation for the rollout-strategies section.

---

## Curated Lists

### ramitsurana/awesome-kubernetes
- **URL:** https://github.com/ramitsurana/awesome-kubernetes
- **Purpose:** A curated ("awesome-list" format) index of Kubernetes tools, tutorials, and resources spanning networking, storage, security, and operational tooling.
- **Popularity tier:** Very popular within the awesome-list category.
- **Why it matters:** Useful as a jumping-off point when a reader needs a tool this chapter doesn't cover in depth (e.g., a specific Ingress controller or service mesh) — a broader map of the ecosystem surrounding the topics in this module.
- **Relation to module:** General supplementary reference, not core reading.

---

## Notes on use

- Star counts change constantly; do not cite a specific number in course material — link the repo and let the reader observe the current count, or use the tier descriptors above.
- When exercises reference "the KServe quickstart" or "the KEDA RabbitMQ scaler example," point learners to the specific `docs/` or `samples/` subdirectory within these repos rather than the repo root, since these are large multi-purpose monorepos.
