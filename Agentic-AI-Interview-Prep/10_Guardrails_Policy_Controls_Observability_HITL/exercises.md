# Guardrails, Policy Controls, Observability, and HITL — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual/design questions.

1. **Build a Pydantic AI agent with structured output.** Define a Pydantic model (e.g., `TicketTriage` with `category: Literal[...]`, `priority: int`, `needs_human: bool`) and build an agent with `output_type=TicketTriage`. Feed it 5 sample support messages and confirm every response validates against the schema. Break it intentionally (ask a question the model can't confidently categorize) and observe what happens.

2. **Add a dependency-injected tool.** Extend the agent from exercise 1 with a `deps_type` dataclass holding a fake in-memory "customer database," and one `@agent.tool` that looks up a customer's order history via `ctx.deps`. Write a unit test that swaps in a second fake database with different data and confirms the agent's tool call uses it, without touching the agent definition.

3. **Configure a Guardrails AI PII validator.** Install `guardrails-ai`, pull `DetectPII` from the Hub, and configure a `Guard` with `on_fail="fix"`. Run it against 10 sample LLM outputs, 5 containing PII (emails, phone numbers) and 5 not. Report false positive/negative rates and what "fix" mode actually does to the flagged text.

4. **Configure a jailbreak/topic guardrail.** Using either Guardrails AI's `DetectJailbreak`/`RestrictToTopic` or NeMo Guardrails' input rails, build a config that rejects 5 known jailbreak-style prompts (e.g., "ignore previous instructions and...") while still allowing 5 legitimate prompts through unmodified. Tune until you get zero false rejections on the legitimate set.

5. **Conceptual: guardrail library choice.** Given a team building (a) a structured-data extraction pipeline and (b) a multi-turn sales chatbot that must never make pricing promises, argue which of Guardrails AI or NeMo Guardrails fits each better, and why the *shape* of the constraint (single-response validation vs multi-turn dialog policy) drives the choice more than either library's feature list.

6. **Design a role-based tool access layer.** For an agent with tools `lookup_order`, `issue_refund_under_50`, `issue_refund_any_amount`, and `delete_account`, design (in code or pseudocode) a role-to-tool-allowlist mechanism enforced at tool-registration time, not just in the system prompt. Write a test proving a low-privilege role's agent session cannot even see the `delete_account` tool.

7. **Build a rate limiter and a cost budget check.** Implement a token-bucket rate limiter per API key and a separate running per-session dollar-budget check that raises before a call would exceed budget. Explain in a short paragraph why these are two different controls that can each fail independently.

8. **Instrument a 3-step agent run with OpenTelemetry.** Wrap a simple retrieve -> generate -> tool-call pipeline with OpenTelemetry spans using `gen_ai.*`-style attributes: capture `gen_ai.usage.input_tokens`/`output_tokens` and latency per span, with a parent span for the whole run. Export to console or a local collector and produce a trace waterfall for one successful and one failing run.

9. **Add cost and latency roll-ups to a dashboard.** Using the traces from exercise 8, aggregate per-request cost (tokens x price) and compute p50/p95/p99 latency across 50 simulated runs. Identify which span contributes most to p99 tail latency and propose a concrete fix.

10. **Design an approval-gate workflow for a high-risk action.** For an agent that can issue refunds up to $500, design the full approval flow: what the reviewer sees, what happens on approve/deny, what happens on no response within a defined timeout, and what gets logged for audit. Implement it with Pydantic AI's `requires_approval=True` / `DeferredToolRequests` (or LangGraph's `interrupt()`), including the resume path.

11. **Build the escalation path.** Extend exercise 10 so an unanswered approval request auto-escalates to a second reviewer queue after 15 minutes and auto-denies (not auto-approves) after 2 hours with no response. Justify the default-deny choice in one paragraph.

12. **Conceptual: content policy enforcement, prompt vs deterministic check.** Given a system prompt instruction "never recommend a competitor's product," design a deterministic output-side check that doesn't rely solely on the model following that instruction, and explain a concrete way an adversarial user could defeat the prompt-only version.

13. **Feedback capture loop.** Design a schema for logging every human approval/denial decision (action proposed, reviewer, decision, timestamp, free-text reason) and describe how you'd turn 60 days of this log into (a) a permanent regression eval set and (b) a proposed prompt or policy change, using a concrete hypothetical cluster of denials as the example.

14. **End-to-end critique.** Given a described production agent that can issue refunds, has no rate limiting, logs full user PII in plaintext traces, and has a single "approve all" button with no context shown to reviewers, produce a prioritized list of 5 concrete fixes across guardrails, policy controls, observability, and HITL design, and justify the order.

15. **Conceptual: OpenTelemetry conventions maturity.** Explain what it means that OpenTelemetry's GenAI semantic conventions are currently in "Development" stability status, and what that implies for a team choosing to standardize on `gen_ai.*` attribute names today versus building a thin internal abstraction layer over them.
