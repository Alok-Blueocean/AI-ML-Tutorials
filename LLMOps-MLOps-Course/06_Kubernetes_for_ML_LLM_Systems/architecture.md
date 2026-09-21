# Architecture Deep-Dive — Module 06: Kubernetes for ML and LLM Systems

This file complements `tutorial.md` with larger, more detailed ASCII architecture diagrams,
request-flow sequence diagrams, and decision trees for choosing between the tools this module
covers. Read `tutorial.md` first for the theory; use this file as the visual/reference layer
you come back to while designing a real system or answering a system-design interview question.

---

## 1. Full-Stack Reference Architecture — ML/LLM Serving on Kubernetes

This is the "zoomed out" picture: every concept in the tutorial, assembled into one cluster.
Not every production system needs every box below — treat it as a menu, not a mandate.

```
                                              INTERNET / INTERNAL CLIENTS
                                                        |
                                                        v
                                   +---------------------------------------------+
                                   |     Cloud Load Balancer (L4) / DNS            |
                                   +---------------------+-------------------------+
                                                         |
                                                         v
                                   +---------------------------------------------+
                                   |          Ingress Controller (L7)              |
                                   |   NGINX / AWS ALB Controller / Istio Gateway  |
                                   |   host/path routing, TLS termination          |
                                   +---------------------+-------------------------+
                                                         |
                        +--------------------------------+--------------------------------+
                        |                                                                  |
                        v                                                                  v
          +--------------------------+                                    +--------------------------+
          |   AI Gateway / Router     |                                    |  Classical-ML API Gateway |
          |  (model selection, rate   |                                    |  (routes by model name /   |
          |   limit, cost tracking,    |                                    |   version to Service)      |
          |   auth)                   |                                    +-------------+--------------+
          +------------+--------------+                                                  |
                       |                                                                  |
     ----------------------------------------                                             |
    |                    |                    |                                            |
    v                    v                    v                                            v
+---------+       +------------+      +--------------+                          +-------------------+
| KServe   |       | KServe      |      | llm-d /       |                          |  Deployment +      |
| Inference|       | LLMInference|      | disaggregated |                          |  Service + HPA      |
| Service   |       | Service     |      | prefill/decode|                          |  (churn/fraud model)|
| (predict- |       | (generative  |      | inference mesh|                          +-------------------+
|  ive ML)  |       |  inference)  |      | (frontier      |
+----+-----+       +------+-------+      |  scale only)   |
     |                     |               +-------+--------+
     v                     v                       |
+-----------+      +---------------+       +-----------------+--------------------+
| Predictor  |      | Predictor      |       |  Prefill pool    |   Decode pool      |
| (sklearn/  |      | (vLLM/TGI,      |       |  (compute-bound, |   (mem-bandwidth-  |
|  xgboost/   |      |  continuous     |       |   large batch,    |   bound, tight      |
|  Triton)    |      |  batching)      |       |   fewer, bigger    |   per-token          |
+-----------+      +------+---------+       |   GPU nodes)      |   latency, more      |
      |                    |                 +-----------------+   smaller GPU nodes) |
      v                    v                                    +----------------------+
 GPU/CPU node pool    GPU node pool
 (device plugin /     (device plugin, MIG or
  time-slicing)         time-slicing, KV-cache
                         aware scheduling)

===================================  CONTROL / SUPPORT PLANE  ===================================

  +----------------------+   +------------------------+   +---------------------------+
  |  Autoscaling layer     |   |  Stateful backing        |   |  Rollout / delivery layer   |
  |  HPA (CPU/mem/custom)  |   |  services (StatefulSet)  |   |  Argo Rollouts (canary,      |
  |  VPA (right-sizing)    |   |  - vector DB (Qdrant/     |   |  blue-green, AnalysisTemplate|
  |  KEDA (queue depth,     |   |    Milvus, sharded)       |   |  querying Prometheus)        |
  |  scale-to-zero)         |   |  - self-hosted Redis      |   |  KServe native canaryTraffic |
  +-----------+------------+   |    Cluster / feature store|   |  Percent (simpler cases)     |
              |                +------------+-------------+   +---------------------------+
              v                             |
  +----------------------+                  v
  |  Metrics pipeline      |   +------------------------+
  |  DCGM exporter (GPU)   |   |  Persistent Volumes      |
  |  App /metrics (vLLM)   |   |  (fast-ssd StorageClass)  |
  |  -> Prometheus          |   +------------------------+
  |  -> Prometheus Adapter  |
  |  -> Custom/External     |
  |     Metrics API         |
  +----------------------+

  +----------------------------------------------------------------------------------+
  |  Platform / packaging layer                                                        |
  |  Helm charts (one chart, many values-*.yaml per model/environment)                  |
  |  Kubeflow (Notebooks, Pipelines, Katib, Trainer) -- optional, upstream of serving    |
  +----------------------------------------------------------------------------------+

  +----------------------------------------------------------------------------------+
  |  Node-level GPU stack (per GPU node, managed as one unit by NVIDIA GPU Operator)     |
  |  NVIDIA driver -> Container Toolkit -> Device Plugin (DaemonSet) -> Node Feature      |
  |  Discovery (labels) -> DCGM Exporter (telemetry) -> time-slicing / MIG config          |
  +----------------------------------------------------------------------------------+
```

