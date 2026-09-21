# Module 06 — Kubernetes for ML and LLM Systems

> "Kubernetes doesn't run your model. Kubernetes runs the *promise* that your model will keep running." — a useful way to reframe every section below.

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Explain what a Pod, Deployment, Service, and Ingress each solve, and wire them together into a working HTTP-facing ML inference service.
2. Write correct CPU/memory `requests`/`limits` and GPU resource declarations for training and inference workloads, and explain why GPUs behave differently from CPU/memory in the Kubernetes scheduler.
3. Choose between HPA, VPA, and KEDA for a given ML/LLM autoscaling problem — including scaling on GPU utilization and on queue depth — and write the corresponding manifests.
4. Explain when a StatefulSet (rather than a Deployment) is the correct primitive for a stateful ML component (sharded vector DB, distributed inference cluster, leader-elected model server).
5. Deploy a model to Kubernetes using KServe's `InferenceService`, and explain how Kubeflow, KServe, and the newer `LLMInferenceService` / llm-d ecosystem relate to one another as of mid-2026.
6. Package a set of ML manifests into a Helm chart with environment-specific `values.yaml` overrides.
7. Implement a rolling update, a canary rollout, and a blue-green deployment for a model-serving workload, and explain the risk profile and rollback story of each.
8. Produce a defensible, criteria-based answer to "Kubernetes vs. ECS vs. Serverless for ML/LLM workloads" rather than a reflexive "always use K8s."

### Prerequisites

- Comfortable with Python and have trained/served at least one ML model (from earlier modules in this course, or equivalent experience).
- Basic Docker literacy: you can write a `Dockerfile`, build an image, and run a container. (If not, treat Module 05 or equivalent as a hard prerequisite — this module assumes containerization is a solved problem for you and focuses on *orchestrating* those containers.)
- A running Kubernetes cluster to experiment against. For learning, any of the following work fine: `kind`, `minikube`, `k3d`, or a managed cluster (EKS/GKE/AKS) with a small node pool. GPU examples require either a cloud GPU node pool or can be read without execution — the manifests are correct either way.
- `kubectl` and `helm` CLIs installed.

### Key Terminology

| Term | One-line definition |
|---|---|
| **Pod** | The smallest deployable unit in Kubernetes — one or more containers that share network namespace and storage, scheduled together onto the same node. |
| **Deployment** | A controller that manages a set of identical, stateless Pod replicas via an underlying ReplicaSet, and knows how to roll updates out and back. |
| **Service** | A stable virtual IP/DNS name that load-balances traffic across the set of Pods matching a label selector, decoupling clients from Pod churn. |
| **Ingress** | A cluster-level rule set (implemented by an Ingress controller) that routes external HTTP(S) traffic to internal Services based on host/path. |
| **Requests / Limits** | The minimum a container is guaranteed (`requests`, used for scheduling) and the maximum it may consume (`limits`, enforced at runtime) for a resource such as CPU, memory, or GPU. |
| **Device Plugin** | A kubelet extension mechanism that lets vendors (NVIDIA, AMD, Intel) advertise specialized hardware (GPUs) as schedulable resources like `nvidia.com/gpu`. |
| **HPA** | Horizontal Pod Autoscaler — adds/removes Pod replicas based on observed metrics (CPU, memory, custom, or external). |
| **VPA** | Vertical Pod Autoscaler — recommends/applies changes to a Pod's `requests`/`limits` rather than its replica count. |
| **KEDA** | Kubernetes Event-Driven Autoscaling — an HPA-compatible controller that can scale (including to/from zero) on external event sources like queue depth, Kafka lag, or Prometheus queries. |
| **StatefulSet** | A controller for Pods that need stable network identity, stable storage, and ordered start/stop — used for anything where "which specific replica" matters. |
| **Kubeflow** | An open-source, Kubernetes-native ML/AI platform providing pipelines, notebooks, hyperparameter tuning (Katib), distributed training, and serving (via KServe) as composable components. |
| **KServe** | A Kubernetes CRD-based model-serving layer (`InferenceService`) providing autoscaling (including scale-to-zero), canary rollout, and multi-framework runtime support out of the box. |
| **Helm** | The de-facto Kubernetes package manager: templated YAML ("charts") plus a release/values model for repeatable, parameterized deployments. |
| **Canary deployment** | Gradually shifting a small, increasing percentage of production traffic to a new version while watching metrics, before full cutover. |
| **Blue-green deployment** | Running two complete environments (old = blue, new = green) and switching traffic atomically from one to the other, keeping the old one warm for instant rollback. |

---

## 2. Why This Topic Matters — Where It Fits in the MLOps/LLMOps Lifecycle

Every earlier module in this course has been about getting a model or an LLM pipeline to a *correct, reproducible, containerized* state: trained, versioned, tracked, packaged. Kubernetes is where that artifact meets the real world — the layer that keeps it running, scales it under load, recovers it after failure, and lets you change it without downtime.

Concretely, in the lifecycle:

```
 Data →  Training →  Model Registry →  Container Image →  [ THIS MODULE ]  →  Live traffic
                                              (Docker)          Kubernetes           |
                                                                                     v
                                                              Observability (Module 07/08),
                                                              CI/CD (Module 09), governance...
```

A model sitting in a registry, or a container image sitting in a registry, generates zero business value. Something has to:

- **Place** the container on a machine with enough CPU/RAM/GPU (scheduling).
- **Keep it alive** — restart it if it crashes, replace the node if the node dies (self-healing).
- **Route traffic to it** without hard-coding IPs, and without downtime when it's replaced (Service discovery/load balancing).
- **Scale it** as load changes — including scaling GPU-hungry LLM inference pods judiciously, because GPUs are the single most expensive line item in most LLM platforms' cloud bills.
- **Roll out new versions safely** — a bad new model version or a bad new prompt-template/RAG code path should be catchable and reversible before it eats the whole fleet.

For classical ML (a scikit-learn or XGBoost model behind a REST endpoint), Kubernetes is *one good option among several* (see the comparison in §3.8). For LLM systems specifically, Kubernetes has become close to the industry default for anything beyond "call a hosted API," because:

1. **GPU scheduling is a first-class, solved problem in K8s** (via device plugins and the GPU Operator), whereas most serverless platforms either don't support GPUs at all or support them in limited, expensive, cold-start-heavy ways.
2. **The open-source LLM-serving ecosystem is Kubernetes-native.** vLLM, Ray Serve, KServe, Triton, and the new llm-d disaggregated-inference framework are all designed assuming a Kubernetes control plane underneath them.
3. **Multi-tenant GPU sharing** (time-slicing, MIG, bin-packing many small models onto one GPU) needs a scheduler with enough expressiveness to reason about topology, affinity, and custom resources — which is exactly what the K8s scheduler + extended resources + operators give you.
4. **Autoscaling on a proxy for demand other than CPU** — queue depth, tokens/sec, requests-in-flight — is essential for LLM workloads, where CPU utilization is a poor proxy for load (a GPU can be pegged at 95% while the host CPU sits at 5%). HPA-with-custom-metrics and KEDA exist specifically to close this gap.

By the time an MLOps engineer is asked in a senior interview "how would you serve a 70B-parameter model to 10,000 requests/minute with a p99 latency SLA," the expected answer lives almost entirely in the vocabulary this module teaches: node pools, device plugins, InferenceService autoscaling policies, queue-depth-based KEDA scalers, and progressive rollout strategies.

---

## 3. Main Concepts

### 3.1 Pods, Deployments, Services, and Ingress — The Load-Bearing Foundations

#### Theory

**Pod.** Kubernetes never schedules a "container" directly — it schedules a Pod, which wraps one or more containers that are guaranteed to land on the same node and share a network namespace (same `localhost`, same IP) and optionally storage volumes. For ML serving, the vast majority of Pods are single-container (the inference server), but multi-container Pods matter for two recurring ML patterns:

- **Sidecar pattern**: a model server container plus a sidecar that tails logs to a collector, or performs mTLS termination (service mesh), or runs a model-download `initContainer` that pulls large weights from object storage before the main container starts.
- **Ambassador/adapter pattern**: a sidecar that translates a legacy prediction protocol into the one your gateway expects.

Pods are *ephemeral and disposable by design*. This is the single most important mental model shift for engineers coming from VM-based ops: you never SSH into a Pod expecting it to still be there tomorrow, you never store state on its local disk expecting it to survive a restart, and you never hard-code its IP anywhere. Pods die (rescheduled, evicted, node drained) and Kubernetes creates *new* Pods with *new* IPs to replace them. This is precisely why Deployments and Services exist.

**Deployment.** A Deployment is a declarative statement: "I want N replicas of this Pod template running, and I want you (the controller) to reconcile reality toward that desired state forever, and to know how to transition safely between old and new versions of the template." Under the hood, a Deployment doesn't manage Pods directly — it manages a **ReplicaSet**, and every time you change the Pod template (e.g., bump the image tag), the Deployment controller creates a *new* ReplicaSet and orchestrates a rollout between old and new ReplicaSets according to a `strategy` (rolling update by default). This is why `kubectl rollout undo` works: the old ReplicaSet (with 0 replicas, scaled down) is still sitting there as a rollback target.

Deployments are the right primitive whenever **all replicas are interchangeable** — any replica can serve any request, none has special identity, and losing any one of them and replacing it with a fresh one changes nothing observable to the client. This is true for the overwhelming majority of stateless model-serving pods, batch-inference workers, and API gateways.

