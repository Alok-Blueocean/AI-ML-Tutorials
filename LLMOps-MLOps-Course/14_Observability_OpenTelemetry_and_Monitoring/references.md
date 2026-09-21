# References — Official Docs, Papers, and Engineering Blogs

## OpenTelemetry — official documentation

### OpenTelemetry Documentation (root)
- **URL:** https://opentelemetry.io/docs/
- **What it teaches:** The full official documentation tree — concepts, language-specific SDKs, Collector configuration, and the semantic conventions index.
- **Difficulty / reading time:** Reference material, not linear reading; budget 30-45 minutes for a first orientation pass through the concepts section.

### Traces (concept page)
- **URL:** https://opentelemetry.io/docs/concepts/signals/traces/
- **What it teaches:** What a span is (name, timestamps, parent reference, trace ID, attributes, events, links, status, span kind) and how spans nest into a trace via context propagation. Confirmed live and current as of this research.
- **Difficulty / reading time:** Beginner; ~15 minutes.

### Inside the LLM Call: GenAI Observability with OpenTelemetry (official blog, 2026)
- **URL:** https://opentelemetry.io/blog/2026/genai-observability/
- **What it teaches:** A 2026 walkthrough of GenAI-specific observability using the OTel semantic conventions — span hierarchies for `invoke_agent` → `chat` → `execute_tool`, the `gen_ai.request.model` / token-usage attributes, GenAI-aware operation-duration metrics, and the explicit privacy default that **no prompt content or tool arguments are captured unless you opt in** — a direct, official confirmation of this module's "never log raw user text by default" rule.
- **Difficulty / reading time:** Intermediate; ~20-25 minutes.

### OpenTelemetry GenAI Semantic Conventions (spec repo)
- **URL:** https://github.com/open-telemetry/semantic-conventions-genai
- **What it teaches:** The living specification for `gen_ai.*` span and event attributes — request/response model names, token usage, finish reasons, and (as a separate, opt-in mechanism) prompt/completion content capture. Still experimental/evolving as of early-to-mid 2026; check the changelog before pinning attribute names in production code.
- **Difficulty / reading time:** Intermediate-advanced; spec-reading, ~30-40 minutes to skim the current attribute tables.

### MLflow: OpenTelemetry GenAI Semantic Conventions
- **URL:** https://mlflow.org/docs/latest/genai/tracing/opentelemetry/genai-semconv/
- **What it teaches:** How a mainstream MLOps tool (MLflow, covered in Module 13 of this course) maps its own tracing UI onto the official OTel GenAI semantic conventions — a good "seeing it applied by a tool you already know" bridge from the abstract spec to a concrete product.
- **Difficulty / reading time:** Intermediate; ~15-20 minutes.

### Datadog: OpenTelemetry Instrumentation for LLM Observability
- **URL:** https://docs.datadoghq.com/llm_observability/instrumentation/otel_instrumentation/
- **What it teaches:** How a major commercial observability vendor ingests OTel GenAI spans into its LLM Observability product — useful as a real-world example of vendor adoption of the (still-experimental) GenAI semantic conventions, and as a comparison point for "build your own OTel pipeline vs. buy a vendor product."
- **Difficulty / reading time:** Intermediate; ~15 minutes.

### Portkey: OpenTelemetry for LLM Observability
- **URL:** https://portkey.ai/docs/product/observability/opentelemetry
- **What it teaches:** How an LLM gateway product exports OTel-compatible traces/metrics for downstream ingestion by any OTel backend — a practical illustration of the "instrument once, export anywhere" value proposition central to why this module teaches OTel rather than a single vendor SDK.
- **Difficulty / reading time:** Beginner-intermediate; ~10-15 minutes.

---

## Prometheus and Grafana — official documentation

### Prometheus: Alerting (practices)
- **URL:** https://prometheus.io/docs/practices/alerting/
- **What it teaches:** The philosophy underneath good alerting rules — alert on symptoms visible to users, not every possible internal cause; keep rules simple; distinguish online-system alerting (latency/error-rate at the highest feasible layer) from batch-job and capacity alerting; and the point that a monitoring pipeline must itself be verified end-to-end (blackbox testing), not just each component in isolation. Confirmed live and current.
- **Difficulty / reading time:** Beginner-intermediate; ~15-20 minutes. Directly underlies this module's alerting-design and on-call-runbook section.

### Prometheus: Alerting Rules (configuration reference)
- **URL:** https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- **What it teaches:** The concrete YAML syntax for alerting rules — `expr`, `for` (pending duration before firing, to avoid flapping alerts on transient spikes), `labels`, and `annotations` used to template the alert message a human on-call actually reads.
- **Difficulty / reading time:** Intermediate; ~15 minutes, best read while writing an actual rule for TTFT/P95/error-rate/cache-hit-rate thresholds.

