# GitHub Repositories — Observability, OpenTelemetry, and Production Monitoring

Star counts below were pulled directly from the GitHub API at research time (mid/late-2026) and
are real, verified snapshot values — not estimates. They will keep climbing; treat them as an
order-of-magnitude popularity signal, not a fixed fact.

---

## Core OpenTelemetry project

### `open-telemetry/opentelemetry-specification`
- **Stars (verified snapshot):** ~4,300
- **Popularity tier:** Very popular / the canonical spec repo for the whole OTel ecosystem
- **Purpose:** The formal specification all OTel SDKs (Python, Java, Go, .NET, JS, ...) must implement — defines the API/SDK split, the data model for spans/metrics/logs, context propagation semantics, and the sampling model.
- **Relation to this module:** Read this when you need the ground truth for "what is a span actually required to contain" rather than a blog post's paraphrase — useful when your OTel exporter and your Grafana/Tempo backend disagree about how to render a span.

### `open-telemetry/opentelemetry-python`
- **Stars (verified snapshot):** ~2,500
- **Popularity tier:** Very popular / the standard choice for Python services
- **Purpose:** The official Python API + SDK implementation — `TracerProvider`, `MeterProvider`, span processors, exporters (OTLP, console).
- **Relation to this module:** This is what you `pip install` to hand-instrument the retrieval/generation/tool-call spans this module's chapter builds — read the `examples/` directory for span-creation patterns before writing your own.

### `open-telemetry/opentelemetry-python-contrib`
- **Stars (verified snapshot):** ~1,080
- **Popularity tier:** Popular / standard companion repo
- **Purpose:** Community and vendor-contributed auto-instrumentation packages (Flask, FastAPI, requests, gRPC, Django, etc.) that wrap common libraries so they emit spans automatically without manual code changes.
- **Relation to this module:** If your LLM service is a FastAPI app in front of a vector DB client and an LLM SDK, auto-instrumentation from this repo gives you HTTP-layer spans for free — you then add manual spans only around the LLM-specific steps (retrieval, generation, tool calls) that this module cares about.

### `open-telemetry/opentelemetry-collector`
- **Stars (verified snapshot):** ~7,300
- **Popularity tier:** Very popular / the standard telemetry-pipeline gateway
- **Purpose:** The vendor-agnostic Collector binary — receives telemetry (OTLP and other formats), applies processors (batching, filtering, PII scrubbing, sampling), and exports to one or more backends (Prometheus, Tempo, Jaeger, Datadog, etc.) without coupling your application code to a specific vendor.
- **Relation to this module:** The natural place to implement the module's privacy-safe-telemetry rules (hash PII, strip raw user text) as a processor stage that runs once, centrally, instead of re-implementing the scrubbing logic in every service.

### `open-telemetry/semantic-conventions`
- **Stars (verified snapshot):** ~620
- **Popularity tier:** Moderate / niche but authoritative
- **Purpose:** Defines the standard attribute names and span shapes (`http.*`, `db.*`, `messaging.*`, etc.) so that a span from any instrumented library uses consistent field names.
- **Relation to this module:** The general-purpose semantic-conventions repo; note that GenAI-specific conventions have since moved out to their own repo (below) as the GenAI SIG's work matured.

### `open-telemetry/semantic-conventions-genai`
- **Stars (verified snapshot):** ~200 (younger, spun out of the main semantic-conventions repo)
- **Popularity tier:** Emerging / the one to watch for this specific module
- **Purpose:** The dedicated home for the OpenTelemetry GenAI Semantic Conventions — the standard `gen_ai.*` attribute vocabulary (`gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reasons`) and span/event shapes for LLM client calls and agent spans. Still marked experimental as of early 2026, but already adopted by multiple observability vendors.
- **Relation to this module:** This is the single most directly relevant repo for the "instrumenting an LLM service with OpenTelemetry" section — it is the emerging standard for exactly the spans this module teaches you to create around generation and tool calls, and reading its open issues/PRs is the fastest way to see where the spec is still unstable.