**Service.** DNS-based, label-selector-based traffic routing. A Service doesn't point at specific Pods; it continuously watches for Pods matching a `selector` (e.g., `app: fraud-model, version: v3`) and load-balances across whichever ones currently exist and are marked `Ready`. This is what makes rolling updates possible without client-visible errors: as new Pods become Ready and old ones are terminated, the Service's endpoint list updates automatically and traffic simply shifts.

| Service type | What it does | Typical ML use |
|---|---|---|
| `ClusterIP` (default) | Internal-only virtual IP, reachable from inside the cluster | Model server reachable only by an internal gateway/feature-store/orchestrator |
| `NodePort` | Exposes a static port on every node's IP | Rare in production; mostly for local dev/demo clusters |
| `LoadBalancer` | Provisions a cloud load balancer (ELB/GLB/ALB) | Public-facing inference endpoint, one per exposed service (costly at scale — hence Ingress) |
| `Headless` (`clusterIP: None`) | No load-balancing; DNS resolves directly to each Pod's IP | Stateful ML clusters (sharded vector DB, distributed inference peers) that need to address a *specific* replica — pairs with StatefulSets |

**Ingress.** Provisioning one cloud LoadBalancer per model is expensive and doesn't scale past a handful of services. Ingress is a single entry point (usually one or two cloud load balancers total) with L7 routing rules — "path `/v1/models/fraud-model` → Service `fraud-model-svc`; path `/v1/models/churn-model` → Service `churn-model-svc`." The actual routing logic is implemented by an **Ingress controller** (NGINX Ingress Controller, AWS Load Balancer Controller, Traefik, Istio Gateway, etc.) that you install separately — the `Ingress` object itself is only a spec, inert without a controller watching it. For LLM-serving gateways specifically, many production stacks now put a purpose-built AI gateway (e.g., an OpenAI-API-compatible router doing model selection, rate limiting, and cost tracking) behind the Ingress, rather than routing directly to a single model's Service.

**When NOT to reach for raw Deployment+Service+Ingress:** if what you actually need is model-serving-specific behavior — scale-to-zero when idle, canary traffic splitting, per-model autoscaling on inference concurrency, standardized multi-framework runtimes — hand-rolling all of that on top of bare Deployments is exactly the undifferentiated heavy lifting that KServe (§3.5) exists to remove. Treat §3.1's primitives as the foundation everything else is built from, not as the final answer for serving.

#### Architecture

```
                                   Internet
                                      |
                                      v
                          +---------------------+
                          |  Ingress Controller  |   (NGINX / ALB / Istio Gateway)
                          |  rules: host/path     |
                          +----------+-----------+
                                      |
                     -----------------+-----------------
                    |                                    |
                    v                                    v
         +--------------------+              +-----------------------+
         | Service: model-a   |              | Service: model-b      |
         | type: ClusterIP    |              | type: ClusterIP        |
         | selector: app=a    |              | selector: app=b        |
         +----------+---------+              +-----------+------------+
                    |                                     |
        ------------+------------                         |
       |            |            |                        |
       v            v            v                        v
  +--------+   +--------+   +--------+              +--------+
  | Pod a1 |   | Pod a2 |   | Pod a3 |              | Pod b1 |
  | (Deploy-  managed replicas of one              | Deploy- |
  |  ment)      Deployment "model-a")               ment)   |
  +--------+   +--------+   +--------+              +--------+
```

#### Examples

**Beginner** — a single-container FastAPI model server, Deployment + ClusterIP Service.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: churn-model
  labels:
    app: churn-model
spec:
  replicas: 3
  selector:
    matchLabels:
      app: churn-model
  template:
    metadata:
      labels:
        app: churn-model
        version: v1
    spec:
      containers:
        - name: churn-model
          image: registry.example.com/churn-model:1.4.2
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet: { path: /health, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /health, port: 8080 }
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits:   { cpu: "1",    memory: "1Gi" }
---
apiVersion: v1
kind: Service
metadata:
  name: churn-model-svc
spec:
  selector:
    app: churn-model
  ports:
    - port: 80
      targetPort: 8080
```

**Intermediate** — adding Ingress with path-based routing for two models behind one domain.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ml-gateway
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: models.internal.example.com
      http:
        paths:
          - path: /churn(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service: { name: churn-model-svc, port: { number: 80 } }
          - path: /fraud(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service: { name: fraud-model-svc, port: { number: 80 } }
```

**Production-grade** — the beginner Deployment above, hardened: explicit `terminationGracePeriodSeconds` for in-flight requests, `preStop` hook to drain connections, `podAntiAffinity` to spread replicas across nodes/zones for availability, and a `PodDisruptionBudget` so voluntary disruptions (node drains, cluster upgrades) never take down more than one replica at a time.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: churn-model
spec:
  replicas: 6
  selector: { matchLabels: { app: churn-model } }
  template:
    metadata: { labels: { app: churn-model, version: v1 } }
    spec:
      terminationGracePeriodSeconds: 30
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector: { matchLabels: { app: churn-model } }
                topologyKey: topology.kubernetes.io/zone
      containers:
        - name: churn-model
          image: registry.example.com/churn-model:1.4.2
          lifecycle:
            preStop:
              exec: { command: ["sh", "-c", "sleep 10"] }   # let in-flight requests finish before SIGTERM
          readinessProbe:
            httpGet: { path: /health, port: 8080 }
          resources:
            requests: { cpu: "500m", memory: "1Gi" }
            limits:   { cpu: "2",    memory: "2Gi" }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: churn-model-pdb
spec:
  minAvailable: 5          # out of 6 — cluster upgrades can only evict 1 at a time
  selector: { matchLabels: { app: churn-model } }
```

---

### 3.2 Resource Requests/Limits and GPU Scheduling

#### Theory

Every container declares `resources.requests` (what it's *guaranteed*, used by the scheduler to decide which node has room) and `resources.limits` (the hard ceiling, enforced by the kubelet/container runtime). Getting this wrong is one of the most common causes of both wasted spend and mysterious production incidents:

- **CPU** is a *compressible* resource: exceeding your CPU limit gets you throttled (slower), not killed.
- **Memory** is *incompressible*: exceeding your memory limit gets your container **OOMKilled** — an abrupt SIGKILL, not a graceful shutdown. This matters enormously for ML serving, where a single large batch or a memory-hungry tokenizer call can spike RSS momentarily; under-provisioned memory limits are a top cause of "random" pod restarts in production inference services.
- Requests also determine each Pod's **QoS class**: `Guaranteed` (requests == limits for every resource — highest eviction priority, i.e., evicted last under node pressure), `Burstable` (requests < limits), or `BestEffort` (no requests/limits set at all — evicted first). For latency-sensitive production inference, prefer `Guaranteed` or at least a tight `Burstable` band; never ship `BestEffort` to production.

**GPUs are fundamentally different from CPU/memory in Kubernetes**, and this trips up almost every engineer new to ML-on-K8s:

1. GPUs are exposed via the **device plugin** framework, not native scheduler resource types. The NVIDIA device plugin runs as a DaemonSet, discovers GPUs on each node, and advertises them to the kubelet as an **extended resource**, `nvidia.com/gpu`.
2. Extended resources like `nvidia.com/gpu` **must be specified only in `limits`**, never in `requests` alone (if you set a limit without a request, Kubernetes defaults the request to match the limit automatically) — and critically, **fractional values are not allowed** by the stock device plugin (`nvidia.com/gpu: 1`, not `0.3`). A GPU, unlike CPU, is not natively divisible by Kubernetes' own resource model.
3. This "one whole GPU or nothing" default is exactly the inefficiency problem that the **NVIDIA GPU Operator's time-slicing** feature and, at the hardware level, **NVIDIA MIG** (Multi-Instance GPU, on A100/H100-class cards) solve — both let you oversubscribe or physically partition a GPU across multiple Pods, which is essential for cost efficiency when a single inference replica doesn't saturate an entire GPU (very common for smaller models, or for LLM inference with modest concurrency). Time-slicing shares a whole GPU's compute in a round-robin fashion (no memory isolation — Pods can still OOM each other on VRAM); MIG gives hardware-level partitioning with real isolation, at the cost of fixed partition shapes.
4. The **NVIDIA GPU Operator** exists because "GPU support" is not just the device plugin — it's the device plugin *plus* the correct driver version, the NVIDIA Container Toolkit, node feature discovery (labeling nodes with GPU model/count), and DCGM-based GPU telemetry, all of which must be kept in lockstep. The Operator packages this entire stack as Kubernetes-native custom resources so a platform team manages "GPU nodes" as one declarative unit instead of hand-provisioning drivers via node bootstrap scripts.

**When to use GPU nodes at all vs. CPU-only**: not every model needs a GPU. Classical ML (scikit-learn, XGBoost, small tree ensembles) almost always runs cheaper and just as fast on CPU. Reserve GPU node pools for models where inference-time compute genuinely benefits from parallel matrix math at scale — deep learning models, and essentially all LLM inference above a few hundred million parameters. Mixing this up (GPU nodes for a logistic regression, CPU nodes for a 13B-parameter LLM) is a surprisingly common and expensive mistake in real platform teams.

#### Architecture

```
                         Node (GPU-enabled)
   +---------------------------------------------------------------+
   |  kubelet                                                       |
   |     ^                                                          |
   |     | advertises "nvidia.com/gpu: 4"                            |
   |     |                                                          |
   |  +--+-----------------------+     +---------------------------+ |
   |  | NVIDIA device plugin     |     | GPU Operator-managed:      | |
   |  | (DaemonSet)              |<--->| - NVIDIA driver             | |
   |  +--------------------------+     | - Container Toolkit         | |
   |                                    | - Node Feature Discovery    | |
   |                                    | - DCGM exporter (metrics)   | |
   |                                    +---------------------------+ |
   |                                                                  |
   |  +--------------+  +--------------+  +--------------+           |
   |  |  Pod (LLM-1)  |  |  Pod (LLM-2)  |  |  Pod (LLM-3)  |          |
   |  |  limits:      |  |  limits:      |  |  limits:      |          |
   |  |  gpu: 1       |  |  gpu: 1       |  |  gpu: 2       |          |
   |  +--------------+  +--------------+  +--------------+           |
   |         GPU0            GPU1           GPU2 + GPU3               |
   +---------------------------------------------------------------+

   With time-slicing enabled, N pods can share ONE physical GPU (no
   requests>1 needed on the node side — the plugin just reports more
   "virtual" nvidia.com/gpu units than physical GPUs exist).
