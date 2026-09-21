# Module 06 — Exercises: Kubernetes for ML and LLM Systems

These exercises assume you have `kubectl` and `helm` installed and a working cluster to point them
at — `kind`, `minikube`, or `k3d` are all fine for Exercises 1-6 and 8; a managed cluster with a
GPU node pool (or a willingness to do the reasoning-only variant) is needed for the GPU-flavored
parts of Exercise 4. Work through these in order — each one builds on manifests or mental models
produced by the one before it, mirroring how a real platform's YAML accretes over time rather than
being written once from scratch.

---

## Exercise 1 — Pods, Deployments, and Self-Healing, From First Principles

**Goal:** Build direct, hands-on intuition for "Pods are disposable" (Section 3.1) before touching
autoscaling or anything CRD-based.

**Tasks:**
1. Apply the beginner `deployment.yaml` + `Service` from Section 3.1 (a FastAPI/Flask stub is fine
   — reuse a model-serving image from Module 05 if you have one, or any `hashicorp/http-echo`-style
   placeholder that responds on `/health`).
2. Run `kubectl get pods -o wide` and note the Pod names and IPs. Delete one Pod directly:
   `kubectl delete pod <pod-name>`.
3. Immediately re-run `kubectl get pods -o wide`. Confirm a **new** Pod appeared with a **different**
   name and a **different** IP, and that the total replica count returned to the desired count
   within seconds.
4. Port-forward to the Service (not the Pod) — `kubectl port-forward svc/<service-name> 8080:80` —
   and confirm requests succeed continuously (poll every second in a loop) across the moment you
   delete another Pod. Note the brief blip, if any, and explain why the Service's DNS name never
   changed even though every Pod behind it did.
5. Write 2-3 sentences (using the words "ReplicaSet," "Deployment," and "Service") explaining why
   deleting a Pod directly is safe here in a way it would not be safe against a bare `docker run`
   container with no orchestrator watching it.

**Done when:** You have observed, in your own terminal output, a Pod's name/IP churn while client
traffic through the Service kept working — not just read about it.

---

## Exercise 2 — Ingress Routing and a Deliberately Broken Readiness Probe

**Goal:** Wire up Ingress path-based routing (Section 3.1) and directly observe why a Service only
routes to Pods that are `Ready`, not merely alive.

**Tasks:**
1. Install an Ingress controller in your cluster (e.g., `ingress-nginx` via its standard Helm
   chart) if one isn't already present.
2. Deploy two model Deployments + Services (`model-a`, `model-b`) and one `Ingress` object routing
   `/model-a` and `/model-b` to their respective Services, following the intermediate example in
   Section 3.1.
3. Confirm both paths resolve correctly through the Ingress controller's external address.
4. Now deliberately break `model-a`'s readiness: edit its readiness probe to point at a path that
   returns 500 (or patch the container to never mark itself ready). Apply the change and watch
   `kubectl get pods` — note the Pod stays `Running` but is never `Ready` (`1/1` never becomes
   `Ready` in the READY column).
5. Send requests through the Ingress to `/model-a` and confirm they now fail (503, or connection
   refused, depending on your controller) even though `kubectl get pods` shows the Pod as
   `Running`. Explain in one paragraph why "Running" and "Ready" are different states and why a
   Service/Ingress must gate on the latter.

**Done when:** You have reproduced the alive-but-not-ready gap yourself and can point to the exact
`kubectl get pods` output column and the exact HTTP behavior that demonstrates it.

---

## Exercise 3 — Requests, Limits, QoS Classes, and a Real OOMKill

**Goal:** Move "CPU is compressible, memory is not" (Section 3.2) from a rule you've read to a
failure you've caused and diagnosed yourself.

**Tasks:**
1. Deploy a small Python service whose `/predict` handler deliberately allocates a large, sized
   in-memory buffer per request (e.g., `bytearray(200_000_000)` — 200MB — held briefly before
   returning), simulating a memory-hungry batch/tokenizer step.
