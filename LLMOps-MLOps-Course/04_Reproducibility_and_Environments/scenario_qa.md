# Module 04 — Scenario-Based Q&A

**Situation:** A model that scored 0.94 F1 in a notebook on Tuesday scores 0.81 F1 after being redeployed on Thursday — same code, same data, same model weights. What would you do and why?

Model answer: Treat this as the module's core lesson made concrete: output is a function of code, data, model weights, *and* environment, and the honest equation has all four terms. Compare environments directly — run `pip list --format=json` (or `pip freeze`) on the known-good machine and the failing one, diff them, and look for a version mismatch in a numerically sensitive library first (NumPy, PyTorch, a BLAS backend). Confirm root cause with a minimal repro (e.g., compute a fixed operation under both versions and compare outputs), then fix by pinning the suspect package back and re-testing. Prevent recurrence by adding an automated environment-diff check to CI, run before promotion, not just during incident response.

---

**Situation:** A teammate's `requirements.txt` contains `numpy==1.26.4`, and they argue this is fully pinned and therefore reproducible.

Model answer: Correct the misconception directly: a version pin alone assumes the package registry never re-uploads or corrupts an artifact under the same version string, and does nothing to protect against a compromised mirror or an accidentally-published broken build. Version pinning protects against drift over time on the same registry, not against tampering or artifact substitution — hash verification is the layer that closes that gap. Recommend generating a lockfile with hashes (`pip-compile --generate-hashes`, or a tool that does this natively like `uv`, Poetry, or `conda-lock`) and installing with `--require-hashes` so pip refuses to install anything whose downloaded bytes don't match the recorded SHA-256.

---

**Situation:** A Dockerfile uses `FROM python:3.11-slim`, and a teammate insists this is already pinned because it names a specific minor version.

Model answer: Explain why this is a false sense of security: the tag names a minor version, but it's a mutable pointer — the maintainers republish images under that same tag whenever they patch the underlying OS packages, so two builds of the identical Dockerfile, weeks apart, can pull different bytes. The fix is digest pinning: `FROM python:3.11-slim@sha256:<digest>`, which can only ever resolve to one exact set of bytes. Pair this with an automated update process (Renovate or Dependabot configured to open PRs on digest changes), since digest pinning without an update path just trades "silent drift" for "silently frozen on a stale, potentially vulnerable image."

---

**Situation:** Your team is building a GPU-serving image and someone reaches for `nvidia/cuda:latest` out of habit, the same way they'd use `python:3-slim` for a quick script.

Model answer: Stop them immediately — as of mid-2026, the `nvidia/cuda:latest` tag is deprecated and unavailable on both Docker Hub and NGC, so pulling it returns a `manifest unknown` error; this isn't a style suggestion anymore, it's a hard operational requirement. Explain the deeper point underneath the specific fact: the real mistake is relying on any floating tag for a GPU base image, version-only tags included (`12.4-runtime-ubuntu22.04` is still a mutable pointer subject to patch republishing) — pin to an explicit version and digest, and additionally verify the PyTorch/framework build matches the pinned CUDA version (e.g., a `+cu124` wheel against a cu124 base).

---

**Situation:** A data scientist runs `conda env export > environment.yml` after finishing a project and stores it as "the reproducible environment," planning to use it to recreate the setup on a new machine six months later.

Model answer: Warn that `conda env export` (the full, non-`--from-history` form) is not a substitute for a real lockfile — it captures a point-in-time resolution on the exporting machine's specific platform, and is not guaranteed to reproduce identically elsewhere or later, because it doesn't freeze the full transitive dependency graph with hashes. Recommend `conda-lock lock --file environment.yml --platform linux-64 --platform osx-arm64` (or whatever platforms are actually needed) to produce a genuinely hash-pinned, per-platform `conda-lock.yml`, and treat the human-edited `environment.yml` as a declaration of intent, not the reproducibility guarantee itself.

---

**Situation:** A production incident is traced to a transitive dependency — a package nobody on the team directly declared, required by something they did declare — that silently bumped to a new major version and changed a default behavior.

Model answer: Use this as the concrete argument for lockfile-based pinning over direct-dependency-only pinning: a `requirements.in` or `pyproject.toml` with loose ranges only constrains what you explicitly wrote, but transitive dependencies are just as capable of introducing drift and are invisible to a review of your direct declarations alone. The fix is a machine-generated lockfile (`uv.lock`, `poetry.lock`, or a hash-pinned `requirements.txt`) that pins the entire resolved dependency tree, direct and transitive, so a transitive bump can only happen through a deliberate `uv lock`/`poetry lock` re-resolution that a human reviews, not silently on every fresh install.

---

