# Module 05 — Example Projects: Docker for ML and LLM Systems

Three projects, increasing in scope and production-realism, all built on this module's material. Each specifies scope and what it demonstrates — build them in order; the Medium project's image is the thing the Production project wraps in CI/CD and scanning, and the Production project's Compose stack is what Module 06 later re-expresses as Kubernetes manifests.

---

## Mini — Containerize a Single Scikit-Learn Model as a FastAPI Microservice

**Scope:** Take a small, already-trained scikit-learn model (e.g., a logistic regression or random forest on the Iris or a similar toy tabular dataset — reuse the Iris model if you built one in an earlier module) and package it as a single, well-formed Docker image.

**What to build:**
- A FastAPI app exposing `/predict`, `/health`, `/ready`, and `/metrics` (Prometheus format), following the exact three-endpoint contract from Section 3.2.
- A **multi-stage** Dockerfile: builder stage installs dependencies into a venv, final stage on `python:3.12-slim` copies only the venv and app code, runs as a non-root `USER`, and defines a `HEALTHCHECK` pointed at `/ready`.
- A correct `.dockerignore`.
- A pinned base image tag (not `:latest`).

**What it demonstrates:**
- The image/container distinction and multi-stage build mechanics (Section 3.1-3.2) applied end to end on the simplest possible model-serving payload.
- That you can reason about layer ordering and prove a cache hit/miss with `docker build` output.
- The liveness/readiness/metrics contract that every later, larger project in this module (and Module 06) builds on.

**Definition of done:** `docker build` produces an image under a size budget you set yourself (e.g., under 300MB); `docker run` starts it; `curl` against all four endpoints returns correct responses; a one-line code change rebuilds in a few seconds due to cache hits on the dependency layer; `docker image history` shows no compiler/build tooling in the final image.

---

## Medium — Multi-Service RAG Stack With Docker Compose and a Local Vector DB

**Scope:** Move from "one container" to a realistic multi-service local development stack: a retrieval-augmented-generation API backed by a vector database and a cache, fully declared and orchestrated via Docker Compose v2, with monitoring wired in from day one.

**What to build:**
- An `api` service (FastAPI) with endpoints for ingesting documents, querying with retrieval + a cached LLM call, plus `/health`, `/ready`, `/metrics`.
- A `qdrant` (or equivalent) vector database service with a named, persisted volume.
- A `redis` service used for both session state and a semantic/response cache, with a real `healthcheck:` block.
- `prometheus` scraping `api` and `qdrant`, and `grafana` with at least one dashboard (latency, request count, cache hit rate).
- A full `docker-compose.yml` on a user-defined bridge network, with `depends_on: condition: service_healthy` correctly gating startup order, and CPU/memory `deploy.resources.limits` set on every service.
- A deliberate small chaos test: stop `redis` mid-run and verify (and document) whether `api` degrades gracefully or fails hard — then decide, and implement, the retry/fallback behavior you want.

**What it demonstrates:**
- Section 3.5's full multi-service Compose pattern: service discovery by container name, health-gated startup ordering, and the difference between startup-order guarantees and runtime resilience (Architecture doc Section 3 — the `depends_on` caveat).
- That the resulting Compose file already has the same conceptual shape (services, network, health checks, resource limits) that Module 06 will re-express as Kubernetes Deployments/Services/PVCs — this project is deliberately built to make that transition structural rather than a redesign.
- Practical vector-DB and cache integration patterns used in real RAG systems.

**Definition of done:** `docker compose up -d --build` brings up all five services with all health checks passing; an end-to-end RAG request (ingest → embed → retrieve → generate → cache) succeeds via `curl`; Grafana shows live request-rate/latency data; the Redis chaos test result is documented along with the fix you implemented (or the deliberate decision not to, with reasoning); `docker compose down -v` cleanly removes all state.

---

## Production-Grade — CI-Gated, GPU-Backed LLM Serving Platform With Security Scanning

**Scope:** Extend the Medium project into something that would plausibly survive a real production review: add a real GPU-backed LLM-serving component, a CI/CD pipeline that builds, scans, and gates every image before it can be deployed, and full digest-pinned, non-root, secrets-clean image hygiene throughout — this is the direct predecessor artifact to what Module 06 turns into a Kubernetes deployment.

**What to build:**
- Everything from the Medium project, **plus** an `llm-server` service built on `vllm/vllm-openai:latest` (or an equivalent OpenAI-compatible engine if GPU resources are constrained), with the Hugging Face cache volume-mounted (never re-downloading weights on restart), `--gpus`/`--shm-size` correctly configured, and `api` calling it over the Compose network by service name.
- A GitHub Actions (or equivalent) pipeline that: builds every image with BuildKit, runs Trivy **and** Grype against each (as a cross-check per the Architecture doc's decision tree), fails the build on any HIGH/CRITICAL finding with an available fix, and only then pushes each image to a registry tagged with the git SHA **and** pinned by digest.
- A documented, reviewed CVE allow-list (with expiry dates) for any findings that are genuine false positives or non-exploitable in this deployment context — not a blanket suppression.
- Every image running as a non-root `USER`, with zero secrets baked into any layer (verify this yourself with `docker history` against every image as a final gate) — API keys and the HF token injected only at container-start time.
- A measured image-size report (`docker image history` / `dive`) for every service, with the multi-stage builder-vs-final size delta recorded for the GPU image specifically.
- A one-page runbook mapping every Compose primitive used (`mlnet` network, named volumes, `depends_on`/`healthcheck`, resource limits, `--gpus`) to the Kubernetes primitive it will become in Module 06.

**What it demonstrates:**
- The full Section 3.7 CI-gating pattern as a merge-blocking, non-optional step — the 2026 baseline expectation, not periodic hardening.
- Correct GPU containerization end to end: CUDA variant discipline, NVIDIA Container Toolkit usage, and the operational weight-cache-volume lesson from Section 3.4, under a real CI pipeline rather than a manual `docker run`.
- Full production image hygiene (digest pinning, non-root, secret-free layers, measured size) as a single coherent, auditable pipeline output — precisely the artifact a Kubernetes Deployment (Module 06) will reference by digest.
- The ability to reason about — and defend, in an interview-style review — every choice in the pipeline: why Trivy and Grype both, why digest over tag, why the allow-list has expiry dates, why the LLM server's cold-start time matters for autoscaling design.

**Definition of done:** a single CI run, triggered by a pull request, builds all images, runs both scanners, blocks the merge on a deliberately introduced known-CVE dependency, and passes cleanly once fixed; the resulting images are pushed by digest to a registry; `docker compose up` (using the pushed digests, not local builds) brings up the full stack including the GPU-backed `llm-server` with a warm model cache; an end-to-end RAG request that traverses `api → qdrant → redis → llm-server` succeeds and is visible in Grafana; the runbook's Compose-to-Kubernetes mapping table is complete and accurate enough to hand to someone about to start Module 06 cold.
