# Architecture Deep Dive — Docker for ML and LLM Systems

This companion document expands on `tutorial.md` with larger, more detailed architecture diagrams, a full request/build sequence diagram, and a decision tree for choosing between the tools and patterns covered in this module. Read `tutorial.md` first — this file assumes you already know the vocabulary (image, layer, multi-stage build, NVIDIA Container Toolkit, Compose, Trivy/Grype).

---

## 1. End-to-End System Architecture: From Dockerfile to Running LLM Stack

The diagram below shows the full path from source code to a running, observed, GPU-backed LLM service — the "widest" view this module supports before Kubernetes (Module 06) takes over the orchestration layer.

```
 ┌───────────────────────────────────────────────────────────────────────────────────────┐
 │                                   DEVELOPER MACHINE / CI RUNNER                         │
 │                                                                                          │
 │   repo/                                                                                 │
 │   ├── api/                                                                              │
 │   │   ├── Dockerfile            (multi-stage: builder -> final)                        │
 │   │   ├── .dockerignore                                                                 │
 │   │   ├── requirements.txt                                                              │
 │   │   └── app/main.py           (FastAPI: /predict /health /ready /metrics)             │
 │   ├── llm/                                                                              │
 │   │   └── Dockerfile            (FROM vllm/vllm-openai:latest + config)                 │
 │   ├── monitoring/prometheus.yml                                                          │
 │   └── docker-compose.yml                                                                │
 │                                                                                          │
 │        docker build ──► BuildKit ──► layer cache lookup ──► image (local store)         │
 │                                            │                                            │
 │                                   HIT? reuse layer   MISS? execute instruction,          │
 │                                                        create new content-addressed      │
 │                                                        layer, cache it for next build    │
 └───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                              │  trivy / grype scan (CI gate — Section 3.7)
                                              │  PASS ──► docker push (tag + digest)
                                              ▼
                              ┌───────────────────────────────┐
                              │   IMAGE REGISTRY (GHCR/ECR/    │
                              │   Harbor/Docker Hub)           │
                              │   my-org/rag-api@sha256:abcd..│
                              │   my-org/vllm-server@sha256:.. │
                              └───────────────┬───────────────┘
                                              │  docker compose up  (pulls by tag/digest)
                                              ▼
 ┌───────────────────────────────────────────────────────────────────────────────────────┐
 │                     DOCKER HOST  (dev laptop OR a single GPU VM)                        │
 │                                                                                          │
 │   user-defined bridge network: "mlnet"      (DNS: container name == hostname)            │
 │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌───────────────────────────┐ │
 │  │  api          │   │  qdrant      │   │  redis       │   │  llm-server (vLLM)         │ │
 │  │  FastAPI      │──▶│  vector DB   │   │  cache /     │   │  OpenAI-compatible API      │ │
 │  │  :8000        │   │  :6333       │   │  session     │   │  :8000 (mapped 8001)        │ │
 │  │               │──────────────────────▶  :6379       │   │                             │ │
 │  │  /predict     │   │  RAG vectors │   │              │   │  --gpus all                 │ │
 │  │  /health      │   │  persisted   │   │  persisted   │   │  --shm-size=16g              │ │
 │  │  /ready       │   │  volume      │   │  volume      │   │  -v ~/.cache/huggingface:... │ │
 │  │  /metrics     │   └──────────────┘   └──────────────┘   │  GPU device passthrough via  │ │
 │  └──────┬───────┘                                          │  NVIDIA Container Toolkit    │ │
 │         │ scraped by                                        └──────────┬────────────────┘ │
 │         ▼                                                              │ scraped by        │
 │  ┌──────────────────────────────────────────────────────────────────┐  │                   │
 │  │  prometheus  :9090   (scrape_configs: api, qdrant, llm-server)   │◀─┘                   │
 │  └──────────────────────────────┬───────────────────────────────────┘                       │
 │                                 ▼                                                            │
 │                       ┌──────────────────┐                                                  │
 │                       │  grafana  :3000   │  dashboards: latency, TTFT, KV-cache util,       │
 │                       │                   │  queue depth, error rate                        │
 │                       └──────────────────┘                                                  │
 │                                                                                              │
 │  ┌────────────────────────────────────────────────────────────────────────────────────┐    │
 │  │  HOST KERNEL: NVIDIA driver + nvidia-container-toolkit runtime hook                 │    │
 │  │  /dev/nvidia0, /dev/nvidiactl, driver userspace libs bind-mounted into "llm-server"  │    │
 │  └────────────────────────────────────────────────────────────────────────────────────┘    │
 └───────────────────────────────────────────────────────────────────────────────────────┘
```

