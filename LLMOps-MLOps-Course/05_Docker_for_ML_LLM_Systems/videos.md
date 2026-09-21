# Videos — Docker for ML and LLM Systems

Curated, verified videos to accompany this module. Ranked with a star rating (5 = essential). Watch the "Core" tier first; treat "Deep dive" and "GPU/LLM-specific" tiers as follow-up once you have the Dockerfile/Compose basics down.

---

## Core Docker Fundamentals

### 1. Docker Tutorial for Beginners — A Full DevOps Course
- **Creator/Channel:** freeCodeCamp.org (YouTube)
- **URL:** https://www.youtube.com/watch?v=fqMOX6JJhGo
- **Duration:** ~3 hours (full-course format)
- **Difficulty:** Beginner
- **Rating:** ★★★★★
- **Why it's worth watching:** The most complete free walkthrough of Docker mechanics — images, containers, volumes, networking, Dockerfile syntax, Docker Hub — taught hands-on in a terminal rather than slides. It's the video most engineers cite as "how I actually learned Docker."
- **Complements:** The opening section of this chapter (images vs. containers, layers, basic Dockerfile syntax) before you get into ML-specific concerns.

### 2. Docker Tutorial for Beginners (3-hour full course)
- **Creator/Channel:** TechWorld with Nana (YouTube)
- **URL:** https://www.youtube.com/c/techworldwithnana (search "Docker Tutorial for Beginners \[FULL COURSE in 3 Hours\]")
- **Duration:** ~3 hours
- **Difficulty:** Beginner
- **Rating:** ★★★★★
- **Why it's worth watching:** Nana pairs simple animated diagrams with live terminal demos, and is unusually good at explaining *why* a practice exists (e.g., why layer order matters for caching) rather than just *how* to do it. Frequently recommended alongside freeCodeCamp's course as one of the two best entry points to Docker.
- **Complements:** Section 1–3 of this chapter (images/containers/layers, Dockerfile basics, `.dockerignore`).

### 3. Ultimate Docker Compose Tutorial
- **Creator/Channel:** TechWorld with Nana (YouTube)
- **Duration:** ~1 hour
- **Difficulty:** Beginner–Intermediate
- **Rating:** ★★★★☆
- **Why it's worth watching:** A focused, single-purpose walkthrough of Compose syntax (services, networks, volumes, `depends_on`, environment variables, service-name-based DNS) without the noise of a full Docker course. Good refresher once you already know plain Docker and just need Compose semantics.
- **Complements:** The Docker Compose multi-service section of this chapter (API + vector DB + Redis + monitoring stack).

---

## GPU and LLM-Serving Specific

### 4. Local NVIDIA GPU Setup for Machine Learning using Docker (TensorFlow)
- **Creator/Channel:** (independent ML/DevOps creator, verified via search — channel publishes GPU/Docker/TensorFlow setup content)
- **URL:** https://www.youtube.com/watch?v=RJlhCqRZHv4
- **Difficulty:** Intermediate
- **Rating:** ★★★☆☆
- **Why it's worth watching:** Walks through installing NVIDIA drivers, the NVIDIA Container Toolkit, and verifying `--gpus all` works end-to-end with a real training/inference container — the exact failure-prone setup step that trips up most people the first time they try to give a container GPU access.
- **Complements:** The "GPU-enabled containers" section of this chapter (nvidia-container-toolkit installation, `--gpus all`, CUDA base images).

### 5. NVIDIA Container Toolkit — official documentation walkthroughs (docs.nvidia.com)
- **Creator/Channel:** NVIDIA (official docs, not a single video, but NVIDIA also publishes companion walkthrough recordings on NVIDIA Developer channels for GTC talks on container-based ML infra)
- **Difficulty:** Intermediate–Advanced
- **Rating:** ★★★★☆ (as a reference companion, not a linear "video course")
- **Why it's worth watching/using:** Authoritative source for exactly which combinations of driver version, CUDA base image tag, and toolkit version are compatible — this compatibility matrix is the single most common source of "works on my machine" GPU container failures in production ML teams.
- **Complements:** The CUDA base image selection and GPU runtime configuration sections of this chapter.

---

## Production Patterns & MLOps Context

### 6. MLOps Zoomcamp — Module 2/4 material touching Docker for model deployment
- **Creator/Channel:** DataTalksClub (YouTube + GitHub, free course)
- **URL (course home):** https://github.com/DataTalksClub/mlops-zoomcamp
- **Difficulty:** Intermediate
- **Rating:** ★★★★☆
- **Why it's worth watching:** This is a full, free, project-based MLOps course (not just a Docker tutorial), and its deployment modules show Docker used the way real ML teams use it — packaging a trained model behind a Flask/FastAPI service, building the image, and wiring it into the rest of the stack (MLflow, orchestration, monitoring) rather than Docker in isolation.
- **Complements:** The "packaging a Python ML service" section, and previews concepts (monitoring, orchestration) covered in later modules of this textbook.

### 7. Learn Docker – Full DevOps Course for Deploying Containerized Apps
- **Creator/Channel:** freeCodeCamp.org (YouTube)
- **URL:** https://www.youtube.com/watch?v=rjjES5IsPdg
- **Difficulty:** Intermediate
- **Rating:** ★★★☆☆
- **Why it's worth watching:** Takes a full-stack app from local Dockerfile through Compose to a real deployment, which mirrors the "local dev → image build → orchestration hand-off" narrative arc of this chapter and the Kubernetes chapter that follows it.
- **Complements:** The closing section of this chapter ("how this feeds into Kubernetes").

---

## Notes on selection

- Preference was given to channels/videos independently verifiable via search (freeCodeCamp and TechWorld with Nana are both well-established, widely cited channels for Docker education) over single-creator tutorials of unknown provenance.
- No exact view counts, subscriber counts, or precise runtimes are asserted beyond what is publicly confirmed; durations are approximate where the source describes them as such (e.g., "full course in 3 hours").
- There is, as of mid-2026, no single canonical "Docker for LLM serving" flagship video from an official vLLM or NVIDIA channel that fully supersedes reading the official docs directly — for the vLLM Docker workflow specifically, treat the official docs (see `references.md`) as primary and these videos as supporting context.