```

#### Examples

**Beginner** — plain CPU resource management for a scikit-learn model.

```yaml
resources:
  requests: { cpu: "500m", memory: "512Mi" }
  limits:   { cpu: "1",    memory: "1Gi" }
```

**Intermediate** — a single-GPU inference Pod (e.g., a 7B LLM served via vLLM or TGI):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llama-7b-server
spec:
  replicas: 2
  selector: { matchLabels: { app: llama-7b } }
  template:
    metadata: { labels: { app: llama-7b } }
    spec:
      nodeSelector:
        nvidia.com/gpu.product: "NVIDIA-A10G"   # label set by Node Feature Discovery
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Exists"
          effect: "NoSchedule"
      containers:
        - name: vllm-server
          image: vllm/vllm-openai:latest
          args: ["--model", "meta-llama/Llama-2-7b-chat-hf"]
          resources:
            requests: { cpu: "4", memory: "16Gi" }
            limits:
              cpu: "8"
              memory: "24Gi"
              nvidia.com/gpu: 1
          ports: [{ containerPort: 8000 }]
```

**Production-grade** — GPU time-slicing config (applied via the GPU Operator's `ClusterPolicy`/ConfigMap) letting 4 lightweight model-serving Pods share one physical GPU, for models whose inference doesn't need a full accelerator:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  a10g-4-slices: |
    version: v1
    sharing:
      timeSlicing:
        resources:
          - name: nvidia.com/gpu
            replicas: 4        # kubelet will now advertise 4x "nvidia.com/gpu" per physical GPU
---
# Referenced from the GPU Operator's ClusterPolicy CR (devicePlugin.config.name),
# then a node is opted in via a label:
#   kubectl label node <gpu-node> nvidia.com/device-plugin.config=a10g-4-slices
```

With this in place, four small-model Pods can each request `nvidia.com/gpu: 1` and land on the *same* physical GPU, round-robin sharing compute — appropriate for low-QPS, latency-tolerant endpoints, but the wrong choice for a latency-SLA'd, high-throughput LLM endpoint (where you want the whole GPU, or MIG-level isolation, not time-sliced contention).

#### Comparison: GPU sharing strategies

| Strategy | Isolation | Granularity | Best for |
|---|---|---|---|
| Whole-GPU assignment (default) | Full (dedicated GPU) | 1 GPU per unit | Latency-SLA production LLM inference, training |
| Time-slicing (GPU Operator) | None (compute round-robins; VRAM shared, can OOM) | Configurable virtual replicas | Dev/test, low-QPS endpoints, bin-packing many small models |
| MIG (A100/H100+) | Hardware-level partitions | Fixed shapes (e.g., 7x 1g.5gb) | Multi-tenant production with real isolation needs, mixed workload sizes |

---

### 3.3 Autoscaling: HPA, VPA, and KEDA

#### Theory

**HPA (Horizontal Pod Autoscaler)** watches a metric and adjusts **replica count**. Out of the box it supports CPU and memory (`autoscaling/v2` `type: Resource`), but the real power for ML workloads is `type: Pods`, `type: Object`, or `type: External` — letting it scale on *anything exposed through the Custom/External Metrics API*, most commonly backed by Prometheus Adapter. This is what lets you scale an inference Deployment on **requests-in-flight**, **p95 latency**, or **GPU utilization** rather than raw CPU, which — as noted above — is frequently a misleading signal for GPU-bound inference.

**VPA (Vertical Pod Autoscaler)** does the opposite axis: instead of more replicas, it recommends (or, in `Auto`/`Recreate` mode, applies) better `requests`/`limits` values based on observed historical usage, and works via three components — a **Recommender** (watches usage, computes suggestions), an **Updater** (evicts Pods that deviate too far from the recommendation so they get recreated with new values), and an **admission controller webhook** (rewrites Pod specs at creation time to the recommended values). VPA is invaluable for right-sizing workloads whose resource needs you don't know in advance (a new model with unknown memory footprint) — but it comes with an important, frequently-tested-in-interviews caveat: **HPA and VPA should not both actively manage the same resource metric on the same workload simultaneously.** If VPA is resizing CPU requests while HPA is also scaling replica count off CPU utilization, the two controllers can fight each other (VPA changes what "100% CPU" even means for the Pod, confusing HPA's target). The safe pattern is either VPA-only (fixed replica count, right-sized Pods) or HPA-on-a-different-metric (e.g., HPA on custom/external metric, VPA in recommendation-only mode for that same Deployment).

**KEDA (Kubernetes Event-Driven Autoscaling)** is, structurally, a metrics adapter plus a controller that implements the Custom/External Metrics API on HPA's behalf — meaning KEDA doesn't replace HPA, it feeds it. What KEDA adds that raw HPA-with-Prometheus-Adapter doesn't give you as easily:

- A huge, maintained catalog of **scalers** for real-world event sources: RabbitMQ queue length, Kafka consumer lag, AWS SQS queue depth, Redis list length, Prometheus queries, cron schedules, and dozens more.
- **Scale-to-zero.** Plain HPA cannot scale a Deployment to 0 replicas (its minimum is 1); KEDA can, by decoupling "is there any work at all" (its own lightweight polling loop) from "how many replicas do we need" (delegated to HPA once above zero). This is the mechanism behind cost-efficient batch-inference workers and low-traffic LLM endpoints that shouldn't burn GPU-hours when idle.
- A `ScaledJob` CRD for **queue-consumer batch workloads** (each unit of work spawns a Kubernetes Job rather than being consumed by a long-lived Pod) — a very natural fit for async LLM batch-inference pipelines (e.g., "drain this SQS queue of embedding jobs").

For LLM inference specifically, **queue-depth-based scaling via KEDA is often the most honest signal available**, because for many LLM-serving architectures the true bottleneck is "number of requests waiting for a GPU slot," which a message queue (or even a Prometheus gauge exposing in-flight-request count from the inference server itself, e.g., vLLM's metrics endpoint) represents far more faithfully than CPU ever will.

**When NOT to autoscale on CPU for LLM inference**: if your LLM server is GPU-bound (which it almost always is), CPU utilization will often sit low and flat regardless of load — the GPU is the saturated resource, not the CPU. Wiring HPA to CPU in this situation gives you an autoscaler that essentially never fires when it should. Scale on GPU utilization (via DCGM exporter → Prometheus → Prometheus Adapter → HPA External metric), on concurrent-requests/queue-depth (via KEDA), or ideally both.

#### Architecture

```
   +-------------------+        +----------------------+
   |  Prometheus        |<-------|  DCGM Exporter        |  (GPU util per pod)
   |  (scrapes metrics)  |        +----------------------+
   +---------+----------+
             |
             v
   +-------------------+       +--------------------------+
   | Prometheus Adapter |------>| Custom/External Metrics   |
   | (exposes metrics    |       | API (k8s aggregation      |
   |  via k8s API)       |       |  layer)                   |
   +-------------------+       +-------------+--------------+
                                              |
                          +-------------------+-------------------+
                          |                                       |
                          v                                       v
                 +-----------------+                    +------------------+
                 |      HPA         |<------ scales ---->|   Deployment      |
                 | (external metric:|                    | (LLM inference     |
                 |  gpu_util or      |                    |  server Pods)      |
                 |  queue_depth)     |                    +------------------+
                 +-----------------+
                          ^
                          |  (KEDA sits "in front of" HPA and can
                          |   also drive replicas to 0)
                 +-----------------+       +------------------+
                 |  KEDA ScaledObject|<----->|  RabbitMQ / SQS   |
                 |  (queue-length     |       |  queue depth      |
                 |   scaler)          |       +------------------+
                 +-----------------+
```

#### Examples

**Beginner** — HPA on CPU (fine for a CPU-bound, classical-ML REST endpoint):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: churn-model-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: churn-model
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 65 }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # avoid flapping: wait 5 min of sustained low load
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

**Intermediate** — HPA on a custom metric (GPU utilization) for an LLM inference Deployment:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llama-7b-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llama-7b-server
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: External
      external:
        metric:
          name: DCGM_FI_DEV_GPU_UTIL       # exposed via Prometheus Adapter
          selector:
            matchLabels: { deployment: llama-7b-server }
        target:
          type: AverageValue
          averageValue: "75"
```

**Production-grade** — KEDA `ScaledObject` scaling an async LLM batch-inference worker on RabbitMQ queue depth, including scale-to-zero:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: llm-batch-worker-scaler
spec:
  scaleTargetRef:
    name: llm-batch-worker            # Deployment of worker Pods consuming the queue
  minReplicaCount: 0                  # scale to zero when the queue is empty
  maxReplicaCount: 50
  cooldownPeriod: 120                 # seconds of empty queue before scaling to 0
  pollingInterval: 15
  triggers:
    - type: rabbitmq
      metadata:
        queueName: llm-inference-jobs
        mode: QueueLength
        value: "10"                   # target: 10 messages per replica
        host: amqp://rabbitmq.mq-namespace.svc.cluster.local:5672
      authenticationRef:
        name: rabbitmq-trigger-auth
```

Under the hood, KEDA creates and manages a standard HPA object for you once replicas are > 0 — it is not a replacement mechanism, it is a metrics source *and* the scale-to-zero controller sitting in front of HPA.

#### Comparison table: HPA vs. VPA vs. KEDA

| | HPA | VPA | KEDA |
|---|---|---|---|
| Scales | Replica count | Pod resource requests/limits | Replica count (incl. to/from 0) |
| Native metrics | CPU, memory | Historical CPU/memory usage | None natively — 60+ event-source scalers |
| Scale-to-zero | No (min 1) | N/A | Yes |
| Typical ML use | Stateless CPU-bound model APIs | Right-sizing unknown-footprint workloads | Queue/event-driven batch inference, bursty LLM workloads |
| Conflicts with | VPA (same metric, same workload) | HPA (same metric, same workload) | None (extends HPA, doesn't compete) |
| Requires extra install | No (built into K8s) | Yes (`kubernetes/autoscaler` VPA component) | Yes (KEDA controller) |

---

### 3.4 StatefulSets for Stateful ML Services

#### Theory

A Deployment's Pods are interchangeable and anonymous — Pod names are randomly suffixed (`churn-model-7d8f9c-x2kq9`), and if a Pod dies, its replacement gets a *new* name and a *new* IP, with no promise of reusing the same storage.

A **StatefulSet** exists for the class of workloads where "which specific instance" matters:

- **Stable, predictable network identity**: Pods are named `<statefulset-name>-0`, `-1`, `-2`, ... and each gets a stable DNS entry via a paired **headless Service** (`<pod-name>.<service-name>.<namespace>.svc.cluster.local`), which persists across restarts/rescheduling.
- **Stable storage per identity**: each ordinal gets its own `PersistentVolumeClaim`, created from a `volumeClaimTemplate`, and — critically — that specific PVC is *reattached* to the same-ordinal Pod if it's rescheduled, rather than a fresh empty volume being provisioned.
- **Ordered, graceful scale up/down**: Pod-0 is created and becomes Ready before Pod-1 is created (and the reverse on scale-down/deletion) — essential for clustered systems with leader-election or replication bootstrapping semantics (e.g., "node 0 must be the initial primary").

In ML/LLM systems, the recurring cases where this matters:

1. **Self-hosted vector databases** (Milvus, Weaviate, Qdrant in clustered mode, self-managed pgvector replicas) — sharded/replicated stateful stores are the textbook StatefulSet use case, identical to how you'd run Kafka, Elasticsearch, or Cassandra.
2. **Distributed inference / distributed training clusters with peer identity** — e.g., a multi-node tensor-parallel or pipeline-parallel LLM inference deployment (think: a large model sharded across 8 GPU nodes) where rank-0 needs a stable, discoverable address for the other ranks to connect to, and where losing "rank 3" and getting back some anonymously-named replacement Pod is not equivalent to actually restoring rank 3's role in the topology.
3. **Feature store / online-store backends** you self-host rather than consume as a managed service (e.g., a self-run Redis Cluster backing a feature store's online layer).
4. **Leader-elected model-serving coordinators**, where one replica acts as a router/coordinator and others are workers, and the coordinator role is tied to a specific stable identity rather than "whichever Pod happens to be running."

**When NOT to use a StatefulSet**: if you can offload the stateful part to a managed service (RDS, managed Redis, a hosted vector DB like Pinecone, a managed Kafka), do that instead and keep your own Kubernetes footprint stateless. StatefulSets are meaningfully more operationally demanding than Deployments — you own volume lifecycle, backup/restore, and often manual intervention for split-brain or stuck-terminating Pods. The senior-engineer default should be "avoid running your own stateful infrastructure on K8s unless there's a specific reason a managed alternative doesn't fit" — self-hosting is usually a deliberate cost/control tradeoff, not a default.

#### Architecture

```
              Headless Service: vectordb-svc (clusterIP: None)
                                |
        -----------------------+------------------------
       |                        |                        |
       v                        v                        v
  vectordb-0.vectordb-svc  vectordb-1.vectordb-svc   vectordb-2.vectordb-svc
  +----------------+       +----------------+        +----------------+
  |  Pod ordinal 0  |       |  Pod ordinal 1  |        |  Pod ordinal 2  |
  |  (shard leader)  |       |  (shard replica) |        |  (shard replica) |
  +-------+--------+       +--------+-------+        +--------+-------+
          |                          |                          |
          v                          v                          v
    PVC: data-vectordb-0       PVC: data-vectordb-1       PVC: data-vectordb-2
    (reattaches to ordinal 0    (reattaches to ordinal 1    (reattaches to ordinal 2
     on reschedule)              on reschedule)              on reschedule)

  Startup order: 0 -> 1 -> 2 (each Ready before next starts)
  Scale-down order: 2 -> 1 -> 0 (reverse)
```

#### Examples

**Beginner** — minimal StatefulSet skeleton (illustrative, not a production vector DB manifest):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vectordb-svc
spec:
  clusterIP: None
  selector: { app: vectordb }
  ports: [{ port: 19530 }]
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: vectordb
spec:
  serviceName: vectordb-svc
  replicas: 3
  selector: { matchLabels: { app: vectordb } }
  template:
    metadata: { labels: { app: vectordb } }
    spec:
      containers:
        - name: vectordb
          image: qdrant/qdrant:latest
          ports: [{ containerPort: 19530 }]
          volumeMounts:
            - name: data
              mountPath: /qdrant/storage
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 50Gi } }
```

**Intermediate** — adding a `podManagementPolicy: Parallel` override for cases where ordering doesn't matter but stable identity still does (faster scale-up for read replicas that don't need sequential bootstrap):

