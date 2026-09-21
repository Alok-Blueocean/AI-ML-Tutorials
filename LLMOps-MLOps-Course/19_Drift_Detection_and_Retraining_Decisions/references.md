# Module 19 — References: Drift Detection and Retraining Decisions

Curated references for continuous drift monitoring, statistical drift detection, and retraining-decision frameworks. Organized by category, with a note on what each teaches and why it's included in this module specifically.

---

## Official Tool Documentation

### Evidently AI
- **Docs:** https://docs.evidentlyai.com/
- **GitHub:** https://github.com/evidentlyai/evidently
- **What it teaches:** How to generate structured "reports" comparing a reference and current dataset, with prebuilt presets (`DataDriftPreset`, `DataQualityPreset`, `TargetDriftPreset`, classification/regression performance presets). Read the docs' "Data Drift" and "How it works" sections specifically for the exact statistical test each column type defaults to (Evidently auto-selects PSI, KS, Wasserstein distance, chi-squared, or Jensen-Shannon divergence depending on column type and sample size — worth understanding rather than treating as a black box). Also covers integrating drift checks as CI/CD gates and Evidently's hosted monitoring platform for continuous dashboards.

### NannyML
- **Docs:** https://nannyml.readthedocs.io/
- **GitHub:** https://github.com/NannyML/nannyml
- **What it teaches:** Label-free performance estimation — specifically CBPE (Confidence-Based Performance Estimation) for classification and DLE (Direct Loss Estimation) for regression. Read the "How it works" / algorithm deep-dive pages to understand the actual assumption being made (that the confidence-to-correctness relationship learned on a labeled reference period continues to hold in the unlabeled analysis period) and where that assumption can break down. Also documents NannyML's own multivariate drift detection (data reconstruction with PCA) as a complement to univariate PSI/KS.

### whylogs
- **Docs:** https://docs.whylabs.ai/docs/whylogs-overview/
- **GitHub:** https://github.com/whylabs/whylogs
- **What it teaches:** Lightweight, mergeable statistical profiling designed for streaming/high-volume pipelines — profiles are small enough to compute inline and merge across distributed workers/time windows without re-touching raw data. Useful contrast against Evidently's more batch/report-centric model.

### WhyLabs Platform
- **Docs:** https://docs.whylabs.ai/
- **What it teaches:** The hosted monitoring/alerting layer built on top of whylogs profiles — drift monitors, anomaly detection, and alerting configuration for teams that want a managed platform rather than self-hosting the comparison/alerting logic.

### Arize Phoenix
- **Docs:** https://docs.arize.com/phoenix/
- **GitHub:** https://github.com/Arize-ai/phoenix
- **What it teaches:** LLM/embedding-centric observability — embedding drift visualization (UMAP projections of embedding space over time), trace-level LLM debugging, and evaluation integrations. The natural reference for extending this module's embedding-distance drift methods (§3.2) into a full visual, explorable workflow, and pairs directly with Module 18's agent tracing material.

### Arize AI (platform docs, broader than Phoenix)
- **Docs:** https://docs.arize.com/arize/
- **What it teaches:** Arize's hosted ML observability platform more broadly — drift, data quality, and performance monitoring for both classical ML and LLM workloads, useful for comparing a "build vs. buy" tradeoff against a self-hosted Evidently/NannyML/whylogs stack.

### Deepchecks
- **Docs:** https://docs.deepchecks.com/
- **GitHub:** https://github.com/deepchecks/deepchecks
- **What it teaches:** "Check suite" style validation — train-test drift checks, data integrity checks, and model performance checks runnable as a batch/CI step. Read the "Train-Test Validation" suite docs specifically for how Deepchecks frames drift checks as one category among a broader pre-deployment validation battery, a useful contrast to Evidently/NannyML's more continuously-running orientation.

### Great Expectations (GX)
- **Docs:** https://docs.greatexpectations.io/
- **GitHub:** https://github.com/great-expectations/great_expectations
- **What it teaches:** Data-quality/validation expectations (schema, null rates, ranges, some distributional checks) as a pipeline gate — the data-engineering half of the drift story. Useful for understanding where GX's "is this data well-formed" concern ends and where this module's "has the distribution meaningfully shifted" concern begins.

### MLflow
- **Docs:** https://mlflow.org/docs/latest/index.html
- **What it teaches:** Not a drift tool itself, but the registry/experiment-tracking system (Module 13) that a retraining decision from this module's decision matrix writes back into — new run, new model version, stage transition. Read the Model Registry docs specifically for how to record "this version was promoted because of a drift-triggered retrain" as first-class lineage metadata.

### OpenTelemetry
- **Docs:** https://opentelemetry.io/docs/
- **GenAI Semantic Conventions:** https://opentelemetry.io/docs/specs/semconv/gen-ai/
- **What it teaches:** The telemetry substrate (Module 14) that feeds this module's drift jobs — traces/metrics/logs are the raw material; drift monitoring is a batch/streaming analysis layered on top of that same data.