Key structural observations worth internalizing:

1. **Everything above the registry is build-time**; everything below it is run-time. The registry is the hard boundary — once an image is pushed by digest, what runs is fixed, regardless of what happens to the source repo afterward.
2. **The GPU only ever "appears" inside one container** (`llm-server`) via a host-kernel-level hook — it is not something Docker virtualizes; it is a real device bind-mounted through, which is why the host driver version and the container's CUDA toolkit version must be mutually compatible (Section 3.3 of `tutorial.md`).
3. **Prometheus scraping direction is inverted relative to the request flow** — the API and LLM server are passive HTTP servers exposing `/metrics`; Prometheus is the active client that pulls on an interval. This pull model (vs. push) is why every service needing observability must expose an HTTP metrics endpoint, not "send" metrics anywhere.
4. **This entire diagram is a single-host rehearsal** of what Module 06 turns into a multi-node Kubernetes cluster: `mlnet` becomes a Kubernetes `Service`/CNI network, each service becomes a `Deployment` + `Service`, the named volumes become `PersistentVolumeClaim`s, and `--gpus all` becomes a `resources.limits["nvidia.com/gpu"]` request handled by the NVIDIA device plugin.

---

## 2. Multi-Stage Build Internals: Layer Graph and Cache Keying

A more mechanically precise view of what BuildKit actually does with a multi-stage Dockerfile — this is the mental model that explains *why* reordering instructions changes build time so dramatically.

```
Dockerfile instructions                 BuildKit's internal DAG (not linear!)
────────────────────────                ─────────────────────────────────────

FROM python:3.12-slim AS builder   ──►  [L0: base image builder]
RUN apt-get install build-essential──►  [L1: cache-key = hash(L0 + instruction text)]
COPY requirements.txt .            ──►  [L2: cache-key = hash(L1 + file content hash)]
RUN pip install -r requirements.txt──►  [L3: cache-key = hash(L2 + instruction text)]
                                              │
FROM python:3.12-slim AS final     ──►  [L0': independent base image final]  <- parallelizable
RUN useradd appuser                ──►  [L1': hash(L0' + instruction text)]
COPY --from=builder /venv /venv    ──►  [L2'*: depends on L3 above via --from]
COPY ./app ./app                   ──►  [L3'*: cache-key = hash(L2' + file content hash)]
USER appuser                       ──►  [L4': metadata-only, near-free]

BuildKit executes [L0..L3] and [L0'..L1'] IN PARALLEL (independent stages),
then joins at COPY --from=builder, which is the only real dependency edge
between the two stages' layer graphs.
```

**Why a one-line code change is cheap, and why a one-line `requirements.txt` change is not:**

```
Change to app/main.py only:
  L0 L1 L2 L3  (builder stages)         ALL CACHE HITS  — 0 seconds
  L0' L1'                               CACHE HIT       — 0 seconds
  L2'  (COPY --from=builder)            CACHE HIT       — 0 seconds (input unchanged)
  L3'  (COPY ./app ./app)               CACHE MISS      — re-executed (fast, just a copy)
  L4'                                   re-executed (free)
  ──► total rebuild: near-instant

Change to requirements.txt:
  L0 L1                                  CACHE HIT
  L2  (COPY requirements.txt)            CACHE MISS — file hash changed
  L3  (RUN pip install ...)              CACHE MISS — must re-run, potentially minutes
  L0' L1'                                CACHE HIT
  L2' (COPY --from=builder)              CACHE MISS — upstream input (L3) changed
  L3' L4'                                CACHE MISS (cascades)
  ──► total rebuild: full dependency reinstall time
```