**Situation:** Your CI pipeline's Docker build takes 11 minutes on every single commit, even for one-line documentation changes, because `COPY . .` happens before `pip install -r requirements.txt` in the Dockerfile.

Model answer: Diagnose this as a classic layer-ordering bug: copying application code before installing dependencies busts the pip-install cache layer on every commit, since Docker's cache invalidates top-down from the first changed layer, and application code changes far more often than dependencies do. Fix by reordering to: base image → system dependencies → `COPY requirements.txt` (lockfile only) → `pip install --require-hashes` → `COPY` application code last. This single reordering typically drops the build time for a documentation-only commit from minutes to seconds, since the expensive dependency-install layer now stays cached across code-only changes.

---

**Situation:** A candidate in an interview explains that they'd containerize an LLM inference service using a single-stage Dockerfile that installs `gcc`, `build-essential`, and CUDA `devel` headers, since "the build needs them anyway."

Model answer: Push back on the single-stage design specifically: build-time-only tooling (compilers, headers) has zero runtime value and, if shipped in the final image, bloats it by hundreds of megabytes to gigabytes while widening the CVE-scanning surface for no benefit. Recommend a multi-stage build: a "builder" stage with the CUDA `devel` variant and compilers that produces a resolved virtualenv or compiled wheels, and a "runtime" final stage (CUDA `runtime`, not `devel`) that only copies the already-resolved artifacts across — the compilers and headers never appear in what actually ships.

---

**Situation:** A model behaves differently across two GPU nodes even though both nodes report the same driver version, same CUDA toolkit, and `temperature=0` (greedy decoding) is set on the LLM.

Model answer: Explain this is expected, not a bug to chase indefinitely: non-associative floating-point reduction across parallel GPU threads means even greedy decoding isn't guaranteed bit-for-bit deterministic across different kernel or hardware configurations — `temperature=0` controls the sampling strategy, not the underlying numerical determinism of the compute kernels themselves. If bit-for-bit reproducibility is a genuine hard requirement (e.g., for a compliance audit), that requires pinning the entire stack down to specific kernel implementations and potentially disabling certain parallel reduction optimizations, and even then perfect determinism across different physical GPU models isn't guaranteed — set expectations accordingly rather than treating small output variance as necessarily a pinning failure.

---

**Situation:** Your team promotes a service by rebuilding a fresh Docker image at each of dev, staging, and production, using "the same Dockerfile" each time, and a subtle bug appears only in production that nobody can reproduce in staging.

Model answer: Identify the root cause as the core anti-pattern this module targets: rebuilding at each stage means every rebuild is a fresh opportunity for a lockfile to re-resolve slightly differently or a base image tag to have moved, even from an ostensibly identical Dockerfile. Fix by building the artifact exactly once (referenced immutably by its image digest) and promoting that same digest through dev → staging → production, never rebuilding at any later stage — this guarantees production runs exactly what staging already validated, eliminating the entire class of "identical Dockerfile, different bytes" bugs by construction.

---

**Situation:** An "emergency" production hotfix is requested, and someone proposes deploying directly from a developer's laptop straight to production, skipping the staging gate, "just this once, because it's urgent."

Model answer: Push back hard on this specific pattern, even under urgency — an emergency is exactly when environment drift is most likely to be under-checked, precisely because the normal automated gates (health check, integration test, load test, evaluation gate) are what's being proposed to skip. Recommend the fastest safe path instead: run the full automated staging gate but expedite the human sign-off step (get it reviewed immediately, not skipped), since the gate itself typically runs in minutes and the actual bottleneck is usually process latency, not the checks' runtime — never bypass the checks designed to catch the exact failure mode most likely under time pressure.

---

**Situation:** A teammate asks why the course recommends `uv` as a mid-2026 default over the team's existing, working `pip-tools` + `pyenv` setup, given that both produce hash-pinned lockfiles.

Model answer: Clarify this isn't about reproducibility guarantees — both approaches genuinely provide pin+hash reproducibility when used correctly. The practical advantage of `uv` is install speed: resolving and installing a stack like `torch` + `transformers`, which can take pip several minutes, often completes in well under 30 seconds cold with `uv`, which matters meaningfully when this happens on every CI run and every container build. Recommend migrating for a new project by default, but frame it as a velocity improvement, not a correctness fix — an existing, working `pip-tools` setup with genuine hash-pinning isn't broken and migrating it is a judgment call about migration cost versus the ongoing speed benefit, not an urgent correctness gap.

---

**Situation:** Your team's environment needs a specific compiled CUDA toolkit build and a couple of non-Python native libraries that aren't cleanly available as PyPI wheels for your target platform. Someone proposes using `uv` anyway, "since it's the fastest."

