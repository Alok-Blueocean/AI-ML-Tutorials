# References — Module 06: Kubernetes for ML and LLM Systems

Official documentation, engineering blogs, and papers. Organized by chapter section.

---

## 1. Kubernetes Fundamentals (Pods, Deployments, Services, Ingress)

- **Kubernetes Documentation — Workloads: Pods**
  https://kubernetes.io/docs/concepts/workloads/pods/
  What it teaches: The Pod as the atomic scheduling unit; multi-container pods, init containers, pod lifecycle.
  Difficulty: Beginner. Reading time: ~20 min.

- **Kubernetes Documentation — Deployments**
  https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
  What it teaches: Declarative updates, ReplicaSets, rolling update strategy fields (`maxSurge`, `maxUnavailable`), rollback via `kubectl rollout undo`.
  Difficulty: Beginner. Reading time: ~25 min.

- **Kubernetes Documentation — Service**
  https://kubernetes.io/docs/concepts/services-networking/service/
  What it teaches: ClusterIP/NodePort/LoadBalancer service types, endpoint selection via label selectors, headless services (relevant to StatefulSets).
  Difficulty: Beginner. Reading time: ~25 min.

- **Kubernetes Documentation — Ingress**
  https://kubernetes.io/docs/concepts/services-networking/ingress/
  What it teaches: HTTP/HTTPS routing into cluster services, path/host-based rules, relationship to Ingress controllers (NGINX, ALB, etc.).
  Difficulty: Beginner-Intermediate. Reading time: ~20 min.

---

## 2. Resource Requests/Limits and GPU Scheduling

- **Kubernetes Documentation — Managing Resources for Containers**
  https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
  What it teaches: The semantics of `requests` vs `limits` for CPU/memory, QoS classes (Guaranteed/Burstable/BestEffort), and OOMKill behavior.
  Difficulty: Intermediate. Reading time: ~20 min.

- **Kubernetes Documentation — Schedule GPUs**
  https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/
  What it teaches: The extended-resource mechanism (`nvidia.com/gpu`), device plugin DaemonSet deployment model, and the rule that GPU resources can only be specified in `limits` (not `requests`) since GPUs are not shareable/overcommittable like CPU.
  Difficulty: Intermediate. Reading time: ~15 min.

- **Kubernetes Documentation — Device Plugins**
  https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/
  What it teaches: The general device-plugin framework kubelet exposes, which NVIDIA's plugin (and Intel's, AMD's) implement.
  Difficulty: Advanced. Reading time: ~20 min.

- **NVIDIA GPU Operator Documentation — Time-Slicing GPUs in Kubernetes**
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html
  What it teaches: How to oversubscribe GPUs across multiple pods via time-slicing configuration on top of the base device plugin — critical for cost efficiency when GPU-bound inference workloads don't saturate a full GPU.
  Difficulty: Advanced. Reading time: ~20 min.

- **NVIDIA GPU Operator Documentation — About the NVIDIA GPU Operator**
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html
  What it teaches: The operator pattern for managing the full GPU software stack (driver, toolkit, device plugin, DCGM monitoring, node feature discovery) as one Helm-installable unit.
  Difficulty: Intermediate. Reading time: ~15 min.

---

## 3. Autoscaling: HPA / VPA / KEDA

- **Kubernetes Documentation — Horizontal Pod Autoscaling**
  https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/
  What it teaches: HPA v2 API, scaling on CPU/memory and custom/external metrics, the `behavior` stanza for controlling scale-up/scale-down rate.
  Difficulty: Intermediate. Reading time: ~25 min.

- **Kubernetes Documentation — Vertical Pod Autoscaling**
  https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/
  What it teaches: VPA's recommender/updater/admission-controller architecture; why VPA and HPA should generally not both control the same resource metric simultaneously.
  Difficulty: Advanced. Reading time: ~15 min.

- **KEDA Official Documentation**
  https://keda.sh/docs/
  What it teaches: The `ScaledObject` and `ScaledJob` CRDs, the full catalog of 60+ scalers (RabbitMQ, Kafka, AWS SQS, Prometheus, cron), and scale-to-zero semantics — the core reference for queue-depth-based autoscaling of ML inference workers.
  Difficulty: Intermediate. Reading time: ~30-45 min for the core concepts pages.

- **kubernetes/autoscaler — Vertical Pod Autoscaler README**
  https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/README.md
  What it teaches: Installation, `VerticalPodAutoscaler` object spec, update modes (`Off`, `Initial`, `Recreate`, `Auto`).
  Difficulty: Advanced. Reading time: ~20 min.

