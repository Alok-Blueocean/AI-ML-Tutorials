# Module 05 — Docker for ML and LLM Systems

> "Works on my machine" is not a deployment strategy. In production MLOps/LLMOps, the container *is* the unit of deployment, the unit of scaling, and — increasingly — the unit of security audit. If you cannot reason precisely about what is inside your image, you cannot reason about what is running in production.

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Explain the difference between a Docker **image** and a **container**, and why that distinction matters for reproducible ML training and serving.
2. Read and write a Dockerfile using **layers**, the **build cache**, and **multi-stage builds** to produce small, fast, reproducible images for Python ML services.
3. Build and run **GPU-enabled containers** using the NVIDIA Container Toolkit and the correct CUDA base image variant (`base` / `runtime` / `devel`) for each build stage.
4. Package a **FastAPI-based model-serving microservice** and an **LLM-serving service** (vLLM) into production-grade Docker images.
5. Compose a **local multi-service ML/LLM stack** — API, vector database, Redis cache, and a Prometheus/Grafana monitoring stack — using Docker Compose v2.
6. Apply **image size optimization** techniques (multi-stage builds, layer ordering, `.dockerignore`, slim/distroless bases).
7. Run **container vulnerability scanning** (Trivy, Grype) as a gating step in CI, and interpret severity thresholds.
8. Articulate how containerization decisions made here directly determine what is *possible* one layer up, in Kubernetes (Module 06).

### Prerequisites

- Comfortable with Python (virtual environments, `pip`, packaging a small web service).
- Basic Linux command line (paths, permissions, processes).
- Familiarity with what a REST API is (we will containerize a FastAPI service).
- Module 04 concepts: you should already know *why* MLOps needs reproducible environments; this module makes that reproducibility concrete.
- No prior Docker experience is assumed — but if you have used `docker run hello-world` before, you'll move faster through Section 3.1.

### Key Terminology

| Term | Definition |
|---|---|
| **Image** | An immutable, layered, read-only filesystem snapshot plus metadata (entrypoint, env vars, exposed ports) used as a template to create containers. |
| **Container** | A running (or stopped) instance of an image — the image's filesystem plus a thin writable layer, isolated via Linux namespaces and cgroups. |
| **Layer** | A single filesystem diff produced by one Dockerfile instruction (mostly `RUN`, `COPY`, `ADD`); layers are content-addressed and cached/shared across images. |
| **Dockerfile** | The declarative build recipe: an ordered list of instructions that BuildKit executes to produce an image. |
| **BuildKit** | Docker's modern build engine (default since Docker Engine 23+) — parallelizes independent build stages, supports better caching, secrets, and SBOM/provenance attestations. |
| **Multi-stage build** | A Dockerfile with more than one `FROM`, where later stages selectively `COPY --from=<stage>` artifacts from earlier stages, discarding build-time tooling from the final image. |
| **Registry** | A server (Docker Hub, GHCR, ECR, GCR, a private Harbor instance) that stores and serves images by name:tag and digest. |
| **Tag vs digest** | A tag (`vllm/vllm-openai:latest`) is a mutable pointer; a digest (`@sha256:...`) is an immutable content hash. Production pins to digests. |
| **CUDA base image variants** | `base` (just the CUDA runtime libraries), `runtime` (`base` + math libraries needed to *run* CUDA apps), `devel` (`runtime` + compilers/headers needed to *build* CUDA apps). |
| **NVIDIA Container Toolkit** | The host-side component (`nvidia-container-toolkit`) that lets the Docker/containerd runtime expose host GPUs and driver libraries inside containers via `--gpus`. |
| **Docker Compose** | A declarative format (and CLI, `docker compose`) for defining and running multi-container applications as a single unit for local dev / single-node deployments. |
| **SBOM** | Software Bill of Materials — a machine-readable manifest of every package/library inside an image, used for vulnerability matching and license auditing. |
| **CVE scanning** | Static analysis of an image's installed packages against vulnerability databases (Trivy, Grype) to catch known CVEs before deployment. |
| **Image sprawl** | The anti-pattern of many near-duplicate, oversized images accumulating in a registry, each with its own security and storage cost. |

---

## 2. Why This Topic Matters, and Where It Fits in the MLOps/LLMOps Lifecycle

Module 04 established *why* MLOps needs reproducibility: a model that trained successfully on a data scientist's laptop must produce the same behavior when served to production traffic six months later, on different hardware, run by a different team. Docker is the primary mechanism the industry has converged on to make that guarantee tractable.

Think of the MLOps/LLMOps lifecycle as a pipeline of environments:

```
 Notebook  --->  Training job   --->  Model registry  --->  Serving image  --->  Cluster (K8s)
 (laptop)        (batch/GPU)          (MLflow/S3/etc)       (Docker)            (Module 06)
      |                |                     |                    |                   |
      +----------------+---------------------+--------------------+-------------------+
                      Every arrow above is a place where "it worked before" breaks
                      unless the *environment* travels with the *artifact*.
```

Docker solves the environment-travel problem by packaging the OS-level dependencies (CUDA version, system libraries like `libgomp`, Python interpreter version, pinned pip packages, and the model-serving binary itself) into one artifact that is byte-for-byte identical everywhere it runs. This matters more in ML/LLM systems than in typical web backends for several concrete reasons:

