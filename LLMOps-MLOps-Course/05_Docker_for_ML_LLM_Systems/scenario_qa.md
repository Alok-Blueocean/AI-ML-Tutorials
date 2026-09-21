# Module 05 — Scenario-Based Q&A

**Situation:** A teammate is debugging a production issue by SSHing into a running container and directly editing a config file to test a fix, planning to "make it permanent later." What would you do and why?

Model answer: Stop this pattern immediately and explain the image/container distinction it violates: the image is the immutable, versioned source of truth (analogous to a class definition), and the container is a disposable instance (analogous to an object) — mutating a running container is like mutating one object instance and hoping the class definition updates itself. The fix belongs in the Dockerfile or application code, rebuilt into a new image, and redeployed. Manually patched containers are invisible to version control, will be silently lost on the next restart or rollout, and break the entire premise that "what's deployed" is inspectable and reproducible from the image digest alone.

---

**Situation:** A CI build for a small FastAPI model-serving service currently takes 6 minutes for every commit, including one-line README changes, because the Dockerfile does `COPY . .` before `RUN pip install -r requirements.txt`.

Model answer: Diagnose this as the single most common Dockerfile performance mistake: copying application code before installing dependencies busts BuildKit's cached dependency-install layer on every commit, since a cache miss at any layer invalidates every layer after it, and application code changes far more often than `requirements.txt` does. Reorder to copy only `requirements.txt` first, run the install, then copy application code last — a code-only change now only invalidates the final `COPY` layer, and the multi-minute dependency install stays cached, typically dropping a documentation-only commit's build time to seconds.

---

**Situation:** An engineer is building a GPU-enabled inference image and includes `nvidia/cuda:12.4.1-devel-ubuntu22.04` as the single, final base image, reasoning "devel has everything runtime has, plus more, so it's safer to include."

Model answer: Correct this directly: `devel` should never ship in a final image — it includes the full CUDA toolkit (`nvcc` compiler, headers, static libraries) needed only to *build* CUDA applications, none of which is needed to *run* one, and shipping it bloats the image by gigabytes while needlessly widening the CVE-scanning surface. Recommend a multi-stage build: a builder stage `FROM nvidia/cuda:12.4.1-devel-ubuntu22.04` that compiles/installs what's needed, and a final stage `FROM nvidia/cuda:12.4.1-runtime-ubuntu22.04` that only copies over the already-built artifacts — `runtime` provides the math libraries (cuBLAS, cuFFT) needed to actually run a compiled CUDA/PyTorch application, which is all the shipped image needs.

---

**Situation:** A multi-worker PyTorch training container is crashing intermittently with cryptic `Bus error` messages, and the team has spent two days reviewing the training script for bugs with no luck.

Model answer: Redirect the investigation to the container configuration, not the application code: Docker's default shared-memory size (`/dev/shm`, 64MB) is a classic and easy-to-miss cause of exactly this crash pattern, since PyTorch's DataLoader multi-worker processes and NCCL multi-GPU communication both rely on shared memory that default limits don't accommodate. Fix by explicitly raising `--shm-size` (e.g., `docker run --shm-size=8g ...`, or the Compose/Kubernetes equivalent) — this single container-level flag, not a code change, is very likely to resolve it, and the two days spent reviewing training logic for a phantom bug illustrates why this failure mode is worth knowing by name.

---

**Situation:** A security review discovers a Hugging Face access token was baked into a Docker image via `ENV HF_TOKEN=abc123...` in the Dockerfile, and a later commit removed that line — but the team believes the secret is now safe since it's "not in the current version."

Model answer: Correct this misunderstanding urgently: the value persists in the image's layer history indefinitely and is trivially extractable via `docker history` or by anyone who can pull any tagged version of that image, even the ones predating the "removal" commit — removing an `ENV` line in a later layer does not erase it from earlier layers already baked into the image. Treat the token as compromised: rotate it immediately, and fix the underlying practice by injecting secrets at container runtime (environment variables passed to `docker run`/a Kubernetes Secret) or via BuildKit's `--secret` mount, neither of which persists into a layer.

---

**Situation:** A teammate wants to skip CVE scanning in CI for a new LLM-serving image "just for this release" because Trivy is flagging a HIGH severity finding and the deadline is tomorrow.

Model answer: Push back on disabling the gate rather than triaging the finding — this is explicitly named as a common mistake ("ignoring or blanket-suppressing CVE scan failures") that quietly defeats the entire purpose of the gate, and LLM-serving images specifically tend to have a large transitive dependency footprint while being frequently exposed to untrusted user input, raising the real stakes of skipping this check. Recommend triaging within the deadline instead: check whether a fix is available (`ignore-unfixed: true` already excludes CVEs with no patch), and if the finding is a genuine false positive or unexploitable in this deployment context, add it to a reviewed allow-list with an expiry date — not a blanket bypass of the gate itself.

---

**Situation:** A team's Docker Compose file for their RAG stack has grown into the actual production deployment mechanism over the past year, since "it already works and nobody got around to setting up Kubernetes."

