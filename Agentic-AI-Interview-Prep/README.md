# Agentic AI Interview Prep

Built for a senior (8-10 yrs) data-science role centered on agentic AI solutions. Every topic folder here corresponds directly to a line in that role's qualifications, and every topic gets the full treatment — `tutorial.md`, `exercises.md`, `references.md`, `projects.md`, `scenario_qa.md` — since the qualification list itself was already the "most important topics" filter.

Sibling to `../LLMOps-MLOps-Course/` (deeper, longer-form) and `../ML-DL-NLP-Interview-Prep/` (classical ML/DL/NLP). This kit is scoped specifically to the agentic-AI qualification stack: RAG, NLQ, summarization, prompt engineering/in-context learning, single- and multi-agent design, tool calling/MCP, LLM evaluation, guardrails/observability, and AWS Bedrock.

## Topics

| # | Topic | Maps to qualification |
|---|---|---|
| 01 | [Retrieval-Augmented Generation](01_Retrieval_Augmented_Generation/) | "Hands-on expertise in RAG" |
| 02 | [Natural Language Querying (NLQ)](02_Natural_Language_Querying_NLQ/) | "...NLQ..." |
| 03 | [Summarization](03_Summarization/) | "...summarization..." |
| 04 | [Prompt Engineering & In-Context Learning](04_Prompt_Engineering_and_In_Context_Learning/) | "...prompt engineering, in-context learning..." |
| 05 | [Autonomous Agent Design](05_Autonomous_Agent_Design/) | "...autonomous agent design" |
| 06 | [Multi-Agent Systems — LangGraph](06_Multi_Agent_Systems_LangGraph/) | "single-agent and multi-agent systems using frameworks such as LangGraph" |
| 07 | [Multi-Agent Systems — Semantic Kernel & Microsoft Agent Framework](07_Multi_Agent_Systems_Semantic_Kernel_and_MS_Agent_Framework/) | "...Semantic Kernel..." — updated for the current market: AutoGen is maintenance-mode, **and so is Semantic Kernel now** — Microsoft Agent Framework (GA April 2026) is the current answer for both |
| 08 | [Tool/Function Calling & MCP](08_Tool_Function_Calling_and_MCP/) | "integrating tools and enterprise systems through function/tool calling and MCP" |
| 09 | [LLM Evaluation & Hallucination Reduction](09_LLM_Evaluation_and_Hallucination_Reduction/) | "LLM-as-a-judge, regression testing, quality benchmarking, hallucination reduction, guardrail implementation" |
| 10 | [Guardrails, Policy Controls, Observability & HITL](10_Guardrails_Policy_Controls_Observability_HITL/) | "Pydantic AI, production-safe AI applications with guardrails, policy controls, observability, and human-in-the-loop workflows" |
| 11 | [AWS Bedrock for Agentic AI](11_AWS_Bedrock_for_Agentic_AI/) | "AWS preferred, especially Amazon Bedrock, including Agents, Guardrails, and evaluation" |
| 12 | [Knowledge Graphs — RDF, OWL, Property Graphs, Neo4j](12_Knowledge_Graphs_RDF_OWL_Neo4j/) | From a related posting: "design and model knowledge graphs using RDF, OWL, and property graphs... utilize graph databases, particularly Neo4j... integrate LLMs with knowledge graphs for retrieval and reasoning" |

## Suggested order

1. **01 → 04** (the core GenAI application skills: RAG, NLQ, summarization, prompting)
2. **05 → 07** (agent architecture: single-agent patterns, then the two Microsoft-ecosystem multi-agent frameworks)
3. **08** (tool calling and MCP — the integration layer every agent above depends on)
4. **09 → 10** (evaluation and safety — what separates a demo from a production system)
5. **11** (AWS Bedrock — how a cloud-managed stack compares to everything built by hand in 05-10)
6. **12** (Knowledge graphs — a distinct data-modeling and retrieval track; pairs directly with topic 01's GraphRAG/multi-hop-RAG section and topic 02's NLQ text-to-query pattern, but stands on its own for RDF/OWL/Neo4j-specific roles)

## Important — this space moves fast; read this before trusting anything from memory

Every `references.md` file in this kit was built from live WebSearch/WebFetch research done at the time of writing (not from training-data memory), because several load-bearing facts changed recently enough that stale answers would actively hurt you in an interview:

- **AutoGen is in maintenance mode.**
- **Semantic Kernel is now ALSO in maintenance mode** (confirmed directly from its own GitHub README).
- **Microsoft Agent Framework (MAF)** reached **1.0 GA in April 2026** and is the official successor to *both* — combining AutoGen's multi-agent abstractions with Semantic Kernel's enterprise features (type safety, middleware, telemetry), plus new graph-based workflows. If asked about Microsoft's agent stack in 2026, lead with MAF; mention SK/AutoGen only as legacy context.
- **Amazon Bedrock Agents was renamed "Agents Classic" and closed to new customers (July 30, 2026).** The current recommended path is **Amazon Bedrock AgentCore** (Runtime, Gateway, Memory, Identity, Observability, Evaluations) — a framework-agnostic platform that also runs LangGraph/CrewAI/Strands agents. Topic 11 covers Agents Classic (still runs existing production systems, still interview-relevant) but frames AgentCore as the current answer.
- **MCP (Model Context Protocol)** has moved substantially — current spec is a large stateless-wire-protocol revision; adoption is now broad (OpenAI, Google, and Microsoft have all adopted it). Verify anything MCP-specific against `modelcontextprotocol.io` directly rather than a blog post — one was caught mid-research citing an already-stale spec version.
- **τ-bench → τ²-bench → τ³-bench**: the agent-evaluation benchmark has been superseded twice since it was first released.
- **Pydantic AI** is more mature than its "newer framework" reputation suggests — v2.45+, ~20k GitHub stars, with a first-party human-in-the-loop tool-approval mechanism built into the agent loop itself (not bolted on).
- **Vanna.ai** (a leading open-source text-to-SQL tool) was **archived/read-only on March 29, 2026** — don't present it as a current option (topic 02).
- **Spider 1.0** (the standard text-to-SQL benchmark) retired in Feb 2024, superseded by **Spider 2.0**; exact-match SQL evaluation is now widely considered obsolete in favor of execution accuracy (topic 02).
- **EchoLeak (CVE-2025-32711)** — a real, zero-click indirect prompt-injection exploit against Microsoft 365 Copilot — is a live, citable example for prompt-injection discussions (topic 04).
- **LangChain and LlamaIndex both restructured their RAG/summarization docs** in 2025-2026 toward a "Deep Agents" retrieve-offload-delegate framing using subagents, replacing the old fixed retrieval-chain / map-reduce-chain tutorials — a real signal that the field is shifting from fixed pipelines toward agentic orchestration even for "plain" RAG and summarization (topics 01, 03).
- **DSPy** (~38k stars) remains the leading automatic-prompt-optimization framework, now with real competition from **TextGrad** (a different mental model — textual gradients from a critic model vs. DSPy's compiled-module search) (topic 04).
- Industry commentary increasingly frames "prompt engineering" as being absorbed into broader agent-engineering work, sometimes now called **"context engineering"** — worth using deliberately when positioning an 8-10 year senior candidate (topic 04).
- **Microsoft's own GraphRAG project is now largely in maintenance mode** (bug/security fixes only, no new features) — its maintainers explicitly cite that frontier-model capabilities have moved on since its July 2024 release. **LightRAG** (HKUDS, ~39.8k stars) has overtaken it in popularity and positions itself as a lower-cost, more efficient alternative — know both names, not just "GraphRAG" (topic 12).
- **Neo4j's GenAI tooling is three separately-maintained projects at different maturity levels**, not one product: `neo4j-graphrag-python` (flagship, active), the LLM Knowledge Graph Builder (active product), and `text2cypher` (a smaller, lower-activity Labs research repo) — and **Cypher itself forked into "Cypher 25" (active) vs. "Cypher 5" (frozen)** as of Neo4j 2025.06 (topic 12).

Every `references.md` file marks anything that wasn't individually fetched-and-confirmed this session as `(not URL-verified this session)`, with enough author/title detail to find it yourself — nothing was fabricated, but treat those specific items as "very likely real, double-check the exact link before relying on it."

## Scenario Q&A — what's covered

Every topic's `scenario_qa.md` uses "Situation → what would you do and why" format grounded in real, currently-reported interview-question patterns, not generic definitions. Notable deliberate inclusions:
- Recommending **Microsoft Agent Framework over AutoGen/Semantic Kernel** for new work (07), and migrating an existing SK system toward MAF (07).
- Debugging a LangGraph `InvalidUpdateError` from concurrent state writes, and choosing a checkpointer backend for production (06).
- An LLM-as-judge giving inconsistent verdicts (bias diagnosis), and a hallucination slipping past guardrails into a customer-facing response (09).
- Designing a human-in-the-loop approval gate for a high-risk action, e.g. an agent that can issue refunds (10).
- Choosing Bedrock's managed Agents/Guardrails vs. a custom-built stack for a client engagement (11).
- Diagnosing entity-resolution failures fragmenting a knowledge graph into duplicate nodes, catching a hallucinated relationship in a GraphRAG answer via a deterministic triple-existence check, and fixing ontology drift from ungoverned relationship-type proliferation (12).