**How to read this diagram in an interview**: start from the top (client) and narrate downward —
"traffic hits a load balancer, then an Ingress/gateway for L7 routing and model selection, then
either a classical-ML Deployment+Service+HPA stack or a KServe/LLMInferenceService for
standardized model serving, backed by GPU node pools whose hardware stack is managed by the GPU
Operator, scaled by HPA/VPA/KEDA reading from a Prometheus-based metrics pipeline, with stateful
dependencies on StatefulSets, all packaged via Helm and rolled out progressively via Argo Rollouts
or KServe's native canary field." That one paragraph *is* the module.

---

## 2. GPU Node Architecture — Device Plugin, Operator, and Sharing Strategies

```
                                   NODE (bare metal or cloud GPU instance)
   +-------------------------------------------------------------------------------------+
   |                                                                                       |
   |   Physical GPUs: [ GPU0 ] [ GPU1 ] [ GPU2 ] [ GPU3 ]                                    |
   |                                                                                       |
   |   +-----------------------------------------------------------------------------+     |
   |   |  NVIDIA GPU Operator-managed stack (installed as ClusterPolicy CR)            |     |
   |   |                                                                               |     |
   |   |   [NVIDIA Driver DaemonSet]  --loads kernel module-->  physical GPUs           |     |
   |   |            |                                                                   |     |
   |   |            v                                                                   |     |
   |   |   [NVIDIA Container Toolkit] --lets containers see GPU devices-->              |     |
   |   |            |                                                                   |     |
   |   |            v                                                                   |     |
   |   |   [Device Plugin DaemonSet] --advertises resource to kubelet-->                 |     |
   |   |            |                    "nvidia.com/gpu: 4"   (or more, if time-sliced) |     |
   |   |            v                                                                   |     |
   |   |   [Node Feature Discovery]  --labels node-->                                    |     |
   |   |            "nvidia.com/gpu.product=NVIDIA-A100-SXM4-80GB"                       |     |
   |   |            |                                                                   |     |
   |   |            v                                                                   |     |
   |   |   [DCGM Exporter]  --scraped by Prometheus-->  per-Pod/per-GPU utilization,     |     |
   |   |                       memory, temperature, ECC error metrics                     |     |
   |   +-----------------------------------------------------------------------------+     |
   |                                                                                       |
   |   kubelet (advertises "nvidia.com/gpu": N to the scheduler)                             |
   |                                                                                       |
   |   +----------------+   +----------------+   +----------------+   +----------------+     |
   |   |  Pod A          |   |  Pod B          |   |  Pod C          |   |  Pod D          |     |
   |   |  limits:        |   |  limits:        |   |  limits:        |   |  limits:        |     |
   |   |   nvidia.com/    |   |   nvidia.com/    |   |   nvidia.com/    |   |   nvidia.com/    |     |
   |   |   gpu: 1         |   |   gpu: 1         |   |   gpu: 1         |   |   gpu: 1         |     |
   |   +--------+--------+   +--------+--------+   +--------+--------+   +--------+--------+     |
   |            |                     |                     |                     |             |
   |            v                     v                     v                     v             |
   |         GPU0                  GPU1                  GPU2                  GPU3             |
   |     (dedicated,             (dedicated,             (MIG: 3x                 (time-sliced:  |
   |      whole-GPU)               whole-GPU)             1g.10gb                 4 virtual       |
   |                                                       instances --            replicas share |
   |                                                       hardware               round-robin,    |
   |                                                       isolated)               no isolation)  |
   +-------------------------------------------------------------------------------------+

   Three sharing models coexist per-node via labeling + ClusterPolicy config:
     - Whole-GPU (default): 1 Pod : 1 physical GPU, full isolation, best for SLA'd inference.
     - MIG (A100/H100-class only): hardware partitions, fixed shapes, real isolation.
     - Time-slicing: software round-robin, N virtual replicas per physical GPU, no VRAM isolation.
```

