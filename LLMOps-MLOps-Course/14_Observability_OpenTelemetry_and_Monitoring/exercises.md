# Module 14 — Exercises: Observability, OpenTelemetry, and Production Monitoring

These exercises progress from "print a single trace to your console" to "prove your entire
alert-to-page pipeline actually works, with a stopwatch." Each one builds on artifacts from the
previous one where noted. Use the code in `tutorial.md` (`build_telemetry`, the FastAPI OTel
instrumentation, the Prometheus metric definitions, the Grafana JSON panel, the alerting-rules YAML)
as your starting scaffold rather than writing everything from scratch — this module is about wiring
traces, metrics, dashboards, and alerts into one coherent, privacy-safe system, not about
re-deriving OpenTelemetry or PromQL from first principles.

You do not need real GPU access or a paid LLM API to do any of this. A function that
`time.sleep(random.uniform(0.1, 0.8))` and returns a canned string is a perfectly legitimate stand-in
for "the LLM call" throughout — the point of every exercise is the observability wiring around that
call, not the call itself. Where an exercise says "chat" or "LLM call," read it as "your real or
mocked model call, your choice."

---

## Exercise 1 — Trace your first LLM call with OpenTelemetry (Beginner)

**Goal.** Instrument a minimal request handler with a root span and one nested `chat` child span,
export to the console (no Collector needed yet), and read the resulting trace with your own eyes.

**Task.**
1. `pip install opentelemetry-sdk opentelemetry-exporter-otlp` (the console exporter ships with the
   SDK, no extra package needed).
2. Set up a `TracerProvider` with a `Resource` carrying `service.name`, and a `SimpleSpanProcessor`
   wrapping `ConsoleSpanExporter` (see `tutorial.md` §3.2 for the provider-setup shape; swap the
   `BatchSpanProcessor`/OTLP exporter shown there for `SimpleSpanProcessor`/`ConsoleSpanExporter` for
   this exercise only — batching hides output until flush, which defeats the point here).
3. Write `handle_request(query, model_version)`: a root span named `handle_request`, containing a
   nested `chat` span. The `chat` span should set `gen_ai.request.model`, sleep a random 200-800ms to
   simulate generation, and set `gen_ai.usage.output_tokens` to a random int before exiting.
4. Run it 5 times and capture the printed span dumps.

**Done looks like:**
- All 5 runs print two span records each, and in every run the parent (`handle_request`) and child
  (`chat`) span share the exact same `trace_id` while having two different `span_id` values, with the
  child's `parent_span_id` equal to the parent's `span_id`.
- The 5 runs show 5 different durations for the `chat` span, tracking your randomized sleep — proving
  you're reading real timing data, not a fixed stub.
- You can explain in one sentence why this is meaningfully different from wrapping the same code in
  `print(f"start chat at {time.time()}")` / `print(f"end chat at {time.time()}")` — specifically, what
  a trace gives you (structured parent/child linkage, a shared ID across the whole tree, machine-
  queryable attributes) that print statements do not.

---

## Exercise 2 — Build the privacy-safe telemetry record and correlate it with the trace (Beginner/Intermediate)

**Goal.** Implement the privacy boundary from `tutorial.md` §3.1 — hash the user identifier, never
log raw query text — and emit the result as a log record that inherits the enclosing span's
`trace_id`, so a debugging engineer can pivot from "this log line" to "the exact trace it came from."

**Task.**
1. Implement `hash_user_id(raw_user_id)` using SHA-256 plus a server-side pepper string (a local
   constant is fine for this exercise; note in a comment that it belongs in a secrets manager in
   production).
2. Implement a deliberately simple `pii_scan(query) -> bool` that flags obvious patterns (an email
   regex, a 9-digit run that could be an SSN-shaped string). It doesn't need to be good — it needs to
   exist and return a boolean.
3. Build a telemetry dict following the `build_telemetry()` shape in `tutorial.md` §3.1: include
   `query_len_chars` and `pii_detected`, and deliberately do **not** include `raw_query` anywhere in
   the dict.
4. Emit the record via `opentelemetry._logs.get_logger(...).emit(...)` from *inside* the `chat` span
   from Exercise 1, so the SDK attaches the active span's `trace_id`/`span_id` automatically.
