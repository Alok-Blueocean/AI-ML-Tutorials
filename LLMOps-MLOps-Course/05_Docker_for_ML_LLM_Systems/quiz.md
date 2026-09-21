# Module 05 — Quiz: Docker for ML and LLM Systems

Instructions: Attempt every question before checking the answer key at the end. Questions 1-8 are multiple-choice (one best answer unless stated otherwise); questions 9-15 are short-answer. This quiz covers `tutorial.md` and `architecture.md`.

---

**Q1.** Which statement correctly distinguishes a Docker *image* from a *container*?

A. An image is a running process; a container is the file that describes it.
B. An image is an immutable, layered, read-only template; a container is a running (or stopped) instance with its own thin writable layer on top of that image.
C. Images and containers are two names for the same artifact, used interchangeably in modern Docker documentation.
D. A container becomes an image once you run `docker commit` on it, and only then can it be started.

---

**Q2.** You have a Dockerfile that copies the entire application directory (`COPY . .`) *before* running `pip install -r requirements.txt`. What is the direct consequence?

A. The build will fail because `requirements.txt` is not yet present.
B. Every change to any application source file — even a comment — invalidates the cache for the `pip install` layer, forcing a full dependency reinstall on every build.
C. Nothing; BuildKit reorders instructions automatically for optimal caching.
D. It only matters for images larger than 1GB.

---

**Q3.** Which `nvidia/cuda` image variant should appear in the **final, shipped stage** of a multi-stage build for a GPU inference service (assuming no custom CUDA kernel compilation happens at container startup)?

A. `devel` — you need the compiler in case something needs to be built at runtime.
B. `runtime` (or `base`, if sufficient) — it has the CUDA math libraries needed to *run* a compiled CUDA/PyTorch application, without the compiler toolchain.
C. Whichever variant is smallest on disk, regardless of contents.
D. There is no meaningful difference between the variants for shipping purposes.

---

**Q4.** In the module's production-recommended default for packaging an LLM-serving engine like vLLM, what is the recommended approach?

A. Always build vLLM from source in your own Dockerfile, so you fully control every dependency.
B. Start from the official `vllm/vllm-openai` image, layering only your own config/customization on top, and rebuild from source only when a custom feature or patch is genuinely needed.
C. Never use Docker for LLM serving engines; run them directly on bare metal only.
D. Package the model weights directly into the image at build time for every deployment.

---

**Q5.** What is the primary operational risk of baking a secret (e.g., a Hugging Face token) into an image via `ENV HF_TOKEN=...` in a Dockerfile, even if a later instruction appears to remove or overwrite it?

A. None — as long as the final `docker inspect` output doesn't show the variable, it is safe.
B. The secret persists in that layer's history indefinitely and is extractable via `docker history` or by anyone who can pull the image, regardless of what later layers do.
C. It only matters if the image is pushed to a public registry; private registries are immune.
D. Docker automatically encrypts all `ENV` values at rest, so this is a non-issue.

---

**Q6.** Which of the following is **NOT** one of the three HTTP endpoints the module recommends every production ML service expose?

A. `/health` (liveness)
B. `/ready` (readiness — model actually loaded)
C. `/metrics` (Prometheus exposition format)
D. `/admin` (privileged control-plane operations)

---

**Q7.** In Docker Compose, what does `depends_on: <service>: condition: service_healthy` actually guarantee?

A. That if the dependency service crashes at any point after startup, the dependent service will automatically be restarted or blocked from receiving traffic.
B. That the dependent service will only be **started** after the dependency reports healthy — a one-time startup-order gate, not an ongoing runtime resilience guarantee.
C. That the two services will always run on the same underlying host.
D. That Compose will automatically retry failed requests between the two services indefinitely.

---

**Q8.** Which combination of techniques, per the module's ranked list, has the single biggest impact on reducing final image size for compiled/ML dependencies? (Select the single best answer.)

A. Switching from `python:3.12` to `python:3.12-alpine` alone.
B. Multi-stage builds — separating build-time tooling from the runtime image.
C. Adding more `.dockerignore` entries.
D. Running `docker system prune` after every build.