1. **Dependency fragility.** PyTorch, CUDA, cuDNN, and NCCL are version-sensitive in ways that ordinary web frameworks are not. A driver/CUDA/PyTorch mismatch can silently produce wrong numerical results (not just crashes), which is far more dangerous in an inference pipeline.
2. **GPU-specific packaging.** Unlike a stateless web service, ML/LLM containers must correctly bridge into host GPU drivers — this is a whole additional axis of environment-parity risk (Section 3.3).
3. **Large, slow-changing artifacts.** Model weights and CUDA toolkits are gigabytes in size. Image layering and caching strategy directly determine whether your CI pipeline takes 90 seconds or 40 minutes, and whether your autoscaler can spin up a new inference replica in 10 seconds or 10 minutes (a cold-start problem that is *the* central capacity-planning issue in LLM serving).
4. **It is the direct prerequisite for Kubernetes.** Every Pod in Module 06 is, at its core, "run this container image with these resource requests." If the image is bloated, insecure, or slow to pull, no amount of Kubernetes sophistication fixes it. Containerization competence is a hard gate to orchestration competence.
5. **It is where security first gets enforced.** A production ML platform's attack surface is the union of every package in every image it runs. CVE scanning at build time is now (as of 2026) treated as table stakes in the same way unit tests are — not a "nice to have hardening step" but a merge-blocking CI gate.

In short: this module converts the abstract MLOps principle of "reproducibility" into a concrete, testable artifact — the image — and everything from Kubernetes autoscaling to canary deployments to cost control downstream depends on getting this artifact right.

---

## 3. Main Concepts

### 3.1 Images vs. Containers: Theory, Architecture, Examples

**Theory.** An **image** is a read-only template: a stack of filesystem layers plus JSON metadata (default command, environment variables, working directory, exposed ports, labels). A **container** is what you get when the container runtime (containerd/runc under the hood of the Docker Engine) takes that image, adds one thin writable layer on top, and starts a process inside a set of Linux namespaces (PID, network, mount, UTS, IPC) and cgroups (CPU/memory/GPU limits). The image never changes; you can start a hundred containers from the same image, each with its own isolated writable layer and process tree, and none of them affects the underlying image or each other's writable layer.

