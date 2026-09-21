# Module 14 — Example Projects: Observability, OpenTelemetry, and Production Monitoring

Three projects, increasing in scope and production-realism, all built on this module's material.
Build them in order — the Medium project's Collector-fronted, fully-traced agent is the exact
workload the Production project scales to multiple replicas, layers a sampling policy onto, and
wraps in a full three-dashboard, blackbox-tested alerting stack. None of these require a paid LLM API
or GPU access: a mocked model call (randomized sleep + canned tokens) is a legitimate stand-in
throughout, because the point of every project is the observability wiring around the call, not the
call's own quality.

---

## Mini — Single-Endpoint LLM Service with Traces and Core Metrics

**Scope:** Wrap one LLM call (real or mocked) in a FastAPI `/chat` endpoint and instrument it with the
minimum viable observability stack: one traced request, privacy-safe telemetry, and the core
Prometheus metrics — enough to answer "is this one request fine" and "is the fleet fine right now,"
but with no dashboard beyond the essentials and no alerting yet.

**What to build:**
- A FastAPI `/chat` endpoint wrapping a single LLM call, with `FastAPIInstrumentor` auto-instrumenting
  the HTTP layer and one nested `chat` span carrying `gen_ai.request.model` and
  `gen_ai.usage.{input,output}_tokens` attributes (`tutorial.md` §3.2).
- Privacy-safe telemetry per §3.1: a hashed `user_id_hash` (SHA-256 + pepper), `query_len_chars`
  instead of raw query text, and `model_version`/`prompt_version` on every record — emitted as an OTel
  log correlated to the request's `trace_id`, never as a disconnected third logging system.
- OTLP export to a local Jaeger (or Tempo) instance — traces should be inspectable in a real trace UI,
  not just printed to console.
- The core Prometheus metric set from §3.4: `TTFT_SECONDS` and `E2E_LATENCY_SECONDS` histograms,
  `REQUESTS_TOTAL` (labeled by `status`) and `COST_USD_TOTAL` counters, scraped by a local Prometheus.
- One minimal Grafana dashboard with at least 4 panels: P95/P99 TTFT, P95/P99 E2E latency, error rate,
  and cost accumulated over the last hour.

**What it demonstrates:**
- The three-signal mapping from `tutorial.md` §2 ("what happened on this request" / "is the fleet
  healthy" / "why did this go wrong") applied to the simplest possible service shape.
- Correct metric-type selection (histogram for anything you'll ever want a percentile on, counter for
  anything monotonic) from day one — the choice §3.4 calls not retroactively fixable.
- Privacy-by-default logging as the *first* thing built, not a retrofit — no raw query text anywhere
  in the trace or log storage, verified by your own inspection, not assumed.

**Definition of done:** hitting `/chat` 20 times produces 20 traces visible in Jaeger, each with a
correctly nested `handle_request` → `chat` span pair and no raw query content anywhere in the trace or
log system (verified by grepping your Jaeger/log storage for a distinctive substring of a query you
actually sent); the Grafana dashboard shows non-zero, plausible values on all 4 panels after those 20
requests; you can point at the P95 panel's query and show it is a `histogram_quantile()` call, not an
average, and explain in one sentence why that distinction will matter later.

---

## Medium — RAG/Agentic Service with Full Span Hierarchy, Live-Ops Dashboard, and Routed Alerting

**Scope:** Extend the Mini project into a multi-stage agentic pipeline — retrieval, reranking,
generation, and a tool call — instrumented through a real OpenTelemetry Collector (not direct export),
with the complete metric set including cache-hit rate, the full six-panel live-ops dashboard from
`tutorial.md` §3.5, and Prometheus alerting rules routed through Alertmanager to genuinely distinct
channels with a runbook behind every one.

**What to build:**
- An OTel Collector as a separate hop between your app and your backends (receivers: OTLP; processors:
  batch; exporters: to your trace backend and to Prometheus) — per §3.2's architecture, this is the
  point where PII-scrubbing, sampling, or backend fan-out live, not application code.
- The full span hierarchy from §3.3: `retrieve_context` → `rerank` → `chat` → (conditionally)
  `execute_tool: <name>` → a follow-up `chat`, with span attributes/events matching the tutorial's
  agentic-loop example (`tool_call_requested` as a span event, `gen_ai.tool.name` on the tool span).
- A cache layer (even a trivial in-memory dict keyed by a normalized query) instrumented with
  `CACHE_LOOKUPS_TOTAL{result="hit"|"miss"}`, exercised by synthetic load with enough repeated queries
  to produce a real, nonzero, measured cache-hit rate — not a hardcoded panel value.
- The full six-panel live-ops Grafana dashboard from §3.5's wireframe, with thresholds set to specific
  documented numbers.
- At least 3 Prometheus alerting rules (e.g., `HighTTFT`, `ErrorRateSpike`, and either
  `HighFallbackUsage` or `LowCacheHitRate`) with distinct `severity` labels, correct `for` windows, and
  Alertmanager routing to at least two different receivers (real or mocked) — proving differential
  routing works, not just that two labels exist.
- One runbook entry per alert, in the format shown in `tutorial.md` §3.6.

**What it demonstrates:**
- The Collector-as-decoupling-point pattern (§3.2) — you can point to exactly where sampling or
  PII-scrubbing configuration would go without touching application code.
- The trace-waterfall diagnostic skill from §3.3: given any one of your captured traces, you can state
  which stage dominated total latency without reading the underlying code.
- The dashboard-and-alert-threshold agreement principle from §3.5 — your live-ops panel colors and your
  alert `expr` thresholds are the same numbers, read from one place.
- Symptom-based, severity-differentiated alerting (§3.6) instead of one generic catch-all alert.

**Definition of done:** at least one captured trace where you can state, without reading code, which
stage dominated total latency, cross-checked against how you actually built that request to behave;
the cache-hit-rate panel shows a measured value driven by real repeated synthetic queries, not a
constant; a deliberately injected sustained error burst fires `ErrorRateSpike` and reaches its
critical-severity receiver, while a deliberately injected sustained cache-hit-rate drop fires only the
info-severity receiver — demonstrated, not assumed from the YAML; every alert that can fire has a
runbook file next to it.

---

## Production-Grade — Multi-Replica Observability Platform with Sampling Strategy, Three Dashboards, and a Blackbox-Verified Alert Pipeline

**Scope:** Scale the Medium project's service to multiple replicas behind a shared Collector, add an
explicit, documented sampling policy, build all three dashboards from `tutorial.md` §3.5's
audience/time-horizon table (live-ops, weekly/trend, per-deploy comparison), add a content-quality
signal that catches the class of failure Exercise 7 diagnoses, add a written retention/encryption
policy, blackbox-test the entire alert-to-page pipeline with a recorded timing, and produce a
defensible, project-specific OTel-vs-Langfuse-vs-LangSmith decision document.

**What to build:**
- 2-3 replicas of the Medium project's service (behind a simple reverse proxy, or a real Kubernetes
  Deployment if you want to reuse Module 06 skills), all exporting to one shared Collector — proving
  the Collector-as-single-ingest-point pattern actually holds under more than one instance.