---

**Q9 (short answer).** Explain, in 2-3 sentences, why "layer order matters enormously" in a Dockerfile — connect your answer to how BuildKit computes cache keys.

---

**Q10 (short answer).** A teammate's PyTorch training container is crashing intermittently with a `Bus error` during multi-worker data loading, but only inside Docker — it never happens when the same code runs directly on the host. What is the most likely container-level (not application-level) root cause, and what is the fix?

---

**Q11 (short answer).** Why must the host's NVIDIA driver be *newer* than (or compatible with) the CUDA toolkit version inside the container, and not the other way around? What concretely goes wrong if you get this backwards?

---

**Q12 (short answer).** Describe, in your own words, why Docker Compose is described in this module as "a low-fidelity rehearsal" of Kubernetes rather than a production deployment tool. Name at least two specific capabilities Kubernetes provides that Compose does not.

---

**Q13 (short answer).** As of 2026, why is CVE scanning (Trivy/Grype) treated as a merge-blocking CI gate for ML/LLM images specifically — what about these images' dependency footprint and exposure makes this more urgent than for a typical CRUD web service? Give at least two concrete reasons.

---

**Q14 (short answer).** You are asked to review a colleague's Dockerfile for a GPU-enabled inference service and notice `FROM nvidia/cuda:latest` and no `USER` instruction anywhere. List the two distinct problems this creates and what you would tell them to do instead for each.

---

**Q15 (short answer).** Explain the difference between a **tag** and a **digest** when referring to a container image, and why production deployments should prefer pinning to a digest for the most critical images.

---
---

## Answer Key

**A1.** **B.** An image is the immutable, layered read-only template (analogous to a class); a container is a running instance with its own thin writable layer (analogous to an object). You never fix production by mutating a running container — you rebuild the image and redeploy.

**A2.** **B.** BuildKit's cache key for a layer depends on the instruction plus everything below it; copying code before installing dependencies means *any* source change (even whitespace) invalidates the layer that installs dependencies, forcing a full, often multi-minute, reinstall on every build. (Option C is wrong — BuildKit does not reorder your instructions for you; you must order them correctly yourself.)

**A3.** **B.** `runtime` (or `base` if sufficient) contains the CUDA math libraries (cuBLAS, cuFFT, etc.) needed to *run* a compiled CUDA/PyTorch application without the compiler toolchain, headers, and `nvcc` that `devel` adds — those are builder-stage-only and should never ship, since they only add size and CVE surface with zero runtime benefit.

**A4.** **B.** The pragmatic 2026 default is to start from the maintainer's official `vllm/vllm-openai` image (tested against a known-good CUDA/PyTorch/vLLM version combination) and layer your own config on top; building fully from source is sometimes necessary but is slower and higher-maintenance, appropriate only when you need a genuinely custom patch or feature.

**A5.** **B.** Docker image layers are immutable and content-addressed; once a secret is written into any layer (via `ENV` or `COPY` of a file containing it), it remains recoverable from that layer's history forever, regardless of later instructions that appear to remove or overwrite it. The fix is runtime injection (env vars passed at `docker run`/Kubernetes Secret) or BuildKit's `--secret` mount, which is never persisted into a layer.

**A6.** **D.** `/health`, `/ready`, and `/metrics` are the three endpoints called out repeatedly in the module (they tie the `HEALTHCHECK` instruction, Compose `healthcheck:` blocks, Kubernetes liveness/readiness probes, and Prometheus scraping into one coherent contract). `/admin` is not part of this pattern and, if it existed, would need to be locked down separately, not exposed as part of the health/observability contract.

**A7.** **B.** It is a one-time startup-order gate only. If the dependency crashes and restarts *after* the dependent service has already started, Compose does not automatically restart or block the dependent service — any ongoing runtime resilience (retries, circuit breakers) has to be implemented in application code. Kubernetes' liveness/readiness probes extend this into a continuously-enforced guarantee, which Compose does not provide.