This is the concrete mechanism behind the "least-to-most frequently changing" ordering rule — it isn't a style preference, it's a direct consequence of how content-addressed cache keys propagate through the DAG.

---

## 3. Sequence Diagram: `docker compose up` for the Full RAG + LLM Stack

This traces, in order, what actually happens on the wire and in each process from the moment a developer runs `docker compose up -d --build` to the first successful end-to-end RAG request. ASCII sequence notation: participants across the top, time flowing downward, arrows are messages/calls.

```
Developer   Docker CLI   BuildKit   Registry   qdrant   redis   api(FastAPI)  llm-server(vLLM)  Prometheus
   │            │            │          │         │        │          │             │              │
   │ compose up │            │          │         │        │          │             │              │
   │ -d --build │            │          │         │        │          │             │              │
   ├───────────►│            │          │         │        │          │             │              │
   │            │ build api  │          │         │        │          │             │              │
   │            ├───────────►│          │         │        │          │             │              │
   │            │            │ (layer cache check, build stages, Section 2 above)   │              │
   │            │◄───────────┤ image: my-org/rag-api:local                         │              │
   │            │            │          │         │        │          │             │              │
   │            │ pull qdrant, redis, prometheus, grafana (if not cached locally)   │              │
   │            ├────────────────────────►│         │        │          │             │              │
   │            │◄────────────────────────┤         │        │          │             │              │
   │            │            │          │         │        │          │             │              │
   │            │ create network "mlnet"  │         │        │          │             │              │
   │            │            │          │         │        │          │             │              │
   │            │ start qdrant, redis, llm-server (no depends_on between these)      │              │
   │            ├────────────────────────────────►start────►start─────────────────►start            │
   │            │            │          │         │        │          │             │              │
   │            │            │          │         │        │          │  vLLM: load weights          │
   │            │            │          │         │        │          │  from HF cache volume         │
   │            │            │          │         │        │          │  (10s-minutes depending       │
   │            │            │          │         │        │          │   on model size + whether     │
   │            │            │          │         │        │          │   cache was warm)              │
   │            │            │          │         │        │          │             │              │
   │            │ healthcheck poll: qdrant /readyz  │        │          │             │              │
   │            ├────────────────────────►│         │        │          │             │              │
   │            │◄────────────────────────┤ 200 OK  │        │          │             │              │
   │            │ healthcheck poll: redis PING       │        │          │             │              │
   │            ├─────────────────────────────────►│          │             │              │
   │            │◄─────────────────────────────────┤ PONG     │             │              │
   │            │            │          │  (both "service_healthy" now — api's        │              │
   │            │            │          │   depends_on condition is satisfied)         │              │
   │            │            │          │         │        │          │             │              │
   │            │ start api (depends_on: qdrant, redis healthy)         │             │              │
   │            ├───────────────────────────────────────────────────►start           │              │
   │            │            │          │         │        │  lifespan(): load model  │              │
   │            │            │          │         │        │  pickle -> ready=True    │              │
   │            │            │          │         │        │          │             │              │
   │            │ healthcheck poll: api /health      │        │          │             │              │
   │            ├───────────────────────────────────────────────────►│              │              │
   │            │◄───────────────────────────────────────────────────┤ 200 OK        │              │
   │            │            │          │         │        │          │             │              │
   │◄───────────┤ "Started" (all services up)      │        │          │             │              │
   │            │            │          │         │        │          │             │              │
   │  curl POST /predict     │          │         │        │          │             │              │
   ├─────────────────────────────────────────────────────────────────►│             │              │
   │            │            │          │         │        │  1. embed query          │             │              │
   │            │            │          │         │        │  2. query qdrant ────────►│              │
   │            │            │          │         │        │◄──── top-k chunks ────────┤              │
   │            │            │          │         │        │  3. check redis cache      │              │
   │            │            │          │  ┌────────────────►│                          │              │
   │            │            │          │  │      │◄────────┤ cache miss                │              │
   │            │            │          │  └──────┘         │  4. call llm-server ──────────────────►│
   │            │            │          │         │          │                          │  generate    │
   │            │            │          │         │          │◄────── completion ───────┤  tokens      │
   │            │            │          │         │          │  5. write-through cache ─►│              │
   │◄─────────────────────────────────────────────┤ 6. return response, record metrics  │              │
   │            │            │          │         │          │  (Counter.inc, Histogram.observe)       │
   │            │            │          │         │          │                          │              │
   │            │            │          │         │          │  (async, on the 15s scrape_interval)     │
   │            │            │          │         │          │◄──────────── GET /metrics ───────────────┤
   │            │            │          │         │          │─────────── 200 (Prometheus format) ─────►│
```

