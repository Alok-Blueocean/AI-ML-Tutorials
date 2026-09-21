# Multi-Agent Systems: Semantic Kernel and Microsoft Agent Framework — Condensed Study Notes

Docs referenced throughout: `learn.microsoft.com/en-us/agent-framework/` (Microsoft Agent Framework) and `learn.microsoft.com/en-us/semantic-kernel/` (Semantic Kernel, now framed by Microsoft itself as legacy/maintenance-mode).

## The Current State of Microsoft's Agent Stack — Read This First

- As of this writing (verified via the frameworks' own official docs and GitHub repos), **both Semantic Kernel and AutoGen are in maintenance mode**. Microsoft's actively-developed successor to both is the **Microsoft Agent Framework (MAF)**, which reached a production-ready 1.0 release in April 2026 and is on active releases since (the Python `agent-framework` package was at version 1.19.0 as of September 2026, marked "Production/Stable").
- This isn't an inference from indirect signals — it's stated directly. The Semantic Kernel GitHub README now reads: "Semantic Kernel is now Microsoft Agent Framework! Microsoft Agent Framework (MAF) is the enterprise-ready successor to Semantic Kernel." The AutoGen README carries an explicit banner: "Maintenance Mode: AutoGen is now in maintenance mode. It will not receive new features or enhancements and is community managed going forward," and likewise points to MAF as "the enterprise-ready successor to AutoGen."
- Real-world example: a candidate in a 2026 interview who answers "I'd use AutoGen for the multi-agent piece and Semantic Kernel for the enterprise plugin layer" is describing an architecture built on two frameworks whose own maintainers now point away from them for new work — the answer signals stale knowledge even if the reasoning about *why* those frameworks existed is otherwise correct.

## Semantic Kernel: What It Was and Why It Still Matters as Legacy Context

- **Kernel**: the central object every SK agent depended on — a lightweight dependency-injection container that wires together AI services (chat completion, embeddings), registered plugins, and configuration. Every agent class in SK required a `Kernel` instance, even an empty one.
- **Plugins / functions**: native code or prompt templates exposed to the model as callable functions. In Python, a method decorated `@kernel_function` becomes something the model can invoke; multiple such functions are grouped into a `Plugin` and registered on the `Kernel`.
- **Planners (historic, now removed)**: SK originally shipped "planner" abstractions (notably the Stepwise and Handlebars planners) that used prompts to get the model to choose which functions to call, before native LLM function-calling existed. Once OpenAI and other providers added native function-calling, SK evolved to use that directly instead — the Stepwise and Handlebars planners have since been **deprecated and removed from the package entirely** (not just discouraged), with a documented migration guide for anyone still depending on them.
- **Agents-in-SK**: SK's Agent Framework layered specific agent classes on top of the Kernel — `ChatCompletionAgent` for plain chat-completion-based inference, `OpenAIAssistantAgent` for the OpenAI Assistants API, `AzureAIAgent` for the Foundry Agent Service — each requiring its own thread type (e.g., `ChatHistoryAgentThread`, `OpenAIAssistantAgentThread`) that the caller had to know about and create manually.
- Real-world example: a support-ticket assistant built in 2024 on SK, with a `TicketingPlugin` class whose methods are decorated `@kernel_function` and registered on a `Kernel` passed into a `ChatCompletionAgent` — a system that is very plausibly still running in production in 2026, and exactly the kind of system a senior candidate might be asked how to modernize.

```python
# Semantic Kernel pattern — shown for legacy/interview context, not as a recommended starting point for new work.
from semantic_kernel.functions import kernel_function
from semantic_kernel.agents import ChatCompletionAgent
from semantic_kernel.connectors.ai.open_ai import OpenAIChatCompletion

class TicketingPlugin:
    @kernel_function(name="get_ticket", description="Look up a support ticket by ID")
    def get_ticket(self, ticket_id: str) -> str:
        return lookup_ticket(ticket_id)

agent = ChatCompletionAgent(
    service=OpenAIChatCompletion(),
    name="SupportAgent",
    instructions="Help resolve support tickets using the ticketing plugin.",
    plugins=[TicketingPlugin()],
)
```

*(SK API surface simplified for clarity — verify exact current signatures against Microsoft Learn before quoting from memory; this is presented as legacy/maintenance-mode context, not the recommended path for new work.)*

## Microsoft Agent Framework: The Current Answer

- MAF is described in its own official overview as combining "AutoGen's simple abstractions for single- and multi-agent patterns with Semantic Kernel's enterprise-grade features such as session-based state management, type safety, filters, telemetry, and extensive model and embedding support," plus new graph-based **workflows** for explicit multi-agent orchestration control. Microsoft's own docs state plainly: "Agent Framework is the next generation of both Semantic Kernel and AutoGen," built by the same teams behind both predecessors.
- It's open-source and multi-language: .NET and Python are the primary, fully-featured surfaces; a Go SDK exists but is explicitly called out as being in **public preview**, missing features (declarative agents, RAG, CodeAct, functional workflows) that .NET and Python already have — don't assume feature parity across languages in an interview answer.
- The `Kernel`-as-DI-container concept disappears entirely. Agents are built directly from a chat client: `chat_client.as_agent(instructions=..., tools=[...])`. There's no `Plugin` wrapping requirement — a plain Python function with type hints and a docstring becomes a tool, and its name/docstring become the tool's name/description automatically (an optional `@tool` decorator lets you override those explicitly).
- Sessions replace SK's per-provider thread types: `agent.create_session()` gives you a session the agent manages, instead of the caller having to know and construct the right `AgentThread` subclass for each provider.
- Real-world example: rewriting the SK-based `TicketingPlugin` above for MAF removes the `@kernel_function` decorator, the `Plugin` wrapper, and the `Kernel` entirely — `get_ticket` becomes a plain function passed straight into `tools=[get_ticket]` at agent-creation time.

```python
# Microsoft Agent Framework equivalent of the SK example above.
from agent_framework.openai import OpenAIChatClient

def get_ticket(ticket_id: str) -> str:
    """Look up a support ticket by ID."""
    return lookup_ticket(ticket_id)

agent = OpenAIChatClient().as_agent(
    instructions="Help resolve support tickets using the ticketing tool.",
    tools=[get_ticket],
)
response = await agent.run("What's the status of ticket 4521?")
```

*(Verified against Microsoft's official Semantic-Kernel-to-Agent-Framework migration guide as of this session; MAF's Python API is under active development, so re-check exact method names before quoting from memory in an interview.)*

## Bridging Existing Semantic Kernel Code Instead of Rewriting Everything at Once

- MAF explicitly supports a **compatibility bridge**: an existing `KernelFunction` (from a plugin method or even a prompt template) can be converted into an MAF-compatible tool via `.as_agent_framework_tool()`, without touching the original SK code. This requires `semantic-kernel` version 1.38 or higher.
- This matters for the realistic migration scenario: a production SK system with dozens of plugins doesn't need a big-bang rewrite. Individual `KernelFunction`s (including ones backed by SK's `VectorStore` search integrations) can be bridged into new MAF agents one at a time, letting a team migrate incrementally while both systems run.
- Real-world example: a team migrating the ticketing bot doesn't need to reimplement every plugin on day one — they wrap the existing `TicketingPlugin`'s `KernelFunction`s with `.as_agent_framework_tool()`, ship a new MAF-based agent using the old plugin code unchanged, and only rewrite plugins as plain functions opportunistically over subsequent sprints.