### `open-telemetry/opentelemetry-demo`
- **Stars (verified snapshot):** ~3,200
- **Popularity tier:** Popular / the reference demo app for the whole project
- **Purpose:** "Astronomy Shop" — a full microservice e-commerce demo instrumented end-to-end with OpenTelemetry traces, metrics, and logs across ~10+ services in multiple languages, deployable via Docker Compose or Kubernetes/Helm.
- **Relation to this module:** Even though it's not LLM-specific, it's the best available reference for "what does a fully-instrumented, multi-service, trace-correlated system look like end to end" — clone it, run it, and look at how spans propagate across service boundaries before designing your own LLM service's trace tree.

---

## LLM-specific observability (built on OpenTelemetry)

### `traceloop/openllmetry`
- **Stars (verified snapshot):** ~7,300
- **Popularity tier:** Very popular within the LLM-observability niche
- **Purpose:** Open-source instrumentation library that wraps popular LLM SDKs and frameworks (OpenAI, Anthropic, LangChain, LlamaIndex, vector DBs, etc.) to automatically emit OTel-compliant spans — so you get GenAI observability with a couple of lines of setup rather than hand-writing every span.
- **Relation to this module:** The most practical reference implementation of "OTel spans around retrieval/generation/tool calls" this module asks you to build — read its instrumentation source for one SDK (e.g., the OpenAI wrapper) to see exactly which attributes it attaches to a generation span.

### `Arize-ai/openinference`
- **Stars (verified snapshot):** ~1,100
- **Popularity tier:** Popular / a second credible OTel-based LLM instrumentation standard
- **Purpose:** Arize's own OpenTelemetry-based instrumentation spec + libraries for AI observability, designed to interoperate with their Phoenix observability tool but usable standalone with any OTel backend.
- **Relation to this module:** Useful as a point of comparison against OpenLLMetry and the official GenAI semantic conventions — three different (converging but not identical) attempts to standardize the same span shapes is itself an instructive lesson about how young this part of the ecosystem still is in 2026.

### `Arize-ai/phoenix`
- **Stars (verified snapshot):** ~10,800
- **Popularity tier:** Very popular / a leading open-source LLM observability + eval UI
- **Purpose:** Self-hostable UI for tracing, evaluating, and debugging LLM/agent applications, built on the OpenInference instrumentation above — trace visualization, prompt/response inspection, and eval-score overlays.
- **Relation to this module:** A concrete "what does an LLM-trace UI look like" reference to compare against Grafana+Tempo, Langfuse, and LangSmith when you get to the comparison-table section of this module.

### `langfuse/langfuse`
- **Stars (verified snapshot):** ~32,000
- **Popularity tier:** Very popular / one of the two dominant dedicated LLM-observability platforms (with LangSmith)
- **Purpose:** Open-source (MIT) LLM engineering platform — tracing, evaluation (LLM-as-judge and code-based), prompt management, datasets, and a playground; self-hosting is a first-class deployment mode, and its own SDKs are built on OpenTelemetry so any OTel-emitting service can feed it.
- **Relation to this module:** The open-source, framework-agnostic side of this module's OTel-vs-Langfuse-vs-LangSmith comparison table — worth running locally (via its docker-compose) to see what a purpose-built LLM trace UI adds on top of raw OTel spans (prompt/response pairing, cost rollups, eval scores attached to traces).

### `SigNoz/signoz`
- **Stars (verified snapshot):** ~31,700
- **Popularity tier:** Very popular / a leading open-source general observability platform, OTel-native
- **Purpose:** Self-hostable, OpenTelemetry-native APM — unified traces, metrics, and logs (an open-source alternative to a commercial "LGTM"-style or Datadog-style stack), with dashboards and alerting built in.
- **Relation to this module:** A good reference architecture if you want a single self-hosted backend for all three OTel signals rather than assembling Prometheus + Tempo + Loki + Grafana yourself — useful contrast case when this module discusses build-vs-buy tradeoffs for the observability backend.

