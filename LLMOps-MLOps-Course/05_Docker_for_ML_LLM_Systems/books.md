# Books — Docker for ML and LLM Systems

There is no single canonical "Docker for ML/LLMOps" book — the discipline sits at the intersection of a general containerization book and an MLOps systems-design book. The reading list below combines both.

---

## 1. Docker Deep Dive
- **Author:** Nigel Poulton
- **Edition (verified):** 4th edition, published January 2025, Packt Publishing (~312 pages). Regularly updated; described by BookAuthority as a top-rated all-time Docker book.
- **Formats:** Leanpub (PDF/ePub/Kindle), Amazon, O'Reilly Learning Platform, Packt.
- **What it teaches:** The definitive deep-dive into Docker internals — the image/container distinction, the layered filesystem and content-addressable storage, the build process and BuildKit, multi-stage builds, networking modes, volumes, and the container runtime (containerd/runc) underneath the `docker` CLI. It goes noticeably deeper than blog-level tutorials on *why* layers cache the way they do and *how* the image manifest/registry model works.
- **Difficulty:** Beginner → Intermediate (starts from zero, but the internals chapters reward careful reading even for experienced engineers).
- **Estimated reading time:** 6–8 hours for the core chapters relevant to this module (images, containers, Dockerfiles, multi-stage builds, image distribution).
- **Why it matters for this module:** This chapter assumes you understand layers, caching, and multi-stage builds at an intuitive level — Poulton's book is the best single source to solidify that mental model before applying it to ML-specific concerns (large model weights, CUDA layers, GPU runtimes) where getting the fundamentals wrong is expensive (multi-gigabyte images, slow CI, bloated attack surface).

---

## 2. Designing Machine Learning Systems: An Iterative Process for Production-Ready Applications
- **Author:** Chip Huyen
- **Publisher:** O'Reilly, 2022 (still the standard reference as of mid-2026; no second edition has been announced)
- **Relevant chapter:** Chapter 10, "Infrastructure and Tooling for MLOps" — covers containers (Docker) as part of the standardized-environment layer of the MLOps stack, alongside the broader deployment discussion of wrapping a model in a service (e.g., FastAPI) and containerizing it for delivery to a cloud provider or orchestrator.
- **What it teaches:** *Why* containerization matters at the systems level for ML specifically — reproducible dev/prod parity across teams with different local environments, the gap between "containerize the model" and "serve it reliably to thousands of concurrent users with low latency and monitoring," and where Docker sits relative to the rest of the MLOps stack (experiment tracking, orchestration, model stores, monitoring).
- **Difficulty:** Intermediate.
- **Estimated reading time:** 45–60 minutes for Chapter 10 alone; the book overall is a multi-week read but only this chapter is required for this module.
- **Why it matters for this module:** Grounds the mechanical Docker skills in the actual production motivation — this chapter of the textbook is deliberately hands-on (Dockerfiles, Compose), and Huyen's chapter supplies the "why does an ML platform team care about this" framing that keeps the hands-on work from feeling like generic DevOps trivia.

---

## 3. Docker official documentation, read as a book (Build, Compose, and Reference sections)
- **Author/Publisher:** Docker, Inc. (docs.docker.com)
- **What it teaches:** The current, authoritative, and continuously updated source for Dockerfile syntax, BuildKit build best practices, multi-stage build guidance, and the Compose Specification. Because Docker's tooling (BuildKit, Compose v2, `docker build` flags) changes faster than any print book can track, the official docs are the place to verify exact current syntax and defaults.
- **Difficulty:** Beginner → Advanced depending on section.
- **Estimated reading time:** 2–3 hours to read the "Building best practices," "Dockerfile reference," and "Compose file reference" sections relevant to this module.
- **Why it matters for this module:** Treat this as the living reference that supersedes any book's specific syntax examples. By mid-2026, BuildKit is the default builder, Compose v2 (the `docker compose` subcommand, not the standalone `docker-compose` binary) is standard, and the Compose Specification has unified what used to be separate v2/v3 file format versions — a 2022–2023-era book chapter on Compose file versions is now largely obsolete, and the official reference is the only place guaranteed to reflect this.
- **See also:** `references.md` in this module for direct links to the specific official doc pages used to write this chapter.

---

## 4. Kubernetes: Up & Running (early chapters on containers)
- **Authors:** Brendan Burns, Joe Beda, Kelsey Hightower, Lachlan Evenson
- **Publisher:** O'Reilly (multiple editions; a current edition is actively maintained as of 2026)
- **Relevant chapters:** The early "Creating and Running Containers" chapters, which cover Docker image building from the perspective of "this is the artifact Kubernetes will schedule."
- **What it teaches:** Bridges the container-building skills of this chapter to the orchestration concerns of the next module — image tagging/versioning discipline, registry push/pull workflows, and what makes an image "well-behaved" under an orchestrator (clean signal handling, no reliance on local state, sensible health endpoints).
- **Difficulty:** Intermediate.
- **Estimated reading time:** 1–1.5 hours for the relevant early chapters.
- **Why it matters for this module:** This module explicitly ends with "how this feeds into Kubernetes" — this book's early chapters are the natural next read, written by Kubernetes maintainers themselves, so the transition from Docker Compose (single-host, local dev) to Kubernetes (multi-host, production orchestration) is framed correctly from the start rather than needing correction later.

---

## Skip / deprioritize
- Older (pre-2021) Docker books that predate BuildKit, Compose v2, and the `nvidia-container-toolkit` rename (from `nvidia-docker2`) will teach outdated syntax for exactly the areas this module cares about (multi-stage builds, GPU runtime configuration). If you already own one, use it only for the conceptual sections (namespaces, cgroups, layered filesystems), and defer to official docs for command syntax.
