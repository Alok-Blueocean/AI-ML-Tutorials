# Module 14 — Observability, OpenTelemetry, and Production Monitoring

> "You cannot debug what you cannot see, and you must not see what you have no right to look at." — the two halves of this module in one sentence.

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Explain why LLM observability is a *harder* problem than classical service observability, and map the three OpenTelemetry signal types (traces, metrics, logs) onto an LLM inference pipeline.
2. Design a privacy-safe telemetry schema for an LLM service — hashing identifiers, versioning models/prompts, and defaulting to metadata-only logging unless raw-content capture is explicitly authorized by policy.
3. Instrument a FastAPI-based LLM service with OpenTelemetry: create a trace with nested spans for retrieval, generation, and tool calls, and attach `gen_ai.*` semantic-convention attributes.
4. Design a Prometheus metrics schema for LLM serving — TTFT, inter-token latency (TPOT), end-to-end latency, cost per query, error rate, cache-hit rate — using the correct metric type (histogram vs. counter vs. gauge) for each.
5. Build a Grafana dashboard layout that surfaces the "few signals that reveal user experience, cost, and reliability" rather than a wall of noise.
6. Write alerting rules with correct thresholds, `for` durations, severities, and routing, and produce an on-call runbook entry for each alert.
7. Compare OpenTelemetry, Langfuse, and LangSmith for LLM observability specifically, and defend a build-vs-buy decision with named tradeoffs.
8. Identify the most common observability mistakes in production LLM systems (log-everything anti-pattern, alert fatigue, missing trace propagation, PII leakage through "helpful" debug logs) and correct them.

### Prerequisites