This is precisely analogous to the relationship between a **class** and an **object** in object-oriented programming: the image is the class definition, the container is an instantiated object. Just as you don't mutate a class definition by mutating one instance, you don't (and shouldn't) rely on mutating a running container to fix production — you rebuild the image and roll out new containers.

**Why this matters for ML systems specifically:** because model weights, tokenizer files, and Python dependencies are baked into the image at build time, "the model that's running" is a well-defined, inspectable, versioned artifact — you can `docker inspect` it, diff two image digests, and know exactly what changed between an incident and a known-good deployment. This is the foundation of rollback: rolling back a bad model deployment is "point the deployment at the previous image digest," not "hope someone remembers what pip packages were installed six weeks ago."

**Architecture (layered filesystem):**

```
        IMAGE (read-only, shared, content-addressed)
        ┌───────────────────────────────────────────┐
        │ Layer 4: COPY app/ /app/                   │  <- your code (changes often)
        ├───────────────────────────────────────────┤
        │ Layer 3: RUN pip install -r requirements    │  <- deps (changes sometimes)
        ├───────────────────────────────────────────┤
        │ Layer 2: RUN apt-get install libgomp1 ...  │  <- system libs (changes rarely)
        ├───────────────────────────────────────────┤
        │ Layer 1: FROM python:3.12-slim             │  <- base OS (changes rarely)
        └───────────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
   │ Container A │ │ Container B │ │ Container C │   <- each has its own thin
   │ writable    │ │ writable    │ │ writable    │      read-write layer on top;
   │ layer       │ │ layer       │ │ layer       │      all share layers 1-4 on disk
   └─────────────┘ └─────────────┘ └─────────────┘
```

**Examples:**

- *Beginner*: `docker run python:3.12-slim python -c "print('hello')"` — pulls the image (if not cached), creates a container, runs one process, container exits. Nothing was mutated in the image.
- *Intermediate*: Build your own image (`docker build -t my-model:1.0 .`), then run three containers from it concurrently (`docker run -d --name a my-model:1.0`, `... --name b ...`, `... --name c ...`) — three isolated inference processes from one image.
- *Production-grade*: A CI pipeline builds `registry.internal/fraud-model:git-sha-abcd123`, pushes it by digest, and a Kubernetes Deployment references that exact digest. Ten replicas run as ten containers from the one immutable image; a rollback is a one-line change to the digest reference, not a rebuild.

### 3.2 Layers, Caching, and Multi-Stage Builds

**Theory — what problem this solves.** Naively, if you rebuild an image every time you change one line of application code, you don't want Docker to re-download and re-install every Python package from scratch — that could take many minutes for a PyTorch/vLLM stack. BuildKit caches each layer keyed by (roughly) the instruction plus the state of the layer below it. If nothing changed in an earlier layer and the instruction text is identical, BuildKit reuses the cached layer instead of re-executing it.

The critical, non-obvious consequence: **layer order matters enormously**, because a cache miss on layer *N* invalidates every layer after it. This gives us a golden rule:

> Order your Dockerfile from **least frequently changing** to **most frequently changing**.

```
GOOD ORDER                          BAD ORDER
──────────────                      ─────────
FROM base image                     FROM base image
COPY requirements.txt .             COPY . .                <- app code copied first
RUN pip install -r requirements.txt RUN pip install -r requirements.txt
COPY . .                            (^ cache-busts on EVERY code change,
                                        even a one-line README edit,
                                        forcing a full dependency reinstall)
```

**Multi-stage builds** solve a different but related problem: many ML packages need heavyweight *build-time* tooling (a C/C++ compiler, CUDA `devel` headers, `git`, build wheels) that are completely unnecessary at *run time*. If you install compilers and headers in your final image, you (a) bloat the image by hundreds of megabytes to gigabytes, (b) increase the CVE-scanning surface for no runtime benefit, and (c) slow down every `docker pull` and every Kubernetes pod cold-start. Multi-stage builds let you use a "fat" builder stage, then copy *only the compiled artifacts* into a "thin" final stage.

**Architecture:**

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│  STAGE 1: "builder"         │        │  STAGE 2: "final"            │
│  FROM python:3.12  AS builder│        │  FROM python:3.12-slim       │
│                              │        │                               │
│  - gcc, build-essential      │  COPY  │  (no compilers, no headers)  │
│  - pip wheel / compile deps  │ --from │                               │
│  - produces .whl / venv      │ =build │  COPY --from=builder /venv   │
│                              │  er    │         /venv                │
└─────────────────────────────┘   ───▶ └─────────────────────────────┘
        (discarded after build)              (this is what ships)
```

**Code — a complete, production-shaped multi-stage Dockerfile for a Python ML inference service:**

```dockerfile
# syntax=docker/dockerfile:1
# ---------- Stage 1: build dependencies ----------
FROM python:3.12-slim AS builder

# Build-time-only system deps (compilers for any packages needing native builds)
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Copy ONLY the dependency manifest first -> maximizes cache hits
COPY requirements.txt .

# Build into an isolated virtualenv we can copy wholesale into the final stage
RUN python -m venv /venv \
    && /venv/bin/pip install --no-cache-dir --upgrade pip \
    && /venv/bin/pip install --no-cache-dir -r requirements.txt

# ---------- Stage 2: minimal runtime ----------
FROM python:3.12-slim AS final

# Run as non-root (security best practice — see Section 5/6)
RUN useradd --create-home --uid 1000 appuser
WORKDIR /app

# Bring in ONLY the built virtualenv, not the compilers used to build it
COPY --from=builder /venv /venv
ENV PATH="/venv/bin:$PATH"

# Copy application code LAST — this is the layer that changes most often
COPY --chown=appuser:appuser ./app ./app

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s CMD \
    python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

ENTRYPOINT ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Note the deliberate ordering: base image → build tooling → dependency manifest → dependency install → app code. A one-line code change now only invalidates the final `COPY` layer; the multi-minute dependency install layer stays cached.

**Code — the Python service this Dockerfile actually packages** (`app/main.py`), so the `HEALTHCHECK` and `ENTRYPOINT` above resolve to something real. Note the three endpoints every production ML service should expose — liveness, readiness (model actually loaded), and metrics — which is what makes the monitoring stack in Section 3.5 meaningful rather than decorative:

```python
# app/main.py
import time
import pickle
from contextlib import asynccontextmanager

from fastapi import FastAPI, Response
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from pydantic import BaseModel

MODEL_PATH = "app/model/fraud_model.pkl"

REQUEST_COUNT = Counter("inference_requests_total", "Total inference requests", ["status"])
REQUEST_LATENCY = Histogram("inference_latency_seconds", "Inference latency")

model_state = {"model": None, "ready": False}


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Load once at container startup, not per-request -> keeps p99 latency low
    with open(MODEL_PATH, "rb") as f:
        model_state["model"] = pickle.load(f)
    model_state["ready"] = True
    yield
    model_state.clear()


app = FastAPI(lifespan=lifespan)


class PredictRequest(BaseModel):
    features: list[float]


@app.get("/health")
def health():
    # Liveness: process is up and can answer HTTP at all
    return {"status": "alive"}


@app.get("/ready")
def ready():
    # Readiness: model is actually loaded and safe to receive traffic
    if not model_state["ready"]:
        return Response(status_code=503, content='{"status": "loading"}')
    return {"status": "ready"}


@app.get("/metrics")
def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)


@app.post("/predict")
def predict(req: PredictRequest):
    start = time.perf_counter()
    try:
        score = model_state["model"].predict_proba([req.features])[0][1]
        REQUEST_COUNT.labels(status="success").inc()
        return {"fraud_probability": float(score)}
    except Exception:
        REQUEST_COUNT.labels(status="error").inc()
        raise
    finally:
        REQUEST_LATENCY.observe(time.perf_counter() - start)
```

This is the piece that ties the Dockerfile, the Kubernetes liveness/readiness probes (Module 06), and the Prometheus scrape target (Section 3.5) into one coherent whole: the *same* three endpoints get referenced by the `HEALTHCHECK` instruction, a Compose `healthcheck:` block, and later a Kubernetes `livenessProbe`/`readinessProbe` — this is not incidental, it is the same contract expressed in three different orchestration layers.

**`.dockerignore`** (always pair with a Dockerfile — it keeps build context small and prevents accidentally baking in secrets or local venvs):

```
.git
.venv
__pycache__/
*.pyc
.env
.pytest_cache
notebooks/
*.ipynb
tests/
.mlflow/
data/raw/
```

### 3.3 GPU-Enabled Containers: NVIDIA Container Toolkit and CUDA Base Images

**Theory — what problem this solves.** A container is, by default, isolated from host devices. GPUs are host devices (`/dev/nvidia0`, etc.) backed by a host kernel driver. Naively, a container has no way to see the GPU or the matching userspace driver libraries. The **NVIDIA Container Toolkit** (package name `nvidia-container-toolkit` — note the deprecated old names `nvidia-docker`/`nvidia-docker2` you may see in older tutorials no longer apply) is a host-side runtime hook that, when a container is started with `--gpus`, mounts the necessary device nodes and driver libraries into the container's namespace. Since Docker Engine 19.03, GPU support is native via `docker run --gpus all ...`; you no longer need the old separate `nvidia-docker` wrapper CLI at all.

A subtlety that trips up almost everyone the first time: **the container does not need to match the host's exact driver version, but it does need a CUDA *toolkit* version the host driver is new enough to support.** The NVIDIA driver on the host is backward-compatible with older CUDA toolkit versions inside the container (within NVIDIA's documented compatibility matrix), but not the reverse — you cannot run a container built against CUDA 12.5 on a host whose driver only supports up to CUDA 12.1.