### vLLM: Metrics (design doc)
- **URL:** https://docs.vllm.ai/en/stable/design/metrics/
- **What it teaches:** The concrete, real Prometheus metric names vLLM exposes at `/metrics`: gauges (`vllm:num_requests_running`, `vllm:kv_cache_usage_perc`), counters (`vllm:prompt_tokens_total`, `vllm:generation_tokens_total`, `vllm:prefix_cache_queries`/`hits`), and — most relevant to this module — histograms for `vllm:time_to_first_token_seconds`, `vllm:inter_token_latency_seconds` (TPOT), `vllm:e2e_request_latency_seconds`, and `vllm:request_queue_time_seconds`. Notes that percentiles must be computed client-side from histogram buckets via `histogram_quantile()`, since Prometheus does not store percentiles natively.
- **Difficulty / reading time:** Intermediate; ~15-20 minutes. This is the single most concrete real-world reference for the module's TTFT/P95/P99/cache-hit-rate metric design section.

### Grafana: Dashboards documentation
- **URL:** https://grafana.com/docs/grafana/latest/dashboards/
- **What it teaches:** Panel types, templated dashboard variables (e.g., filtering by model version or prompt version — directly useful for this module's "version the model/prompt in every log line" pattern), and alerting rules configured from within a dashboard panel.
- **Difficulty / reading time:** Beginner-intermediate; ~20 minutes for the core dashboard-building workflow.

---

## SRE foundations (why the alerting/metrics design looks the way it does)

### Google SRE Book: Monitoring Distributed Systems (Chapter 6)
- **URL:** https://sre.google/sre-book/monitoring-distributed-systems/
- **What it teaches:** The four golden signals (latency, traffic, errors, saturation) and the foundational alerting philosophy — alert only on conditions that are urgent, actionable, and user-visible; prefer symptom-based over cause-based alerting; keep alerting rules simple enough to reason about under pager pressure. Confirmed live, free to read.
- **Difficulty / reading time:** Beginner-intermediate; ~30-40 minutes. This chapter is the intellectual ancestor of both the Prometheus alerting-practices guide above and this module's `matrix_monitor.py` alerting pattern — read it first, the Prometheus doc second, to see the philosophy translated into PromQL practice.

---

## LLM observability platform comparison (for the OTel vs. Langfuse vs. LangSmith table)

### Langfuse: "LangSmith Alternative?" FAQ
- **URL:** https://langfuse.com/faq/all/langsmith-alternative
- **What it teaches:** A structured, vendor-authored (so read with appropriate skepticism, but directionally accurate and cross-checked against independent comparisons) comparison covering licensing (Langfuse MIT open-source vs. LangSmith proprietary), self-hosting (first-class in Langfuse vs. Enterprise-only add-on in LangSmith), OpenTelemetry support (Langfuse's SDKs are built directly on OTel; LangSmith supports OTel ingestion but is deepest with LangChain/LangGraph), pricing model differences, and evaluation feature parity. Notably: Langfuse was acquired by ClickHouse as part of a large Series D round in January 2026, signaling how central LLM observability has become to the broader data-infrastructure stack.
- **Difficulty / reading time:** Beginner-intermediate; ~10 minutes. Use as the primary source for this module's comparison table, cross-checked against at least one independent (non-vendor) comparison article before finalizing exact claims.

### OpenTelemetry for LLMs: A Complete Guide (2026)
- **URL:** https://openobserve.ai/blog/opentelemetry-for-llms/
- **What it teaches:** A vendor-neutral-leaning walkthrough of applying generic OTel instrumentation to LLM services, useful as a second data point alongside the official OTel GenAI blog post above, covering the practical gap between "OTel gives you the plumbing" and "a purpose-built LLM observability platform gives you prompt/response-aware UI on top of that plumbing" — the core distinction this module's comparison table needs to make crisp.
- **Difficulty / reading time:** Intermediate; ~20 minutes.

---

## How these sources map onto the module's backbone

| Module backbone element | Primary reference above |
|---|---|
| Hash PII / never log raw user text | OpenTelemetry official GenAI blog (opt-in-only content capture) |
| Version model/prompt in every log line | Grafana dashboards (templated variables by version label) |
| TTFT, P95/P99 latency | vLLM metrics design doc |
| Cost per query | AI Engineering (Chip Huyen) — see books.md |
| Error rate | Google SRE Book Ch. 6 (golden signals) + Prometheus alerting practices |
| Cache-hit rate | vLLM metrics design doc (`prefix_cache_queries`/`hits`) |
| `matrix_monitor.py` alerting pattern | Prometheus alerting rules reference + Google SRE Book Ch. 6 |
| OTel spans around retrieval/generation/tool calls | OpenTelemetry GenAI semantic conventions repo + OpenLLMetry (github.md) |
| OTel vs. Langfuse vs. LangSmith | Langfuse LangSmith-alternative FAQ + OpenObserve 2026 guide |
