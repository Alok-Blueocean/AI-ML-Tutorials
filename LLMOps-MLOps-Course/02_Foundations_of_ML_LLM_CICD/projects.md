# Module 02 — Example Projects: Foundations of ML and LLM CI/CD

Three projects, increasing in scope and production-realism, all built around this
module's central theme: standard DevOps CI/CD is insufficient for ML/LLM systems
because data, model artifacts, and (for LLMs) prompt/retrieval config can each regress
independently of code — so the pipeline needs data validation, model evaluation, and
(for LLMs) response-quality gates, wired together with fail-fast ordering, structured
logging, and enforced non-zero exit codes. Pick the one that matches your available
time and infrastructure, or work through all three in sequence as a portfolio arc.

---

## Mini Project — A Single Schema + Row-Count Gate on a Toy Dataset

**Scope.** Using a small synthetic or public tabular dataset (e.g. a toy churn dataset,
or `sklearn.datasets.load_breast_cancer` reshaped into a "batch" mental model):

1. Write a Pandera `DataFrameSchema` enforcing column presence, types, and value
   constraints (e.g. a binary label column restricted to `{0, 1}`).
2. Add a `MIN_ROWS` completeness check.
3. Wrap both in a single `validate_data.py` script that emits structured JSON output
   and calls `sys.exit(1)` on any failure, `sys.exit(0)` on success.
4. Prove the gate works by running it against (a) a clean batch, (b) a batch with a
   corrupted column, and (c) a truncated batch — and showing the correct exit code
   each time (`echo $?` / `$LASTEXITCODE`).

**What it demonstrates.** The core mechanical insight of the whole module in its
smallest possible form: a validation *check* only becomes a validation *gate* once its
failure produces a non-zero exit code that something downstream actually reads and
obeys. This is the smallest artifact that still proves you understand the difference
between "the check ran" and "the check enforced anything" — the single most common
interview follow-up on this topic (`tutorial.md` §7, Q8).

**Time estimate:** half a day.

---

## Medium Project — Full Churn-Retrain Pipeline with Both Gates and GitHub Actions

**Scope.** Build the complete daily churn-retrain pipeline from `tutorial.md` §3.3,
end to end:

1. `fetch_data.py` — pulls (or simulates pulling) a rolling window of transactional
   data.
2. `validate_data.py` — Gate 1: Pandera schema + row-count + a KS-test or hand-rolled
   PSI drift check against a saved baseline distribution.
3. `train.py` — trains an XGBoost or scikit-learn classifier, and writes a SHA-256
   dataset hash alongside the model artifact (the idempotency mechanism).
4. `evaluate_model.py` — Gate 2: computes AUC on a held-out split, logs it to MLflow,
   and blocks (via `sys.exit(1)`) if it falls below a baseline threshold.
5. `register_model.py` — registers the candidate to a local MLflow Model Registry,
   tagged with the dataset hash, only reachable if both gates passed.
6. A GitHub Actions workflow (`.github/workflows/churn-retrain.yml`) wiring all of the
   above together with a `schedule` trigger, `workflow_dispatch` for manual runs, a
   `concurrency` group, fail-fast step ordering, and Slack (or stubbed webhook)
   notifications on both success and gate-blocked failure.
7. Demonstrate all three outcomes from `tutorial.md` §3.3's example: happy path
   (registers), Gate 1 blocked (truncated/corrupted data, training never runs), and
   Gate 2 blocked (trains fine, AUC below baseline, never registers).

**What it demonstrates.** The complete minimum-viable production pattern for a
classical ML continuous-training loop: trigger-driven automation, both data and model
gates enforced in the correct fail-fast order, idempotent re-runs via a content hash,
and a registry that only ever moves forward on the happy path. This is the right scope
for a take-home assignment or a strong course capstone for the classical-ML half of
this module.

**Time estimate:** 2-4 days.

---

## Production-Grade Project — Full Three-Gate ML/LLM CI/CD/CT Platform with Monitoring

**Scope.** Build the system depicted in `architecture.md` §1, extended to include the
LLM-specific Gate 3, deployed (even on a single cloud VM or small Kubernetes cluster)
rather than simulated locally:

1. **Source of truth layer.** A real Git repo for pipeline code/schemas/eval configs, a
   data store (even object storage standing in for a data lake) for raw batches, and —
   for the LLM leg — a prompt/config registry (MLflow 3's GenAI Prompt Registry is a
   reasonable default) versioning system-prompt templates independently of model code.
2. **Gate 1 — data validation, complete.** Pandera schema, row-count, and drift
   (KS-test and/or PSI) checks running against a versioned baseline, with a documented,
   owned threshold — not a value set once and forgotten.
3. **Gate 2 — model evaluation.** `mlflow.evaluate()` (or an equivalent harness)
   comparing a candidate against the current registered baseline before any
   registration/promotion step runs.
4. **Gate 3 — response/prompt quality.** A fixed, versioned golden eval set (10s–100s
   of prompts) scored by an LLM-as-judge, with the judge's own scoring criteria
   documented and periodically spot-checked by a human for drift/gameability.
5. **Orchestration.** A real trigger-driven orchestrator (GitHub Actions is sufficient;
   Airflow/Metaflow/Vertex Pipelines if you want the closer Netflix/Uber-style analogue)
   running all three gates in fail-fast order, with structured JSON logs from every
   gate and a hard non-zero exit on any failure — verified with an explicit test of the
   failure path, not just the happy path.
6. **Registry + staged rollout.** MLflow Model Registry (real Postgres + object storage
   backing store, not SQLite) plus a canary/staged rollout step (even a simple
   percentage-based traffic split) sitting *after* the offline gates, since offline
   evals alone cannot fully capture real-world usage-distribution shift.
7. **Monitoring/CT feedback loop.** The *same* drift/quality metric computations used in
   Gate 1/2/3 also run continuously against live traffic, feeding a dashboard; wire an
   anomaly in that dashboard (e.g., PSI crossing 0.25 on live traffic) to trigger an
   out-of-schedule retrain — closing the loop back to the orchestration layer, which is
   what makes the system genuinely Continuous Training rather than just Continuous
   Deployment.
8. **Incident drill.** Deliberately inject a regression at each of the three gates (bad
   schema, underperforming model, degraded prompt) one at a time, and confirm: (a) the
   correct gate blocks it, (b) the previous version keeps serving, (c) the pipeline's
   idempotency (dataset hash) survives a retry, and (d) the monitoring layer would have
   independently caught the same issue post-deploy if the pre-deploy gate had somehow
   been bypassed.

**What it demonstrates.** The complete, senior-level version of this module: a platform
where code, data, model, and prompt are each independently versioned and independently
gated, where the pre-deploy gate and the post-deploy monitor share the same metric
definitions (per `architecture.md` §1's key structural point), and where every gate
failure is provably enforced (non-zero exit, structured logs) rather than merely
logged. This is portfolio-grade evidence for senior MLOps/LLMOps roles and directly
rehearses the reasoning behind the case studies in `tutorial.md` §4 (Netflix,
Uber/Michelangelo, frontier LLM labs) and the incident-response style interview
questions in §7.

**Time estimate:** 1-3 weeks, depending on how much real infrastructure (managed
Postgres, Kubernetes, a live LLM API budget) you stand up versus simulate/stub.