---

## 3. Autoscaling Signal Pipeline — Detailed Data Flow

```
   INFERENCE PODS                          METRICS PIPELINE                         SCALERS
   +----------------+
   | vLLM / Triton   |---(a) app metrics---> +------------------+
   | /metrics          |     (queue depth,     |   Prometheus      |
   | endpoint           |      requests-in-     |   (scrapes on a    |
   +----------------+      flight, tokens/s)   |    ~15-30s interval)|
                                              +--------+---------+
   +----------------+                                  |
   | DCGM Exporter    |---(b) GPU metrics------------->|
   | (per-node          |     (util %, VRAM used)        |
   |  DaemonSet)        |                                v
   +----------------+                        +------------------------+
                                              |  Prometheus Adapter      |
   +----------------+                        |  (rule: PromQL expr -->  |
   | RabbitMQ / SQS   |                        |   k8s Custom/External    |
   | (queue depth)     |                        |   Metrics API object)   |
   +--------+---------+                        +-----------+-------------+
            |                                              |
            | (c) polled directly                          | (d) queried by HPA
            v                                              v
   +----------------------+                       +--------------------+
   |  KEDA polling loop     |                       |   HPA controller     |
   |  (every pollingInterval|---(e) manages------->  |   (reconcile loop,   |
   |   seconds)             |   an HPA object on      |    ~15s default)      |
   +----------------------+   KEDA's behalf         +----------+-----------+
            |                                                  |
            | (f) if minReplicas=0 and no events:                |
            |     scales target Deployment to 0 directly          |
            | (g) if events present: hands off to the HPA          |
            v     it created, which scales 1..N                    v
   +---------------------------------------------------------------------+
   |                     Target Deployment / InferenceService               |
   |                     replicas: adjusted up/down within                  |
   |                     [minReplicaCount, maxReplicaCount]                 |
   +---------------------------------------------------------------------+

   Key timing intuition:
   - Prometheus scrape interval + Adapter refresh + HPA's own reconcile loop
     (default ~15s, with a scale-up/down "stabilization window" on top) means
     end-to-end reaction time to a load spike is typically low tens of seconds
     to a few minutes -- NOT instantaneous. For LLM inference, this is why
     production systems keep a warm minReplicas floor rather than relying
     purely on reactive autoscaling for latency-critical traffic.
```

---

## 4. Sequence Diagram — Canary Rollout of a New Model Version (Argo Rollouts + Prometheus Analysis)

```
Actor: ML Engineer      Argo Rollouts        Traffic Router       Old Pods (v3)   New Pods (v4)   Prometheus
      |                  Controller           (mesh/ingress)         (90%->0%)     (10%->100%)
      |                      |                      |                    |              |              |
      |--kubectl apply------>|                      |                    |              |              |
      |  (image: v4)         |                      |                    |              |              |
      |                      |--create v4 ReplicaSet------------------------------------>|              |
      |                      |                      |                    |              |              |
      |                      |--wait for v4 Ready---------------------------------------->|              |
      |                      |<--Ready (readinessProbe passes)---------------------------|              |
      |                      |                      |                    |              |              |
      |                      |--set traffic weight: v3=90%, v4=10%------->|                    |              |
      |                      |                      |--route 90%-------->|                    |
      |                      |                      |--route 10%------------------------->|              |
      |                      |                      |                    |              |              |
      |                      |--pause 5m            |                    |              |              |
      |                      |--query AnalysisTemplate------------------------------------------------->|
      |                      |                      |                    |              |   <--success-rate, p99 latency--|
      |                      |<---------------------------------------------------------------------------|
      |                      |                      |                    |              |              |
      |         [if metrics pass threshold]         |                    |              |              |
      |                      |--set traffic weight: v3=50%, v4=50%------->|                    |              |
      |                      |                      |--route 50%-------->|                    |
      |                      |                      |--route 50%------------------------->|              |
      |                      |--pause 5m, re-query analysis-------------------------------------------->|
      |                      |<---------------------------------------------------------------------------|
      |         [metrics pass again]                |                    |              |              |
      |                      |--set traffic weight: v3=0%, v4=100%------->|                    |              |
      |                      |--scale down v3 ReplicaSet to 0------------>|                    |              |
      |                      |                      |                    |  (terminated,|              |
      |                      |                      |                    |   drained via|              |
      |                      |                      |                    |   preStop)   |              |
      |<--rollout: Healthy---|                      |                    |              |              |
      |                      |                      |                    |              |              |
      |         [ALTERNATE PATH: if any analysis query fails threshold]     |              |              |
      |                      |--abort rollout------>|                    |              |              |
      |                      |--set traffic weight: v3=100%, v4=0%------->|                    |              |
      |                      |--scale v4 ReplicaSet to 0------------------------------->|              |
      |<--rollout: Degraded, auto-rolled-back-------|                    |              |              |
      |  (engineer notified via alert, no manual     |                    |              |              |
      |   rollback action was required)             |                    |              |              |
```