Model answer: Flag this directly as one of the named common mistakes — treating `docker-compose.yml` as a production deployment mechanism rather than a local-dev/integration-test tool. Compose lacks the self-healing, multi-node scheduling, and rolling-update guarantees Kubernetes provides; a single-host Compose deployment is a single point of failure with no automatic recovery from a host-level failure and no way to scale beyond one machine's capacity. Recommend treating the existing Compose file as a valuable asset for local dev and CI integration tests (which it already demonstrably is), while planning a migration to Kubernetes as the actual production target — the fact that the Compose services/networking/health-check shape already mirrors what Kubernetes manifests need is a head start, not evidence Compose itself is production-ready.

---

**Situation:** An LLM-serving vLLM container is restarting several times a week due to routine redeployments, and each restart causes a multi-minute outage because the multi-gigabyte model weights re-download from Hugging Face every single time.

Model answer: Diagnose this as a missing volume mount for the model cache — without mounting the Hugging Face cache directory (e.g., `-v ~/.cache/huggingface:/root/.cache/huggingface`), every container restart re-downloads weight files from the hub, which is slow, wastes egress bandwidth, and turns a routine restart into a multi-minute outage. Fix by mounting the cache directory as a persistent volume so weights survive container restarts — this is a purely operational fix (a volume mount, not a code or model change) that directly addresses the actual symptom.

---

**Situation:** A junior engineer, building their first ML service image, chooses `python:3.12-alpine` as the base for a CPU-only service using `scipy` and `pandas`, reasoning "smaller is always better."

Model answer: Caution against the blanket "smaller is always better" heuristic here: Alpine's `musl` libc can break some compiled ML wheels that expect `glibc`, and this needs to be verified before committing to it, not assumed safe because the image is smaller. Recommend testing the actual `pip install` and import of `scipy`/`pandas` inside the Alpine-based image specifically before adopting it — if compatibility holds, Alpine's size win is real and worth keeping; if it doesn't, fall back to `python:3.12-slim`, which is the recommended default for most production Python services precisely because it doesn't carry this compatibility risk.

---

**Situation:** Your team is deciding whether to build a custom Dockerfile for vLLM from source, or use the official `vllm/vllm-openai` image and layer configuration on top, and an engineer argues building from source gives "more control."

Model answer: Recommend the official image as the default, and be specific about why "more control" isn't automatically better here: vLLM has intricate CUDA/PyTorch version pinning and a nontrivial build process, and the maintainers' official image is tested against that exact combination — building from scratch reintroduces exactly the CUDA/framework version-compatibility risk this module warns is easy to get subtly wrong. Reserve building from source for a specific, justified need (a custom patch, unusual hardware, a feature not yet in the published image), and when you do, use vLLM's documented `VLLM_USE_PRECOMPILED=1` build flag to avoid a full from-scratch compile — check the current docs before each upgrade, since flags and defaults shift release to release.

---

**Situation:** A production incident postmortem finds that the currently-deployed image for a fraud-scoring service is tagged `fraud-model:latest`, and nobody can determine exactly which code and dependency versions are actually running without inspecting the live container directly.

Model answer: Diagnose the root problem as using a mutable tag (`:latest`) as the deployment reference instead of an immutable digest — `:latest` silently changes what ships over time and defeats the entire point of reproducibility, since two deployments "of the same tag" weeks apart can be genuinely different images. Fix by always pinning to a specific version tag for development convenience, and for anything deployed to production, referencing the image by its immutable content digest (`@sha256:...`) — a Kubernetes Deployment or any deploy script should reference the digest a CI build produced, not a floating tag, so "what's running" is always answerable without live inspection.

---

**Situation:** A teammate proposes running containers as root by default in a new service, arguing "it's simpler and we don't add a `USER` instruction unless something forces us to."

Model answer: Push back — running as root is the implicit default if no `USER` instruction is added, and it means a container escape or an exploited application vulnerability has host-root-equivalent implications inside the container's namespace, which is a meaningfully worse security posture for no functional benefit in the vast majority of ML/LLM services. Recommend adding a non-root `USER` (e.g., `useradd --create-home --uid 1000 appuser` followed by `USER appuser`) as a default practice for every new service, reserving root only for the rare, explicitly justified case where a container genuinely needs elevated privileges (which should itself be flagged for extra scrutiny, not treated as routine).

---

**Situation:** A cost review shows the team's container registry storage bill has grown substantially, and an audit finds dozens of near-duplicate, multi-gigabyte images accumulated from GPU-serving builds over the past year.

Model answer: Name this as "image sprawl" and address the size problem at its source rather than just deleting old images: check whether these images are using multi-stage builds (the single biggest lever, routinely cutting size 50-90% for compiled/ML dependencies), whether `devel` CUDA variants are leaking into final stages, and whether `.dockerignore` is excluding notebooks, test fixtures, and raw datasets from the build context. Use `docker image history` or a tool like `dive` to visually inspect which layers actually contribute the bytes before assuming the images are already optimized — the registry storage cost and the CVE-scanning surface are both direct, measurable consequences of unresolved size issues, not just a monitoring afterthought.

---

