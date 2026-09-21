# Capstone: Production LLMOps Platform

## What This Module Is About

This capstone is not a new topic — it's the "put it all together" project for the course. Over the previous modules you learned individual pieces of MLOps/LLMOps: data pipelines, experiment tracking, model packaging, deployment, prompt management, evaluation, monitoring, and governance. In real jobs, nobody deploys these pieces in isolation. This module asks you to design (and ideally build a small version of) a single, coherent platform that takes an LLM-powered application from an idea to something running safely in production.

Think of it like a capstone project in any engineering course: the value isn't in learning something brand new, it's in proving you can combine everything you already know into one working system, and explain the tradeoffs you made along the way.

## Why It Matters

Companies rarely hire someone to be "the prompt person" or "the vector database person." They need engineers who understand how the whole lifecycle connects — because a weak link anywhere (no versioning, no monitoring, no rollback plan) can take down the entire product, even if every individual component was built well. Interviewers and hiring managers also love capstone projects because they reveal how you think about a system end-to-end, not just whether you memorized a tool's API.

## The Main Concepts, in Plain Terms

A production LLMOps platform is usually described as a pipeline with feedback loops. Here are the pieces, in the order data and requests typically flow:

1. **Data & Knowledge Layer** — Where your source documents, embeddings, and any fine-tuning datasets live. Includes versioning so you know exactly what data produced which model or index.

2. **Model & Prompt Layer** — The LLM itself (hosted API or self-hosted), plus your prompt templates, system instructions, and any fine-tuned adapters. Both prompts and models should be version-controlled like code.

3. **Orchestration Layer** — The glue code that chains steps together: retrieval (RAG), tool calls, multi-step agents, or simple single-shot calls. This is where business logic lives.

4. **Serving & Infrastructure Layer** — How the application is exposed to users: APIs, containers, autoscaling, caching, and rate limiting. This is standard MLOps/DevOps territory applied to LLM workloads.

5. **Evaluation Layer** — Automated tests that check quality before and after deployment: golden datasets, LLM-as-judge scoring, regression tests. This answers "did we make it better or worse?"

6. **Observability Layer** — Logging, tracing, and dashboards that show latency, cost per request, token usage, and error rates in real time. This answers "is it healthy right now?"

7. **Safety & Governance Layer** — Guardrails (content filters, PII redaction), access controls, audit logs, and human review processes for high-risk outputs. This answers "can we trust it, and can we prove that to auditors?"

8. **Feedback Loop** — User feedback, flagged outputs, and production logs feed back into the data layer to improve future versions. This closes the loop and is what separates a one-off demo from a real "ops" practice.

A simple way to remember this: **Data → Model/Prompt → Orchestration → Serving → Eval → Observability → Safety → Feedback**, all wrapped in version control and CI/CD.

## A Simple Example: The Skeleton of a Capstone Architecture

You don't need exotic infrastructure to demonstrate the idea. A minimal but complete example might be expressed as a config file describing the pipeline stages:

```yaml
# capstone-platform.yaml
app: customer-support-assistant

data:
  source: docs/knowledge_base/
  embedding_model: text-embedding-3-small
  vector_store: chroma

model:
  provider: anthropic
  name: claude-sonnet
  prompt_template: prompts/support_v3.txt   # version-controlled

orchestration:
  type: rag
  retriever_top_k: 5
  fallback: "escalate_to_human"

serving:
  api: fastapi
  cache: redis
  rate_limit: 60/min

evaluation:
  golden_set: eval/golden_qa.jsonl
  metrics: [faithfulness, relevance, latency]
  gate: "block deploy if faithfulness < 0.85"

observability:
  tracing: enabled
  dashboards: [cost_per_query, error_rate, p95_latency]

safety:
  pii_redaction: enabled
  content_filter: enabled
  human_review: "low_confidence_responses"
```

This single file communicates the whole architecture at a glance — which is exactly what you want your capstone deliverable to do: a clear map of every stage, even if some stages are stubbed out or simplified for the project.

## Key Takeaways / Best Practices

- **Think end-to-end, not component-by-component.** The goal of the capstone is to show the connections between stages, not to showcase the fanciest model.
- **Version everything.** Data, prompts, models, and configs should all be tracked so any production behavior can be traced back to an exact cause.
- **Automate the quality gate.** Deployments should be blocked automatically if evaluation metrics fall below a threshold — don't rely on someone remembering to check manually.
- **Observability is not optional.** If you can't see latency, cost, and errors in production, you can't operate the system responsibly.
- **Build in a human fallback.** Every LLM system should have a defined path for when the model is uncertain or wrong (escalation, human review, or a safe default answer).
- **Keep the feedback loop closed.** Production logs and user feedback should have a defined path back into your evaluation and training data, not just sit in a log file.
- **Document tradeoffs, not just architecture.** Be ready to explain why you chose a given vector store, caching strategy, or evaluation metric — that reasoning is often what's actually being graded or interviewed for.
- **Start small, stay complete.** A capstone with a tiny dataset and a simplified model, but all eight layers represented, is more valuable than a polished single layer with the rest missing.