### Prometheus / Grafana
- **Prometheus docs:** https://prometheus.io/docs/
- **Grafana docs:** https://grafana.com/docs/grafana/latest/
- **What it teaches:** The metrics-storage and dashboarding layer used to operationalize drift scores (PSI per feature, refusal rate, eval score) as time series with alerting rules — directly reused from Module 14's alerting patterns, applied here to drift-specific metrics and thresholds.

---

## Papers and Foundational References

- **Kullback, S. & Leibler, R.A. (1951), "On Information and Sufficiency."** The original formulation of KL divergence — the information-theoretic quantity PSI is built from. Worth reading the original definition to understand why KL is asymmetric and why that asymmetry matters when choosing a direction to report.
- **Massey, F.J. (1951), "The Kolmogorov-Smirnov Test for Goodness of Fit."** The original KS test paper — useful for understanding the test's actual null hypothesis and why it was designed as a distribution-free (nonparametric) goodness-of-fit test, which is exactly why it needs no binning assumption unlike PSI.
- **Siddiqi, N. (2006), "Credit Risk Scorecards: Developing and Implementing Intelligent Credit Scoring."** The standard reference for PSI's origin and its conventional 0.1/0.25 interpretability bands in credit-risk population monitoring — the source of the thresholds most drift-monitoring tooling still cites today, and useful context for why those specific numbers exist rather than being arbitrary.
- **Quionero-Candela, J., Sugiyama, M., Schwaighofer, A., & Lawrence, N. (2009), "Dataset Shift in Machine Learning."** The foundational text formalizing covariate shift, prior probability shift, and concept drift as distinct, precisely defined phenomena — the academic backbone of the input/behavioral/quality drift taxonomy this module teaches in applied form.
- **Moreno-Torres, J.G. et al. (2012), "A Unifying View on Dataset Shift in Classification."** A widely-cited survey unifying terminology across the dataset-shift literature (useful if the field's overlapping vocabulary — covariate shift, concept drift, prior shift, dataset shift — gets confusing).
- **Gama, J. et al. (2014), "A Survey on Concept Drift Adaptation."** A comprehensive survey of concept-drift detection algorithms and adaptation strategies in the streaming-data ML literature — useful background for understanding drift-detection approaches beyond the batch-comparison methods (PSI/KS) this module emphasizes, e.g., sequential/streaming drift detectors like ADWIN and DDM.
- **Sculley, D. et al. (2015), "Hidden Technical Debt in Machine Learning Systems" (NeurIPS).** Not drift-specific, but the canonical paper articulating why production ML systems accumulate silent, unmonitored risk — directly motivates why a dedicated drift-monitoring discipline (rather than "the model passed offline eval once") is necessary at all.
- **NannyML's public research on CBPE and DLE** (see NannyML's documentation and engineering blog, linked above) — the algorithmic basis for label-free performance estimation referenced in §3.5; read this after the Quionero-Candela paper to see how a specific modern tool operationalizes the dataset-shift theory into a practical estimator.

---

## Well-Known Engineering Blogs and Talks (Context, Not Citations)

- **Uber Engineering blog, Michelangelo ML platform posts** — describes centralized feature stores and monitoring built for a highly heterogeneous, multi-market deployment; useful real-world grounding for the "baseline per segment, not one global baseline" pattern discussed in §6's case-study section. Treat as architectural context, not a confirmed drift-monitoring implementation spec.
- **Netflix Technology Blog, ML/personalization and Metaflow posts** — describes the scale and diversity of models Netflix operates and the platform tooling (Metaflow) built around them; useful grounding for why distinguishing seasonal/expected drift from genuine staleness matters at scale (§6).
- **Google SRE Book, "Monitoring Distributed Systems" chapter** (https://sre.google/sre-book/monitoring-distributed-systems/) — not ML-specific, but the canonical source for the "alert on symptoms, not causes" and "avoid alert fatigue" principles this module's alerting section (§3.4) directly inherits from classical SRE practice.
- **OpenAI and Anthropic model-versioning/deprecation documentation** (see each provider's own API docs for model snapshot naming and deprecation policies) — read alongside §6's case-study discussion of pinning to dated model snapshots as a mitigation for upstream-induced behavioral drift; these are primary-source docs on how model aliasing actually works, not drift-monitoring guidance per se.

---

## Cross-References Within This Course

- **Module 03 (Versioning, Registries, and Rollback)** — the rollback mechanism this module's decision matrix routes into when behavioral drift correlates with a recent deploy.
- **Module 09-11 (Prompt Lifecycle, Evaluation Datasets, LLM-as-Judge)** — the source of the "quality drift" / eval-score signal used throughout §3.1, §3.3, §3.4.
- **Module 13 (Experiment Tracking and MLflow Deep Dive)** — where a drift-triggered retraining decision's resulting model version and lineage get recorded.
- **Module 14 (Observability, OpenTelemetry, and Production Monitoring)** — the raw telemetry substrate and alerting/runbook design principles this module specializes for drift specifically.
- **Module 18 (Tracing and Debugging Agentic Systems)** — trace-level debugging that complements this module's aggregate/statistical view once a specific drifted behavior needs root-causing at the individual-request level.
