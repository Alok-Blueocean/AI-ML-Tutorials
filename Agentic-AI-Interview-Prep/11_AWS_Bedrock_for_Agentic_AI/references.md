# AWS Bedrock for Agentic AI — References

All links below were fetched and content-verified this session (2026-09-18) unless explicitly marked otherwise. Bedrock's agent stack changed structurally this year (Agents Classic maintenance mode, AgentCore as the new platform) — do not trust any pre-2026 blog post or course video on this topic without cross-checking against the official docs URLs below first.

## Official Docs — Bedrock Agents / AgentCore

- Amazon Bedrock Agents overview (now labeled "Amazon Bedrock Agents Classic" in-page). https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html
- Amazon Bedrock Agents Classic maintenance mode — the critical page: confirms no-new-customers as of July 30, 2026, existing-customer impact, and the full migration FAQ/procedure to AgentCore. https://docs.aws.amazon.com/bedrock/latest/userguide/agents-classic-maintenance-mode.html
- Use action groups to define actions for your agent to perform. https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-create.html
- How Amazon Bedrock Agents works (build-time components, runtime orchestration loop, ReAct default). https://docs.aws.amazon.com/bedrock/latest/userguide/agents-how.html
- Return control to the agent developer (return of control mechanics and worked example). https://docs.aws.amazon.com/bedrock/latest/userguide/agents-returncontrol.html
- Tutorial: Building a simple Amazon Bedrock agent (console + Lambda + boto3 walkthrough — good exercise 2 template). https://docs.aws.amazon.com/bedrock/latest/userguide/agent-tutorial.html
- What is Amazon Bedrock AgentCore? (Runtime, Gateway, Memory, Identity, Code Interpreter, Browser, Observability, Evaluations, Optimization, Policy, Registry). https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html

## Official Docs — Bedrock Guardrails

- Detect and filter harmful content using Amazon Bedrock Guardrails (overview of all policy types, including Automated Reasoning checks). https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html
- Block denied topics to help remove harmful content (exact config fields, best practices, 200 vs 1,000 character limits by tier). https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-denied-topics.html
- Remove a specific list of words and phrases with word filters (profanity filter + custom word list, up to 10,000 items). https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-word-filters.html
- Remove PII from conversations by using sensitive information filters (full entity type list by category, Block vs Mask, tool-use coverage gap). https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-sensitive-filters.html
- Use contextual grounding check to filter hallucinations in responses (grounding vs relevance scoring, thresholds, worked examples). https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html
- Safeguard tiers for guardrails policies (Classic vs Standard tier — new in 2026, cross-Region inference requirement, migration guidance). https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-tiers.html
- Automated Reasoning checks (mentioned and described in the Guardrails overview page above; the dedicated sub-page for this feature was not URL-verified this session — WebFetch could not extract its content, likely a rendering issue, not a broken link. Treat the tutorial.md description of this feature as sourced from the overview page only).

## Official Docs — Bedrock Knowledge Bases

- Retrieve data and generate AI responses with Amazon Bedrock Knowledge Bases (Managed Knowledge Base vs. Customer-managed Knowledge Base split — this distinction is new/restructured in 2026). https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html
- Build a knowledge base with vector stores (customer-managed path, general flow). https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build.html
- Create a knowledge base by connecting to a data source (the exact "quick create" vector store list: Amazon OpenSearch Serverless, Amazon Aurora PostgreSQL Serverless, Amazon Neptune Analytics, Amazon S3 Vectors). https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-create.html
- Supported models and Regions for Amazon Bedrock knowledge bases (embedding models, parsing models, reranking support). https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-supported.html
- How content chunking works for knowledge bases (standard/fixed-size/default/no-chunking, hierarchical, semantic, multimodal chunking). https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking-parsing.html

## Official Docs — Bedrock Evaluation