- **Comparison of Autoscaling Frameworks for Containerised Machine-Learning Applications in a Local and Cloud Environment** (arXiv paper)
  https://arxiv.org/pdf/2311.18659
  What it teaches: An empirical, academic comparison of Kubernetes-native autoscaling approaches specifically for ML-serving workloads across local and cloud environments — the closest thing to a research-grade source on this exact topic.
  Difficulty: Advanced. Reading time: ~40-60 min.

---

## 4. StatefulSets

- **Kubernetes Documentation — StatefulSets**
  https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
  What it teaches: Stable, unique network identities and ordered, graceful deployment/scaling for stateful workloads; when to prefer StatefulSet over Deployment (e.g., for sharded vector-DB nodes, distributed inference servers requiring stable peer identity, or leader-election-based ML serving clusters).
  Difficulty: Intermediate-Advanced. Reading time: ~25 min.

---

## 5. Kubeflow and KServe

- **Kubeflow — Introduction**
  https://www.kubeflow.org/docs/started/introduction/
  What it teaches: The overall Kubeflow architecture and component map (Notebooks, Pipelines, Katib, Trainer, KServe) and how they compose into an end-to-end ML platform on Kubernetes.
  Difficulty: Beginner-Intermediate. Reading time: ~15 min.

- **Kubeflow — KServe Ecosystem Page**
  https://www.kubeflow.org/docs/ecosystem/kserve/
  What it teaches: How KServe is positioned within Kubeflow specifically (as opposed to standalone) and the install path via the Kubeflow manifests.
  Difficulty: Intermediate. Reading time: ~10 min.

- **KServe Official Documentation**
  https://kserve.github.io/website/
  What it teaches: The full `InferenceService` CRD reference, supported runtimes (TensorFlow Serving, Triton, Hugging Face Server, XGBoost, LightGBM), predictor/transformer/explainer component model, canary rollout support built into `InferenceService`, and autoscaling (Knative-based scale-to-zero, concurrency-based scaling).
  Difficulty: Intermediate. Reading time: ~1-2 hours for the core sections.

- **KServe — Deploy Your First Predictive Model with InferenceService**
  https://kserve.github.io/website/docs/getting-started/predictive-first-isvc
  What it teaches: A concrete, step-by-step first deployment — the canonical "hello world" this chapter's hands-on KServe exercise should mirror.
  Difficulty: Beginner-Intermediate. Reading time: ~20 min hands-on.

- **KServe — Understanding LLMInferenceService**
  https://kserve.github.io/website/docs/model-serving/generative-inference/llmisvc/llmisvc-overview
  What it teaches: KServe's newer (2025/2026-era) generative-inference-specific CRD, purpose-built for LLM serving (as distinct from the original predictive-model-focused `InferenceService`) — reflects how KServe has evolved specifically to address LLM serving concerns like continuous batching and multi-accelerator scheduling.
  Difficulty: Advanced. Reading time: ~30 min.

- **KServe — Resources / Concepts**
  https://kserve.github.io/website/docs/concepts/resources
  What it teaches: How resource requests/limits (including GPU) are specified inside an `InferenceService` predictor spec — directly relevant to combining the GPU scheduling section with the KServe section.
  Difficulty: Intermediate. Reading time: ~10 min.

- **NVIDIA Developer Blog — Deploying Disaggregated LLM Inference Workloads on Kubernetes**
  https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/
  What it teaches: How NVIDIA reasons about deploying prefill/decode-disaggregated LLM inference (the llm-d / Dynamo pattern) on Kubernetes — the frontier-scale evolution beyond a single-replica KServe InferenceService, useful context for why the ecosystem introduced projects like llm-d in 2025-2026.
  Difficulty: Advanced. Reading time: ~25 min.

- **CNCF Blog — Welcome llm-d to the CNCF: Evolving Kubernetes into SOTA AI Infrastructure**
  https://www.cncf.io/blog/2026/03/24/welcome-llm-d-to-the-cncf-evolving-kubernetes-into-sota-ai-infrastructure/
  What it teaches: Why IBM Research, Red Hat, and Google Cloud donated llm-d (a distributed, vLLM-based, Kubernetes-native LLM inference framework) to CNCF in March 2026 — important 2026-era context showing that "Kubernetes for LLM serving" is an actively evolving frontier beyond the KServe fundamentals this chapter teaches directly.
  Difficulty: Intermediate. Reading time: ~10 min.

