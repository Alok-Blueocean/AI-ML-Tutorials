# Module 06 Cheat Sheet — Kubernetes for ML and LLM Systems

Condensed, skimmable reference. Full explanations live in `tutorial.md`; deep dives and decision
trees in `architecture.md`.

---

## Core Mental Model

| Concept | One-line definition |
|---|---|
| **Pod** | Smallest deployable unit; one or more containers, same node, same network namespace. Disposable — never SSH in expecting it tomorrow. |
| **Deployment** | Manages identical, stateless replicas via a ReplicaSet; knows how to roll forward and back. |
| **Service** | Stable DNS/VIP load-balancing across `Ready` Pods matching a label selector. |
| **Ingress** | L7 host/path routing to Services, via a separately-installed Ingress controller (NGINX/ALB/Istio). |
| **StatefulSet** | For workloads where "which specific replica" matters — stable identity + stable, reattached storage + ordered start/stop. |
| **HPA / VPA / KEDA** | Scale replica count / scale requests-limits / event-driven scale-including-to-zero. |
| **KServe `InferenceService`** | CRD that materializes standardized, canary-capable, autoscaling model-serving infra from one YAML object. |
| **Helm** | Templated YAML ("chart") + versioned release/values model for repeatable, environment-promotable deployments. |

---

## Requests / Limits / QoS

```yaml
resources:
  requests: { cpu: "500m", memory: "512Mi" }   # guaranteed; scheduler uses this
  limits:   { cpu: "1",    memory: "1Gi"   }   # hard ceiling; kubelet enforces this
```

| Resource | Over-limit behavior |
|---|---|
| CPU | **Throttled** (compressible) — slower, not killed |
| Memory | **OOMKilled** (incompressible) — abrupt SIGKILL, no graceful shutdown |

| QoS class | Condition | Eviction priority |
|---|---|---|
| `Guaranteed` | `requests == limits` for every resource | Evicted **last** |
| `Burstable` | `requests < limits` | Evicted middle |
| `BestEffort` | No requests/limits set | Evicted **first** — never ship to prod |

---

## GPU Scheduling Rules

```yaml
resources:
  requests: { cpu: "4", memory: "16Gi" }
  limits:
    cpu: "8"
    memory: "24Gi"
    nvidia.com/gpu: 1        # LIMITS ONLY. Whole units only. No "0.5".
```

- GPUs = extended resources via **device plugin** (DaemonSet) → advertised as `nvidia.com/gpu`.
- Set GPU only in `limits`; Kubernetes auto-copies it to `requests`. **Fractional values rejected.**
- **GPU Operator** = driver + Container Toolkit + device plugin + Node Feature Discovery + DCGM
  exporter, managed as one `ClusterPolicy` CR.

| Strategy | Isolation | Best for |
|---|---|---|
| Whole-GPU (default) | Full | Latency-SLA production inference, training |
| Time-slicing | None (VRAM shared, can OOM) | Dev/test, low-QPS, bin-packing small models |
| MIG (A100/H100+) | Hardware partitions | Multi-tenant prod needing real isolation |

---

## Autoscaling Decision Rule

```
GPU-bound inference?  →  Scale on GPU util (DCGM→Prom→Adapter→HPA External) and/or
                          queue depth / concurrency (KEDA) — NEVER plain CPU.
CPU-bound classical ML? →  Plain HPA on cpu Resource metric is fine.
Unknown resource footprint? → VPA (recommendation-only or Auto), NOT more replicas.
Need scale-to-zero / event source (queue, cron, Kafka lag)? → KEDA ScaledObject/ScaledJob.

NEVER: HPA + VPA actively managing the SAME metric on the SAME workload — they fight.
```

```yaml
# HPA on external GPU-util metric
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: External
      external:
        metric: { name: DCGM_FI_DEV_GPU_UTIL }
        target: { type: AverageValue, averageValue: "75" }
  behavior:
    scaleDown: { stabilizationWindowSeconds: 300 }   # avoid flapping
```

```yaml
# KEDA ScaledObject — queue-depth, scale-to-zero
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
spec:
  scaleTargetRef: { name: llm-batch-worker }
  minReplicaCount: 0
  maxReplicaCount: 50
  cooldownPeriod: 120
  triggers:
    - type: rabbitmq
      metadata: { queueName: llm-inference-jobs, mode: QueueLength, value: "10" }
```

KEDA does not replace HPA — it manages a standard HPA object once replicas > 0, and adds the
scale-to-zero decision layer in front of it.

---

## StatefulSet Skeleton

```yaml
apiVersion: v1
kind: Service
metadata: { name: vectordb-svc }
spec: { clusterIP: None, selector: { app: vectordb } }   # headless
---
apiVersion: apps/v1
kind: StatefulSet
spec:
  serviceName: vectordb-svc
  replicas: 3
  volumeClaimTemplates:
    - metadata: { name: data }
      spec: { accessModes: ["ReadWriteOnce"], resources: { requests: { storage: 50Gi } } }
```

- Pod names: `vectordb-0`, `-1`, `-2` — stable, reused across reschedules.
- DNS: `<pod>.<headless-svc>.<ns>.svc.cluster.local`.
- Startup order: 0→1→2 (each Ready before next). Scale-down: reverse.
- **Gives you**: identity + storage reattachment. **Does NOT give you**: correct
  rejoin/resync/leader-election logic — that's the application's job.
- **Default bias**: prefer a managed service (RDS, managed Redis, hosted vector DB) over
  self-hosting on a StatefulSet unless there's a specific reason it doesn't fit.

---