**CUDA base image variants** (`nvidia/cuda` on Docker Hub) — choosing the right one per build stage is a major, easily-missed size/security optimization:

| Variant | Contains | Use in |
|---|---|---|
| `base` | Minimal CUDA runtime, just enough to launch CUDA driver API calls | Rarely used alone for ML; too minimal for most frameworks |
| `runtime` | `base` + CUDA math libraries (cuBLAS, cuFFT, etc.) needed to *run* a compiled CUDA/PyTorch application | **Final stage** of a multi-stage build (inference/serving image) |
| `devel` | `runtime` + full CUDA toolkit: `nvcc` compiler, headers, static libraries | **Builder stage only** — never ship this in your final image |

```dockerfile
# syntax=docker/dockerfile:1
# ---------- Stage 1: build custom CUDA kernels / compile deps ----------
FROM nvidia/cuda:12.4.1-devel-ubuntu22.04 AS builder
RUN apt-get update && apt-get install -y --no-install-recommends python3.11 python3-pip git \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/deps -r requirements.txt

# ---------- Stage 2: minimal GPU runtime ----------
FROM nvidia/cuda:12.4.1-runtime-ubuntu22.04 AS final
RUN apt-get update && apt-get install -y --no-install-recommends python3.11 \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /deps /usr/lib/python3.11/site-packages
COPY ./app /app
WORKDIR /app
ENTRYPOINT ["python3", "serve.py"]
```

Running it, exposing all GPUs (or a subset via `--gpus '"device=0,1"'`):

```bash
# Verify the host toolkit is correctly installed
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi

# Run your GPU-enabled image, capping visible GPUs and memory-pinning behavior
docker run -d --gpus all \
    --shm-size=8g \
    -p 8000:8000 \
    my-gpu-service:1.0
```

`--shm-size` is worth calling out explicitly: PyTorch's DataLoader and NCCL multi-process communication use `/dev/shm`, and Docker's tiny default shared-memory size (64MB) is a classic silent-crash cause in ML containers — always raise it explicitly for training or multi-GPU inference workloads.

### 3.4 Packaging an LLM-Serving Service (vLLM)

**Theory.** LLM inference servers differ from typical ML microservices in three ways that change how you containerize them: (1) the "model" is not a small pickled file baked into the image but a multi-gigabyte set of weight files, usually pulled at container start from a model hub or object store rather than baked in; (2) the serving engine itself (vLLM, TensorRT-LLM, TGI) is a complex piece of software with its own CUDA/PyTorch version pinning, making "build your own image from scratch" often more error-prone than using the maintainer's official image; (3) startup time (model load + CUDA graph capture + KV-cache allocation) is on the order of tens of seconds to minutes, which materially affects autoscaling design in Kubernetes.

The pragmatic, production-recommended default in 2026 is: **use the official `vllm/vllm-openai` image** as your base, layering only your own config, and rebuild yourself only when you need a custom feature or patched dependency. Building vLLM entirely from source is realistic but slow; the vLLM project documents `--build-arg VLLM_USE_PRECOMPILED=1` specifically to cut down custom build times by reusing prebuilt wheels — check vLLM's `latest` docs branch before every upgrade, since flags and defaults shift release to release (see references.md).

**Example — minimal production Compose service running vLLM's OpenAI-compatible server:**

```dockerfile
# syntax=docker/dockerfile:1
FROM vllm/vllm-openai:latest

# Bake in org-specific defaults / a curated model allowlist config, if any
COPY server_config.yaml /etc/vllm/server_config.yaml

# vLLM's own entrypoint accepts these as CLI args; we set sane defaults via CMD
CMD ["--model", "meta-llama/Llama-3.1-8B-Instruct", \
     "--dtype", "bfloat16", \
     "--gpu-memory-utilization", "0.90", \
     "--max-model-len", "8192", \
     "--tensor-parallel-size", "1"]
```

```bash
docker run -d --name llm-server \
    --gpus all \
    --shm-size=16g \
    -p 8001:8000 \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    -e HUGGING_FACE_HUB_TOKEN=${HF_TOKEN} \
    my-org/vllm-server:1.0
```

Mounting the Hugging Face cache as a volume is important operationally: without it, every container restart re-downloads multi-gigabyte weight files from the hub, which is slow, wastes egress bandwidth, and turns a routine restart into a multi-minute outage.

### 3.5 Docker Compose for Local Multi-Service ML/LLM Development

**Theory — what problem this solves.** A realistic LLM application is not one container; it is an API service, a vector database (for RAG retrieval), a Redis cache (for session state / semantic caching), and often a metrics stack (Prometheus + Grafana) to observe it all while developing. Manually running `docker run` for each with the right networking, volumes, and startup order is tedious and non-reproducible across teammates' laptops. Docker Compose (the integrated `docker compose` v2 CLI plugin — the standalone Python `docker-compose` v1 binary is deprecated and should not appear in new material) lets you declare the entire stack as one YAML file, start it with one command, and get automatic DNS-based service discovery between containers (each service is reachable by its service name as a hostname).

**Architecture:**

