# Module 03 — Example Projects: Versioning, Registries, and Rollback

Three projects, increasing in scope and production-realism, all built around this module's central theme: version and log the full deployment triple, promote by code-driven gates, and roll back atomically using lineage. Pick the one that matches your available time and infrastructure — or work through all three in sequence as a portfolio arc.

---

## Mini Project — "Alias, Don't Stage": A Single Model's Full Registry Lifecycle

**Scope.** Using a single toy dataset (e.g. `sklearn.datasets.load_wine` or `load_breast_cancer`) and a local MLflow server (`mlflow server --backend-store-uri sqlite:///mlflow.db`), take one model through its entire registry lifecycle end to end:

1. Train and log two versions of the model (e.g. differing `max_depth` or `n_estimators`).
2. Register both, tag each with a semantic version and a `dataset_version` tag.
3. Point `@challenger` at the newer version.
4. Write a `passes_promotion_criteria()` function with an absolute floor and a relative regression guard versus `@champion`.
5. Promote (or correctly block promotion) based on the gate's output — never manually repoint the alias.
6. Serve a prediction by resolving `models:/<name>@champion`, never a hardcoded version.

**What it demonstrates.** Fluency with the *current* (alias-based, not deprecated stage-based) MLflow Model Registry API; the discipline of code-driven promotion instead of manual judgment; and the "serving resolves aliases, never versions" pattern that makes rollback trivial later. This is the smallest project that still touches every core idea in Section 3.2 and 3.3 of `tutorial.md`, and it's a strong, quick artifact to show in an interview portfolio or take-home.

**Time estimate:** half a day to a day.

---

## Medium Project — Full Deployment Triple with Prompt Registry and a `release.yaml`

**Scope.** Extend the Mini project into a small LLM-adjacent pipeline that actually exercises all three legs of the deployment triple:

1. Keep the model registry lifecycle from the Mini project (a classifier is fine — it doesn't need to be an LLM itself; the point is the *triple*, not the model architecture).
2. Add a prompt leg: register a prompt via `mlflow.genai.register_prompt(...)` that describes how a downstream service should interpret the model's output (e.g. "explain this fraud-flag decision to a reviewer"), version it, and set a `production` alias.
3. Add a (lightweight, stubbed is fine) dataset leg: a versioned reference dataset with a semantic version tag (`claims-golden:5.0.0`-style), even if it's just a CSV under Git/DVC tracking.
4. Assemble a hand-maintained `release.yaml` referencing all three legs plus a `compatibility` block with at least one real, checkable constraint.
5. Write `validate_release.py`: a script that loads `release.yaml`, resolves each version against the registry, and enforces the compatibility rules — failing loudly (non-zero exit, clear message) if any rule is violated.
6. Simulate one incident: deliberately introduce a "bad" release (e.g. bump the prompt in a way that breaks the compatibility contract, or manually mark a release `BAD` in a dedicated `releases` MLflow experiment), then implement and run `get_last_known_good()` + `restore_full_tuple()` to roll back all three legs atomically, writing a rollback audit event.

**What it demonstrates.** The full "deployment triple as the unit of reproducibility" concept from Section 3.1 in practice — not just the model leg most tutorials stop at. It shows you can operationalize a `release.yaml` as an enforced contract (via CI-style validation) rather than a decorative document, and that you understand full-tuple rollback well enough to implement it, including the "last known good, not just previous" distinction that trips up naive implementations. This is the right scope for a take-home assignment or a strong course capstone.

**Time estimate:** 2-4 days.

---

## Production-Grade Project — A Governed Release Platform with Canary Rollout and Automated Rollback

**Scope.** Build the system depicted in `architecture.md` Section 1, end to end, deployed (even if on a single cloud VM or a small Kubernetes cluster) rather than simulated locally:

1. **Registry layer.** MLflow backed by a real Postgres instance and object storage (S3/GCS/Azure Blob or MinIO for a self-hosted equivalent) — not SQLite. Register real model versions and prompt versions with full metadata: run reference, dataset version, evaluation metrics, environment/library versions.
2. **Promotion gate as CI.** A GitHub Actions (or equivalent) pipeline that runs on every candidate registration: computes absolute accuracy, relative regression delta vs. `@champion`, a latency benchmark, and (if your model/prompt combination is LLM-based) a safety/LLM-judge score against a small red-team eval set. Gate must be automated end-to-end; wire in a required human-approval step if you model a regulated-domain scenario.
3. **Release manifest as code.** `release.yaml` committed to Git per release, validated in CI (compatibility rules enforced, not just documented), with the CI job blocking merge if validation fails.
4. **Canary traffic router.** A real (or realistically simulated with a lightweight proxy/service-mesh rule) traffic split between `@champion` and `@challenger`, ramping 5% → 25% → 100% on a timer, with each stage gated by live metrics rather than a fixed clock alone.
5. **Inference logging.** Every request logs the full deployment triple (model version, prompt version, dataset version, input hash, output, latency) to structured logs or a feature store — this is what makes the rest of the system auditable.
6. **Monitoring and alerting.** Business-quality metrics (accuracy proxy, error/flag rate, or an LLM-judge-based quality score computed on a sample of live traffic) wired into the same alerting system as infra health (e.g. Prometheus/Grafana + a paging tool), so a quality regression pages on-call exactly like an infra outage would.
7. **Lineage store + rollback engine.** A queryable store of `release_id → (model, prompt, dataset, status)` history (an MLflow experiment with tagged runs is sufficient) backing a `get_last_known_good()` query, and a rollback engine that repoints all three legs atomically and writes an audit event on every invocation — whether triggered automatically by an alert or manually by an on-call engineer.
8. **Incident drill.** Actually run the scenario from Section 3.4's incident narrative against your system: inject a synthetic regression, let the alert fire, execute the rollback path, and produce a short postmortem document showing the full timeline (alert → confirm → identify current tuple → query LKG → restore → audit event → recovery).

**What it demonstrates.** This is the complete, senior-level version of the module: a system where "what's running, how did it get there, and how do we get back" are all answerable in seconds from tooling, not tribal knowledge — the exact bar the tutorial sets in its opening framing. It shows you can operate, not just train, ML/LLM systems: real backing stores, enforced (not decorative) compatibility rules, canaries gated by live metrics, and a rollback engine proven under a drill rather than just described in a README. This is portfolio-grade evidence for senior MLOps/LLMOps roles and directly rehearses the exact incident-response interview question in Section 7 of `tutorial.md`.

**Time estimate:** 1-3 weeks, depending on how much infrastructure (Kubernetes, service mesh, managed Postgres) you stand up versus simulate.
