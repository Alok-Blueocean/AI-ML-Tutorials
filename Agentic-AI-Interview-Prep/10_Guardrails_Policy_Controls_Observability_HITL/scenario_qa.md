# Guardrails, Policy Controls, Observability, and HITL — Scenario-Based Q&A

**Situation:** You're designing an agent that can issue refunds up to $500 autonomously and escalate larger amounts. Leadership wants a human-in-the-loop approval gate for any refund over $200. What would you do and why?

Model answer: Mark the refund tool as conditionally requiring approval (e.g., Pydantic AI's `ApprovalRequired` raised only when `amount > 200`, or an equivalent LangGraph `interrupt()` gate) so small refunds still execute autonomously and only the risky subset pauses. The approval UX must show the reviewer the order history, the requested amount, and the agent's stated justification — not a bare approve/deny button, since context-free approval degrades into rubber-stamping. Define a timeout policy: unanswered after 15 minutes escalates to a supervisor queue, unanswered after 2 hours auto-denies (never auto-approves) and notifies the customer. Log every decision — reviewer identity, timestamp, what they saw, and their decision — to an immutable audit store, since this is exactly the record a finance or compliance review will ask for later. Finally, feed denied requests back into a regression eval set so recurring denial patterns become permanent test cases rather than anecdotes.

---

**Situation:** Your RAG-based agent occasionally executes a tool call using arguments that don't match what the user actually asked for, and no one noticed until a customer complained. What would you do and why?

Model answer: This is an observability gap, not necessarily a model-quality problem — treat "the model made a mistake" as an unproven hypothesis until you can see the trace. Instrument the agent with spans for each step (retrieval, each LLM call, each tool call) carrying token counts, latency, and the actual arguments/outputs at each step, following OpenTelemetry's `gen_ai.*`-style conventions so the trace is portable across backends. Reproduce the failure by pulling the specific trace and reading the tool-call span's arguments against the immediately preceding chat span's output — the bug is often in context assembly (stale or missing prior tool output in the next prompt) rather than the model itself. Add an automated check that flags tool calls whose arguments don't reference any entity mentioned in the last user turn, and add this case to a regression suite so a fix doesn't silently regress later.

---

**Situation:** A stakeholder asks why you can't just tell the model "never reveal the system prompt or discuss competitors" in the prompt and call it done. What would you do and why?

Model answer: Explain that prompt instructions are a request the model tries to follow, not an enforced constraint — a sufficiently adversarial or persistent user can often talk around them (prompt injection, role-play framing, translation tricks), and red-team testing routinely finds this. Recommend a layered approach: keep the prompt instruction (it helps in the common case and costs nothing), but add a deterministic output-side guardrail (e.g., a Guardrails AI validator or a regex/classifier check for system-prompt fragments or competitor names) that runs regardless of how the model was talked into producing the leak, and an input-side jailbreak/prompt-injection detector before the request even reaches the model. Frame it as defense in depth: the prompt is the first, weakest layer, not the only layer.

---

**Situation:** Your team wants to adopt Guardrails AI for output validation but a colleague argues "it's just regex and toxic-language checks, we could build that ourselves in a day." What would you do and why?

Model answer: Agree that the core library is thin by design — it's a Guard/validator execution framework, and most real capability comes from named validators on the Guardrails Hub (PII detectors like `DetectPII`/`PresidioGlinerPII`, jailbreak/prompt-injection detectors, topic restriction validators). Push back on "build it ourselves" by pointing out the actual cost isn't the regex, it's maintaining detection accuracy (false positive/negative tuning) across many risk categories over time, which is exactly what a maintained Hub validator amortizes across many users. Recommend prototyping with 2-3 Hub validators relevant to the actual risk surface (e.g., PII plus jailbreak) rather than either extreme — full adoption of every validator, or rejecting the framework outright — and measuring false-positive rate on real traffic before committing further.

---

**Situation:** Leadership asks for a single dashboard number that tells them "is our agent platform healthy" across cost, latency, and reliability. What would you do and why?

Model answer: Push back gently on the idea of a single number — cost, latency, and reliability fail independently and a composite score hides which one is actually broken. Propose a small set of top-line signals instead: p95/p99 end-to-end latency (not mean, since tail latency is what users feel), cost per successful run (and cost per tenant, since a small number of runaway sessions can dominate spend), tool-call error rate, and rate of runs that hit an approval gate versus runs that complete autonomously (a rising ratio signals either genuinely riskier traffic or an overly conservative guardrail). Back every top-line number with the ability to drill into the underlying trace for any anomaly, since a healthy-looking aggregate can hide a single misbehaving span, and a dashboard without drill-down just relocates the debugging problem instead of solving it.

---

**Situation:** A rate limiter is in place, but your LLM cost still spiked 10x overnight with no unusual request volume. What would you do and why?

Model answer: Rate limiting caps request *count*, not cost per request — the spike is more consistent with either much larger prompts per call (e.g., a bug stuffing an ever-growing conversation history or retrieved-context window into every request) or a shift to a more expensive model/route. Pull cost-per-request from traces (token counts x price per span) rather than looking at request volume alone, and check whether `gen_ai.usage.input_tokens` grew for a specific workflow. Add a per-session/per-tenant dollar budget check as a second, independent control from rate limiting, since the two catch different failure modes and neither substitutes for the other.

---

**Situation:** You're asked to add OpenTelemetry tracing to an existing agent and a colleague suggests just logging full prompts and responses to a text file instead, since "it's basically the same information." What would you do and why?

Model answer: Explain the concrete gaps: raw text logs give you input/output for one call but not the structured, queryable, cross-run view a trace gives you — no per-step latency breakdown, no easy aggregation of token counts or cost across thousands of runs, no standard way to filter "show me every trace where the retrieval step scored below threshold." Also flag the privacy risk of logging full raw prompts/responses by default, since user input can contain PII with no warning; structured spans let you capture token counts, latency, and coarse metadata by default and treat raw content capture as an explicit, narrowly-scoped opt-in. Recommend adopting OpenTelemetry's GenAI semantic conventions specifically because they're portable across backends (Jaeger, Langfuse, a vendor APM) via OTLP, while flagging that these conventions are still in "Development" stability status and attribute names may shift.

---

**Situation:** Two weeks after launching an agent's human-in-the-loop approval queue, reviewers are approving 98% of requests in under 3 seconds each. What would you do and why?

Model answer: Treat this as a strong signal the approval gate has degraded into rubber-stamping, not evidence the agent is 98% trustworthy. Investigate whether the approval UI actually surfaces enough context to make a real decision in 3 seconds — if reviewers see only "approve refund: yes/no" with no order history or reasoning, they've learned that clicking approve is faster than doing the underlying investigation. Fixes: reduce the volume of low-risk requests reaching the gate at all (raise the autonomous-approval threshold for genuinely low-risk cases, informed by the historical approval rate), and for the requests that remain, require the reviewer to affirmatively see the key risk factors (flag amount, account history anomalies) before the approve button is even enabled. Track reviewer decision time and denial rate over time as a health metric for the gate itself, not just for the agent.

---

**Situation:** Your company operates in the EU and a regulator asks for proof that a specific automated refund decision from three months ago went through appropriate human oversight. What would you do and why?

Model answer: This is exactly what audit logging exists for — pull the immutable log entry for that specific action: what was proposed, what context the reviewer saw, who the reviewer was, their decision, and the timestamp, plus the underlying trace showing the agent's tool call and reasoning that led to the proposal. If the log only captured "approved: true" with no context or reviewer identity, that's a compliance gap to fix immediately: audit logs need to capture enough to reconstruct *why* a human believed the action was correct, not just that a click occurred. Recommend a policy of retaining approval-gate audit records longer than routine operational trace data, since compliance review windows often exceed typical trace retention.

---

**Situation:** A team building an agent framework internally debates whether human-in-the-loop approval should be implemented as an application-level feature (outside the agent loop) or as a first-class capability of the agent framework itself. What would you do and why?

Model answer: Prefer a framework-level primitive when one exists and fits, because it standardizes the pause/resume state machine (e.g., Pydantic AI's `DeferredToolRequests`/`DeferredToolResults`, or LangGraph's `interrupt()`/checkpointed `thread_id`) instead of every team reinventing how to persist agent state across an indefinite human-response wait. The main risk of application-level HITL bolted on afterward is state management: naively blocking inside a request handler for a human response that might take hours doesn't survive a process restart, while a framework's durable-execution-aware approval mechanism does. Recommend adopting the framework primitive where available, and reserving custom application-level logic for the parts that are genuinely business-specific — the approval UI, the escalation policy, and the audit log — rather than the state machine itself.
