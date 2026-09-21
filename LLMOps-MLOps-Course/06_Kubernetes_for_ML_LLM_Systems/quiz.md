# Module 06 — Quiz: Kubernetes for ML and LLM Systems

Instructions: Attempt every question before checking the answer key at the end. Questions 1-8 are
multiple-choice (one best answer unless stated otherwise); questions 9-15 are short-answer. This
quiz covers `tutorial.md` and `architecture.md`.

---

**Q1.** Why does Kubernetes schedule a *Pod* rather than a container directly?

A. Because containers cannot run on Kubernetes without being wrapped in a virtual machine first.
B. A Pod is the unit that guarantees co-located containers share a network namespace and can
   optionally share storage — the container runtime has no native concept of "these containers
   must land on the same node together."
C. It's purely a historical naming choice with no technical consequence.
D. Pods only exist for GPU workloads; CPU workloads schedule containers directly.

---

**Q2.** A container exceeds its memory `limit`. What happens, and how does this differ from
exceeding its CPU `limit`?

A. Both result in the same graceful throttling behavior; memory and CPU limits are enforced
   identically.
B. Exceeding the memory limit gets the container `OOMKilled` (an abrupt SIGKILL) because memory is
   incompressible; exceeding the CPU limit only throttles it (slower execution), because CPU is
   compressible.
C. Exceeding either limit simply logs a warning with no runtime enforcement.
D. Exceeding the memory limit pauses the container until memory frees up elsewhere on the node.

---

**Q3.** Why can't you request `nvidia.com/gpu: "0.5"` the way you can request `cpu: "500m"`?

A. Kubernetes intentionally blocks fractional GPU requests for licensing reasons.
B. GPUs are exposed as extended resources via the device-plugin framework, which advertises whole
   schedulable units; the stock NVIDIA device plugin does not support fractional values, unlike
   the natively divisible CPU/memory resource model.
C. Fractional GPU requests work fine as long as you also set an equal fractional `requests` value.
D. GPUs can only be requested in Pods that also request at least 1 full CPU.

---

**Q4.** An LLM inference Deployment is GPU-bound. Its HPA is configured with
`type: Resource, name: cpu, averageUtilization: 65`. What will most likely happen under heavy load?

A. The HPA will scale correctly, since CPU utilization always tracks GPU load proportionally.
B. The HPA will likely never fire (or fire far too late), because CPU utilization stays low and
   flat while the GPU is the actually-saturated resource — CPU is a misleading proxy for load here.
C. The HPA will scale to its `maxReplicas` immediately regardless of the metric.
D. HPA cannot be configured with a CPU metric for GPU workloads at all — the manifest would be
   rejected by the API server.

---

**Q5.** What is the safe pattern when both HPA and VPA are relevant to the same Deployment?

A. Always run both in `Auto` mode on the same metric for maximum responsiveness.
B. Never use VPA on any workload that also uses HPA, under any configuration.
C. Have each control a different axis — e.g., VPA in recommendation-only mode for right-sizing
   requests/limits, while HPA scales replicas off a distinct metric (not the one VPA is adjusting).
D. Run VPA first to completion, then delete it before ever installing HPA.

---

**Q6.** What does KEDA add on top of plain HPA-with-Prometheus-Adapter that is most relevant for
cost-efficient, bursty LLM batch-inference workloads?

A. KEDA replaces HPA entirely with an incompatible scaling mechanism.
B. KEDA can scale a Deployment to and from **zero** replicas (via its own polling loop, handing off
   to a standard HPA object it manages once above zero) and ships a large catalog of event-source
   scalers (queue depth, Kafka lag, cron, etc.) that plain HPA has no native concept of.
C. KEDA only works with Redis as an event source.
D. KEDA's sole function is providing GPU-utilization metrics; it has no queue-based scalers.

---

**Q7.** Which of the following is the clearest example of a workload that justifies a
**StatefulSet** rather than a Deployment?

A. A stateless FastAPI wrapper around a scikit-learn model with three interchangeable replicas.
B. A sharded, clustered vector database where each shard needs a stable network identity and its
   own persistent storage reattached correctly after a restart.
C. An API gateway that routes requests to multiple backend model services.
D. A batch-inference worker consuming from a queue, where any worker can pick up any message.

---

**Q8.** What does KServe's `InferenceService` give you over a hand-rolled Deployment + Service +
HPA that is most specific to model serving (select the single best answer)?

A. It removes the need for a container image entirely — models run directly as Kubernetes-native
   binaries.
B. Standardized multi-framework serving runtimes, a Predictor/Transformer/Explainer request
   pipeline, native canary traffic-percentage splitting, and (in serverless mode) concurrency-based
   autoscaling including scale-to-zero — all as declarative CRD fields.