Two operational lessons this sequence makes visible that are easy to miss when reading YAML alone:

- **`depends_on` with `condition: service_healthy` controls startup order, not runtime resilience.** If `qdrant` later crashes and restarts, `api` is not automatically restarted or blocked — this is purely a one-time startup gate. Runtime resilience (retry/circuit-breaker on the `qdrant` client call) has to be implemented in application code; Compose does not provide it. Kubernetes' liveness/readiness probes (Module 06) extend this same idea into an ongoing, continuously-enforced guarantee.
- **The vLLM model-load step is the dominant term in "time to first successful request."** In a Kubernetes autoscaling context, this exact step becomes the cold-start latency that determines whether a scale-up event can respond to a traffic spike in time — which is why production LLM-serving architectures invest heavily in keeping a warm pool of replicas rather than scaling purely reactively from zero.

---

## 4. Decision Tree: Choosing Your Containerization Approach

Use this when starting a new ML/LLM service, or when deciding how to evolve an existing one.

```
                         ┌───────────────────────────────────────┐
                         │  Does the workload need a GPU at all?  │
                         └───────────────┬─────────────┬─────────┘
                                    NO   │             │  YES
                        ┌────────────────┘             └───────────────────┐
                        ▼                                                  ▼
        ┌───────────────────────────────┐            ┌──────────────────────────────────────┐
        │ CPU-only service               │            │ Is it an LLM/text-generation server   │
        │ (feature service, classical ML, │            │ (vs. a custom training/inference job)? │
        │ light embedding model, API glue)│            └───────────┬──────────────┬───────────┘
        └───────────────┬───────────────┘                    YES  │              │  NO
                        ▼                                          ▼              ▼
        ┌───────────────────────────────┐          ┌───────────────────────┐  ┌─────────────────────────┐
        │ python:3.12-slim base          │          │ Is a supported engine  │  │ Multi-stage build:       │
        │ multi-stage build if any        │          │ (vLLM/TGI/TensorRT-LLM)│  │ nvidia/cuda:*-devel-*    │
        │ compiled deps (numpy/scipy/     │          │ image sufficient, or   │  │  (builder) ──►           │
        │ tokenizers) needed at build     │          │ do you need a fully    │  │ nvidia/cuda:*-runtime-*  │
        │ time only                       │          │ custom serving stack?  │  │  (final)                 │
        └───────────────┬───────────────┘          └──────┬───────┬─────────┘  │ + NVIDIA Container       │
                        │                            SUPPORTED│    │CUSTOM      │   Toolkit + --gpus        │
                        ▼                          sufficient│    │needed      │ + --shm-size raised       │
        ┌───────────────────────────────┐                  ▼    ▼            └─────────────┬────────────┘
        │ Consider Alpine/distroless      │      ┌──────────────────┐                       │
        │ ONLY if wheel compatibility     │      │ FROM vllm/vllm-  │       ┌────────────────┴────────────┐
        │ is verified (musl libc risk)    │      │ openai:latest    │       │ Validate against NVIDIA NGC   │
        └───────────────────────────────┘      │ + your config    │       │ reference Dockerfiles first — │
                                                 │ + volume-mount   │       │ CUDA/cuDNN/framework version   │
                                                 │   model cache    │       │ compatibility is easy to get   │
                                                 └──────────────────┘       │ subtly wrong from scratch      │
                                                                            └───────────────────────────────┘

                         ┌─────────────────────────────────────────────────────┐
                         │ How many services does the running system have?      │
                         └───────────────┬───────────────────────┬─────────────┘
                                    ONE  │                        │  MULTIPLE (API + DB + cache + ...)
                                          ▼                        ▼
                         ┌─────────────────────────┐  ┌─────────────────────────────────────┐
                         │ Plain `docker run` /      │  │ Docker Compose (v2, `docker compose`) │
                         │ single Dockerfile is       │  │ for local dev + CI integration tests  │
                         │ enough for local dev       │  └───────────────┬───────────────────────┘
                         └─────────────────────────┘                    ▼
                                                          ┌─────────────────────────────────────┐
                                                          │ Is this going to run in shared/multi- │
                                                          │ node production, need autoscaling,     │
                                                          │ rolling updates, or self-healing?      │
                                                          └───────────┬─────────────┬─────────────┘
                                                                 NO  │             │  YES
                                                                      ▼             ▼
                                                     ┌─────────────────────┐  ┌───────────────────────┐
                                                     │ Compose may be       │  │ Kubernetes (Module 06) │
                                                     │ sufficient (small,   │  │ — Compose becomes the  │
                                                     │ single-node internal │  │ local-dev/CI rehearsal │
                                                     │ tool)                │  │ only, never the deploy  │
                                                     └─────────────────────┘  │ target                 │
                                                                              └───────────────────────┘

                         ┌─────────────────────────────────────────────────────┐
                         │ Scanning: Trivy vs. Grype vs. both?                   │
                         └───────────────┬───────────────────────┬─────────────┘
                              Need SBOM +│                       │ Need broader VEX
                              misconfig + │                      │ (OpenVEX/CSAF) suppression
                              secret scan │                      │ workflow for known-safe CVEs
                              in one tool │                      │
                                          ▼                       ▼
                         ┌─────────────────────┐   ┌─────────────────────────────┐
                         │ Trivy — broadest      │   │ Grype — pair with an SBOM    │
                         │ single-tool coverage,  │   │ tool (Syft) and VEX docs for  │
                         │ good default CI gate   │   │ fine-grained suppression      │
                         └─────────────────────┘   └─────────────────────────────┘
                                          │                       │
                                          └───────────┬───────────┘
                                                       ▼
                                     Many production pipelines run BOTH as a
                                     cross-check (different vuln DB update cadences
                                     mean they occasionally disagree) — this is
                                     normal, not a sign either tool is broken.
```

