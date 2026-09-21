# References — Docker for ML and LLM Systems

Official documentation, engineering resources, and specifications used as the basis for this chapter. Ordered roughly by how the chapter uses them.

---

## Docker core concepts and build system

### Dockerfile reference
- **URL:** https://docs.docker.com/reference/dockerfile/
- **Publisher:** Docker, Inc. (official)
- **What it teaches:** The complete, authoritative syntax for every Dockerfile instruction (`FROM`, `RUN`, `COPY`, `ADD`, `ENV`, `ARG`, `ENTRYPOINT` vs `CMD`, `HEALTHCHECK`, `USER`, etc.), including current BuildKit-specific syntax extensions.
- **Difficulty / reading time:** Beginner–Intermediate reference; 30–45 min to skim fully, used ongoing as a lookup.

### Dockerfile overview (build concepts)
- **URL:** https://docs.docker.com/build/concepts/dockerfile/
- **Publisher:** Docker, Inc. (official)
- **What it teaches:** Conceptual framing of what a Dockerfile *is* to BuildKit — the instruction-to-layer mapping, build context, and how caching keys are derived from instructions and their inputs.
- **Difficulty / reading time:** Beginner; 15–20 min.

### Building best practices
- **URL:** https://docs.docker.com/build/building/best-practices/
- **Publisher:** Docker, Inc. (official)
- **What it teaches:** The canonical best-practices list this chapter draws heavily from — choosing minimal/appropriate base images, ordering instructions to maximize cache hits, using multi-stage builds to keep build tools out of the final image, and minimizing layer count via combined `RUN` commands.
- **Difficulty / reading time:** Beginner–Intermediate; 20–30 min. Essential reading before writing production Dockerfiles.

### Image-building best practices (interactive workshop)
- **URL:** https://docs.docker.com/get-started/workshop/09_image_best/
- **Publisher:** Docker, Inc. (official, part of the "Get Started" workshop track)
- **What it teaches:** A hands-on walkthrough of restructuring a Dockerfile to improve cache behavior — e.g., copying dependency manifests (`package.json`, `requirements.txt`) before copying full source, so dependency-install layers aren't invalidated by every source change.
- **Difficulty / reading time:** Beginner; 20 min, hands-on.

### Compose file reference (Compose Specification)
- **URL:** https://docs.docker.com/reference/compose-file/
- **Publisher:** Docker, Inc. (official)
- **What it teaches:** The unified Compose Specification (superseding the old v2/v3 file-format-version split) covering `services`, `networks`, `volumes`, `depends_on` with healthcheck conditions, `build` blocks, and environment/secret handling.
- **Difficulty / reading time:** Intermediate; 45–60 min for the sections used in this chapter (services, build, healthcheck-gated `depends_on`).
- **Mid-2026 note:** Compose v2 (the `docker compose` subcommand, built into the Docker CLI) is standard; the legacy standalone Python `docker-compose` v1 binary is deprecated and should not appear in new material.

### Compose Build Specification
- **URL:** https://docs.docker.com/reference/compose-file/build/
- **Publisher:** Docker, Inc. (official)
- **What it teaches:** How to define per-service build context, Dockerfile path, build args, and target stage (for multi-stage builds) directly inside a `docker-compose.yml`, so `docker compose up --build` can build and run in one step.
- **Difficulty / reading time:** Intermediate; 15 min.

---

## GPU-enabled containers

### NVIDIA Container Toolkit — installation guide
- **URL:** https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/
- **Publisher:** NVIDIA (official)
- **What it teaches:** How to install and configure the NVIDIA Container Toolkit (the modern successor to the older `nvidia-docker2` package) so Docker, containerd, or Podman can expose host GPUs to containers; covers the `nvidia` runtime, the `--gpus` flag, and the CDI (Container Device Interface) configuration path.
- **Difficulty / reading time:** Intermediate; 30–45 min for install + verification.
- **Note:** As of Docker Engine 19.03+, GPUs are natively supported as first-class devices via `docker run --gpus`, removing the need for the old `nvidia-docker` wrapper CLI — a detail worth calling out explicitly since older tutorials/blog posts still reference `nvidia-docker run`.

### Running a sample workload (NVIDIA Container Toolkit)
- **URL:** https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/sample-workload.html
- **Publisher:** NVIDIA (official)
- **What it teaches:** A minimal end-to-end verification recipe (run `nvidia-smi` inside a container) to confirm the toolkit + driver + runtime chain is correctly wired before layering ML frameworks on top.
- **Difficulty / reading time:** Beginner; 10 min, hands-on.

### `nvidia/cuda` Docker Hub image
- **URL:** https://hub.docker.com/r/nvidia/cuda
- **Publisher:** NVIDIA (official)
- **What it teaches:** The tagging scheme for official CUDA base images (CUDA version, cuDNN inclusion, `base`/`runtime`/`devel` variants, OS base), which directly determines final image size and what's available at build time vs. runtime — the `devel` variant carries the full CUDA toolkit/compiler and is appropriate only for the build stage of a multi-stage Dockerfile, while `runtime` or `base` variants belong in the final stage.
- **Difficulty / reading time:** Intermediate; 15–20 min to understand the tag taxonomy.

