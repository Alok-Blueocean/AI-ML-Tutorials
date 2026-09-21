# Tracing and Debugging Agentic Systems — Scenario-Based Q&A

**Situation:** A support bot confidently gives a wrong answer about a billing policy, and your engineering lead's first reaction is "we need a better prompt or a bigger model." What would you do and why?

Model answer: Resist prompt-tweaking as the first move and pull the trace instead — a wrong final answer is usually a step failure wearing a final-answer costume, and generation is only the fourth place in the pipeline a failure can originate even though it's the first place a human notices. Check the retrieval span's top similarity score before touching the prompt at all; if it's below your floor (e.g., 0.43 against a 0.6 threshold), the model faithfully summarized irrelevant context, which is a retrieval bug, not a generation bug. Only after ruling out retrieval and context-assembly failures does it make sense to look at the generation step itself.

---

**Situation:** Your team ships a RAG pipeline that only logs `(user_query, final_answer)` pairs, and a production incident takes four hours to diagnose because nobody can tell which step actually broke. What would you do and why?

Model answer: Name this as the "invisible middle" problem and fix it architecturally, not by staring harder at transcripts. Instrument every decision point — retrieval, re-ranking, context assembly, generation, each tool call — as its own named, timed span with meaningful attributes (document count, top similarity score, token counts, tool status), before the next incident, not during it. A single span wrapping the whole handler recreates the invisible middle inside the tracing tool, so insist on one span per decision point, not one span per request.

---

**Situation:** After adding OpenTelemetry tracing, an engineer wraps the entire request handler in one span called `handle_request` and calls the pipeline "fully traced." What would you do and why?

Model answer: Push back — this gives you total latency and nothing else, and it's arguably worse than no tracing because it looks like observability was added when it wasn't. Require one span per decision point in the failure taxonomy (retrieval, re-ranking, generation, each tool call), each carrying attributes specific to what it did (retrieval needs doc count and top score; generation needs token counts and model name; a tool call needs status and error type). A trace waterfall is only as useful as the granularity and honesty of its spans — a span with a name but no useful attributes tells you an operation happened, not whether it succeeded.

---

**Situation:** A retrieval-augmented pipeline logs show a tool call to an order-lookup API returned HTTP 200, but the agent's final answer was still wrong. What would you do and why?

Model answer: Don't stop at "the tool call succeeded" — an HTTP 200 only confirms the call didn't error, not that its content was what the agent's next step actually needed. Inspect the tool span's actual response payload alongside the agent's subsequent reasoning to check whether the agent misinterpreted a technically-valid-but-unexpected response shape (e.g., an empty result list, a different schema than expected). This is a common and easy-to-miss failure mode: teams check tool status but not tool response content, and it looks like "the tool worked" right up until you inspect what it actually returned.

---

**Situation:** Your team adds a groundedness check after generation, but round-trip latency jumps noticeably and a stakeholder asks whether the guardrail is worth it. What would you do and why?

Model answer: Check whether the groundedness check itself is implemented as a full LLM-judge call — if so, it's plausible the guardrail now costs nearly as much latency/tokens as the original generation, which defeats its purpose. Replace it with a cheap, fast judge (a small classifier or distilled model) for the hot-path guardrail check, and reserve full LLM-judge evaluation for offline or sampled analysis where latency doesn't matter. Quantify the actual latency added by the cheap version and weigh it against the measured reduction in ungrounded answers reaching users, rather than debating the tradeoff in the abstract.

---

**Situation:** An on-call engineer, paged for a bad agent response at 2 a.m., fixes the one specific query they were shown and closes the incident. Two weeks later, a nearly identical failure recurs at scale. What would you do and why?

Model answer: Point out the process gap: fixing the one instance you happened to see, instead of querying for the cluster, systematically under-prioritizes the actual highest-leverage fix and lets the same structural bug keep recurring. The correct incident-response step is to filter traces for the same structural signature (e.g., `retrieval.top_score < 0.6` in the same time window) and count how many other requests match — a single query looks like a one-off, but forty queries sharing that signature is a systemic bug with a priority. Require every incident to end with that cluster check and, ideally, a new guardrail or monitoring signature so the same failure mode can't silently recur unnoticed.