**Situation:** A new engineer asks why the module's FastAPI example exposes three separate endpoints — `/health`, `/ready`, and `/metrics` — instead of just one `/status` endpoint that returns everything.

Model answer: Explain that these three endpoints serve structurally different purposes that get consumed by different systems and answer different questions: `/health` (liveness) answers "is the process up and able to respond to HTTP at all," `/ready` (readiness) answers the ML-specific question "is the model actually loaded and safe to receive real traffic" — these can diverge, e.g., right after container start the process is alive but the model isn't loaded yet — and `/metrics` exposes Prometheus-format data for the observability stack to scrape on its own interval. Collapsing them into one endpoint would force every consumer (a Docker `HEALTHCHECK`, a Kubernetes liveness/readiness probe, and Prometheus) to parse out the specific signal it needs from a combined payload, and would prevent expressing "alive but not ready" as a distinct, actionable state.

---

**Situation:** A team switches its multi-service local development setup from `docker-compose` (the standalone Python v1 binary, from an older tutorial) to the `docker compose` v2 plugin, and a teammate asks if this actually matters beyond a minor CLI syntax change.

Model answer: Confirm it matters beyond syntax: the standalone `docker-compose` v1 binary is deprecated, and new material and new projects should use the integrated `docker compose` v2 CLI plugin exclusively — v1's Python-based implementation is no longer where active development and bug fixes land. Beyond the deprecation status itself, this is also a good moment to verify the whole Compose file follows current conventions (healthcheck-gated `depends_on`, named volumes, resource limits) rather than assuming an old file just needs a command-prefix swap.

---

**Situation:** A GPU-serving image passes all local tests on an engineer's workstation with an NVIDIA driver supporting CUDA 12.6, but fails to start on a production host whose driver only supports up to CUDA 12.1.

Model answer: Diagnose this as a CUDA toolkit/driver compatibility mismatch, and explain the asymmetry precisely: the host driver is backward-compatible with *older* CUDA toolkit versions inside the container, within NVIDIA's documented compatibility matrix, but not the reverse — a container built against a newer CUDA toolkit than the host driver supports will fail. Fix by rebuilding against a CUDA toolkit version the production fleet's driver actually supports (or coordinating a driver upgrade across the fleet first), and going forward, standardize the CUDA toolkit version used in CI/build images against the oldest driver version still running anywhere in the production fleet, not against whatever happens to be newest on a single developer's workstation.

---

**Situation:** A teammate wants to skip Trivy/Grype scanning entirely for a small internal admin dashboard image, arguing "it's the same rigor we use for the customer-facing LLM service, and this one has way less exposure."

Model answer: Agree that scanning rigor can reasonably scale with exposure, but recommend against skipping entirely rather than tuning: even an internal-only image draws from the same package ecosystem and can still carry exploitable CVEs, particularly if it has any network exposure at all (even internal-network-only) or handles any sensitive data. A reasonable middle ground is keeping the scan as a CI step for every image without exception, but allowing a looser blocking threshold for genuinely low-exposure internal tools (e.g., block on CRITICAL only, track HIGH in a dashboard) rather than the stricter HIGH+CRITICAL blocking bar used for the customer-facing, untrusted-input-exposed LLM service — visibility stays universal even when the enforcement threshold varies by risk.

---

**Situation:** An LLM-serving image built with `docker build` locally is exactly 4.2GB. After moving to a proper multi-stage build with the correct `runtime` CUDA variant and a tightened `.dockerignore`, it drops to 1.1GB — but cold-start time in Kubernetes doesn't seem to have improved much despite the smaller image.

Model answer: Point out that image pull time is only one term in cold-start latency for an LLM-serving pod — the other major term is the vLLM model-weight load itself (potentially tens of seconds to minutes depending on model size), which is independent of the container image's own size once the Hugging Face cache is properly volume-mounted. Recommend measuring the two phases separately (image pull time vs. model load time) before concluding the size optimization "didn't help" — it likely did meaningfully improve pull time and registry cost, but the dominant cold-start term for this specific workload is the weight-loading step, which needs a different fix (a warm replica pool, or pre-baking weights onto node-local storage) rather than further image shrinking.

---

**Situation:** A new hire, coming from a pure web-development background, asks why ML/LLM teams seem to care so much more about Dockerfile layer ordering and image size than the web services teams they're used to.

Model answer: Name the ML/LLM-specific reasons directly: dependency fragility (PyTorch/CUDA/cuDNN/NCCL are version-sensitive in ways that can silently produce wrong numerical results, not just crashes, unlike a typical web framework mismatch), GPU-specific packaging (bridging into host GPU drivers is a whole additional axis of environment-parity risk with no web-service equivalent), and the sheer size of the artifacts involved (multi-gigabyte model weights and CUDA toolkits mean layering/caching strategy directly determines whether CI takes 90 seconds or 40 minutes, and whether an autoscaler can spin up a new inference replica in 10 seconds or 10 minutes). This last point especially — cold-start time as a capacity-planning concern — has no strong analogue in typical stateless web service scaling, where a new replica usually starts in a couple of seconds regardless of image care.