- An explicit sampling policy configured in the Collector: either a probabilistic sampler at a
  documented rate, or (if your Collector build supports it) a `tail_sampling` processor configured to
  always retain error/slow traces while sampling down "boring" successful ones — with a short written
  justification for whichever you chose, referencing the cost/scaling tradeoff in `tutorial.md` §6.
- All three dashboards from §3.5's table:
  - **Live-ops** (reused/extended from the Medium project).
  - **Weekly/trend**, with cost-per-query trend, cache-hit-rate trend, error-rate trend, request
    volume, and model-version rollout progress, populated over several days to weeks of synthetic
    data with at least one realistic dip-and-recovery.
  - **Per-deploy comparison**, showing old-vs-new `model_version` side by side after you simulate a
    version rollout (e.g., route 10% of synthetic traffic to a "v2" model_version label for a period).
- A content-quality signal addressing Common Mistake #10 (`tutorial.md` §5): a sampled, asynchronous
  check on a fraction of responses (a rule-based or keyword-based stand-in for a real LLM-as-judge
  check is acceptable) exposed as its own Prometheus metric, with its own alert — directly answering
  Exercise 7's diagnosis with a working implementation.
- The full 6-alert set from §3.6's table (TTFT degradation, error spike, latency spike, high fallback
  usage, low cache-hit rate, cost budget — or your own well-justified equivalents), routed through
  Alertmanager to at least 3 differentiated channels (warning/Slack-equivalent, critical/page-
  equivalent, info/weekly-digest-equivalent), each with a runbook.
- A periodic blackbox synthetic test (a small script run on a schedule) implementing the "monitoring
  the monitors" guidance from §6 — it injects a known synthetic failure and verifies the alert fires
  and is actually received end to end, logging a pass/fail result and a timing.
- A short written retention/encryption policy: retention window, encryption-at-rest approach (even if
  simulated locally rather than wired to real cloud KMS), and what you'd need to produce if a regulator
  or auditor asked to see your data-handling policy for this service.
- A one-to-two-page OTel-vs-Langfuse-vs-LangSmith decision document for this specific project, using
  the comparison structure from §3.7, naming at least two tradeoffs specific to your own build (not
  generic tutorial prose) that drove your choice.

**What it demonstrates:**
- The complete senior-level observability ownership loop: a sampling strategy reasoned about under
  realistic multi-replica volume, dashboards matched to distinct audiences and time horizons rather
  than one overloaded screen, content-correctness monitoring that goes beyond infrastructure metrics,
  a genuinely-tested (not merely configured) alerting pipeline, privacy/retention treated as a designed
  and documented control from the start, and a defensible build-vs-buy call.
- That "all green dashboards" cannot mask a content-quality failure in your system the way it did in
  the Exercise 7 scenario — your content-quality alert is proof, not a hypothetical.
- That your alerting pipeline has been *proven* to work end to end with a timestamp, not just trusted
  because the YAML looks correct.

**Definition of done:** all 3 dashboards are populated with real or realistic synthetic data and
cross-checked against each other (e.g., the weekly dashboard's daily cost aggregation matches the sum
of the live-ops dashboard's hourly cost panel over the same days); your Collector's sampling
configuration is checked into your project with written reasoning for what is sampled at 100% versus
probabilistically; the content-quality alert has fired at least once against a deliberately-planted
stale/incorrect response in your synthetic data, and its runbook was followed to a documented
resolution; your blackbox synthetic test has at least one logged successful run showing the full path
from injected failure to received page, with a recorded timestamp delta; your decision document names
at least two tradeoffs specific to your own project's constraints (traffic volume, team size, existing
stack, data-residency needs) for why you chose your particular combination of OTel, Langfuse, and/or
LangSmith, rather than restating the tutorial's comparison table generically.