## Multi-Agent Orchestration Patterns in Microsoft Agent Framework

MAF's **workflows** capability provides named, built-in multi-agent orchestration patterns (distinct from LangGraph's approach in topic 06 of composing your own graph from lower-level node/edge primitives):

- **Sequential**: agents execute one after another in a defined order — e.g., a draft-writer agent followed by an editor agent.
- **Concurrent**: multiple agents execute in parallel on the same input, useful when you want several independent perspectives before synthesizing a result.
- **Handoff**: agents transfer control to each other based on context — conceptually the same shape as LangGraph's swarm/peer-to-peer pattern from topic 06, expressed as a first-class named orchestration rather than something you wire up yourself from graph primitives.
- **Group Chat**: agents collaborate in a shared, visible conversation — this is the pattern AutoGen originally pioneered, now offered as a built-in MAF orchestration.
- **Magentic**: a manager agent dynamically coordinates specialized agents on open-ended tasks, closer to LangGraph's supervisor pattern but for less rigidly-defined workflows than Sequential provides.
- All of these orchestrations support **human-in-the-loop** via approval-required tools that pause the workflow pending review before a tool executes — the same underlying need as LangGraph's `interrupt()`, addressed through MAF's own tool-approval mechanism rather than a general-purpose `interrupt()` call.
- Real-world example: the same claims-processing domain used throughout topic 06 modeled in MAF as a Sequential orchestration (intake -> extraction -> decision), with a Handoff to a fraud-specialist agent when red flags appear during extraction, and an approval-required tool gating any `pay_claim` call above $5,000 — functionally the same design as the LangGraph version, expressed through MAF's named orchestrations and tool-approval primitive instead of a hand-composed graph and `interrupt()`.

