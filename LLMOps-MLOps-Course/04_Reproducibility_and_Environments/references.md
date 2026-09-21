# References — Reproducibility and Environment Management

Official documentation, papers, and engineering resources, grouped by the three source-transcript themes.

---

## 1. Environment drift, the reproducibility paradox, and pin-and-hash philosophy

### Improving Reproducibility in Machine Learning Research (NeurIPS 2019 Reproducibility Program Report)
- **URL:** https://arxiv.org/abs/2003.12206 (also published in JMLR: https://jmlr.org/papers/volume22/20-303/20-303.pdf)
- **Type:** Peer-reviewed research paper
- **What it teaches:** Empirical documentation of why ML experiments fail to reproduce — undocumented hyperparameters, unpinned dependencies, unspecified hardware, missing code/data availability — and the design of the resulting NeurIPS reproducibility checklist.
- **Difficulty / reading time:** Intermediate; 45–60 minutes.

### NeurIPS Paper Checklist Guidelines
- **URL:** https://neurips.cc/public/guides/PaperChecklist
- **Type:** Official conference documentation
- **What it teaches:** The current, living checklist authors must complete, covering reproducibility, transparency, and experimental rigor — useful as a "what should have been pinned/logged" checklist even outside of paper submission.
- **Difficulty / reading time:** Beginner; 15 minutes.

### The Twelve-Factor App — Factor II (Dependencies), Factor III (Config), Factor X (Dev/prod parity)
- **URL:** https://12factor.net/dependencies · https://12factor.net/config · https://12factor.net/dev-prod-parity
- **Type:** Official methodology documentation (Heroku engineering)
- **What it teaches:** The platform-agnostic case for explicit dependency declaration, config/code separation, and minimizing dev/prod environment gaps — the conceptual ancestor of this module's drift-prevention and promotion principles.
- **Difficulty / reading time:** Beginner; 30–45 minutes for the three factors.

### PyTorch Reproducibility Notes
- **URL:** https://docs.pytorch.org/docs/stable/notes/randomness.html
- **Type:** Official framework documentation
- **What it teaches:** How to (and why you often cannot fully) achieve deterministic PyTorch runs — `torch.manual_seed`, `torch.backends.cudnn.deterministic`, and the specific CUDA operations (e.g., atomic adds) that remain sources of nondeterminism even after seeding. Directly relevant to why "CUDA drift" is a distinct category from library-version drift.
- **Difficulty / reading time:** Intermediate; 20–30 minutes.

---

## 2. Docker/Conda reproducibility, SHA-pinned base images, multi-stage builds

### Multi-stage builds — Docker official documentation
- **URL:** https://docs.docker.com/build/building/multi-stage/
- **Type:** Official documentation
- **What it teaches:** The canonical reference for multi-stage `Dockerfile` syntax — `FROM ... AS <name>`, `COPY --from=<stage>`, and why separating build and runtime stages minimizes final image size and attack surface. Directly underlies the "5-layer Docker build pattern" this module teaches.
- **Difficulty / reading time:** Beginner; 20 minutes.

### Image digests — Docker Docs
- **URL:** https://docs.docker.com/dhi/core-concepts/digests/
- **Type:** Official documentation
- **What it teaches:** What a SHA-256 image digest is, why it is immutable (unlike a tag, which can be repointed at new content), and how pulling `image@sha256:...` guarantees byte-identical images across time and machines — the mechanism behind "SHA-pinned base images."
- **Difficulty / reading time:** Beginner; 15 minutes.

### NVIDIA CUDA container images — supported tags and deep learning frameworks user guide
- **URL:** https://docs.nvidia.com/deeplearning/frameworks/user-guide/index.html · tag reference: https://gitlab.com/nvidia/container-images/cuda/blob/master/doc/supported-tags.md
- **Type:** Official vendor documentation
- **What it teaches:** The `runtime` vs. `devel` CUDA image variants, how tags map to CUDA/cuDNN/TensorRT versions, and why the `latest` tag is deprecated/unavailable for these images (pinning to an explicit tag like `13.3.1-runtime-ubuntu24.04`, or better, its SHA digest, is mandatory practice, not optional).
- **Difficulty / reading time:** Intermediate; 20–30 minutes.
- **2026 note:** `nvidia/cuda:latest` returns a "manifest unknown" error — this is a hard requirement to pin, not a style preference, as of mid-2026.

### Poetry — Dependency specification & Managing dependencies
- **URL:** https://python-poetry.org/docs/dependency-specification/ · https://python-poetry.org/docs/managing-dependencies/
- **Type:** Official documentation
- **What it teaches:** How `poetry.lock` captures the fully resolved dependency graph (including transitive dependencies and hashes), and how `poetry install --sync` reproduces an environment exactly from that lockfile.
- **Difficulty / reading time:** Beginner–Intermediate; 30 minutes.

### uv — Locking environments (Astral official docs)
- **URL:** https://docs.astral.sh/uv/pip/compile/ · https://docs.astral.sh/uv/concepts/projects/layout/
- **Type:** Official documentation
- **What it teaches:** How `uv.lock` provides deterministic, cross-platform dependency resolution, and the project layout conventions uv expects. As of mid-2026, uv is the fastest-growing option in this space and worth treating as a first-class alternative to pip-tools/Poetry, not a footnote.
- **Difficulty / reading time:** Beginner–Intermediate; 20–30 minutes.