**What this sequence makes explicit that prose can't**: the rollback path is symmetric and
automatic — the same controller that promoted traffic toward v4 is the one that reverts it,
triggered purely by the Prometheus query result, with no human in the loop for the abort
decision itself (only for the initial `kubectl apply` and for post-incident investigation).

---

## 5. Sequence Diagram — A Single Inference Request Through a Scale-From-Zero KServe Endpoint

```
Client         Ingress/Gateway    Knative Activator    Deployment (0 replicas)    New Pod (cold start)
  |                  |                    |                      |                        |
  |--POST /predict-->|                    |                      |                        |
  |                  |--forward---------->|                      |                        |
  |                  |                    |--check: replicas=0?  |                        |
  |                  |                    |--buffer request       |                        |
  |                  |                    |--trigger scale-up---->|                        |
  |                  |                    |                      |--schedule Pod---------->|
  |                  |                    |                      |                          |--pull image (if not cached)
  |                  |                    |                      |                          |--load model weights into GPU mem
  |                  |                    |                      |                          |--readinessProbe passes
  |                  |                    |<--Pod Ready-----------------------------------|
  |                  |                    |--release buffered request-------------------->|
  |                  |                    |                      |                          |--run inference
  |                  |                    |<-----------------response----------------------|
  |                  |<--response---------|                      |                          |
  |<--response-------|                    |                      |                          |
  |                  |                    |                      |                          |
  |     [Total latency for this FIRST request = cold-start tax: image pull (if cold) +      |
  |      model load time + inference time -- for a 7B+ LLM this is commonly seconds to       |
  |      low tens of seconds, which is why minReplicas=0 is a COST decision, not a free       |
  |      lunch, and why latency-SLA'd endpoints keep minReplicas >= 1.]                        |
```

---

## 6. Decision Tree — Kubernetes vs. ECS vs. Serverless for a Given ML/LLM Workload

```
                          START: "How should I deploy this model/workload?"
                                            |
                                            v
                     +---------------------------------------------+
                     | Does it need a GPU (deep learning / LLM       |
                     | inference above a few hundred M params)?      |
                     +---------------------+-------------------------+
                            YES            |            NO
                             |             |             |
                             v             |             v
              +--------------------------+ |  +----------------------------------+
              | Is traffic sustained /     | |  | Is traffic genuinely spiky/low-   |
              | predictable (even if        | |  | average, with tolerable cold      |
              | variable through the day)?  | |  | starts (sub-second to a few sec)? |
              +------------+---------------+ |  +---------------+--------------------+
                    YES    |    NO            |         YES      |        NO
                     |     |     |            |          |       |         |
                     v     |     v            |          v       |         v
        +-----------------+  +--------------------+  +--------+  |  +----------------------+
        | KUBERNETES        |  | Consider serverless |  |SERVER-|  |  Is it AWS-only, team  |
        | (EKS/GKE/AKS)      |  | GPU (Cloud Run GPU,  |  | LESS  |  |  wants less ops than   |
        | -- mature GPU       |  | SageMaker Serverless)|  |(Lambda|  |  K8s, moderate steady  |
        | scheduling,         |  | ONLY if cold starts   |  |/Cloud |  |  traffic?              |
        | KServe/llm-d        |  | truly tolerable for    |  | Run)  |  +----------+-------------+
        | ecosystem, multi-   |  | your model size --      |  +-------+       YES  |   NO
        | GPU sharing (MIG/   |  | otherwise fall back      |                  |    |    |
        | time-slicing)       |  | to Kubernetes with       |                  v    |    v
        +-----------------+  | minReplicas>=1 (warm)     |            +----------+ | +-----------+
                              +--------------------------+            | ECS/       | | KUBERNETES  |
                                                                        | Fargate    | | (need porta-|
                                                                        +----------+ | bility/rich  |
                                                                                     | ML ecosystem)|
                                                                                     +-------------+

  Cross-cutting overrides (apply regardless of the path above):
   - Need multi-cloud portability or on-prem/hybrid? --> bias toward Kubernetes.
   - Small team, no dedicated platform engineers, single simple model? --> bias toward
     serverless or ECS even if the tree above says Kubernetes; the "right" tool also has to
     be a tool your team can actually operate safely.
   - Already running a K8s platform for other workloads? --> marginal cost of adding one more
     model to the same cluster is usually lower than standing up a second orchestration
     paradigm just for that model -- bias toward Kubernetes for consistency.
```