```yaml
spec:
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate: { partition: 0 }
```

**Production-grade** — combining a StatefulSet with anti-affinity (spread shards across nodes/zones) and a `PodDisruptionBudget`, mirroring how you'd actually run a self-managed clustered vector store or a Kafka-backed feature store safely through node maintenance:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: vectordb
spec:
  serviceName: vectordb-svc
  replicas: 3
  selector: { matchLabels: { app: vectordb } }
  template:
    metadata: { labels: { app: vectordb } }
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector: { matchLabels: { app: vectordb } }
              topologyKey: kubernetes.io/hostname   # never co-locate two shards on one node
      containers:
        - name: vectordb
          image: qdrant/qdrant:v1.11.0
          resources:
            requests: { cpu: "2", memory: "8Gi" }
            limits:   { cpu: "4", memory: "16Gi" }
          volumeMounts: [{ name: data, mountPath: /qdrant/storage }]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources: { requests: { storage: 200Gi } }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: vectordb-pdb }
spec:
  maxUnavailable: 1
  selector: { matchLabels: { app: vectordb } }
```

---

### 3.5 Kubeflow and KServe — Model Serving on Kubernetes

#### Theory

Everything in §3.1–§3.4 is *general-purpose Kubernetes* — none of it knows anything about "model" as a concept. You could build a full model-serving platform entirely out of Deployments/Services/HPA/StatefulSets, and many teams did, for years. But you'd be re-solving the same handful of problems every model-serving team solves: standardized multi-framework runtime containers, request/response transformation pipelines, canary rollout mechanics specific to model versions, scale-to-zero for idle models, and a consistent "predict" API contract. **Kubeflow** and **KServe** exist to give you that layer as reusable, declarative infrastructure instead of bespoke YAML per team.

**Kubeflow** is the broader platform: notebooks (interactive dev environments on the cluster), Pipelines (Argo-Workflows-based DAG orchestration for training/eval pipelines — conceptually parallel to Airflow but native to K8s and typically used for ML-specific stages), Katib (hyperparameter tuning as a Kubernetes-native CRD), a Trainer component for distributed training jobs, and KServe for serving — all composable, none mandatory in isolation. As of the Kubeflow 1.11 release (December 2025), the project explicitly rebranded itself the **"Kubeflow AI Reference Platform,"** shifting its center of gravity toward generative-AI workflows — distributed LLM fine-tuning and training pipelines — rather than the classical-ML-pipelines framing that dominated Kubeflow's earlier years. If you learned Kubeflow from 2023-2024-era material, treat this as a real shift, not cosmetic rebranding: expect first-class primitives and reference architectures for things like distributed fine-tuning jobs that didn't exist in that form previously.

**KServe** is the serving-specific piece, and the one you'll touch most often day-to-day, whether or not the rest of Kubeflow is in play (KServe is fully usable standalone). Its core abstraction is the **`InferenceService`** custom resource — a single YAML object that, when applied, causes KServe's controller to materialize the Deployments, Services, HPA-equivalents, and (depending on install mode) Knative-based scale-to-zero infrastructure needed to serve a model. What it gives you over hand-rolled Deployments:

- **Standardized multi-framework runtimes**: point at a model artifact in object storage and declare its framework (`sklearn`, `xgboost`, `pytorch`, Triton, a custom Hugging Face server, etc.) — KServe selects/provisions the matching serving runtime container, so you're not writing a Dockerfile+FastAPI wrapper for every model.
- **Predictor / Transformer / Explainer component model**: a request can flow through an optional pre/post-processing Transformer, into the Predictor, and results can optionally be explained by an Explainer component — all as separate, independently-scalable containers in the same logical InferenceService.
- **Built-in canary rollout support**: `InferenceService` has native fields for splitting traffic between a `default` and a `canary` model revision by percentage, without you writing custom Argo Rollouts logic for the model-serving case specifically (though for more elaborate progressive-delivery policies, teams often still layer Argo Rollouts on top — see §3.7).
- **Autoscaling on concurrency, including scale-to-zero**, when running on the Knative-based serverless mode — genuinely valuable for the "many low-traffic models" pattern common in enterprises serving dozens-to-hundreds of narrow models.
- **KServe reached CNCF Incubating status on September 29, 2025** — a meaningful production-maturity signal (graduated from Sandbox, vetted governance, broad multi-vendor adoption) worth knowing for any "build vs. adopt" conversation with a platform team lead.

**The 2025-2026 frontier: `LLMInferenceService` and llm-d.** The original `InferenceService` CRD was designed primarily around predictive-ML serving assumptions (one model, one forward pass, request/response). LLM inference has different concerns entirely — continuous batching across many in-flight generations, KV-cache management, multi-accelerator/tensor-parallel scheduling, and (at the frontier) *disaggregating* the prefill and decode phases of generation onto separate, differently-provisioned pools of GPUs, since prefill is compute-bound and decode is memory-bandwidth-bound and they don't want the same hardware profile. KServe addressed this with a **dedicated `LLMInferenceService` CRD**, purpose-built for generative-inference concerns, distinct from the classic `InferenceService`. And at a level beyond even that, **llm-d** — a Kubernetes-native, vLLM-based, disaggregated prefill/decode distributed inference framework from IBM Research, Red Hat, and Google Cloud — joined CNCF as a Sandbox project in March 2026, signaling that "serve one model behind one InferenceService" is no longer the ceiling of what production LLM-on-Kubernetes looks like at the highest-scale end. For this module's hands-on purposes, master `InferenceService` first (it is still the default, most mature, most broadly-applicable entry point); treat `LLMInferenceService` and llm-d as the documented forward-looking direction you should be aware of, especially if you're targeting a role serving frontier-scale open models.

**When to use KServe vs. hand-rolled Deployments**: use KServe once you have more than a couple of models, need consistent operational behavior across them (canary, autoscaling, standardized health/metrics), or are building a platform other teams will self-serve onto. For a single, simple, low-change-frequency model, a hand-rolled Deployment+Service can genuinely be simpler and easier to debug — don't reach for a CRD-based platform layer to solve a one-model problem.

#### Architecture

```
                         kubectl apply -f inferenceservice.yaml
                                       |
                                       v
                        +---------------------------+
                        |   KServe Controller         |
                        |   (watches InferenceService |
                        |    custom resources)         |
                        +--------------+--------------+
                                       |
                    materializes and manages:
                                       |
        +------------------------------+-----------------------------+
        |                              |                              |
        v                              v                              v
 +---------------+           +------------------+           +------------------+
 |  Transformer    |  --->    |    Predictor      |  --->    |    Explainer      |
 |  (pre/post-      |          |  (the actual model |          |  (optional: SHAP/  |
 |   processing)     |          |   runtime container|          |   LIME-style        |
 |                  |          |   e.g. Triton,      |          |   explanations)      |
 +---------------+           |   vLLM, sklearn-    |           +------------------+
                              |   server)            |
                              +---------+----------+
                                        |
                          traffic split: default vs canary %
                                        |
                     +------------------+-------------------+
                     v                                       v
            default revision (e.g. v3, 90%)         canary revision (v4, 10%)

    Underlying: Knative Serving (if serverless mode) handles scale-to-zero,
    request-based autoscaling, and revision management.
