# Videos — Module 06: Kubernetes for ML and LLM Systems

Curated, verified videos. Each entry lists what it complements in the chapter (pods/services/ingress, GPU scheduling, autoscaling, KServe/Kubeflow, Helm, rollout strategies). Durations are only given where they could be reasonably confirmed from listing metadata; otherwise omitted rather than guessed.

---

## Foundations: Pods, Deployments, Services, Ingress

### ★★★★★ Kubernetes Tutorial for Beginners [Full Course in 4 Hours]
- **Creator/Channel:** TechWorld with Nana
- **Duration:** ~4 hours
- **Difficulty:** Beginner
- **Why it's worth watching:** This is the most widely recommended free Kubernetes course on YouTube. It walks through the core object model (Pods, Deployments, Services, Ingress, Namespaces) with a real Minikube cluster, then continues into ConfigMaps/Secrets, Helm, and StatefulSets (deploying a stateful MongoDB app) — which maps almost one-to-one onto this chapter's structure.
- **Complements:** The "Kubernetes fundamentals" section (pods/deployments/services/ingress), the Helm section, and the StatefulSets section.

### ★★★★☆ Kubernetes Crash Course for Absolute Beginners
- **Creator/Channel:** TechWorld with Nana
- **Duration:** ~1 hour
- **Difficulty:** Beginner
- **Why it's worth watching:** A shorter, denser version of the above for readers who already know Docker and just need the Kubernetes object model refreshed before jumping into GPU/ML-specific material. Good as a fast primer if you don't have 4 hours.
- **Complements:** Opening section on pods/deployments/services architecture.

---

## GPU Scheduling and Device Plugins

### ★★★★☆ Nvidia GPUs On Kubernetes
- **Creator/Channel:** DevOps Toolkit (Viktor Farcic)
- **Difficulty:** Intermediate
- **Why it's worth watching:** Viktor Farcic is a CNCF Ambassador and one of the most trusted voices for practical, demo-driven Kubernetes operations content. This video walks through actually getting NVIDIA GPUs recognized and scheduled inside a Kubernetes cluster (device plugin registration, `nvidia.com/gpu` extended resource, node capacity), which is exactly the mechanism this chapter's GPU scheduling section explains conceptually.
- **Complements:** The "resource requests/limits and GPU scheduling (nvidia device plugin)" section — watch after reading the manifest examples to see them applied live.

### ★★★☆☆ GPUs in Kubernetes the Easy Way? NVIDIA GPU Operator Overview
- **Creator/Channel:** DevOps Toolkit (Viktor Farcic)
- **Difficulty:** Intermediate
- **Why it's worth watching:** Covers the GPU Operator (which bundles the device plugin, driver installation, DCGM monitoring, and node feature discovery into a single Helm-installable operator) as the production alternative to hand-installing the bare device plugin. Useful for contrasting "device plugin only" vs. "full GPU Operator" approaches discussed in the chapter.
- **Complements:** The GPU scheduling section's discussion of the NVIDIA GPU Operator vs. the raw device plugin DaemonSet.

### ★★★★☆ When Kubeflow Meets Cilium: Debugging 60% Idle GPUs in Kubernetes
- **Creator/Channel:** CNCF (Cloud Native Computing Foundation) — KubeCon + CloudNativeCon India 2026 session, referenced on the CNCF blog
- **Difficulty:** Advanced
- **Why it's worth watching:** A real production war story about diagnosing GPU underutilization caused by networking/scheduling interactions between Kubeflow and Cilium. This is the kind of "GPUs are scheduled but sitting idle" failure mode that senior MLOps engineers get interview-quizzed on, and it grounds the abstract scheduling material in an actual incident.
- **Complements:** The GPU scheduling section, specifically the discussion of why "pod is Running" does not mean "GPU is being used efficiently."

---

## Autoscaling: HPA / VPA / KEDA