## Migrating an Existing Semantic Kernel System Toward Microsoft Agent Framework

Concrete, documented differences worth naming precisely in an interview:

- **Namespace/package**: `semantic_kernel` imports become `agent_framework` imports; the package is split into a core package plus provider-specific packages (e.g., `agent-framework-openai`, `agent-framework-foundry`) rather than one monolithic package with extras.
- **No `Kernel` dependency**: agents are built directly from a chat client rather than requiring a `Kernel` instance.
- **No mandatory plugin wrapping**: tools are plain functions with type hints and docstrings; the `@kernel_function` decorator and `Plugin`/`KernelPluginFactory` machinery are no longer required (though bridgeable, as above).
- **Agent type consolidation**: SK's multiple provider-specific agent classes (`ChatCompletionAgent`, `OpenAIAssistantAgent`, `AzureAIAgent`) collapse into a smaller set of MAF types built around a common chat-client abstraction.
- **Sessions replace manually-typed threads**: `agent.create_session()` instead of constructing the right `AgentThread` subclass yourself.
- **Method renames**: `invoke`/`invoke_stream` become `run`/`run(..., stream=True)`; return types change from an async-iterator-of-messages pattern to a single `AgentResponse` object.
- Real-world example: migrating the support bot doesn't require deciding "SK or MAF" as an irreversible switch — Microsoft's own migration guide is written around exactly this kind of function-by-function, agent-by-agent transition, using the `.as_agent_framework_tool()` bridge to keep existing plugin code working while new agent code is written against MAF.

## Migrating an Existing AutoGen System Toward Microsoft Agent Framework

- AutoGen's core abstractions — conversable agents and group-chat-style multi-agent conversation — map most directly onto MAF's **Group Chat** orchestration, with MAF adding the enterprise plumbing (typed state, telemetry, middleware, session management) that AutoGen's research-oriented design didn't prioritize.
- AutoGen's more experimental hand-off-style patterns map onto MAF's dedicated **Handoff** orchestration.
- Real-world example: a research prototype built on AutoGen's group-chat pattern for a multi-agent debate/critique system is a reasonable candidate for a Group Chat orchestration rewrite in MAF if it's being hardened for production use, since the conversational shape carries over directly while the surrounding state/observability gaps get filled in.

## What to Say in an Interview