```

#### Sequence: a request through a canary-enabled InferenceService

```
Client         Ingress/Gateway     KServe routing      Predictor v3 (90%)   Predictor v4 (10%, canary)
  |                  |                    |                    |                       |
  |--- POST /predict-->|                   |                    |                       |
  |                  |--- forward -------->|                    |                       |
  |                  |                    |-- weighted route -->|  (90% of traffic)      |
  |                  |                    |-- weighted route ------------------------->|  (10%)
  |                  |                    |                    |--- response ---------->|
  |                  |<---------------------- response --------|                       |
  |<---- response ----|                    |                                            |
```

#### Examples

**Beginner** — a minimal `InferenceService` for a scikit-learn model stored in object storage:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: churn-model
spec:
  predictor:
    sklearn:
      storageUri: "gs://my-models/churn-model/v1"
      resources:
        requests: { cpu: "1", memory: "1Gi" }
        limits:   { cpu: "2", memory: "2Gi" }
```

**Intermediate** — a custom Hugging Face predictor with autoscaling on concurrency and a GPU:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sentiment-llm
  annotations:
    autoscaling.knative.dev/target: "5"    # target concurrent requests per replica
spec:
  predictor:
    minReplicas: 0          # scale-to-zero when idle
    maxReplicas: 6
    containers:
      - name: kserve-container
        image: registry.example.com/hf-sentiment-server:2.1
        resources:
          requests: { cpu: "2", memory: "8Gi" }
          limits:   { cpu: "4", memory: "16Gi", nvidia.com/gpu: 1 }
```

**Production-grade** — canary rollout of a new model revision, splitting 90/10 with an explicit rollback lever, plus a Transformer for request pre-processing:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: fraud-model
spec:
  predictor:
    canaryTrafficPercent: 10       # 10% of traffic to the new revision below
    minReplicas: 3
    maxReplicas: 15
    containers:
      - name: kserve-container
        image: registry.example.com/fraud-model:v4      # new candidate revision
        resources:
          requests: { cpu: "1", memory: "2Gi" }
          limits:   { cpu: "2", memory: "4Gi" }
  transformer:
    containers:
      - name: feature-transformer
        image: registry.example.com/fraud-feature-transform:1.0
```

Promotion is a one-line change (`canaryTrafficPercent: 100`, or removing the field to make the new image the sole `default`); rollback is reverting the image reference or dropping `canaryTrafficPercent` back to 0 — both far lower-ceremony than reimplementing this logic on bare Deployments.

---

### 3.6 Helm — Packaging ML Deployments

#### Theory

Every manifest shown so far has real values baked in (image tags, replica counts, resource sizes) — fine for a tutorial, untenable for a real platform running the same model across dev/staging/prod, or the same *shape* of model-serving stack across dozens of models. **Helm** solves this with **charts**: a directory structure (`Chart.yaml` metadata, `values.yaml` defaults, a `templates/` directory of Go-templated YAML) that gets rendered and applied as one versioned, named **release**. The payoff for ML platforms specifically:

- **One chart, many models**: a single "model-serving" chart with `values.yaml` fields for image, resource sizes, autoscaling thresholds, and GPU requirements lets a data science team ship a new model by writing a 15-line `values-mymodel.yaml`, not by hand-authoring Deployment/Service/HPA YAML from scratch.
- **Environment promotion**: `helm upgrade --install churn-model ./chart -f values-prod.yaml` vs `-f values-staging.yaml` — the *same* chart, different values, is the backbone of "promote the same artifact through environments" rather than "different YAML per environment that drifts."
- **Native rollback**: `helm rollback churn-model 1` reverts to a previous release revision — Helm tracks release history itself, layered on top of (not replacing) Kubernetes' own Deployment rollout history.
- **Dependency management**: an ML platform chart can declare a dependency on, e.g., a shared Redis or a KServe runtime chart, via `Chart.yaml`'s `dependencies` field.

**When Helm is overkill**: for a single, rarely-changing manifest set with no cross-environment parameterization need, plain `kubectl apply -f` (or Kustomize, for lighter-weight overlay-based environment diffs without templating) is simpler and has less to learn/debug. Helm's templating (Go templates + Sprig functions) has a real learning curve and its own class of bugs (whitespace-sensitive templates, `nil` vs empty-string edge cases) — don't introduce it for a single static manifest just because it's the "proper" tool.

#### Examples

**Beginner** — a minimal chart layout for a model-serving Deployment:

```
model-serving-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── hpa.yaml
```

```yaml
# values.yaml
image:
  repository: registry.example.com/churn-model
  tag: "1.4.2"
replicaCount: 3
resources:
  requests: { cpu: "250m", memory: "512Mi" }
  limits:   { cpu: "1",    memory: "1Gi" }
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilization: 65
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: { app: {{ .Release.Name }} }
  template:
    metadata:
      labels: { app: {{ .Release.Name }} }
    spec:
      containers:
        - name: model
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**Intermediate** — environment-specific values overrides:

```yaml
# values-prod.yaml
image: { tag: "1.4.2" }
replicaCount: 6
autoscaling: { minReplicas: 6, maxReplicas: 40 }
resources:
  requests: { cpu: "1", memory: "2Gi" }
  limits:   { cpu: "2", memory: "4Gi" }
```

```bash
helm upgrade --install churn-model ./model-serving-chart \
  -f ./model-serving-chart/values.yaml \
  -f ./model-serving-chart/values-prod.yaml \
  --namespace ml-prod --create-namespace