## KServe `InferenceService` — Minimal + Canary

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata: { name: churn-model }
spec:
  predictor:
    sklearn:
      storageUri: "gs://my-models/churn-model/v1"
      resources: { requests: { cpu: "1", memory: "1Gi" }, limits: { cpu: "2", memory: "2Gi" } }
```

```yaml
# Canary: 10% to new revision, one-line promote/rollback
spec:
  predictor:
    canaryTrafficPercent: 10
    containers:
      - image: registry.example.com/fraud-model:v4
```
Promote: `canaryTrafficPercent: 100` (or drop the field). Rollback: revert image or set back to 0.

**Decision rule:** use KServe once you have more than a couple of models, need consistent canary
/autoscaling/health behavior across them, or are building a self-serve platform. One simple,
low-change model → plain Deployment+Service is fine.

`LLMInferenceService` (2025-2026): purpose-built CRD for generative inference (continuous batching,
KV-cache, disaggregated prefill/decode) — master `InferenceService` first; treat this and **llm-d**
as the forward-looking frontier layer.

---

## Helm Essentials

```
model-serving-chart/
├── Chart.yaml
├── values.yaml
└── templates/{deployment,service,hpa}.yaml
```

```bash
helm upgrade --install churn-model ./chart \
  -f ./chart/values.yaml -f ./chart/values-prod.yaml \
  --namespace ml-prod --create-namespace

helm rollback churn-model 1        # revert to a previous release revision
helm list -A                        # see all releases across namespaces
```

**Decision rule:** Helm once you need cross-environment parameterization or "one chart, many
models." A single static, rarely-changing manifest → plain `kubectl apply -f` (or Kustomize) is
simpler. Pin chart versions in CI; never `helm upgrade` against a floating `latest`.

---

## Rollout Strategies — Pick One

| Strategy | Mechanism | Traffic control | Rollback | Compute cost | Best for |
|---|---|---|---|---|---|
| Rolling update (native) | `maxSurge`/`maxUnavailable`, readiness-gated | None (all-or-nothing) | Re-rollout old ReplicaSet | 1x (+surge) | Low-risk, easily-reversible changes |
| Canary (Argo Rollouts / KServe native) | Weighted `steps`, `AnalysisTemplate` gates on Prometheus | Fine-grained % | Automatic on failed analysis | Partial extra | Default for ML/LLM version changes |
| Blue-green (Argo Rollouts) | Full duplicate env, atomic cutover | All-or-nothing but instant | Instant (flip back) | 2x during overlap | Unusually risky changes (new inference engine) |

```yaml
# Argo Rollouts canary + automated analysis gate
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - analysis: { templates: [{ templateName: success-rate-and-latency }] }
        - setWeight: 50
        - pause: { duration: 5m }
        - analysis: { templates: [{ templateName: success-rate-and-latency }] }
        - setWeight: 100
```

**Key insight:** a bad model returns HTTP 200 all day — canary analysis for ML must include
model-quality signals (prediction drift, confidence calibration, eval scores), not just
latency/error-rate.

---

## Kubernetes vs. ECS vs. Serverless — Quick Rule

```
GPU-bound + sustained/variable traffic?           → Kubernetes
GPU-bound + genuinely spiky, cold-start-tolerant?  → Serverless GPU, else fall back to K8s (min>=1)
CPU-only + spiky/low-volume?                       → Serverless (Lambda/Cloud Run)
AWS-only, want less ops than K8s, steady traffic?  → ECS/Fargate
Multi-cloud/on-prem portability needed?             → bias Kubernetes (any workload)
Already running K8s for other workloads?            → bias Kubernetes (marginal cost is low)
Small team, no platform engineers, one simple model?→ bias serverless/ECS even if GPU-capable
```
It's normal for a mature platform to run all three for different workload shapes at once.

---

## Top Mistakes to Never Make

1. `nvidia.com/gpu: "0.5"` or GPU in `requests` only — GPUs are whole-unit, `limits`-only.
2. Autoscaling GPU-bound inference on CPU utilization — it never fires when it should.
3. HPA and VPA both actively managing the same metric on the same workload.
4. Shipping `BestEffort` QoS (no requests/limits) to production — first thing evicted.
5. Treating a StatefulSet Pod like a Deployment Pod — identity/storage ≠ correct rejoin logic.
6. Plain rolling update + "no HTTP errors" as sufficient validation for a model version change.
7. Reaching for Kubernetes by default for one simple, low-traffic model.
8. Missing `terminationGracePeriodSeconds`/`preStop` on latency-sensitive services — abrupt
   in-flight-request cutoff during rollouts/scale-down.
9. Installing all of Kubeflow when only KServe (standalone) is needed.

---

## Production Checklist (copy into your PR template)

```
[ ] Every container has explicit, right-sized requests AND limits (no BestEffort in prod)
[ ] GPU workloads: whole nvidia.com/gpu units in limits; time-slicing/MIG chosen deliberately
[ ] Autoscaling metric matches true bottleneck (GPU util/queue depth, not blind CPU)
[ ] HPA and VPA are not both managing the same metric on the same workload
[ ] readinessProbe/livenessProbe set; terminationGracePeriodSeconds/preStop for graceful drain
[ ] PodDisruptionBudget in place for any multi-replica production service
[ ] Stateful components: StatefulSet + anti-affinity + PDB, or offloaded to a managed service
[ ] Model serving standardized behind KServe (or equivalent) past a handful of models
[ ] Packaged as a Helm chart with environment-specific values-*.yaml, versioned in CI
[ ] Rollouts use canary (KServe native or Argo Rollouts) with automated, metric-gated
    promotion/rollback, including model-quality signals
[ ] ResourceQuota/LimitRange set per namespace in any multi-tenant cluster
[ ] Kubernetes-vs-ECS-vs-serverless choice documented per workload against explicit criteria
```
