# Guardrails, Policy Controls, Observability, and Human-in-the-Loop — Condensed Study Notes

## Pydantic AI: Typed Agents and Structured Output

- Pydantic AI is a Python agent framework from the Pydantic team, positioned as "a typed, extensible agent loop with every model a string swap away." Every agent declares an `output_type` (a Pydantic model, dataclass, TypedDict, or union of these), and the framework validates the model's response against it before returning — no manual JSON parsing or regex extraction of the answer.

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class CityLocation(BaseModel):
    city: str
    country: str

agent = Agent('openai:gpt-5', output_type=CityLocation)
result = agent.run_sync('Where were the 2012 Olympics held?')
print(result.output)  # city='London' country='United Kingdom' — a validated CityLocation, not raw text
```

- Real-world example: an internal expense-approval agent must return a strict `{amount: float, category: Literal[...], needs_review: bool}` shape so a downstream finance system can consume it directly — `output_type` makes a malformed or partially-filled response a validation error the agent framework catches and retries, instead of a broken payload reaching finance's database.

## Pydantic AI: Dependency Injection for Tools

- Tools and system prompts receive a typed `RunContext[Deps]` whose first argument is the dependency object declared via `Agent(deps_type=Deps)`. This is the same dependency-injection idea as FastAPI: swap a real HTTP client/DB connection for a test double without touching agent logic.

```python
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext

@dataclass
class SupportDeps:
    db: DatabaseConn
    account_id: str

agent = Agent('openai:gpt-5', deps_type=SupportDeps)

@agent.tool
async def get_balance(ctx: RunContext[SupportDeps]) -> float:
    return await ctx.deps.db.fetch_balance(ctx.deps.account_id)

result = agent.run_sync('What is my balance?', deps=SupportDeps(db=real_db, account_id='acct_1'))
```

- Real-world example: a customer-support agent's test suite runs the exact same agent and tool code against a fake in-memory `DatabaseConn` instead of production Postgres, because `SupportDeps` is just a constructor argument — no mocking framework or monkeypatching of the LLM call itself is needed.

## Pydantic AI: Human-in-the-Loop Tool Approval (Deferred Tools)

- Pydantic AI has first-party support (current as of the 2.x releases) for marking a tool as requiring approval. When the model calls it, the run stops and returns a `DeferredToolRequests` object instead of a final answer; the caller collects a human decision and resumes the run with `DeferredToolResults`.

```python
@agent.tool_plain(requires_approval=True)
def delete_file(path: str) -> str:
    return f'File {path!r} deleted'

# first run: model calls delete_file -> agent.run_sync returns DeferredToolRequests
# after a human decides:
from pydantic_ai import DeferredToolResults, ToolDenied
results = DeferredToolResults()
results.approvals[call.tool_call_id] = True          # or ToolDenied('not authorized')
final = agent.run_sync(message_history=messages, deferred_tool_results=results)
```

- A tool can also raise `ApprovalRequired()` conditionally (e.g., only when a specific file path or dollar amount is touched), so most calls execute immediately and only the risky subset pauses for a human.

- Real-world example: an ops agent that can restart services and delete stale files marks only `delete_file` and `restart_prod_service` with `requires_approval=True`; read-only diagnostic tools run unattended, and the pause-and-resume pattern lets a Slack bot present the two risky calls to an on-call engineer without redesigning the agent loop.

## Pydantic AI: Multi-Model Support, Streaming, Durable Execution

- One agent definition runs against "virtually every model and provider (OpenAI, Anthropic, Google, Bedrock, Azure AI Foundry, Groq, Mistral, xAI, Ollama, and dozens more), swappable with a string" — `Agent('anthropic:claude-sonnet-5')` vs `Agent('openai:gpt-5')` is the only line that changes.
- Durable execution integrates with workflow engines (Temporal, DBOS, Prefect, Restate) so a long-running or human-in-the-loop agent survives process restarts instead of losing state if the pausing tool-approval step takes hours.
- Real-world example: a document-review agent pauses on `requires_approval=True` for a legal sign-off that might not happen until the next business day — durable execution means the agent process doesn't need to stay alive (or hold an open connection) for that entire window.

## Guardrails AI: Input/Output Validation Framework

- Guardrails AI (`guardrails-ai/guardrails` on GitHub, ~7.4k stars) wraps an LLM call with **Guards** that run **validators** on input and/or output. A validator can fix, reask, or raise/exception on failure. The **Guardrails Hub** is a marketplace of pre-built validators rather than one bundled with the core library.

```python
from guardrails import Guard
from guardrails.hub import DetectPII, ToxicLanguage