```

**Production-grade** — a chart that conditionally includes GPU resources and a KServe `InferenceService` template, letting the same chart serve both classical-ML and LLM workloads:

```yaml
# templates/inferenceservice.yaml
{{- if .Values.kserve.enabled }}
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: {{ .Release.Name }}
spec:
  predictor:
    minReplicas: {{ .Values.autoscaling.minReplicas }}
    maxReplicas: {{ .Values.autoscaling.maxReplicas }}
    containers:
      - name: kserve-container
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          requests:
            cpu: {{ .Values.resources.requests.cpu }}
            memory: {{ .Values.resources.requests.memory }}
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}
            {{- if .Values.gpu.enabled }}
            nvidia.com/gpu: {{ .Values.gpu.count }}
            {{- end }}
{{- end }}
```

This pattern — one chart, a `kserve.enabled` / `gpu.enabled` toggle, per-model `values-*.yaml` — is close to how a real internal "ML platform" Helm chart looks: a small set of reusable templates, a large and growing set of thin per-model values files owned by the model teams themselves.

---

### 3.7 Rolling Updates, Canary, and Blue-Green Deployments

#### Theory

Deploying a new model or code version is where most production ML incidents actually happen — not in training, in the *rollout*. Three strategies, in increasing order of safety and operational complexity:

**Rolling update (native Deployment mechanism).** Kubernetes replaces old Pods with new ones incrementally, controlled by `maxSurge` (how many extra Pods can exist above the desired count during rollout) and `maxUnavailable` (how many can be below it). It's built in, requires no extra tooling, and is *readiness-gated* — a new Pod only receives traffic once its `readinessProbe` passes. Its weakness for ML specifically: it's an **all-or-nothing eventual cutover** with no traffic-percentage control and no automated metric-based promotion/rollback — by the time you've noticed the new model regressed accuracy or latency, a meaningful fraction (often 100%, depending on how fast the rollout completes and how quickly you're watching) of your fleet is already on the bad version.

**Canary deployment.** Route a small, explicit percentage of *live production traffic* to the new version while the old version continues serving the rest, watch real metrics (error rate, latency, model-specific metrics like prediction-distribution drift or hallucination-rate proxies for LLMs), and only then progressively increase the new version's share — or roll back immediately, having only exposed a small fraction of users/requests to the regression. This is the gold standard for ML rollouts specifically because **model quality regressions are often invisible to standard infra health checks** (a bad model version returns HTTP 200 all day) — canary analysis needs to include ML-specific signals (prediction confidence distribution shift, business-metric proxies, LLM output-quality evals on a sample of live traffic), not just latency/error-rate.

**Blue-green deployment.** Run the full new version ("green") completely in parallel with the full old version ("blue"), fully warmed and validated (including synthetic/shadow traffic if desired), then cut traffic over atomically (e.g., by flipping a Service selector or a load balancer target). The old version stays running, untouched, for a defined bake period, so rollback is instant (flip back) rather than a gradual re-rollout. The cost: you're running **2x the compute** for the overlap window — often genuinely expensive for GPU-backed LLM inference fleets, which is exactly why canary (partial capacity, partial traffic) is usually preferred over blue-green for large GPU fleets specifically, while blue-green remains attractive for smaller/cheaper services or for changes you consider unusually risky (e.g., a new inference *engine*, not just a new model checkpoint).

**Where Argo Rollouts fits in:** native Kubernetes Deployments cannot do weighted canary traffic splitting or automated metric-based promotion/rollback on their own — they only know "surge/unavailable" during a rolling update. **Argo Rollouts** replaces the Deployment controller with a `Rollout` CRD that adds: canary `steps` (e.g., 10% → wait → analyze → 25% → wait → analyze → 100%), integration with a traffic-splitting layer (a service mesh like Istio, or an Ingress controller like NGINX/ALB that supports weighted routing, or a mesh-less "traffic split via replica-count ratio" fallback), and an `AnalysisTemplate` that queries Prometheus (or Datadog, or a custom webhook) after each step and **automatically aborts and rolls back** if the query fails a threshold — turning canary analysis from "an engineer stares at a dashboard" into "the rollout automatically halts and reverts on a bad metric," which is the difference between canary-as-ceremony and canary-as-real-safety-mechanism. Argo Rollouts also natively supports blue-green (`activeService`/`previewService` fields), giving you one tool for both strategies.

**KServe's relationship to this**: recall from §3.5 that `InferenceService` has native `canaryTrafficPercent` support — for pure model-serving canaries, this is often sufficient and simpler than standing up Argo Rollouts. Reach for Argo Rollouts when you need step-based progressive delivery with automated analysis gates across a broader set of workloads (not just KServe-managed ones), or when your canary policy needs to be more elaborate than a single fixed percentage.

#### Architecture

```
 ROLLING UPDATE (native)                CANARY (Argo Rollouts)                BLUE-GREEN (Argo Rollouts)

 v1 v1 v1 v1                             v1 v1 v1 v1  (100%)                  BLUE (v1): v1 v1 v1 v1  <- active traffic
   |  gradual replace, no traffic %        |  step 1: 10% -> v2                GREEN (v2): v2 v2 v2 v2 <- warming, no traffic
   v                                       v  (analyze metrics)                        |
 v1 v2 v1 v2                             v1 v1 v1 v2  (90/10)                          |  cutover (atomic)
   |                                       |  step 2: 25% -> v2                        v
   v                                       v  (analyze metrics)                BLUE (v1): idle, kept warm for rollback
 v2 v2 v1 v2                             v1 v1 v2 v2  (75/25)                 GREEN (v2): v2 v2 v2 v2 <- now active traffic
   |                                       |  ... continues to 100% or
   v                                       |  auto-aborts + rolls back
 v2 v2 v2 v2  (done)                       v  on failed analysis
                                         v2 v2 v2 v2  (done, or reverted)
```

#### Examples

**Beginner** — native rolling update tuning:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0     # never drop below desired replica count during rollout
```

**Intermediate** — Argo Rollouts canary with fixed steps (no automated analysis yet):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: fraud-model
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 10m }
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
  selector:
    matchLabels: { app: fraud-model }
  template:
    metadata: { labels: { app: fraud-model } }
    spec:
      containers:
        - name: fraud-model
          image: registry.example.com/fraud-model:v4
```

**Production-grade** — canary with an automated `AnalysisTemplate` gate (auto-rollback on latency/error-rate regression) — the mechanism that turns canary into a genuine safety net rather than a manual watch-and-hope process:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate-and-latency
spec:
  metrics:
    - name: success-rate
      interval: 2m
      successCondition: result[0] >= 0.98
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{app="fraud-model",status=~"2.."}[2m]))
            /
            sum(rate(http_requests_total{app="fraud-model"}[2m]))
    - name: p99-latency-ms
      interval: 2m
      successCondition: result[0] <= 500
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{app="fraud-model"}[2m])) by (le)
            ) * 1000
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: fraud-model
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: success-rate-and-latency
        - setWeight: 50
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: success-rate-and-latency
        - setWeight: 100
  # ...selector/template as above
```

If either analysis metric fails its condition `failureLimit` times, Argo Rollouts automatically aborts the rollout and scales the canary back to zero — no human has to be watching in real time for the rollback to happen, which is the entire point.

---

### 3.8 Kubernetes vs. ECS vs. Serverless for ML/LLM Workloads

#### Theory

This is one of the most common senior-level system-design interview questions in MLOps/LLMOps, and the wrong answer is "Kubernetes, always" — the right answer is a criteria-based tradeoff, because each option genuinely wins in different situations.