---

## 5. Image Size Contribution: A Worked "Where Did the Gigabytes Go" Breakdown

A concrete illustration of why base-image and stage choice dominates final image size for a GPU-enabled LLM-adjacent service (illustrative proportions, not measured benchmark numbers):

```
 nvidia/cuda:12.4.1-devel-ubuntu22.04 image (builder stage — NEVER shipped)
 ┌─────────────────────────────────────────────────────────────┐
 │ nvcc compiler, headers, static libs        ███████████████  │  <- discarded entirely
 │ full CUDA toolkit                          ██████████████   │     after multi-stage COPY
 │ ubuntu base                                ███              │
 └─────────────────────────────────────────────────────────────┘
                              │
                              │ COPY --from=builder (only compiled artifacts / venv cross over)
                              ▼
 nvidia/cuda:12.4.1-runtime-ubuntu22.04 image (final — this is what ships)
 ┌─────────────────────────────────────────────────────────────┐
 │ your application code                      ▏  (tiny)        │
 │ python venv (torch runtime wheels, no CUDA │████             │
 │   toolkit bundled separately since base     │                │
 │   image already provides CUDA libs)         │                │
 │ cuBLAS/cuFFT/etc (runtime math libs)        │███              │
 │ ubuntu base + python3 runtime               │███              │
 └─────────────────────────────────────────────────────────────┘
     ──► final image is a small fraction of the builder stage's footprint
```

The general pattern generalizes beyond CUDA: **any build-time-only tool (compilers, headers, package-manager caches, test fixtures, `.git` history) that leaks into the final stage is pure waste** — it contributes zero runtime capability while adding pull latency, storage cost, and CVE-scanning surface. The `dive` tool (see `github.md`) is the recommended way to verify, layer by layer, that no such leakage has occurred before shipping.

---

## Further Reading

See `references.md`, `github.md`, `books.md`, and `videos.md` in this same module folder for the full source list this architecture deep-dive and `tutorial.md` are built on.