---

**Situation:** A weekly review shows the retrieval-quality team has been individually patching different low-scoring queries for a month, one ticket at a time, with no measurable improvement in the overall failure rate. What would you do and why?

Model answer: Diagnose this as a symptom-tagging problem rather than a difficulty problem — if failures are tagged by surface symptom ("wrong answer," "user complained") instead of structural cause, they don't cluster into anything actionable, and every fix looks like a one-off. Introduce the filter-cluster-prioritize methodology: pull all failing traces (judge score below threshold) over a fixed window, tag each by structural cause using its own span evidence (retrieval score, truncation flag, tool status), count cluster sizes, and fix the largest cluster first. A worked pattern like "37 of 47 failures share a below-floor retrieval score" turns one month of scattered patches into one high-leverage fix (raise the threshold, add a re-ranker).

---

**Situation:** Your team is choosing a tracing tool and one engineer argues for LangSmith purely because "it's the most popular," without checking anything else. What would you do and why?

Model answer: Push back on choosing by popularity alone and evaluate fit against your actual architecture and constraints instead. LangSmith's deepest value comes from tight LangChain/LangGraph integration and a hosted-first model — a strong fit if you're already all-in on that ecosystem and a hosted SaaS billing model is acceptable. If your pipeline is framework-agnostic, self-hosting or data-residency matters, or retrieval-quality debugging is the primary pain point, Arize Phoenix's OpenInference-based, self-hostable design is a better match; Langfuse is the strongest option if open-source self-hosting plus built-in prompt management both matter. The right choice differs by architecture — check self-hosting needs and framework fit before picking based on brand familiarity.

---

**Situation:** A trace shows a retrieval span with a healthy top score of 0.81, but the generated answer still doesn't use any of the retrieved facts and instead seems to have invented its own explanation. What would you do and why?

