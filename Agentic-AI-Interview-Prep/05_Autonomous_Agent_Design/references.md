# Autonomous Agent Design — References

All links below were checked this session (fetched or confirmed via search results) unless explicitly marked otherwise.

## Papers

- Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" (2022). https://arxiv.org/abs/2210.03629
- Shinn et al., "Reflexion: Language Agents with Verbal Reinforcement Learning" (2023). https://arxiv.org/abs/2303.11366
- Yao et al., "Tree of Thoughts: Deliberate Problem Solving with Large Language Models" (2023). https://arxiv.org/abs/2305.10601
- Princeton NLP's official Tree-of-Thoughts code release (companion to the paper above). https://github.com/princeton-nlp/tree-of-thought-llm

## Official Guidance

- Anthropic, "Building Effective Agents" — the workflows-vs-agents framing used throughout this topic (predefined code paths vs. LLM-directed control flow), plus the orchestrator-workers pattern referenced in the single-agent-vs-multi-agent discussion. https://www.anthropic.com/engineering/building-effective-agents

## Benchmarks (for evaluating agent reliability/failure modes)

- AgentBench (Tsinghua/THUDM) — general LLM-as-agent benchmark across environments including tool use; has an added AgentBench FC (function-calling) variant. https://github.com/THUDM/AgentBench
- tau2-bench (Sierra) — successor to the original tau-bench; evaluates tool-agent-user interactions in domains like airline/retail/telecom. (not URL-verified this session — carried forward from a prior verified session per `LLMOps-MLOps-Course/Verified_Study_Resources_and_Corrections.md`; re-check the exact repo path before citing in an interview)

## Articles / Interview Prep

- Openlayer, "AI Agent Failure Modes: Tool-Calling Errors, Infinite Loops & Propagation" — directly covers the failure taxonomy used in this topic's tutorial. https://www.openlayer.com/blog/ai-agent-failure-modes-tool-calling-loops-propagation
- systemdesignhandbook.com, "Multi-Agent System Design: A Complete Guide" — good source of the "start with the smallest design, add agents only when a stated constraint requires it" framing used in the decision-criteria section. https://www.systemdesignhandbook.com/guides/multi-agent-system-design/
- interviewnode.com, "How to Talk About Multi-Agent Systems and Orchestrations in ML Interviews." https://interviewnode.com/post/how-to-talk-about-multi-agent-systems-and-orchestrations-in-ml-interviews

## Cross-Reference

- Agent observability (tracing, span-level failure logging, root-cause clustering of hallucination triggers) is covered in depth in the sibling LLMOps course: `LLMOps-MLOps-Course/18_Tracing_and_Debugging_Agentic_Systems/`. This topic deliberately keeps that material brief and defers to that module for production observability tooling.

## Notes on What to Prioritize

Interview signal consistently points to: being able to name and defend the ReAct loop from memory (thought/action/observation), articulating Reflexion's actual mechanism (verbal self-critique persisted across attempts, not just "it retries"), and — most heavily tested at the senior level — giving a crisp, defensible answer to "when would you NOT use multi-agent orchestration." Interviewers specifically probe whether candidates default to multi-agent complexity without justifying it against a single well-tooled agent.
