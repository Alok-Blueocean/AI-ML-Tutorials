# Agentic Systems and Tool Calling — Scenario-Based Q&A

**Situation:** An agent with a `delete_record(id)` tool occasionally deletes the wrong customer record, and the team's first instinct is to add a stricter prompt telling the model "be careful before deleting." What would you do and why?

Model answer: Treat this as a design/permissions problem, not a prompting problem — prompting a model to "be careful" is not a reliable safety control, and an occasional wrong deletion is exactly the failure mode least-privilege access exists to prevent. Require human confirmation before any destructive tool call executes, especially one with no undo, rather than trusting the model's judgment on when to be cautious. Add argument validation before execution (does this ID exist, does it belong to the expected account context) and an audit log of every delete call with its arguments, so a wrong call is both prevented where possible and traceable when it isn't.

---

**Situation:** Your support agent, wired up with a `search_knowledge_base` tool, starts calling the same tool with slightly reworded queries five or six times in a row without ever producing a final answer, burning tokens and time. What would you do and why?

Model answer: This is an unbounded agent loop, and the fix is architectural, not a smarter prompt. Set an explicit maximum number of steps/tool calls per task, and define what happens when that limit is hit — return a clear "unable to complete" message rather than silently truncating. Separately, investigate why the model kept retrying: often it's because the tool's description or results don't clearly signal "no results found, further rewording won't help," so tighten the tool's description and response format to give the model an unambiguous stopping signal. Add loop detection (e.g., flag near-identical repeated calls) as a second line of defense independent of the step cap.

---

**Situation:** A tool's JSON schema says a `date` parameter is a string, but the model sometimes passes `"tomorrow"` or `"next Friday"` instead of an ISO date, and your backend function crashes trying to parse it. What would you do and why?

Model answer: Never trust the model's structured output as pre-validated input — always validate and parse tool arguments in your own code before executing the underlying function, exactly as you would validate any external API input. Add explicit validation (and, where feasible, normalization — resolving relative dates against a known "today" passed in the tool description) before the call reaches business logic, and return a structured error back to the model rather than letting an exception propagate, so the model has a chance to retry with a corrected argument. Also tighten the tool's parameter description to explicitly require an ISO 8601 date, since clearer schemas and descriptions measurably reduce malformed arguments in the first place.

---

**Situation:** Your team wants to give an internal coding agent a `run_shell_command` tool so it can install dependencies and run tests autonomously, but security is uneasy about the blast radius. What would you do and why?

Model answer: Security's instinct is correct — a raw shell-execution tool is one of the highest-blast-radius tools you can hand an agent, since it inherits whatever permissions the executing process has. Run it inside a tightly sandboxed, ephemeral environment (a container with no access to production credentials, secrets, or the broader network) rather than the host running the agent orchestration itself. Apply command allow-listing or at minimum deny dangerous patterns, require human approval for anything outside a pre-approved command set, and log every command executed with its output for audit. Treat tool access itself as a security boundary requiring least privilege, exactly as you would for any service account.

---

**Situation:** A stakeholder wants to know why an agent-based feature costs 8x more per request than the equivalent single-prompt classifier it's replacing, and whether that's expected. What would you do and why?

Model answer: Explain the cost driver concretely rather than treating it as mysterious: an agent loop makes multiple LLM calls per task (reasoning steps, tool-call decisions, and re-prompting after each tool observation), so cost scales with the number of loop iterations, not a single call. Break down the average number of steps per task and the token cost per step to show where the multiplier actually comes from. Then evaluate whether the task genuinely needs agentic flexibility or whether a simpler, cheaper single-prompt or fixed-chain approach would achieve comparable quality — reach for a multi-step agent only when the task's complexity actually justifies it, not by default.

---

**Situation:** An agent tasked with drafting and sending customer emails is found to have sent an email with a hallucinated discount code that doesn't exist in the system. What would you do and why?

Model answer: Treat any agent action with real-world, external-facing consequences as requiring a guardrail before execution, not after. Add a validation step between "model proposes to send" and "email actually sends" that checks referenced facts (like a discount code) against the system of record, rejecting or flagging the action if it can't be verified. For anything customer-facing and hard to undo, add a human-in-the-loop approval step, at least until the guardrail's false-negative rate is proven low over time. Also audit the tool's prompt/context to check whether the model even had access to real discount codes — if not, it was forced to guess, which is a context/tooling gap as much as a model behavior problem.