### ★★★★★ Argo Rollouts - Canary Deployments Made Easy in Kubernetes
- **Creator/Channel:** DevOps Toolkit (Viktor Farcic)
- **Difficulty:** Intermediate
- **Why it's worth watching:** A hands-on walkthrough of Argo Rollouts implementing canary releases as a Kubernetes CRD layered on top of a normal Deployment — showing traffic-shifting, analysis steps, and automated rollback. This is the deepest practical treatment of the "canary/blue-green deployments on K8s" part of the chapter available on YouTube from a reliable source.
- **Complements:** The "rolling updates and canary/blue-green deployments" section.

### ★★★★☆ KServe (Kubeflow KFServing) Live Coding Session
- **Creator/Channel:** MLOps Community (MLOps Meetup #83), speaker Theofilos Papapanagiotou
- **Difficulty:** Intermediate
- **Why it's worth watching:** A live-coded deployment of a model via KServe's InferenceService, including autoscaling behavior (scale-to-zero, concurrency-based scaling) demonstrated interactively rather than just described in slides.
- **Complements:** The KServe section, specifically the InferenceService YAML manifest walkthrough and its built-in HPA/Knative-based autoscaling.

### ★★★☆☆ Serverless Machine Learning Model Inference on Kubernetes with KServe
- **Creator/Channel:** CNCF / conference talk by Stavros Kontopoulos
- **Difficulty:** Intermediate
- **Why it's worth watching:** Explains the "serverless" scale-to-zero and request-based autoscaling model that KServe layers on top of Knative Serving, and why that differs from a plain HPA on CPU. Directly useful for understanding when KServe's built-in autoscaler is preferable to hand-rolled KEDA ScaledObjects.
- **Complements:** The KServe section and the HPA/KEDA comparison discussion.

### ★★★★☆ Serving Machine Learning Models at Scale Using KServe
- **Creator/Channel:** Bloomberg Engineering (conference talk, Yuzhui Liu)
- **Difficulty:** Advanced
- **Why it's worth watching:** A large-scale production perspective (Bloomberg) on operating KServe at scale — multi-model serving, resource isolation, and operational lessons learned, useful context beyond the "hello world" tutorials.
- **Complements:** The KServe production patterns and multi-model serving discussion.

---

## Kubeflow

### ★★★☆☆ Kubeflow Essentials 7-1: KServe (Get Started)
- **Creator/Channel:** Kubeflow community / YouTube
- **Difficulty:** Beginner-Intermediate
- **Why it's worth watching:** Part of a structured "Kubeflow Essentials" playlist that treats KServe as a Kubeflow component (rather than standalone), showing how model serving fits into the wider Kubeflow Pipelines → Katib → KServe lifecycle this chapter references.
- **Complements:** The Kubeflow section's explanation of how KServe fits into the broader Kubeflow platform.

---

## Conference Context (2025-2026 State of the Art)

### ★★★★☆ LLM-D: Multi-Accelerator LLM Inference on Kubernetes
- **Creator/Channel:** KubeCon + CloudNativeCon North America 2025 (Red Hat, Erwan Gallen)
- **Difficulty:** Advanced
- **Why it's worth watching:** llm-d is the newest (2026, now CNCF Sandbox) Kubernetes-native distributed inference stack, built on vLLM, for multi-accelerator LLM serving with disaggregated prefill/decode. This is genuinely new ground beyond what a 2024-era Kubernetes-for-ML course would cover, and it's the direction the ecosystem is moving for large-model serving at scale — worth knowing about even though this chapter's hands-on examples focus on KServe.
- **Complements:** A forward-looking coda to the KServe section — frame it as "KServe/Triton for general model serving today; llm-d for frontier-scale disaggregated LLM inference as the ecosystem matures."

---

## Notes on selection

- All entries above were verified to exist via search; no durations were invented for videos where an exact runtime could not be confirmed.
- Prioritized channels with an established track record on Kubernetes operational content (TechWorld with Nana, DevOps Toolkit/Viktor Farcic — a CNCF Ambassador) and official conference recordings (CNCF, KubeCon, MLOps Community) over generic tutorial-mill channels.
- If a specific video becomes unavailable, search the channel names above directly — they consistently produce equivalent-quality replacement content on the same topics.