- **CNCF Blog — KServe Becomes a CNCF Incubating Project**
  https://www.cncf.io/blog/2025/11/11/kserve-becomes-a-cncf-incubating-project/
  What it teaches: KServe's governance/maturity status (accepted to CNCF Incubating, Sept 29, 2025) — relevant for framing KServe's production-readiness and community backing when making build-vs-adopt decisions.
  Difficulty: Beginner. Reading time: ~5 min.

---

## 6. Helm

- **Helm Official Documentation**
  https://helm.sh/docs/
  What it teaches: Chart structure (`Chart.yaml`, `values.yaml`, `templates/`), templating with Go templates and Sprig functions, releases, upgrades/rollbacks, and dependency management.
  Difficulty: Beginner-Intermediate. Reading time: ~1 hour for the core "Using Helm" guide.

- **Helm — The Chart Best Practices Guide**
  https://helm.sh/docs/chart_best_practices/
  What it teaches: Official conventions for chart structure, values design, template formatting, labels/annotations, and CRD handling — the standard a senior engineer's Helm charts should be held to.
  Difficulty: Intermediate. Reading time: ~30-40 min.

---

## 7. Rolling Updates, Canary, and Blue-Green Deployments

- **Kubernetes Documentation — Performing a Rolling Update (Deployments)**
  https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment
  What it teaches: The native rolling-update mechanics available without any additional controller — `maxSurge`/`maxUnavailable`, readiness-gated traffic cutover.
  Difficulty: Beginner-Intermediate. Reading time: ~15 min.

- **Argo Rollouts Official Documentation — Concepts**
  https://argo-rollouts.readthedocs.io/en/stable/concepts/
  https://argoproj.github.io/rollouts/
  What it teaches: The `Rollout` CRD's `strategy` block for canary (with `steps`, weighted traffic, and `analysis` for automated metric-based promotion/rollback) and blue-green (active/preview service split) — the production-grade progressive-delivery layer this chapter's canary/blue-green section is built on.
  Difficulty: Intermediate-Advanced. Reading time: ~45-60 min.

- **Red Hat Blog — How to Do Blue/Green and Canary Deployments with Argo Rollouts**
  https://www.redhat.com/en/blog/blue-green-canary-argo-rollouts
  What it teaches: A practical, engineering-blog-level walkthrough contrasting the two strategies with concrete manifest diffs.
  Difficulty: Intermediate. Reading time: ~15-20 min.

---

## 8. Kubernetes vs ECS vs Serverless

- **AWS Documentation — Modern Apps Strategy on AWS: How to Choose**
  https://docs.aws.amazon.com/decision-guides/latest/modern-apps-strategy-on-aws-how-to-choose/modern-apps-strategy-on-aws-how-to-choose.html
  What it teaches: AWS's own decision framework across EKS, ECS, Fargate, and Lambda — the official source for reasoning about the "when to choose which" comparison table in this chapter.
  Difficulty: Intermediate. Reading time: ~20-30 min.

- **Comparison of Autoscaling Frameworks for Containerised ML Applications** (arXiv, cited above)
  https://arxiv.org/pdf/2311.18659
  Difficulty: Advanced. Reading time: ~40-60 min. (Cross-referenced here as it also bears directly on the K8s-vs-alternatives cost/operational tradeoff discussion for ML workloads.)

---

## Notes on currency (mid-2026)

- Kubernetes is on the v1.35/v1.36 line as of mid-2026 (v1.34 released August 2025, v1.35 released December 2025/January 2026); the HPA v2 API and VPA (still beta) discussed above are stable across these versions — manifests in this chapter should target `autoscaling/v2`.
- KServe was accepted as a CNCF Incubating project in September 2025 — a signal of production maturity that did not exist in earlier (2023-2024-era) MLOps course material.
- Kubeflow's 1.11 release (December 2025) repositioned the project as the "Kubeflow AI Reference Platform," explicitly sharpening focus toward generative AI and distributed LLM fine-tuning/training rather than purely classical-ML pipelines — worth noting explicitly in the chapter as a shift from 2024-era framing of Kubeflow as "just an ML pipelines tool."
- llm-d (Kubernetes-native, vLLM-based, disaggregated-prefill/decode LLM inference) joined CNCF as a Sandbox project in March 2026 — the clearest sign that "Kubernetes for LLM systems" is actively growing a dedicated sub-ecosystem beyond general-purpose KServe/Triton serving, and is worth a forward-looking callout in the chapter even though hands-on examples should still center on the more mature KServe.
