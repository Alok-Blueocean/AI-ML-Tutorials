# Kubernetes for ML/LLM Systems — Scenario-Based Q&A

**Situation:** An engineer new to ML-on-K8s writes `resources: { limits: { nvidia.com/gpu: 0.5 } }` to try to give a lightweight model server "half a GPU," and the Pod is rejected at admission. What would you do and why?

Model answer: Explain that this isn't a YAML syntax error — it's a fundamental property of the device-plugin model. GPUs are advertised to the kubelet as whole-unit extended resources by the NVIDIA device plugin, and the stock plugin does not support fractional values; a GPU is not natively divisible the way CPU millicores are. To actually share a GPU across multiple lightweight Pods, enable GPU Operator time-slicing (configuring the device plugin to advertise, say, 4 virtual `nvidia.com/gpu` units per physical GPU) and have each Pod still request whole units (`nvidia.com/gpu: 1`) against that virtualized pool — understanding that time-slicing gives no memory isolation, so it's appropriate for dev/test and low-QPS endpoints, not latency-SLA'd production inference, where MIG or whole-GPU assignment is the correct choice instead.

---

**Situation:** A production inference Deployment scaled via HPA-on-CPU never scales up during a traffic spike, even though users are reporting slow responses and the on-call dashboard shows GPUs pegged near 100%. What would you do and why?

Model answer: Recognize this as the classic GPU-bound-workload-scaled-on-the-wrong-metric failure: CPU utilization is a poor proxy for load on a GPU-bound LLM inference service, and it will sit low and flat while the actual bottleneck (the GPU) is saturated, so HPA-on-CPU simply never fires. Fix it by wiring DCGM exporter metrics through Prometheus and the Prometheus Adapter so HPA can scale on an External metric (GPU utilization), and/or add a KEDA `ScaledObject` scaling on queue depth or in-flight-request count from the inference server's own metrics endpoint (e.g., vLLM's). Don't just raise the CPU target threshold — that treats the symptom, not the wrong-metric root cause.

---

**Situation:** A platform team enables both an HPA and a VPA on the same Deployment, targeting CPU, "to get the best of both worlds" — automatic right-sizing and automatic replica scaling. Within days they notice replica counts oscillating unpredictably. What would you do and why?

Model answer: Diagnose this as the documented HPA/VPA conflict: VPA is changing the Pod's CPU `requests`/`limits` based on observed usage, which changes what "100% CPU utilization" even means for that Pod, which in turn confuses HPA's target calculation on the same metric — the two controllers end up fighting each other's decisions. Resolve by picking one axis per metric per workload: either run VPA in recommendation-only mode (no auto-apply) and let a human periodically adjust requests/limits while HPA scales replicas on CPU, or keep VPA fully active but move HPA to a different metric entirely (e.g., an external queue-depth or request-concurrency metric) so the two controllers are no longer contesting the same signal.

---

**Situation:** During a routine node-pool upgrade, a StatefulSet-backed self-hosted vector database goes into a split-brain state, with two Pods each believing they're the shard leader. What would you do and why?

Model answer: Point out that StatefulSets guarantee stable network identity and stable storage reattachment, not correct distributed-systems rejoin/resync logic — that's the application's responsibility, and it's a common misunderstanding to treat StatefulSet guarantees as if they solved leader-election and split-brain automatically. Investigate whether the node drain violated the StatefulSet's intended ordered scale-down (0→N reverse order) or whether a `PodDisruptionBudget` was missing/misconfigured, allowing more than one shard to go down simultaneously during the upgrade. Add a `PodDisruptionBudget` with `maxUnavailable: 1`, and separately verify the vector database's own clustering layer has correct quorum/fencing behavior — the fix is as much an application-config problem as a Kubernetes one.

---

**Situation:** Leadership asks whether the team should migrate its dozens of small, low-traffic internal ML models off Kubernetes onto AWS Lambda to "cut costs," since most of these models sit idle most of the day. What would you do and why?

