# Books — Module 06: Kubernetes for ML and LLM Systems

These are general-purpose Kubernetes references (there is no widely-adopted "Kubernetes for ML" book specifically, so this list combines the standard canon with chapters/sections most relevant to ML/LLM serving workloads). Reading times are estimates based on publisher-listed page/hour counts where available.

---

## 1. Kubernetes: Up and Running (3rd Edition)
- **Authors:** Brendan Burns, Joe Beda, Kelsey Hightower, Lachlan Evenson (O'Reilly, 2022)
- **What it teaches:** The definitive from-the-source introduction to Kubernetes, written by engineers who worked on Kubernetes at Google from its inception. Covers the full object model (Pods, Deployments, Services, Ingress, ConfigMaps/Secrets, DaemonSets, Jobs, StatefulSets) plus cluster architecture, storage, RBAC, and multi-cluster operations. The book explicitly frames Kubernetes as suitable for "online services, machine learning applications, or even a cluster of Raspberry Pi computers."
- **Difficulty:** Beginner to Intermediate
- **Estimated reading time:** ~8 hours / 328 pages (publisher-listed)
- **Why it matters for this module:** This is the correct starting reference for every YAML primitive used in this chapter (Deployment, Service, Ingress, HPA). Read Chapters 1-10 before attempting the GPU/KServe material — those chapters assume you're comfortable with the base object model.

## 2. Kubernetes Patterns (2nd Edition): Reusable Elements for Designing Cloud-Native Applications
- **Authors:** Bilgin Ibryam, Roland Huß (O'Reilly / Red Hat, 2023)
- **What it teaches:** A pattern-language approach to Kubernetes — Foundational patterns (Predictable Demands, Declarative Deployment, Health Probe), Behavioral patterns (Batch Job, Periodic Job, Daemon Service — directly relevant to model-serving Deployments and StatefulSets), Structural patterns (Sidecar, Init Container, Adapter — relevant to model-server sidecars for logging/metrics), and Configuration patterns.
- **Difficulty:** Intermediate
- **Estimated reading time:** ~6-7 hours for the relevant pattern chapters (full book is longer)
- **Why it matters for this module:** The "Predictable Demands" pattern is literally the resource requests/limits discussion this chapter covers, formalized with production rationale (why setting requests without limits, or limits without requests, causes cluster instability). The "Daemon Service" pattern underlies why node-level GPU device plugins are deployed as DaemonSets rather than Deployments.

## 3. Kubernetes in Action (2nd Edition)
- **Authors:** Marko Lukša, Kevin Conner (Manning, 2023 update)
- **What it teaches:** An end-to-end, deeply mechanical explanation of how Kubernetes actually works internally — the API server, controllers, scheduler, kubelet, and how a Deployment's rollout actually mutates ReplicaSets under the hood. Strong on the "why" behind rolling updates, readiness/liveness probes, and StatefulSet pod identity guarantees.
- **Difficulty:** Intermediate (assumes some container/Docker background, no prior Kubernetes needed)
- **Estimated reading time:** ~15+ hours (it is a large, thorough book — read selectively: chapters on Deployments, StatefulSets, and autoscaling are the ones directly relevant here)
- **Why it matters for this module:** Its treatment of rolling updates (`maxSurge`/`maxUnavailable`, readiness gates) and StatefulSet ordering/stable-identity guarantees is the clearest available explanation of the mechanics this chapter's "rolling updates and StatefulSets" sections rely on. Use it as the reference to go deeper after the chapter, not before.

## 4. Programming Kubernetes: Developing Cloud-Native Applications
- **Authors:** Michael Hausenblas, Stefan Schimanski (O'Reilly, 2019)
- **What it teaches:** How to write Kubernetes-native controllers and Custom Resource Definitions (CRDs) using client-go. This is exactly the layer that Kubeflow, KServe, KEDA, and Argo Rollouts are all built on — every one of them ships a CRD (InferenceService, ScaledObject, Rollout) plus a controller reconciling it.
- **Difficulty:** Advanced (assumes solid Go and Kubernetes API familiarity)
- **Estimated reading time:** ~6 hours / 270 pages
- **Why it matters for this module:** You do not need to write your own operator to use KServe or KEDA, but understanding the controller/reconciliation-loop pattern this book teaches is what separates engineers who can debug a stuck `InferenceService` or `ScaledObject` from those who can only apply YAML and hope. Read the "Custom Resources" and "Shipping Controllers and Operators" chapters for the conceptual model; you do not need the full API-server-extension material for this module.

## 5. Designing Machine Learning Systems — Chapter 9-10 context (supplementary, not Kubernetes-specific)
- **Author:** Chip Huyen (O'Reilly, 2022)
- **What it teaches:** While not a Kubernetes book, the deployment and infrastructure chapters give the "why" for autoscaling on inference latency/queue depth rather than CPU, and for treating model servers as a distinct workload class from ordinary web services — the framing this chapter's HPA/KEDA/queue-depth-scaling section builds on.
- **Difficulty:** Intermediate
- **Estimated reading time:** ~1-2 hours for the relevant chapters
- **Why it matters for this module:** Gives the ML-systems reasoning (batching, latency SLOs, cost-per-inference) that justifies *why* this chapter recommends KEDA queue-depth scaling over plain CPU-based HPA for LLM inference workloads — the mechanism (KEDA) is Kubernetes-specific, but the motivation is an MLOps systems-design concern this book covers well.

---

## Suggested reading order for this module

1. Skim relevant chapters of **Kubernetes: Up and Running** (object model) if you're new to K8s.
2. Read **Kubernetes Patterns** — Predictable Demands and Daemon Service patterns — alongside this chapter's GPU scheduling section.
3. Read **Kubernetes in Action**'s StatefulSets and rolling-update chapters alongside this chapter's corresponding sections.
4. Treat **Programming Kubernetes** and the Huyen chapters as optional depth for readers heading toward platform-engineering or ML-infra-lead roles.