2. Set `resources.requests.memory: "128Mi"` and `resources.limits.memory: "256Mi"`. Deploy it.
3. Fire 5-10 concurrent requests at `/predict` (a simple `for i in {1..10}; do curl ... & done` loop
   is enough) and watch `kubectl get pods -w` in another terminal.
4. Confirm the Pod gets `OOMKilled` (check `kubectl describe pod <pod>` for
   `Last State: Terminated, Reason: OOMKilled`) and that this is an abrupt kill, not a graceful
   shutdown — no `preStop` hook runs, no graceful drain happens.
5. Fix it two different ways and confirm each independently resolves the crash: (a) raise the
   memory limit to a realistic peak (e.g., `512Mi`), and separately (b) reduce concurrency at the
   application layer (e.g., a semaphore limiting concurrent in-flight requests to 2). Record which
   fix you'd actually ship to production and why.
6. Redeploy with `requests == limits` for both CPU and memory and confirm via
   `kubectl get pod <pod> -o jsonpath='{.status.qosClass}'` that the Pod's QoS class is now
   `Guaranteed` rather than `Burstable`.

**Done when:** You have a `kubectl describe pod` output showing a real `OOMKilled` event you
caused, a fix that resolves it, and a confirmed QoS class change you triggered yourself.

---

## Exercise 4 — GPU Scheduling Reasoning: Fractional Requests, Device Plugins, and Sharing Strategies

**Goal:** Apply Section 3.2's GPU model even if you don't have physical GPU hardware to test
against — this is as much a reasoning exercise as a hands-on one.

**If you have access to a GPU node pool (cloud or on-prem) with the NVIDIA device plugin
installed:**
1. Confirm the node advertises the extended resource: `kubectl describe node <gpu-node> | grep -A2 nvidia.com/gpu`.
2. Deploy the intermediate `llama-7b-server`-style Deployment from Section 3.2 with
   `nvidia.com/gpu: 1` in `limits` only (no `requests` for the GPU key) and confirm via
   `kubectl get pod <pod> -o yaml` that Kubernetes auto-populated the matching `requests` entry.
3. Attempt to request a fractional GPU (`nvidia.com/gpu: "0.5"`) and confirm the Pod is rejected
   (or fails to schedule) — capture the exact error/event.
4. If your cluster has GPU Operator time-slicing configured (or you can configure it), apply a
   `time-slicing-config` ConfigMap like the production example, label a node with it, and confirm
   `kubectl describe node` now advertises more virtual `nvidia.com/gpu` units than physical GPUs
   exist. Schedule two small Pods each requesting `nvidia.com/gpu: 1` onto the same physical GPU
   and confirm both land successfully.

**If you do NOT have GPU hardware:**
1. Write out the exact `kubectl describe node` output you would expect to see on a healthy
   4-GPU A10G node with the GPU Operator installed (advertised resource name, node labels from
   Node Feature Discovery, and what the DCGM exporter would be scraping).
2. Write the exact Pod spec that would be **rejected** by the scheduler (a fractional GPU request)
   and explain, citing the device-plugin mechanism specifically, why it fails — contrast this with
   why the equivalent fractional CPU request (`cpu: "500m"`) succeeds.
3. Given a hypothetical mixed workload — one latency-SLA'd 13B-parameter LLM endpoint and four
   low-QPS, latency-tolerant small-model endpoints, all needing GPU access, on a 2-GPU node pool —
   design (in a short table or diagram) which sharing strategy (whole-GPU, time-slicing, MIG) you'd
   assign to each workload and justify each choice against the comparison table in Section 3.2.

**Done when:** Either you have real evidence (a rejected fractional-GPU event, a confirmed
time-sliced co-schedule) or a fully reasoned, correctly-justified written design for the mixed
workload scenario — a real interviewer would accept either as demonstrating mastery.

---

## Exercise 5 — Autoscaling on the Right Signal: HPA on CPU vs. KEDA on Queue Depth