---

**Situation:** Your team is building a multi-agent system with a "researcher" agent, a "writer" agent, and a "reviewer" agent, but debugging why the final output is wrong is taking hours because it's unclear which agent introduced the error. What would you do and why?

Model answer: This points to a tracing/observability gap, not necessarily a design flaw in the multi-agent split itself. Instrument each agent's inputs, outputs, and tool calls as separate, inspectable steps (a trace showing exactly what the researcher retrieved, what the writer produced from it, and what the reviewer flagged or missed) so a wrong final answer can be attributed to a specific agent's specific step rather than requiring you to re-run the whole pipeline and guess. If tracing reveals the split itself is the problem — e.g., agents talking past each other, unclear handoff contracts — simplify: a single well-scoped agent is easier to debug than a crew of three, and multi-agent designs are worth their complexity only when the task genuinely benefits from role separation.

---

**Situation:** An engineer proposes letting the agent decide dynamically, at runtime, whether to use LangGraph-style explicit state-machine control or free-form ReAct looping, "so it's more flexible." What would you do and why?

Model answer: Push back on making the control-flow paradigm itself a runtime decision — that adds a meta-layer of unpredictability on top of an already probabilistic system, making the agent harder to test, debug, and reason about. Pick one control-flow approach deliberately based on the task's actual shape: an explicit graph/state-machine (LangGraph-style) when you need predictable branching, retries, and auditable state transitions; a looser ReAct loop when the task's steps genuinely can't be predetermined. The flexibility that matters is in what the agent does within a well-defined loop structure, not in swapping the loop structure itself at runtime.

---

**Situation:** A production incident report says an agent "got stuck" for 20 minutes calling a flaky external API tool that kept timing out, before someone manually killed the process. What would you do and why?

Model answer: Add both a per-call timeout and a bounded retry policy on the tool itself, independent of the agent's own step limit — a tool that can hang indefinitely defeats any step-count safeguard because the agent never even gets to decide on a next step. Distinguish transient failures (worth a small number of retries with backoff) from persistent failures (should surface as a tool error back to the model quickly, so it can adapt or report the task as blocked) rather than retrying blindly forever. Also add an overall wall-clock timeout for the whole task as a last-resort safeguard, separate from step count, so a single slow tool can't stall the entire agent run.

---

**Situation:** Your company is integrating tools from three different internal teams into one agent, and each team built their own bespoke API with inconsistent auth, input formats, and error handling. What would you do and why?