guard = Guard().use_many(
    DetectPII(pii_entities=["EMAIL_ADDRESS", "PHONE_NUMBER"], on_fail="fix"),
    ToxicLanguage(on_fail="exception"),
)
outcome = guard.validate(llm_response_text)
```

- Named hub validators relevant to guardrail interviews: `DetectPII` / `GuardrailsPII` / `PresidioGlinerPII` for PII redaction; `DetectJailbreak`, `DetectPromptInjection`, `DetectSystemPromptLeakage` for adversarial input; `ToxicLanguage`, `NSFWText`, `RestrictToTopic`, `SensitiveTopic` for content-policy enforcement.

- Real-world example: a healthcare intake chatbot runs `DetectPII` on every outbound response before it's logged or emailed, because logging a patient's phone number in plaintext support tickets is itself a compliance violation, independent of what the model was asked to do.

## NeMo Guardrails: Programmable Rails

- NVIDIA NeMo Guardrails (GitHub org renamed to `NVIDIA-NeMo`, repo `NVIDIA-NeMo/Guardrails`, ~7.2k stars, Apache 2.0) adds five rail types around an LLM: **input**, **dialog**, **retrieval**, **execution**, and **output** rails, configured largely via a YAML config plus the Colang flow-definition language for dialog rails.
- Jailbreak detection can run as a self-check (a second LLM call classifying "is this a jailbreak attempt"), pattern-based heuristics, or an NVIDIA NemoGuard NIM model; topic control works similarly via dialog rails or a NemoGuard Topic Control NIM for semantic topic detection.
- Real-world example: a banking chatbot's dialog rail defines an explicit Colang flow so any user attempt to steer the conversation toward "give investment advice" is redirected to a canned compliance-safe response, regardless of how the request is phrased — this is a stronger guarantee than a keyword blocklist because it's a structured conversation-flow constraint, not pattern matching on the raw prompt.

## Guardrails AI vs NeMo Guardrails vs Roll-Your-Own

| | Guardrails AI | NeMo Guardrails | Roll-your-own regex/LLM-judge |
|---|---|---|---|
| Model | Python decorators + Hub validator marketplace | YAML config + Colang dialog flows | Whatever you write |
| Strength | Fast to bolt onto an existing pipeline, strong structured-output validation | Strong for multi-turn dialog policy and topic steering | Full control, zero dependency risk |
| Weak point | Validators vary in quality/maintenance across the Hub | Colang has a learning curve; heavier to adopt for a single input/output check | You own detection accuracy and false-positive tuning yourself |

- Real-world example: a team needing "reject PII in structured JSON output" reaches for Guardrails AI; a team building a multi-turn sales-conversation bot that must never wander into pricing promises it can't honor reaches for NeMo Guardrails' dialog rails, because the constraint is about conversation *flow*, not a single response.

## Policy Controls: Role-Based Tool Access

- An agent with a fixed toolset should still gate *which* tools a given caller/session can invoke, based on role, not just what the model decides to call.

```python
ALLOWED_TOOLS = {
    "support_agent": {"lookup_order", "issue_refund_under_50"},
    "support_lead":  {"lookup_order", "issue_refund_under_50", "issue_refund_any_amount"},
}

def check_tool_access(role: str, tool_name: str) -> bool:
    return tool_name in ALLOWED_TOOLS.get(role, set())