---

## 7. Decision Tree — HPA vs. VPA vs. KEDA (and how they combine)

```
                    START: "What kind of scaling problem do I have?"
                                        |
                                        v
                +----------------------------------------------+
                | Do I need MORE/FEWER REPLICAS (horizontal),    |
                | or BETTER-SIZED requests/limits (vertical)?    |
                +---------------------+--------------------------+
                   HORIZONTAL          |          VERTICAL
                       |               |               |
                       v               |               v
        +-----------------------+     |     +---------------------------+
        | Is the trigger a queue/  |     |     | Use VPA in "Auto" or       |
        | event source (Kafka lag, |     |     | "Recreate" mode to right-  |
        | SQS depth, cron), or do   |     |     | size CPU/memory requests   |
        | I need scale-to-zero?     |     |     | based on historical usage. |
        +------------+--------------+     |     | (Do NOT also run HPA on    |
              YES     |     NO            |     |  the same metric for the   |
               |      |      |            |     |  same workload -- pick one |
               v      |      v            |     |  axis. If you need both    |
        +----------+  |  +----------------+     |  scaling axes, run VPA in   |
        | KEDA        |  | Is the true       |     |  recommendation-only mode  |
        | ScaledObject|  | bottleneck signal  |     |  and drive HPA off a       |
        | (or          |  | CPU, or something  |     |  DIFFERENT metric.)        |
        |  ScaledJob   |  | else (GPU util,    |     +---------------------------+
        |  for batch)  |  | requests-in-flight,|
        +----------+  |  | custom app metric)?|
                       |  +---------+----------+
                       |    CPU-OK       OTHER
                       |      |            |
                       |      v            v
                       |  +--------+  +--------------------------+
                       |  | HPA     |  | HPA with Custom/External   |
                       |  | (type:  |  | metric (Prometheus Adapter |
                       |  | Resource|  | exposing GPU util, queue    |
                       |  | on cpu) |  | depth, tokens/sec, etc.)    |
                       |  +--------+  +--------------------------+
                       |
                       v
              (KEDA creates and manages a standard HPA object internally
               once replicas > 0 -- it is not a replacement for HPA,
               it is a richer metrics source AND the scale-to-zero layer
               sitting in front of it.)
```

---

## 8. Decision Tree — Deployment vs. StatefulSet vs. KServe InferenceService

```
        START: "What Kubernetes primitive should own this ML/LLM workload?"
                                  |
                                  v
        +---------------------------------------------+
        | Does "which specific replica" matter -- stable |
        | network identity, stable per-replica storage,   |
        | or ordered startup (e.g., sharded vector DB,     |
        | distributed inference ranks, leader-elected      |
        | coordinator)?                                     |
        +---------------------+---------------------------+
                YES            |            NO
                 |             |             |
                 v             |             v
        +------------------+  |  +--------------------------------+
        | Can this be        |  |  | Is this "model serving" in the  |
        | offloaded to a      |  |  | sense of: needs standardized     |
        | MANAGED service      |  |  | multi-framework runtime, canary  |
        | instead (RDS,        |  |  | traffic split, concurrency-based |
        | managed Redis,       |  |  | autoscaling / scale-to-zero?     |
        | hosted vector DB)?   |  |  +---------------+-------------------+
        +--------+-----------+  |         YES         |         NO
             YES  |    NO       |          |           |          |
              |   |    |        |          v           |          v
              v   |    v        |  +----------------+  |  +------------------+
        +--------+ | +--------+ |  | KServe           |  |  | Plain Deployment  |
        | Use the | | | State-  | |  | InferenceService  |  |  | + Service (+ HPA  |
        | managed | | | fulSet   | |  | (or LLMInference- |  |  |  if needed)       |
        | service, | | | + head-  | |  | Service for        |  |  |  -- simplest       |
        | keep your| | | less Svc | |  | generative/LLM      |  |  |  option, right for |
        | own K8s   | | | + PVC     | |  | concerns)           |  |  |  a single simple   |
        | footprint | | | volume-   | |  +----------------+  |  |  model with low     |
        | stateless | | | ClaimTem- | |                        |  |  change frequency  |
        +--------+ | | plates    | |                        |  +------------------+
                    | +----------+ |
                    |               |
                    v               v
             (This is the       (Note: even when using a
              senior-engineer    StatefulSet for the DB layer,
              DEFAULT -- avoid   the model-serving layer in front
              self-hosting        of it is very often still a
              stateful infra      Deployment or an InferenceService --
              unless there's a    the two decisions are independent.)
              specific reason a
              managed alternative
              doesn't fit: cost
              control, data
              residency, feature
              gaps in the managed
              offering, etc.)
```