Model answer: This is exactly the integration-sprawl problem MCP-style standardization exists to solve. Rather than writing custom, one-off glue code to adapt each team's bespoke API into the agent's tool-calling format, propose adopting a common protocol (an MCP server per team's tool surface, or at minimum an internal convention for auth, schemas, and error responses) so any agent or framework can plug into any tool without bespoke integration work per pairing. This decouples "which tools exist" from "which agent framework happens to be calling them," and makes onboarding a fourth team's tools additive rather than another one-off integration project.

---

**Situation:** A user manages to get your customer-facing agent to reveal its system prompt and internal tool definitions by asking it clever meta-questions. What would you do and why?

Model answer: Treat this as expected adversarial behavior to defend against, not a one-off exploit to patch reactively. System prompts and tool definitions should never contain secrets (API keys, internal-only business logic that would be harmful if exposed) in the first place, since prompt leakage of some form should be assumed possible regardless of countermeasures. Add explicit instructions and, more reliably, an output-filtering guardrail that detects and blocks responses resembling the system prompt or tool schema before they reach the user. Accept that determined users may still partially succeed, and design the system so a leaked prompt is merely embarrassing, not a security incident, by keeping genuinely sensitive logic and credentials server-side and never in the prompt at all.

---

**Situation:** An agent given both a `read_database` tool and a `write_database` tool ends up, in one run, reading stale data, writing based on it, and then reading its own stale write back into context — corrupting a multi-step calculation. What would you do and why?

Model answer: Diagnose this as a state-consistency/ordering bug rather than a "the model got confused" issue — the agent's tools let it violate an implicit precondition (don't act on data you haven't confirmed is current) that nothing in the system enforced. Add explicit versioning or timestamps to read results so the agent (and your own validation layer) can detect staleness, and consider separating "read" and "write" into distinct phases with a validation checkpoint between them for multi-step calculations, rather than letting the agent freely interleave reads and writes with no consistency guarantee. This is a case where tightening the tool contract, not the prompt, fixes the bug.

---

**Situation:** Leadership wants to know whether your team should build a custom agent loop in-house or adopt a framework like LangGraph or CrewAI for an upcoming project. What would you do and why?

Model answer: Frame the decision around how much the project needs explicit control versus quick expressiveness, not around which framework is currently popular. LangGraph earns its complexity when you need fine-grained control over branching, retries, and state — closer to a state machine than a free-form loop — which matters for production systems with complex failure handling. CrewAI's role-based framing is faster for expressing a multi-agent collaboration quickly but offers less fine-grained control. A hand-rolled loop can be the right call for a simple, well-understood, single-agent task where a framework's abstractions would add overhead without adding value. In all cases, remind the team that frameworks manage plumbing (state, retries, coordination), not judgment — the tools, prompts, and guardrails still have to be designed regardless of framework choice.

---

**Situation:** During testing, an agent with access to a `send_slack_message` tool posts an internal debugging message to a real, customer-visible Slack channel by mistake. What would you do and why?

Model answer: Treat this as a missing environment-isolation failure, not a one-off model mistake. Tool implementations should be environment-aware by construction — a test/staging agent should be wired to test/staging tool endpoints (a sandbox Slack workspace or channel) that are physically incapable of reaching production channels, rather than relying on the model or a prompt instruction to "remember" it's in a test run. Audit every tool with external side effects for this same risk, and add a hard configuration check (not just a convention) that fails loudly if a non-production agent is ever pointed at a production tool endpoint.

---

**Situation:** Your agent framework logs show the agent frequently selects a `generic_search` tool when a much more precise `lookup_order_by_id` tool was clearly the better fit for the query. What would you do and why?

Model answer: Diagnose this as a tool-description quality problem before assuming the model is simply bad at tool selection — vague or overlapping tool descriptions are the single most common cause of wrong-tool selection, since the model is choosing based on what it can infer from the name and description alone. Rewrite both tools' descriptions to be more specific and mutually distinguishing (clarify exactly when `lookup_order_by_id` applies versus when `generic_search` is the fallback), and add a few-shot example in the system prompt showing correct selection for a similar ambiguous case. Measure tool-selection accuracy before and after the change on a fixed test set rather than assuming the fix worked from anecdotal spot-checks.

---

**Situation:** A team member argues that since the agent framework handles retries and state automatically, they don't need to add their own error handling around tool execution code. What would you do and why?

Model answer: Correct this misunderstanding directly — a framework's retry/state management operates at the level of the agent loop (re-prompting the model, resuming from a checkpoint), not at the level of your tool's own internal correctness. If your tool function itself throws an unhandled exception, doesn't validate its inputs, or fails silently on a malformed response from a downstream API, no amount of framework-level looping fixes that; it just means the framework retries the same broken call. Application code invoked by a tool needs the same defensive engineering (input validation, explicit error returns, logging) as any other production code path — the framework's job is orchestration, not making your tool implementations correct.

---

**Situation:** An agent that automatically triages and closes support tickets is found, after a few weeks in production, to be closing a small but real number of tickets that actually needed human follow-up. What would you do and why?

Model answer: Investigate whether this is a confidence-calibration gap rather than a fundamentally broken triage policy — the agent may be making close/no-close decisions without any notion of its own uncertainty, treating every case as equally clear-cut. Add a confidence or ambiguity check before the "close ticket" tool call executes, routing low-confidence cases to a human reviewer instead of auto-closing them, and track the false-closure rate as an explicit metric going forward, not just overall throughput. For an action this consequential (a closed ticket a customer can't easily reopen), consider requiring human-in-the-loop approval for closures until the false-closure rate is proven low enough to trust fully autonomous operation.
