# Module 18 — References: Tracing and Debugging Agentic Systems

Curated from well-established, real tools and specifications. Each entry notes specifically what it teaches relative to this module's content, so you know why it's here rather than just that it exists.

---

## Tracing Tools and Their Documentation

- **Arize Phoenix — official docs (Arize AI).** The primary tool this module centers on. Covers: launching a local/notebook Phoenix session, the OpenInference instrumentation packages for OpenAI/Anthropic/LangChain/LlamaIndex/etc., the trace waterfall UI, the Python client for pulling spans/traces as DataFrames, and Phoenix's built-in LLM-judge evaluators for retrieval relevance and response groundedness. Read this to go from this module's code snippets to a real, running instance with your own pipeline.
- **OpenInference specification (open-source, used by Phoenix and others).** The semantic-convention spec defining span kinds (`RETRIEVER`, `RERANKER`, `LLM`, `TOOL`, `CHAIN`, `AGENT`) and attribute keys used throughout §3.3-3.4. Read this to understand exactly what attribute names to use so your custom spans are compatible with Phoenix's (and other OpenInference-consuming tools') built-in views, rather than inventing your own ad hoc naming.
- **LangSmith — official docs (LangChain, Inc.).** Covers LangChain/LangGraph-native tracing, the `@traceable` decorator for framework-agnostic instrumentation, dataset/experiment tracking, and its evaluator framework. Read this if your pipeline is built on LangChain/LangGraph and you want the deepest native integration option compared in §3.5.
- **Langfuse — official docs (Langfuse GmbH, open-source).** Covers the `@observe` decorator, OpenTelemetry-compatible SDK usage, self-hosted Docker Compose deployment, prompt management/versioning, and its scoring/evaluation pipelines. Read this for the fully open-source, framework-agnostic, self-hostable alternative discussed in §3.5.

## OpenTelemetry and Semantic Conventions

- **OpenTelemetry official documentation (CNCF project).** The foundational tracing/metrics/logs standard this whole module's instrumentation code builds on (traces, spans, span processors, exporters, context propagation). Read this if any of the `TracerProvider`/`start_as_current_span` code in §3.3 is unfamiliar — it is the general-purpose layer beneath all the LLM-specific tooling.
- **OpenTelemetry GenAI Semantic Conventions (part of the OTel specification).** Defines the vendor-neutral `gen_ai.*` attribute namespace (`gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, etc.) referenced alongside OpenInference in §3.3. Read this to understand the more general, cross-vendor convention that OpenInference's LLM-application-specific conventions sit alongside.
- **W3C Trace Context specification.** The underlying standard for how a trace ID and span ID propagate across process/service boundaries via HTTP headers. Read this if you are debugging a case where spans from a distributed agent (e.g., a tool call made to a separate microservice) are not appearing correctly nested in the same trace.

## Evaluation and Judge-Scoring Tools (Support the Filtering/Clustering Step)

- **Arize Phoenix's evaluation library.** Built-in evaluators for retrieval relevance, hallucination/groundedness detection, and QA correctness, designed specifically to produce the `judge_score` column §3.6's clustering code filters on. Read this to avoid hand-rolling an LLM-judge prompt from scratch when a maintained one already exists for this exact use case.
- **Ragas (open-source RAG evaluation library).** A widely-used framework specifically for RAG-pipeline metrics — faithfulness (groundedness), answer relevance, context precision/recall. Read this as a complementary/alternative source of the groundedness-check logic sketched in §3.2 and §3.4, purpose-built rather than hand-written.
- **This course's own Module 09-11 (Prompt Lifecycle & Statistical Evaluation, Building Evaluation Datasets, LLM-as-Judge Design and Limits).** Directly prerequisite material for the `judge_score` this module's clustering pipeline assumes already exists — read those modules first if you have not, since this module does not re-teach how to build a reliable LLM judge.

## Related Observability Tooling (General ML/Data Monitoring, Referenced for Context)

These are not agent-tracing tools specifically, but are well-established observability/monitoring tools whose concepts (drift detection, data quality validation, statistical monitoring) are the natural next layer once tracing is in place — most directly relevant to this course's Module 19 (Drift Detection and Retraining), included here because root-cause clusters found via this module's methodology (§3.6) are exactly the kind of signal that should feed into that layer.

- **Evidently AI — official docs.** Open-source library for data drift, prediction drift, and data quality reports/dashboards; relevant to turning a recurring failure cluster (e.g., a slowly degrading retrieval-score distribution) into a monitored, alertable drift metric over time.
- **whylogs / WhyLabs — official docs.** A lightweight, privacy-conscious statistical profiling library (whylogs) plus a hosted monitoring platform (WhyLabs) for tracking distributional statistics of model inputs/outputs over time at scale — relevant to the "export for dashboards/trend analysis" capability mentioned in §3.4/§3.6.
- **NannyML — official docs.** Open-source library focused specifically on estimating performance degradation (including in the absence of ground-truth labels) — conceptually relevant to detecting that a retrieval-quality or groundedness metric is trending worse before it becomes a full-blown incident.
- **Great Expectations — official docs.** A widely-used data-quality/validation framework; conceptually the same "fail small, validate before you proceed" philosophy this module's §3.2 guardrail pattern applies at the pipeline-step level, but applied instead to structured data pipelines.
- **Deepchecks — official docs.** An open-source suite of validation checks for data and model quality, including LLM-specific evaluation suites; another concrete implementation of the "validate systematically, don't eyeball it" philosophy this module teaches for agent traces.

## Experiment/Run Tracking (Referenced for Continuity with Earlier Modules)

- **MLflow — official docs, specifically MLflow Tracing.** MLflow has added LLM/GenAI tracing capabilities (span-based tracing compatible with OpenTelemetry concepts) alongside its long-standing experiment tracking. Read this if your team already has MLflow as its experiment-tracking backbone (this course's Module 13) and wants to evaluate consolidating trace storage there rather than adopting a separate dedicated tool.

## How to Use This List

Do not attempt to read every reference before starting the exercises. A practical order: (1) OpenTelemetry docs enough to understand spans/tracers, (2) Arize Phoenix docs enough to get Exercise 3 running, (3) OpenInference spec once you're writing custom spans in Exercise 4, (4) Ragas or Phoenix's evaluation docs once you reach the groundedness-check and clustering exercises (5, 6), (5) LangSmith/Langfuse docs only once you reach Exercise 7's hands-on comparison.
