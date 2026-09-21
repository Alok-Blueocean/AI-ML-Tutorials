# Observability, OpenTelemetry, and Production Monitoring — Scenario-Based Q&A

**Situation:** Your LLM support bot returns HTTP 200 for every request, and your uptime dashboard shows 100% availability all week — yet customer complaints about wrong answers are climbing. What's the gap in your monitoring, and how do you close it?

Model answer: Explain that this is the core way LLM observability differs from classical service observability — a classical service either returns a prediction or throws a 500, but an LLM can return a fluent, confident, completely wrong answer with a perfectly healthy HTTP 200 and no metric obviously moving. Uptime and error-rate monitoring alone can't see this; you need a content-quality signal layered on top, typically an LLM-as-judge (Module 11) run asynchronously on a sampled slice of live production traffic, with its output tracked as its own metric alongside uptime and latency. Close the gap by adding that sampled judge signal to the dashboard specifically, not by tuning the existing infrastructure metrics further.

---

**Situation:** A new engineer's first instinct when debugging a slow request is to add `print()` statements logging the full request and response at every step of the pipeline, and ships it to production "just for now." What's wrong with this, and what should they do instead?

Model answer: Flag two separate problems. First, this is the log-everything anti-pattern — unstructured prints scattered through a pipeline create noise that's hard to query and easy to forget about, rather than a queryable, structured signal. Second, and more urgent, logging full raw request/response content by default is a privacy and compliance risk, since an LLM's input is often literally what the user typed and can contain PII. Replace it with proper OpenTelemetry instrumentation: a trace with nested spans for each pipeline stage (retrieval, generation, tool calls) carrying timing and metadata attributes, with raw content excluded by default and only captured behind an explicit, policy-approved debug flag for a bounded window.

---

**Situation:** You're asked to instrument a FastAPI-based RAG service so an engineer can answer "what exactly happened for this one slow request" without guessing. What do you build?

Model answer: Build a trace per request with nested spans for each meaningful stage — a `retrieve_context` span, a `call_llm` span, and a `invoke_tool` span if tools were used — each carrying start time, duration, and relevant attributes using OpenTelemetry's `gen_ai.*` semantic conventions (e.g., `gen_ai.request.model`, `gen_ai.usage.input_tokens`) so the span means the same thing to any backend that consumes it. This lets an engineer open the exact trace ID for the slow request and see, at a glance, whether the time went into retrieval, generation, or a tool call, instead of grepping through unstructured logs trying to reconstruct the timeline by hand.

---

**Situation:** Legal flags that your telemetry pipeline has been capturing full user queries and full model responses in every trace span's attributes, and this has been running for four months. What immediate and structural fixes do you make?

Model answer: Immediately stop capturing raw content by default and treat this as a live compliance incident, not a backlog item — assess with legal/security what retention and deletion obligations apply to the four months of already-captured data. Structurally, redesign the telemetry schema around privacy-safe defaults: hash user and session identifiers instead of logging them raw, version models and prompts by id rather than logging their content, and default every span/log to metadata-only (token counts, latency, model version, score) with raw-content capture requiring an explicit, policy-approved, time-bounded flag rather than being the default behavior anyone can trigger by writing an instrumentation line.

---

**Situation:** Your team wants to track P95 latency for LLM responses, and an engineer implements it by storing the latency of the most recent request in a variable and displaying that. What's wrong with this metric design, and what's the fix?

Model answer: A single most-recent value can't produce a percentile — P95 requires a distribution of observations over a time window, not one point-in-time reading. Use a histogram metric type, which buckets observed values so percentile queries can be computed over any time range, and choose bucket boundaries from real observed traffic rather than guessing, so the buckets actually bracket the SLA-relevant range. This is also the right moment to check that counters (error count, cache hits) are implemented as ever-increasing counters with ratios computed at query time, and that only genuinely point-in-time values (like current queue depth) use a gauge — using the wrong metric type for each signal is a common instrumentation mistake.

---

**Situation:** A stakeholder asks for a Grafana dashboard "with everything on it" for the new LLM feature, and by the time it's built it has 25 panels. Nobody looks at it during the next incident because nobody can find the relevant panel fast enough. What would you redesign?

Model answer: Redesign around the small set of signals that actually reveal user experience, cost, and reliability — TTFT and P95/P99 latency for perceived responsiveness, error rate and cache-hit rate for reliability and efficiency, and cost per query for budget — rather than a comprehensive wall of every metric the system emits. Move everything else to a secondary drill-down dashboard reachable from the primary one, so the primary screen answers "is this healthy right now" in a glance during an incident, which is the actual job a dashboard needs to do under pressure — more panels is not more signal, it's slower triage.

---

**Situation:** Your on-call engineer has started silencing a specific alert every time it fires because "it's usually nothing" and investigating takes too long to figure out what to even check. What's the underlying problem, and how do you fix it?

Model answer: This is a runbook gap producing alert fatigue, not an alerting-threshold problem by itself — if an alert fires and the engineer doesn't know what to check, silencing it is the rational response to an unclear signal, and it will eventually cause a real incident to be dismissed too. Pair every alert with a written runbook entry: what the alert means, what to check first, and what action to take, specific enough that a 2 a.m. on-call engineer with no context on this system can follow it. Separately, verify the threshold itself is calibrated against real historical data — if it's also firing too often on genuinely benign fluctuation, tighten the threshold or `for` duration alongside adding the runbook.

---

**Situation:** During an incident, an engineer pulls up the trace for a slow request and sees a `call_llm` span with no parent-child relationship to the `retrieve_context` span that ran just before it in the same request — the trace looks like two disconnected fragments. What's the likely cause, and how do you fix it?

