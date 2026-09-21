# Module 04 — Reproducibility and Environment Management

> "Works on my machine" is not a punchline. It is a deployment incident that has not happened yet.

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Explain, with intuition and mechanism, why ML and LLM systems drift even when "the code hasn't changed" — and classify drift into library, CUDA/driver, Python-interpreter, and system-dependency drift.
2. Articulate the **reproducibility paradox** and apply the "pin everything, hash everything" discipline to Python, Conda, and container environments.
3. Debug a drift-induced production incident methodically: recognize the symptom, diff environments, isolate the mismatch, confirm root cause, and lock the fix in place so it cannot silently recur.
4. Design a production-grade, SHA-pinned, multi-stage Dockerfile for an ML/LLM service using a 5-layer build pattern that maximizes both determinism and cache efficiency.
5. Choose and justify a lockfile strategy (`pip-tools`, Poetry, `uv`, `conda-lock`) for a given team and workload, and explain the mid-2026 shift toward `uv` as a default.
6. Design a dev → staging → production promotion pipeline with automated gates, and explain — and defend in an interview — the principle "promote the artifact forward, never rebuild it."
7. Build a `promotion.yaml`-style manifest that gives you an auditable record of what was promoted, when, and under what evidence.

### Prerequisites

- Comfort with Python packaging basics (`pip`, `requirements.txt`, virtual environments).
- Basic Docker literacy: images, layers, `docker build`, `docker run`.
- Familiarity with the general MLOps lifecycle (data → training → registry → serving) from earlier modules — this module is the "environment" layer that cuts across all of those stages.
- No prior Kubernetes or CI/CD expertise is assumed, though it helps.

### Key Terminology

| Term | Definition |
|---|---|
| **Environment drift** | Any unintended change in the software/hardware stack (libraries, Python interpreter, CUDA/driver, OS packages) that causes identical code and data to behave differently over time or across machines. |
| **Reproducibility** | The property that the same code + same dependencies + same inputs produce the same outputs, every time, on every machine. |
| **Pinning** | Specifying an *exact* version (`numpy==1.26.4`) instead of a range (`numpy>=1.23`), removing ambiguity about what gets installed. |
| **Hashing / hash verification** | Recording and checking a cryptographic digest (e.g., SHA-256) of a package artifact or base image so you verify you got the *exact bytes* you intended, not just "a release with the same version string." |
| **Lockfile** | A machine-generated, fully resolved, hash-pinned manifest of an entire dependency tree (direct + transitive), used to reproduce an identical environment deterministically. |
| **Digest pinning (image)** | Referencing a container base image by its immutable content digest (`@sha256:...`) rather than a mutable tag like `latest` or even `3.11-slim`. |
| **Transitive dependency** | A dependency of a dependency — not declared directly by you, but required by something you did declare, and just as capable of introducing drift. |
| **Environment snapshot** | A structured record (YAML/JSON) capturing exact versions, image tags, and hashes used to produce a given model artifact or run. |
| **Promotion (environment promotion)** | Moving a *specific, already-built and already-validated* artifact through dev → staging → production, gated by increasingly strict checks — as opposed to rebuilding at each stage. |
| **Promotion manifest** | A record (e.g., `promotion.yaml`) of what artifact moved, from which stage to which, which checks passed, and who/what approved it. |
| **Model registry alias** | A movable, human-meaningful pointer (e.g., `@champion`, `@challenger`) to a specific model version, replacing the older rigid "Staging/Production" stage enum in tools like MLflow. |

---

## 2. Why This Topic Matters and Where It Fits in the MLOps/LLMOps Lifecycle

Every MLOps lifecycle diagram you've seen so far in this course has boxes like "Data," "Training," "Registry," "Serving," "Monitoring." Reproducibility and environment management is not another box next to those — it is the **substrate underneath all of them**. It is the reason a model that scored 94% F1 in a notebook on Tuesday can silently degrade to 81% F1 when redeployed on Thursday, with *zero* changes to the model weights, the training code, or the incoming data.

```
                 ┌─────────────────────────────────────────────────────────┐
                 │                  THE ENVIRONMENT LAYER                   │
                 │  (Python interpreter, libraries, CUDA/driver, OS libs,   │
                 │   container base image, hardware — all versioned,       │
                 │   pinned, hashed, and reproducible)                     │
                 └─────────────────────────────────────────────────────────┘
                              ▲        ▲        ▲        ▲        ▲
                              │        │        │        │        │
                       ┌──────┴──┐ ┌───┴───┐ ┌──┴────┐ ┌─┴─────┐ ┌┴────────┐
                       │  Data   │ │Training│ │Registry│ │Serving│ │Monitoring│
                       │ pipeline│ │  jobs  │ │        │ │       │ │          │
                       └─────────┘ └───────┘ └────────┘ └───────┘ └──────────┘
```

Most MLOps curricula treat "data drift" and "concept drift" as the primary threats to a production ML system, and rightly spend a lot of time on statistical monitoring for them (covered elsewhere in this course). But there is a *third* category of drift that is arguably more insidious because it is invisible to data-quality dashboards: **environment drift**. Your feature distributions can look perfectly stable, your input schema can validate cleanly, and your model can still produce different outputs — because `numpy` silently changed its floating-point summation order, or a transformers version changed a default tokenizer behavior, or the CUDA runtime changed how a kernel rounds.

For **LLM systems** specifically, this problem is amplified:

- LLM serving stacks (vLLM, TGI, SGLang, Triton) sit on top of an unusually deep and fast-moving hardware/software chain: CUDA → cuDNN → PyTorch → attention kernels (FlashAttention/FlashInfer) → the serving framework itself. A single link changing can alter numerics, throughput, or even crash the process.
- Quantization schemes (GPTQ, AWQ, GGUF, FP8) are exquisitely sensitive to kernel and library versions — the *same* quantized checkpoint can produce measurably different generations under different backend versions.
- Prompt-level and decoding-level reproducibility (temperature=0 does not guarantee determinism on GPU due to non-associative floating point reductions across parallel threads) means LLM teams must be even more disciplined about pinning than classic ML teams, since they have less "user-agnostic ground truth" to alert them that something drifted.

This is why senior MLOps/LLMOps interviews probe this topic heavily: **anyone can train a model in a notebook; the differentiator for a senior engineer is knowing how to make that model's behavior deterministic and portable across dev, CI, staging, and production, for months or years.** Reproducibility is also a compliance and audit requirement in regulated industries (finance, healthcare, insurance) — "what exact software produced this prediction six months ago" is a question you must be able to answer.

Where this sits in the bigger picture:

```
  ┌────────────┐   ┌───────────┐   ┌────────────┐   ┌──────────┐   ┌──────────────┐
  │ Data        │→ │ Feature/   │→ │ Training    │→ │ Model     │→ │ Serving /     │
  │ Versioning  │   │ Experiment │   │ (this module│   │ Registry  │   │ Promotion     │
  │ (Module 03) │   │ Tracking   │   │ underlies   │   │ (aliases, │   │ (this module) │
  │             │   │            │   │ all of it) │   │ tags)     │   │               │
  └────────────┘   └───────────┘   └────────────┘   └──────────┘   └──────────────┘
                                          ▲
                                          │
                         Module 04: Reproducibility & Environments
                         (pin, hash, snapshot, containerize, promote)
```

---

## 3. Main Concepts

### 3.1 Environment Drift — Theory

**What it is.** Environment drift is any change to the software or hardware stack surrounding your code/data/model that causes behavior to change even though the "logical" artifact (code + data + model weights) did not. It is the fourth silent variable in the equation most people write as:

```
output = f(code, data, model_weights)
```

The honest equation is:

```
output = f(code, data, model_weights, environment)
```

**Why it happens.** No package manager, by default, guarantees you get the same thing twice. `pip install numpy` today and `pip install numpy` in six months can resolve to different releases. Docker's `FROM python:3.11` today and in six months can point to different underlying image digests, because tags are mutable pointers, not fixed content. GPU stacks add another dimension: the CUDA toolkit version baked into a container, the host's NVIDIA driver, and the specific compute capability of the physical GPU all interact, and small mismatches produce anything from silent numerical differences to outright launch failures.

**Categories of drift (from the transcripts, expanded):**

| Drift type | Root cause | Typical symptom | Example |
|---|---|---|---|
| **Library version drift** | A pip/conda package resolves to a different version at install time than at original build time | Numeric differences, changed defaults, deprecation-driven behavior changes | NumPy 1.23 vs 1.24 changes an internal default reduction order; `pandas` changes a `groupby` sort default |
| **CUDA/driver drift** | GPU software stack (CUDA toolkit, cuDNN, NVIDIA driver) differs between training/dev and serving | Slower or faster kernels, precision differences, outright `CUDA error: no kernel image is available` | Model built against CUDA 12.1 kernels served on a host with a CUDA 11.8 driver |
| **Python interpreter drift** | Code runs under a different CPython minor version | Different `dict`/`set` iteration edge cases, changed stdlib behavior, C-extension ABI mismatches | A C-extension wheel built for Python 3.10 silently falls back to a slower pure-Python path on 3.12 |
| **System dependency drift** | OS-level libraries, build toolchains, drivers, and binaries change (often via base image updates) | Segfaults, missing `.so` files, subtly different linear algebra results (different BLAS backend) | `libgomp`/OpenBLAS vs MKL differences change floating-point results in matrix multiplication |