**A8.** **B.** Multi-stage builds are called out as the single biggest lever, routinely cutting image size by 50-90% for compiled/ML dependencies, because they let you discard the entire build-time toolchain (compilers, headers, intermediate build artifacts) and ship only the compiled result.

**A9.** BuildKit computes each layer's cache key from (roughly) the instruction text plus the cache key/state of the layer immediately below it, forming a dependency chain. A cache miss at any layer *N* therefore invalidates every layer after it in that chain, even if those later layers' own instructions didn't change — so instructions that change rarely (base image, system deps, dependency manifest + install) belong early, and instructions that change often (application code) belong last, to minimize how much gets invalidated on a typical commit.

**A10.** Most likely cause: Docker's default shared-memory size (`/dev/shm`, 64MB) is too small for PyTorch's DataLoader worker processes (and NCCL communication for multi-GPU setups), which rely on `/dev/shm` for inter-process communication — the host isn't subject to this limit, which is why it only reproduces inside the container. Fix: explicitly raise it at container start, e.g. `docker run --shm-size=8g ...` (or the equivalent `shm_size:` field in Compose, or an `emptyDir` with `medium: Memory` and an explicit `sizeLimit` in Kubernetes).

**A11.** NVIDIA's compatibility model is backward-compatible in one direction only: a host driver is compatible with an equal or *older* CUDA toolkit version inside the container (within NVIDIA's documented matrix), but a container built against a newer CUDA toolkit than the host driver supports will fail to run correctly (or at all) — the driver simply doesn't know how to service API calls from a CUDA runtime newer than itself. Getting this backwards typically surfaces as CUDA initialization errors or unsupported-operation failures at container startup, not a graceful fallback.

**A12.** Compose models the same conceptual shape as a Kubernetes deployment — services, a network for service discovery, health checks, dependency ordering, resource limits — but only for a single host/daemon, with no built-in self-healing, no multi-node scheduling, and no rolling/canary update mechanism. Two specific Kubernetes capabilities Compose lacks: (1) automatic, continuous self-healing/restart-and-reschedule when a container or node fails at runtime (Compose's health checks gate startup order only, per Q7/A7); (2) rolling or canary updates across multiple nodes with traffic shifting and automated rollback, which Compose has no native concept of at all.

**A13.** Two concrete reasons: (1) ML/LLM images tend to have an unusually large transitive dependency footprint — CUDA userspace libraries, PyTorch/ML framework stacks, native-compiled tokenizer libraries (e.g., Rust-based), and web-framework serving stacks all stacked together — which multiplies the number of packages that could carry a known CVE compared to a typical lightweight web service image. (2) LLM-serving endpoints are frequently exposed directly to untrusted user input (prompts, uploaded files), which increases the practical exploitability of any given vulnerability relative to an internal-only batch job — making the same CVE a higher-severity real-world risk in this context.

**A14.** Two distinct problems: (1) `FROM nvidia/cuda:latest` is an unpinned mutable tag — the exact contents of the image can silently change over time (a new push to `latest` changes what "the same Dockerfile" produces), breaking reproducibility; fix by pinning to a specific version tag (e.g., `nvidia/cuda:12.4.1-runtime-ubuntu22.04`), and for the most critical production images, to an immutable digest. (2) No `USER` instruction means the container runs as root by default; a container escape or any code-execution vulnerability in the running process then has host-root-equivalent implications inside its namespace — fix by adding a non-root user (`useradd --create-home --uid 1000 appuser` and `USER appuser`) as shown in the module's production Dockerfile.

**A15.** A **tag** (e.g., `vllm/vllm-openai:latest`) is a mutable, human-readable pointer that can be reassigned to a different underlying image at any time — pulling the "same" tag tomorrow can silently produce a different image than it did today. A **digest** (e.g., `@sha256:abcd...`) is an immutable, content-addressed hash of the image's exact contents — pulling the same digest always produces byte-for-byte the same image. Production deployments prefer digest-pinning for critical images because it guarantees that what was tested/scanned in CI is precisely and provably what runs in production, and that a rollback ("point at the previous digest") is unambiguous, whereas a tag alone offers no such guarantee.