---

## Metrics backend and dashboards (Prometheus / Grafana stack)

### `prometheus/prometheus`
- **Stars (verified snapshot):** ~65,000
- **Popularity tier:** Very popular / the de facto standard metrics database for cloud-native systems
- **Purpose:** The Prometheus server itself — pull-based metrics scraping, PromQL query engine, and the alerting-rule evaluation engine that feeds Alertmanager.
- **Relation to this module:** The backend this module's Prometheus-metrics-design section targets directly — read the docs on histogram bucket design before deciding your TTFT/latency bucket boundaries, since bad bucket choices silently make `histogram_quantile()` inaccurate.

### `prometheus/alertmanager`
- **Stars (verified snapshot):** ~8,600
- **Popularity tier:** Popular / the standard companion to Prometheus for alert routing
- **Purpose:** Deduplicates, groups, and routes firing alerts to notification channels (PagerDuty, Slack, email, etc.), with silencing and inhibition rules.
- **Relation to this module:** The piece that turns a PromQL alerting rule into an actual page — directly relevant to the module's on-call runbook and alerting-design section (grouping related alerts so one incident doesn't fire 20 separate pages).

### `grafana/grafana`
- **Stars (verified snapshot):** ~75,900
- **Popularity tier:** Very popular / the de facto standard dashboarding tool across the observability industry
- **Purpose:** The dashboarding and visualization layer over Prometheus, Loki, Tempo, and dozens of other data sources; supports templated variables, alerting, and panel-level drill-downs.
- **Relation to this module:** The tool this module's "designing Grafana dashboards for LLM services" section targets — study its official LLM/AI-observability community dashboards (searchable in-app under Dashboards → community) as starting templates rather than building panels from scratch.

### `grafana/tempo`
- **Stars (verified snapshot):** ~5,400
- **Popularity tier:** Popular / Grafana's own distributed-tracing backend
- **Purpose:** A high-volume, low-cost distributed tracing backend designed to ingest OTLP traces directly and correlate with Loki logs and Prometheus/Mimir metrics inside Grafana.
- **Relation to this module:** One practical path to close the loop this module teaches (traces → metrics → logs correlation) entirely within the Grafana ecosystem, as an alternative to Jaeger or a vendor SaaS.

### `grafana/loki`
- **Stars (verified snapshot):** ~28,600
- **Popularity tier:** Very popular / the standard log-aggregation companion to Prometheus/Grafana
- **Purpose:** "Like Prometheus, but for logs" — indexes only metadata/labels (not full text) for cost-efficient log storage and querying via LogQL.
- **Relation to this module:** Where the module's structured `telemetry.json` log lines would land in a Grafana-native stack; its label-based indexing model is a direct parallel to Prometheus's cardinality-discipline lesson.

---

## Reference LLM-serving stack with built-in metrics

### `vllm-project/vllm`
- **Stars (verified snapshot):** ~87,700
- **Popularity tier:** Extremely popular / the leading open-source high-throughput LLM inference engine
- **Purpose:** A production-grade LLM serving engine; exposes a native Prometheus `/metrics` endpoint out of the box with histograms for time-to-first-token, inter-token latency, end-to-end request latency, queue time, and gauges/counters for KV-cache utilization and prefix-cache hit rate.
- **Relation to this module:** The best concrete, real-world example of exactly the metric set this module's backbone asks for (TTFT, P95/P99 latency, cache-hit rate) already implemented as real Prometheus metrics with sane histogram buckets — read its `design/metrics` documentation as a template even if you're calling a hosted API (OpenAI/Anthropic) rather than self-hosting, since the metric *names and shapes* it uses have become a de facto convention other serving stacks (TGI, etc.) converge toward.
