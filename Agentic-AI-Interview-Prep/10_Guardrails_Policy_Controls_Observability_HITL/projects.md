# Guardrails, Policy Controls, Observability, and HITL — Projects

## Small: Type-Safe Support Agent with a PII Output Guard

Build a Pydantic AI agent for triaging support tickets: a `TicketTriage` output model, one dependency-injected tool for looking up account history, and a Guardrails AI `DetectPII` check wrapped around every outbound response before it's returned to the caller. Log every guard trigger (what was caught, what "fix" mode changed) to a local file. This proves you can combine a typed agent framework with a bolt-on output guardrail, and that you understand the guard runs on the *response*, not the agent's internal reasoning.

## Medium: Traced Multi-Step Agent with Cost/Latency Dashboard

Build a 3-4 step agent (retrieve context, call an LLM, call one or two tools) instrumented end to end with OpenTelemetry spans following `gen_ai.*`-style attributes — one parent span per run, child spans per step, token counts and latency on each. Export traces to a local collector or console, aggregate 100+ simulated runs into a small dashboard (script or notebook is fine) showing per-step p50/p95/p99 latency and per-run cost. Add a rate limiter and a per-session cost budget in front of the agent. This proves you can operationalize observability rather than just describe it, and forces you to actually find and explain a real tail-latency contributor instead of an average.

## Large: Refund Agent with Full Guardrail, Policy, and Approval Stack

Build an agent that can look up orders and issue refunds, with: role-based tool access (only a "lead" role can approve refunds over $50), a rate limiter and cost budget per session, a Guardrails AI or NeMo Guardrails input check against prompt-injection/jailbreak attempts, full OpenTelemetry tracing of every run, and a real human-in-the-loop approval gate (Pydantic AI's `requires_approval`/`DeferredToolRequests` or LangGraph's `interrupt()`) with a timeout that auto-escalates after 15 minutes and auto-denies after 2 hours. Log every approval decision with reviewer, timestamp, and reason to a durable audit log, and build a small script that mines 50 synthetic denial records into a proposed prompt/policy fix. This is close to a real production agent's control-plane design and demonstrates every piece of this topic working together, not in isolation.
