# Module 05 Cheat Sheet — Docker for ML and LLM Systems

Condensed, skimmable reference. Full explanations live in `tutorial.md`; deep dives in `architecture.md`.

---

## Core Mental Model

| Concept | One-line definition |
|---|---|
| **Image** | Immutable, layered, read-only template (= a class). |
| **Container** | Running instance = image + thin writable layer (= an object). Never "fix" it by hand — rebuild the image. |
| **Layer** | One filesystem diff per `RUN`/`COPY`/`ADD`; content-addressed, cached, shared across images. |
| **Tag vs. digest** | Tag = mutable pointer (`:latest`). Digest = immutable hash (`@sha256:...`). **Pin critical prod images to digest.** |
| **Multi-stage build** | Multiple `FROM`s; `COPY --from=<stage>` pulls only compiled artifacts into a slim final stage. |

---

## The Golden Caching Rule

> Order Dockerfile instructions **least-frequently-changing → most-frequently-changing**.
> A cache miss at layer *N* invalidates every layer after it.

```dockerfile
# GOOD
FROM python:3.12-slim
COPY requirements.txt .
RUN pip install -r requirements.txt   # cached unless requirements.txt changes
COPY . .                              # only this busts on routine code edits
```

```dockerfile
# BAD — any code change re-triggers full dependency reinstall
FROM python:3.12-slim
COPY . .
RUN pip install -r requirements.txt
```

---

