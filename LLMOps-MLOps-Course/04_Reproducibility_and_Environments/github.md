# GitHub Repositories — Reproducibility and Environment Management

---

### jazzband/pip-tools
**URL:** https://github.com/jazzband/pip-tools
**Purpose:** Compiles a `requirements.in` (loose, human-authored constraints) into a fully pinned, hash-verifiable `requirements.txt` (`pip-compile`), and synchronizes a virtual environment to exactly match it (`pip-sync`).
**Popularity tier:** Very popular / widely adopted — a long-standing, actively maintained community-governed project (Jazzband collective) with several thousand stars and regular releases; effectively the reference implementation of "pin-and-hash-everything" for plain pip workflows.
**Why it matters:** This is the most direct, minimal-dependency implementation of the lockfile philosophy this module teaches for teams that want to stay on pip rather than adopt Poetry or uv. `pip-compile --generate-hashes` is what makes a `requirements.txt` behave like a real lockfile instead of a wish list.
**Relation to this module:** Core tool referenced in the lockfile-strategies section; shows the "before Poetry/uv" generation of the same idea, which is still the standard in many regulated/legacy ML shops.

---

### astral-sh/uv
**URL:** https://github.com/astral-sh/uv
**Purpose:** A single Rust-based binary that replaces pip, pip-tools, virtualenv, pyenv, and (largely) Poetry — dependency resolution, lockfile generation (`uv.lock`), virtual environment management, and Python version management in one tool.
**Popularity tier:** Extremely popular and rapidly growing — one of the fastest-adopted Python tooling projects in recent years, now a common default in new ML/LLM service repos as of 2026.
**Why it matters:** By mid-2026, `uv` has become the pragmatic default for new Python projects needing reproducible, fast, cross-platform lockfiles — a torch+transformers install that takes pip 3–5 minutes completes in well under a minute with uv's caching and parallel resolver. Any 2024-era source that only discusses pip-tools/Poetry/conda is now missing this option.
**Relation to this module:** Directly relevant to the lockfile-strategy comparison table in this module — presented as the modern alternative worth evaluating alongside pip-tools, Poetry, and conda-lock.

---

### python-poetry/poetry
**URL:** https://github.com/python-poetry/poetry
**Purpose:** Dependency management, packaging, and virtual environment tooling built around a single `pyproject.toml` manifest and a resolver-generated `poetry.lock`.
**Popularity tier:** Very popular / widely adopted — one of the most-used Python packaging tools, with broad ecosystem plugin support and long-term maintenance.
**Why it matters:** Demonstrates a fully resolver-driven lockfile (transitive dependencies included, not just top-level pins) with reproducible, cross-platform installs — the property that plain `pip freeze` cannot guarantee, which is exactly the limitation called out in the module's drift-debugging discussion.
**Relation to this module:** Reference implementation for the "lockfile vs. requirements.txt" distinction taught in this module — a `poetry.lock` records the full resolved dependency graph plus hashes, not just what happened to be installed at freeze time.

---

### conda/conda-lock
**URL:** https://github.com/conda/conda-lock (organization: conda-incubator / conda)
**Purpose:** Generates fully reproducible, per-platform lockfiles for conda/mamba environments by performing the conda solve once and recording the exact resolved package set (including non-Python binary dependencies like CUDA, MKL, BLAS) for each target platform.
**Popularity tier:** Popular within the conda/scientific-Python ecosystem — the de facto standard third-party tool for making conda environments reproducible, since `conda env export` alone does not guarantee this.
**Why it matters:** Directly addresses the module's stated "conda export limitations" — a plain `conda env export --from-history` or full export can still resolve differently on a different platform or at a different time. conda-lock closes that gap for the CUDA/native-library-heavy dependencies that pip-based lockfiles cannot pin.
**Relation to this module:** The tool that completes the Docker/Conda reproducibility story in Part 2 — used specifically where GPU/CUDA/native-library pinning is required, which pip-tools/Poetry alone cannot fully guarantee.

---

### hadolint/hadolint
**URL:** https://github.com/hadolint/hadolint
**Purpose:** A Dockerfile linter (built on ShellCheck) that catches anti-patterns — unpinned base images, missing `--no-install-recommends`, use of `latest`, unnecessary layers — before a build ever runs.
**Popularity tier:** Very popular / widely adopted, and integrated into GitHub Actions, GitLab CI, and pre-commit workflows across the industry.
**Why it matters:** Automates enforcement of the exact discipline this module teaches (SHA-pinned base images, minimal layers) so that "pin everything" becomes a CI-enforced gate rather than a code-review reminder that inevitably gets skipped under deadline pressure.
**Relation to this module:** A natural addition to the automated staging-gate checks in Part 3 — Dockerfile hygiene is a pre-build gate, upstream of the runtime staging checks.

---

### drivendataorg/cookiecutter-data-science
**URL:** https://github.com/drivendataorg/cookiecutter-data-science
**Purpose:** A standardized, opinionated project-directory template for data science/ML repositories — separating raw/interim/processed data, notebooks, source code, and environment specification files into a consistent, reproducible layout.
**Popularity tier:** Very popular / widely adopted — one of the most commonly cited data-science project scaffolding tools, maintained by DrivenData.
**Why it matters:** Reproducibility is not only about the runtime environment — it's also about *where* environment specification files live and how consistently they're named/discovered across a team's projects. This template operationalizes that consistency at the repo-structure level.
**Relation to this module:** Useful reference structure for where `requirements.in`/`pyproject.toml`/`environment.yml`/`Dockerfile` should live in a real project, complementing the environment-content focus of this module with a project-layout convention.

---

### GokuMohandas/Made-With-ML (and the companion mlops-course)
**URL:** https://github.com/GokuMohandas/Made-With-ML
**Purpose:** A free, open-source, project-based course (madewithml.com) covering the full ML lifecycle — data, modeling, serving, testing, reproducibility, monitoring, and data engineering — with runnable code for every lesson.
**Popularity tier:** Very popular / widely adopted — one of the most widely referenced open MLOps curricula, with a large community following.
**Why it matters:** Explicitly includes a "reproducibility" lesson as part of a coherent, end-to-end course rather than treating it as an isolated concern — useful for seeing how environment reproducibility interacts with the rest of the ML lifecycle (data versioning, CI, monitoring) that later modules of this textbook also cover.
**Relation to this module:** A good "second opinion" resource — cross-check this module's Docker/lockfile/promotion content against Made With ML's treatment of the same lifecycle stage.