- If asked "would you use AutoGen or Semantic Kernel for a new multi-agent project on Microsoft's stack in 2026?" — the current-knowledge answer is neither: name **Microsoft Agent Framework**, and explain that it's the explicit, same-team successor to both, not a third competing option.
- If asked to design a Microsoft-stack multi-agent system from scratch, lead with MAF's named orchestrations (Sequential/Concurrent/Handoff/Group Chat/Magentic) and its tool-approval-based human-in-the-loop mechanism; mention Semantic Kernel and AutoGen only as prior art or as context for a migration question, not as the primary recommendation.
- If asked about an *existing* production system built on Semantic Kernel, the correct framing is maintenance-mode reality, not obsolescence panic: it keeps running, a full rewrite is rarely justified on day one, and the realistic path is incremental migration using the documented bridge (`.as_agent_framework_tool()`) rather than a rewrite-everything mandate.
- Real-world example: a system-design interview prompt like "design a multi-agent claims-processing system on Microsoft's stack" is best answered by naming MAF's Sequential/Handoff orchestrations and its approval-required tools directly, the same way topic 06 would answer the equivalent LangGraph prompt with supervisor/swarm and `interrupt()` — reaching for AutoGen or bare Semantic Kernel here is the single most common stale-knowledge signal on this topic.

## How This Relates to LangGraph (Topic 06)

- Both frameworks solve the same underlying problems — multi-agent coordination, state management, human approval gates — but at different altitudes. LangGraph gives you low-level graph primitives (nodes, edges, your own conditional routing) and expects you to compose the shape you need, including a supervisor or swarm pattern from scratch or via a helper library. MAF gives you pre-built, named orchestrations (Sequential, Concurrent, Handoff, Group Chat, Magentic) that cover the same shapes without you wiring the graph yourself.
- Practically: LangGraph's supervisor/orchestrator-worker pattern (topic 06) corresponds most closely to MAF's Magentic orchestration for open-ended coordination or Sequential for fixed-order coordination; LangGraph's swarm/peer-to-peer hand-off pattern corresponds to MAF's Handoff orchestration.
- Real-world example: a team already committed to LangGraph for a non-Microsoft-hosted deployment has no strong reason to introduce MAF alongside it — the two are competing answers to the same architectural question, not complementary layers, and mixing them typically adds integration overhead rather than removing it.

## Confidence Note on Sparser Documentation

- LangGraph (topic 06) has had years to accumulate mature, stable documentation and a large third-party tutorial ecosystem. MAF reached its 1.0 GA far more recently (April 2026), so official documentation for some newer or more advanced surfaces — the exact Magentic manager API, the newer "harness agent" capability referenced in MAF's own docs — is thinner and more likely to change. Treat the code sketches in this document as illustrative of confirmed, documented concepts (verified against Microsoft's own docs and GitHub repos this session), not as memorized exact syntax — re-check current docs before quoting specific method signatures in an interview, exactly as topic 06 cautions for LangGraph's own evolving Python API.

## Quick Gotchas Worth Naming in an Interview

- Don't answer "AutoGen" or bare "Semantic Kernel" when asked for a 2026 recommendation on Microsoft's agent stack — both projects' own READMEs now carry maintenance-mode/successor notices pointing to Microsoft Agent Framework.
- Semantic Kernel's Stepwise and Handlebars planners are **removed from the package**, not merely deprecated-but-usable — native function calling is the only planning mechanism in current Semantic Kernel, and there's a dedicated migration guide for anyone still depending on the old planners.
- MAF has no `Plugin`/`Kernel` concept — tools are plain functions, and the DI-container role the `Kernel` used to play is gone. Don't describe MAF agent-building using SK plugin vocabulary in an interview; it signals you're describing the predecessor, not the current framework.
- MAF's Go SDK is public preview only, missing declarative agents, RAG, CodeAct, and functional workflows that .NET and Python already support — don't claim cross-language feature parity.
- MAF "doesn't replace Semantic Kernel and AutoGen — it builds on them," per Microsoft's own announcement; existing systems on either predecessor continue to function and receive support, so "migrate immediately" is not automatically the right recommendation for a stable production system with no active pain point.