C. It guarantees zero cold-start latency for every model regardless of size.
D. It is a drop-in replacement for Helm; you no longer need Helm once KServe is installed.

---

**Q9 (short answer).** Explain, referencing the ReplicaSet mechanism specifically, why
`kubectl rollout undo` works for a Deployment — what object is actually still sitting around making
that rollback possible?

---

**Q10 (short answer).** A teammate asks: "Our vector database runs as a StatefulSet with 3
replicas and persistent volumes — doesn't that mean it will always self-heal correctly after any
failure, the same way a Deployment does?" What is wrong with this framing? Be specific about what a
StatefulSet *does* guarantee versus what it does *not*.

---

**Q11 (short answer).** Contrast time-slicing and MIG as GPU-sharing strategies: what does each
actually share, what isolation guarantee (if any) does each provide, and for which class of
workload would you pick each?

---

**Q12 (short answer).** Why is a plain rolling update considered insufficient, on its own, as a
rollout strategy for a *model* version change specifically — as opposed to, say, a stateless
microservice with no ML-quality dimension? Name the specific kind of regression a rolling update's
readiness-gating cannot catch.

---

**Q13 (short answer).** Explain what an Argo Rollouts `AnalysisTemplate` does during a canary
rollout, and why "the rollback happens automatically, without a human watching a dashboard in real
time" is the actual point of the mechanism — not merely a nice-to-have.

---

**Q14 (short answer).** A stakeholder says "we should just run everything on Kubernetes since it's
the most powerful and flexible option." Using the criteria from the Kubernetes-vs-ECS-vs-serverless
comparison, give a concrete counter-scenario where Kubernetes would plausibly be the *wrong*
choice, and explain why.

---

**Q15 (short answer).** Your team's Helm chart currently hardcodes `image.tag: "1.4.2"` and
`replicaCount: 3` directly inside `templates/deployment.yaml`, with no `values.yaml` at all. What
specific capability does this design lose compared to a proper `values.yaml`-driven chart, and
what is the one-line `helm upgrade` command you would use once fixed to promote the exact same
chart to a `values-prod.yaml`-configured production namespace?

---
---

## Answer Key

**A1.** **B.** A Pod is the abstraction that guarantees co-scheduled containers land on the same
node and share a network namespace (and optionally storage) — a capability the container runtime
itself doesn't provide. This is what makes sidecar and init-container patterns (log shippers,
model-download init containers) possible without hand-built orchestration logic.

**A2.** **B.** Memory is an incompressible resource — the kernel cannot "slow down" memory
consumption, so exceeding the limit triggers an abrupt `OOMKilled` (SIGKILL) with no graceful
shutdown. CPU is compressible — the container is throttled (runs slower) but not killed. This
matters enormously for ML serving, where a single large batch or tokenizer call can spike RSS
momentarily; under-provisioned memory limits are a leading cause of "random" restarts in
production inference services.

**A3.** **B.** GPUs are advertised to the kubelet as extended resources by the device-plugin
framework (a DaemonSet, e.g., the NVIDIA device plugin), not via Kubernetes' native, natively
divisible CPU/memory resource model. The stock device plugin advertises whole GPU units only;
fractional requests are rejected. Oversubscribing a physical GPU requires an explicit sharing
mechanism layered on top — GPU Operator time-slicing or NVIDIA MIG.

**A4.** **B.** GPU-bound inference workloads routinely show low, flat CPU utilization even while
the GPU is fully saturated, because the GPU (not the CPU) is doing the compute-heavy work. Wiring
HPA to CPU in this situation produces an autoscaler that essentially never fires when it should —
the fix is scaling on GPU utilization (DCGM exporter → Prometheus → Prometheus Adapter → HPA
`External` metric) and/or queue depth/concurrency via KEDA.

**A5.** **C.** VPA and HPA should not both actively manage the same resource metric on the same
workload — VPA changing a Pod's requests/limits for a resource changes what "100% utilization" of
that resource even means, which can cause HPA to react to VPA's own changes and vice versa. The
safe pattern is either VPA-only (fixed replica count, right-sized Pods) or HPA on a different
metric while VPA runs in recommendation-only mode for the metric VPA would otherwise actively
manage.

**A6.** **B.** KEDA's two most relevant additions for this use case: scale-to-zero (impossible with
plain HPA, whose floor is 1 replica) via its own lightweight polling loop that decouples "is there
any work at all" from "how many replicas," and a large maintained catalog of event-source scalers
(queue depth, Kafka lag, cron, Prometheus queries, and more) that go well beyond CPU/memory. KEDA
does not replace HPA — it creates and manages a standard HPA object once replicas are above zero.

**A7.** **B.** A sharded/clustered vector database needs stable, predictable network identity per
shard (so peers/clients can address a *specific* shard) and stable storage reattached to the
correct ordinal after a restart — exactly the StatefulSet guarantee. The other three options
(stateless model wrapper, gateway, queue-consuming batch worker) all have interchangeable replicas
where "which specific instance" doesn't matter, making Deployment the correct primitive.

