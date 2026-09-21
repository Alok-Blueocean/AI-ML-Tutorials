# Multi-Agent Systems: LangGraph — References

All links below were fetched and confirmed live this session unless explicitly marked otherwise.

## Official Docs

- LangGraph overview (docs moved here from the old `langchain-ai.github.io/langgraph/` URL, confirmed live). https://docs.langchain.com/oss/python/langgraph/overview
- Human-in-the-loop / `interrupt()` guide — confirms the `interrupt()` + `Command(resume=...)` pattern used throughout the tutorial, and the detail that a resumed node restarts from its beginning rather than resuming mid-node. https://docs.langchain.com/oss/python/langgraph/interrupts
- Persistence and checkpointing guide — confirms the three checkpointer implementations named in the tutorial (`InMemorySaver`/`MemorySaver`, `SqliteSaver`, Postgres-backed `PostgresSaver`/`AsyncPostgresSaver`) and the `thread_id` mechanism. Note the documented constraint that `thread_id` must stay under 255 characters on Postgres. https://docs.langchain.com/oss/python/langgraph/persistence
- Subgraphs guide — covers using a compiled graph as a node in a parent graph, the two composition styles (calling a subgraph inside a node vs. adding it directly as a node), and the three subgraph persistence modes (per-invocation, per-thread, stateless). https://docs.langchain.com/oss/python/langgraph/use-subgraphs
- Workflows and agents guide — covers the workflow patterns (prompt chaining, parallelization, routing, orchestrator-worker, evaluator-optimizer) that the supervisor/orchestrator-worker section of the tutorial builds on. This is LangChain's general agent-patterns page, not multi-agent-specific. https://docs.langchain.com/oss/python/langgraph/workflows-agents

## Multi-Agent Supervisor Pattern

- `langgraph-supervisor-py` — the official LangChain-AI-maintained library implementing the supervisor pattern (central supervisor coordinating specialized agents via tool-based handoffs, with support for hierarchical multi-level supervisors). This is the current canonical reference for the supervisor pattern; a previous standalone "multi-agent supervisor tutorial" page under `docs.langchain.com/oss/python/langgraph/` appears to have been removed or relocated in a docs reorganization — could not locate a direct replacement URL this session, so this repo is cited instead. https://github.com/langchain-ai/langgraph-supervisor-py

## GitHub / Core Repo

- LangGraph — official repository. Described in its own README as "low-level orchestration framework for building stateful agents," with durable execution, human-in-the-loop, memory, and LangSmith-based debugging called out as core capabilities. https://github.com/langchain-ai/langgraph

## LangSmith (Observability/Eval — Separate Product)

- LangSmith docs home — tracing, dataset-based evaluation, and production monitoring for LangGraph (and other) agents; confirms LangSmith is a distinct product from LangGraph itself, not a built-in LangGraph feature. https://docs.langchain.com/langsmith/home

## Articles / Tutorials

- LangChain, "LangGraph: Multi-Agent Workflows" — LangChain's own engineering blog post walking through three concrete multi-agent architectural patterns built on LangGraph nodes. Cited as an official-adjacent source (LangChain's blog, not third-party), included because it's the most direct worked example of multi-agent patterns available. https://www.langchain.com/blog/langgraph-multi-agent-workflows
- DataCamp, "How to Build LangGraph Agents: Hands-On Tutorial" — genuinely third-party tutorial covering state/nodes/edges/memory fundamentals and a ReAct agent walkthrough; useful as an independent second explanation of the core primitives, though it does not itself cover supervisor/multi-agent patterns in depth. https://www.datacamp.com/tutorial/langgraph-agents

## Notes on What to Prioritize

Interview signal for this topic concentrates on: being able to draw the state/nodes/edges model from memory and explain cycles vs. DAGs precisely, naming the exact mechanism behind human-in-the-loop (`interrupt()` + `Command(resume=...)`, checkpointer-backed), being precise about what checkpointing does and doesn't guarantee (between-step, not mid-step, durability), and giving a crisp, criteria-based answer to supervisor vs. swarm rather than a default preference. As with topic 05, interviewers specifically probe whether a candidate reaches for a multi-agent LangGraph design when a single well-tooled agent (topic 05) would suffice.