**Problems it solves (when you address it) / problems it causes (when you don't).** Addressing drift head-on gives you: trustworthy A/B tests (you know a metric delta is due to the model, not the runtime), safe rollbacks (an old image behaves exactly as it did when it was last verified), and audit-ability (you can answer "what exactly produced this prediction" months later). Left unaddressed, it causes the classic and expensive failure mode: an incident that *looks* like a model or data problem, burns days of a data science team's time chasing feature pipelines, and turns out to be a bumped transitive dependency.

**Tradeoffs.** Perfect pinning has a cost: it slows down adopting security patches and new features, and if taken to an extreme (pinning every transitive dependency by hand) becomes a maintenance burden. The resolution, covered in section 3.3, is to pin everything *automatically* via lockfiles and automate the *update* path (Renovate/Dependabot bots that open PRs), rather than pinning manually and letting the pins go stale.

**When to invest heavily vs. not.** A one-off exploratory notebook that never leaves your laptop does not need SHA-pinned Docker images. Anything that (a) will be run by more than one person, (b) will run in CI, or (c) will ever reach production needs at least lockfile-level pinning. Anything customer-facing or regulated needs the full stack: pinned+hashed lockfile, SHA-pinned container image, and environment snapshot metadata stored alongside the model artifact.

#### Architecture: Where Drift Enters the Stack

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                        SOURCES OF ENVIRONMENT DRIFT                     │
 ├───────────────┬───────────────┬───────────────┬────────────────────────┤
 │  Hardware      │  OS / System   │  Language      │  Application-level     │
 │  layer         │  layer         │  runtime       │  dependency layer      │
 ├───────────────┼───────────────┼───────────────┼────────────────────────┤
 │ GPU model      │ Base image     │ CPython        │ numpy / torch /        │
 │ NVIDIA driver  │ digest         │ minor version   │ transformers /         │
 │ CUDA toolkit   │ apt packages   │ (3.10 vs 3.11) │ vllm / fastapi / ...   │
 │ compute cap.   │ (libgomp, curl,│ ABI of C ext.  │ + ALL transitive deps  │
 │                │  build-essential)│               │                        │
 └───────────────┴───────────────┴───────────────┴────────────────────────┘
        │                │                │                    │
        └────────────────┴────────┬───────┴────────────────────┘
                                   ▼
                    Same code + same data + same weights
                    → DIFFERENT output/behavior in production
```

#### Debugging Example: Recognizing and Isolating Drift

**Beginner example** — a data scientist notices a notebook produces different output on a colleague's laptop:

```bash
# Step 1: Recognize the symptom
# "Model accuracy in staging dropped from 0.91 to 0.84, no code changes."

# Step 2: Compare environments
pip freeze > dev_env.txt         # run on the known-good machine
pip freeze > staging_env.txt     # run on the machine exhibiting the problem
diff dev_env.txt staging_env.txt
```

**Intermediate example** — the diff surfaces the mismatch:

```diff
< numpy==1.23.5
---
> numpy==1.26.4
```

```python
# Step 4: confirm root cause with a minimal repro
import numpy as np
print(np.__version__)
arr = np.random.RandomState(42).randn(10_000, 512).astype(np.float32)
print(arr.sum())   # compare printed value across the two environments
```

If the sum (or a downstream model metric computed from `arr`) differs meaningfully between the two NumPy versions, you have your root cause candidate. Confirm by pinning `numpy==1.23.5` in the staging environment and re-running the evaluation — if accuracy recovers, drift is confirmed as the cause.

**Production-grade example** — automate this comparison so a human never has to run it manually mid-incident:

```python
# tools/env_diff.py — run in CI on every deploy, diff against the last-known-good snapshot
import subprocess
import json
import sys

def freeze() -> dict[str, str]:
    out = subprocess.check_output(["pip", "list", "--format=json"])
    return {pkg["name"]: pkg["version"] for pkg in json.loads(out)}

def diff_against_baseline(baseline_path: str) -> int:
    with open(baseline_path) as f:
        baseline = json.load(f)
    current = freeze()
    mismatches = {
        name: (baseline.get(name), current.get(name))
        for name in set(baseline) | set(current)
        if baseline.get(name) != current.get(name)
    }
    if mismatches:
        print("ENVIRONMENT DRIFT DETECTED:", file=sys.stderr)
        for name, (old, new) in sorted(mismatches.items()):
            print(f"  {name}: baseline={old} current={new}", file=sys.stderr)
        return 1
    print("Environment matches baseline. No drift detected.")
    return 0

if __name__ == "__main__":
    sys.exit(diff_against_baseline("environment-baseline.json"))
```

Wiring this into a CI job means drift is caught *before* the image is promoted, not after users notice accuracy problems — turning the reactive "recognize symptom → compare → isolate → root-cause" loop from the transcript into a proactive gate (this connects directly to section 3.5, Promotion Gates).

---

### 3.2 The Reproducibility Paradox and the Pin-and-Hash Philosophy

**Theory.** The paradox: an engineer can honestly, truthfully say "it works" — the notebook runs top to bottom, the tests pass, the outputs look right — and still be wrong that the system is reproducible, because "works" was only ever demonstrated on *one* machine at *one* point in time. Reproducibility is not a property of a single successful run; it is a property of a *process* that produces the same result across machines, across time, and across team members. This is why "works on my machine" is treated in this course as a **deployment incident that has not happened yet**, not a joke.

The practical resolution to the paradox has two parts, and it's important to understand *why* you need both, not just one:

1. **Pin everything.** A version pin (`torch==2.4.1`) tells the installer *which release* you want. This removes range-based ambiguity (`torch>=2.0` could resolve to wildly different releases six months apart).
2. **Hash everything.** A version pin alone assumes the package registry never re-uploads or corrupts an artifact under the same version string, and it does nothing to protect you against a compromised mirror or an accidentally-published broken build. A hash (SHA-256 digest of the actual file) verifies you received the *exact bytes* you expect — this is a lockfile-level guarantee, and it doubles as a supply-chain security control.

**When pinning alone is insufficient:** version pinning protects against *drift over time* on the same registry, but not against *tampering* or *artifact substitution*. Hash verification is the layer that closes that gap. This is precisely why `pip-compile --generate-hashes`, Poetry's lockfile, `uv.lock`, and `conda-lock` all embed hashes, not just versions — treating pinning as sufficient without hashing is a common gap that interviewers probe for.

**Tradeoffs and when NOT to over-apply this.** For quick local experimentation before any code is shared, full hash-pinning is overkill and will slow you down with no benefit — use a loose `requirements-dev.txt` or an ad hoc virtualenv. The moment code needs to be reproduced by someone else (a teammate, CI, a production host) is the moment you need pin+hash discipline.

#### Architecture: The Two Layers of the Pin-and-Hash Guarantee

```
                     ┌───────────────────────────────────────┐
                     │            YOUR DECLARATION            │
                     │      (pyproject.toml / requirements.in)│
                     │        "I want torch ~= 2.4"           │
                     └────────────────┬────────────────────────┘
                                      │ resolve (once, deliberately)
                                      ▼
                     ┌───────────────────────────────────────┐
                     │              LOCKFILE                  │
                     │  (requirements.txt --generate-hashes / │
                     │   poetry.lock / uv.lock / conda-lock.yml)│
                     │                                         │
                     │  torch==2.4.1 \                        │
                     │    --hash=sha256:9f2c1e...              │
                     │  (+ every transitive dependency,        │
                     │   version + hash)                       │
                     └────────────────┬────────────────────────┘
                                      │ install (many times, deterministically)
                                      ▼
                     ┌───────────────────────────────────────┐
                     │         IDENTICAL ENVIRONMENT          │
                     │   dev laptop == CI runner == staging   │
                     │           == production                │
                     └───────────────────────────────────────┘
```

#### Code: Pin + Hash Across Three Tool Ecosystems

**pip-tools (plain pip ecosystem)**

```bash
# requirements.in — human-edited, loose ranges are fine here
echo "torch~=2.4" > requirements.in
echo "transformers~=4.44" >> requirements.in
echo "fastapi~=0.115" >> requirements.in

# Compile a fully pinned, hash-verified lockfile (machine-generated, never hand-edited)
pip-compile --generate-hashes --output-file=requirements.txt requirements.in

# Install with hash enforcement — pip refuses to install anything
# whose downloaded artifact doesn't match the recorded hash
pip install --require-hashes -r requirements.txt
```

**Poetry**

```toml
# pyproject.toml
[tool.poetry.dependencies]
python = "^3.11"
torch = "2.4.1"
transformers = "4.44.2"
```

```bash
poetry lock          # generates poetry.lock with resolved versions + hashes
poetry install --sync  # installs exactly what's in the lock, removes anything extra
```

**uv (Astral) — the mid-2026 fast default**

```bash
uv init llm-service && cd llm-service
uv add torch==2.4.1 transformers==4.44.2 fastapi==0.115.0
# uv.lock is written automatically, fully pinned + hashed
uv sync --frozen   # installs exactly what's locked; fails if pyproject and lock disagree
```

> **Why `uv` matters in 2026.** As noted in this module's `references.md`, `uv` (a Rust-based tool from Astral) has become a fast-growing default for Python dependency locking in ML repositories by mid-2026, frequently replacing the combination of `pip-tools` + `Poetry` + `pyenv` with one tool and one native `uv.lock` file. Its main practical advantage for ML/LLM work specifically is install speed: resolving and installing a stack like `torch` + `transformers`, which can take pip several minutes, reportedly completes in well under 30 seconds cold with `uv` — a meaningful difference when this happens on every CI run and every container build. It does not change the underlying reproducibility guarantee (that still comes from the lockfile + hashes); it changes how painful it is to maintain that guarantee day to day.

**conda-lock (for Conda-based / mixed-language environments)**

```yaml
# environment.yml — the input, still loose
name: llm-training
channels:
  - conda-forge
dependencies:
  - python=3.11
  - pytorch=2.4
  - cudatoolkit=12.1
  - pip
  - pip:
    - transformers==4.44.2
```

```bash
conda-lock lock --file environment.yml --platform linux-64 --platform osx-arm64
# produces conda-lock.yml: fully pinned + hashed, per-platform
conda-lock install --name llm-training conda-lock.yml
```

`conda env export` alone (even `--from-history`) is **not** a substitute for `conda-lock`: as this module's research notes point out, plain export can still re-resolve differently on a different platform or at a different point in time, because it does not freeze the full transitive dependency graph with hashes the way `conda-lock` does. Treat `conda env export --from-history` as a *human-readable declaration of intent* and `conda-lock.yml` as the *actual reproducibility guarantee*.

#### Comparison Table: Lockfile Tools

| Tool | Ecosystem | Hash-pinning | Cross-platform lock | Speed (cold install) | Best fit |
|---|---|---|---|---|---|
| `pip-tools` | pure pip | Yes (`--generate-hashes`) | No (single-platform per lock) | Moderate | Teams already on plain pip, want minimal tooling change |
| Poetry | pip-compatible + own resolver | Yes (`poetry.lock`) | Partial (locks per-arch markers) | Moderate–slow on heavy ML deps | Library/package authors, teams wanting dependency-graph tooling + publishing |
| **uv** | pip-compatible, Rust resolver | Yes (`uv.lock`) | Yes | **Fast** (order-of-magnitude vs pip) | Default recommendation for new ML/LLM repos in 2026 |
| conda-lock | Conda/mamba | Yes (`conda-lock.yml`) | Yes (per listed platform) | Moderate | Mixed Python/native/CUDA stacks where Conda channels matter (e.g., specific CUDA toolkit builds) |

---

### 3.3 Reproducible Containers — Docker, SHA-Pinned Base Images, and the 5-Layer Build

**Theory.** A lockfile solves reproducibility for *Python packages*. It does not solve reproducibility for the *rest of the environment*: the OS, system libraries, the CUDA toolkit, the Python interpreter build itself. Containers close that gap by packaging the whole stack as one artifact. But — and this is the crux of the lesson from the transcripts — **a container is only as reproducible as the base image reference you build it from.** `FROM python:3.11-slim` looks pinned (it names a specific minor version!) but the tag `3.11-slim` is a *mutable pointer*: the maintainers republish it under the same tag whenever they patch the underlying OS packages. Two builds of the identical Dockerfile, weeks apart, can pull different bytes.

The fix is **digest pinning**: referencing the image by its immutable SHA-256 content digest.

```dockerfile
# Fragile: "3.11-slim" is a moving target
FROM python:3.11-slim

# Reproducible: this digest can only ever resolve to one exact set of bytes
FROM python:3.11-slim@sha256:2ec5a8dcda681ee...  # illustrative digest
```

This matters even more for GPU base images. As of mid-2026, the `nvidia/cuda:latest` tag is deprecated and unavailable on both Docker Hub and NGC — pulling it returns a `manifest unknown` error. This is not merely a style recommendation anymore; **explicit version + digest pinning of CUDA base images is now a hard operational requirement**, not a nice-to-have, because the convenient-but-dangerous floating tag has been removed as an option entirely for `latest`, and even version-only tags (`12.4-runtime-ubuntu22.04`) are still mutable pointers subject to patch republishing.

**Problems this solves.** Digest pinning gives you byte-for-byte identical starting points across time and machines. It also, as a side effect, locks in whatever CVE/security patch state existed at pin time — which is why digest pinning must be paired with an *automated update process* (see below), or you trade "silent drift" for "silently frozen on an old vulnerable image," which is its own production risk.

**Tradeoffs.** Digest-pinned images are harder to read (a 71-character hex string tells a human nothing about what's inside) and, without automation, go stale. The standard mitigation: use Renovate or Dependabot to open automated PRs whenever a tracked image tag's digest changes, so a human reviews and merges a bump rather than the build silently drifting *or* silently freezing forever.

#### The 5-Layer Docker Build Pattern

The core insight from the source material is that reproducibility and build performance are not in tension — a well-ordered Dockerfile gets you both, because Docker's layer cache is keyed on the *hash of layer inputs*, so ordering from "least frequently changing" to "most frequently changing" means only the layers that actually changed get rebuilt.

```
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 5:  COPY application code, ENTRYPOINT / CMD                    │
│           → changes on every commit                                   │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 4:  pip install --require-hashes -r requirements.txt            │
│           → changes only when dependencies change                     │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 3:  COPY requirements.txt (lockfile only, not source code)       │
│           → changes only when dependencies change                     │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 2:  apt-get install <system deps: libgomp1, curl, build tools>   │
│           → changes rarely                                            │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 1:  FROM <base image>@sha256:<digest>                           │
│           → fixed, immutable, identical every single build             │
└─────────────────────────────────────────────────────────────────────┘
       ▲
       │  Docker cache invalidates top-down from the FIRST layer that changed;
       │  everything below an unchanged layer is reused from cache.
```

The classic bug this pattern prevents: "NumPy is 1.23 locally but 1.24 in production, even though the Dockerfile looks identical." This happens when `requirements.txt` is unpinned (or copied *after* application code so Docker "helpfully" reuses a stale cached install layer that predates a dependency bump) — the fix is both digest-pinning the base image *and* hash-pinning the lockfile, copied in its own layer, before application code.

#### Production-Grade Dockerfile — Multi-Stage, SHA-Pinned, Hash-Verified

This example builds an LLM inference microservice (FastAPI + `transformers` + `torch`), using a multi-stage build so the final image ships without build tools, compilers, or pip's package cache — smaller attack surface, smaller image, faster cold starts.

```dockerfile
# syntax=docker/dockerfile:1

########################################
# Stage 1 — "builder": compiles wheels, installs pinned+hashed deps
########################################
# Layer 1: SHA-pinned base image (illustrative digest — always resolve your own
# current digest with `docker pull` + `docker inspect --format='{{index .RepoDigests 0}}'`)
FROM python:3.11-slim-bookworm@sha256:2ec5a8dcda681ee8a0d67a680bb64a70c2ceb90fb03917fa1cb8f3b2d5a0f0f7 AS builder

# Layer 2: system dependencies needed only to BUILD wheels (gcc, headers)
# Pinned apt versions where the distro provides them; apt itself is pinned
# transitively by the base image digest above.
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential=12.9 \
        curl=7.88.1-10+deb12u8 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Layer 3: copy ONLY the lockfile first — this is the cache-efficiency trick.
# Application code changes on every commit; the lockfile changes rarely.
# As long as requirements.txt is unchanged, Docker reuses the pip install layer
# below even if every other file in the repo has changed.
COPY requirements.txt .

# Layer 4: hash-verified, fully pinned install.
# --require-hashes makes pip REFUSE to install anything whose downloaded
# artifact doesn't match the recorded SHA-256 in the lockfile.
RUN pip install --no-cache-dir --require-hashes -r requirements.txt

########################################
# Stage 2 — "runtime": minimal final image, no compilers, no pip cache
########################################
FROM python:3.11-slim-bookworm@sha256:2ec5a8dcda681ee8a0d67a680bb64a70c2ceb90fb03917fa1cb8f3b2d5a0f0f7 AS runtime

# Only the runtime system libs actually needed (no build-essential in final image)
RUN apt-get update && apt-get install -y --no-install-recommends \
        libgomp1=12.2.0-14 \
    && rm -rf /var/lib/apt/lists/*

# Copy the already-resolved site-packages from the builder stage —
# no re-resolution happens here, so no chance of drift between stages.
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

WORKDIR /app

# Layer 5: application code — the ONLY layer expected to change on every commit
COPY ./app ./app
COPY ./config ./config

# Run as non-root for defense in depth
RUN useradd --create-home --shell /bin/bash appuser
USER appuser

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    HF_HOME=/app/.cache/huggingface

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
    CMD curl -f http://localhost:8000/healthz || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**GPU variant note:** for a GPU-serving image, stage 1's `FROM` becomes an NVIDIA CUDA base such as `nvidia/cuda:12.4.1-runtime-ubuntu22.04@sha256:<digest>` (never `:latest` — see above), and you additionally pin the PyTorch build to match: `torch==2.4.1+cu124` sourced from the PyTorch CUDA wheel index, recorded with its hash in the lockfile exactly like any other dependency.

#### Keeping SHA-Pinned Digests From Going Stale

Digest pinning trades "silent drift" for a new responsibility: someone has to *deliberately* bump the digest to pick up patches. The standard production pattern:

```yaml
# renovate.json (excerpt) — automatically opens a PR whenever a tracked
# base image's digest changes, so a human reviews and merges the bump
# instead of the build silently drifting OR silently freezing on a stale, vulnerable image
{
  "packageRules": [
    {
      "matchDatasources": ["docker"],
      "matchPackageNames": ["python", "nvidia/cuda"],
      "pinDigests": true,
      "schedule": ["before 6am on monday"]
    }
  ]
}
```

This closes the loop: pin for determinism, automate the update path so pinning doesn't become "frozen forever on a CVE."

---

### 3.4 Environment Metadata — Snapshots as First-Class Experiment Data

**Theory.** Pinning and hashing make an environment reproducible *if you know what to reproduce*. The missing piece is *recording* that environment alongside every model artifact and every run — otherwise reproducibility is only as good as someone's memory of "I think I used torch 2.3 back then." The principle from the transcript: **treat environment metadata as first-class experiment data, exactly like metrics, hyperparameters, and dataset versions** — not an afterthought bolted on post-hoc.

```yaml
# environment-snapshot.yml — stored alongside the model artifact / run ID
run_id: "run-2026-07-14-8841"
python_version: "3.11.9"
pytorch_version: "2.4.1+cu124"
transformers_version: "4.44.2"
cuda_version: "12.4"
base_image: "myregistry/llm-service@sha256:9d1a3f...b02c"
requirements_lock_hash: "sha256:7c4e21...af90"   # hash of the requirements.txt itself
gpu: "NVIDIA A100-SXM4-80GB"
created_at: "2026-07-14T09:12:03Z"
git_commit: "a1b2c3d"
```

Storing `requirements_lock_hash` (a hash *of the lockfile*, distinct from the per-package hashes inside it) gives you a single value you can compare across two runs to know instantly, without diffing line by line, whether their entire dependency universe was identical.

**Where this metadata should live in practice:** alongside model metrics and parameters in your experiment tracker (e.g., an MLflow run's tags/params), and/or as an artifact attached to the model version in your registry — so anyone inspecting a deployed model version can answer "what environment produced this" without archaeology.

```python
import mlflow

with mlflow.start_run(run_name="llm-finetune-run-8841") as run:
    mlflow.log_params({"learning_rate": 2e-5, "epochs": 3})
    mlflow.log_metrics({"eval_loss": 0.412})

    # Environment metadata logged as first-class run data, not a side note
    mlflow.log_dict(
        {
            "python_version": "3.11.9",
            "pytorch_version": "2.4.1+cu124",
            "transformers_version": "4.44.2",
            "cuda_version": "12.4",
            "base_image_digest": "sha256:9d1a3f...b02c",
        },
        artifact_file="environment_snapshot.json",
    )
    mlflow.log_artifact("requirements.txt")  # the actual hash-pinned lockfile
```

---

### 3.5 Environment Promotion — Dev → Staging → Production

**Theory.** Once you can reliably *build* a reproducible artifact, the next discipline is how you *move* it through environments without reintroducing the very drift you just eliminated. The core anti-pattern this section targets: rebuilding the artifact fresh at each stage ("dev build," then a separate "staging build," then a separate "prod build"). Every rebuild is a fresh opportunity for a lockfile to re-resolve slightly differently, a base image tag to have moved, or a transitive dependency to have quietly bumped — precisely the failure mode sections 3.1–3.3 exist to prevent. **The fix is to build once, and promote the same immutable artifact forward.**

```
   DEV                      STAGING                    PRODUCTION
┌─────────┐   build once  ┌──────────┐   promote     ┌───────────┐
│ build &  │──────────────▶│ same      │──────────────▶│ same       │
│ test     │   (image X)   │ image X   │  (image X)    │ image X    │
└─────────┘               └──────────┘               └───────────┘
     │                          │                           │
     ▼                          ▼                           ▼
 unit tests +              eval gate + smoke            load test +
 data checks               test + integration           sign-off/approval
```

**Required checks per stage (from the transcript, formalized):**

| Transition | Required checks |
|---|---|
| dev → staging | All unit tests pass; data/schema validation checks pass |
| staging → production | Evaluation gate (quality metric above threshold on held-out/canary data); load test (throughput + latency SLOs); human sign-off/approval |

**Automated staging gate (concrete sequence from the transcript):**

1. Check for broken dependencies / schema mismatches.
2. Build the Docker image from a SHA-pinned base.
3. Run the `/healthz` health-check endpoint.
4. Run an integration test against ~20 real production-like samples, verifying output schema matches expectations.
5. Run a short load test (e.g., 100 requests/second for 60 seconds), asserting P95 latency stays below a threshold (e.g., 2000 ms).
6. If all checks pass: push the *same* image to the production registry and begin a rolling deployment. If any check fails: **stop and do not proceed** — do not attempt to "push through" with the same image on a re-run without investigation.

**Key principles (expanded):**

1. **Never skip stages.** A "quick fix" that goes dev → prod directly bypasses every safety net this whole module builds — an "emergency" is exactly when environment drift is most likely to be under-checked.
2. **Automated gates over manual checking.** Humans should approve based on evidence a machine already gathered, not manually re-run each check by hand under time pressure.
3. **Stop on failure.** A failed gate halts the pipeline; it does not soft-fail into production with a warning.
4. **Log everything.** Every promotion decision — inputs, check results, approver — is recorded, not just the final outcome.
5. **Promote the artifact forward; never rebuild it.** This is the single most important principle in this section, worth restating precisely: once an artifact (a container image, referenced by its digest) has passed staging's checks, that *exact same image* — not a new build from the same Dockerfile — is what gets deployed to production. Production must run *exactly what was already validated*, not a nominally-identical rebuild.

#### The Promotion Manifest

```yaml
# promotion.yaml
artifact:
  name: "llm-support-classifier"
  version: "1.4.2"
  image_digest: "sha256:9d1a3f0c...b02cff"
promotion:
  from_stage: "staging"
  to_stage: "production"
  timestamp: "2026-07-29T18:42:11Z"
checks:
  evaluation_score:
    metric: "f1_macro"
    value: 0.913
    threshold: 0.90
    passed: true
  smoke_test:
    samples_tested: 20
    schema_valid: true
    passed: true
  load_test:
    rps: 100
    duration_seconds: 60
    p95_latency_ms: 1420
    threshold_ms: 2000
    passed: true
  approval:
    approved_by: "a.ranjan@castsoftware.com"
    approved_at: "2026-07-29T18:45:00Z"
    passed: true
result: "PROMOTED"
```

This manifest is the audit trail: given any production incident, you can answer "what exactly was promoted, when, and on what evidence" without reconstructing it from Slack messages and memory.

#### A Full Promotion Pipeline (GitHub Actions)

```yaml
# .github/workflows/promote.yml
name: Environment Promotion Pipeline

on:
  push:
    branches: [main]

jobs:
  build-once:
    runs-on: ubuntu-latest
    outputs:
      image_digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - name: Build image (built exactly once, promoted thereafter)
        id: build
        run: |
          docker build -t registry.internal/llm-service:${{ github.sha }} .
          docker push registry.internal/llm-service:${{ github.sha }}
          DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' registry.internal/llm-service:${{ github.sha }})
          echo "digest=$DIGEST" >> "$GITHUB_OUTPUT"

  promote-to-staging:
    needs: build-once
    runs-on: ubuntu-latest
    steps:
      - name: Unit tests + data checks
        run: pytest tests/unit tests/data_checks -q
      - name: Deploy image (by digest, not tag) to staging
        run: ./deploy.sh staging "${{ needs.build-once.outputs.image_digest }}"

  automated-staging-gate:
    needs: promote-to-staging
    runs-on: ubuntu-latest
    steps:
      - name: Health check
        run: curl -f https://staging.internal/healthz
      - name: Integration test (20 production-like samples, schema check)
        run: pytest tests/integration -q
      - name: Load test (100 rps / 60s, assert P95 < 2000ms)
        run: |
          locust -f tests/load/locustfile.py --headless \
                 -u 100 -r 100 --run-time 60s --host https://staging.internal \
                 --check-fail-ratio 0.01
      - name: Evaluation gate
        run: python tools/eval_gate.py --min-f1 0.90

  promote-to-production:
    needs: automated-staging-gate
    runs-on: ubuntu-latest
    environment:
      name: production
      # environment protection rule in GitHub requires manual approval here —
      # this is the human "sign-off" step from the transcript, evidence-backed
      # by the artifacts from the prior jobs, not a blind click
    steps:
      - name: Promote the SAME image digest — do not rebuild
        run: ./deploy.sh production "${{ needs.build-once.outputs.image_digest }}"
      - name: Write promotion manifest
        run: python tools/write_promotion_manifest.py --digest "${{ needs.build-once.outputs.image_digest }}"
```

Note the structural detail that encodes "promote the artifact forward, don't rebuild": the image is built exactly once, in `build-once`, and every subsequent job references `needs.build-once.outputs.image_digest` — there is no second `docker build` anywhere in the pipeline.

#### Model Registry Aliases Instead of Rigid Stages

A related, common mistake is conflating *environment promotion* (this section) with *model registry stage labels*. As of mid-2026, MLflow's old Staging/Production/Archived **stages** have been deprecated since MLflow 2.9 in favor of flexible **model version aliases** (e.g., `@champion`, `@challenger`) plus tags — and are slated for removal in a future major release. Don't build new `promotion.yaml` tooling around the old stage enum; model registries have moved to aliasing:

```python
from mlflow import MlflowClient

client = MlflowClient()

# Modern (2026) pattern: aliases, not stages
client.set_registered_model_alias(
    name="llm-support-classifier", alias="champion", version="14"
)
client.set_registered_model_alias(
    name="llm-support-classifier", alias="challenger", version="15"
)

# Serving code resolves by alias, not by a hardcoded version number:
model_uri = "models:/llm-support-classifier@champion"
```

This keeps the *model registry's* notion of "what's live" (an alias) cleanly separate from the *environment promotion pipeline's* notion of "what's been validated at each stage" (the `promotion.yaml` manifest / image digest) — they're complementary, not the same mechanism.

---

## 4. Real-World Case Studies (Reasoned Inference)

> These are informed architectural inferences based on publicly known engineering-blog patterns and industry norms — not confirmed internal implementation details.

**A frontier LLM lab (a system like OpenAI's or Anthropic's serving stack)** would very plausibly pin every layer of its inference stack far more aggressively than a typical enterprise ML team, because a frontier model's inference stack (custom CUDA kernels, custom attention implementations, quantization) is *itself* part of the product's competitive differentiation and safety profile — an unnoticed kernel-level numerical change could shift model behavior in ways that matter for both quality and safety evaluations. It is reasonable to infer such organizations maintain internally-built, digest-pinned base images for every accelerator generation they support, with dedicated infrastructure teams whose sole job is qualifying a new CUDA/driver/kernel combination against a large evaluation suite before it's allowed anywhere near production traffic — essentially the "automated staging gate" pattern in this module, but with model-quality evals substituted for/added to load tests.

**Google**, drawing on its published MLOps guidance (see this module's `references.md` for the official whitepaper and architecture doc), would plausibly implement environment promotion through its CI/CD-for-ML reference architecture — pipelines defined as code, with a build stage producing a single versioned artifact (a container image referenced by digest in a registry like Artifact Registry) promoted through progressively stricter environments, gated by automated evaluation components very similar to the "evaluation gate" pattern above.

**Netflix**, given its well-documented internal platform investment (e.g., Metaflow), would plausibly bake environment specification directly into its ML workflow definitions — a workflow declares its dependencies (often via Conda environments or container images) as part of the workflow definition itself, so that re-running a workflow from months ago pulls the exact environment it originally ran in, rather than "whatever the cluster happens to have installed today."

**Uber**, with its scale of real-time ML serving (fraud, ETA prediction, marketplace models) would plausibly enforce SHA/digest-pinned base images as a hard platform-level requirement rather than a per-team convention, likely with a central platform team owning a small set of qualified, patched base images that product teams build on top of — reducing the surface area of "which exact CUDA/driver combination is running where" from hundreds of ad hoc choices to a handful of centrally-vetted options.

**Databricks**, given its MLflow ownership, is the most directly verifiable case here: MLflow's own documented shift from stage-based promotion to alias-based promotion (see section 3.5) is a public, confirmed change in the tool itself — a strong signal that the industry consensus (which Databricks helped shape) has moved away from rigid stage enums toward flexible, tag/alias-driven promotion metadata, exactly the shift this module recommends building new pipelines around.

**NVIDIA**, as the maintainer of the CUDA base images referenced throughout this module, is the direct source of the `nvidia/cuda:latest` deprecation described in section 3.3 — any team building GPU-serving containers should treat NVIDIA's own container registry documentation as the canonical source for current, non-deprecated tag and digest conventions rather than relying on older tutorials.

---

## 5. Common Mistakes

1. **Pinning versions but not hashes.** `numpy==1.26.4` in a plain `requirements.txt` still trusts that PyPI (or a mirror) serves the exact same bytes for that version string forever. Always generate hashes (`pip-compile --generate-hashes`, or use a tool that does this natively like `uv`/Poetry/`conda-lock`).
2. **Treating `FROM python:3.11-slim` as pinned.** It names a minor version, but the tag is a mutable pointer republished on patches. Pin the digest.
3. **Using `nvidia/cuda:latest` (or any `:latest` tag) for GPU base images.** As of mid-2026 `latest` isn't even resolvable for `nvidia/cuda` — but the deeper mistake is relying on *any* floating tag, version-only tags included.
4. **`conda env export` (full, non-`--from-history`) mistaken for a lockfile.** It captures a point-in-time resolution on *your* platform; it is not guaranteed to reproduce identically elsewhere or later. Use `conda-lock` for the actual guarantee.
5. **Copying application source code before the lockfile in a Dockerfile.** This busts the dependency-install cache layer on every commit, even when dependencies haven't changed — slow builds *and*, if done carelessly with unpinned deps, an opportunity for a slightly different resolve each time.
6. **Rebuilding a "fresh" image at each promotion stage instead of promoting the same digest.** This is the single most common violation of "promote forward, don't rebuild," and it silently reintroduces every drift risk the rest of the pipeline was built to eliminate.
7. **Manually re-running promotion checks under incident pressure instead of trusting automated gates — or worse, skipping the gate "just this once."** The "emergency dev-to-prod" path is exactly when drift is most likely and least checked.
8. **Recording model metrics/hyperparameters in the experiment tracker but not the environment snapshot.** Half the reproducibility story is missing if you can reproduce the hyperparameters but not the runtime that interpreted them.
9. **Pinning digests once and never updating them.** Without an automated bump process (Renovate/Dependabot), digest pinning trades "silent drift" for "silently frozen on an old, potentially vulnerable image" — neither is safe.
10. **Building new promotion tooling around MLflow's deprecated Staging/Production stage enum** instead of the alias-based model (`@champion`/`@challenger`) that has been the recommended pattern since MLflow 2.9.
11. **Assuming GPU determinism from `temperature=0` alone.** Non-associative floating-point reduction across parallel GPU threads means even greedy decoding isn't bit-for-bit deterministic across different kernel/driver versions — pin the whole stack, not just the sampling parameters.

---

## 6. Best Practices and Production Tips

**When to use full pin+hash+digest+promotion discipline:** any code that will run in CI, will be touched by more than one person, or has any path to production. **When it's overkill:** disposable local exploration that never leaves a single laptop and is never depended on by anyone else — use a lightweight virtualenv there and upgrade the rigor the moment the code graduates.

**Alternatives and how to choose between them:** see the decision tree in `architecture.md` for a full walkthrough of `pip-tools` vs Poetry vs `uv` vs `conda-lock`, and container digest-pinning vs. tag-pinning.

**Cost:** the main cost of this discipline is velocity — pinning and gating slow down how fast a new dependency version or a "quick" prod fix can land. This is a deliberate trade for reliability, and it's mitigated (not eliminated) by automating the update path (Renovate/Dependabot) so the *review* burden, not the *rebuild* burden, is what remains on humans.

**Scaling:** as a platform grows from one team to many, centralize base-image and lockfile ownership (a platform/infra team maintains a small number of qualified, digest-pinned base images; product teams build `FROM` them) rather than letting every team independently pin (and independently drift) its own base images.

**Monitoring:** wire an environment-diff check (like the `env_diff.py` example in section 3.1) into CI as a standing gate, not just an incident-response tool — catching drift before promotion is strictly cheaper than debugging it in production.

**Security:** hash verification and digest pinning are as much supply-chain security controls as they are reproducibility controls — treat CVE scanning of pinned base images (e.g., Trivy/Grype in CI) as part of the same pipeline that enforces pinning, not a separate concern.

**Performance:** the 5-layer Docker build pattern is a rare case where the "correct" reproducibility practice (separating stable from volatile layers) is *also* the fastest practice (maximal cache reuse) — there's no tradeoff to negotiate there, only a pattern to apply correctly.

**Never skip stages, even under incident pressure** — an "emergency hotfix" that bypasses staging's automated gate is precisely the scenario in which environment-drift-induced incidents are most likely to recur, because it's the one path that skips the checks designed to catch them.

---

## 7. Interview Questions

**Q1: Why can "the code hasn't changed" still mean the model's production behavior changed?**
Because output is a function of code, data, model weights, *and environment*. Library version drift, CUDA/driver drift, Python interpreter drift, or system dependency drift can all change numerical behavior or code paths without touching a single line of the application code. A senior answer names the specific drift categories and gives a concrete mechanism (e.g., a NumPy version bump changing floating-point reduction order).

**Q2: What's the difference between pinning a version and hashing a package, and why do you need both?**
Pinning (`==1.26.4`) fixes *which release* you want, removing range-based ambiguity. Hashing verifies the *actual bytes* of the artifact you download match what you expect, protecting against a compromised mirror, a corrupted upload, or any other case where the same version string doesn't guarantee identical content. Lockfiles (pip-tools, Poetry, uv, conda-lock) provide both simultaneously.

**Q3: Why is `FROM python:3.11-slim` not actually a "pinned" base image, and what's the fix?**
The tag names a minor version but is a mutable pointer — the maintainers republish images under that same tag when they patch the underlying OS. Two builds of an identical Dockerfile, weeks apart, can pull different bytes. The fix is to pin by the immutable content digest: `FROM python:3.11-slim@sha256:<digest>`.

**Q4: Walk me through the 5-layer Docker build pattern and explain why it improves both reproducibility and build speed simultaneously.**
Layer 1: SHA-pinned base image (fixed). Layer 2: system dependencies (rarely change). Layer 3: copy the lockfile only (not source code). Layer 4: hash-verified `pip install --require-hashes`. Layer 5: application source code (changes every commit). Because Docker's cache invalidates top-down from the first changed layer, ordering from least-to-most volatile means only genuinely changed layers rebuild — the same ordering that guarantees "dependencies only reinstall when dependencies actually change" is what makes the build fast.

**Q5: Explain the principle "promote the artifact forward, don't rebuild it." Why does rebuilding at each stage undermine reproducibility?**
Every rebuild is a fresh dependency resolution and a fresh base-image pull — each one is an opportunity for drift to creep back in, even from an ostensibly-identical Dockerfile, if any referenced tag has moved. The fix is to build the artifact exactly once (referenced immutably by its image digest), run it through progressively stricter checks at each stage, and deploy that *same* digest to production — guaranteeing production runs exactly what staging already validated, not a nominally-identical twin.

**Q6: How would you debug a production accuracy drop that you suspect is caused by environment drift rather than a model or data problem, and how would you prevent recurrence?**
Recognize the symptom (metric drop, no code/data change); compare environments (`pip freeze` or `pip list --format=json` diffed between the healthy and unhealthy environment, ideally automated in CI rather than manual); isolate the specific mismatched package(s); confirm root cause by pinning the suspect package back to the old version and re-testing; then prevent recurrence by locking the environment (hash-pinned lockfile + digest-pinned base image) and adding an automated environment-diff gate to CI so a future mismatch is caught before promotion, not after a metric regresses in production.

**Q7: Why has MLflow deprecated Staging/Production stages in favor of aliases, and what should you build new promotion tooling around?**
The rigid stage enum (Staging/Production/Archived) forced every model version into one of a small fixed set of global states, which doesn't map well onto real workflows needing concepts like champion/challenger, multiple simultaneous production consumers pinned to different versions, or shadow deployments. Aliases (`@champion`, `@challenger`) are flexible, arbitrary tags a team defines for its own workflow. New promotion pipelines (mid-2026 onward) should resolve models by alias, and track stage-transition evidence (checks, approvals) in a separate promotion manifest, not by relying on the deprecated stage field.

**Q8: When would you choose `uv` over Poetry or `conda-lock`, and when would `conda-lock` still be the right choice despite `uv`'s speed advantage?**
`uv` is the strong default for pure-Python (or Python + prebuilt-wheel, e.g., PyPI CUDA wheels) projects in 2026 because of its dramatically faster resolve/install times and native lockfile, especially valuable when torch/transformers-scale installs happen repeatedly in CI or container builds. `conda-lock` remains the better choice when the environment genuinely needs Conda's cross-language package management — e.g., pinning a specific CUDA toolkit build, non-Python native libraries, or packages that aren't cleanly available as PyPI wheels for your target platform — situations where Conda's channel-based resolution is doing real work `uv`'s pip-compatible resolver can't replace.

---

## 8. Summary, Key Takeaways, and Production Checklist

**Summary.** ML and LLM systems drift not only because data changes but because the software and hardware stack underneath them changes — silently, and often invisibly to standard model/data monitoring. Reproducibility is achieved through discipline, not luck: pin every dependency to an exact version, hash-verify every artifact, pin container base images by immutable digest (not floating tags), structure Docker builds to separate stable from volatile layers, record environment metadata as first-class experiment data, and promote a single immutable artifact through dev → staging → production behind automated, evidence-based gates rather than rebuilding at each stage.

**Key Takeaways:**

- Environment is a fourth, often-overlooked input to ML/LLM system behavior, alongside code, data, and weights.
- "Works on my machine" is a warning sign, not reassurance — reproducibility is a property of a repeatable *process*, not a single successful run.
- Pin everything (exact versions) *and* hash everything (artifact-level verification) — pinning alone doesn't protect against artifact substitution.
- Container tags, even version-specific ones, are mutable; only digest pinning (`@sha256:...`) is truly immutable — and it must be paired with an automated update process (Renovate/Dependabot) or it silently freezes on stale, potentially vulnerable images.
- The 5-layer Docker build pattern (base image → system deps → lockfile → hash-verified install → app code) achieves reproducibility and build-cache efficiency simultaneously.
- `uv` is the pragmatic mid-2026 default for locking Python ML dependencies; `conda-lock` remains necessary for genuinely cross-language/CUDA-toolkit-pinned environments.
- Promote the same immutable artifact forward through environments; never rebuild at each stage.
- Model registry promotion (aliases like `@champion`) and environment-promotion manifests (`promotion.yaml`) are complementary, not the same mechanism — and MLflow's stage enum is deprecated in favor of aliases as of MLflow 2.9.

**Production Checklist:**

- [ ] Every dependency in every service is pinned to an exact version, not a range.
- [ ] The lockfile embeds hashes (`--generate-hashes` / native to Poetry, uv, conda-lock) and installs enforce them (`--require-hashes` / `--frozen` / `--sync`).
- [ ] Every container base image is referenced by SHA-256 digest, never `latest`, never a bare version tag.
- [ ] Digest bumps flow through an automated PR process (Renovate/Dependabot), not manual, ad hoc updates.
- [ ] Dockerfiles are ordered base image → system deps → lockfile copy → hash-verified install → app code, and use multi-stage builds to keep runtime images minimal.
- [ ] Every training run and deployed model version has an attached environment snapshot (Python/CUDA/framework versions, base image digest, lockfile hash) stored alongside its metrics and parameters.
- [ ] CI includes an automated environment-diff check against a known-good baseline, run before promotion, not just during incident response.
- [ ] The promotion pipeline builds the artifact exactly once and references it by immutable digest at every subsequent stage — no stage performs a second build.
- [ ] Staging → production promotion requires an automated gate (health check, integration test on realistic samples, load test against explicit latency/throughput SLOs, evaluation gate) plus human sign-off, all logged in a promotion manifest.
- [ ] No pipeline path allows skipping dev → staging → production, even for urgent fixes.
- [ ] Model registry promotion uses aliases/tags (e.g., `@champion`/`@challenger`), not a deprecated rigid stage enum.

---

## 9. Further Reading

Curated official documentation, GitHub repositories, notable facts, and further video/book resources for this module are maintained separately (not repeated here to avoid duplication) in this same folder:

- `references.md` — official docs (Docker multi-stage builds, image digests, NVIDIA CUDA container guide, PyTorch reproducibility notes, Poetry/uv/conda-lock docs, Google Cloud MLOps architecture, MLflow Model Registry workflows, The Twelve-Factor App, NeurIPS Paper Checklist).
- `github.md` — key repositories (`pip-tools`, `uv`, `poetry`, `conda-lock`, `hadolint`, `cookiecutter-data-science`, `Made-With-ML`).
- `videos.md` / `books.md` — supplementary video and book resources for deeper study.