```

- Real-world example: a refund agent exposes `issue_refund_any_amount` only when the calling context's role is `support_lead`; a front-line agent session simply never gets that tool registered, so a prompt-injection attempt to "issue a $10,000 refund" fails at the tool-access layer even if it somehow talked the model into trying.

## Policy Controls: Rate Limiting and Cost Controls

- Rate limiting caps requests/tokens per user, per API key, or per tenant over a time window (token bucket or sliding window); cost controls cap total spend, often via a running per-session or per-tenant token/dollar budget checked before each LLM call.

```python
def enforce_budget(session, estimated_cost_usd: float):
    if session.spent_usd + estimated_cost_usd > session.budget_usd:
        raise BudgetExceeded(f"session {session.id} would exceed ${session.budget_usd} budget")
    session.spent_usd += estimated_cost_usd
```

- Real-world example: a multi-tenant SaaS agent platform caps each free-tier tenant at $2/day of LLM spend; when an agent gets stuck in a tool-calling loop (a reasoning failure), the budget check — not the loop-detection logic — is what actually stops the runaway bill before it reaches four figures.

## Policy Controls: Content Policy Enforcement

- Content policy is the set of business/legal rules about what the system may say or do, enforced as guardrail checks (see above) plus explicit deny-lists of actions (e.g., "never recommend a competitor's product," "never give medical dosage advice") that are checked deterministically, not left to prompt instructions alone, because prompted instructions can be argued around by an adversarial or unusually persistent user.
- Real-world example: a legal-assistant product's system prompt says "don't give specific legal advice," but the team also adds a deterministic output check that blocks any response containing phrases like "you should sue" or "I recommend filing" — belt-and-suspenders, because relying on the prompt alone measurably failed red-team testing.

## Observability: Tracing a Multi-Step Agent Run

- A trace is the end-to-end record of one agent run; each discrete unit of work (an LLM call, a tool call, a retrieval step) is a span with a start time, duration, and attributes, nested under a parent so the whole run renders as a waterfall. For an agent specifically, OpenTelemetry's GenAI conventions define a top-level `invoke_agent` span, with `chat` spans per LLM call and `execute_tool` spans per tool invocation nested underneath.

```
invoke_agent (refund_agent)                         [820ms]
├── chat (gpt-5)                                     [310ms]  gen_ai.usage.input_tokens=412, output_tokens=38
├── execute_tool (lookup_order)                      [140ms]
├── chat (gpt-5)                                      [280ms]  gen_ai.usage.input_tokens=460, output_tokens=52
└── execute_tool (issue_refund)  [requires_approval]  [pending -> 90ms after human approval]
```

- Real-world example: a refund agent occasionally issues a refund for the wrong order number; tracing shows the `lookup_order` tool call returned the correct order, but the second `chat` span's input didn't include the tool result at all (a context-assembly bug in the agent loop) — without span-level tracing this looks indistinguishable from "the model made a mistake."

## Observability: OpenTelemetry GenAI Semantic Conventions

- OpenTelemetry defines a standard `gen_ai.*` attribute vocabulary (`gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.operation.name`, `gen_ai.response.finish_reasons`, `gen_ai.conversation.id`) so a span produced by any framework or vendor means the same thing to any observability backend. As of this session, these conventions live in a dedicated `open-telemetry/semantic-conventions-genai` repository and are explicitly marked **Development** status (not yet stable) — expect attribute names to still shift across releases.
- Real-world example: a platform team standardizes on `gen_ai.*` attributes specifically so they can swap their tracing backend (e.g., from a self-hosted Jaeger/Grafana stack to a vendor like Langfuse or an APM vendor) without re-instrumenting every agent — the same OpenTelemetry SDK output is portable across backends via the OTLP exporter.

## Observability: Token, Cost, and Latency Monitoring

- Per-span token counts (`gen_ai.usage.input_tokens` / `output_tokens`) roll up into per-request cost (tokens x provider price) and per-tenant/per-feature cost dashboards; latency is tracked as both end-to-end (full agent run) and per-step (which span is the bottleneck), with p95/p99 tracked separately from the mean because tail latency is what individual users actually feel.
- Real-world example: a dashboard shows mean agent-run latency well within SLA, but p99 is 8x the mean — segmenting spans shows a single retrieval step occasionally times out against a cold vector-index shard; fixing that one span's timeout/retry behavior fixes the tail without touching the "average" system at all.

## Human-in-the-Loop: Approval Gates for High-Risk Actions

- An approval gate is an explicit pause point before an irreversible or high-cost action (refunds, deletions, financial transfers, external emails) where the agent hands control to a human reviewer instead of executing autonomously. The pattern generalizes across frameworks: LangGraph exposes it via `interrupt()` (the run pauses, state is checkpointed against a `thread_id`, and resuming requires passing a `Command(resume=...)` back in); Pydantic AI exposes it via `requires_approval=True` / `ApprovalRequired` and `DeferredToolRequests`/`DeferredToolResults`.
- Design questions an approval gate must answer explicitly: what does the reviewer see (the proposed action plus the reasoning/context that led to it, not just "approve refund: yes/no"); what happens if no one responds in time (auto-deny is safer by default than auto-approve); and how is the decision logged (who approved, when, what they saw) for audit.
- Real-world example: a refund agent's approval gate shows the reviewer the order history, the refund amount, and the model's stated justification, not just a bare approve/deny button — reviewers who can't see *why* the agent wants to refund $400 either rubber-stamp everything (defeating the gate) or reject everything (defeating the agent).

## Human-in-the-Loop: Escalation Patterns

- Escalation handles the case where the first-line reviewer can't or won't decide: a timeout with no response escalates to a supervisor queue or auto-denies (never auto-approves an unreviewed high-risk action); a reviewer explicitly deferring escalates immediately; repeated agent retries of a denied action escalate to a human review of the agent's behavior itself, not just the individual request.
- Real-world example: a support-agent refund queue auto-escalates any approval request untouched for 15 minutes to a team-lead queue, and auto-denies (with a "contact support" message to the customer) if nothing happens within 2 hours — the customer is never left in a silent limbo waiting on a human who may be on vacation.

## Human-in-the-Loop: Feedback Capture for Continuous Improvement

- Every approval/denial decision, plus any free-text reviewer comment, is itself training/eval signal: denied actions with a stated reason feed back into either prompt/policy tuning (if the agent is requesting the wrong thing systematically) or into the regression eval set (so a fixed failure mode has a permanent test case).
- Real-world example: after two months of a refund agent's approval queue, 30% of denials cluster around one specific store policy edge case (partial refunds on discounted items) — that pattern becomes both a prompt fix and a new eval case, rather than staying as anecdotal reviewer frustration.

## Quick Gotchas Worth Naming in an Interview

- `requires_approval=True` in Pydantic AI pauses the *entire agent run*, not just that one tool call — the caller must persist `message_history` and resume with `deferred_tool_results`; treating it like a synchronous confirmation dialog inside the same process call is a common design mistake.
- Guardrails AI's core library ships very few validators itself — real usage means pulling named validators from the Guardrails Hub, and Hub validator quality/maintenance varies, so treat "we use Guardrails AI" as "we use Guardrails AI plus whichever specific validators we picked," not a complete answer.
- OpenTelemetry's GenAI semantic conventions are explicitly unstable ("Development" status) as of this session — pin attribute names or expect a migration when they stabilize; don't present them in an interview as a long-finalized standard.
- Rate limiting and cost controls are not the same control: a request can be well within rate limits (few calls) and still blow a cost budget (each call is huge, e.g., a long context stuffed with retrieved documents) — both checks are needed, not one or the other.
- An approval gate that shows a reviewer only "approve/deny" with no context degrades into rubber-stamping within days — the gate's value is entirely a function of how much relevant context it surfaces, not the mere existence of a human click.
- A timeout policy that defaults to auto-approve on no-response silently converts a human-in-the-loop system into a fully autonomous one during any on-call gap — default-deny (or default-escalate) is the safe failure mode for high-risk actions, symmetrically with why circuit breakers default open, not closed.