5. Now deliberately do the wrong thing once: add `"raw_query": raw_query` as an attribute, run it, look
   at the emitted record, then delete that line and write a one-sentence comment directly above where
   it was, explaining specifically why it would have been a live incident in production (not just "bad
   practice" — name what could leak and to whom).

**Done looks like:**
- Your emitted log record's `trace_id` is byte-for-byte identical to the enclosing `chat` span's
  `trace_id` from Exercise 1 — you've verified the correlation, not just assumed it.
- Two different `raw_user_id` values produce two different, unrecoverable-looking hashes — you
  attempted to eyeball-reverse one hash back to a plausible user ID and confirmed you can't.
- No raw query text appears anywhere in your final (corrected) emitted output, verified by grepping
  your captured output for a distinctive substring of a query you actually sent.
- The comment you wrote about the deliberately-wrong `raw_query` line names a concrete, specific harm
  (e.g., "this log sink is queried by a broader analytics team who never agreed to see user PII, and
  this pipeline has 30-day retention with no additional access control").

---

## Exercise 3 — Instrument a RAG pipeline with the full span hierarchy (Intermediate)

**Goal.** Build the `retrieve_context` → `rerank` → `chat` → `execute_tool` span tree from
`tutorial.md` §3.3, export it through a real backend (Jaeger or Tempo), and practice reading a trace
waterfall to answer "where did the time go" without reading any code.

**Task.**
1. Run Jaeger all-in-one locally via Docker (`docker run -p 16686:16686 -p 4317:4317
   jaegertracing/all-in-one`, or an OTel Collector in front of Tempo if you prefer that path).
2. Point your `OTLPSpanExporter` at it and switch to a `BatchSpanProcessor` (Exercise 1's
   `SimpleSpanProcessor` was only for the console-exporter case).
3. Instrument, nested exactly as in the tutorial's architecture diagram: `retrieve_context` (mock
   returning a random doc count and a fake top similarity score as attributes), `rerank` (mock,
   randomized duration), `chat` (mock LLM call with token-count attributes), and — for roughly half
   your simulated requests — a nested `execute_tool: <name>` span under `chat`, followed by a second
   `chat` span for the "follow-up with tool result" case, matching the agentic-loop shape in §3.3.
4. Fire 10 requests with intentionally varied simulated latency: make 3 of them have a slow
   `retrieve_context` (800ms+), make 3 have a slow second `chat` call (1500ms+ output), and let the
   rest be unremarkable.
5. Open the Jaeger UI, find your 10 traces, and — using only the waterfall view, no code-reading —
   identify which 3 have retrieval as the dominant cost and which 3 have the second `chat` call as the
   dominant cost.

**Done looks like:**
- Jaeger shows 10 traces, each with 4-5 correctly nested spans matching the tutorial's tree shape
  (screenshots or a written span list per trace are both acceptable evidence).
- You correctly identify, from the UI alone, all 3 retrieval-dominant traces and all 3
  second-`chat`-dominant traces — cross-checked against which ones you actually built to be slow in
  step 4.
- Every `chat` span, visible in the Jaeger span-detail panel, carries `gen_ai.request.model` and at
  least one `gen_ai.usage.*` attribute.
- You can state, in under a minute, the same style of diagnosis the tutorial models in §3.3 ("retrieval
  and reranking together cost X ms — fine; the second model call dominated total latency because of Y
  output tokens — the fix would be about Z, not retrieval").

---

## Exercise 4 — Design the core Prometheus metrics, then trigger and fix a cardinality bug (Intermediate)

**Goal.** Implement the histogram/counter/gauge metric set from `tutorial.md` §3.4, scrape it with a
real Prometheus instance, compute P95/P99 via PromQL — then deliberately cause a cardinality
explosion and prove you can both detect and fix it.

**Task.**
1. Implement `TTFT_SECONDS` and `E2E_LATENCY_SECONDS` as histograms, `REQUESTS_TOTAL` and
   `CACHE_LOOKUPS_TOTAL` as counters, and `INFLIGHT_REQUESTS` as a gauge, matching the definitions in
   `tutorial.md` §3.4. Expose them at `/metrics` (e.g., via `prometheus_client.make_asgi_app()` mounted
   into your FastAPI app).
2. Write a local `prometheus.yml` scrape config targeting your app, and run Prometheus via Docker
   against it.
3. Write a small load generator that fires 200 requests with randomized simulated TTFT/latency, then
   run these two PromQL queries and record both results:
   - `histogram_quantile(0.95, sum(rate(llm_time_to_first_token_seconds_bucket[5m])) by (le))`
   - a naive average: `rate(llm_time_to_first_token_seconds_sum[5m]) / rate(llm_time_to_first_token_seconds_count[5m])`
4. Now add a high-cardinality label to `REQUESTS_TOTAL` — a per-request unique ID or a raw
   incrementing counter used as a label value — regenerate the same 200-request load, and check
   Prometheus's **Status → TSDB status** page for the "number of series."
5. Revert the bad label and confirm the series count returns to its small, bounded baseline.

**Done looks like:**
- Your two queries from step 3 return visibly different numbers (P95 noticeably higher than the
  average) — written down, not just eyeballed, so you have concrete before/after evidence that an
  average would have hidden the tail.
- After step 4, the TSDB status page shows a "number of series" roughly proportional to your request
  count (i.e., approaching 200 additional series from one metric), which you have recorded as a
  specific before/after number pair.
- After step 5, series count returns to its original small, bounded value.
- You can explain, in one sentence, why this is a production *outage* risk (not just messiness) at
  real request volumes — tie it explicitly to Common Mistake #3 in `tutorial.md` §5.

---

## Exercise 5 — Build the live-ops Grafana dashboard with matching thresholds (Intermediate/Advanced)

**Goal.** Build the six-panel "Live Ops" dashboard wireframe from `tutorial.md` §3.5 as a real,
provisionable Grafana dashboard wired to your Exercise 4 Prometheus instance, with thresholds you will
reuse, unchanged, when you write alert rules in Exercise 6.

**Task.**
1. Run Grafana via Docker and add your Prometheus instance as a data source.
2. Build the six panels from the wireframe: P95/P99 TTFT, P95/P99 E2E latency, error rate (red
   threshold line at whatever you choose — document the number), in-flight requests, cache-hit rate
   (threshold line at 20%), and cost accumulated over the last 1 hour.
3. Set each panel's threshold coloring to specific, written-down numeric values — reuse the tutorial's
   TTFT figures (800ms yellow / 1000ms red) as a starting point, adjusting only if you document why.
4. Export the dashboard's JSON model (Grafana's "Export" / "View JSON" feature) and check it into your
   project directory, so the dashboard is reproducible from a file, not only from manual clicking.
5. Re-run your Exercise 4 load generator, but inject a 2-minute burst where 10% of requests raise an
   exception, and watch the error-rate panel change color live.

**Done looks like:**
- A dashboard JSON file in your repo that, when provisioned fresh (e.g., after a
  `docker compose down && docker compose up`), reproduces the identical six-panel layout with no
  manual re-clicking.
- The error-rate panel visibly crosses into its warning/critical color band during the injected burst
  and returns to green within roughly one scrape interval after the burst ends.
- You can point to one single file or table where both this dashboard's thresholds and your
  (not-yet-written) Exercise 6 alert thresholds are defined, and explain why reading from one shared
  source matters more than getting any individual number "right."

---

## Exercise 6 — Write Prometheus alerting rules with correct `for` windows and a runbook (Advanced)

**Goal.** Implement the `HighTTFT` / `ErrorRateSpike` / `HighFallbackUsage`-style alerting rules from
`tutorial.md` §3.6, route them through Alertmanager to two genuinely different receivers, and write
one complete runbook entry.

**Task.**
1. Write `alerting_rules.yml` with at least 3 alerts (e.g., TTFT degradation, error-rate spike, and
   low-cache-hit-rate-as-info), each with an `expr`, a `for` duration, `labels` (`severity`, `team`),
   and `annotations` (`summary`, `runbook_url`) — following the exact shape shown in `tutorial.md`
   §3.6.
2. Configure Alertmanager with two distinct routes: `severity: warning` → one receiver (a real Slack
   incoming webhook, or a small local HTTP server that just logs what it receives), `severity:
   critical` → a *different* receiver. The point is proving differential routing works, not which
   specific tool receives it.
3. Extend your load generator to sustain P95 TTFT above your chosen threshold for longer than the
   `for` duration you configured, and confirm two distinct states in the Prometheus/Alertmanager UI:
   the alert sitting in **pending** while the condition is true but the `for` window hasn't elapsed
   yet, then transitioning to **firing** once it has.
4. Write one full runbook entry for your `ErrorRateSpike` alert, matching the `HighFallbackUsage`
   example format in `tutorial.md` §3.6 exactly (what it means, first 3 steps, an escalation
   condition).
5. Fire a single, isolated slow request (not sustained) and confirm no alert ever enters even the
   `pending` state — proving your `for` window is correctly filtering transient noise, not just
   sustained problems.

**Done looks like:**
- Captured evidence (screenshot or logged state transition) of your alert moving from absent → pending
  → firing, with timestamps showing the `pending` duration matches your configured `for` value.
- Your two receivers each show they received exactly the alert(s) routed to their severity and not the
  other's — a concrete demonstration, not an assumption from reading the YAML.
- A written runbook entry a colleague with zero context on your system could follow without asking you
  a clarifying question.
- A demonstrated single-blip request that produces no alert at all, with a one-sentence explanation of
  why that's the correct behavior, not a gap in your alerting.

---

## Exercise 7 — Diagnose a broken observability setup from symptoms only (Advanced, no code)

**Goal.** Practice the diagnostic reasoning a senior engineer needs when dashboards are green,
nothing has paged, and the system is still failing users — this exercise is a written analysis only,
mirroring `tutorial.md` Interview Question 8 and Common Mistake #10.

**Task.** You're told: *"Our LLM support bot has full Grafana dashboards — latency, cost, error rate,
all green — and no alerts have fired in three weeks. Yesterday we discovered it had been telling a
subset of users incorrect refund-policy information for those three weeks. We found out from a
support escalation, not from our dashboards."* Write a structured diagnosis covering:
1. Which specific Common Mistake from `tutorial.md` §5 this is a direct instance of, and, in your own
   words, why "all green" was fully *consistent* with this failure happening the entire time — i.e.,
   why none of latency, cost, or error rate could possibly have caught it.
2. One concrete new signal (not "add more logging" or "add more evals" as a vague answer) that would
   have caught this — specify exactly which pipeline stage or span it attaches to, what it measures,
   and what threshold/routing you would give the resulting alert.
3. Given the privacy-safe-by-default design in `tutorial.md` §3.1, explain how you would validate this
   new signal's design without introducing a raw-content-logging regression — i.e., what would and
   would not need to touch raw user query/response text to work.

**Done looks like:**
- Your diagnosis explicitly names Common Mistake #10 ("HTTP 200 does not imply the system behaved
  correctly") and explains, metric by metric, why latency/cost/error-rate are each structurally blind
  to this failure mode.
- Your proposed signal is specific enough that another engineer could implement it from your
  description alone (e.g., naming an intent classifier to select refund-policy-related responses, a
  sampling rate, a judge-model rubric check against the current policy document, and a rolling-window
  pass-rate threshold) rather than a generic "add evals."
- You've explicitly reasoned about which parts of your proposed signal require raw-content access
  (likely: yes, for the sampled judge check) and argued why that specific, narrow, sampled use case is
  the kind of audited opt-in the tutorial describes as acceptable, rather than a blanket logging change.

---

## Exercise 8 — Capstone: full agentic-service observability, blackbox-tested end to end (Capstone)

**Goal.** Combine Exercises 1-6 into one coherent service and prove the *entire* pipeline — trace →
metric → dashboard → alert → page — actually works, by injecting a failure and timing how long it
takes to reach a receiver, per the "monitoring the monitors" guidance in `tutorial.md` §6.

**Task.**
1. Build one FastAPI service simulating a RAG-plus-tool-calling agent, fully instrumented: the full
   OTel span tree from Exercise 3, the privacy-safe telemetry from Exercise 2, and the full Prometheus
   metric set from Exercise 4.
2. Run an OTel Collector, Jaeger/Tempo, Prometheus, Grafana, and Alertmanager together as one stack
   (a single `docker-compose.yml` is the natural shape for this).
3. Build both the live-ops dashboard (Exercise 5) and a second, weekly-trend dashboard (per the
   `tutorial.md` §3.5 audience/time-horizon table) with at least cost-per-query and cache-hit-rate
   trend panels, populated with several days of synthetic data (you can fabricate a plausible
   multi-day series with one deliberate dip and recovery).
4. Wire at least 3 alerts with correct severities, `for` windows, and routing (Exercise 6), each with a
   runbook file.
5. Run one synthetic blackbox test: inject a known failure (e.g., force a 15% error rate for 6
   minutes) end to end, and record — with real timestamps, not an estimate — exactly how long it takes
   from "failure injected" to "message received at your mock Slack/PagerDuty receiver."
6. Write a one-page, postmortem-style document as if this were a real production incident: what fired,
   when, what the runbook said to do, whether the on-call steps actually resolved anything meaningful
   in your simulated scenario, and one concrete recommendation for tightening the pipeline further
   (e.g., a tighter `for` window, or the missing content-quality signal from Exercise 7).

**Done looks like:**
- The full stack comes up from one command (or a short, documented sequence) and all five components
  (Collector, trace backend, Prometheus, Grafana, Alertmanager) are reachable and wired to each other.
- You have one recorded, timestamped time-to-page number for your injected failure.
- Both dashboards are populated with real or realistically synthetic data, and their thresholds
  visually agree with your alert thresholds (the same numbers, read from the same source).
- The postmortem document is detailed enough that a manager reading it would conclude you can own an
  observability stack end to end — not just its individual pieces in isolation.
