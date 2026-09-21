# Videos — Reproducibility and Environment Management

Curated, verifiable videos that complement this module's three source transcripts: (1) environment drift & the reproducibility paradox, (2) Docker/Conda reproducibility with SHA-pinned images, (3) dev → staging → prod promotion. Videos are ordered roughly the way you should watch them alongside the tutorial, not strictly by star rating.

---

### 1. Tutorial: Package and Dependency Management with Poetry
- **Creator/Channel:** PyVideo / PyConDE & PyData (recorded talk by Steph Samson), hosted on YouTube
- **URL:** https://www.youtube.com/watch?v=6ey-nNRvBrk
- **Difficulty:** Beginner → Intermediate
- **Rating:** ★★★★☆
- **Why it's worth watching:** A full tutorial-length walkthrough of `pyproject.toml`, dependency resolution, and `poetry.lock` — the exact mechanism the tutorial's "pin-and-hash-everything" philosophy depends on for Python-level reproducibility. Shows the resolver in action, not just the CLI surface.
- **Complements:** Part 1 (drift types) and the lockfile-strategy section of this module — grounds *why* a resolver-generated lockfile behaves differently from a hand-maintained `requirements.txt`.

---

### 2. Docker Multi-stage builds explained
- **Creator/Channel:** Independent DevOps educator (widely circulated short-form explainer), YouTube
- **URL:** https://www.youtube.com/watch?v=V0kTEk7YA70
- **Difficulty:** Beginner
- **Rating:** ★★★★☆
- **Why it's worth watching:** Concise (under 10 minutes) explanation of why and how multi-stage Dockerfiles separate build-time tooling from the runtime image — the exact pattern behind the "5-layer Docker build" concept in the source transcript. Good primer before reading the full production Dockerfile in this module.
- **Complements:** Part 2 (Docker/Conda reproducibility) — specifically the multi-stage build layer structure.

---

### 3. Docker Multi-stage for Production-ready Container Images
- **Creator/Channel:** Independent container-tooling educator, YouTube
- **URL:** https://www.youtube.com/watch?v=EkOCLmvwEhc
- **Difficulty:** Intermediate
- **Rating:** ★★★☆☆
- **Why it's worth watching:** Goes one level deeper than the 8-minute explainer — covers build-cache reuse across stages and slimming the final runtime layer, both directly relevant to keeping ML/CUDA images from ballooning to multiple gigabytes.
- **Complements:** Part 2 — the "why 5 layers, and why in this order" reasoning.

---

### 4. MLOps Zoomcamp — Experiment Tracking with Weights & Biases
- **Creator/Channel:** DataTalksClub (MLOps Zoomcamp), YouTube — presented by Soumik Rakshit (Weights & Biases)
- **URL:** https://www.youtube.com/watch?v=yNyqFMwEyL4
- **Difficulty:** Intermediate
- **Rating:** ★★★★☆
- **Why it's worth watching:** DataTalksClub's MLOps Zoomcamp is a well-regarded, free, project-based open curriculum; this session ties experiment tracking to reproducibility — logging exact code version, config, environment, and data snapshot alongside metrics, which is the practical complement to "pin everything" for training-time reproducibility (as opposed to just serving-time environment pinning).
- **Complements:** Part 1 (reproducibility paradox) — extends the idea from "environment" reproducibility to "run" reproducibility (code + data + config + environment).

---

### 5. Weights & Biases End-to-End Demo
- **Creator/Channel:** Weights & Biases (official channel)
- **URL:** https://www.youtube.com/watch?v=tHAFujRhZLA
- **Difficulty:** Intermediate
- **Rating:** ★★★☆☆
- **Why it's worth watching:** Official-channel walkthrough of versioning datasets, models, and environments together as linked artifacts — useful for understanding how a promotion pipeline (module part 3) can carry full lineage metadata forward instead of re-deriving it at each stage.
- **Complements:** Part 3 (promotion pipeline) — artifact lineage as the mechanism behind "promote the artifact forward, don't rebuild."

---

### 6. Day 3/40 — Multi-Stage Docker Build (Docker Tutorial for Beginners series)
- **Creator/Channel:** CKA/DevOps full-course series, YouTube
- **URL:** https://www.youtube.com/watch?v=ajetvJmBvFo
- **Difficulty:** Beginner
- **Rating:** ★★★☆☆
- **Why it's worth watching:** Part of a structured 40-day Docker series; useful if you want the broader Docker fundamentals (layers, caching, `.dockerignore`) surrounding the specific multi-stage pattern used in this module's production Dockerfile.
- **Complements:** Part 2 — general Docker layer-caching fundamentals that make the "5-layer build" efficient in CI.

---

## Notes on scope
No official Anthropic, OpenAI, or Google video specifically titled around "ML environment drift" or "dev/staging/prod model promotion manifests" was found to exist as a standalone talk — this is a practitioner-blog-and-course-heavy topic rather than a keynote-conference-talk-heavy one. Where a topic (e.g., MLflow stage transitions, Docker digest pinning) is better documented in official written docs than in video form, see `references.md` instead of a fabricated video entry.
