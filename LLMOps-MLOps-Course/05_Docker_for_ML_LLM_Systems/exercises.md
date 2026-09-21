# Module 05 — Exercises: Docker for ML and LLM Systems

These exercises assume you have Docker Engine 24+ (BuildKit is the default builder) installed and working (`docker version` succeeds). GPU exercises (6-7) require a host with an NVIDIA GPU and the NVIDIA Container Toolkit installed — if you don't have one, read the "no-GPU alternative" note under each and do the reasoning exercise instead; do not skip the thinking.

Work through these in order. Each one builds on artifacts (files, images, mental models) produced by the one before it.

---

## Exercise 1 — Images vs. Containers, From First Principles

**Goal:** Build direct, hands-on intuition for the image/container distinction from Section 3.1 before touching a Dockerfile.

**Tasks:**
1. Run `docker run -it python:3.12-slim bash`. Inside the container, create a file: `echo "hello" > /tmp/proof.txt`. Exit the container.
2. Run `docker run -it python:3.12-slim bash` again (a fresh container from the same image) and confirm `/tmp/proof.txt` does **not** exist.
3. Run `docker ps -a` to find the *first* container's ID (it's stopped, not gone). Run `docker start -ai <container_id>` and confirm `/tmp/proof.txt` **does** exist this time.
4. Explain in your own words (2-3 sentences) why step 2 and step 3 gave different results, using the words "image," "container," and "writable layer."

**Done when:** You can articulate, without looking anything up, why mutating a running container is not the same as mutating an image — and why that fact is the entire basis for "rebuild and redeploy" as a rollback strategy.

---

## Exercise 2 — Fix a Slow-Rebuilding Dockerfile

**Goal:** Internalize the layer-caching rule from Section 3.2 by fixing a deliberately bad Dockerfile.

**Setup:** Create this project structure:
```
bad-build/
├── Dockerfile
├── requirements.txt   (put: fastapi==0.115.0, uvicorn==0.30.6, pydantic==2.9.2)
└── app/main.py         (any minimal FastAPI app — reuse the tutorial's app/main.py)
```

Write the intentionally bad Dockerfile:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

**Tasks:**
1. Build it: `time docker build -t bad-build:1 .` Note the wall-clock time.
2. Change one comment/whitespace line in `app/main.py` only. Rebuild and time it again. Notice the dependency install re-runs.
3. Rewrite the Dockerfile applying the "least-to-most frequently changing" rule from Section 3.2 (copy `requirements.txt` first, install, *then* copy app code). Add a `.dockerignore` per the tutorial's example.
4. Repeat step 2 against your fixed Dockerfile and confirm the pip-install layer now shows `CACHED` in the build output.

**Done when:** You can show (via `docker build` output, not just claim) that a code-only change produces a `CACHED` hit on the dependency layer in the fixed version but not the original, and you can explain the cache-key mechanism (Architecture doc Section 2) that causes this.

---

## Exercise 3 — Multi-Stage Build and Measured Size Reduction

**Goal:** Move from "I understand multi-stage builds conceptually" to "I can prove the size reduction with real numbers on my machine" (Section 3.2, Section 3.6).

