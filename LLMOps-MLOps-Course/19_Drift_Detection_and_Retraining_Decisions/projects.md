# Module 19 — Projects: Drift Detection and Retraining Decisions

Three projects of increasing scope, each meant to be a portfolio-quality artifact demonstrating a distinct level of production maturity in drift monitoring and retraining decisions.

---

## Project 1 (Mini) — Tabular Drift-Detection CLI

### Scope

Build a small, self-contained command-line tool that takes two CSV files (a baseline snapshot and a current snapshot of the same schema) and produces a per-feature drift report.

**Requirements:**
- Accepts `--baseline baseline.csv --current current.csv --output report.json` (or similar) via `argparse`/`click`.
- Computes PSI and KS statistic/p-value for every numeric column, using baseline-derived bin edges (per §3.2's implementation — do not use a library that hides this detail; implement PSI/KS yourself or wrap SciPy's `ks_2samp` explicitly).
- Computes a simple categorical drift check (e.g., chi-squared or a proportion-difference table) for non-numeric columns.
- Outputs a structured JSON report: per-column PSI, PSI verdict (stable/moderate/significant), KS statistic, KS p-value, and an overall `dataset_drift_detected` boolean if more than N% of columns exceed the PSI warning threshold.
- Exits with a non-zero status code if `dataset_drift_detected` is true — so it can be wired into a CI pipeline as a gate (e.g., "fail this build if the new training data has drifted significantly from the last validated baseline").
- Includes a README explaining the tool's usage and a worked example with a synthetic dataset showing both a "no drift" and a "drift detected" run.

### What It Demonstrates

The statistical foundation of drift detection (§3.2) implemented correctly and packaged as a reusable, scriptable tool — the kind of utility a small team builds before adopting a heavier library like Evidently, and useful evidence that you understand the mechanics rather than only knowing how to call a library function.

---

## Project 2 (Medium) — Full Input/Behavioral/Quality Drift Pipeline with Dashboard and Alerting

### Scope

Build an end-to-end drift-monitoring pipeline for a simulated (or real, if you have one) deployed model/LLM service, covering all three drift tracks from §3.1 plus a dashboard and Slack alerting.

**Requirements:**
- **Input drift job:** using Evidently AI (or your own PSI/KS implementation from Project 1) to compare a rolling window of "production" feature data against a frozen baseline, on a schedule (a cron job, GitHub Actions scheduled workflow, or an Airflow/Prefect DAG if you've covered orchestration).
- **Behavioral drift job** (if working with an LLM use case): response-length and refusal-rate tracking from a synthetic or real response log, per §3.1/§3.4.
- **Quality drift job:** either a real labeled evaluation set scored periodically, or a NannyML CBPE-style label-free performance estimate if you have a classification use case with delayed labels.
- **Decision matrix:** implement and wire in the `decide_action` logic from §3.3, driven by the three jobs' outputs.
- **Dashboard:** a Grafana dashboard (backed by Prometheus, if you push metrics there) or a Streamlit/Dash app, showing PSI per feature, response length trend, refusal rate trend, and eval score trend together, matching the panel layout in §3.4.
- **Alerting:** real (or realistically mocked with a test webhook service like webhook.site) Slack alerting with tuned thresholds — demonstrate the calibration procedure from §3.4 with a short writeup showing how you derived your specific threshold values from historical/simulated data, not picked them arbitrarily.
- **Incident runbook:** at least one simulated incident walked through the full five-step runbook (§3.6) with a documented `IncidentRecord` artifact.

### What It Demonstrates

The full "detect → decide → alert → respond" loop this module is built around, integrating a real drift-detection library (Evidently and/or NannyML) rather than only hand-rolled statistics, plus the operational discipline (tuned thresholds, decision matrix, runbook) that separates a working demo from a production-credible monitoring system.

---

## Project 3 (Production-Grade) — Automated Drift-to-Retrain Observability Platform

### Scope

Build a complete, deployable drift-observability platform that closes the loop from detection through to an automated (or human-gated) retraining trigger, integrated with a model registry and CI/CD, for a model/LLM service you also deploy (reusing infrastructure from Modules 05/06 and Module 13 if available).

**Requirements:**
- **Telemetry ingestion:** OpenTelemetry-instrumented service (per Module 14) emitting the metadata needed for drift jobs — request features, response metadata (length, refusal flags), and periodic sampled evaluation scores — without leaking raw PII, per Module 14's privacy-safe telemetry design.
- **Drift-detection layer:** scheduled jobs computing input drift (PSI/KS or Evidently), behavioral drift, and quality drift (NannyML CBPE or a scheduled LLM-judge evaluation run per Module 11), all writing to a durable metrics store (Prometheus/warehouse table).
- **Decision engine:** a service or scheduled job implementing the full decision matrix (§3.3) against live metrics, producing a recommended action with supporting evidence (not just a label — the actual PSI values, refusal rate delta, eval score delta that justify the recommendation).
- **Human-gated automation:** the decision engine's output triggers a structured Slack message (or a ticket in an issue tracker) with the recommended action and evidence, requiring a human approval click/comment before any retraining pipeline or rollback actually executes — full automation without a human gate is explicitly out of scope here; the goal is decision support with an audit trail, not unattended autonomous retraining.
- **Retraining trigger integration:** on approval, the platform kicks off an actual retraining pipeline run (can be a real training job or a realistic stub) that registers its output as a new MLflow model version, tagged with the drift evidence that triggered it (linking back to the specific `IncidentRecord`).
- **Rollback integration:** on a rollback decision, the platform actually executes a rollback against your deployment (Module 03/06 rollback mechanism), not just logs the recommendation.
- **Full runbook automation:** every decision-engine run produces a persisted `IncidentRecord`-style document (database row or file), and a periodic (weekly) digest report is generated summarizing the period's drift signals, decisions made, and outcomes — matching §3.4's weekly report and §3.6's documentation discipline.
- **Documentation:** an architecture diagram (can extend the ASCII diagrams in this module's tutorial), a runbook document usable by a real on-call engineer with no prior context, and a short "why human-gated, not fully autonomous" design-decision writeup.

### What It Demonstrates

A genuinely production-shaped MLOps observability platform: real telemetry, real drift statistics, a transparent and auditable decision layer, integration with existing versioning/registry/deployment infrastructure from earlier modules, and — critically — a deliberate, justified choice about where human judgment stays in the loop rather than naively automating the entire retrain/rollback decision end to end. This is the project to point to as evidence of "I can design and operate the continuous monitoring and drift-response layer of a real ML/LLM system," which is the stated goal underlying this whole module.