### Containers For Deep Learning Frameworks User Guide
- **URL:** https://docs.nvidia.com/deeplearning/frameworks/user-guide/index.html
- **Publisher:** NVIDIA (official)
- **What it teaches:** How NVIDIA's own NGC container catalog packages deep learning frameworks (PyTorch, TensorFlow, Triton) with pre-integrated, version-matched CUDA/cuDNN stacks — useful as a reference for what a well-engineered GPU ML image looks like, and as an alternative to building your own CUDA base image from scratch.
- **Difficulty / reading time:** Advanced; 30–40 min for the relevant sections.

---

## LLM serving

### vLLM — Using Docker (official docs)
- **URL:** https://docs.vllm.ai/en/latest/deployment/docker/
- **Publisher:** vLLM project (official)
- **What it teaches:** The official, maintained Docker deployment workflow for vLLM — the pre-built `vllm/vllm-openai` image, the `--build-arg VLLM_USE_PRECOMPILED=1` flag to speed up custom builds, GPU exposure via `--gpus all`, HuggingFace cache mounting, shared-memory sizing (`--ipc=host` or `--shm-size`) for multi-process tensor-parallel workers, and ARM64 build support.
- **Difficulty / reading time:** Intermediate; 30–40 min. This is the primary source for the "LLM-serving service in Docker" section of this chapter — always check the `latest` (not a pinned old version) docs page, since vLLM's Docker workflow has changed materially release over release.

### vLLM — Prometheus & Grafana observability example
- **URL:** https://github.com/vllm-project/vllm/blob/main/examples/observability/prometheus_grafana/README.md
- **Publisher:** vLLM project (official)
- **What it teaches:** A real, working example wiring vLLM's built-in Prometheus metrics endpoint (request latency, TTFT, throughput, KV-cache utilization) to a Prometheus + Grafana stack via Docker Compose.
- **Difficulty / reading time:** Intermediate; 20–30 min, hands-on.
- **Relation to this module:** Direct source reference for the monitoring-stack portion of this chapter's Compose example.

---

## Security scanning

### Trivy documentation — Container Image scanning
- **URL:** https://trivy.dev/docs/latest/guide/target/container_image/
- **Publisher:** Aqua Security (official, Trivy is a CNCF-adjacent open-source project)
- **What it teaches:** How to scan a built image for OS-package and language-dependency CVEs, generate an SBOM, and gate CI on a severity threshold (e.g., fail build on any CRITICAL/HIGH finding). Also covers secret and misconfiguration scanning within the same tool.
- **Difficulty / reading time:** Beginner–Intermediate; 20–30 min.

### Grype (Anchore) — official repository and docs
- **URL:** https://github.com/anchore/grype
- **Publisher:** Anchore (official)
- **What it teaches:** Same category as Trivy (CVE scanning for images/filesystems/SBOMs) with broader explicit VEX (OpenVEX/CSAF) support for suppressing/annotating known-irrelevant findings — relevant when a scanner flags a CVE in a code path your service never exercises.
- **Difficulty / reading time:** Intermediate; 20 min.
- **Why cover both Trivy and Grype:** They're the two most commonly deployed open-source scanners in production CI pipelines, and interviewers/production teams expect familiarity with the *category* (SBOM-based CVE scanning gated in CI) more than a specific tool preference.

---

## MLOps context (why this matters beyond Docker itself)

### Designing Machine Learning Systems, Chapter 10 ("Infrastructure and Tooling for MLOps")
- **Author:** Chip Huyen (O'Reilly, 2022)
- **What it teaches:** Where containerization sits in the broader MLOps infrastructure stack, and the gap between "the model is containerized" and "the model is served reliably in production."
- **Difficulty / reading time:** Intermediate; 45–60 min for the chapter.
- (Full citation and edition detail in `books.md`.)

### MLOps Zoomcamp (DataTalksClub)
- **URL:** https://github.com/DataTalksClub/mlops-zoomcamp
- **Publisher:** DataTalks.Club (free, open community course)
- **What it teaches:** End-to-end, project-based use of Docker as one piece of a full ML deployment pipeline (alongside experiment tracking, orchestration, and monitoring), rather than Docker in isolation.
- **Difficulty / reading time:** Intermediate; the Docker-relevant modules take 2–3 hours hands-on.

---

## Mid-2026 currency notes

- **BuildKit** is the default Docker build backend (has been since Docker Engine 23+); legacy builder syntax/quirks referenced in pre-2022 material are no longer the default path.
- **Compose v2** (`docker compose`, integrated into the Docker CLI as a plugin) is standard; the standalone Python `docker-compose` v1 binary is legacy and should be avoided in new tutorials/examples.
- **`nvidia-container-toolkit`** is the current, correct package name; `nvidia-docker` / `nvidia-docker2` are the deprecated predecessor names still seen in older blog posts and Stack Overflow answers.
- For **vLLM**, always consult the `latest` docs branch (docs.vllm.ai/en/latest/...) rather than a version-pinned URL when writing new material — the Docker deployment flags and precompiled-wheel build-arg support have changed across recent releases.
- **CVE scanning in CI** (Trivy/Grype gating on severity thresholds) is now considered a baseline expectation for production ML/LLM image pipelines, not an optional hardening step — this is worth stating explicitly in the chapter since some 2023-era MLOps material still treats it as optional.
