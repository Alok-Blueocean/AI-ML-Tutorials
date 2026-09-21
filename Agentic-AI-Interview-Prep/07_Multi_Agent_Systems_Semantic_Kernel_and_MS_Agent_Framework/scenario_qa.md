# Multi-Agent Systems: Semantic Kernel and Microsoft Agent Framework — Scenario-Based Q&A

**Situation:** You're kicking off a brand-new multi-agent project on Microsoft's stack in late 2026. A colleague suggests "AutoGen for the multi-agent orchestration and Semantic Kernel for the enterprise plugin layer, since that's the standard combo." What would you do and why?

Model answer: Push back on the premise, not just the naming. Both AutoGen and Semantic Kernel are officially in maintenance mode — their own GitHub READMEs say so explicitly, with Microsoft Agent Framework named as the direct, same-team successor to both. The "combine AutoGen's orchestration with SK's enterprise features" idea your colleague is describing is exactly what Microsoft Agent Framework already does natively, as a single framework, rather than two aging ones stitched together. Recommend Microsoft Agent Framework for the new project: it gives you AutoGen-style multi-agent patterns (Sequential, Concurrent, Handoff, Group Chat, Magentic) plus Semantic-Kernel-grade enterprise features (typed state, telemetry, middleware) in one actively-developed framework. Reserve AutoGen/Semantic Kernel knowledge for maintaining existing systems, not for new architecture decisions.

---

**Situation:** Your team owns a Semantic Kernel-based production system that's been stable and unmodified for over a year. Leadership asks whether it should be migrated to Microsoft Agent Framework this quarter. What would you do and why?

Model answer: Don't default to "yes, migrate" just because MAF is newer — Microsoft's own announcement is explicit that MAF "doesn't replace Semantic Kernel and AutoGen — it builds on them," and both predecessors remain supported. Ask what's actually driving the request: is there a missing capability (a newer model provider integration, a workflow shape SK can't express), a support/security concern, or team bandwidth being spent working around SK's limitations? If none of those are present and the system is stable, recommend deferring a full migration and instead adopting the incremental bridge path — new features get built as MAF agents using `.as_agent_framework_tool()` to reuse existing `KernelFunction`s, so the codebase migrates gradually as it's touched rather than through a risky, unscheduled big-bang rewrite of working code.

---

**Situation:** A junior engineer is building a new customer-support agent and writes a `ChatCompletionAgent` with a `Kernel` and a `@kernel_function`-decorated plugin, saying "this is the standard Semantic Kernel pattern from the tutorials I found." What would you do and why?

Model answer: Flag that most SK tutorials predate the shift to Microsoft Agent Framework and are teaching a pattern Microsoft itself now frames as legacy. Redirect them to build the same agent in MAF instead: a plain function with type hints and a docstring passed directly into `tools=[...]` on an agent built from a chat client, no `Kernel` or `Plugin` wrapping required. If they specifically need to reuse SK code that already exists elsewhere in the codebase, show them the `.as_agent_framework_tool()` bridge rather than reverting to the full SK pattern for new code. The teaching moment here is as much about where they're getting their reference material as about the specific API.

---

**Situation:** You're asked in an interview to design a multi-agent expense-report-review system for a company standardized on Azure/Microsoft tooling. What would you do and why?

Model answer: Lead with Microsoft Agent Framework, not Semantic Kernel or AutoGen. Propose a Sequential orchestration for the fixed-order parts (intake -> policy check -> approval-tier routing) with a Handoff to a specialist agent for flagged/ambiguous expenses that need deeper investigation, and an approval-required tool gating any auto-approval above a spending threshold — the same human-in-the-loop shape as topic 06's LangGraph claims example, expressed through MAF's tool-approval mechanism. Mention Semantic Kernel only if asked about integrating with an existing SK-based system already in production, and frame it explicitly as legacy integration, not the new system's foundation.

---

**Situation:** Someone on your team read an older article recommending "AutoGen's group chat pattern for multi-agent debate/consensus tasks" and wants to prototype a new feature that way. What would you do and why?

Model answer: Acknowledge the underlying pattern is sound — group-chat-style multi-agent debate is a real, useful shape — but point out AutoGen itself is in maintenance mode and won't receive new features. Recommend prototyping the same pattern using MAF's Group Chat orchestration instead, which is the direct, actively-developed descendant of exactly the AutoGen capability being referenced. This is a good example of separating a good architectural idea (group-chat multi-agent coordination) from the specific now-legacy framework that popularized it.

---

**Situation:** A production Semantic Kernel agent uses SK's `VectorStore` integration (e.g., Azure AI Search) via a `create_search_function`-generated `KernelFunction`. The team wants to move the agent layer to Microsoft Agent Framework but doesn't want to rebuild the vector search integration. What would you do and why?

Model answer: This is exactly the case the `.as_agent_framework_tool()` bridge is designed for — it works on `KernelFunction`s generated from SK's `VectorStore` search integrations, not just plain plugin methods. Convert the existing search function into an MAF tool with the bridge, keep the underlying SK `VectorStore` connector code untouched, and wire the bridged tool into the new MAF agent. This lets the team migrate the agent/orchestration layer first — arguably where most of the value of moving to MAF is — while deferring a rewrite of infrastructure-level integrations that aren't the bottleneck.

---

**Situation:** During a system-design interview, you're asked to compare Microsoft Agent Framework's Handoff orchestration to LangGraph's swarm pattern and explain when you'd reach for a Microsoft-stack answer versus a LangGraph answer at all. What would you do and why?

Model answer: First answer the conceptual comparison: both express the same idea — agents transferring control to each other directly based on context, with no fixed central coordinator — Handoff as a named, built-in MAF orchestration, swarm as a shape you compose yourself from LangGraph's lower-level graph primitives (or a swarm helper library). Then answer the "which stack" question honestly: it's rarely a technical differentiator at this level of the decision — pick based on where the team already has investment (Azure/Microsoft ecosystem and .NET or Python vs. an existing LangChain/LangGraph codebase), which model providers and hosting you need first-class support for, and which framework's operational tooling (observability, deployment) your infrastructure already integrates with, rather than a claim that one is architecturally superior for this specific pattern.

---

**Situation:** A stakeholder who read about Microsoft Agent Framework's Go SDK wants to build a new multi-agent Go service using it, citing "Microsoft's own multi-language framework." What would you do and why?

Model answer: Check current status before committing: MAF's Go SDK is explicitly documented as public preview, missing declarative agents, RAG, CodeAct, and functional workflows that the Python and .NET SDKs already have. Recommend confirming which specific capabilities the project needs against that gap list before committing to Go — if the project needs any of the missing pieces, either wait, use Python/.NET for the agent layer with a Go wrapper for the rest of the service, or accept the preview-level risk explicitly and get stakeholder sign-off on it rather than assuming full parity.

---

**Situation:** In a retrospective, a teammate says "our AutoGen-based multi-agent prototype from last year never made it to production, so multi-agent architectures on Microsoft's stack must not be production-ready." What would you do and why?

Model answer: Separate the framework's fate from the architecture's viability. AutoGen was explicitly framed by its own maintainers as "experimental multi-agent orchestration from research" — a reasonable prototyping tool, but not positioned as a production-hardened target, which is precisely the gap Microsoft Agent Framework was built to close by adding enterprise-grade state management, telemetry, and observability on top of the same multi-agent ideas. The conclusion to draw isn't "multi-agent doesn't work on this stack," it's "the prototype was built on the research-grade predecessor, not the production-grade successor" — recommend re-evaluating the same architecture on MAF before writing off the approach.