- **Kubernetes (self-managed or EKS/GKE/AKS)**: maximum control and portability, the richest GPU-scheduling and ML-ecosystem story (KServe, Kubeflow, KEDA, Ray, vLLM's native K8s operators), but the highest operational floor — you (or your platform team) own cluster upgrades, node pool management, networking, and a real learning curve for every engineer touching it.
- **ECS (AWS Elastic Container Service, especially on Fargate)**: a simpler, AWS-native container orchestrator. Far less to learn and operate than K8s, tight native integration with the rest of AWS (IAM, ALB, CloudWatch, Secrets Manager) — but locked to AWS, a much thinner ecosystem for ML-specific concerns (no KServe/Kubeflow equivalent; you build the model-serving conventions yourself), and historically weaker/newer GPU support than K8s's mature device-plugin ecosystem.
- **Serverless (AWS Lambda, Cloud Run, Azure Container Apps, SageMaker Serverless Inference, etc.)**: zero infrastructure management, true pay-per-invocation billing, and instant scale-to-zero — excellent for spiky, low-average-utilization, small-payload workloads. The dealbreakers for LLM/large-model serving specifically: **cold starts** are brutal when the "cold" cost includes loading multi-gigabyte model weights onto a GPU (Lambda in particular has historically had limited-to-no GPU support at all; Cloud Run and similar have added GPU support but with real cold-start and quota constraints), execution-time and payload-size limits that don't fit long-generation LLM streaming responses well, and per-invocation billing that gets *more* expensive than provisioned capacity once request volume is sustained rather than spiky.

| Criterion | Kubernetes | ECS (Fargate) | Serverless (Lambda/Cloud Run/etc.) |
|---|---|---|---|
| GPU support maturity | Best (device plugins, GPU Operator, MIG, time-slicing, whole LLM-serving ecosystem) | Improving but thinner; less mature multi-GPU/sharing story | Weakest historically; improving (Cloud Run GPU) but cold-start-heavy for large models |
| Operational overhead | Highest (cluster + node pool + upgrade lifecycle ownership) | Medium (AWS manages control plane; you still design task/service topology) | Lowest (fully managed) |
| Cost model | Pay for provisioned nodes (can be optimized via autoscaling/spot) | Pay for provisioned/Fargate vCPU-seconds | Pay per invocation/duration — cheapest at low/spiky volume, can exceed provisioned cost at sustained high volume |
| Cold start for large models | Manageable (keep min replicas warm, or accept K8s-native scale-to-zero via Knative/KServe with pre-warming strategies) | Similar considerations, less tooling for it | Often severe for multi-GB model weights; frequently a hard blocker for large-LLM serving |
| Portability | High (any cloud, on-prem, hybrid) | AWS-only | Vendor-specific, least portable |
| Ecosystem for ML-specific concerns | Richest (KServe, Kubeflow, KEDA, Ray, Argo) | Thin — mostly hand-built | Thin — mostly hand-built, some managed ML-specific serverless products exist (e.g., SageMaker Serverless) |
| Best fit | Sustained-traffic model/LLM serving, multi-team platform, GPU-heavy workloads, need for portability | Simpler AWS-only workloads, teams wanting less K8s operational burden, moderate/steady traffic | Spiky/low-volume traffic, lightweight (non-GPU or small-model) inference, event-driven glue/preprocessing steps, batch jobs with idle gaps |

**A practical decision heuristic**: if the workload is GPU-bound and traffic is sustained (even if variable), Kubernetes usually wins on total cost and capability once you're past a small handful of models. If traffic is genuinely spiky/low-average and the model is small enough to tolerate cold starts (or is CPU-only), serverless often wins on both cost and operational simplicity. ECS occupies the middle ground for teams already fully committed to AWS who want less operational surface than K8s but more control than pure serverless — a very reasonable choice for a smaller ML team that doesn't (yet) need multi-model-platform-scale tooling. It is also entirely normal, and often correct, for a mature platform to run **all three simultaneously** for different workload shapes (e.g., K8s for the core LLM-serving fleet, Lambda for lightweight preprocessing/webhook glue, ECS for a handful of steady internal batch services) — "which one" is a per-workload decision, not an org-wide religion.

---

## 4. Real-World Case Studies (Reasoned Inference)

> These describe how organizations with public engineering-blog patterns and industry-standard practices would plausibly architect these systems, based on general Kubernetes/ML-serving norms as of mid-2026 — not confirmed internal specifics.

**A frontier LLM lab (an OpenAI/Anthropic-scale organization)** serving models at massive, highly variable global QPS would plausibly run model inference on Kubernetes (or a very similar internally-built scheduler) primarily to get fine-grained control over GPU fleet scheduling: bin-packing many replicas across heterogeneous GPU generations, and — reflecting the frontier the llm-d project represents — likely running **disaggregated prefill/decode** pools, where prefill (compute-bound, benefits from batching many prompts) and decode (memory-bandwidth-bound, benefits from tight latency-per-token) run on separately-sized and separately-autoscaled node pools rather than one homogeneous fleet, plus continuous-batching-aware inference servers (vLLM-style) rather than one-request-per-Pod service semantics. Canary/gradual-rollout discipline for new model checkpoints would matter enormously here given blast radius — expect automated, metric-gated progressive rollouts (the Argo-Rollouts-style pattern) rather than manual promotion, likely combined with extensive shadow-traffic evaluation before any live-traffic canary begins.

**A large-scale consumer recommendation platform (a Netflix/Spotify-scale organization)** running thousands of models (per-market, per-feature, per-experiment-arm recommendation models) would plausibly lean hard on a KServe-or-equivalent standardized serving layer specifically *because* of the model count — hand-rolling Deployment/Service/HPA per model doesn't scale organizationally past a few dozen models, whereas a chart/CRD-based "any team can ship a model by writing 20 lines of config" platform does. Given how many of these models are small and traffic-per-model is often low (long-tail experiment arms, niche markets), scale-to-zero and GPU/CPU bin-packing (time-slicing-style sharing, or simply running most of these on CPU where the recommendation model doesn't need a GPU at all) would plausibly matter more here than at a frontier-LLM-lab, where the marginal model is enormous and heavily used.

**A ride-sharing/marketplace platform (an Uber-scale organization)** with real-time pricing/ETA/fraud models under strict latency SLAs would plausibly favor StatefulSet-backed, self-managed feature-store and low-latency serving infrastructure specifically where sub-millisecond feature lookups matter enough to justify running (rather than consuming as managed) a clustered in-memory store, paired with HPA/KEDA scaling model-serving replicas on queue-depth or requests-in-flight rather than CPU, given how latency-sensitive and bursty (rush hour, weather events, local demand spikes) marketplace traffic characteristically is.

**A cloud/data platform vendor (a Databricks-scale organization)** building a multi-tenant ML platform product for external customers would plausibly invest heavily in the Helm-chart-as-product-surface pattern — customers/internal teams configuring a shared, versioned serving chart via values files — plus strict `ResourceQuota`/`LimitRange`/namespace-isolation patterns per tenant (a concern this module has touched on lightly via requests/limits, and which becomes existential at true multi-tenant SaaS scale), since a single noisy or misconfigured tenant workload must never be able to starve another tenant's GPU allocation.

**A hardware/infrastructure vendor (an NVIDIA-scale organization)** would plausibly be the natural owner and reference implementer of exactly the GPU Operator / device-plugin / MIG / time-slicing stack this module describes — worth remembering that the tooling covered in §3.2 isn't incidental tooling any team could have built equally well; it exists because the hardware vendor itself has the strongest incentive and deepest low-level access to make GPU scheduling on Kubernetes correct and efficient, which is exactly why the GPU Operator (rather than a community-maintained alternative) is the de-facto standard.

---

## 5. Common Mistakes

1. **Requesting GPUs as fractional values or in `requests` instead of `limits`.** The stock NVIDIA device plugin does not support `nvidia.com/gpu: 0.5` — GPUs are whole-unit extended resources. Engineers coming from CPU/memory intuition frequently try this and get an immediately-rejected Pod spec (or, worse, silently-ignored fractional truncation depending on the client tooling).
2. **Autoscaling GPU-bound LLM inference on CPU utilization.** CPU sits low and flat while the GPU is saturated; the HPA never fires when it should, and engineers conclude "autoscaling doesn't work for our LLM service" when the real bug is the choice of metric.
3. **Running HPA and VPA on the same metric for the same workload.** The two controllers can fight each other's decisions; pick one axis per resource metric per workload.
4. **Setting `requests == limits == "as much as I can spare"` with no real capacity planning**, leading to either chronic OOMKills (limits too tight for real peak usage) or wildly wasteful spend (limits far above anything ever used) — both are symptoms of skipping the VPA-recommender or load-testing step that should inform these numbers.
5. **Treating a StatefulSet's Pods as if they were Deployment Pods** — e.g., deleting Pod-1 out of a 3-node cluster expecting it to come back identically without checking whether the paired PVC and any manual bootstrap/rejoin steps for that specific ordinal are handled by the application itself. StatefulSet gives you stable *identity and storage attachment*; it does not automatically give a distributed system correct rejoin/resync logic — that's still the application's job.
6. **Shipping a `BestEffort` QoS Pod (no requests/limits at all) to production.** It will be the very first thing evicted under any node memory pressure, regardless of how important the workload actually is.
7. **Using a plain rolling update for a new model version and considering "no HTTP errors" sufficient validation.** A model that regressed silently (bad predictions, hallucinations, drifted confidence calibration) returns HTTP 200 all day — canary analysis for ML rollouts must include model-quality signals, not just infra health.
8. **Reaching for Kubernetes by default for a single, simple, low-traffic model** rather than evaluating serverless/ECS options — and conversely, forcing a sustained-traffic, GPU-heavy LLM workload onto a serverless platform because "it's simpler," and then being surprised by cold-start latency and per-invocation cost blowing past provisioned-capacity cost.
9. **Forgetting `terminationGracePeriodSeconds`/`preStop` hooks on latency-sensitive services**, causing in-flight requests to be abruptly cut off during routine rollouts or scale-downs rather than allowed to drain.
10. **Installing KServe/Kubeflow as a monolithic all-or-nothing platform decision** when only the serving layer (KServe standalone) is actually needed — both are composable; you don't have to adopt the entire Kubeflow surface area to get `InferenceService`.

---

## 6. Best Practices and Production Tips

- **When to use Kubernetes at all**: multiple models/teams, sustained (even if variable) traffic, GPU-heavy workloads, or a genuine multi-cloud/portability requirement. When NOT to: a single low-traffic model, a small team without platform-engineering capacity, or spiky/low-volume traffic better served by serverless — see §3.8's decision heuristic.
- **Resource governance**: set `ResourceQuota` and `LimitRange` per namespace in any multi-tenant cluster from day one — retrofitting quotas onto an already-running, already-contentious cluster is far more painful than starting with them.
- **GPU efficiency**: default to whole-GPU allocation for latency-SLA'd production inference; use time-slicing for dev/test and genuinely low-QPS endpoints; evaluate MIG when you need real isolation across multiple tenants/workloads sharing modern (A100/H100-class) hardware.
- **Autoscaling metric selection**: for GPU-bound inference, scale on GPU utilization and/or queue depth/concurrency (KEDA), not CPU. Always set a `behavior.scaleDown.stabilizationWindowSeconds` to avoid flapping on noisy, bursty traffic.
- **Observability is a prerequisite, not an add-on, for safe rollouts**: canary analysis (whether via KServe's native canary field or Argo Rollouts' `AnalysisTemplate`) is only as good as the metrics feeding it — Module 07/08-level observability (Prometheus, structured logging, LLM-specific eval signals) needs to already be wired up before you can trust an automated promotion/rollback gate.
- **Cost**: GPUs are almost always the dominant line item for LLM infrastructure — prioritize autoscaling correctness (scaling down aggressively when idle, scale-to-zero via KEDA/Knative for low-traffic endpoints) and right-sized `requests`/`limits` (informed by VPA recommendations or load testing) over almost any other cost lever.
- **Security**: run model-serving containers as non-root, set `readOnlyRootFilesystem` where the runtime allows it, scope `ServiceAccount` permissions tightly (a model server almost never needs broad cluster RBAC), and treat model-weight storage credentials (object storage access for `storageUri` in KServe) as sensitive secrets, not baked into images.
- **Rollout strategy selection**: default to KServe's native canary for pure model-serving canaries; reach for Argo Rollouts when you need multi-step progressive delivery with automated analysis gates, or blue-green for unusually risky changes (new inference engine, not just new checkpoint) where doubling compute for a bounded window is an acceptable cost for instant-rollback safety.
- **Helm hygiene**: keep charts thin and values-driven; avoid embedding business logic in template conditionals beyond simple feature toggles (`gpu.enabled`, `kserve.enabled`); pin chart versions in CI, never `helm upgrade` against a floating `latest`.
- **Stay current but pragmatic**: `LLMInferenceService` and llm-d represent where the ecosystem is heading for frontier-scale LLM serving, but for the overwhelming majority of production workloads in mid-2026, the mature `InferenceService` (KServe) + HPA/KEDA + Argo Rollouts stack described in this module remains the correct, production-proven default. Track the newer projects; don't rebuild on them before they're stable for your risk tolerance.