Model answer: Classify this specifically as a generation failure, not a retrieval or context failure — the taxonomy matters here because a healthy retrieval score rules out the two most common upstream causes, narrowing the investigation to whether the model is actually grounding its answer in the supplied context. Check the context-assembly span for truncation or ordering problems first (a subtler context failure that a healthy top score alone doesn't rule out), and if the full relevant chunk really was present and intact, the fix is a generation-layer one: a stricter "answer only from the following context" instruction, a lower temperature, or a citation requirement — not a retrieval change, which would address a problem this trace shows doesn't exist.

---

**Situation:** After a silent upstream change, your agent's average response length nearly doubles and its tool-call pattern changes, but nobody notices until a customer complains three weeks later. What would you do and why?

Model answer: This is exactly the kind of behavioral-drift signal that should be caught by monitoring, not by a customer complaint — treat the delay as a gap in observability, not just bad luck. Recommend a recurring (e.g., weekly) job that filters and clusters trace attributes over time — response length, tool-call frequency, refusal rate — and flags trend shifts, not just single-incident debugging sessions. Cross-reference the timing of the shift against deploy logs (prompt changes, model version bumps, vendor-side updates) to identify the likely cause, and add a monitoring signature for this specific pattern so a similar shift is caught automatically next time rather than three weeks later via a complaint.

---

**Situation:** An engineer proposes retrying the entire pipeline from the top — re-retrieve, re-generate, everything — whenever any step fails, arguing it's simpler than targeted retries. What would you do and why?

Model answer: Push back — retrying the whole pipeline wastes latency and cost and frequently reproduces the exact same failure, since nothing about the upstream state that actually caused the failure has changed. Recommend step-scoped retries instead: if context assembly failed, re-retrieve; if generation failed a groundedness check, regenerate with a corrective instruction using the same already-validated context — don't re-run retrieval, which already succeeded. This requires each step's guardrail to tag *which* step failed (via `FailureReason`-style structured tags), which is exactly why guardrail decisions need to be logged as span attributes rather than collapsed into a single generic error.

---

**Situation:** A teammate silently swallows every guardrail failure and returns a generic "something went wrong" message with no further detail logged, to "keep the logs clean." What would you do and why?

Model answer: Flag this as destroying the evidence trail that root-cause clustering depends on — a generic caught-and-suppressed error with no structured tag makes every failure look identical in the logs, which means you can no longer distinguish a retrieval failure from a tool failure from an ungrounded generation after the fact. Require every guardrail branch to log a structured `FailureReason` (or equivalent tag) as a span attribute even when the user-facing message stays generic — the user sees a clean fallback message, but the trace still carries enough evidence to be clustered and prioritized later. Clean user-facing behavior and rich internal diagnostics are not in tension; conflating them is the actual mistake here.

---

**Situation:** Your agent occasionally calls the same tool three or four times in a row with nearly identical arguments before finally producing an answer, wasting tokens and latency. What would you do and why?

Model answer: Classify this as a reasoning/decision failure in the taxonomy, distinct from a retrieval or tool failure — the tool itself is working, but the agent's control-flow logic is making a poor repeated choice. Check the trace for whether each call's arguments and observations were meaningfully different (suggesting the agent is legitimately refining its approach) or nearly identical (suggesting it isn't registering that the tool's response already answered its question). Add explicit loop/repetition detection as a guardrail, and improve the tool's description or the few-shot examples in the prompt to make it clearer when a call has already produced a sufficient result, so the agent has a clear signal to stop.

---

**Situation:** A leadership review asks for hard numbers on whether the AI assistant is "hallucinating too much" before a wider rollout, and the only evidence anyone has is a handful of screenshots from unhappy users. What would you do and why?

Model answer: Screenshots are anecdotes, not evidence a rollout decision should be based on — build a labeled evaluation set and a recurring trace-based measurement instead. Pull a representative window of production traces, score them with an LLM judge (or a cheaper proxy) for groundedness/correctness, and report the failure rate plus the cluster breakdown of *why* those failures happened (retrieval below floor, truncation, out-of-scope, tool error). This converts "too much" from a subjective impression into a specific, trackable number with a trend line, and gives leadership a concrete target (e.g., "reduce the retrieval-below-floor cluster from 12% to under 5%") rather than a vague mandate to "make it better."

---

**Situation:** Your team's tracing setup captures spans correctly for synchronous calls, but traces for requests that go through an async worker queue show up as disconnected, broken fragments instead of one coherent trace. What would you do and why?

Model answer: Diagnose this as a trace-context propagation gap across the async/queue boundary — trace context (the trace ID and parent span ID) has to be explicitly forwarded into the queued message and picked up again by the worker, or the worker's spans start a brand-new, unrelated trace. Fix by propagating the OpenTelemetry context through the queue's message metadata (most OTel-compatible queue integrations support this directly) so the worker's spans nest correctly under the original request's trace. Verify the fix by confirming a single trace ID now spans both the producer and the worker side in the waterfall view, not just that spans exist on both sides independently.

---

**Situation:** A retrieval-quality dashboard shows retrieval score distribution looking stable, but the eval/judge quality score has been quietly trending downward for three weeks with no obvious single incident. What would you do and why?

Model answer: Don't default to "the model is degrading" — when quality drift shows up without a corresponding input or behavioral drift signal, investigate the evaluation/measurement pipeline itself first, since a stable-looking retrieval score with degrading judged quality can also mean the judge, the golden set, or the labeling process has drifted rather than the system under test. Check the judge's calibration against a small human-reviewed sample from the same period, and check whether the golden evaluation set itself has gone stale relative to what the system now actually handles. Only after ruling out a measurement-side explanation should you treat this as a genuine, unexplained quality regression worth a deeper trace-level investigation.

---

**Situation:** Your team wants to reduce the number of dashboards and alerts by combining retrieval score, tool error rate, and generation quality into one single "health score" per request. What would you do and why?

Model answer: Push back — collapsing distinct failure signals into one blended score recreates the exact anti-pattern this whole discipline exists to avoid, since a single number can't tell an on-call engineer *which* step broke, only that something did. Keep retrieval, context, generation, tool, and reasoning signals as separate, span-level attributes so the moment an alert fires, its type already narrows down where to look. If dashboard clutter is the real complaint, solve that with better dashboard organization (one panel per failure type, filterable) rather than by destroying the granularity that makes root-cause clustering possible in the first place.
