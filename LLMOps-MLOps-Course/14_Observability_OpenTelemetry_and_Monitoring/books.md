# Books — Observability, OpenTelemetry, and Production Monitoring

## 1. Learning OpenTelemetry
**Austin Parker, Ted Young, Juraci Paixão Kröhling — O'Reilly, 2024**

- **What it teaches:** Written by OpenTelemetry governance-committee members and maintainers, this is the closest thing to an authoritative OTel textbook: the signal model (traces, metrics, logs), the Collector's pipeline architecture (receivers → processors → exporters), context propagation across process boundaries, sampling strategies (head-based vs. tail-based), and how to design semantic conventions for a new domain — which is exactly what the GenAI Semantic Conventions SIG has been doing for LLM spans since 2024.
- **Difficulty:** Intermediate
- **Estimated reading time:** ~7-9 hours cover-to-cover; the chapters on the Collector and on semantic conventions (2-3 hours) are the highest-value subset for this module.
- **Why it matters for this module:** This module asks you to instrument an LLM service with spans around retrieval/generation/tool calls — you cannot design that instrumentation well without understanding *why* OTel separates the API from the SDK, why the Collector exists as a separate hop instead of exporting straight to a backend, and how sampling decisions affect whether your P99 latency traces actually survive to be queried later. This book is the primary mechanics reference underneath the GenAI-specific material.

---

## 2. Observability Engineering: Achieving Production Excellence
**Charity Majors, Liz Fong-Jones, George Miranda — O'Reilly, 2022**

- **What it teaches:** The conceptual case for observability over traditional monitoring: why dashboards built from a fixed set of pre-aggregated metrics can't answer novel questions about production behavior, why high-cardinality, high-dimensionality structured events (not just counters) are what let you debug "why is *this* request slow" instead of only "is the system slow on average," and how to build a culture of debugging in production with real telemetry instead of only in staging with logs.
- **Difficulty:** Intermediate
- **Estimated reading time:** ~8-10 hours; Part I (the observability vs. monitoring distinction) and the chapters on structured events and SLOs are the most directly relevant (~3 hours).
- **Why it matters for this module:** This is the conceptual justification for why the module's `telemetry.json` pattern logs structured, high-cardinality fields (model version, prompt version, cache hit, cost) per request rather than only exporting five aggregate counters — it is the difference between "the error rate went up" and "these specific requests, on this specific prompt version, against this specific retrieval index, failed."

---

## 3. Designing Machine Learning Systems
**Chip Huyen — O'Reilly, 2022**

- **What it teaches:** Chapter 8, "Data Distribution Shifts and Monitoring," lays out ML-specific monitoring on top of standard software monitoring: software-level metrics (latency, throughput, errors) vs. ML-specific metrics (accuracy proxies, prediction distributions, feature distributions), the different types of distribution shift, and why "observability" for ML systems has to include model behavior, not just service health.
- **Difficulty:** Beginner → Intermediate
- **Estimated reading time:** Chapter 8 alone is ~45-60 minutes; useful to have read the whole book from earlier modules in this course, but not required for this module specifically.
- **Why it matters for this module:** Frames the split this module cares about — infrastructure-level signals (TTFT, P95/P99 latency, error rate, cache-hit rate) that OpenTelemetry and Prometheus are built for, versus behavior-level signals (is the model's output quality drifting) that need separate evaluation pipelines (see the Module 11/12 material on LLM-as-judge and quality gates). This module is squarely about the infrastructure-level half of that split, and this chapter explains why that half is necessary but not sufficient.

---

## 4. AI Engineering: Building Applications with Foundation Models
**Chip Huyen — O'Reilly, 2025**

- **What it teaches:** A 2025-era treatment of building production LLM applications, including a dedicated treatment of evaluation, cost, and latency trade-offs specific to foundation-model APIs (as opposed to self-hosted classical ML models) — token-based cost accounting, streaming latency (TTFT vs. total generation time), and the operational realities of calling third-party model APIs at scale.
- **Difficulty:** Intermediate
- **Estimated reading time:** The chapters on cost/latency and on production monitoring for LLM applications are ~2-3 hours combined; full book is substantially longer.
- **Why it matters for this module:** This is the most current book-length source that treats "cost per query" and "TTFT" as first-class production metrics for LLM services specifically (rather than importing them by analogy from classical web-service SRE) — directly backing the module's inference-metrics backbone.

---

## 5. Site Reliability Engineering: How Google Runs Production Systems
**Betsy Beyer, Chris Jones, Jennifer Petoff, Niall Richard Murphy (eds.) — O'Reilly, 2016 (free online at sre.google/books)**

- **What it teaches:** Chapter 6, "Monitoring Distributed Systems," defines the four golden signals (latency, traffic, errors, saturation) and the symptom-based alerting philosophy that essentially all modern alerting design — including Prometheus's own documentation — is built on: alert on user-visible symptoms, keep alerting rules simple, and make sure every page warrants a human response.
- **Difficulty:** Beginner → Intermediate
- **Estimated reading time:** Chapter 6 is ~30-40 minutes; free to read at https://sre.google/sre-book/monitoring-distributed-systems/.
- **Why it matters for this module:** This is the origin of the alerting philosophy underneath the module's `matrix_monitor.py` pattern and the on-call runbook design — the four golden signals map almost directly onto the LLM-service metrics this module tracks (latency → TTFT/P95/P99, traffic → QPS, errors → error rate, saturation → GPU/queue utilization and cache pressure).

---

## 6. Prometheus: Up & Running, 2nd Edition
**Brian Brazil — O'Reilly, 2023**

- **What it teaches:** Written by a Prometheus core developer: the data model (metric types — counter, gauge, histogram, summary), PromQL query language, instrumentation best practices (label cardinality discipline, naming conventions), Alertmanager routing/grouping/inhibition, and long-term storage/federation patterns.
- **Difficulty:** Intermediate
- **Estimated reading time:** ~6-8 hours full book; the chapters on histograms and on alerting rules (~2 hours) are the highest-value subset here.
- **Why it matters for this module:** Directly underneath the "designing Prometheus metrics and Grafana dashboards for LLM services" part of this module's scope — in particular, understanding why TTFT and per-token latency must be histograms (not gauges or summaries) is what makes P95/P99 computation via `histogram_quantile()` correct, and understanding label cardinality is what stops a naive "one label per user_id" mistake from taking down your Prometheus server.
