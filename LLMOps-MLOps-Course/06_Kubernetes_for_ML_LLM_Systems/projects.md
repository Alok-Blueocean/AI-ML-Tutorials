# Module 06 — Example Projects: Kubernetes for ML and LLM Systems

Three projects, increasing in scope and production-realism, all built on this module's material.
Each specifies scope and what it demonstrates — build them in order; the Medium project's
autoscaled `InferenceService` is the workload the Production project wraps in Helm, Argo Rollouts,
and multi-tenant governance, and the Production project's stack is what Module 07/08 later
instruments with full observability and Module 09 wires into CI/CD.

---

## Mini — Deploy a Single Model as a Health-Gated Deployment Behind an Ingress

**Scope:** Take a single, already-containerized model (reuse the FastAPI churn-model image from
Module 05, or any equivalent) and run it as a correctly-configured Kubernetes Deployment, exposed
through a Service and an Ingress, on a local cluster (`kind`/`minikube`/`k3d`).

**What to build:**
- A `Deployment` with 3 replicas, explicit `requests`/`limits` for CPU and memory, and both
  `readinessProbe` and `livenessProbe` pointed at distinct `/ready` and `/health` endpoints
  (Section 3.1/3.2).
- A `ClusterIP` `Service` selecting the Deployment's Pods.
- An `Ingress` (with an installed Ingress controller) routing a path (e.g., `/churn`) to that
  Service.
- A `PodDisruptionBudget` ensuring at least 2 of 3 replicas stay available during voluntary
  disruptions.

**What it demonstrates:**
- The foundational Pod/Deployment/Service/Ingress mechanics from Section 3.1 applied end to end on
  the simplest possible serving payload.
- Correct `requests`/`limits` reasoning and QoS-class awareness (Section 3.2) on a real workload.
- That you can prove self-healing and readiness-gating yourself (`kubectl delete pod`, then watch
  traffic through the Service keep flowing) rather than taking it on faith.

**Definition of done:** `kubectl apply -f` brings up the full stack; deleting any one Pod directly
results in a same-named-Deployment replacement within seconds with zero client-visible downtime
through the Service; a request through the Ingress path returns a correct prediction; the
`PodDisruptionBudget` visibly blocks a `kubectl drain` from evicting more than one Pod at a time.

---

## Medium — Autoscaled KServe `InferenceService` With Queue-Depth-Based Scale-to-Zero

**Scope:** Move from a hand-rolled Deployment to KServe's standardized serving layer for an LLM (or
LLM-shaped) endpoint, with real autoscaling wired to the metric that actually reflects load rather
than CPU, and a canary rollout of a new model revision.

