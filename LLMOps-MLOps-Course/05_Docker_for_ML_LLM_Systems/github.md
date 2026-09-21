# GitHub Repositories — Docker for ML and LLM Systems

Real, verifiable repositories relevant to containerizing ML and LLM services. Popularity is described in tiers rather than fabricated star counts (exact counts drift constantly and were not independently re-verified at write time).

---

## Core tooling repositories

### `vllm-project/vllm`
- **URL:** https://github.com/vllm-project/vllm
- **Purpose:** The vLLM inference and serving engine itself — the reference implementation this chapter uses as its "LLM-serving service" packaging example. Ships an official Docker image (`vllm/vllm-openai` on Docker Hub) and includes a `docker/` directory with the actual production Dockerfile used to build it, plus an `examples/observability/prometheus_grafana/` directory with a working docker-compose setup for metrics.
- **Popularity tier:** Very popular / widely adopted — one of the de facto standard open-source LLM inference servers used across industry as of 2026.
- **Why it matters:** Rather than inventing a toy Dockerfile for LLM serving, this repo's actual `docker/Dockerfile` and observability compose example are real, production-grade references for GPU base images, multi-stage build layout for a CUDA + Python + large-dependency-tree service, and Prometheus/Grafana wiring for LLM-serving metrics (TTFT, tokens/sec, queue depth).
- **Relation to this module:** Primary source for the "packaging an LLM-serving service in Docker" and "Docker Compose for local multi-service dev" sections.

### `anchore/grype`
- **URL:** https://github.com/anchore/grype
- **Purpose:** Open-source vulnerability scanner for container images, filesystems, and SBOMs, maintained by Anchore.
- **Popularity tier:** Very popular / widely adopted — a standard choice (alongside Trivy) for container image CVE scanning in CI/CD pipelines.
- **Why it matters:** Supports 30+ package ecosystems (Python, Java, JS, Go, OS packages, and more), OpenVEX/CSAF VEX filtering to suppress known-irrelevant findings, and drop-in CI integration (CLI or GitHub Action).
- **Relation to this module:** Primary reference implementation for the "security scanning (Trivy/Grype)" section — used here as the second, comparison tool alongside Trivy so learners see that image scanning is a category with more than one production-grade option, not a single-vendor lock-in.

### `aquasecurity/trivy` (referenced via official docs at trivy.dev; GitHub org: aquasecurity)
- **Purpose:** All-in-one vulnerability, misconfiguration, secret, and SBOM scanner for container images, IaC, and Kubernetes manifests, maintained by Aqua Security.
- **Popularity tier:** Very popular / widely adopted — commonly the default choice referenced in Docker's and GitHub Actions' own security-scanning documentation and examples.
- **Why it matters:** Ships as a single static binary with no separate database service to run; auto-downloads and caches its vulnerability DB; scans images, filesystems, git repos, and running clusters with one consistent tool.
- **Relation to this module:** Primary reference implementation for the "security scanning" section's Trivy examples (`trivy image myimage:tag`, CI gating on severity thresholds).

### `bentoml/BentoML`
- **URL:** https://github.com/bentoml/BentoML
- **Purpose:** A Python framework for packaging trained ML models (traditional ML and LLM) into production-ready inference APIs, with built-in Docker image generation ("Bento" build artifacts that compile to a Dockerfile/image).
- **Popularity tier:** Very popular / widely adopted in the ML model-serving space.
- **Why it matters:** Demonstrates an alternative, higher-level pattern to hand-writing a Dockerfile for an ML service — BentoML auto-generates a reproducible, dependency-pinned Docker image from a model + service definition, which is a useful contrast to the "roll your own Dockerfile" approach this chapter teaches by hand.
- **Relation to this module:** Complementary reference for the "packaging a Python ML service" section — shown as the "framework does it for you" alternative to the manual multi-stage Dockerfile pattern taught in this chapter.

---

## Reference / example compose stacks