```
                        docker compose up
                               │
                 ┌─────────────┼─────────────────────────────┐
                 ▼             ▼                              ▼
        ┌────────────────┐ ┌────────────────┐        ┌────────────────┐
        │  api            │ │  qdrant         │        │  redis          │
        │  (FastAPI)      │ │  (vector DB)    │        │  (cache)        │
        │  :8000          │ │  :6333          │        │  :6379          │
        └───────┬────────┘ └───────┬────────┘        └───────┬────────┘
                │  DNS: "qdrant"    │  DNS: "redis"            │
                └───────────────────┴──────────────────────────┘
                               all on user-defined bridge network "mlnet"
                                              │
                       ┌──────────────────────┼──────────────────────┐
                       ▼                                             ▼
              ┌────────────────┐                            ┌────────────────┐
              │  prometheus     │  scrapes /metrics from     │  grafana        │
              │  :9090          │◀─── api, qdrant, redis ───▶│  :3000          │
              └────────────────┘   exporters                │  (dashboards)   │
                                                              └────────────────┘
```

**Code — a full, working `docker-compose.yml`:**

```yaml
# docker-compose.yml
name: llm-rag-stack

services:
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      target: final
    image: my-org/rag-api:local
    ports:
      - "8000:8000"
    environment:
      - QDRANT_URL=http://qdrant:6333
      - REDIS_URL=redis://redis:6379/0
      - LOG_LEVEL=info
    depends_on:
      qdrant:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks: [mlnet]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 15s
      timeout: 3s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2g

  qdrant:
    image: qdrant/qdrant:v1.11.0
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage
    networks: [mlnet]
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:6333/readyz || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7.4-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: ["redis-server", "--save", "60", "1"]
    networks: [mlnet]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  prometheus:
    image: prom/prometheus:v2.55.1
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks: [mlnet]

  grafana:
    image: grafana/grafana:11.2.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD:-admin}
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on: [prometheus]
    networks: [mlnet]

networks:
  mlnet:
    driver: bridge

volumes:
  qdrant_data:
  redis_data:
  prometheus_data:
  grafana_data:
```

A minimal `monitoring/prometheus.yml` scrape config to accompany it:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "rag-api"
    static_configs:
      - targets: ["api:8000"]
  - job_name: "qdrant"
    static_configs:
      - targets: ["qdrant:6333"]