**What to build:**
- KServe installed standalone (no full Kubeflow needed) on your cluster.
- An `InferenceService` serving a small open LLM or a Hugging Face sentiment/classification model
  (CPU-friendly substitute is fine if you don't have GPU access), with `minReplicas: 0` and
  concurrency-based autoscaling (`autoscaling.knative.dev/target`) per Section 3.5's intermediate
  example.
- A KEDA `ScaledObject` fronting a separate async batch-inference worker Deployment, scaling on
  RabbitMQ (or Redis-list) queue depth, including a confirmed scale-to-zero after the queue drains
  (Section 3.3's production example).
- A canary rollout of a "v2" model revision using `canaryTrafficPercent: 10`, with a version marker
  in the response payload so you can measure the actual observed traffic split, followed by full
  promotion.
- A DCGM-exporter-backed (or synthetic, if CPU-only) Prometheus metric feeding an HPA `External`
  metric on a second, GPU-simulated Deployment, to exercise the "scale on GPU util/queue depth, not
  CPU" pattern even without real GPU hardware.

**What it demonstrates:**
- KServe's value proposition over bare Deployments — standardized runtime + native canary field
  (Section 3.5) — applied to a real request flow.
- The full HPA/KEDA autoscaling decision framework from Section 3.3, including scale-to-zero, on a
  workload shape that actually needs it (bursty, cost-sensitive inference).
- Correct reasoning about *why* CPU is the wrong autoscaling signal for GPU/LLM-bound workloads,
  backed by your own measured evidence rather than a memorized rule.

**Definition of done:** the `InferenceService` scales from zero to serving and back to zero across
an idle period; the canary step shows a traffic split within a few percentage points of the
configured 90/10; the KEDA-scaled batch worker demonstrably scales to 0 replicas after
`cooldownPeriod` with an empty queue and back up under a message burst; you have a recorded
before/after showing CPU-based HPA failing to react to synthetic GPU-bound load versus the
external-metric HPA reacting correctly.

---

## Production-Grade — Helm-Packaged, Multi-Tenant Platform With Argo Rollouts and Governance

**Scope:** Extend the Medium project into a platform other teams could plausibly self-serve onto:
one reusable Helm chart serving both classical-ML and LLM workloads across dev/staging/prod, fully
governed with resource quotas, progressive delivery with automated rollback, and a documented
Kubernetes-vs-alternatives justification — this is the direct predecessor artifact to what Module
07/08 instruments with observability and Module 09 wraps in a full CI/CD pipeline.

**What to build:**
- A single Helm chart (Section 3.6) with `gpu.enabled` and `kserve.enabled` toggles, templated
  `Deployment`/`InferenceService`/`HPA`/`PodDisruptionBudget` resources, and `values-dev.yaml` /
  `values-staging.yaml` / `values-prod.yaml` overrides — demonstrated by promoting the *same* chart
  across three namespaces with independently tracked Helm release revisions (`helm list -A`,
  `helm rollback`).
- `ResourceQuota` and `LimitRange` objects per namespace, with a deliberate test proving a
  misconfigured or over-requesting workload in one "tenant" namespace cannot starve another
  namespace's allocation (Section 6's multi-tenant governance note, informed by the Databricks-style
  case study in Section 4).
- Argo Rollouts replacing the prod release's Deployment with a `Rollout` object, canary steps with
  an `AnalysisTemplate` querying a real Prometheus instance for success-rate and p99-latency
  thresholds (Section 3.7's production example), and a proven automated abort-and-rollback triggered
  by a deliberately-induced bad metric — not a manually-invoked `kubectl rollout undo`.
- A StatefulSet-backed vector database (Qdrant or equivalent) with anti-affinity and a
  `PodDisruptionBudget`, backing a RAG-style `InferenceService`, with a documented PVC-reattachment
  test proving data survives a Pod deletion at a specific ordinal (Section 3.4).
- A GPU node-pool section (real if you have access, fully reasoned/documented if not) specifying
  the sharing strategy — whole-GPU, time-slicing, or MIG — chosen per workload class, justified
  against the comparison table in Section 3.2.
- A one-page runbook mapping every primitive used back to the module's decision trees (Deployment
  vs. StatefulSet vs. `InferenceService`; HPA vs. VPA vs. KEDA; rollout strategy selection;
  Kubernetes vs. ECS vs. serverless), with the production checklist from `tutorial.md` Section 8
  checked off item by item against your actual manifests.

**What it demonstrates:**
- The full Section 3.6 Helm packaging pattern as a genuine platform-product surface — one chart,
  many models/environments — rather than a tutorial-only exercise.
- Multi-tenant resource governance (`ResourceQuota`/`LimitRange`) as a day-one concern, not a
  retrofit, mirroring how a Databricks-scale multi-tenant ML platform vendor would plausibly need
  to isolate noisy or misconfigured tenants from each other.
- Argo Rollouts' automated-analysis-gated canary as a genuine safety mechanism (an unattended
  abort-and-rollback you triggered and observed), not a manual "engineer watches a dashboard"
  process — the production-grade distinction Section 3.7 draws explicitly.
- End-to-end command of every major decision tree in this module (Deployment vs. StatefulSet vs.
  KServe; HPA vs. VPA vs. KEDA; rollout strategy; platform choice), defensible against interview-style
  "why did you choose X over Y here" questioning.

**Definition of done:** a single Helm chart deployed as three independently-versioned releases
across dev/staging/prod namespaces; a documented, tested `ResourceQuota` boundary that provably
prevents cross-tenant starvation; one Argo Rollouts run that promoted cleanly on good metrics and a
second run where a deliberately bad metric triggered a fully automatic abort-and-rollback with zero
manual intervention; a StatefulSet vector store with a proven PVC-reattachment test; and a completed
runbook with every production-checklist item checked against real, applied manifests — an artifact
detailed enough to hand to a platform team lead for review, and to walk an interviewer through
end to end.