---

## 7. Interview Questions

1. **Q: Why can't you request a fractional GPU (e.g., `nvidia.com/gpu: 0.5`) the way you can request `cpu: "500m"`?**
   A: GPUs are exposed to Kubernetes via the device-plugin extended-resource mechanism, not the native, natively-divisible CPU/memory resource model. The stock NVIDIA device plugin advertises whole GPU units and does not support fractional requests; to oversubscribe a physical GPU across multiple Pods you need an explicit sharing mechanism layered on top — GPU Operator time-slicing (software round-robin, no memory isolation) or NVIDIA MIG (hardware-level partitioning with real isolation) on supported cards.

2. **Q: You've set up HPA to scale an LLM inference Deployment on CPU utilization, but it never scales up even under heavy load. Why, and how do you fix it?**
   A: The inference workload is almost certainly GPU-bound, not CPU-bound — the GPU saturates while CPU utilization stays low and flat, so the CPU metric never crosses the scaling threshold. Fix by scaling on a metric that actually reflects load: GPU utilization exposed via DCGM exporter → Prometheus → Prometheus Adapter → HPA `External` metric, and/or queue-depth/concurrent-requests via KEDA.

3. **Q: When would you choose a StatefulSet over a Deployment for an ML component, and what does a StatefulSet NOT give you for free?**
   A: Choose StatefulSet when replica identity matters — stable network address and stable, reattached storage per ordinal, plus ordered startup/shutdown — e.g., a sharded/clustered vector database, or distributed inference/training ranks needing stable peer discovery. It does not give you correct distributed-system rejoin/resync logic automatically; that's still the application's responsibility. It also carries meaningfully higher operational overhead than a Deployment, so prefer a managed stateful service when one fits instead of self-hosting on a StatefulSet by default.

4. **Q: Explain why running HPA and VPA on the same resource metric for the same workload is risky.**
   A: VPA changes the Pod's `requests`/`limits` for a resource (say CPU), which changes the denominator HPA uses to compute utilization percentage for that same resource — the two controllers can end up reacting to each other's changes, causing instability/flapping. The safe pattern is to let each control a different axis: VPA manages a resource's sizing (or runs recommendation-only) while HPA scales replicas on a distinct metric (e.g., a custom/external metric), or use VPA alone (fixed replica count) when you don't also need horizontal scaling.

5. **Q: What specifically does KServe's `InferenceService` give you that a hand-rolled Deployment + Service + HPA doesn't?**
   A: Standardized multi-framework serving runtimes (point at a model artifact and a framework name rather than writing a custom server), a composable Predictor/Transformer/Explainer request pipeline, native canary traffic-percentage splitting without custom rollout tooling, and (in serverless/Knative mode) concurrency-based autoscaling including scale-to-zero — all as declarative CRD fields rather than bespoke per-model infrastructure code.

6. **Q: How does KEDA relate to HPA — is it a replacement?**
   A: No. KEDA implements the Kubernetes Custom/External Metrics API on HPA's behalf, backed by a large catalog of event-source scalers (queues, Kafka lag, Prometheus queries, cron, etc.), and adds scale-to-zero (which plain HPA cannot do, since its floor is 1 replica) by decoupling "is there work at all" from "how many replicas," delegating the latter to a standard HPA object it creates and manages once replicas are above zero.

7. **Q: Contrast canary and blue-green deployment for a model-serving workload — when would you pick each?**
   A: Canary routes a small, increasing percentage of live traffic to the new version alongside the old, ideally with automated metric-based promotion/rollback gates (Argo Rollouts' `AnalysisTemplate`), limiting blast radius while only paying for partial extra capacity during the rollout. Blue-green runs a full duplicate environment and cuts traffic over atomically with instant rollback, at the cost of running 2x compute during the overlap — often too expensive for large GPU fleets, but attractive for smaller services or unusually risky changes (e.g., a new inference engine) where instant, guaranteed rollback outweighs the doubled-compute cost.

8. **Q: A stakeholder asks "why not just use Kubernetes for everything, since it's the most powerful option?" How do you respond?**
   A: Power isn't free — Kubernetes carries the highest operational floor of the three options (cluster/node-pool/upgrade ownership, real learning curve). For workloads with genuinely spiky, low-average traffic and models small/tolerant enough for cold starts, serverless wins on both cost and simplicity; for teams fully committed to AWS wanting less operational surface than K8s with more control than serverless, ECS is a reasonable middle ground. The decision should be made per-workload against criteria (GPU need, traffic pattern, portability requirement, team's platform-engineering capacity), not defaulted to the most capable tool.

---

## 8. Summary, Key Takeaways, and Production Checklist

### Summary

Kubernetes gives ML and LLM systems the operational substrate that a trained model or containerized LLM pipeline needs to become a reliable, scalable, safely-updatable production service: Pods/Deployments/Services/Ingress for the basic serve-and-route mechanics; GPU-aware scheduling via device plugins and the GPU Operator for the accelerator-heavy reality of deep learning and LLM inference; HPA/VPA/KEDA for scaling on the *right* signal (often not CPU) including down to zero for cost efficiency; StatefulSets for the minority of components where replica identity genuinely matters; KServe (optionally under the broader Kubeflow umbrella) for a standardized, canary-and-autoscaling-aware model-serving layer rather than bespoke per-model infrastructure; Helm for packaging and promoting that infrastructure across environments and models; and progressive-delivery strategies (rolling/canary/blue-green, with Argo Rollouts for the automated-analysis-gated versions) for making version changes something you can catch and reverse rather than something you hope goes well. The field keeps moving — KServe's CNCF Incubating status, Kubeflow's generative-AI-centered 1.11 repositioning, and llm-d's CNCF Sandbox entry all happened within the year before this writing — but the fundamentals in this module are the stable core that new frontier tooling is being built *on top of*, not around.

### Key Takeaways

- Pods are disposable; Deployments/Services exist precisely because Pods are disposable.
- GPUs are whole-unit extended resources in `limits`, not natively divisible like CPU — sharing requires explicit tooling (time-slicing or MIG).
- Autoscale GPU-bound ML/LLM workloads on GPU utilization or queue depth/concurrency, not CPU.
- HPA and VPA should not both actively manage the same metric on the same workload.
- StatefulSets are for identity-sensitive stateful components — not a default upgrade from Deployment.
- KServe's `InferenceService` (and the newer `LLMInferenceService` for generative inference) removes the need to hand-build standardized serving/canary/autoscaling infrastructure per model.
- Helm makes a serving stack reusable and promotable across models and environments; don't reach for it when a static manifest genuinely suffices.
- Canary with automated, metric-gated analysis is the production-grade rollout default for ML; plain rolling updates lack traffic-percentage control and don't catch silent model-quality regressions.
- Kubernetes vs. ECS vs. serverless is a per-workload decision driven by GPU need, traffic pattern, portability requirement, and team capacity — not a religion.

### Production Checklist

- [ ] Every container has explicit, right-sized `requests` AND `limits` (no `BestEffort` Pods in production).
- [ ] GPU workloads request whole `nvidia.com/gpu` units in `limits`; time-slicing/MIG decision made deliberately, not by default.
- [ ] Autoscaling metric matches the true bottleneck (GPU utilization/queue depth for inference, not blind CPU).
- [ ] HPA and VPA are not both managing the same metric on the same workload.
- [ ] `readinessProbe`/`livenessProbe` configured; `terminationGracePeriodSeconds`/`preStop` set for graceful drain on latency-sensitive services.
- [ ] `PodDisruptionBudget` in place for any multi-replica production service.
- [ ] Stateful components (vector DBs, clustered stores) run as StatefulSets with anti-affinity and PDBs, or — preferably where feasible — offloaded to a managed service.
- [ ] Model serving standardized behind KServe (or an equivalent) rather than N bespoke Deployment stacks, once past a handful of models.
- [ ] Deployment packaged as a Helm chart with environment-specific `values-*.yaml`, versioned in CI.
- [ ] Rollouts use canary (KServe native or Argo Rollouts) with automated, metric-gated promotion/rollback — including model-quality signals, not just infra health — for any change with real blast-radius risk.
- [ ] `ResourceQuota`/`LimitRange` set per namespace in any multi-tenant cluster.
- [ ] Kubernetes-vs-ECS-vs-serverless choice documented per workload against explicit criteria (GPU need, traffic shape, portability, team capacity) — not assumed.

---

## 9. Further Reading

Detailed citations, official documentation links, curated videos, recommended books, and reference GitHub repositories for every concept in this chapter are maintained separately in this same module folder: see **`references.md`**, **`videos.md`**, **`books.md`**, and **`github.md`**. Consult those files for primary-source depth (e.g., the full Kubernetes GPU-scheduling docs, KEDA's scaler catalog, KServe's `InferenceService`/`LLMInferenceService` reference, and Argo Rollouts' `AnalysisTemplate` spec) rather than this chapter re-deriving them at length.