**Goal:** Reproduce, yourself, the single most commonly tested autoscaling mistake in this module
(Section 3.3, Common Mistake #2): wiring HPA to a metric that doesn't reflect true load.

**Tasks:**
1. Deploy a CPU-light, "fake-GPU-bound" inference service — a Deployment whose handler does
   `time.sleep(1.5)` per request (simulating GPU-bound wait time with near-zero real CPU use) —
   with an HPA scaling on `type: Resource, name: cpu, averageUtilization: 65`.
2. Load-test it with enough concurrent requests to clearly saturate its ability to serve traffic
   (e.g., `hey -z 60s -c 30 http://.../predict` or a simple async client). Watch
   `kubectl get hpa -w` for the full 60+ seconds. Confirm the HPA does **not** scale up (or barely
   does), while your load-test client shows rising latency/timeouts.
3. Install KEDA (or Prometheus Adapter, if you want the harder path) in your cluster. Stand up a
   small RabbitMQ (or Redis-list) queue in front of a worker Deployment instead, following the
   production `ScaledObject` example in Section 3.3.
4. Push a burst of messages onto the queue and confirm the KEDA-managed HPA scales worker replicas
   up in response to queue depth, then confirm it scales back down to `minReplicaCount: 0` after
   the `cooldownPeriod` once the queue drains.
5. Write a short paragraph (this is the artifact a senior interview would ask you to produce
   verbally) explaining, using your own two experiments as evidence, why CPU was the wrong signal
   and queue depth was the right one for this workload shape.

**Done when:** You have two `kubectl get hpa -w` transcripts — one showing CPU-based autoscaling
failing to react, one showing KEDA/queue-depth-based autoscaling reacting correctly, including a
confirmed scale-to-zero — plus your own written explanation of why.

---

## Exercise 6 — StatefulSet for a Clustered Vector Store, and Breaking It on Purpose

**Goal:** Internalize what a StatefulSet actually guarantees (Section 3.4) — and, just as
importantly, what it does *not* — by deploying one and deliberately testing its boundaries.

**Tasks:**
1. Deploy the 3-replica StatefulSet + headless Service + `volumeClaimTemplates` example for a
   vector database (Qdrant or similar) from Section 3.4.
2. Confirm each Pod has a stable, predictable name (`vectordb-0`, `-1`, `-2`) and that each is
   individually addressable via
   `vectordb-0.vectordb-svc.<namespace>.svc.cluster.local` (verify with a `kubectl run --rm -it
   dnsutils` throwaway Pod doing `nslookup`).
3. Write a small amount of distinguishable data into `vectordb-0` specifically (a collection/point
   only that ordinal holds, if your chosen vector DB supports single-node writes easily — otherwise
   write a marker file into its PVC-backed directory via `kubectl exec`).
4. Delete Pod `vectordb-0` (`kubectl delete pod vectordb-0`) and watch it get recreated. Confirm
   (a) the new Pod is named `vectordb-0` again (not a random suffix), and (b) the data you wrote
   is still present — because the same PVC reattached to the same ordinal.
5. Now break the "ordering" guarantee's limits: scale the StatefulSet down to 1 replica, then back
   up to 3, and watch `kubectl get pods -w` to confirm Pod-0 becomes Ready before Pod-1 is even
   created (ordered startup) — then explain in writing why this ordering guarantee alone does
   *not* mean your vector DB cluster is actually healthy/rebalanced after this operation (tie back
   to Common Mistake #5 — StatefulSet gives identity and storage, not application-level
   rejoin/resync logic).

**Done when:** You've proven PVC reattachment to the correct ordinal after a Pod deletion with your
own `kubectl exec` evidence, and you've written a clear explanation of the identity-vs-correctness
gap using your own scale-down/scale-up observation as the example.

---

## Exercise 7 — Deploy a Model via KServe `InferenceService`, Then Canary a New Version

**Goal:** Move from hand-rolled Deployment+Service+HPA to KServe's standardized serving layer
(Section 3.5) and exercise its native canary field end to end.

**Tasks:**
1. Install KServe (standalone install mode is fine — you do not need the rest of Kubeflow) into
   your cluster, following its standard quickstart.
2. Deploy the beginner `InferenceService` example for a scikit-learn model pointed at a model
   artifact in object storage (a local MinIO instance is a fine substitute for GCS/S3 for this
   exercise). Confirm you can send a prediction request to the resulting endpoint and get a
   correct response.
3. Build and push a "v2" of the same model (retrain trivially — e.g., change a hyperparameter —
   so the new version's predictions are provably different from v1's on a fixed test input).
4. Apply the production-grade canary example from Section 3.5: set `canaryTrafficPercent: 10`
   pointing at the v2 image/artifact. Send a burst of ~100 requests through the `InferenceService`
   and tally how many responses come from v1 vs. v2 (add a version marker to each model's response
   payload to make this observable). Confirm the observed split is roughly 90/10.
5. Promote v2 fully (`canaryTrafficPercent: 100`, or remove the field and make v2 `default`) and
   confirm 100% of subsequent requests now come from v2. Then simulate a rollback by reverting the
   image reference and confirm traffic returns to v1 — time how long each of these operations
   takes end-to-end.

**Done when:** You have a recorded observed traffic split close to 90/10 during the canary step,
proof of full promotion, and proof of a successful rollback — all against a real KServe
`InferenceService`, not a simulated one.

---

## Exercise 8 (Capstone) — Helm-Packaged, Autoscaled, Canary-Gated Model-Serving Platform

**Goal:** Integrate everything in this module into one coherent, environment-promotable,
progressively-delivered deliverable — the natural artifact a platform team would actually run, and
the bridge into Module 07/08's observability material and Module 09's CI/CD material.

**Tasks:**
1. Package a model-serving Deployment (or `InferenceService`, your choice — justify it in your
   writeup) as a Helm chart with `Chart.yaml`, a `values.yaml` with sensible defaults, and at
   least `deployment.yaml`, `service.yaml`, and `hpa.yaml` templates, following Section 3.6's
   layout. Add a `gpu.enabled` toggle (even if you don't have real GPU hardware — the template
   logic is what's being tested) that conditionally adds `nvidia.com/gpu` to `limits`.
2. Create `values-dev.yaml` and `values-prod.yaml` overrides (different replica counts, resource
   sizes, autoscaling bounds) and demonstrate promoting the *same* chart across two namespaces
   (`ml-dev`, `ml-prod`) via `helm upgrade --install` with different `-f` files. Confirm
   `helm list -A` shows both releases independently versioned.
3. Add a `PodDisruptionBudget` and `podAntiAffinity` to the chart's Deployment template (Section
   3.1's production example), parameterized via `values.yaml` so they can be disabled for the dev
   environment but required for prod.
4. Install Argo Rollouts and convert the prod release's Deployment into a `Rollout` with the
   canary-steps example from Section 3.7, including at least one `AnalysisTemplate` querying
   Prometheus for a success-rate or latency threshold (install `kube-prometheus-stack` if you
   don't already have Prometheus running, and generate a synthetic error-rate metric you can
   control for testing).
5. Trigger a rollout by bumping the chart's `image.tag` value and running `helm upgrade`. Watch
   `kubectl argo rollouts get rollout <name> --watch` progress through the canary steps. Then,
   deliberately make your synthetic error-rate metric fail the `AnalysisTemplate`'s threshold
   mid-rollout and confirm Argo Rollouts **automatically aborts and rolls back** without any manual
   `kubectl rollout undo` from you.
6. Write a one-page runbook (mirroring the production checklist in Section 8 of `tutorial.md`) with
   every box checked against your actual stack, plus a short paragraph justifying, against Section
   3.8's decision tree, why this workload belongs on Kubernetes rather than ECS or serverless.

**Done when:** You have two independently-versioned Helm releases (dev/prod) from one chart, a
working Argo Rollouts canary that progressed on good metrics in one run, a second run where you
proved the automated abort-and-rollback path fires on a deliberately bad metric with no manual
intervention, and a completed runbook — this artifact is what you'd carry into a review with a
platform team lead.