Model answer: This is a missing trace-propagation bug — somewhere between the retrieval call and the generation call, the trace context (trace ID and parent span ID) wasn't correctly passed forward, likely because a new span was started without linking it to the active trace context, or because the retrieval and generation steps cross a service or async boundary that dropped the context. Fix it by explicitly propagating the OpenTelemetry context across any async task, thread, or service boundary in the pipeline, and add a regression check (or code review checklist item) that verifies new spans in the request-handling code are always created as children of the active trace context, since this specific bug tends to reappear whenever new pipeline stages are added.

---

**Situation:** A finance stakeholder asks for real-time visibility into per-query cost so they can catch a cost spike before it becomes a large bill, but your team currently only reconciles LLM API costs from the monthly provider invoice. What do you build?

Model answer: Build a cost-per-query metric computed at request time from the token usage reported by the model API response and the known per-token pricing for the model/version in use, exported as a counter accumulated over rolling windows (e.g., cost accumulated in the trailing hour) so a spike is visible on the dashboard in near real time rather than a month later on an invoice. Attach a Prometheus alert rule on that accumulated-cost metric with a threshold tuned to expected traffic, so an anomalous spike (a runaway retry loop, a prompt regression causing much longer outputs) pages someone within the hour instead of surfacing as a surprising invoice line weeks later.

---

**Situation:** Your team is deciding whether to build LLM observability entirely on raw OpenTelemetry plus Grafana/Prometheus, or adopt a purpose-built tool like Langfuse or LangSmith. How do you frame the tradeoff?

Model answer: Frame it as vendor-neutral flexibility versus LLM-specific convenience. Raw OpenTelemetry with Grafana/Prometheus is vendor-neutral, "instrument once, export anywhere," and integrates with infrastructure the team may already run for other services, but requires building LLM-specific views (prompt/response inspection, conversation-level grouping, judge-score correlation) yourself. Langfuse and LangSmith come with LLM-specific features out of the box — prompt playgrounds, conversation trace views, built-in eval integrations — at the cost of tighter coupling to that vendor's ecosystem and, for LangSmith, tighter coupling to the LangChain ecosystem specifically. A common resolution: use OpenTelemetry as the underlying instrumentation layer (so you're not locked in) and point it at whichever backend, including a purpose-built LLM tool, fits the team's current needs — since OTel's whole design goal is exactly this kind of backend portability.

---

**Situation:** A cache-hit rate dashboard panel shows 45%, and a stakeholder asks whether that's good or bad. How do you actually answer that, versus just reciting the number?

Model answer: Explain that a raw percentage in isolation isn't answerable as good or bad without context — the meaningful question is what it's costing and what it's trending. Pair it with the cost and latency implication directly: at current traffic volume, what would cost and P95 latency look like if the hit rate were 10 points higher, and is that achievable (e.g., via longer prefix reuse, better key design) without a quality tradeoff. Also check the trend, not just the point value — a cache-hit rate that's been declining for two weeks is a different, more urgent story than one that's been stable at 45% for months, even though the current snapshot looks identical in both cases.

---

**Situation:** You're asked to design the observability schema for a new multi-step agentic feature that calls three different tools before producing a final answer. A colleague suggests one big span for the whole request is sufficient. What do you push back on?

Model answer: Push back because a single span for the entire request collapses exactly the information an engineer needs during debugging — which of the three tool calls was slow, which one failed, and what the LLM decided to do in response. Design nested spans mirroring the actual execution structure: a root span for the request, with child spans for each tool invocation and for the generation steps between them, each carrying attributes like tool name, arguments (redacted if sensitive), duration, and success/failure. This turns "the request was slow" into "tool call #2 took 3.8 seconds and the model retried it once," which is the level of detail an on-call engineer actually needs at 2 a.m.

---

**Situation:** Your team notices error rate has been flat at a low, acceptable level according to the dashboard, but a manual review of a sample of "successful" responses finds several that returned an empty or truncated answer due to a silent downstream timeout that was caught and swallowed by an exception handler. What's the instrumentation gap?

Model answer: The error-rate metric is only counting hard failures (exceptions that propagate and result in a non-2xx response), missing soft failures where an exception was caught, handled gracefully at the HTTP layer, but still produced a degraded response the user experienced as broken. Add a distinct metric or span attribute for "degraded response" cases — timeouts, retries exhausted, partial/truncated output — logged explicitly at the point where the exception handler decides to return a fallback, so these cases are visible in aggregate on the dashboard instead of silently counted as successes. This is a common gap: a dashboard is only as honest as what the instrumentation chose to count.

---

**Situation:** In a system design interview, you're asked "how would you design observability for a new production LLM service from scratch?" How do you structure your answer?

Model answer: Structure it around the three OpenTelemetry signal types mapped onto the five questions a production team must always be able to answer. Traces with nested spans (retrieval, generation, tool calls) using `gen_ai.*` semantic conventions answer "what exactly happened for this one request." Metrics — histograms for TTFT/TPOT/P95/P99 latency, counters for errors and cache hits, gauges for point-in-time state — answer "is the system healthy right now in aggregate." Privacy-safe, metadata-only-by-default logs answer "why did this specific thing go wrong" without becoming a compliance risk. Alert rules with calibrated thresholds and `for` durations, each paired with a runbook, answer "who needs to act and what should they do." And a periodically-sampled LLM-judge signal on production traffic answers the question classical observability structurally can't: "is the content itself actually behaving," since an LLM can fail silently behind a healthy HTTP 200. Close by naming the design principle underneath all five: privacy design isn't a bolt-on here, it is the observability design, because LLM inputs are often literally what the user typed.