Model answer: Give a criteria-based answer rather than a reflexive yes or no. If these are small, CPU-only or lightweight models with tolerable cold-start latency, serverless can genuinely win on both cost and operational simplicity for spiky, low-average-utilization traffic. But if any of them are GPU-backed or involve multi-gigabyte model weights, flag that cold starts loading large weights onto a GPU are a severe, often disqualifying problem for serverless platforms, and that per-invocation billing can exceed provisioned-capacity cost once traffic is sustained rather than truly spiky. A more likely right answer: keep GPU-heavy, sustained-traffic models on Kubernetes with KEDA scale-to-zero (which gets most of the cost benefit without serverless's cold-start and payload-size constraints), and migrate only the genuinely small, CPU-only, spiky-traffic models to Lambda or Cloud Run.

---

**Situation:** A model-serving canary rollout shows zero increase in HTTP error rate, so the on-call engineer promotes the new version to 100% traffic — and two days later, product analytics reveals the new model has been silently giving worse recommendations the whole time. What would you do and why?

Model answer: Treat this as a textbook case of "no HTTP errors" being an insufficient canary gate for ML rollouts — a model that regressed in prediction quality, calibration, or hallucination rate returns HTTP 200 all day, so infra health alone cannot catch it. Require canary analysis (via KServe's native canary support or an Argo Rollouts `AnalysisTemplate`) to include model-quality signals — prediction distribution shift, confidence calibration, business-metric proxies, or LLM-eval scores — not just latency and error rate, before any promotion gate fires. As a remediation, roll back immediately to the prior revision (the old ReplicaSet/model revision is still available), and retroactively add the missed quality metric to the automated gate so this class of regression is caught next time without depending on a human noticing a downstream analytics report.

---

**Situation:** A senior engineer proposes running a self-managed, StatefulSet-backed Kafka cluster on the team's Kubernetes cluster to back a new feature store, purely because "we're already on K8s, so we might as well." What would you do and why?

Model answer: Push back with the module's core StatefulSet guidance: if a managed alternative fits (a managed Kafka service, RDS, managed Redis, a hosted vector DB), prefer it and keep your own K8s footprint stateless — StatefulSets are meaningfully more operationally demanding than Deployments, since your team now owns volume lifecycle, backup/restore, and manual intervention for split-brain or stuck-terminating Pods. Ask what specific requirement (cost at extreme scale, data residency, a feature genuinely unavailable managed) justifies taking on that operational burden; "we're already on K8s" alone is not a sufficient reason to self-host stateful infrastructure, since running your own Kafka is a deliberate cost/control tradeoff, not a default.

---

**Situation:** An LLM inference Deployment is OOMKilled repeatedly under bursty traffic, even though average memory usage looks comfortably under the configured limit on dashboards. What would you do and why?

Model answer: Explain that memory is an incompressible resource in Kubernetes — exceeding the limit triggers an abrupt OOMKill (SIGKILL), not graceful throttling like CPU — so "average usage looks fine" is the wrong statistic to trust; what matters is peak RSS during a large batch or a memory-hungry tokenizer call, which a dashboard averaging over a wide window can easily hide. Load-test with realistic burst patterns (not steady-state traffic) to find true peak memory, and either raise the memory limit with margin above that measured peak, or use the VPA recommender (in recommendation-only mode) over a representative traffic window to get a data-driven right-sizing number rather than guessing. Also confirm the Pod isn't running as `BestEffort` or under-provisioned `Burstable` QoS, since that affects eviction priority under node-level memory pressure too.

---

**Situation:** A platform team wants every new model a data scientist trains to be servable in production "within a day," without each team hand-writing a Dockerfile, FastAPI wrapper, and Deployment/Service/HPA trio per model. What would you do and why?

Model answer: Recommend standing up KServe (standalone, without necessarily adopting the full Kubeflow platform) so a data scientist ships a model by writing a short `InferenceService` YAML pointing at a model artifact in object storage with a declared framework, rather than hand-rolling serving infrastructure per model. This gives standardized multi-framework runtimes, built-in canary rollout fields, and — on Knative-based serverless mode — autoscaling including scale-to-zero, all as reusable declarative infrastructure instead of bespoke YAML per team. Frame this explicitly as a "more than a couple of models, need consistent operational behavior" decision per the module's guidance — for a single simple model, a hand-rolled Deployment can still be the right, simpler call.

---

**Situation:** During a blue-green cutover for a new inference engine version, the team is surprised by a doubled cloud bill for the cutover window and questions whether blue-green was the right rollout strategy. What would you do and why?

Model answer: Confirm that doubled compute cost during the overlap window is the expected, inherent cost of blue-green — you're running two complete environments simultaneously specifically to get atomic traffic switching and instant rollback — and that this is a deliberate tradeoff, not a bug. Revisit whether blue-green was actually warranted for this change: the module's guidance is to reserve blue-green for unusually risky changes (like a new inference engine, not just a new checkpoint) where the bounded extra cost buys meaningfully safer, instant rollback; for routine model-checkpoint updates, a canary (via KServe's native field or Argo Rollouts) achieves progressive risk reduction without paying for a full second environment. If the change genuinely warranted blue-green, communicate the cost as an expected, time-bounded line item rather than an anomaly.

---

**Situation:** A newly onboarded engineer deletes a Pod belonging to a Deployment expecting it to simply "go away," and is confused when a new Pod with the same labels appears seconds later. What would you do and why?

Model answer: Use it as a teaching moment about the core mental-model shift for Kubernetes: Pods are ephemeral and disposable by design, and a Deployment's job is to continuously reconcile the cluster toward a declared replica count via its underlying ReplicaSet — deleting one Pod doesn't change the desired state, so the controller immediately creates a replacement. Explain the practical implications: never SSH into a Pod expecting persistence, never hard-code a Pod's IP anywhere, and rely on the Service's label-selector-based routing (which automatically updates its endpoint list as Pods come and go) rather than tracking individual Pod identities for anything but StatefulSet-managed stateful workloads.

---

**Situation:** A model-serving team ships a rolling update, and during the rollout window a burst of in-flight requests gets abruptly cut off mid-response, generating a spike of client-side errors even though the deployment "succeeded" with no failed health checks. What would you do and why?

Model answer: Identify the missing `terminationGracePeriodSeconds` and `preStop` hook as the likely root cause — without them, a Pod receives SIGTERM and can be killed before in-flight requests finish, since the default grace period is short and nothing tells the container to drain connections first. Add an explicit `terminationGracePeriodSeconds` (e.g., 30s) and a `preStop` hook that sleeps briefly or actively drains connections before the container receives SIGTERM, giving in-flight requests time to complete while the Service's endpoint list has already stopped routing new traffic to the terminating Pod. Validate the fix under load by watching client-side error rates during a rollout, not just Kubernetes-reported rollout status.

---

**Situation:** A cost-conscious engineering lead asks why the team's GPU bill hasn't dropped despite adding autoscaling, when traffic is clearly bursty with long idle stretches overnight. What would you do and why?

Model answer: Check whether the autoscaler in use can actually scale to zero — plain HPA's minimum replica count is 1, so "autoscaling" alone doesn't eliminate the idle-GPU cost during overnight lulls. Recommend layering KEDA in front of HPA specifically for its scale-to-zero capability (via its own lightweight polling loop deciding whether any work exists at all, independent of HPA's replica-count logic once above zero), scaling on a genuine demand signal like queue depth or requests-in-flight. Pair this with a review of `minReplicas` settings across all GPU-backed Deployments — a forgotten `minReplicas: 2` "just to be safe" on a low-traffic endpoint silently defeats the entire cost benefit of scale-to-zero infrastructure.

---

**Situation:** A team maintaining Helm charts for a dozen models finds that every new model requires copy-pasting and hand-editing a large chunk of templated YAML, and deployments are starting to drift subtly between models. What would you do and why?

Model answer: Diagnose this as a Helm-hygiene failure — charts that aren't kept thin and values-driven tend to accumulate business logic in template conditionals, which is exactly what makes copy-paste-and-edit feel necessary instead of "add a new values.yaml." Refactor toward one well-parameterized chart (or a small chart library) where per-model differences are expressed entirely through `values.yaml` overrides (image, resource sizing, GPU toggle, autoscaling policy), not through duplicated template files. Pin chart versions in CI and never `helm upgrade` against a floating `latest`, so drift between "what's templated" and "what's actually deployed" becomes visible and auditable rather than accumulating silently across a dozen hand-maintained copies.

---

**Situation:** An engineer configures GPU time-slicing to run four small models on one physical GPU to save cost, and shortly after, one model's Pod starts getting killed with out-of-memory errors that don't correlate with its own request volume. What would you do and why?

Model answer: Explain that time-slicing shares GPU compute round-robin but provides no VRAM isolation between the sharing Pods — one Pod's memory-hungry batch can OOM another Pod sharing the same physical GPU, even though from Kubernetes' perspective each Pod's own resource accounting looks fine. Confirm this is the failure mode by checking whether the OOMs correlate with a neighboring model's traffic rather than the affected model's own load. If real isolation is required (these are production, not just dev/test, workloads), migrate from time-slicing to MIG on A100/H100-class hardware for hardware-level partitioned isolation, or fall back to whole-GPU assignment per replica if the workload's request volume justifies dedicating a full accelerator.

---

**Situation:** A new hire proposes exposing ten different internal models by provisioning ten separate `LoadBalancer`-type Services, one per model, for a soon-to-launch internal platform. What would you do and why?

Model answer: Flag this as a costly, non-scaling pattern — provisioning one cloud load balancer per model is expensive and stops being manageable past a handful of services. Recommend a single Ingress (or a small number of them) with an Ingress controller (NGINX, AWS Load Balancer Controller, Istio Gateway, or a purpose-built AI gateway) doing L7 path/host-based routing to internal `ClusterIP` Services per model instead, consolidating to one or two cloud load balancers total. If the platform will eventually route between many models with per-model traffic-shaping, rate limiting, or cost-tracking needs, suggest evaluating a purpose-built AI gateway sitting behind the Ingress rather than hand-rolling that logic in Ingress annotations.

---

**Situation:** A model-serving Pod spec has no `resources` block at all — an engineer reasoned "it's a small model, it doesn't need limits" — and during a node memory-pressure event, that Pod is the first thing evicted, taking down a customer-facing feature with no warning. What would you do and why?

Model answer: Identify this as a `BestEffort` QoS class problem — with no `requests`/`limits` set at all, the Pod gets the lowest possible scheduling priority and is evicted first under any node resource pressure, regardless of how important the workload actually is to the business. Fix by setting explicit `requests`/`limits` sized from real load-testing or VPA-recommender data, pushing the Pod into at least `Burstable` QoS, and for genuinely latency-sensitive production services, prefer `Guaranteed` QoS (requests equal to limits on every resource) so it's evicted last under node pressure. Treat "no resources block" as a production-readiness blocker in code review going forward, not a shortcut acceptable for "small" workloads.

---

**Situation:** A platform architect is designing the GPU strategy for a new multi-tenant internal ML platform where several teams will share a cluster, and one team's experimental workload has already once starved another team's production model of GPU capacity. What would you do and why?

Model answer: Point to `ResourceQuota` and `LimitRange` as the missing per-namespace resource-governance layer — without them, any tenant's workload can consume unbounded cluster capacity, which is exactly the failure that occurred. Recommend setting these from day one for every tenant namespace (not retrofitting them onto an already-contentious cluster, which is far more painful), sized against each team's negotiated capacity allocation, and pair this with `podAntiAffinity`/topology-aware scheduling so no single node failure or noisy-neighbor workload can degrade another tenant's SLA. Separately audit whether GPU node pools are shared or dedicated per tenant, since quota enforcement alone doesn't fix physical bin-packing contention on shared GPU nodes.

---

**Situation:** A team is deciding where to deploy a new fraud-detection model that needs sub-100ms p99 latency and runs at a steady, predictable 200 requests/second around the clock. A junior engineer suggests AWS Lambda "because it's simpler to manage." What would you do and why?

Model answer: Walk through the decision heuristic rather than accepting "simpler" as sufficient: this workload has sustained, non-spiky traffic and a strict latency SLA, which is precisely the profile where serverless's cold-start risk and per-invocation cost at sustained volume work against it, and where Kubernetes (or ECS, if the team wants less operational surface and is fully AWS-committed) tends to win on both cost and latency predictability once traffic is steady rather than bursty. Recommend Kubernetes with `minReplicas` kept warm (no scale-to-zero, since a cold start would violate the p99 target) and GPU or CPU sizing informed by load testing, reserving serverless for this team's genuinely spiky, low-average-utilization workloads instead.

---

**Situation:** After adopting KServe, a team is confused about whether they now also need to adopt Kubeflow Pipelines, Notebooks, and Katib, since documentation keeps mentioning "Kubeflow" alongside KServe. What would you do and why?

Model answer: Clarify that KServe is fully usable standalone and does not require adopting the rest of the Kubeflow platform — Kubeflow's pipelines, notebooks, and Katib hyperparameter tuning are separate, composable components, none of which is mandatory to get `InferenceService`'s serving capabilities. Recommend installing only what solves an actual current problem (KServe for serving) and evaluating the other Kubeflow components independently against real needs (e.g., Katib only if hyperparameter tuning is a genuine current bottleneck), rather than treating "Kubeflow" as an all-or-nothing platform decision — installing the full surface area when only serving is needed is called out explicitly as a common mistake.