```

Bring the whole stack up, tail logs from one service, and tear it down cleanly:

```bash
docker compose up -d --build
docker compose logs -f api
docker compose ps
docker compose down -v   # -v also removes named volumes (fresh start)
```

**Why this matters beyond "convenience":** the Compose file is effectively a low-fidelity *rehearsal* of the Kubernetes manifests you'll write in Module 06 — the same mental model (services, networking, health checks, resource limits, dependency ordering) reappears there in a more formal, cluster-aware form. Teams that get Compose right onboard onto Kubernetes noticeably faster.

### 3.6 Image Size Optimization

**Theory.** Image size is not a cosmetic concern in ML/LLM systems — it directly drives: registry storage cost, `docker pull` / Kubernetes pod-scheduling latency (a multi-GB image can take minutes to pull onto a fresh node, which is often the dominant term in autoscaling reaction time), CI pipeline duration, and — because more files means more packages means more CVEs — security exposure.

**Techniques, roughly in order of impact:**

1. **Multi-stage builds** (Section 3.2) — the single biggest lever; routinely cuts image size by 50-90% for compiled/ML dependencies.
2. **Choose the smallest sufficient base image.** `python:3.12-slim` over `python:3.12`; `-alpine` variants where compatibility allows (note: Alpine's `musl` libc can break some compiled ML wheels — test before committing); distroless images for the absolute minimal attack surface when you don't need a shell in production.
3. **Combine and clean up `RUN` instructions in the same layer** so intermediate artifacts (apt cache, pip cache, downloaded tarballs) don't survive into a layer:
   ```dockerfile
   RUN apt-get update && apt-get install -y --no-install-recommends foo \
       && rm -rf /var/lib/apt/lists/*
   ```
   (Splitting this into two separate `RUN` lines "leaks" the apt cache into the image forever — it's baked into an earlier layer even if a later layer deletes the files logically, unless it's the *same* layer.)
4. **Pin and prune Python dependencies.** Avoid installing full `torch` (which bundles CUDA) when a CPU-only or NVIDIA-specific-index wheel is available and sufficient; use `pip install --no-cache-dir`.
5. **`.dockerignore` aggressively** — notebooks, test fixtures, `.git`, local venvs, and raw datasets have no business in build context or the image.
6. **Use `docker image history` and tools like `dive`** (see github.md) to visually inspect exactly which layer contributes which bytes — an essential debugging step before assuming you've optimized enough.

**Comparison table — base image tradeoffs:**

| Base | Approx. relative size | Shell for debugging? | Best for |
|---|---|---|---|
| `python:3.12` (full) | Largest | Yes | Local dev only |
| `python:3.12-slim` | Medium | Yes | Most production Python services (recommended default) |
| `python:3.12-alpine` | Small | Yes (busybox) | CPU-only, pure-Python workloads; verify wheel compatibility |
| `gcr.io/distroless/python3` | Smallest | No | Hardened production, once you've fully validated the app |
| `nvidia/cuda:*-runtime-*` | Large (CUDA libs) | Yes | GPU inference final stage |
| `nvidia/cuda:*-devel-*` | Largest | Yes | GPU builder stage only — never ship |

### 3.7 Security Scanning: Trivy and Grype

**Theory — what problem this solves.** Every package you install — including transitive OS packages pulled in by `apt-get install`, not just your `requirements.txt` — is a potential source of a known Common Vulnerability and Exposure (CVE). Manually tracking this is infeasible at any real scale. Trivy and Grype are open-source scanners that generate (or consume) an SBOM for an image and cross-reference every package version against public vulnerability databases (NVD and vendor-specific feeds), reporting matches by severity (LOW/MEDIUM/HIGH/CRITICAL).

As of 2026, running one of these scanners **as a merge-blocking CI gate** — not an optional periodic audit — is the accepted baseline for any ML/LLM image pipeline, in the same way unit tests or linting are non-negotiable CI steps. This is especially true for LLM-serving images, which tend to have a large transitive dependency footprint (CUDA userspace libraries, PyTorch, tokenizers written in Rust, web framework stacks) and are frequently exposed to untrusted user input.

**Architecture — where scanning sits in the pipeline:**

```
 git push
    │
    ▼
 CI: docker build ──────► image (local/registry, untagged-as-prod)
    │
    ▼
 CI: trivy image --severity HIGH,CRITICAL --exit-code 1 <image>
    │
    ├── PASS (no HIGH/CRITICAL, or all allow-listed) ──► push :latest / tag, deploy
    │
    └── FAIL ──► block merge, surface report, developer patches base image / deps
```

**Code — Trivy in GitHub Actions:**

```yaml
# .github/workflows/build-and-scan.yml
name: Build and Scan ML Image

on:
  pull_request:
  push:
    branches: [main]

jobs:
  build-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t my-org/rag-api:${{ github.sha }} ./api

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: my-org/rag-api:${{ github.sha }}
          format: table
          severity: HIGH,CRITICAL
          exit-code: "1"          # fail the build on any HIGH/CRITICAL finding
          ignore-unfixed: true    # don't block on CVEs with no available fix yet

      - name: Push image (only if scan passed)
        if: success()
        run: |
          docker tag my-org/rag-api:${{ github.sha }} registry.internal/rag-api:${{ github.sha }}
          docker push registry.internal/rag-api:${{ github.sha }}
```

Grype offers an equivalent local CLI workflow, useful for a fast pre-push check on a developer machine:

```bash
grype my-org/rag-api:local --fail-on high
```

**Practical guidance on thresholds:** blocking on every LOW/MEDIUM finding produces alert fatigue and teams that disable the gate entirely; the pragmatic default is to block merges on HIGH/CRITICAL with a fixed patch available, track MEDIUM in a dashboard, and maintain a reviewed allow-list (with expiry dates) for findings that are false positives or genuinely unexploitable in your deployment context.

---

## 4. Real-World Case Studies (Reasoned Inference)

*The following are reasoned inferences about plausible architectures at these companies, based on publicly known engineering-blog patterns and industry norms — not confirmed internal implementation details.*

- **A company like OpenAI or Anthropic**, serving foundation models at massive scale, would plausibly maintain a small number of highly optimized, custom-built inference-serving base images (likely built on top of or heavily inspired by engines like vLLM/TensorRT-LLM/in-house engines), with model weights fetched from internal blob storage at pod startup rather than baked into the image, given how frequently weights are updated relative to the serving code. Given the scale of GPU fleets involved, image pull latency and cold-start time would be treated as first-class performance metrics, likely motivating techniques like layer pre-warming on nodes and aggressive multi-stage build discipline to keep the serving image itself (excluding weights) as small as possible.

- **A company like Netflix**, historically a heavy user of container-based microservice deployment (its Titus container platform predates much of today's Kubernetes tooling), would plausibly apply the same rigor to ML-serving containers (e.g., recommendation-ranking models) as to any other production microservice: strict base-image standardization enforced centrally, automated CVE scanning integrated into its internal CI/CD (Spinnaker-style pipelines), and canary-based rollout of new image versions gated on both business and model-quality metrics.

- **A company like Uber**, given its publicly documented Michelangelo ML platform, would plausibly containerize each stage of its feature-engineering-to-serving pipeline uniformly, using layered base images shared across many model types (to maximize registry-level layer cache hits across thousands of model images) and using multi-stage builds heavily since many of its models involve compiled feature transformers alongside Python serving code.

- **A company like Spotify**, running large fleets of recommendation and personalization services on Kubernetes (per its published engineering blog patterns), would plausibly treat Docker Compose as strictly a local-dev/CI-integration-test tool and never as a deployment target, with a strong internal convention that whatever runs in Compose during CI integration tests must be a structurally faithful (if smaller-scale) stand-in for the Kubernetes manifests used in staging/production — precisely the pattern this module models.

- **A company like Databricks**, given that MLflow (which Databricks originated) has first-class support for packaging models as Docker images (`mlflow models build-docker`), would plausibly rely on this tooling — or a close internal descendant of it — to make every registered model in the MLflow Model Registry directly, mechanically buildable into a deployable image, reducing the "last mile" from experiment tracking to production to a single command rather than a bespoke handoff process per team.

- **A company like NVIDIA**, as the maintainer of the CUDA toolkit and the NVIDIA Container Toolkit itself, would plausibly be the reference implementer of GPU container best practices industry-wide (as it publicly is, via NGC — NVIDIA GPU Cloud — container images); teams elsewhere building GPU-enabled ML images would plausibly start from NVIDIA's own published Dockerfiles and NGC base images rather than reinventing CUDA/cuDNN version-compatibility handling from scratch, given how easy it is to get subtly wrong.

- **A company like Amazon or Meta**, operating both very large internal GPU fleets and (in Amazon's case) a public cloud, would plausibly enforce SBOM generation and vulnerability scanning as a hard gate integrated directly into their internal container registries (analogous to how Amazon ECR supports native image scanning) rather than solely as a separate CI step, so that even images pushed outside a "blessed" CI pipeline are still caught before they can be pulled onto a production node.

---

## 5. Common Mistakes

1. **Copying application code before installing dependencies**, busting the dependency-install cache on every code change (Section 3.2) — the single most common Dockerfile performance mistake.
2. **Shipping a `devel` CUDA image (or any image with a full compiler toolchain) to production** instead of `runtime`/`base`, bloating the image by gigabytes and needlessly widening the CVE surface.
3. **Not pinning base image tags** (`FROM python:3` or `FROM nvidia/cuda:latest`) — this silently changes what ships over time and breaks the entire point of reproducibility; always pin to a specific version, and for the most critical production images, to a digest.
4. **Forgetting `--shm-size`** for PyTorch DataLoader / NCCL-heavy workloads, causing mysterious `Bus error` crashes that have nothing to do with application logic.
5. **Baking secrets (API keys, HF tokens) into image layers** via `ENV` or `COPY` of a `.env` file — even if later "removed," the secret persists in the layer history and is trivially extractable with `docker history` or by pulling the image. Use build secrets (`--secret`) or runtime environment injection instead.
6. **Running containers as root** by default (the implicit default if you never add a `USER` instruction) — a container escape then has host root-equivalent implications inside its namespace.
7. **Using the deprecated standalone `docker-compose` v1 binary or ignoring version drift** — new material and new projects should use the `docker compose` v2 plugin exclusively.
8. **Not mounting the Hugging Face / model cache directory as a volume** for LLM-serving containers, causing multi-gigabyte weight re-downloads on every restart.
9. **Treating `docker-compose.yml` as a production deployment mechanism** rather than a local-dev/integration-test tool — Compose lacks the self-healing, multi-node scheduling, and rolling-update guarantees Kubernetes provides (Module 06).
10. **Ignoring or blanket-suppressing CVE scan failures** ("just add `--exit-code 0`" to unblock a deadline) rather than triaging and fixing or consciously allow-listing with an expiry date — this quietly defeats the entire purpose of the gate.
11. **Not setting resource limits** in Compose/local testing, masking OOM behavior that will bite in Kubernetes where limits are enforced by default in most clusters.

---

## 6. Best Practices and Production Tips

**When to use Docker (vs. alternatives):** Docker/OCI containers are the right default for essentially all ML/LLM serving and batch workloads today. The main scenarios where you'd deliberately *not* containerize: extremely latency-sensitive HFT-style systems where even container-runtime overhead matters (rare in ML), or quick throwaway local experimentation where a plain virtualenv is faster to iterate in — but even then, containerize before anything touches shared infrastructure or another teammate's machine.

**Alternatives worth knowing about:** Podman (daemonless, rootless-by-default OCI-compatible alternative to the Docker Engine — API-compatible enough that `alias docker=podman` often works); Buildpacks/`ko` for language-idiomatic image builds without hand-written Dockerfiles; Apptainer/Singularity in HPC/research-cluster contexts where Docker's root-daemon model conflicts with shared-cluster security policies.

**Cost and scaling:**
- Image size is a direct cost lever: smaller images mean less registry storage cost, less network egress on every pull, and faster autoscaling reaction time — often the single biggest lever on tail latency during traffic spikes for LLM serving.
- For GPU workloads, `--gpus` and Kubernetes resource requests should be set to reflect *actual* memory/utilization needs (validated via monitoring, Section 3.5) rather than defaulting to "all GPUs" everywhere, to avoid stranding expensive GPU capacity.

**Monitoring:** every production image should expose a `/health` (liveness) and ideally a `/ready` (readiness, e.g., "model loaded and warm") endpoint, plus a `/metrics` endpoint in Prometheus exposition format — this is what makes the Compose monitoring stack in Section 3.5 (and its Kubernetes descendant in Module 06) actually useful.

**Security:** run as non-root; use multi-stage builds to exclude build tooling; scan every image in CI (Section 3.7); prefer digest-pinning for anything deployed; rotate and never bake in secrets; keep base images patched on a schedule (subscribe to base-image CVE advisories, don't just "build once and forget").

**Performance tradeoffs:** `devel` images build faster locally (no separate builder-stage complexity) but should never ship; Alpine images are smaller but `musl` libc can cause subtle incompatibilities with compiled scientific-Python wheels — validate before adopting for GPU/ML workloads; layer caching in CI is only as good as your CI runner's cache persistence configuration (a cold CI runner with no cache mount will rebuild everything every time, regardless of Dockerfile quality).

**Documentation/process tip:** treat the Dockerfile and Compose file as reviewed, versioned production code — not a one-off script — since (per Section 2) they are the artifact that ultimately determines what runs in production.

---

## 7. Interview Questions

1. **Q: What is the difference between a Docker image and a container, and why does that distinction matter operationally?**
   A: An image is an immutable, layered template; a container is a running instance with its own thin writable layer. Operationally this means you never "fix production" by SSHing into a running container and editing files — you rebuild the image and redeploy, which is what makes rollback, auditability, and horizontal scaling well-defined.

2. **Q: Why does instruction order in a Dockerfile matter, and how would you reorder a Dockerfile that copies application code before installing dependencies?**
   A: BuildKit caches layers; a cache miss invalidates every subsequent layer. Copying code before `pip install` means every code change re-triggers a full dependency reinstall. Fix: `COPY requirements.txt .` → `RUN pip install ...` → `COPY . .`, so only the final layer is invalidated by routine code changes.

3. **Q: Explain the difference between the `base`, `runtime`, and `devel` variants of the `nvidia/cuda` Docker image, and where each belongs in a multi-stage build.**
   A: `base` is minimal CUDA runtime; `runtime` adds math libraries (cuBLAS etc.) needed to *run* CUDA applications; `devel` adds the full toolkit including `nvcc` and headers needed to *compile* CUDA code. `devel` belongs only in a builder stage; the final/shipped stage should use `runtime` (or `base` if sufficient) to minimize image size and CVE surface.

4. **Q: Why would you choose to use the official `vllm/vllm-openai` image rather than building vLLM from source in your own Dockerfile?**
   A: vLLM has intricate CUDA/PyTorch version pinning and a nontrivial build process; the maintainers' official image is tested against that exact combination. Building from source is sometimes necessary (custom patches, specific hardware) — vLLM documents `VLLM_USE_PRECOMPILED=1` to speed this up — but for most teams, layering config/customization on top of the official image is lower-risk and lower-maintenance.

5. **Q: What's wrong with baking a Hugging Face access token into an image via `ENV HF_TOKEN=...` in the Dockerfile?**
   A: The value persists in the image's layer history indefinitely and is extractable via `docker history` or by anyone who can pull the image, even if a later layer "unsets" it. Secrets should be injected at runtime (environment variables passed to `docker run`/Kubernetes Secret) or via BuildKit's `--secret` mount, which is never persisted into a layer.

6. **Q: Why is CVE scanning treated as a CI-blocking step for ML/LLM images specifically, rather than an optional audit?**
   A: ML/LLM images tend to have a large transitive dependency footprint (CUDA userspace libraries, ML frameworks, native-compiled tokenizers) and are frequently exposed to untrusted input (user prompts, uploaded data), increasing both the attack surface and the exploitability of any given CVE. As of 2026 this is considered baseline hygiene, comparable to unit testing, not optional hardening.

7. **Q: When would you use Docker Compose versus Kubernetes for an ML/LLM workload?**
   A: Compose is appropriate for local development and CI integration testing — single-node, no self-healing, no rolling updates, minimal orchestration. Kubernetes is appropriate for anything that needs multi-node scheduling, autoscaling, rolling/canary deployments, or production-grade self-healing. Compose should mirror Kubernetes' service/networking/health-check shape closely so the transition (Module 06) is structural, not a redesign.

8. **Q: A GPU-enabled PyTorch training container is crashing intermittently with a `Bus error` during multi-worker data loading. What's a likely container-level (not application-level) cause, and how do you fix it?**
   A: Docker's default shared-memory size (`/dev/shm`, 64MB) is too small for PyTorch's DataLoader worker processes and NCCL communication. Fix by increasing it explicitly at run time, e.g. `docker run --shm-size=8g ...` (or the equivalent `shm_size` field in Compose / `emptyDir` sizeLimit with `medium: Memory` in Kubernetes).

---

## 8. Summary, Key Takeaways, and Production Checklist

**Summary:** Docker converts the abstract MLOps goal of "reproducible environments" into a concrete, versioned, inspectable artifact — the image. Understanding layers and caching makes your builds fast; understanding multi-stage builds and CUDA image variants makes your images small, secure, and GPU-correct; Docker Compose lets you rehearse a realistic multi-service ML/LLM stack locally before it ever reaches Kubernetes; and CVE scanning turns "we hope our dependencies are safe" into a verifiable, enforced CI gate.

**Key Takeaways:**
- Images are immutable templates; containers are disposable instances — never treat a running container as the source of truth.
- Dockerfile instruction order determines cache efficiency; order least-to-most frequently changing.
- Multi-stage builds separate build-time tooling from runtime — always ship the `runtime`/`base` CUDA variant, never `devel`.
- GPU access requires the NVIDIA Container Toolkit and `--gpus`; raise `--shm-size` for multi-worker/multi-GPU workloads.
- Prefer the official `vllm/vllm-openai` image for LLM serving; mount the model cache as a volume.
- Docker Compose is for local dev/integration testing, structurally mirroring — never replacing — the Kubernetes deployment it rehearses.
- CVE scanning (Trivy/Grype) in CI is a 2026 baseline expectation, not optional hardening.

**Production Checklist:**

```
[ ] Base images pinned to a specific version (digest for critical images), not :latest
[ ] Multi-stage build used; final stage has no compilers/build tooling
[ ] Dockerfile instructions ordered least-to-most frequently changing
[ ] .dockerignore excludes .git, notebooks, tests, local venvs, raw data
[ ] Correct CUDA variant per stage: devel (builder only) vs runtime/base (final)
[ ] Container runs as non-root USER
[ ] No secrets baked into ENV/COPY layers; runtime injection or --secret used
[ ] HEALTHCHECK (or Compose/K8s equivalent) defined and meaningful
[ ] --shm-size raised for PyTorch/NCCL multi-worker workloads
[ ] Model/weights cache mounted as a volume, not re-downloaded per container start
[ ] /metrics endpoint exposed in Prometheus format
[ ] Trivy/Grype scan integrated into CI, blocking on HIGH/CRITICAL with fixes available
[ ] Image size measured and reviewed (docker image history / dive) before shipping
[ ] Compose file mirrors intended Kubernetes service/network/health-check shape
```

---

## 9. Further Reading

Curated official docs, GitHub repositories, books, and videos for this module — including the Dockerfile and Compose specifications, NVIDIA Container Toolkit and CUDA image docs, the vLLM Docker deployment guide, and Trivy/Grype scanning references — are catalogued in **references.md**, **github.md**, **books.md**, and **videos.md** in this same module folder. Consult those files for exact links and version details rather than this chapter, which intentionally focuses on synthesis and application rather than re-listing source material.