---

## 9. Decision Tree — Rollout Strategy for a New Model Version

```
        START: "How do I ship this new model/code version to production?"
                                    |
                                    v
        +-------------------------------------------------+
        | Is the blast radius of a bad version genuinely     |
        | high (large user base, high-stakes decisions e.g.   |
        | fraud/credit/safety, or a brand-new inference        |
        | ENGINE rather than just a new checkpoint)?           |
        +---------------------+-----------------------------+
                YES            |            NO
                 |             |             |
                 v             |             v
    +--------------------+    |    +--------------------------------+
    | Can you afford 2x    |    |    | Is this served via KServe        |
    | compute for a bounded |    |    | InferenceService already?        |
    | overlap window?       |    |    +---------------+-------------------+
    +----------+-----------+    |          YES          |          NO
        YES    |    NO          |           |           |           |
         |     |     |          |           v           |           v
         v     |     v          |  +------------------+ |  +----------------------+
    +--------+ | +------------+ |  | Use native         | |  | Use plain Deployment  |
    | BLUE-   | | | CANARY      | |  | canaryTrafficPercent| |  | rolling update         |
    | GREEN    | | | (Argo       | |  | field -- simplest,  | |  | (maxSurge/max-         |
    | (Argo    | | | Rollouts,   | |  | usually sufficient   | |  | Unavailable) -- fine   |
    | Rollouts  | | | AnalysisTem-| |  | for pure model-      | |  | for low-risk, easily-  |
    | active/    | | | plate gates | |  | serving canaries.     | |  | reversible changes,    |
    | preview    | | | on Prometheus| |                        | |  | but note: no traffic-  |
    | Service)   | | | metrics)    | |                        | |  | percentage control and |
    +--------+ | +------------+ |                        | |  | no automated model-     |
                | (if compute      |                        | |  | quality gate -- pair    |
                |  budget for       |                        | |  | with careful post-      |
                |  2x isn't there,  |                        | |  | deploy monitoring.      |
                |  canary is the    |                        | +----------------------+
                |  fallback even for|
                |  high-blast-radius|
                |  changes -- just  |
                |  start the first  |
                |  step smaller,     |
                |  e.g. 1-5%, and    |
                |  hold longer)      |
                +------------------+
```

---

## 10. Notes on Using These Diagrams

- The full-stack diagram (Section 1) is deliberately maximal — a real system typically implements
  a subset. Use it as a checklist of "things that exist in this space" rather than a blueprint
  every cluster must match.
- The sequence diagrams (Sections 4-5) are the most interview-relevant artifacts in this file:
  being able to draw the canary-rollout-with-auto-rollback sequence from memory, and to explain
  the cold-start sequence's latency components, are both extremely common ways senior MLOps
  interviews probe for real operational understanding versus memorized vocabulary.
- The decision trees (Sections 6-9) intentionally end most leaves with a *reasoned* recommendation
  plus a caveat, not a bare answer — in a live interview, walking the interviewer down the tree
  out loud (stating the criterion at each branch before giving your answer) demonstrates the
  judgment that "always use Kubernetes" answers fail to demonstrate.
- For citations backing every claim embedded in these diagrams (e.g., HPA reconcile intervals,
  KServe's canary field, Argo Rollouts' AnalysisTemplate), see `references.md` in this same folder.