- This course's Modules 01-06 (CI/CD foundations, versioning/registries/rollback, reproducibility, Docker, Kubernetes) — this module assumes you can already containerize and deploy a service and focuses purely on *seeing inside it* once deployed.
- Module 08 (LLM Latency, Cost, and Deployment) — familiarity with TTFT, TPOT, and the token-cost model this module measures.
- Comfortable reading Python, YAML, and PromQL-adjacent query syntax (no prior Prometheus/Grafana experience required — it's taught here).
- A mental model of client-server tracing (even informally: "a request enters, does some work, and leaves — I want to see the timeline of that work").

### Key Terminology

| Term | One-line definition |
|---|---|
| **Observability** | The property of a system that lets you answer *new, previously unasked* questions about its internal state using only its external outputs (traces, metrics, logs) — as opposed to *monitoring*, which only answers questions you anticipated in advance. |
| **Trace** | The end-to-end record of one request's journey through a (possibly distributed) system, made of one or more spans linked by a shared trace ID. |
| **Span** | A single named, timed unit of work within a trace (e.g., "retrieve_context", "call_llm", "invoke_tool"), with a start time, duration, parent span, and key-value attributes. |
| **Metric** | A numeric measurement aggregated over time — counters (only go up), gauges (go up or down), and histograms (distribution of values, enabling percentile queries). |
| **Log** | A discrete, timestamped event record — the least structured but often most detail-rich of the three signals. |
| **OpenTelemetry (OTel)** | The CNCF-graduated, vendor-neutral standard (API + SDK + Collector + semantic conventions) for producing and exporting traces, metrics, and logs — "instrument once, export anywhere." |
| **Semantic conventions** | A standardized vocabulary of attribute names (e.g., `gen_ai.request.model`, `gen_ai.usage.input_tokens`) so that a span produced by any tool/vendor means the same thing to any backend. |
| **TTFT** | Time To First Token — wall-clock time from request receipt to the first streamed output token; the dominant perceived-latency metric for chat-style LLM UX. |
| **TPOT / ITL** | Time Per Output Token / Inter-Token Latency — average time between successive streamed tokens after the first one; governs how "smooth" a streaming response feels. |
| **P95 / P99 latency** | The latency value below which 95%/99% of requests fall — tail-latency measures, because averages hide the worst user experiences. |
| **Cache-hit rate** | The fraction of requests served (fully or partially, e.g., via KV-cache prefix reuse) from cache rather than requiring fresh computation — directly trades off against cost and latency. |
| **PII (Personally Identifiable Information)** | Any data that could identify a specific individual — names, emails, raw free-text queries, session content — which production telemetry must protect by default. |
| **Alert fatigue** | The failure mode where too many low-value or non-actionable alerts train on-call engineers to ignore all alerts, including the real ones. |
| **Runbook** | A written, step-by-step procedure an on-call engineer follows when a specific alert fires — the difference between "a page went off" and "someone knows what to do about it." |

---

## 2. Why This Topic Matters — Where It Fits in the MLOps/LLMOps Lifecycle

Every module before this one has been about getting something *correct and shippable*: a versioned model (Module 03), a reproducible environment (Module 04), a container (Module 05), an orchestrated deployment (Module 06), a well-designed prompt (Module 07), a latency/cost-aware serving plan (Module 08), a statistically sound evaluation (Module 09-11), and a dashboard-gated release process (Module 12) tracked in an experiment registry (Module 13). This module is about the next question, the one that never goes away: **once it's live, how do you know what it's actually doing, right now, for real users, and how do you find out *fast* when it stops doing the right thing?**

```
 Data -> Training -> Registry -> Container -> Kubernetes -> Live traffic
                                                                  |
                                                                  v
                                              +---------------------------------------+
                                              |         THIS MODULE (Module 14)        |
                                              |  Traces / Metrics / Logs / Alerts       |
                                              |  "What is happening, right now, safely" |
                                              +---------------------------------------+
                                                                  |
                                            feeds back into ----> |
                                                                  v
                                     Module 12 dashboards & gates, Module 13 registry
                                     (regressions detected here trigger rollback there)
```

Two things make this materially harder for LLM systems than for the classical ML services this course started with, and both are why this module exists as a full chapter rather than a paragraph in Module 06:

**1. The failure surface is richer and quieter.** A classical model either returns a prediction or throws an HTTP 500. An LLM can return HTTP 200 with a fluent, confident, completely wrong or unsafe answer — no exception is thrown, no metric obviously moves, and yet the system has failed. Observability for LLM systems therefore has to reach further than "is it up and is it fast" into "is the *content* behaving," which is why this module connects directly back to Module 11 (LLM-as-Judge) — judges are often run as an asynchronous observability signal on sampled production traffic, not only offline.

**2. The privacy stakes are structurally higher.** A recommendation model's input is a user ID and some behavioral features. An LLM's input is, quite often, literally what the user typed — which can contain anything: names, health information, financial account numbers, or content the user never intended to have persisted anywhere. A logging system that "just logs the request and response for debugging," a completely normal instinct for a classical service, is a live compliance and security incident waiting to happen for an LLM service. This is the reason the source material for this module leads with privacy-safe telemetry *before* it leads with metrics: in LLMOps, the privacy design is not a bolt-on to observability, it *is* the observability design.

Framed as the three questions every production AI team must be able to answer at any moment:

| Question | Signal that answers it | Covered in |
|---|---|---|
| "What exactly happened for this one request?" | Traces (spans) | §3.2, §3.3 |
| "Is the system, in aggregate, healthy right now?" | Metrics | §3.4, §3.5 |
| "Why did this specific thing go wrong?" | Logs (privacy-safe) | §3.1 |
| "Who needs to act, and what should they do?" | Alerts + runbooks | §3.6 |
| "Should I even be looking at this content?" | Privacy/retention policy | §3.1 (applies to all of the above) |

A senior MLOps/LLMOps engineer is expected to own this whole stack end to end — not just "add some print statements" but design the schema, choose the signal type, wire the export pipeline, define the alert threshold, and write the runbook a 2 a.m. on-call engineer with no context can follow.

---

## 3. Main Concepts

### 3.1 Privacy-Safe Telemetry Design

#### Theory

The core tension this section resolves: **you need enough detail to debug a production incident**, but **you must not create a system that leaks personal data or becomes a second, ungoverned copy of your sensitive user data.** Naively "logging everything" solves the first problem and directly causes the second. The resolution is not "log less" — it's "log different": replace raw content with structured, privacy-safe metadata that is *just as useful for debugging* as the raw content would have been, in the overwhelming majority of cases.

The governing rules, straight from how production teams actually run this (and consistent with the OpenTelemetry GenAI Semantic Conventions' own privacy-by-default stance — see references.md):

1. **Hash PII, don't store it.** User/session identifiers get passed through a one-way hash (SHA-256, ideally with a server-side pepper/salt so the hash cannot be trivially reversed via a rainbow table of known user IDs) before they ever reach a log line. This preserves *attribution* — "these five requests came from the same user, and this user's error rate is elevated" — without preserving *identity*.
2. **Never log raw query text, usernames, emails, or session content by default.** This is the single highest-leverage rule in the whole module. Log the *shape* of the content instead: length in tokens/characters, a coarse category/intent label (if you already classify intent for routing), and a boolean flag for whether an upstream PII detector fired. This gives you 90% of the debugging value ("was this a long, complex query that likely blew past a context window?") with none of the raw-content risk.
3. **Version everything.** Every log line carries `model_version` and `prompt_version` (and, in a RAG system, a retrieval-index version). When behavior changes, the *first* diagnostic question is always "did the model change, or did the prompt change, or did the data change" — and if this isn't in the log line already, you are reconstructing it after the fact from deploy timestamps, which is slower and less reliable.
4. **Time-box retention, and encrypt at rest.** A rolling 30-day retention window in encrypted storage (the source material specifies encrypted S3, which generalizes to "encrypted object storage with lifecycle-managed deletion") is long enough to debug almost anything that surfaces (most incidents are noticed within days, and most regulatory/compliance investigations look at a recent window) while enforcing data minimization — you are not accumulating an ever-growing liability of user data you no longer need and increasingly cannot justify holding.
5. **Default to IDs and metrics; raw content is opt-in, not opt-out.** If a specific, policy-approved use case genuinely requires raw prompt/response capture (e.g., a manually-approved debugging session, or a customer who has explicitly consented to content-based support), that capture must be an explicit, audited, narrowly-scoped opt-in — never the default logging path every request goes through.

**When you might relax this:** an internal developer tool with synthetic-only, non-personal test traffic has a much weaker case for hashing/redaction — don't over-engineer privacy machinery for data that was never sensitive. **When you must never relax this:** anything touching real user input in a regulated domain (health, finance, anything under GDPR/CCPA/HIPAA-adjacent obligations) — there, the "opt-in only" rule for raw content is not a suggestion, it is frequently a legal requirement enforced by DPAs (Data Processing Agreements) and audits.

#### Architecture

```
                     Incoming request (raw user text, session, headers)
                                     |
                                     v
                    +--------------------------------------+
                    |   PRIVACY BOUNDARY (in-process, before |
                    |   anything is written to a log sink)   |
                    |                                        |
                    |  raw_user_id  --SHA-256(+pepper)--> user_id_hash
                    |  raw_query    --classify + measure--> query_len, query_category
                    |  raw_query    --PII scanner-------->  pii_detected: bool
                    |  (raw_query, raw_response never cross this boundary  |
                    |   unless an explicit, audited opt-in flag is set)    |
                    +--------------------------------------+
                                     |
                                     v
                    +--------------------------------------+
                    |         telemetry.json (per request)   |
                    |  request_id, trace_id                  |
                    |  user_id_hash                          |
                    |  model_version, prompt_version          |
                    |  input_tokens, output_tokens, cost_usd  |
                    |  ttft_ms, total_latency_ms              |
                    |  status / error_type                    |
                    |  pii_detected: bool                     |
                    +--------------------------------------+
                                     |
                                     v
                 Encrypted object storage, 30-day rolling retention
                 (lifecycle policy auto-deletes past the window)
                                     |
                                     v
                 Analytics / debugging / offline eval joins (via trace_id)
```

#### Examples

**Beginner** — a minimal privacy-safe telemetry record, matching the `telemetry.json` pattern from the source material:

```python
import hashlib
import time
import uuid

PEPPER = "server-side-secret-not-in-code"  # load from a secrets manager in practice

def hash_user_id(raw_user_id: str) -> str:
    return hashlib.sha256((raw_user_id + PEPPER).encode()).hexdigest()

def build_telemetry(
    raw_user_id: str,
    raw_query: str,
    model_version: str,
    prompt_version: str,
    input_tokens: int,
    output_tokens: int,
    cost_usd: float,
    ttft_ms: float,
    total_latency_ms: float,
    pii_detected: bool,
    status: str = "ok",
    error_type: str | None = None,
) -> dict:
    return {
        "request_id": str(uuid.uuid4()),
        "trace_id": str(uuid.uuid4()),          # normally propagated from the OTel span, see §3.2
        "timestamp": time.time(),
        "user_id_hash": hash_user_id(raw_user_id),
        "model_version": model_version,
        "prompt_version": prompt_version,
        "query_len_chars": len(raw_query),       # NOT raw_query itself
        "input_tokens": input_tokens,
        "output_tokens": output_tokens,
        "cost_usd": round(cost_usd, 6),
        "ttft_ms": ttft_ms,
        "total_latency_ms": total_latency_ms,
        "pii_detected": pii_detected,
        "status": status,
        "error_type": error_type,
    }
```

**Intermediate** — wrapping this as a decorator so every inference call point in the codebase emits telemetry consistently, instead of each call site remembering to do it by hand:

```python
import functools
import time

def with_telemetry(model_version: str, prompt_version: str, telemetry_sink):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(raw_user_id, raw_query, *args, **kwargs):
            start = time.perf_counter()
            first_token_time = None
            try:
                result = fn(raw_user_id, raw_query, *args, **kwargs)
                first_token_time = getattr(result, "ttft_ms", None)
                status, error_type = "ok", None
                return result
            except Exception as exc:
                status, error_type = "error", type(exc).__name__
                raise
            finally:
                total_ms = (time.perf_counter() - start) * 1000
                record = build_telemetry(
                    raw_user_id=raw_user_id,
                    raw_query=raw_query,
                    model_version=model_version,
                    prompt_version=prompt_version,
                    input_tokens=getattr(result, "input_tokens", 0) if status == "ok" else 0,
                    output_tokens=getattr(result, "output_tokens", 0) if status == "ok" else 0,
                    cost_usd=getattr(result, "cost_usd", 0.0) if status == "ok" else 0.0,
                    ttft_ms=first_token_time or 0.0,
                    total_latency_ms=total_ms,
                    pii_detected=pii_scan(raw_query),
                    status=status,
                    error_type=error_type,
                )
                telemetry_sink.emit(record)  # e.g., ship to Kafka -> S3, or directly to OTel
        return wrapper
    return decorator
```

**Production-grade** — the same idea, but the sink is an OpenTelemetry log record with structured attributes (so it travels through the *same* pipeline as your traces and metrics, correlated by `trace_id`, rather than being a third, disconnected logging system):

```python
from opentelemetry import trace
from opentelemetry._logs import get_logger, SeverityNumber

logger = get_logger("llm-service.telemetry")

def emit_otel_telemetry(record: dict) -> None:
    span = trace.get_current_span()
    ctx = span.get_span_context()
    logger.emit(
        body="inference_completed",
        severity_number=SeverityNumber.INFO if record["status"] == "ok" else SeverityNumber.ERROR,
        attributes={
            "user_id_hash": record["user_id_hash"],
            "gen_ai.request.model": record["model_version"],
            "app.prompt_version": record["prompt_version"],
            "gen_ai.usage.input_tokens": record["input_tokens"],
            "gen_ai.usage.output_tokens": record["output_tokens"],
            "app.cost_usd": record["cost_usd"],
            "app.ttft_ms": record["ttft_ms"],
            "app.total_latency_ms": record["total_latency_ms"],
            "app.pii_detected": record["pii_detected"],
            "app.status": record["status"],
        },
        # trace_id/span_id are attached automatically by the SDK from the active context,
        # which is exactly how a log line gets joined back to its trace in the backend UI.
    )
```

Note the attribute names above deliberately reuse the OpenTelemetry GenAI Semantic Conventions (`gen_ai.request.model`, `gen_ai.usage.*`) where a standard name exists, and use an `app.*` namespace prefix for anything that's specific to this system rather than standardized — a convention worth adopting broadly (see references.md for the semantic-conventions repo; treat attribute names there as still-evolving/experimental as of mid-2026).

---

### 3.2 OpenTelemetry Fundamentals for LLM Services

#### Theory

OpenTelemetry (OTel) exists to solve a problem this course has seen the shape of before in Module 03-04: vendor lock-in and reinvention. Before OTel, every observability vendor (Datadog, New Relic, a dozen others) shipped its own proprietary instrumentation SDK. Instrument your code against one vendor's SDK and switching vendors meant re-instrumenting your entire codebase. OTel is the CNCF-graduated, vendor-neutral standard that separates **instrumentation** (what you put in your code) from **export** (where the data goes) — you instrument once, and can point the exporter at Jaeger, Prometheus, Grafana Tempo/Loki/Mimir, Datadog, Honeycomb, or a self-hosted Collector, and switch backends later without touching application code.

The three OTel signal types map directly onto the three questions from §2:

| Signal | Answers | LLM-specific example |
|---|---|---|
| **Traces** | "What happened, in what order, for this one request?" | A trace showing `retrieve_context` (120ms) → `rerank` (30ms) → `call_llm` (1.4s) → `invoke_tool: get_weather` (200ms) |
| **Metrics** | "In aggregate, across all requests, is the system healthy?" | P95 TTFT over the last 5 minutes across all requests |
| **Logs** | "What is the detailed, event-level record of one specific thing?" | "Retrieval returned 0 documents for this request" |

**Why spans, specifically, matter so much more for LLM systems than for a typical CRUD service:** an LLM request is rarely one atomic operation. A modern RAG or agentic pipeline is a *chain* — embed the query, search a vector store, rerank, construct a prompt, call the LLM (which itself might stream tokens over seconds), parse a tool call, execute the tool, call the LLM again with the tool result, and finally return. Any one of those steps can be the slow one or the wrong one, and a single "total latency: 3.2s" metric tells you *that* something is slow but not *which link in the chain*. A trace with one span per step turns "it's slow" into "the reranker is taking 900ms of that 3.2s, investigate there" — which is the entire value proposition of tracing over flat logging.

**The span hierarchy the industry has converged on** (also reflected in the OTel GenAI semantic conventions) for a typical LLM call:

```
invoke_agent (or the top-level request span)
 └── retrieve_context           (RAG retrieval)
 └── rerank                     (optional reranking step)
 └── chat  (a.k.a. "call_llm")  (the actual model call — gen_ai.* attributes live here)
      └── execute_tool: <name>  (if the model requested a tool call)
      └── chat                  (follow-up call with tool result, if agentic loop continues)
```

**When to reach for full distributed tracing vs. simpler request logging:** a single-model, single-hop inference service behind a load balancer genuinely may not need a full tracing pipeline — structured per-request logs correlated by a request ID can suffice, and standing up a full Collector + backend is overhead that isn't earning its keep yet. The crossover point is almost always "more than one network hop or more than one internal processing stage per request" — the moment you add RAG, tool calls, or a multi-model routing layer, tracing stops being a nice-to-have and starts being the only way to answer "which step was slow/wrong" without guessing.

#### Architecture

```
                         Your LLM service process
   +----------------------------------------------------------------+
   |  Application code                                              |
   |    with tracer.start_as_current_span("retrieve_context"): ...  |
   |    with tracer.start_as_current_span("chat"): ...               |
   |                                                                  |
   |            OpenTelemetry API  (what you code against)           |
   |                         |                                       |
   |            OpenTelemetry SDK  (implements the API,               |
   |            batches spans, applies sampling)                     |
   |                         |                                       |
   |               OTLP Exporter (gRPC/HTTP, protobuf)                |
   +----------------------------------------------------------------+
                             |
                             v
   +----------------------------------------------------------------+
   |             OpenTelemetry Collector (separate process/pod)       |
   |                                                                  |
   |   Receivers -> Processors -> Exporters                           |
   |   (OTLP in)   (batch,        (Jaeger / Tempo / Prometheus /      |
   |                PII-scrub,     Loki / Datadog / vendor backend)   |
   |                tail-sample)                                     |
   +----------------------------------------------------------------+
                             |
              +--------------+---------------+
              v              v                v
          Traces DB      Metrics DB        Logs DB
        (Tempo/Jaeger)  (Prometheus/Mimir) (Loki/ELK)
              \              |                /
               \             |               /
                v            v              v
                     Grafana (unified UI)
                 dashboards, trace-to-metrics-to-logs
                 correlation via trace_id
```

The Collector is the load-bearing decision most teams get wrong by skipping it: exporting straight from every service instance to a vendor backend couples your application deploys to your observability vendor and makes it hard to do things like PII scrubbing, sampling, or fan-out to two backends during a migration. The Collector is a separate hop specifically so those concerns live in *infrastructure configuration*, not application code.

#### Code

Instrumenting a FastAPI-based LLM service with nested spans:

```python
from fastapi import FastAPI, Request
from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

resource = Resource.create({"service.name": "llm-inference-service", "service.version": "2.3.1"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="otel-collector:4317", insecure=True))
)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("llm-inference-service")

app = FastAPI()
FastAPIInstrumentor.instrument_app(app)  # auto-instruments the HTTP layer (root span per request)

@app.post("/chat")
async def chat(request: Request):
    body = await request.json()

    with tracer.start_as_current_span("retrieve_context") as retrieve_span:
        docs = retrieve_context(body["query"])
        retrieve_span.set_attribute("app.retrieved_doc_count", len(docs))

    with tracer.start_as_current_span("chat") as chat_span:
        chat_span.set_attribute("gen_ai.system", "openai")
        chat_span.set_attribute("gen_ai.request.model", "gpt-5-mini")
        response = call_llm(body["query"], docs)
        chat_span.set_attribute("gen_ai.usage.input_tokens", response.input_tokens)
        chat_span.set_attribute("gen_ai.usage.output_tokens", response.output_tokens)
        # NOTE: prompt/completion content attributes are deliberately NOT set here by
        # default -- OTel GenAI semantic conventions treat content capture as opt-in,
        # matching the privacy-by-default rule from S3.1.

        if response.tool_call:
            with tracer.start_as_current_span(f"execute_tool: {response.tool_call.name}") as tool_span:
                tool_span.set_attribute("gen_ai.tool.name", response.tool_call.name)
                tool_result = execute_tool(response.tool_call)

    return {"answer": response.text}
```

Every span above is a child of the root HTTP span that `FastAPIInstrumentor` created automatically — nesting is implicit via the active-context mechanism (`start_as_current_span` pushes onto a context var; exiting the `with` block pops it back). This is what produces the parent/child tree a Grafana Tempo or Jaeger UI renders as a waterfall — no manual `parent_id` bookkeeping needed.

#### Comparison: the three OTel signal types

| | Traces | Metrics | Logs |
|---|---|---|---|
| **Granularity** | Per-request, hierarchical | Aggregated over a time window | Per-event |
| **Cardinality tolerance** | High (each trace is independent) | Low — high-cardinality labels (e.g., raw user ID) blow up a metrics backend | Medium |
| **Typical retention** | Short (hours-days, often sampled) | Long (weeks-months, cheap to store aggregated) | Medium (this module: 30 days) |
| **Best question answered** | "What happened on this one request?" | "Is the fleet healthy right now?" | "What exactly went wrong, in detail?" |
| **Storage cost driver** | Trace volume x span count x sampling rate | Number of distinct label combinations (cardinality) | Log volume x retention |

---

### 3.3 Designing Spans Around Retrieval, Generation, and Tool Calls

#### Theory

A generic "instrument your code" tutorial will tell you to wrap function calls in spans. The senior-level skill is choosing *span boundaries that map to decisions someone will actually make while debugging*. For an LLM pipeline, that means a span per distinct *cost center* and *failure domain* — not one span per line of code, and not one span for the whole request.

Concretely, boundaries that earn their keep:

- **`retrieve_context`** — isolates vector-DB/search latency and result quality (attribute: doc count, top similarity score) from everything downstream. If retrieval returns garbage, the LLM's answer being bad is not the LLM's fault — you need this span to know where to look.
- **`rerank`** (if present) — often a hidden latency cost that's easy to blame on "the LLM is slow" when it's actually a cross-encoder reranker.
- **`chat` / `call_llm`** — the actual model invocation. This is where TTFT starts being measurable (the span should record a `gen_ai.server.time_to_first_token` style attribute or event for streaming calls) and where token usage and cost are known.
- **`execute_tool: <name>`** — one span per tool call, named with the tool so a trace waterfall immediately shows *which* tool was slow or failed, not just "a tool call happened."
- **Nested/looped generation** — in an agentic loop (ReAct-style, or LangGraph-style, see Module 07/08's agent material), each iteration of the think-act-observe loop gets its own `chat` span nested under a parent `invoke_agent` span, so a trace of a 5-step agent shows five distinct model calls, not one opaque blob.

**Common anti-pattern:** one giant span wrapping the entire request with all the interesting data as attributes on it. This technically "has a trace" but throws away exactly the information tracing exists to provide — where time was actually spent.

#### Architecture — a full agentic trace

```
Trace: req-8f3a2c1e  (total: 3,180 ms)
│
├─ HTTP POST /chat                                    [0 ms -------- 3180 ms]
│  │
│  ├─ retrieve_context                                [10 ms --- 140 ms]     (130ms, 8 docs)
│  │
│  ├─ rerank                                           [140 ms - 175 ms]     (35ms, top_score=0.91)
│  │
│  ├─ invoke_agent (loop)
│  │   │
│  │   ├─ chat  (gen_ai.request.model=gpt-5-mini)      [180 ms -- 950 ms]    (TTFT=210ms, 340 out tok)
│  │   │      -> model requests tool call: get_order_status
│  │   │
│  │   ├─ execute_tool: get_order_status               [955 ms -- 1120 ms]  (165ms, status=200)
│  │   │
│  │   └─ chat  (follow-up, with tool result)           [1125 ms - 3170 ms]  (TTFT=190ms, 900 out tok)
│  │
│  └─ [response streamed to client]                     [ends 3180 ms]
```

Reading this waterfall top to bottom is precisely how a senior engineer triages "why was this request slow" in under a minute: retrieval and reranking together cost 165ms (fine), the first model call was fast, the tool call was fast, and the *second* model call (900 output tokens) dominated total latency — meaning the fix, if one is needed, is about output length/streaming perception, not retrieval or tools.

#### Examples

**Beginner** — a synchronous, single-call trace (no RAG, no tools): one root span, one `chat` child span. Sufficient for a simple wrapper-around-an-API service.

**Intermediate** — RAG without tools: `retrieve_context` → `chat`, as shown in the FastAPI example in §3.2.

**Production-grade** — the full agentic loop shown in the architecture diagram above, instrumented with LangGraph (each graph node becomes a span automatically when the LangGraph OTel integration, or a manual callback handler, is wired in):

```python
from opentelemetry import trace

tracer = trace.get_tracer("agent-service")

def agent_node(state):
    with tracer.start_as_current_span(f"chat", kind=trace.SpanKind.CLIENT) as span:
        span.set_attribute("gen_ai.request.model", state["model"])
        span.set_attribute("app.agent_step", state["step_count"])
        response = llm.invoke(state["messages"])
        span.set_attribute("gen_ai.usage.output_tokens", response.usage.output_tokens)
        if response.tool_calls:
            span.add_event("tool_call_requested", {"tool_name": response.tool_calls[0].name})
        return {"messages": [response]}

def tool_node(state):
    tool_call = state["messages"][-1].tool_calls[0]
    with tracer.start_as_current_span(f"execute_tool: {tool_call.name}") as span:
        span.set_attribute("gen_ai.tool.name", tool_call.name)
        try:
            result = tools[tool_call.name].invoke(tool_call.args)
            span.set_status(trace.Status(trace.StatusCode.OK))
        except Exception as exc:
            span.set_status(trace.Status(trace.StatusCode.ERROR, str(exc)))
            raise
        return {"messages": [result]}
```

---

### 3.4 Metrics Design: TTFT, P95/P99, Cost, Error Rate, Cache-Hit Rate

#### Theory

The source material is emphatic on a point that's easy to under-value until you've lived through an incident: **teams should track a small set of key performance indicators, not everything they could possibly measure.** More metrics is not more insight past a certain point — it's more dashboard noise and more alert-rule maintenance burden. The discipline is choosing metrics that each answer a distinct question a human needs answered fast.

| Metric | Question it answers | Why P95/P99 (not average) matters |
|---|---|---|
| **TTFT (Time To First Token)** | "How long does the user stare at a blank screen before anything appears?" | The dominant *perceived*-latency metric for streaming chat UX — users tolerate a slow-but-streaming response far better than a long silent wait. |
| **Total / P95 / P99 latency** | "How long does the whole request take, and how bad is the tail?" | Averages hide the worst experiences. If P50 is 400ms but P99 is 8s, 1% of your users are having a terrible time and the average will never show you that. |
| **Cost per query** | "Are we spending what we think we're spending?" | Token-based pricing means a single verbose prompt template change, or a runaway agent loop, can silently 10x your bill with no error thrown anywhere. |
| **Error rate** | "Is the system correctly serving requests?" | HTTP errors *and* soft failures (e.g., output-parsing failures, guardrail rejections) both count — a 200 response that fails downstream parsing is still a failure from the user's perspective. |
| **Cache-hit rate** | "Are we needlessly recomputing things we've already computed?" | Directly trades off against both cost and latency — this is usually the single highest-leverage lever for improving both at once. |

**Why histograms, specifically, for latency and TTFT — not gauges or averages.** Prometheus (the de facto standard metrics backend this section targets) does not store percentiles natively; a "P95" is *computed at query time* from a histogram's bucket counts via `histogram_quantile()`. If you only expose a gauge of "current latency" or a running average, you have permanently thrown away the distribution shape and can never recover P95/P99 after the fact. This is one of the most common and costly metrics-design mistakes: choosing the wrong metric *type* up front is not fixable retroactively — you can only fix it going forward, losing all historical percentile visibility for the gap.

**vLLM as a concrete, real-world reference design** (see references.md/github.md for the full doc): vLLM's native `/metrics` endpoint exposes exactly this pattern — histograms named `vllm:time_to_first_token_seconds`, `vllm:inter_token_latency_seconds` (TPOT), and `vllm:e2e_request_latency_seconds`, plus counters for token totals and cache queries/hits, and gauges for current KV-cache utilization. If you are designing metrics for a custom LLM-serving layer and want a battle-tested naming/typing template, this is the one to copy.

#### Architecture

```
                    LLM service instances (N replicas)
        +------------+   +------------+   +------------+
        |  replica 1  |   |  replica 2  |   |  replica 3  |
        |  /metrics   |   |  /metrics   |   |  /metrics   |
        +------------+   +------------+   +------------+
               |               |               |
               +-------+-------+-------+-------+
                       |  Prometheus scrapes    |
                       |  every N seconds        |
                       v
              +-------------------+
              |    Prometheus      |   stores time series,
              |  (TSDB + rules)    |   evaluates alerting rules
              +-------------------+
                 |              |
                 v              v
        +----------------+  +------------------+
        |  Alertmanager   |  |     Grafana       |
        |  (routes fired   |  |  (dashboards,     |
        |   alerts to      |  |   ad-hoc PromQL,  |
        |   Slack/Pager)   |  |   correlated with |
        +----------------+  |   Tempo traces)    |
                             +------------------+
```

#### Code — Prometheus metric definitions in Python

```python
from prometheus_client import Counter, Histogram, Gauge

# Histograms: for anything you need percentiles on. Buckets should bracket
# your actual SLA-relevant range -- pick them from real traffic, don't guess.
TTFT_SECONDS = Histogram(
    "llm_time_to_first_token_seconds",
    "Time to first streamed token",
    buckets=(0.05, 0.1, 0.2, 0.3, 0.5, 0.75, 1.0, 1.5, 2.0, 3.0, 5.0),
    labelnames=["model_version"],
)

E2E_LATENCY_SECONDS = Histogram(
    "llm_e2e_request_latency_seconds",
    "End-to-end request latency",
    buckets=(0.1, 0.25, 0.5, 1.0, 2.0, 4.0, 8.0, 15.0, 30.0),
    labelnames=["model_version"],
)

# Counters: only ever go up -- ratios (error rate, cache-hit rate) are
# computed at query time from two counters, never stored as a raw ratio.
REQUESTS_TOTAL = Counter(
    "llm_requests_total", "Total requests", labelnames=["model_version", "status"]
)
CACHE_LOOKUPS_TOTAL = Counter(
    "llm_cache_lookups_total", "Cache lookups", labelnames=["result"]  # result=hit|miss
)
COST_USD_TOTAL = Counter(
    "llm_cost_usd_total", "Cumulative cost in USD", labelnames=["model_version"]
)

# Gauges: current point-in-time value, can go up or down.
INFLIGHT_REQUESTS = Gauge("llm_inflight_requests", "Requests currently being processed")
```

```python
# Recording, at the call site:
import time

def handle_request(query, model_version):
    INFLIGHT_REQUESTS.inc()
    start = time.perf_counter()
    try:
        first_token_at = None
        for token in stream_llm_response(query):
            if first_token_at is None:
                first_token_at = time.perf_counter()
                TTFT_SECONDS.labels(model_version=model_version).observe(first_token_at - start)
            yield token
        REQUESTS_TOTAL.labels(model_version=model_version, status="ok").inc()
    except Exception:
        REQUESTS_TOTAL.labels(model_version=model_version, status="error").inc()
        raise
    finally:
        E2E_LATENCY_SECONDS.labels(model_version=model_version).observe(time.perf_counter() - start)
        INFLIGHT_REQUESTS.dec()
```

**Critical cardinality warning:** every distinct combination of label values becomes its own time series. `model_version` (a handful of values) is a safe label. `user_id_hash` as a label would be catastrophic — Prometheus cardinality explosion, one of the most common production-incident-causing mistakes in this space (see §5). If you need per-user analysis, that's a job for traces/logs joined by `trace_id`, not a metric label.

#### PromQL — computing percentiles and rates

```promql
# P95 TTFT over the last 5 minutes, per model version
histogram_quantile(0.95,
  sum(rate(llm_time_to_first_token_seconds_bucket[5m])) by (le, model_version)
)

# Error rate over the last 5 minutes
sum(rate(llm_requests_total{status="error"}[5m]))
/
sum(rate(llm_requests_total[5m]))

# Cache-hit rate over the last 30 minutes
sum(rate(llm_cache_lookups_total{result="hit"}[30m]))
/
sum(rate(llm_cache_lookups_total[30m]))

# Cost accumulated in the trailing 1 hour
sum(increase(llm_cost_usd_total[1h]))
```

---

### 3.5 Grafana Dashboard Design for LLM Services

#### Theory

The source material's instinct — "a custom dashboard view with rolling P95 latency and 1-hour cost accumulation" plus "a weekly summary view" — points at a real design principle: **different audiences and different time horizons need different dashboards**, and cramming both into one screen serves neither well.

| Dashboard | Audience | Time horizon | Core panels |
|---|---|---|---|
| **Live ops dashboard** | On-call engineer, during/after an incident | Last 15 min – 6 hours, rolling | P95/P99 TTFT & latency, error rate, in-flight requests, cache-hit rate |
| **Weekly/trend dashboard** | Team lead, product owner, cost owner | Last 7-30 days | Cost-per-query trend, cache-hit-rate trend, error-rate trend, request volume, model-version rollout progress |
| **Per-deploy comparison dashboard** | Engineer validating a new prompt/model version (ties to Module 12's quality gates) | Since last deploy vs. prior version | Side-by-side latency/cost/error rate, old vs. new model/prompt version |

**Design rule that separates a useful dashboard from a wall of noise:** every panel should be answerable with "so what do I do about this number" in one sentence. If a panel's answer is "huh, interesting," it belongs in an ad-hoc exploration view, not the default on-call dashboard.

#### Architecture — panel layout (ASCII wireframe)

```
+----------------------------------------------------------------------+
|  LLM Service — Live Ops                              [Last 15m v]     |
+----------------------------------------------------------------------+
| P95/P99 TTFT (ms)        | P95/P99 E2E Latency (ms) | Error Rate (%)  |
|  [ time series, two      |  [ time series, two      |  [ time series, |
|    lines: p95, p99 ]     |    lines: p95, p99 ]     |    red thresh   |
|                          |                          |    line at 0.5%]|
+--------------------------+--------------------------+-----------------+
| In-flight Requests       | Cache-Hit Rate (%)       | Cost (last 1h)  |
|  [ gauge / time series ] |  [ time series, thresh   |  [ big number + |
|                          |    line at 20% ]         |    sparkline ]  |
+--------------------------+--------------------------+-----------------+
| Requests by model_version (stacked area)     | Recent errors (table, |
|                                               |  linked to trace_id)  |
+-----------------------------------------------+-----------------------+
```

#### Code — Grafana panel as PromQL + JSON model (excerpt)

```json
{
  "title": "P95/P99 TTFT (ms)",
  "type": "timeseries",
  "targets": [
    {
      "expr": "histogram_quantile(0.95, sum(rate(llm_time_to_first_token_seconds_bucket[5m])) by (le)) * 1000",
      "legendFormat": "P95"
    },
    {
      "expr": "histogram_quantile(0.99, sum(rate(llm_time_to_first_token_seconds_bucket[5m])) by (le)) * 1000",
      "legendFormat": "P99"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "ms",
      "thresholds": {
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 800 },
          { "color": "red", "value": 1000 }
        ]
      }
    }
  }
}
```

Note the threshold values (800ms yellow, 1000ms red) directly mirror the alert threshold discussed in §3.6 — a well-designed dashboard and its alert rules should visually agree, so an on-call engineer glancing at a red panel immediately understands why they were paged.

---

### 3.6 Alerting Design and On-Call Runbooks

#### Theory

The source material's `matrix_monitor.py` pattern captures the essential alerting discipline in miniature: **read a short rolling window, check a small number of specific conditions, and route different conditions to different actions.** This generalizes into the core alerting principles this section teaches:

1. **Alert on symptoms, not every possible cause** (this is also core Google SRE Book / Prometheus alerting-practices guidance — see references.md). Page on "P95 TTFT is high" (a symptom users feel), not on every internal signal that *could* cause it (GC pauses, a specific pod's CPU, a specific dependency's latency) — those belong in the dashboard for the engineer to *investigate with*, once paged, not as separate pages themselves.
2. **Use short rolling windows with a `for` duration**, not instantaneous point-in-time checks. A single bad data point (one slow request during a GC pause) should not page anyone — `for: 5m` (fire only if the condition holds continuously for 5 minutes) is what separates a real degradation from noise. This is precisely why the source material's "consecutive logs" framing for the TTFT alert matters: consecutive, not one-off.
3. **Separate problem types into separate alerts with separate severities and separate routing.** Error spike, latency spike, and elevated fallback-model usage are three different problems with three different fixes — collapsing them into one generic "something's wrong" alert forces every on-call engineer to re-diagnose from scratch every time, which is exactly the waste that specific alerts prevent.
4. **Every alert that can fire must have a runbook.** An alert with no runbook trains engineers to either ignore it or spend the first 20 minutes of an incident figuring out what to even check — both are avoidable with five minutes of runbook-writing at alert-design time, not at 2 a.m.

**Concrete threshold table (grounded in the source material, generalized into a full rule set):**

| Alert | Condition | Window | Severity | Action |
|---|---|---|---|---|
| TTFT degradation | P95 TTFT > 1000ms | 5 min sustained | Warning -> Slack | Investigate model load, autoscaling lag |
| Error spike | Error rate > 0.5% (some teams use 3% for a noisier baseline — tune to your own SLO) | 5 min sustained | Critical -> Page | Check recent deploys first, roll back if correlated |
| Latency spike | P95 E2E latency > 100ms above rolling baseline (source material's `matrix_monitor.py` threshold; recalibrate per SLA) | 5 min sustained | Warning -> Slack | Check downstream dependency latency, retrieval/tool spans |
| High fallback usage | Fallback-model usage rate > 10% | 5 min sustained | Critical -> Page on-call | Primary model likely down/rate-limited; check provider status |
| Low cache-hit rate | Cache-hit rate < 20% | 30 min sustained | Info -> weekly review, not a page | Review cache key design, prompt-prefix stability |
| Cost budget | Daily cost projection exceeds budget | Daily rolling | Warning -> Slack + email | Check for runaway agent loops, verbose prompt regressions |

#### Architecture — the alert-to-response pipeline

```
Prometheus (evaluates alerting rules every scrape interval)
        |
        | rule fires if condition true for the full `for:` duration
        v
Alertmanager
        |
        +-- groups related alerts (avoid 50 separate pages for one root cause)
        +-- deduplicates
        +-- routes by severity/team label
        |
        +---------------------+----------------------+
        v                     v                       v
    Slack channel        PagerDuty/Opsgenie      Weekly digest email
    (Warning severity)   (Critical -> pages       (Info severity,
                          on-call engineer)         trend-only alerts)
                                |
                                v
                    On-call engineer opens the
                    linked runbook page (see below)
```

#### Code — the source pattern, expanded into production shape

```python
# matrix_monitor.py -- expanded from the source material's pattern.
# In production this logic is usually expressed as Prometheus alerting
# rules (see YAML below) rather than a hand-rolled polling loop, but the
# polling-loop form is worth understanding first because it makes the
# "read a window, check conditions, route to different actions" logic explicit.

from dataclasses import dataclass

@dataclass
class WindowMetrics:
    error_rate: float
    p95_latency_ms: float
    fallback_rate: float

def check_and_alert(window: WindowMetrics, alert_sink, page_sink) -> None:
    if window.error_rate > 0.03:
        alert_sink.send("Error spike: error_rate=%.3f over last 5m" % window.error_rate)

    if window.p95_latency_ms > 1000:
        alert_sink.send("Latency spike: p95=%.0fms over last 5m" % window.p95_latency_ms)

    if window.fallback_rate > 0.10:
        page_sink.page(
            "High fallback usage: %.1f%% of requests used the fallback model. "
            "Primary model may be degraded or rate-limited." % (window.fallback_rate * 100)
        )
```

The production-grade equivalent as a Prometheus alerting rule (this is what actually runs continuously in most real stacks, rather than a custom polling script):

```yaml
groups:
  - name: llm-service-alerts
    rules:
      - alert: HighTTFT
        expr: |
          histogram_quantile(0.95, sum(rate(llm_time_to_first_token_seconds_bucket[5m])) by (le)) > 1.0
        for: 5m
        labels:
          severity: warning
          team: llm-platform
        annotations:
          summary: "P95 TTFT above 1000ms for 5+ minutes"
          runbook_url: "https://runbooks.internal/llm-service/high-ttft"

      - alert: ErrorRateSpike
        expr: |
          sum(rate(llm_requests_total{status="error"}[5m])) / sum(rate(llm_requests_total[5m])) > 0.005
        for: 5m
        labels:
          severity: critical
          team: llm-platform
        annotations:
          summary: "Error rate above 0.5% for 5+ minutes"
          runbook_url: "https://runbooks.internal/llm-service/error-spike"

      - alert: HighFallbackUsage
        expr: |
          sum(rate(llm_requests_total{model_version="fallback"}[5m])) / sum(rate(llm_requests_total[5m])) > 0.10
        for: 5m
        labels:
          severity: critical
          team: llm-platform
        annotations:
          summary: "Fallback model serving >10% of traffic — primary model likely degraded"
          runbook_url: "https://runbooks.internal/llm-service/fallback-spike"
```

**Sample on-call runbook entry** (this is the artifact that makes an alert actionable — write one of these for every alert you ship):

```
ALERT: HighFallbackUsage
--------------------------------------------------------------
WHAT IT MEANS: More than 10% of requests in the last 5 minutes
were served by the fallback model, meaning the primary model
provider is failing health checks, rate-limiting us, or timing out.

FIRST 3 STEPS:
 1. Check the primary provider's status page / your own health-check
    dashboard for the primary model endpoint.
 2. Check Grafana "Requests by model_version" panel to confirm this
    is a step-change (provider incident) vs. gradual drift (capacity).
 3. If provider-side: no action needed beyond confirming fallback
    quality is acceptable; post a status update to #llm-platform.
    If capacity-side: consider scaling the primary deployment or
    raising its rate-limit ceiling.

ESCALATION: If fallback usage exceeds 50% for 15+ minutes, escalate
to the on-call lead — this indicates a near-total primary outage.

RELATED: See "ErrorRateSpike" runbook if error rate is also elevated.
```

---

### 3.7 OpenTelemetry vs. Langfuse vs. LangSmith for LLM Observability

#### Theory

By mid-2026 there is a real, non-obvious build-vs-buy decision here, and the honest answer for most teams is "some of both" — OTel as the vendor-neutral instrumentation and transport layer, with either a self-hosted or SaaS LLM-specific backend consuming it. The three options solve overlapping but distinct problems:

- **OpenTelemetry** is a *standard*, not a product. It defines how to produce and transport traces/metrics/logs and (via the GenAI semantic conventions) a vocabulary for LLM-specific attributes. It has no opinion on storage, dashboards, or evaluation UI — you point its exporter at whatever backend you choose.
- **Langfuse** is an open-source (MIT-licensed), LLM-observability-*specific* product with OTel-native SDKs, self-hostable-first, that adds LLM-specific UI on top of traces: prompt-version comparison, cost breakdowns, dataset-based evaluation, and human-annotation workflows. Its acquisition by ClickHouse (announced as part of a $400M Series D round in January 2026) is a strong signal of how strategically important this exact problem space has become to the broader data-infrastructure market.
- **LangSmith** is LangChain's proprietary, deeply-integrated observability/evaluation product — the tightest integration if your stack is already LangChain/LangGraph, with self-hosting available only on Enterprise tiers, and a newer "LangSmith Engine" feature (added around May 2026) for automated failure clustering across production traces.

| Dimension | OpenTelemetry (raw) | Langfuse | LangSmith |
|---|---|---|---|
| **License / hosting model** | Open standard; you own the whole pipeline | Open-source (MIT), self-hostable-first, also offers cloud | Proprietary; self-hosting is Enterprise-only |
| **LLM-specific UI out of the box** | None — you or your backend build it | Yes: prompt/version comparisons, cost views, eval datasets, annotation queues | Yes: deepest with LangChain/LangGraph, plus automated failure clustering (LangSmith Engine) |
| **Vendor lock-in risk** | Lowest — export anywhere | Low — OTel-native SDKs, standard export | Higher — proprietary format/tightest coupling to LangChain ecosystem |
| **Best fit** | Teams building a custom pipeline, or needing to fan-out to multiple backends, or with strict data-residency/self-hosting requirements | Teams wanting an LLM-native product without vendor lock-in, especially if not on LangChain | Teams already deep in LangChain/LangGraph who want the most integrated experience and don't mind the proprietary tradeoff |
| **Pricing model** | Free (you pay for your own backend infra) | Free self-hosted; unit-based pricing for cloud | Proprietary pricing; self-host requires Enterprise contract |
| **Semantic conventions maturity (mid-2026)** | Still experimental/evolving — attribute names can change | Adopts OTel GenAI conventions where they exist | Own internal schema, LangChain-ecosystem-specific |

**When to use which, concretely:**
- **Pure OTel + your own backend (Tempo/Loki/Prometheus/Grafana or a Collector fan-out to a vendor)** when you need multi-backend flexibility, have in-house observability platform expertise already (common at large tech companies — see §4), or have strict data-residency requirements that rule out sending traces to a third party at all.
- **Langfuse** when you want LLM-native features (prompt comparison, eval datasets, cost dashboards) without committing to a single proprietary vendor, and self-hosting or the open-source model matters to you.
- **LangSmith** when your team's orchestration layer is already LangChain/LangGraph and you want the path of least resistance with the deepest first-party integration, and you're comfortable with a proprietary, primarily-SaaS product.

**Common mistake in this decision:** treating it as mutually exclusive. A very common, pragmatic production pattern is: instrument with OTel (so you're never locked into one observability vendor for *infrastructure-level* traces/metrics), and additionally send LLM-specific spans to Langfuse or LangSmith for the LLM-native UI (prompt diffing, eval-dataset linking) that a generic backend won't give you out of the box.

---

## 4. Real-World Case Studies (Reasoned Inference)

The following are reasoned inferences about how organizations with public engineering-blog patterns would plausibly architect LLM observability, not confirmed internal specifics.

**A frontier AI lab serving a chat product at OpenAI/Anthropic/Google scale** would very likely run its own internal, purpose-built observability stack rather than a generic off-the-shelf Grafana setup, for the simple reason that request volume at that scale makes naive full-fidelity tracing prohibitively expensive — this is exactly the kind of environment where **tail-based sampling** (only fully retain traces that look interesting: errors, high latency, flagged content) becomes essential rather than optional, and where cost-per-query tracking is tied directly into internal capacity-planning and GPU-fleet-allocation systems rather than being a standalone dashboard. Given the safety stakes involved, such a system would also plausibly route a sampled fraction of production traffic through an asynchronous, judge-model-based content-quality check (connecting back to this course's Module 11) as a genuine observability signal alongside latency/cost, not merely an offline eval step.

**Netflix or Spotify**, both publicly known for deep investment in their own internal observability tooling (Netflix's Atlas, and both companies' well-documented A/B-testing-and-metrics cultures), would plausibly extend that existing muscle to LLM-based features (recommendation explanations, conversational search) rather than adopting a wholesale new stack — the reasonable inference is that they'd wire LLM-specific `gen_ai.*`-style metrics into their *existing* metrics pipeline and alerting culture, since the organizational cost of running two parallel observability stacks would outweigh the benefit of a purpose-built LLM tool for what is, for them, one feature category among very many.

**Uber**, with a long public track record of investing in internal ML platforms (Michelangelo) and a business model highly sensitive to tail latency (a slow ETA prediction has direct rider-experience and matching-efficiency consequences), would plausibly apply the same tail-latency discipline to any LLM-based features (support chatbots, driver-facing assistants) that it has historically applied to its core ML — meaning P99, not just P95, would be a first-class SLO metric, and cost-per-query would be scrutinized heavily given the sheer request volume such a company operates at.

**Databricks**, as a company that both builds and sells MLOps/data-infrastructure tooling (including MLflow, which this course covers in Module 13, and which has its own OTel GenAI tracing integration), would plausibly dogfood exactly the OTel-plus-MLflow-tracing pattern described in this module for its own internal LLM features, and its public documentation of MLflow's OTel semantic-convention mapping (see references.md) is a reasonable signal of the internal engineering opinion that OTel-standard attribute naming is worth aligning to even inside a vendor's own product.

**NVIDIA**, given its central role in the inference-serving stack (Triton Inference Server, and its investment in the broader vLLM/serving ecosystem), would plausibly treat GPU-level telemetry (utilization, memory, KV-cache pressure) as a first-class signal integrated directly alongside request-level metrics like TTFT/TPOT — because for GPU-bound LLM serving, the causal chain from "GPU memory pressure" to "TTFT degradation" is short and direct, making it a natural pairing for any dashboard NVIDIA-adjacent teams would plausibly build.

---

## 5. Common Mistakes

1. **Logging raw prompts/responses "just in case," by default.** The single most damaging mistake this module addresses — turns a debugging convenience into a standing privacy/compliance liability. Fix: metadata-first logging, raw-content capture explicitly opt-in and audited.
2. **Using a gauge or an average for latency instead of a histogram.** This is not fixable retroactively — once you've thrown away the distribution, you cannot recover P95/P99 for that period. Fix: histograms from day one for anything you'll ever want a percentile on.
3. **High-cardinality metric labels** (raw user IDs, raw query text, unbounded free-form strings as label values) causing Prometheus cardinality explosion and backend cost/performance collapse. Fix: keep metric labels to a small, bounded set of values (model version, status, region); push per-user/per-request detail into traces/logs joined by `trace_id`.
4. **One giant span per request instead of spans per pipeline stage.** Defeats the purpose of tracing — you still can't tell which stage was slow. Fix: span boundaries at each real cost/failure center (retrieval, rerank, chat, tool call).
5. **Alerting on every metric that moves, instead of a small set of symptom-level alerts.** Causes alert fatigue, which causes real alerts to get ignored. Fix: alert on user-visible symptoms with `for:` windows; keep cause-level signals in dashboards for investigation, not as separate pages.
6. **Shipping an alert with no runbook.** Forces every on-call engineer to rediscover the correct response from scratch, every time, usually at 2 a.m. Fix: no alert ships without an accompanying runbook entry.
7. **Treating OTel semantic-convention attribute names as stable/final.** The GenAI semantic conventions were still marked experimental as of early/mid-2026 — hardcoding assumptions about exact attribute names without checking the current spec risks silent breakage on an SDK upgrade. Fix: pin your OTel SDK/semconv versions deliberately and review changelogs on upgrade.
8. **No trace-ID correlation between logs, metrics, and traces.** Without a shared `trace_id` threaded through all three signals, you can't jump from "this metric spiked" to "here's the exact trace that caused it" — you're left correlating by timestamp, which is slow and error-prone during an incident.
9. **Forgetting retention/cost planning for observability data itself.** Traces and logs at production LLM volume are not free to store — unbounded retention of full-fidelity traces is itself a cost/compliance problem mirroring the raw-content-logging mistake above.
10. **Confusing "the model responded successfully" with "the system behaved correctly."** An LLM can return HTTP 200 with a wrong, unsafe, or off-policy answer — observability that only watches HTTP status codes and latency will miss this entirely; content-quality signals (sampled judge evaluation, guardrail-rejection rates) must be part of the same monitoring picture.

---

## 6. Best Practices and Production Tips

**When to use full OTel tracing vs. simpler logging:** reach for full tracing the moment a request involves more than one hop or processing stage (RAG, tool calls, multi-model routing); for a genuinely single-call wrapper service, structured per-request logs correlated by a request ID can be sufficient and lighter-weight.

**When NOT to over-invest:** don't stand up a full Collector + multi-backend pipeline for a low-traffic internal prototype — the operational overhead of running and maintaining the observability stack itself can exceed the value it returns at that scale. Revisit once the service has real users and real incidents to learn from.

**Alternatives to self-hosting the whole stack:** Grafana Cloud, Datadog, Honeycomb, and similar vendors will happily ingest OTLP directly, letting you keep vendor-neutral instrumentation while outsourcing the operational burden of running Prometheus/Tempo/Loki/Grafana yourselves — a very reasonable choice for smaller teams.

**Cost:** the three biggest observability cost drivers are (a) trace volume x sampling rate, (b) metric cardinality, (c) log/trace retention window. Tail-based sampling (keep all error/slow traces, sample down the "boring" successful ones) is the single highest-leverage lever for controlling (a) at scale.

**Scaling:** as request volume grows, move from head-based sampling (decide to sample at trace start, cheap but blind to whether the trace turns out interesting) to tail-based sampling (decide after seeing the whole trace, requires buffering but preserves 100% of interesting traces) — this typically requires a dedicated Collector tier rather than in-process sampling decisions.

**Security:** encrypt telemetry at rest and in transit; treat the observability pipeline itself as being in-scope for your access-control and audit posture — a leaked observability backend can leak just as much sensitive data as a leaked primary database if raw content ever ended up in it (another reason the opt-in-only rule for raw content matters).

**Performance tradeoffs:** synchronous, blocking telemetry emission on the request hot path adds latency to every request; batch span/metric export (as shown in the `BatchSpanProcessor` example) and asynchronous log shipping are standard mitigations — never let observability code become the reason your own P95 latency alert fires.

**Monitoring the monitors:** per Prometheus's own alerting-practices guidance, periodically verify the monitoring pipeline end-to-end (a "blackbox" synthetic check that an alert actually fires and actually reaches Slack/PagerDuty) — a broken alerting pipeline that silently stops paging anyone is one of the worst possible failure modes, because it fails exactly when you need it most and gives no signal that it has failed.

---

## 7. Interview Questions

1. **"Why do you need histograms rather than gauges for latency metrics in Prometheus, and what happens if you get this wrong?"**
   *Model answer:* Prometheus computes percentiles at query time from histogram bucket counts via `histogram_quantile()`; it has no native percentile storage. A gauge or average throws away the distribution shape entirely. Getting this wrong is not retroactively fixable — you permanently lose the ability to compute P95/P99 for any period before you switch to a histogram, which matters because averages hide exactly the tail-latency problems that most affect real users.

2. **"Design a privacy-safe telemetry schema for an LLM chat service. What do you log, what do you never log, and why?"**
   *Model answer:* Log request/trace IDs, a one-way SHA-256(+pepper) hash of the user identifier, model and prompt versions, token counts, cost, TTFT/latency, status/error type, and a PII-detected boolean. Never log raw query/response text, usernames, or emails by default — log query length and a coarse category instead. Raw content capture is an explicit, audited, narrowly-scoped opt-in only. Apply a rolling retention window (e.g., 30 days) in encrypted storage.

3. **"A trace shows a request took 3.2 seconds total. How would you use spans to find out where that time actually went, and what would you look for?"**
   *Model answer:* Instrument spans around each pipeline stage (retrieval, rerank, chat/generation, tool calls) so the trace waterfall shows a per-stage time breakdown rather than one opaque total. Look for which child span consumes the largest share of the parent's duration, check whether a `chat` span's duration correlates with output-token count (suggesting streaming/generation length is the driver) versus TTFT being high (suggesting model-load/queueing), and check tool-call spans for slow external dependencies.

4. **"Why is high-cardinality labeling a serious mistake in a Prometheus-based metrics design, and how do you avoid it while still supporting per-user debugging?"**
   *Model answer:* Every distinct label-value combination becomes its own time series; unbounded-cardinality labels (raw user IDs, raw free text) cause the metrics backend's time-series count, memory, and query cost to explode, degrading or crashing the whole system. Avoid it by keeping metric labels to small, bounded sets (model version, status, region) and pushing per-user/per-request granularity into traces and logs, correlated by `trace_id` instead of metric labels.

5. **"Compare OpenTelemetry, Langfuse, and LangSmith for LLM observability. When would you choose each?"**
   *Model answer:* OTel is a vendor-neutral standard for producing/transporting traces-metrics-logs, with no opinion on storage or LLM-specific UI — best when you need backend flexibility or strict data-residency control. Langfuse is an open-source, self-hostable-first, OTel-native product with LLM-specific UI (prompt comparison, eval datasets, cost views) — best when you want LLM-native features without vendor lock-in. LangSmith is LangChain's proprietary, deepest-integrated product for LangChain/LangGraph stacks, with self-hosting Enterprise-only — best when you're already committed to that ecosystem and want the most integrated experience. Many production teams combine OTel instrumentation with one of the LLM-native backends rather than choosing exclusively.

6. **"How would you design alerting so that on-call engineers don't experience alert fatigue, using an LLM service's error rate, latency, and fallback-usage as examples?"**
   *Model answer:* Alert on user-visible symptoms (P95 TTFT, error rate, fallback usage) rather than every internal cause signal; require a sustained `for:` window (e.g., 5 minutes) so transient blips don't page anyone; give each distinct problem type its own alert and severity/routing (error spike pages critical, low cache-hit rate is informational-only); and ship a runbook with every alert so a page always comes with a clear first action.

7. **"What is the OpenTelemetry Collector, and why not export directly from your application to your observability backend?"**
   *Model answer:* The Collector is a separate process/pod that receives telemetry (typically via OTLP) and applies processing (batching, PII scrubbing, sampling) before exporting to one or more backends. Exporting directly from application code couples your deploys to a specific vendor's SDK/format and pushes processing concerns like sampling and redaction into application code; the Collector decouples instrumentation from destination, letting you change backends or add PII scrubbing/sampling in infrastructure configuration without touching application code.

8. **"An LLM returns HTTP 200 but with an incorrect or unsafe answer. Would your standard latency/error-rate/cost dashboard catch this? If not, what would you add?"**
   *Model answer:* No — HTTP status and latency metrics are blind to content correctness; a fluent, confident wrong answer produces no error signal in a purely infrastructure-level dashboard. You'd add a content-quality signal: sampled, asynchronous LLM-as-judge evaluation of production responses (connecting to Module 11), guardrail-rejection rate as its own tracked metric, and user-feedback signals (thumbs down rate, regeneration rate) treated as first-class monitored metrics alongside latency/cost/error rate.

---

## 8. Summary, Key Takeaways, and Production Checklist

### Summary

Production LLM observability rests on two inseparable disciplines: seeing enough to debug and improve the system, and protecting user data while doing so. OpenTelemetry provides the vendor-neutral mechanism — traces (spans around retrieval/generation/tool calls) for per-request diagnosis, metrics (histograms for TTFT/latency, counters for cost/errors/cache) for fleet-level health, and privacy-safe structured logs for detailed event records — all correlated by a shared `trace_id`. Prometheus and Grafana turn those metrics into dashboards built around a deliberately small set of action-driving signals, and alerting rules turn threshold breaches into routed, runbook-backed responses rather than noise. Langfuse and LangSmith layer LLM-native UI on top of this foundation for teams that want it.

### Key Takeaways

- Privacy is not a compliance afterthought bolted onto observability — it is the first design decision: hash identifiers, never log raw content by default, version everything, time-box retention.
- Traces answer "what happened on this one request" by giving you a per-stage timeline; design span boundaries around real cost/failure centers (retrieval, rerank, generation, tool calls), not one span per request or per line.
- Metrics answer "is the fleet healthy right now"; use histograms for anything you need percentiles on — this choice is not retroactively fixable.
- A small set of action-driving dashboard panels beats a wall of every-metric-you-could-measure.
- Alerts should fire on user-visible symptoms with sustained windows, be split by problem type, and always ship with a runbook.
- OTel, Langfuse, and LangSmith are not mutually exclusive — a common, pragmatic pattern is OTel instrumentation feeding both a general backend and an LLM-native product.

### Production Checklist

- [ ] All user/session identifiers are hashed (SHA-256 + server-side pepper) before entering any log/telemetry sink.
- [ ] Raw prompt/response content is never logged by default; capture is an explicit, audited, narrowly-scoped opt-in.
- [ ] Every telemetry record carries `model_version` and `prompt_version` (and retrieval-index version, if RAG).
- [ ] Retention window and encryption-at-rest are configured and enforced via lifecycle policy, not manual cleanup.
- [ ] OTel traces have spans around every distinct pipeline stage (retrieval, rerank, generation, tool calls), each carrying `gen_ai.*`-aligned attributes where a standard name exists.
- [ ] Latency/TTFT metrics are histograms with buckets chosen from real traffic, not gauges or averages.
- [ ] Metric labels are all low-cardinality (model version, status, region) — no raw user IDs or free text as label values.
- [ ] A live-ops dashboard exists with P95/P99 TTFT/latency, error rate, cache-hit rate, and cost, scoped to a short rolling window.
- [ ] A weekly/trend dashboard exists for cost-per-query, cache-hit-rate, and error-rate trends over 7-30 days.
- [ ] Every alert has a `for:` sustained-window condition, a severity/routing tier, and a linked runbook.
- [ ] The alerting pipeline itself is periodically verified end-to-end (a synthetic check that alerts actually fire and actually reach Slack/PagerDuty).
- [ ] A content-quality signal (sampled judge evaluation, guardrail-rejection rate, user-feedback rate) is monitored alongside infrastructure metrics, since HTTP 200 does not imply a correct answer.
- [ ] Build-vs-buy decision for LLM-specific observability UI (raw OTel backend vs. Langfuse vs. LangSmith) is made deliberately, with named tradeoffs, not by default inertia.

---

## 9. Further Reading

This chapter deliberately keeps citations light in the body text. Full source material — official OpenTelemetry/Prometheus/Grafana/vLLM documentation links, GitHub repositories (with verified star counts), books, and companion videos — is catalogued in this same folder:

- `references.md` — official docs, specs, and engineering blogs
- `videos.md` — companion video material
- `books.md` — book-length treatments (notably *Learning OpenTelemetry* and Chip Huyen's *AI Engineering*)
- `github.md` — key open-source repositories (OpenTelemetry, vLLM, Prometheus, Grafana, Langfuse, and more)

See `architecture.md` in this folder for deeper architecture diagrams, a full request-lifecycle sequence diagram, and a tool-selection decision tree.