### conda-lock — PyPI project page and documentation
- **URL:** https://pypi.org/project/conda-lock/
- **Type:** Official package documentation
- **What it teaches:** How conda-lock performs a one-time conda solve per target platform and emits a fully pinned lockfile, closing the reproducibility gap left by plain `conda env export` (which can re-resolve differently across platforms/time).
- **Difficulty / reading time:** Beginner–Intermediate; 20 minutes.

### hadolint — Dockerfile linter documentation
- **URL:** https://github.com/hadolint/hadolint · https://hadolint.com/
- **Type:** Official project documentation
- **What it teaches:** Rule-by-rule reference for Dockerfile anti-patterns (unpinned `FROM`, missing package-manager cache cleanup, running as root, etc.) that can be enforced automatically in CI.
- **Difficulty / reading time:** Beginner; 15–20 minutes.

---

## 3. Environment promotion: dev → staging → prod

### MLOps: Continuous delivery and automation pipelines in machine learning
- **URL:** https://docs.cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning
- **Type:** Official Google Cloud architecture documentation
- **What it teaches:** The canonical reference architecture for CI/CD/CT (continuous training) in ML systems, including how environments/pipelines are promoted across dev, staging, and production stages, and what automated checks belong at each gate.
- **Difficulty / reading time:** Intermediate; 45–60 minutes.

### Practitioners Guide to Machine Learning Operations (MLOps) — Google Cloud whitepaper
- **URL:** https://cloud.google.com/resources/mlops-whitepaper
- **Type:** Official whitepaper (requires providing an email to download, content itself is Google-authored and free)
- **What it teaches:** End-to-end MLOps maturity model and staged rollout practices, complementing the continuous-delivery architecture doc above with organizational/process guidance.
- **Difficulty / reading time:** Intermediate; 45–75 minutes.

### MLflow Model Registry — Workflows and stage transitions
- **URL:** https://mlflow.org/docs/latest/ml/model-registry/workflow/ · https://mlflow.org/docs/latest/ml/model-registry/
- **Type:** Official documentation
- **What it teaches:** How a model registry tracks versioned model artifacts and their promotion state, with each transition recorded as an auditable activity — directly analogous to the promotion.yaml manifest concept in this module.
- **Difficulty / reading time:** Beginner–Intermediate; 30 minutes.
- **2026 note — important drift from older sources:** MLflow's original `Staging`/`Production`/`Archived` **stages** have been deprecated since MLflow 2.9 and are scheduled for removal in a future major release. Any 2022–2023-era tutorial teaching `transition_model_version_stage(stage="Production")` is now teaching a deprecated API. Current MLflow (2.9+, including MLflow 3.x) instead uses flexible **model version aliases** (e.g., `@champion`, `@challenger`) and **tags**, which allow multiple named pointers per model and arbitrary custom labels instead of a fixed four-state machine. When teaching or building a promotion pipeline in mid-2026, model the promotion.yaml manifest and gating logic around aliases/tags, not the deprecated stage enum.

### Databricks: MLOps vs DevOps
- **URL:** https://www.databricks.com/blog/mlops-vs-devops
- **Type:** Official vendor engineering blog
- **What it teaches:** Where standard software CI/CD promotion practices (build once, promote the same artifact) map cleanly onto ML pipelines, and where ML-specific concerns (data validation gates, model quality gates, drift monitoring) require additional stages beyond a typical software promotion pipeline.
- **Difficulty / reading time:** Beginner–Intermediate; 20–30 minutes.

---

## Summary table

| Resource | Theme | Difficulty | Time |
|---|---|---|---|
| NeurIPS 2019 Reproducibility Report | Drift/paradox | Intermediate | 45–60 min |
| NeurIPS Paper Checklist | Drift/paradox | Beginner | 15 min |
| 12-Factor App (II, III, X) | Drift/paradox, promotion | Beginner | 30–45 min |
| PyTorch Reproducibility Notes | Drift (CUDA/determinism) | Intermediate | 20–30 min |
| Docker Multi-stage builds docs | Docker/Conda | Beginner | 20 min |
| Docker Image digests docs | Docker/Conda | Beginner | 15 min |
| NVIDIA CUDA container docs | Docker/Conda | Intermediate | 20–30 min |
| Poetry dependency docs | Lockfiles | Beginner–Int. | 30 min |
| uv locking docs | Lockfiles | Beginner–Int. | 20–30 min |
| conda-lock docs | Lockfiles | Beginner–Int. | 20 min |
| hadolint docs | Docker/Conda | Beginner | 15–20 min |
| Google Cloud MLOps CD architecture | Promotion | Intermediate | 45–60 min |
| Google Cloud MLOps whitepaper | Promotion | Intermediate | 45–75 min |
| MLflow Model Registry workflow docs | Promotion | Beginner–Int. | 30 min |
| Databricks MLOps vs DevOps | Promotion | Beginner–Int. | 20–30 min |