### `qdrant/qdrant` and `qdrant/demo-distributed-deployment-docker`
- **URLs:** https://github.com/qdrant/qdrant · https://github.com/qdrant/demo-distributed-deployment-docker
- **Purpose:** Qdrant is a popular open-source vector database; the demo repo shows a real, working docker-compose setup for running Qdrant (including distributed/clustered mode).
- **Popularity tier:** Qdrant itself is very popular / widely adopted as a vector DB choice for RAG systems; the demo repo is a small, official companion example.
- **Why it matters:** Gives a verified, real docker-compose service definition for a vector database (image name, ports, volume mounts, cluster env vars) rather than an invented one — directly usable as the "vector DB" service in this chapter's multi-service Compose stack.
- **Relation to this module:** Source reference for the vector-DB service block in the chapter's `docker-compose.yml` example.

### `qdrant/prometheus-monitoring`
- **URL:** https://github.com/qdrant/prometheus-monitoring
- **Purpose:** A minimal, official example of wiring Prometheus + Grafana to monitor a Qdrant cluster.
- **Popularity tier:** Small/official example repo (not meant to be a widely-starred project, but authoritative as an official Qdrant example).
- **Why it matters:** A concrete, real Prometheus scrape-config and Grafana dashboard pairing for a vector DB, useful as a template for the monitoring-stack part of the Compose example.
- **Relation to this module:** Reference for the "monitoring stack" portion of the Docker Compose local-dev section.

### `DataTalksClub/mlops-zoomcamp`
- **URL:** https://github.com/DataTalksClub/mlops-zoomcamp
- **Purpose:** Full course materials (notebooks, slides, code) for the free, project-based MLOps Zoomcamp, including modules that containerize a trained model behind a web service as part of the deployment unit.
- **Popularity tier:** Very popular / widely adopted within the MLOps learning community.
- **Why it matters:** Real, complete, community-vetted example code showing Docker used in an end-to-end ML deployment context (not an isolated Docker demo), including how a Dockerized model service fits alongside experiment tracking and orchestration tooling.
- **Relation to this module:** Cross-reference for "packaging a Python ML service" — shows the pattern in the context of a full MLOps pipeline, tying this module back to concepts from earlier in the textbook (experiment tracking, model registries).

### `veggiemonk/awesome-docker`
- **URL:** https://github.com/veggiemonk/awesome-docker
- **Purpose:** A curated "awesome list" of Docker tools, tutorials, and projects.
- **Popularity tier:** Very popular / widely adopted as the canonical "awesome-docker" list (the original that most forks/derivatives are based on).
- **Why it matters:** Useful as a living index to discover current tools (linters, compose helpers, image-size analyzers like `dive`) that move faster than any static textbook chapter can track.
- **Relation to this module:** General-purpose supplementary index for readers who want to go beyond this chapter's curated tool selection (Trivy, Grype, BuildKit) into the broader Docker tooling ecosystem.

### `ajeetraina/awesome-docker-ai-lists`
- **URL:** https://github.com/ajeetraina/awesome-docker-ai-lists
- **Purpose:** A curated collection specifically of Docker resources, tools, and use cases for AI/ML workloads.
- **Popularity tier:** Smaller, newer curated list (niche but directly on-topic).
- **Why it matters:** Narrower and more current than the general awesome-docker list for readers specifically working the AI/ML angle of this module (GPU images, model-serving containers, ML-specific Compose patterns).
- **Relation to this module:** Supplementary index specifically for the ML/LLM angle of this chapter.

---

## Tool worth knowing about

### `wagoodman/dive`
- **URL:** https://github.com/wagoodman/dive
- **Purpose:** A CLI tool for exploring a Docker/OCI image layer-by-layer, showing per-layer filesystem contents, an efficiency score, and total wasted space from duplicated or shadowed files across layers. Supports `dive build -t tag .` to build-and-analyze in one step, and a `CI=true` mode that pass/fails a build based on an efficiency threshold.
- **Popularity tier:** Very popular / widely adopted as the standard tool for visually inspecting where Docker image bloat comes from.
- **Why it matters:** Directly actionable for the "image size optimization" section — rather than guessing which `RUN`/`COPY` instruction bloated an image, `dive` shows exactly which layer and which files did it.
- **Relation to this module:** Recommended hands-on tool for the image-size-optimization exercise in this chapter.