## Production Multi-Stage Dockerfile Skeleton

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS builder
RUN apt-get update && apt-get install -y --no-install-recommends build-essential gcc \
    && rm -rf /var/lib/apt/lists/*          # same RUN = cache doesn't survive into a layer
WORKDIR /build
COPY requirements.txt .
RUN python -m venv /venv \
    && /venv/bin/pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim AS final
RUN useradd --create-home --uid 1000 appuser   # non-root!
WORKDIR /app
COPY --from=builder /venv /venv
ENV PATH="/venv/bin:$PATH"
COPY --chown=appuser:appuser ./app ./app        # last = changes most often
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD \
    python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/ready')" || exit 1
ENTRYPOINT ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Always pair with a `.dockerignore`:**
```
.git
.venv
__pycache__/
*.pyc
.env
notebooks/
*.ipynb
tests/
data/raw/
```

---

## CUDA Base Image Decision Table

| Variant | Contains | Where it belongs |
|---|---|---|
| `base` | Minimal CUDA runtime | Rarely used alone |
| `runtime` | + math libs (cuBLAS, cuFFT) to *run* CUDA apps | **Final stage** (ship this) |
| `devel` | + `nvcc`, headers, compilers, to *build* CUDA apps | **Builder stage only** — never ship |

**Compatibility rule:** host driver ≥ container's CUDA toolkit requirement. Never the reverse.

```bash
# Verify host toolkit / driver
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi

# Run GPU workload — always raise --shm-size for PyTorch DataLoader / NCCL
docker run -d --gpus all --shm-size=8g -p 8000:8000 my-gpu-service:1.0

# Restrict to specific GPUs
docker run --gpus '"device=0,1"' ...
```

**`--shm-size` default is 64MB** — the #1 silent-crash cause (`Bus error`) in multi-worker PyTorch containers. Always raise it explicitly for training/multi-GPU inference.

---

## LLM Serving (vLLM) — Minimal Pattern

```dockerfile
FROM vllm/vllm-openai:latest
COPY server_config.yaml /etc/vllm/server_config.yaml
CMD ["--model", "meta-llama/Llama-3.1-8B-Instruct", \
     "--dtype", "bfloat16", "--gpu-memory-utilization", "0.90", \
     "--max-model-len", "8192", "--tensor-parallel-size", "1"]
```

```bash
docker run -d --gpus all --shm-size=16g -p 8001:8000 \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    -e HUGGING_FACE_HUB_TOKEN=${HF_TOKEN} \
    my-org/vllm-server:1.0
```

**Decision rule:** default to the official `vllm/vllm-openai` image + config layering. Build from source only for a genuinely custom patch/feature (`--build-arg VLLM_USE_PRECOMPILED=1` speeds custom builds).
**Always** volume-mount the HF cache — otherwise every restart re-downloads multi-GB weights (minutes of outage).

---

## The Three Endpoints Every ML Service Must Expose

| Endpoint | Purpose | Wired into |
|---|---|---|
| `/health` | Liveness — process is up | `HEALTHCHECK`, K8s `livenessProbe` |
| `/ready` | Readiness — model loaded & safe for traffic | Compose `healthcheck:`, K8s `readinessProbe` |
| `/metrics` | Prometheus exposition format | Prometheus `scrape_configs` |

Same three-endpoint contract expressed across Docker → Compose → Kubernetes — not incidental.

---

## Docker Compose Essentials

```yaml
services:
  api:
    build: {context: ./api, target: final}
    depends_on:
      redis: {condition: service_healthy}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/ready"]
      interval: 15s
      timeout: 3s
      retries: 5
    networks: [mlnet]
    deploy:
      resources: {limits: {cpus: "2.0", memory: 2g}}
networks:
  mlnet: {driver: bridge}
volumes:
  redis_data:
```

```bash
docker compose up -d --build     # use v2 plugin, never legacy docker-compose v1
docker compose logs -f api
docker compose ps
docker compose down -v           # -v also drops named volumes
```

**Critical caveat:** `depends_on: condition: service_healthy` is a **startup-order gate only** — not ongoing runtime resilience. If a dependency crashes later, Compose does not restart or block the dependent service; that's application-code territory (retries/circuit breakers), or Kubernetes' continuously-enforced probes.

**Compose is for local dev / CI integration tests — never a production deployment target.** It should structurally mirror the Kubernetes manifests it rehearses.

---

## Image Size Optimization — Ranked by Impact

1. **Multi-stage builds** — biggest lever, 50-90% cut for compiled/ML deps.
2. **Smallest sufficient base** — `slim` > full; `alpine` only if `musl` libc compatibility is verified for compiled wheels.
3. **Combine `RUN` + cleanup in one layer:**
   ```dockerfile
   RUN apt-get update && apt-get install -y --no-install-recommends foo \
       && rm -rf /var/lib/apt/lists/*
   ```
   (Splitting into two `RUN`s leaks the cache into the image permanently.)
4. **Prune Python deps** — avoid full `torch` when a CPU-only/CUDA-specific index wheel suffices; `pip install --no-cache-dir`.
5. **Aggressive `.dockerignore`.**
6. **Inspect with `docker image history` / `dive`** before assuming you're done.

| Base | Relative size | Shell? | Best for |
|---|---|---|---|
| `python:3.12` | Largest | Yes | Local dev only |
| `python:3.12-slim` | Medium | Yes | **Default for production** |
| `python:3.12-alpine` | Small | busybox | CPU-only pure-Python, verify wheels |
| `gcr.io/distroless/python3` | Smallest | No | Hardened prod, post-validation |
| `nvidia/cuda:*-runtime-*` | Large | Yes | GPU final stage |
| `nvidia/cuda:*-devel-*` | Largest | Yes | GPU builder stage only |

---

## Security Scanning (Trivy / Grype)

```bash
# CI gate (fails build on HIGH/CRITICAL with a fix available)
trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed my-org/rag-api:sha

# Fast local pre-push check
grype my-org/rag-api:local --fail-on high
```

```yaml
# GitHub Actions
- uses: aquasecurity/trivy-action@0.24.0
  with:
    image-ref: my-org/rag-api:${{ github.sha }}
    severity: HIGH,CRITICAL
    exit-code: "1"
    ignore-unfixed: true
```

**Threshold policy:** block merge on HIGH/CRITICAL with fix available → track MEDIUM on a dashboard → allow-list (with expiry date) genuine false positives. Never `--exit-code 0` to "unblock a deadline" — that defeats the gate entirely.

---

## Decision Rules (Quick Reference)

```
Need a GPU?              NO  → python:3.12-slim, multi-stage if compiled deps
                         YES → LLM serving?  YES → vllm/vllm-openai + config layer
                                              NO  → devel (builder) → runtime (final) + NVIDIA toolkit + --shm-size

How many services?       ONE      → plain docker run / single Dockerfile
                         MULTIPLE → Docker Compose (local dev / CI only)

Compose good enough for prod?  NO (need autoscaling/rolling updates/self-healing) → Kubernetes (Module 06)

Trivy vs Grype?          Single-tool broad coverage → Trivy
                         SBOM + VEX fine-grained suppression → Grype + Syft
                         Production reality → often run BOTH as cross-check
```

---

## Top 5 Mistakes to Never Make

1. `COPY . .` before `pip install` — busts the dependency cache on every code change.
2. Shipping `devel` CUDA images to production instead of `runtime`.
3. `FROM image:latest` (or `nvidia/cuda:latest`) — unpinned, silently mutates over time.
4. Baking secrets into `ENV`/`COPY` layers — permanently recoverable via `docker history`.
5. No `USER` instruction — container runs as root by default.

---

## Production Checklist (copy into your PR template)

```
[ ] Base images pinned to a version (digest for critical images)
[ ] Multi-stage build; final stage has no compilers/build tooling
[ ] Dockerfile ordered least-to-most frequently changing
[ ] .dockerignore excludes .git, notebooks, tests, venvs, raw data
[ ] Correct CUDA variant per stage (devel builder-only, runtime/base final)
[ ] Container runs as non-root USER
[ ] No secrets baked into ENV/COPY; runtime injection or --secret used
[ ] HEALTHCHECK / Compose / K8s probe defined and meaningful
[ ] --shm-size raised for PyTorch/NCCL workloads
[ ] Model/weights cache mounted as a volume
[ ] /metrics endpoint exposed (Prometheus format)
[ ] Trivy/Grype in CI, blocking on HIGH/CRITICAL with fixes available
[ ] Image size measured (docker image history / dive) before shipping
[ ] Compose file mirrors intended Kubernetes service/network/health shape
```