**A8.** **B.** KServe's value specific to model serving is standardizing the parts every serving
team otherwise re-solves per model: multi-framework runtime selection (point at a model artifact
and framework name rather than writing a custom server), a composable
Predictor/Transformer/Explainer pipeline, native canary traffic-percentage fields, and
concurrency-based autoscaling including scale-to-zero in serverless/Knative mode — all as
declarative CRD configuration.

**A9.** A Deployment doesn't manage Pods directly — it manages the Pod template through an
underlying **ReplicaSet**, and every time the Pod template changes (e.g., a new image tag), the
Deployment controller creates a *new* ReplicaSet and orchestrates the rollout between old and new
ReplicaSets. The *old* ReplicaSet is scaled down to 0 replicas but not deleted — it's still sitting
there as a rollback target, which is exactly what `kubectl rollout undo` scales back up when
invoked.

**A10.** A StatefulSet guarantees stable network identity per ordinal, stable storage (the same
PVC reattaches to the same ordinal after a reschedule), and ordered startup/shutdown — but it does
**not** automatically give a distributed system correct rejoin/resync/leader-election logic after
a failure. If a shard leader's Pod is killed and recreated, the StatefulSet correctly gives it back
the same identity and the same disk — but whether that node correctly rejoins the cluster,
resyncs any data it missed, or re-establishes its role is entirely the application's own
responsibility, not something the StatefulSet controller does for you.

**A11.** Time-slicing shares one physical GPU's compute round-robin across multiple Pods in
software (via the GPU Operator), with **no** memory isolation — Pods sharing the GPU can still
exhaust or corrupt each other's VRAM usage patterns (contention, not corruption, but no hard
boundary) — appropriate for dev/test and low-QPS, latency-tolerant endpoints where bin-packing
many small models matters more than guaranteed performance. MIG (on A100/H100-class hardware)
physically partitions a GPU into fixed-shape hardware instances with **real** isolation —
appropriate for multi-tenant production workloads that need guaranteed, isolated performance
across tenants or mixed workload sizes, at the cost of fixed partition shapes rather than
arbitrary flexible sharing ratios.

**A12.** A plain rolling update is readiness-gated on infrastructure health only (a
`readinessProbe` passing means the process responds, not that its predictions are correct) and has
no traffic-percentage control — by the time a regression is noticed, a large or full fraction of
the fleet may already be on the new version. The specific failure mode it cannot catch: a **silent
model-quality regression** — bad predictions, hallucinations, drifted confidence calibration — that
still returns HTTP 200 all day, because standard infra health checks have no visibility into
prediction quality at all.

**A13.** An `AnalysisTemplate` defines one or more metric queries (typically against Prometheus —
e.g., success rate, p99 latency) with a `successCondition` and a `failureLimit`, which Argo
Rollouts automatically evaluates after each canary step (e.g., after shifting to 10% traffic, pause
and query). If the query fails its condition enough times, Argo Rollouts automatically aborts the
rollout and reverts traffic to the old version — with no human needing to be watching a dashboard
in real time for that abort/rollback decision to happen. This converts canary analysis from "an
engineer stares at a graph and manually intervenes" into a genuine, unattended safety mechanism —
the entire point being that regressions get caught and reverted even during off-hours or when no
one happens to be looking.

**A14.** A concrete counter-scenario: a small team running one simple, low-traffic, CPU-only
model with no multi-cloud/portability requirement and no dedicated platform engineers. Here,
Kubernetes' operational floor (cluster/node-pool/upgrade ownership, the learning curve every
engineer touching it needs) is pure overhead relative to the value it provides — a serverless
option (Lambda, Cloud Run) or even ECS would deliver the same business outcome with dramatically
less operational burden and lower cost at that traffic level. The general principle: Kubernetes'
power is real, but the decision should be made per-workload against explicit criteria (GPU need,
traffic pattern/sustained-vs-spiky, portability need, team's platform-engineering capacity) — not
defaulted to "most capable tool" regardless of fit.

**A15.** Hardcoding values directly in `templates/deployment.yaml` loses environment
parameterization and promotion: there is no way to run the same chart with different replica
counts/image tags/resource sizes across dev/staging/prod without editing (and thereby forking) the
template itself, which is exactly the drift Helm's `values.yaml` model exists to prevent. Once
fixed (moving `image.tag` and `replicaCount` into `values.yaml` and templating them with
`{{ .Values.image.tag }}` / `{{ .Values.replicaCount }}`), the promotion command is:
`helm upgrade --install churn-model ./chart -f ./chart/values.yaml -f ./chart/values-prod.yaml --namespace ml-prod --create-namespace`.
