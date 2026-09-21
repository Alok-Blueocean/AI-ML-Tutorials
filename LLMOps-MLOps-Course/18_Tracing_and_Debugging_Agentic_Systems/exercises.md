# Module 18 — Exercises: Tracing and Debugging Agentic Systems

These exercises build on `tutorial.md`. Work through them in order — each assumes the artifacts (code, traces, datasets) produced by the previous one still exist. Use any LLM provider you have API access to; where a specific model is shown it is illustrative, not mandatory.

---

## Exercise 1 (Beginner) — Classify Failures by Taxonomy

**Goal:** Build the muscle memory of separating "wrong answer" into one of the six §3.1 failure types before touching any code.

**Task:** You are given five short incident descriptions (write these yourself, or use the ones below as a starting seed):

1. A support bot answers a refund question using shipping-policy language.
2. An agent calls a weather API, gets a 500 error, and answers "it's sunny" anyway.
3. A coding agent re-reads the same file six times before writing any code.
4. A user asks a 2,000-word question; the model answers only the first paragraph's topic.
5. A user asks about a competitor's product the system was never given documents about, and the bot answers confidently anyway.

For each, write down: (a) which of the six failure types (retrieval / context / generation / tool / reasoning-decision / cascade) it is, (b) one sentence justifying the choice, (c) what evidence you would need from a trace to *confirm* your classification rather than guess it.

**Done looks like:** A short table (markdown or a text file) with columns `incident | failure_type | justification | evidence_needed`, all five rows filled in with no two justifications interchangeable (i.e., each justification should only make sense for its assigned type).

---

## Exercise 2 (Beginner) — Instrument a Toy RAG Pipeline with Manual Spans

**Goal:** Practice turning an uninstrumented function pipeline into a traced one using raw OpenTelemetry, before relying on any vendor's auto-instrumentation.

**Task:** Write a toy RAG pipeline (a fake `retriever.search()` that returns hardcoded documents + scores from a small in-memory list, and a fake `llm.generate()` that just concatenates/echoes for this exercise — no real API calls needed) with three functions: `retrieve_docs`, `build_context`, `generate_answer`. Instrument each as its own OpenTelemetry span (see tutorial §3.3) with at least these attributes: `retrieval.doc_count`, `retrieval.top_score`, `context.token_estimate`, `gen_ai.usage.output_tokens` (a rough estimate is fine). Export spans to a `ConsoleSpanExporter` and run the pipeline for 3 different fake queries, at least one of which should have a top retrieval score below 0.6.

**Done looks like:** Running your script prints three distinct trace trees to the console, each showing three nested/sequential spans with the required attributes populated with real (non-placeholder) values, and you can point to the console output line that shows the low-retrieval-score query.

---

## Exercise 3 (Intermediate) — Stand Up Phoenix and Auto-Instrument a Real LLM Call

**Goal:** Get hands-on with Arize Phoenix's actual UI and auto-instrumentation, not just the manual OTel API.

**Task:** `pip install arize-phoenix openinference-instrumentation-openai` (or the equivalent instrumentor for whichever provider you use). Launch a local Phoenix session (`phoenix.launch_app()`), instrument your LLM client library, and run at least 10 real (or realistic simulated) queries through a small RAG pipeline — reuse or extend Exercise 2's pipeline but with real retrieval (a small local vector store over ~20 short documents is enough) and a real LLM call. Open the Phoenix UI and inspect the trace waterfall for at least 3 of the 10 requests.

**Done looks like:** A screenshot or short written description of the Phoenix waterfall UI showing at least two nested spans per trace (retrieval + generation) with real latency/token numbers, plus a one-paragraph note identifying which of your 10 queries had the lowest retrieval score and whether its generated answer looked, on inspection, less grounded than the others.

---

## Exercise 4 (Intermediate) — Add Custom Spans and a Guardrail Checkpoint

**Goal:** Combine §3.2 (guardrails) and §3.3/3.4 (tracing) into one traced, guarded pipeline.

**Task:** Extend Exercise 3's pipeline with: (a) a custom span (decorator or context manager, your choice) around a groundedness check that runs after generation and before returning the answer, (b) a fail-fast guardrail that returns a clearly labeled `INSUFFICIENT_CONTEXT` response when the top retrieval score is below a threshold you choose and document, and (c) span attributes recording every guardrail's pass/fail outcome (`guardrail.grounded: bool`, `retrieval.below_floor: bool`). Deliberately construct at least 2 of your 10 test queries to trigger each guardrail (e.g., an out-of-corpus query to trigger the retrieval guardrail).