Model answer: Recommend `conda-lock` instead for this specific case, despite `uv`'s speed advantage elsewhere — Conda's channel-based resolution does real work that a pip-compatible resolver like `uv`'s genuinely cannot replace: pinning a specific compiled CUDA toolkit build and non-Python native libraries is exactly the scenario where Conda's cross-language package management matters. Frame the choice as criteria-driven, not "always pick the fastest tool" — the first fork in the lockfile-tool decision tree (does the environment need Conda-specific channel capabilities) is a hard technical constraint, and every fork after that is a judgment call about team investment, but this specific scenario lands squarely on the hard-constraint side.

---

**Situation:** A postmortem reveals that a training run's logged MLflow metrics and hyperparameters are complete, but nobody can determine which PyTorch or CUDA version actually produced the run six months later.

Model answer: Diagnose this as "recording model metrics/hyperparameters but not the environment snapshot" — half the reproducibility story is missing if you can reproduce the hyperparameters but not the runtime that interpreted them. Fix by treating environment metadata as first-class experiment data, logged alongside metrics and parameters on every run: Python version, framework versions, CUDA version, base image digest, and a hash of the lockfile itself (`requirements_lock_hash`), ideally via `mlflow.log_dict()` as a structured artifact — so any future inspection of a run can answer "what environment produced this" without archaeology.

---

**Situation:** A base image was digest-pinned eighteen months ago and has never been updated since; a security audit flags several known CVEs in packages baked into that exact digest.

Model answer: Name this precisely: digest pinning trades "silent drift" for a new responsibility, and without an automated update process, it silently freezes on a stale, potentially vulnerable image — this is exactly the failure mode the module warns pairs with digest pinning if left unaddressed. Recommend wiring Renovate or Dependabot to track the base image and open an automated PR whenever the digest changes, on a defined schedule, so a human reviews and merges the bump deliberately — closing the loop between "pin for determinism" and "don't freeze forever on a CVE" rather than leaving digest pinning as a one-time action.

---

**Situation:** A GPU-heavy PyTorch training container is crashing intermittently with a `Bus error` during multi-worker data loading, and the application code looks correct on inspection.

Model answer: Suspect a container-level cause before an application-level one: Docker's default shared-memory size (`/dev/shm`, 64MB) is frequently too small for PyTorch's DataLoader worker processes and NCCL multi-process communication, and this is a classic, easy-to-miss silent-crash cause in ML containers that has nothing to do with the training logic itself. Fix by explicitly raising `--shm-size` at container run time (e.g., `docker run --shm-size=8g`), or the equivalent `shm_size` field in Compose, rather than continuing to debug the training script for a bug that isn't there.

---

**Situation:** A model registry stores model versions but a related question comes up during a promotion-tooling redesign: should the new `promotion.yaml`-based environment-promotion pipeline be unified with MLflow's model-version stage/alias mechanism, or kept as a separate system?

Model answer: Recommend keeping them complementary but distinct, not unified into one mechanism — the model registry's alias (`@champion`/`@challenger`) answers "what's live" from the model's perspective, while the environment-promotion manifest answers "what's been validated at each stage" from the deployment pipeline's perspective, and conflating them tends to produce a system that can't cleanly express, for example, a model version that's fully validated in staging but not yet promoted to the production alias. Also flag, while reviewing any new promotion tooling, that it should be built around the current alias-based model, not MLflow's deprecated Staging/Production/Archived stage enum, which has been deprecated since MLflow 2.9.

---

**Situation:** A junior engineer wants to add SHA-pinning, hash-verified lockfiles, and a full 5-layer Docker build to a disposable exploratory notebook that only they will ever run, on their own laptop, before any code is shared.

Model answer: Recommend against it for this specific case — full pin+hash+digest discipline is overkill for quick local experimentation before any code is shared, and will slow down iteration with no corresponding benefit while the work stays fully disposable and single-machine. The trigger for adding this rigor is the moment the code needs to be reproduced by someone else — a teammate, CI, or a production host — not before; recommend a lightweight virtualenv for now, with the explicit expectation of upgrading to full lockfile+digest discipline the moment the notebook graduates toward being shared or productionized.

---

**Situation:** An infrastructure team wants every product team at the company to independently choose and maintain its own base images and digest-pinning conventions, arguing it gives teams maximum flexibility.

Model answer: Recommend centralizing base-image ownership instead, using the Uber-scale reasoning from this module's case studies: as a platform grows from one team to many, letting every team independently pin (and independently drift) its own base images multiplies the surface area of "which exact CUDA/driver/OS combination is running where" from a handful of centrally-vetted options to potentially hundreds of ad hoc choices. Recommend a platform/infra team maintaining a small number of qualified, digest-pinned, regularly-patched base images that product teams build `FROM`, trading a small amount of per-team flexibility for a dramatically smaller, more auditable set of environment configurations across the org.
