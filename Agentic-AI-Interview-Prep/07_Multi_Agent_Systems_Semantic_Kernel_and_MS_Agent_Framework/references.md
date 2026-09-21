# Multi-Agent Systems: Semantic Kernel and Microsoft Agent Framework — References

All links below were fetched and confirmed live this session unless explicitly marked otherwise.

## Microsoft Agent Framework — Official Docs

- Microsoft Agent Framework overview ("Why Agent Framework?" section) — the primary source for the framing used throughout this topic: MAF as "the direct successor, created by the same teams" behind Semantic Kernel and AutoGen, combining AutoGen's agent abstractions with SK's enterprise features. https://learn.microsoft.com/en-us/agent-framework/overview/agent-framework-overview
- Semantic Kernel to Microsoft Agent Framework migration guide — the source for every concrete migration detail in the tutorial (namespace changes, `Kernel` removal, tool registration simplification, agent type consolidation, the `.as_agent_framework_tool()` compatibility bridge, `invoke`-to-`run` renames). Dated 2026-04-01, consistent with MAF's April 2026 1.0 timeline. https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-semantic-kernel
- AutoGen to Microsoft Agent Framework migration guide — companion migration guide for teams coming from AutoGen rather than Semantic Kernel; also dated 2026-04-01. https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen
- Workflow capabilities overview — the source for MAF's workflow composition, human-in-the-loop, checkpointing, and multi-agent orchestration surfaces as a category. https://learn.microsoft.com/en-us/agent-framework/workflows/
- Workflow orchestrations page — the source for the five named orchestration patterns used throughout this topic (Sequential, Concurrent, Handoff, Group Chat, Magentic) and the note that all of them support human-in-the-loop via approval-required tools. https://learn.microsoft.com/en-us/agent-framework/workflows/orchestrations/

## Microsoft Agent Framework — Repo and Package

- Microsoft Agent Framework GitHub repository — confirms multi-language support (.NET, Python, Go), active development activity, and links to the migration guides from both Semantic Kernel and AutoGen. https://github.com/microsoft/agent-framework
- `agent-framework` package on PyPI — confirms the current Python package version (1.19.0 as of September 18, 2026) and its "Production/Stable" classifier, used as the basis for this topic's GA/maturity framing. https://pypi.org/project/agent-framework/
- Microsoft, "Introducing Microsoft Agent Framework" (Microsoft Foundry devblog) — the original announcement, confirming MAF "doesn't replace Semantic Kernel and AutoGen — it builds on them" and that both predecessors remain supported while "most investment is now focused on Microsoft Agent Framework." https://devblogs.microsoft.com/foundry/introducing-microsoft-agent-framework-the-open-source-engine-for-agentic-ai-apps/

## Semantic Kernel — Official Docs (Legacy/Maintenance-Mode Context)

- Semantic Kernel GitHub repository — its README now carries the direct notice used throughout this topic: "Semantic Kernel is now Microsoft Agent Framework! Microsoft Agent Framework (MAF) is the enterprise-ready successor to Semantic Kernel," alongside a pointer to the migration guide. https://github.com/microsoft/semantic-kernel
- Semantic Kernel overview (Microsoft Learn) — general introduction to the Kernel/plugins/enterprise-connector model, useful for describing what SK offered as legacy context. https://learn.microsoft.com/en-us/semantic-kernel/overview/
- "What are Planners in Semantic Kernel" — the source confirming the Stepwise and Handlebars planners were deprecated and **removed** from the package (not merely discouraged) in favor of native function calling, with a dedicated Stepwise Planner Migration Guide referenced for anyone still depending on them. https://learn.microsoft.com/en-us/semantic-kernel/concepts/planning

## AutoGen — Maintenance-Mode Notice

- AutoGen GitHub repository — its README carries the explicit banner: "Maintenance Mode: AutoGen is now in maintenance mode. It will not receive new features or enhancements and is community managed going forward," with Microsoft Agent Framework named as "the enterprise-ready successor to AutoGen." Cited for the maintenance-mode framing rather than any specific AutoGen API detail. https://github.com/microsoft/autogen

## Cross-Reference

- LangGraph's own multi-agent orchestration model (supervisor/orchestrator-worker, swarm/peer-to-peer hand-off, subgraphs) is covered in depth in the sibling topic `06_Multi_Agent_Systems_LangGraph/`. This topic deliberately maps MAF's named orchestrations (Sequential, Concurrent, Handoff, Group Chat, Magentic) back onto that topic's vocabulary rather than re-deriving multi-agent concepts from scratch.

## Notes on What to Prioritize

Interview signal for this topic is concentrated almost entirely on currency: being able to state plainly, without hedging, that both Semantic Kernel and AutoGen are in maintenance mode and that Microsoft Agent Framework is their explicit, same-team successor as of its April 2026 1.0 release. A secondary, real signal is being able to describe *why* MAF exists in one sentence (AutoGen's simple multi-agent abstractions plus Semantic Kernel's enterprise features, plus new graph-based workflows) rather than treating it as an unexplained rebrand. A candidate who can also name at least two of the five orchestration patterns (Sequential, Concurrent, Handoff, Group Chat, Magentic) and map one of them onto the LangGraph vocabulary from topic 06 is demonstrating senior-level breadth across the Microsoft and LangChain ecosystems rather than memorized facts about one framework in isolation.