**Done looks like:** Running your 10 queries produces at least 2 traces where `retrieval.below_floor=True` and the pipeline returned `INSUFFICIENT_CONTEXT` without calling the LLM at all (verify this by checking there's no `generate_answer` span, or that it's absent/short-circuited in your trace), and at least 1 trace where the groundedness guardrail fired and you can see the retry/fallback behavior in the span attributes.

---

## Exercise 5 (Intermediate) — Filter and Export Traces Programmatically

**Goal:** Move from "look at the UI" to "query the trace data" — the actual mechanism root-cause clustering depends on.

**Task:** Using the Phoenix Python client (`phoenix.Client().get_spans_dataframe(...)`), pull all spans from Exercise 4's run into a pandas DataFrame. Filter to retrieval spans with `top_score < 0.6`, then join back to the full traces for those trace IDs (i.e., pull every span belonging to those specific trace_ids, not just the retrieval span). Print a summary: how many of your 10 queries fall in this risky set, and for each, whether the guardrail fired as expected.

**Done looks like:** A runnable script that prints a DataFrame (or equivalent table) of exactly the risky trace_ids, their retrieval scores, and a boolean column showing whether the fail-fast guardrail correctly caught each one — with at least one deliberately-planted case where it did, confirmed by the data, not by memory of what you expected.

---

## Exercise 6 (Advanced) — Build a Root-Cause Clustering Report

**Goal:** Implement the full §3.6 filter → cluster → prioritize methodology end to end on a larger, more realistic failure set.

**Task:** Generate (synthetically is fine, but make them varied and realistic) a set of at least 40 "failing" query records, each with fields: `query`, `top_retrieval_score`, `query_tokens`, `context_truncated` (bool), `out_of_scope` (bool), `judge_score` (1-5). Distribute the causes roughly like the tutorial's worked example (majority low-retrieval-score, a smaller cluster of long-query truncation, a small out-of-scope cluster, and a handful of genuinely uncategorized ones). Implement the `tag_failure` and `cluster_failures` functions from tutorial §3.6, run them, and produce a cluster-size report (counts and shares). Write a short prioritized remediation plan (3-5 sentences) based purely on what your cluster report shows — not on which failure "feels" most important.

**Done looks like:** A printed/exported cluster table with counts and shares per `failure_tag`, plus a short written remediation plan whose priority order matches the cluster-size order from your own data (i.e., you fix the largest cluster first, and you can defend that from the numbers, not intuition).

---

## Exercise 7 (Advanced) — Compare Two Tracing Tools Hands-On

**Goal:** Move the Phoenix vs. LangSmith vs. Langfuse comparison in §3.5 from "read a table" to "verified by running both."

**Task:** Pick Phoenix plus one of LangSmith or Langfuse. Instrument the *same* small pipeline (reuse Exercise 3/4's pipeline) with both tools side by side (this may mean running two separate instrumentation passes, since some instrumentors can conflict if active simultaneously — sequence them if needed). Run the same 10 queries through both. Write a comparison covering: setup friction (lines of code / config needed), what the waterfall view showed you in each, what filtering/querying capability each exposed, and which one you'd recommend for this specific toy pipeline and why — tying your recommendation back to the concrete things you observed, not just the tutorial's table.

**Done looks like:** A short written comparison (a table or a few paragraphs) that cites at least one concrete difference you *personally observed* running both tools (not just repeating the tutorial's comparison table), plus a one-sentence recommendation with a stated reason.

---

## Exercise 8 (Production-Level) — End-to-End Incident Simulation and Postmortem

**Goal:** Simulate a full production incident using the §3.7 debugging methodology, from "something's wrong" to a shipped fix and a hardened guardrail/monitoring signature.

**Task:** Take your traced, guarded pipeline from Exercise 4-6. Deliberately introduce a regression (e.g., silently lower your retrieval index's quality by removing/corrupting a subset of documents, or shrink your context-token budget so truncation starts happening more often) without telling yourself in advance which failure type it will produce. Run your test query set through the pipeline, then work the §3.7 methodology step by step as if this were a live incident: reproduce with tracing on, read the waterfall, classify via the taxonomy, check whether an existing guardrail should have caught it, query for the cluster size, prioritize, ship a fix, and add/tighten a guardrail plus a monitoring signature (e.g., extend your weekly cluster report function from the tutorial to flag this specific new signature if its share crosses a threshold).

**Done looks like:** A short written postmortem (half a page to a page) following the §3.7 steps as section headers, with each section backed by an actual artifact from your run (a trace excerpt, a cluster count, a diff of the fix, the new guardrail code, the monitoring check) — not a hypothetical description, an actual reproducible incident-to-fix trail with code and data to back every claim.