- Evaluate the performance of Amazon Bedrock resources (top-level overview: programmatic/automatic, human-based, judge-model, and RAG evaluation job types, in one page). https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html
- Evaluate model performance using another LLM as a judge (generator vs. evaluator model, built-in metrics, supported judge-model list). https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation-judge.html
- Evaluate the performance of RAG sources using Amazon Bedrock evaluations (retrieve-only vs. retrieve-and-generate job types, supported models). https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation-kb.html
- `model-evaluation.html` and `model-evaluation-create.html` under this same docs root were located via search (titles: "Choose the best performing model using Amazon Bedrock evaluations" / a job-creation how-to) but WebFetch could not extract readable content from either on repeated attempts this session — **(not URL-verified this session; use `evaluation.html` above instead, which covers the same ground and was fully verified)**.

## AWS Blog Posts (Evaluation / RAG-eval GA Announcements)

- AWS News Blog, "New RAG evaluation and LLM-as-a-judge capabilities in Amazon Bedrock" — original preview announcement, metric list (helpfulness, correctness, faithfulness, harmfulness, answer refusal), pricing note (evaluation itself has no separate charge beyond model inference). https://aws.amazon.com/blogs/aws/new-rag-evaluation-and-llm-as-a-judge-capabilities-in-amazon-bedrock/
- AWS Machine Learning Blog, "Evaluate models or RAG systems using Amazon Bedrock Evaluations — now generally available" — GA feature set, citation precision/coverage metrics, "bring your own inference responses." https://aws.amazon.com/blogs/machine-learning/evaluate-models-or-rag-systems-using-amazon-bedrock-evaluations-now-generally-available/
- AWS What's New, "Amazon Bedrock now supports RAG Evaluation (generally available)" — confirms March 20, 2025 GA date. https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-rag-evaluation-generally-available/
- AWS Partner Network Blog, "Introducing the Agentic AI Technical Learning Plan in AWS Skill Builder" — confirms this is a real, current (Nov 2025) Skill Builder learning plan covering AgentCore, multi-agent orchestration, LangGraph/CrewAI integration. https://aws.amazon.com/blogs/apn/introducing-the-agentic-ai-technical-learning-plan-in-aws-skill-builder/

## Training

- AWS Skill Builder (skillbuilder.aws) — confirmed real in a prior session; the Agentic AI Technical Learning Plan referenced above lives here. The specific lab URL surfaced by search ("Lab - Explore Amazon Bedrock Agents integrated with Amazon Bedrock Knowledge Bases and Amazon Bedrock Guardrails") was not independently fetched this session — (not URL-verified this session; search directly on skillbuilder.aws instead of trusting a specific deep link, since lab URLs on that platform are known to be reorganized periodically).

## Agent Framework Comparison Background (for the "Bedrock vs. custom" section)

- LangGraph documentation overview (current location, per prior-session correction) — confirms LangGraph is described as "a low-level orchestration framework and runtime for building, managing, and deploying long-running, stateful agents." https://docs.langchain.com/oss/python/langgraph/overview
- Microsoft Agent Framework overview (Microsoft Learn) — confirms Agent Framework is "the direct successor" to both Semantic Kernel and AutoGen, "created by the same teams," and that both predecessor projects are positioned as superseded by it. Page's `ms.date` metadata shows 2026-07-29. https://learn.microsoft.com/en-us/agent-framework/overview/

## Notes on What to Prioritize

The single highest-value thing to get right in this topic for a 2026 interview is the **Agents Classic -> AgentCore transition** — naming it correctly, knowing roughly what moved where (action groups -> Gateway MCP tools, return of control -> inline function tools, knowledge base attachment -> Gateway-fronted retrieval), and being able to say when you'd still reference "Agents Classic" concepts (they still describe how a huge number of production systems work today) versus when you'd recommend AgentCore for new work. Second priority: know the exact vocabulary distinction between denied topics (thematic, contextual) versus word filters (exact string match) versus sensitive-information filters (entity-type PII detection) — interviewers use "how would you block mentions of a competitor" as a trick question specifically to see if a candidate reaches for the wrong one. Third priority: be able to describe retrieve-only vs. retrieve-and-generate RAG evaluation and name faithfulness as the hallucination-detection metric specifically, since "how do you know if your RAG system hallucinates" is asked in almost every applied-AI interview regardless of platform.
