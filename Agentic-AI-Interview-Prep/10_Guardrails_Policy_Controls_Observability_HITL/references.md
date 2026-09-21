# Guardrails, Policy Controls, Observability, and HITL — References

All links below were fetched and content-checked against the claim this session unless explicitly marked otherwise.

## Papers

- No foundational academic paper anchors this topic the way arXiv papers anchor NLP/RAG topics — this is primarily a tooling/systems-design area. If asked for "the paper" in an interview, the honest answer is that guardrails/observability/HITL are engineering practices assembled from production experience, not a single seminal publication.

## Official Docs

- Pydantic AI overview. https://ai.pydantic.dev/ (redirects to https://pydantic.dev/docs/ai/overview/ — both fetched; content: framework description, structured output, dependency injection, multi-model, durable execution, HITL tool approval summary).
- Pydantic AI — dependencies (`RunContext`, `deps_type`). https://pydantic.dev/docs/ai/core-concepts/dependencies/
- Pydantic AI — structured output (`output_type`). https://pydantic.dev/docs/ai/output/
- Pydantic AI — deferred tools / human-in-the-loop approval (`requires_approval`, `ApprovalRequired`, `DeferredToolRequests`, `DeferredToolResults`). https://pydantic.dev/docs/ai/deferred-tools/
- Pydantic AI — durable execution overview (Temporal/DBOS/Prefect/Restate integration). https://pydantic.dev/docs/ai/capabilities/durable_execution/overview/
- `pydantic-ai` on PyPI — version check. https://pypi.org/project/pydantic-ai/ (version 2.45.0 as of this session, released 2026-09-18 — this package ships very frequent releases; re-check the version before quoting it in an interview or doc).
- Guardrails AI docs (introductory page; thin on validator-level detail — see Hub link below for actual validator names). https://www.guardrailsai.com/docs
- Guardrails AI Hub (validator marketplace: `DetectPII`, `GuardrailsPII`, `PresidioGlinerPII`, `DetectJailbreak`, `DetectPromptInjection`, `DetectSystemPromptLeakage`, `ToxicLanguage`, `NSFWText`, `RestrictToTopic`, `SensitiveTopic`). https://guardrailsai.com/hub
- NeMo Guardrails docs (rail types: input/dialog/retrieval/execution/output; jailbreak detection methods; topic control). https://docs.nvidia.com/nemo/guardrails/latest/index.html
- LangGraph — human-in-the-loop interrupts (`interrupt()`, `Command(resume=...)`, `thread_id` checkpointing). https://docs.langchain.com/oss/python/langgraph/interrupts (confirmed: covers approval-gate mechanics and resumption; does **not** document timeout/escalation behavior — that's an application-level pattern you build on top, not something LangGraph provides out of the box).
- LangGraph overview (for general orientation, not fetched independently this session — same docs domain as the interrupts page above, use with the same confidence). https://docs.langchain.com/oss/python/langgraph/overview (not URL-verified this session)
- OpenTelemetry GenAI semantic conventions — attribute registry. https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/ (confirmed: this page now says GenAI attributes have moved and are deprecated at this URL).
- OpenTelemetry GenAI semantic conventions — current home. https://github.com/open-telemetry/semantic-conventions-genai (confirmed: covers GenAI client spans, MCP, and provider-specific conventions; explicitly Development/unstable status as of this session).

## GitHub Repositories

- `pydantic/pydantic-ai` — ~20,000 stars, 2,700 forks. Tagline: "the Python AI SDK: a typed, extensible agent loop with every model a string swap away." https://github.com/pydantic/pydantic-ai
- `guardrails-ai/guardrails` — ~7,400 stars, actively maintained (33 open issues, 48 open PRs at time of check). https://github.com/guardrails-ai/guardrails
- `NVIDIA-NeMo/Guardrails` — ~7,200 stars, Apache 2.0, Python 3.10-3.13, latest tagged release 0.24.1 at time of check. Confirms the org rename from plain `NVIDIA` to `NVIDIA-NeMo`. https://github.com/NVIDIA-NeMo/Guardrails
- `open-telemetry/semantic-conventions-genai` — the current home of GenAI/agent/tool span and attribute definitions (moved out of the main `semantic-conventions` repo). https://github.com/open-telemetry/semantic-conventions-genai
- `microsoft/agent-framework` — Microsoft's actively developed successor to AutoGen and Semantic Kernel, ~13,600 stars. Confirmed GA reached April 2026 (v1.0, Python + .NET); supports checkpointing, streaming, and human-in-the-loop workflow patterns. https://github.com/microsoft/agent-framework
- `microsoft/autogen` — now in maintenance mode (no new features, community-managed); still useful to know for interviews as "the framework Microsoft moved away from, and why." https://github.com/microsoft/autogen (not independently re-fetched this session — status corroborated via web search of multiple 2026 sources, treat the maintenance-mode fact as solid but the repo page itself not directly content-checked here).

## Articles / Interview Prep

- InterviewBit, "LLM Interview Questions and Answers (2026)" — broad LLM interview coverage including guardrails framing. https://www.interviewbit.com/llm-interview-questions-answers/ (not URL-verified this session — surfaced via search only).
- AI Engineering Insider (Substack), "AI Engineering Interview Prep: Observability & Tracing for LLM / Agent Systems." https://aiengineeringinsider.substack.com/p/ai-engineering-interview-prep-observability (not URL-verified this session — surfaced via search only).
- InterviewAIBox, "Agent Observability Interview Questions: Traces, Tool Calls, and Failure Receipts." https://interviewaibox.co/en/blog/agent-observability-interview-guide (not URL-verified this session — surfaced via search only).
- PracHub, "AI Agent System Design Interview: Planning, Tool Execution, Memory, and Human Approval." https://prachub.com/resources/ai-agent-system-design-interview-planning-tool-execution-memory-and-human-approval (not URL-verified this session — surfaced via search only).

## Notes on What to Prioritize

Pydantic AI is genuinely current and fast-moving — it ships releases essentially continuously (version 2.45.0 as of this session's check), and its human-in-the-loop tool-approval feature (`requires_approval`, `DeferredToolRequests`/`DeferredToolResults`) is a real, documented, shipped capability, not a roadmap item — this is worth naming explicitly in an interview since it distinguishes Pydantic AI from frameworks where HITL is bolted on by the application rather than provided by the agent loop itself. Second priority: know the OpenTelemetry GenAI conventions are explicitly unstable ("Development" status, moved to a separate `semantic-conventions-genai` repo) — an interviewer testing depth will notice if you present them as a long-settled standard. Third: be able to name the difference between rate limiting and cost/budget controls as two distinct failure modes, and between a guardrail (a runtime check) and a policy control (an access/spend/routing rule) — interviewers use these terms loosely and a senior candidate should disambiguate them cleanly. Fourth: the approval-gate/escalation/audit-logging design for a high-risk agent action is the single most commonly asked system-design-style question in this space — be ready to whiteboard it end to end, including the default-deny-on-timeout choice and why.
