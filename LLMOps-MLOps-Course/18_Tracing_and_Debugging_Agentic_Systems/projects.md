# Module 18 — Projects: Tracing and Debugging Agentic Systems

Three projects, increasing in scope. Each names what it demonstrates so you (or a reviewer) can judge completion against a concrete capability, not just "it runs."

---

## Project 1 (Mini): Traced RAG Debugger for a Single Document Set

**Scope:** Build a small RAG pipeline (10-30 short documents on a topic of your choice — product docs, a handful of Wikipedia excerpts, your own notes) with real retrieval (any simple vector store: FAISS, Chroma, or even brute-force cosine similarity over embeddings) and a real LLM call for generation. Instrument it with OpenTelemetry-compatible spans per §3.3 and run it inside Arize Phoenix (§3.4), with at least: a `RETRIEVER` span (doc count, top score), an `LLM` span (token usage), and one fail-fast guardrail (§3.2) that returns `INSUFFICIENT_CONTEXT` when the top retrieval score is below a threshold you pick and document. Run at least 15 test queries spanning: clearly in-scope, borderline, and clearly out-of-scope questions.

**What it demonstrates:** You can take an uninstrumented pipeline from zero to a fully traceable one using the small-setup-cost pattern the tutorial emphasizes, correctly separate a retrieval-caused failure from a generation-caused one by reading the waterfall (not by guessing), and implement a guardrail that measurably prevents at least one class of bad answer from reaching a "user" in your test set.

**Suggested acceptance check:** For at least 2 of your out-of-scope test queries, the trace shows the guardrail firing (no downstream `generate_answer` span, or a clearly flagged short-circuit) — and you can produce the Phoenix UI screenshot or exported trace as evidence, not just a claim that it works.

---

## Project 2 (Medium): Root-Cause Clustering Dashboard Across a Larger Query Log

**Scope:** Extend Project 1's pipeline (or build a fresh one) and generate/collect a larger query log — at least 100 queries, a mix of synthetic and, if feasible, real ones from a public FAQ/support dataset. Run all of them through your traced, guarded pipeline, score each response with an LLM judge (reuse Module 11's approach, or Phoenix's/Ragas's built-in evaluators), and implement the full §3.6 filter → tag → cluster → prioritize pipeline. Build a small dashboard (a Jupyter notebook with charts is sufficient, or a lightweight Streamlit/Gradio app if you want a UI) that shows: overall failure rate, cluster sizes/shares (retrieval-below-floor, context-truncated, out-of-scope, uncategorized), and — critically — the ability to click/filter into one cluster and see 3-5 representative trace excerpts for it.

**What it demonstrates:** You can operationalize root-cause clustering as a repeatable analysis over a realistically-sized query log rather than a one-off inspection of a handful of examples, correctly prioritize a fix by cluster size with data to back the decision, and present the analysis in a form a teammate could use to decide what to fix next without re-deriving your methodology from scratch.

**Suggested acceptance check:** Your dashboard/notebook, run on your 100+ query log, produces a cluster table whose largest cluster you can trace back to at least 5 individual full traces (trace_id and all), and a one-paragraph written recommendation whose priority order is directly justified by the cluster-share numbers you computed (not by which failure you noticed first).

---

## Project 3 (Production-Grade): Guarded, Traced, Continuously-Monitored Agentic Pipeline with Tool Use

**Scope:** Build a small but genuinely agentic pipeline — not just RAG, but an agent that can choose between at least 2 tools (e.g., a document-retrieval tool and a second tool such as a calculator, a mock order-lookup API, or a web-search-like function) and decides at runtime which to call, potentially looping. Fully instrument it end to end: every tool call, every retrieval, every generation step as its own span, with a `CHAIN`/`AGENT`-level root span carrying overall outcome attributes. Implement guardrails for at least three of the six §3.1 failure types (e.g., retrieval fail-fast, groundedness check, and a tool-error/loop-detection guardrail that caps repeated identical tool calls). Wire in a recurring (can be simulated as "run this script on a schedule" rather than a literal cron job) root-cause clustering report per §3.6's `weekly_hallucination_trigger_report` pattern, and produce a short comparison write-up (§3.5-style) of at least two tracing tools evaluated against your actual pipeline, with a final tool recommendation and justification. Finish with a written incident-response walkthrough (§3.7) for one deliberately-introduced regression, from detection through fix through an added guardrail/monitoring signature.

**What it demonstrates:** End-to-end production readiness for observability of an agentic (not just RAG) system — covering all six failure types with real evidence for each, a functioning guardrail layer that measurably changes pipeline behavior on failure, a monitoring signal that would catch a regression trending over time rather than only a single bad request, a defensible, evidence-based tool choice between real tracing vendors, and a documented incident-response trail that could be handed to a new team member as a runbook.

**Suggested acceptance check:** A reviewer can pick any one of your three implemented guardrails, deliberately construct an input that should trigger it, run it, and see both (a) the guardrail's effect on the returned answer and (b) the corresponding span attribute recording that it fired — and your written incident walkthrough for the deliberately-introduced regression includes an actual before/after cluster-report diff showing the fix reduced the targeted cluster's size or share.