**Tasks:**
1. Using the fixed Dockerfile from Exercise 2, add a package with a native/compiled dependency to `requirements.txt` (e.g., `numpy==2.1.1` is enough to require build tooling on some base images, or intentionally add `build-essential`-requiring package if your platform doesn't need it for numpy).
2. Build a **single-stage** version that installs `build-essential`, `gcc`, and the requirements directly into the final image. Tag it `size-demo:single-stage`.
3. Build a **multi-stage** version (builder stage with compilers → final stage `COPY --from=builder /venv /venv`, no compilers in final). Tag it `size-demo:multi-stage`.
4. Compare with `docker images size-demo` and record the size difference in MB/GB.
5. Run `docker image history size-demo:single-stage` and `docker image history size-demo:multi-stage` side by side and identify exactly which layer(s) account for the difference.

**Done when:** You have a recorded before/after size number (your own measured number, not a memorized industry figure) and can point to the specific layer in `docker image history` responsible for the bulk of the savings.

---

## Exercise 4 — Add Liveness, Readiness, and Metrics — Then Break Readiness on Purpose

**Goal:** Understand why the three-endpoint contract (`/health`, `/ready`, `/metrics`) in Section 3.2 is not decorative, by observing what happens when readiness is wrong.

**Tasks:**
1. Take the `app/main.py` from the tutorial (or your Exercise 2 app) and containerize it with the full production-shaped Dockerfile from Section 3.2, including the `HEALTHCHECK` instruction.
2. Run it, then poll `docker inspect --format='{{json .State.Health}}' <container>` a few times to watch the health status transition.
3. Now deliberately break readiness: modify `lifespan()` so `model_state["ready"]` is never set to `True` (simulate a model file that fails to load, e.g., point `MODEL_PATH` at a nonexistent file and catch-and-log instead of crashing). Rebuild and rerun.
4. Confirm `/health` still returns 200 (process is alive) while `/ready` returns 503 (not safe for traffic), and that `HEALTHCHECK` (which should point at `/ready`, not `/health` — spot the bug in the tutorial's example if you look closely, or use this as the trigger to reason about which endpoint a HEALTHCHECK/readinessProbe *should* target) reflects an unhealthy container.
5. Write down, in one paragraph, why a load balancer or Kubernetes Service must never route traffic to a container that is alive-but-not-ready, and which of the two endpoints prevents that.

**Done when:** You've observed the alive-but-not-ready state directly in `docker inspect` output (not just reasoned about it), and can explain the liveness-vs-readiness distinction using your own broken build as the example.

---

## Exercise 5 — Compose a Three-Service Local Stack With Real Health-Gated Startup

**Goal:** Build the Compose muscle memory from Section 3.5 with a stack simple enough to fully understand, before tackling the full RAG+LLM stack.

**Tasks:**
1. Write a `docker-compose.yml` with three services: your `api` (from Exercise 4), `redis:7.4-alpine`, and `prometheus`. Give `api` a real dependency on `redis` (e.g., cache one field of the prediction response in Redis) so the connection is not decorative.
2. Add `depends_on: redis: condition: service_healthy` on `api`, and a real `healthcheck:` block on `redis`.
3. Bring the stack up with `docker compose up -d --build`, then deliberately stop Redis mid-run (`docker compose stop redis`) while `api` keeps running. Send a request to `api` that touches Redis and observe what happens.
4. Using this observation, explain (referencing the Architecture doc's Section 3 sequence diagram) why `depends_on: condition: service_healthy` is a **startup-order** guarantee only, not an ongoing resilience guarantee — and sketch (pseudocode is fine, you don't have to implement it) what a retry/circuit-breaker would look like in `api`'s Redis client code.
5. Tear down with `docker compose down -v` and confirm the named volume is gone.

**Done when:** You've reproduced, on your own machine, the exact "depends_on is not runtime resilience" gap the Architecture doc calls out — by breaking it yourself, not by reading about it.

---

## Exercise 6 — GPU-Enabled Container (or Reasoned Walkthrough if No GPU)

**Goal:** Apply the CUDA base-image variant rules (Section 3.3) to a real or hypothetical build.

**If you have an NVIDIA GPU + NVIDIA Container Toolkit installed:**
1. Verify the toolkit: `docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi`. Confirm you see your GPU listed.
2. Write a multi-stage Dockerfile: builder stage on `nvidia/cuda:12.4.1-devel-ubuntu22.04` that `pip install`s a package requiring a CUDA-aware build (or simulate with any package needing compilation), final stage on `nvidia/cuda:12.4.1-runtime-ubuntu22.04`.
3. Run the final image with `--gpus all --shm-size=2g` and confirm your Python process can see the GPU (e.g., via a one-line `torch.cuda.is_available()` check if you have `torch` installed in the image).
4. Deliberately omit `--shm-size` and, if you can construct a workload that uses `/dev/shm` heavily enough (e.g., a multi-worker DataLoader with several workers over a reasonably sized in-memory dataset), reproduce a shared-memory-related failure. If you cannot reproduce a crash reliably on your hardware, instead run `df -h /dev/shm` inside a container started with the default 64MB and again with `--shm-size=2g`, and record the difference.

**If you do NOT have a GPU:**
1. Write out (in a text file) a complete multi-stage Dockerfile that correctly uses `devel` for the builder stage and `runtime` for the final stage, for a hypothetical service that compiles a custom CUDA kernel extension.
2. Annotate every line explaining *why* it's in the builder stage vs. the final stage vs. neither.
3. Write the exact `docker run` command you would use to launch it, including `--gpus`, `--shm-size`, and a bind-mount for a model cache directory, and justify each flag in one sentence.

**Done when:** You have a working (or fully annotated hypothetical) multi-stage GPU Dockerfile, and you can state from memory why `devel` must never appear in the final stage.

---

## Exercise 7 — Package and Run a Real LLM-Serving Container

**Goal:** Apply Section 3.4 to stand up an actual OpenAI-compatible LLM server, including the model-cache volume-mount lesson.

**Tasks:**
1. Pull `vllm/vllm-openai:latest` (or, if GPU/VRAM constrained, substitute a CPU-friendly small model server such as `llama.cpp`'s server image, but keep the volume-mount and restart-cost lesson identical).
2. Run it once **without** mounting a persistent cache volume, and time how long the container takes from `docker run` to the model being ready to answer requests (watch the logs for the "ready" / server-listening line).
3. Stop and remove the container, then run it again **without** the volume mount and time it again — confirm the weights had to be re-fetched (check network activity or the download-progress log lines) and the time is roughly the same as the first run (i.e., no caching benefit).
4. Now run it a third time **with** `-v ~/.cache/huggingface:/root/.cache/huggingface` mounted, stop/remove/rerun it, and time the *second* run with the mount present — confirm it is now dramatically faster because weights were reused from the host-mounted cache.
5. Send a real request to the OpenAI-compatible endpoint (`curl http://localhost:8000/v1/chat/completions -d '{...}'`) and confirm you get a completion back.

**Done when:** You have two recorded timings — cold (no cache mount) vs. warm (cache mount, second run) — proving the operational point Section 3.4 makes about restart cost, on your own hardware and your own numbers.

---

## Exercise 8 (Capstone) — Full Stack, Security-Gated, Kubernetes-Ready

**Goal:** Integrate everything in this module into one coherent, CI-gated, production-shaped deliverable — the natural bridge into Module 06.

**Tasks:**
1. Assemble the full stack from the tutorial's Section 3.5 `docker-compose.yml` (api + qdrant + redis + prometheus + grafana), using your own `api` image built via the multi-stage Dockerfile from Exercise 2/3, with the health/readiness/metrics contract from Exercise 4.
2. Add a `llm-server` service (vLLM or your Exercise 7 substitute) to the Compose file, wired so `api` can call it over the `mlnet` network by service name (no hardcoded IPs).
3. Write a GitHub Actions workflow (or an equivalent local script using `act` or a plain shell script if you don't have CI access) that: builds the `api` image, runs Trivy (or Grype) against it with `--severity HIGH,CRITICAL --exit-code 1`, and only proceeds to `docker compose up` if the scan passes.
4. Deliberately introduce a known-vulnerable pinned package version into `requirements.txt` (check Trivy/Grype's advisory data for a real example, or pin an old version of a package you know has a published CVE) and confirm your pipeline **fails the build** — then fix it and confirm it passes.
5. Produce a one-page "production checklist" (you may reuse the tutorial's Section 8 checklist as a template) with every box checked off against your actual stack, plus a short paragraph mapping each Compose concept (`mlnet`, named volumes, `depends_on`, `healthcheck`) to the Kubernetes primitive it will become in Module 06 (Service/CNI network, PersistentVolumeClaim, init-container-or-readiness-gate pattern, liveness/readinessProbe).

**Done when:** You have a working `docker compose up` bringing up all five-plus services with health-gated startup, a CI (or CI-equivalent) pipeline that provably blocks on a known CVE and passes once fixed, and a completed checklist with the Kubernetes-mapping paragraph — this artifact is what you carry forward into Module 06.
