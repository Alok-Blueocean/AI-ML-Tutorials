# Module 04 — Example Projects: Reproducibility and Environment Management

Three projects of increasing scope, all built around the same theme: making an ML/LLM service's behavior deterministic and portable across machines and time. Each demonstrates a distinct subset of the pin → hash → containerize → snapshot → promote discipline from this module. Build them in order if you're using this module to prepare for senior MLOps/LLMOps interviews — each project's evidence (lockfiles, digests, manifests, CI runs) is itself good interview material to describe concretely.

---

## Mini Project — "Lockfile & Digest Twins"

**Scope.** Take any small existing model-serving script (a toy `scikit-learn` or `numpy`-based prediction function behind a single FastAPI endpoint is enough — the model itself is not the point). Produce a hash-pinned lockfile with `uv` (or `pip-compile --generate-hashes`), and a Dockerfile whose base image is referenced by SHA-256 digest rather than a floating tag. Build the image on two different days (or simulate this by rebuilding after clearing your local Docker build cache) and confirm the resulting image digest is identical both times — the actual, concrete proof of reproducibility, not just an assertion of it.

**What it demonstrates:**
- Understanding of pin vs. hash as two distinct guarantees, applied to a real lockfile.
- Correct use of `@sha256:` digest pinning instead of a mutable tag, including how to retrieve a real digest with `docker inspect`.
- A tangible artifact (two identical digests from two separate builds) you can show in an interview as evidence you understand *why* this matters, not just that it's a best practice.

**Good stopping point:** you have a hash-pinned lockfile, a digest-pinned single-stage Dockerfile, and a short written note (a paragraph is enough) explaining what would have happened differently if you'd used `FROM python:3.11-slim` without the digest.

---

## Medium Project — "5-Layer Multi-Stage Build + Environment Snapshot"

**Scope.** Extend the Mini project into a proper multi-stage, 5-layer Dockerfile (builder stage compiles/installs hash-verified dependencies; runtime stage is minimal, non-root, with a `HEALTHCHECK`) for a slightly more realistic service — a small fine-tuned classifier or a lightweight LLM-based endpoint (e.g., a sentiment or intent classifier using a small open model, served via FastAPI). Wire in an experiment tracker (MLflow is a natural choice given this course's other modules) and log an `environment_snapshot.json` — Python version, framework versions, base image digest, and lockfile hash — as a first-class artifact attached to each training/build run, not a side comment. Add a simple `tools/env_diff.py`-style script that compares a live environment against a committed baseline and wire it into a CI job (even a single GitHub Actions workflow file is enough) so a dependency drift is caught automatically on every pull request.

**What it demonstrates:**
- The full 5-layer Docker build pattern applied correctly, with cache-efficiency behavior you can actually show (touch app code only → dependency layer stays cached; bump a lockfile pin → it correctly invalidates).
- Environment metadata treated as first-class experiment data, queryable alongside metrics and hyperparameters, not reconstructed from memory after the fact.
- A working, automated environment-drift CI gate — the difference between "we know how to fix drift when it happens" and "our pipeline catches drift before it ships."

**Good stopping point:** a green CI run that would fail if you deliberately bumped a dependency pin without updating the baseline, plus at least one MLflow (or equivalent) run showing an attached, inspectable environment snapshot artifact next to its metrics.

---

## Production-Grade Project — "Build-Once Promotion Pipeline with Registry Aliasing"

**Scope.** Build the complete system this module argues for, end to end, for a small but real LLM or ML serving workload:

1. A hash-pinned lockfile (`uv.lock` or equivalent) and a digest-pinned, multi-stage, 5-layer Dockerfile (from the Medium project), with Renovate or Dependabot configured to open PRs on base-image digest changes.
2. A CI pipeline with four distinct stages — `build-once`, `promote-to-staging`, `automated-staging-gate`, `promote-to-production` — where the image is built exactly once and every later stage references that build's image digest; no stage performs a second `docker build`.
3. A real automated staging gate: health check, an integration test against a realistic sample set with schema assertions, a load test with an explicit P95 latency SLO, and an evaluation gate against a quality threshold — with the pipeline hard-stopping (no manifest, no deploy) if any check fails.
4. A `promotion.yaml` manifest generated automatically on every successful promotion, recording the digest, every check's actual values, and the human approver — production's environment-protection rule (or equivalent) enforcing that the approval is real, not rubber-stamped.
5. Model registry integration using aliases (`@champion`/`@challenger`), not the deprecated stage enum — with serving code resolving models exclusively via alias (`models:/<name>@champion`), and a documented process for flipping the alias after a successful promotion.
6. The Medium project's environment-diff CI gate wired in as one of the automated staging-gate checks, so drift detection is structurally part of the promotion pipeline, not a bolt-on.

**What it demonstrates:**
- Full command of "promote the artifact forward, never rebuild it" as working automation, including a demonstrated failure path (a deliberately broken check that correctly halts the pipeline before production).
- An auditable, evidence-backed promotion history (`promotion.yaml` per promotion) sufficient to answer "what exactly was running in production on a given date, and why was it allowed to be there" — the compliance/audit bar named explicitly in this module.
- Correct separation of concerns between environment promotion (the pipeline's job) and model registry state (the registry's job) — the two-mechanism design this module argues is easy to collapse incorrectly and important to keep distinct.
- The complete, senior-level answer to "how would you make an ML/LLM service's behavior deterministic and portable across dev, CI, staging, and production for months or years" — demonstrated with running artifacts, not just described in words.
