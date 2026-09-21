# AWS Bedrock for Agentic AI — Condensed Study Notes

**Session note (2026-09-18):** Amazon Bedrock's agent stack changed structurally this year. Amazon Bedrock Agents was renamed **Amazon Bedrock Agents Classic** and, as of **July 30, 2026**, is no longer open to new customers — existing users with prior usage are allowlisted and unaffected, but new `CreateAgent`/`InvokeInlineAgent` calls from accounts with no prior Bedrock Agents history now return `AccessDeniedException`. AWS's recommended path for new agent development is **Amazon Bedrock AgentCore**. This file covers Agents Classic concepts (action groups, orchestration, return of control) because they are still exam/interview-relevant and still power every existing production deployment, but treats AgentCore as the current answer to "how would you build this today." Say this out loud in an interview — it signals you're tracking the platform, not reciting a 2024 tutorial.

## What Amazon Bedrock Actually Is

- Bedrock is AWS's managed layer for foundation models: model access (Anthropic, Amazon Nova, Meta, Mistral, OpenAI via the Responses API, etc.), plus a stack of managed services built on top — Agents/AgentCore, Guardrails, Knowledge Bases, and Evaluations — so you don't stand up your own vector DB, orchestration loop, and safety layer from scratch.
- The pitch for "agentic AI" specifically: instead of one prompt-in/answer-out call, Bedrock gives you a way to let a model decide which APIs to call, which documents to retrieve, and when it's done, while AWS manages the plumbing (auth, scaling, logging, encryption).
- Real-world example: a consulting team pitching a claims-processing assistant chooses Bedrock over rolling their own agent loop because the client's compliance team already trusts AWS's SOC2/HIPAA posture, and procurement is faster when the vendor is already an approved AWS account rather than a new SaaS contract for an orchestration framework.

## Bedrock Agents Classic — Core Model

- An agent is built from: a foundation model, natural-language **instructions**, at least one **action group** and/or **knowledge base**, and four editable **prompt templates** (pre-processing, orchestration, knowledge-base response generation, post-processing).
- Runtime is driven by `InvokeAgent` and runs a loop: pre-process the input, orchestrate (reason → act → observe, repeatedly), then post-process the final answer. AWS calls each intermediate output an **observation** and each reasoning step a **rationale** — both are visible if you turn on **trace**.
- Real-world example: a travel-assistant agent gets "What should I do today?", reasons that it first needs weather before suggesting activities, calls a `getWeather` action, observes "rainy," and only then calls `suggestActivities` — each of those steps shows up as a separate rationale/observation pair in the trace, which is exactly what you'd screenshot for a client demo of "the agent's reasoning."

## Action Groups

- An action group defines what the agent can *do*. You define it two ways: an **OpenAPI schema** (agent calls real REST-shaped APIs) or a **function schema** (simpler name/parameter list). Optionally attach a Lambda function that actually executes the call; the agent only ever sends parameters to Lambda, it never runs your code directly.
- Real-world example: an insurance company defines a `ClaimsAPI` action group with `getClaimStatus` and `fileClaim` functions, backed by a Lambda that calls their internal claims microservice. The agent decides *when* to call `getClaimStatus` based on the conversation; the Lambda is the only thing with real network access to the claims system, so a hallucinated or malicious tool call still can't reach anything the Lambda doesn't expose.

```json
// Minimal function-schema action group sketch
{
  "actionGroupName": "ClaimsAPI",
  "functionSchema": {
    "functions": [
      {
        "name": "getClaimStatus",
        "description": "Look up the status of an insurance claim by claim ID",
        "parameters": {
          "claimId": { "type": "string", "required": true }
        }
      }
    ]
  },
  "actionGroupExecutor": { "lambda": "arn:aws:lambda:us-east-1:123456789012:function:claims-status" }
}
```

## Orchestration Strategies

- **Default orchestration** uses a **ReAct** (Reason + Act) style loop baked into Bedrock's built-in orchestration prompt template: the model alternates between reasoning about what to do next and taking an action (call a tool, query a knowledge base, or answer), instrumented via the four editable prompt templates.
- **Advanced prompts** let you edit those base templates per stage (add few-shot examples, tighten instructions, insert a Lambda-based parser on a step's output) without leaving the managed loop.
- **Custom orchestration** replaces the built-in loop entirely with your own Lambda function that implements bespoke control flow — used when ReAct's default reasoning pattern doesn't fit (e.g., you need a fixed multi-step pipeline instead of free-form reasoning).
- Real-world example: a legal-research agent has a strict compliance requirement ("always run a citation-check step before returning any answer that quotes case law"). The default ReAct loop can't guarantee that ordering deterministically, so the team implements custom orchestration to hard-code the citation-check step, while still letting the model reason freely within the retrieval and drafting steps.

## Return of Control

- By default, an action group's Lambda executes the call and returns results to the agent automatically. **Return of control** (`customControl: RETURN_CONTROL` on the action group) instead has Bedrock hand the elicited parameters back to *your* application in the `InvokeAgent` response (`invocationInputs` + `invocationId`), so your own code decides how to execute it, then sends the result back via `sessionState.returnControlInvocationResults` using the same `invocationId`.
- This is the escape hatch for anything Bedrock shouldn't execute directly: client-side actions (open a UI modal, ask for a credit-card confirmation), calls that need credentials Bedrock's Lambda role shouldn't hold, or actions that need a human-in-the-loop approval step before firing.
- Real-world example: a banking agent's `wireTransfer` action group is configured with return of control. When the agent decides to call it, the parameters (amount, recipient) come back to the client app instead of auto-executing; the app shows the user a confirmation dialog, and only sends the result back to the agent after the user explicitly approves the transfer — return of control is what makes "the agent proposes, a human disposes" possible without weakening the rest of the automation.

```
InvokeAgent response -> returnControl.invocationInputs[0].functionInvocationInput
    { actionGroup: "WireTransfer", function: "sendWire", parameters: [...] }
# your app executes/gets human approval, then:
InvokeAgent request -> sessionState.returnControlInvocationResults[0].functionResult
    { actionGroup: "WireTransfer", function: "sendWire", responseBody: {...} }
```

## Amazon Bedrock AgentCore — The Current Direction

- AgentCore is a modular, **framework-agnostic** agent platform: it works with LangGraph, CrewAI, LlamaIndex, Strands Agents, OpenAI Agents SDK, or custom code, and with any foundation model (not just Bedrock-hosted ones). Its core services: **Runtime** (serverless execution with session isolation), **Gateway** (turns your APIs/Lambdas into MCP-compatible tools), **Memory** (short- and long-term, shareable across agents), **Identity** (auth/credential brokering to existing IdPs), **Code Interpreter** and **Browser** (sandboxed tool execution), **Observability** (OpenTelemetry-native tracing), plus newer additions: **Evaluations**, **Optimization**, **Policy** (Cedar-based rule enforcement on tool calls), and **Registry**.
- AgentCore also ships a **managed harness**: a config-based, declarative starting point (model + tools + instructions) that's the closest analog to how Bedrock Agents Classic felt to configure, while "code-defined agents on AgentCore" is the option for teams that want full control of the orchestration loop (multi-agent supervisors, custom prompt overrides) but still want AgentCore's managed compute/memory/identity/observability underneath.
- Real-world example: a team migrating an existing Bedrock Agents Classic claims-bot uses the AgentCore CLI's import path — action groups become Gateway-fronted MCP tools, the knowledge base is fronted by Gateway instead of attached directly to the agent config, and return-of-control becomes an "inline function tool" that pauses the harness and hands `tool_use` back to client code — functionally the same escape hatch, different vocabulary.

## Bedrock Guardrails — Policy Types

Guardrails are a standalone resource (guardrail ID + version) you can attach to a model invocation, to an agent, or call directly via `ApplyGuardrail` without invoking any model. Current policy set:

- **Content filters** — block Hate, Insults, Sexual, Violence, Misconduct, and Prompt Attack categories, each with a configurable strength. **Standard tier** additionally detects harmful content hidden inside code (comments, variable/function names, string literals); **Classic tier** does not.
- **Denied topics** — a natural-language topic (name + up to a 200-character definition on Classic tier, 1,000 characters on Standard tier, plus up to 5 sample phrases) that the guardrail blocks contextually. AWS's own docs are explicit: don't use denied topics to catch individual words, names, or competitor mentions — that's what word filters and sensitive-information filters are for; topic filtering evaluates meaning, not string matches.
- **Word filters** — exact-match blocking: a managed, continually-updated **profanity filter**, plus a **custom word filter** (up to 10,000 entries, uploadable via console as `.txt`/`.csv`/S3 object).
- **Sensitive information filters (PII)** — probabilistic, context-aware detection of built-in entity types (NAME, EMAIL, ADDRESS, PHONE, US_SOCIAL_SECURITY_NUMBER, CREDIT_DEBIT_CARD_NUMBER, AWS_ACCESS_KEY, UK_NATIONAL_INSURANCE_NUMBER, and more, grouped as General/Finance/IT/US/Canada/UK-specific) plus custom regex patterns. Each entity supports **Block** or **Mask** (`ANONYMIZE`, replaces the value with a placeholder like `{EMAIL}`) independently for input vs output.
- **Contextual grounding checks** — scores a model's response against a supplied grounding source and user query on two axes, **grounding** (is it factually supported by the source) and **relevance** (does it answer the query), each 0–0.99, with a configurable threshold per axis. Designed for RAG/summarization/QA, explicitly *not* for open-ended conversational chat.
- **Automated Reasoning checks** — validates a response against a set of formally defined logical rules (not a probabilistic model), aimed at catching hallucinations and unstated assumptions with mathematical rather than statistical confidence — useful where "probably grounded" isn't good enough (e.g., stating a specific policy rule correctly).
- **Safeguard tiers (new)** — content filters, prompt-attack detection, and denied topics can each run on **Classic tier** (English/French/Spanish, no cross-Region inference, established behavior) or **Standard tier** (broader language support, code-aware detection, cross-Region inference required, prompt-leakage detection, more accurate prompt-attack handling). This is a live migration decision, not just a config toggle — AWS recommends a phased rollout starting with non-critical workloads.

Real-world example: a bank's customer-service guardrail combines a **denied topics** entry ("Investment Advice," defined narrowly, not "any mention of stocks") to stop the bot from giving unlicensed investment guidance, a **sensitive information filter** set to Mask on NAME/ACCOUNT-adjacent regex patterns so call-summary logs don't retain raw PII, and a **contextual grounding check** with a 0.8 grounding threshold on its RAG-based FAQ answers so it can't invent a fee schedule that isn't in the actual policy document.

```json
// Guardrail policy sketch: denied topic + PII masking
{
  "topicPolicyConfig": {
    "topicsConfig": [{
      "name": "InvestmentAdvice",
      "definition": "Inquiries or recommendations about allocating funds to generate investment returns.",
      "type": "DENY",
      "inputAction": "BLOCK", "outputAction": "BLOCK"
    }],
    "tierConfig": { "tierName": "STANDARD" }
  },
  "sensitiveInformationPolicyConfig": {
    "piiEntitiesConfig": [
      { "type": "NAME", "action": "ANONYMIZE" },
      { "type": "US_SOCIAL_SECURITY_NUMBER", "action": "BLOCK" }
    ]
  }
}
```

## Bedrock Knowledge Bases — Managed RAG

- Two flavors as of 2026: **Managed Knowledge Base** (AWS-recommended default — AWS handles ingestion, indexing, storage auto-scaling, embedding, reranking, and reasoning; connectors for S3, SharePoint, Confluence, Google Drive, OneDrive, and Web Crawler; document-level ACL-based permission filtering at retrieval time; native AgentCore Gateway integration so any MCP-compatible agent can call it as a tool; **Agentic Retrieval** for multi-hop reasoning that decomposes a complex query into sub-queries and retrieves iteratively) vs. **Customer-managed Knowledge Base** (you choose and operate the vector store yourself — OpenSearch Serverless, Aurora PostgreSQL Serverless, Neptune Analytics, or Amazon S3 Vectors via "quick create," or bring your own supported vector store — full control, more ops burden, required if you need a data source type or vector store the managed path doesn't cover).
- **Chunking strategies**: default (~300 tokens, respects sentence boundaries), fixed-size (you set token count + overlap %), no chunking (whole document as one chunk — you lose page-number citation metadata), hierarchical (parent/child chunk sizes; retrieval matches on precise child chunks but returns the broader parent chunk for context — not recommended with S3 Vectors), and semantic chunking (splits on meaning shifts using an embedding-based sentence-similarity breakpoint, at extra cost since it invokes a model). Multimodal content (audio/video/images) is chunked at the embedding-model level instead, not by these text strategies.
- Real-world example: a pharma company builds a Knowledge Base over regulatory submission PDFs. Fixed-size chunking with a large chunk size kept splitting a single dosage table across two chunks (retrieval would grab only half the table); switching to hierarchical chunking — small child chunks for precise matching, larger parent chunks returned for context — fixed the "half a table" problem without ballooning every chunk's size.

```
# Fixed-size chunking config sketch
chunkingConfiguration:
  chunkingStrategy: FIXED_SIZE
  fixedSizeChunkingConfiguration:
    maxTokens: 300
    overlapPercentage: 20
```

## Bedrock Model Evaluation

- **Programmatic (automatic) evaluation jobs** — score a model on built-in metrics (or your own custom metric) using either a built-in prompt dataset or your own, with no humans in the loop. Fast, cheap, good for iterating quickly and for regression-testing between model versions.
- **Human-based evaluation jobs** — route your own dataset to a team of human workers (employees or subject-matter experts) who rate/compare responses; you must supply your own dataset (no built-in datasets for this path).
- **LLM-as-a-judge evaluation** — a second model (the **evaluator model**, distinct from the **generator model** being evaluated) scores each response and produces a natural-language explanation per score. Built-in metrics include correctness, completeness, faithfulness, helpfulness, coherence, relevance, plus responsible-AI metrics (harmfulness, stereotyping, refusal). You can also evaluate a non-Bedrock model entirely by supplying your own inference responses and skipping the generation step ("bring your own inference responses").
- Real-world example: before swapping the production summarization model, a team runs an LLM-as-a-judge evaluation job comparing the current model against a candidate on the same 200-prompt dataset, scoring correctness and faithfulness — and only ships the swap because the candidate's faithfulness score didn't regress, even though its correctness score was slightly higher (a faithfulness regression is the one that turns into a hallucination incident).

## RAG Evaluation (inside Bedrock Knowledge Bases / Evaluations)

- **Retrieve-only** jobs score just the retrieval stage — metrics like context relevance and coverage — useful for isolating "is my chunking/embedding/vector-store choice good" from "is my generation prompt good."
- **Retrieve-and-generate** jobs score the full pipeline end to end: retrieval plus the generated answer, using LLM-as-a-judge metrics — correctness, completeness, faithfulness (this is the hallucination-detection metric), plus responsible-AI metrics (harmfulness, answer refusal, stereotyping) and citation metrics (citation precision, citation coverage).
- Both job types accept a **ground-truth dataset** (expected retrieved text and/or expected response per query) and both support "bring your own inference responses," so you can evaluate a knowledge base or RAG system that isn't even hosted in Bedrock.
- Real-world example: a team debugging "why does our RAG chatbot sometimes cite the wrong policy version" runs a retrieve-only job first and finds context relevance is fine (the right chunks are being retrieved) but coverage is low (only part of the relevant policy is coming back) — pointing them at chunk size, not the generation prompt, as the actual fix.

## Bedrock's Managed Agentic Stack vs. a Self-Built LangGraph/Semantic Kernel Solution

*(This section is the author's own analysis/synthesis for interview framing — AWS does not publish a direct "Bedrock vs. LangGraph" comparison doc; the individual feature claims above are docs-sourced, but the tradeoff framing below is not a citation.)*

- **Lean toward Bedrock's managed path (AgentCore + Guardrails + Knowledge Bases)** when: the client is already AWS-committed and wants fast time-to-market; the team doesn't want to own an orchestration runtime, a vector DB, and a safety-filtering layer as three separate ops burdens; compliance/audit requirements are easier to satisfy against an AWS-attested service (SOC2, HIPAA) than a self-hosted stack; or the agent's tool set is mostly "call internal REST APIs and query one or two knowledge sources" — the boring 80% case managed tooling is built for.
- **Lean toward a self-built LangGraph/Microsoft Agent Framework solution** when: orchestration logic is genuinely custom (complex conditional branching, non-standard multi-agent supervisor patterns not yet well-supported by the managed harness), the team needs full model-provider portability (swap Anthropic/OpenAI/open-weight models freely, run on-prem or multi-cloud) without vendor lock-in to a specific platform's config format, latency budgets are tight enough that a thin hand-rolled loop beats any framework's overhead, or the org already has strong in-house MLOps/LLMOps engineering and building is genuinely cheaper than the accumulated per-call/per-session pricing of a managed platform at their volume.
- Cost shape differs, not just cost amount: Bedrock's managed services are consumption-priced with no separate orchestration fee beyond model inference (AgentCore specifically claims comparable-or-lower inference costs due to more token-efficient internal prompting than Agents Classic) — but every additional managed capability (Gateway, Memory, Evaluations) adds its own metered line item. A self-built LangGraph stack shifts cost from AWS's meter to engineering time and infrastructure you now own and must monitor, patch, and scale yourself.
- Vendor lock-in is real but overstated on both sides: an agent built purely on Bedrock Agents Classic's proprietary config is not portable; AgentCore is explicitly framework-agnostic (works with LangGraph itself, CrewAI, Strands) which materially reduces lock-in versus the old Agents Classic model — so "Bedrock vs. custom" is less binary than it used to be. A reasonable middle path many teams take: use LangGraph (or Microsoft Agent Framework) for orchestration logic you want to own and keep portable, and use Bedrock Guardrails + Knowledge Bases + AgentCore Gateway/Memory/Observability underneath it for the parts that are genuinely undifferentiated infrastructure — you don't have to pick one column of the table for the whole system.
- Note on framework staleness (worth naming in an interview to show currency): **AutoGen** has been in Microsoft's maintenance mode since October 2025, and **Semantic Kernel** followed it into maintenance mode when **Microsoft Agent Framework 1.0** shipped in April 2026, unifying both lineages into one production SDK. If a candidate is asked to compare "Bedrock vs. Semantic Kernel" today, the accurate answer names Microsoft Agent Framework as the actively developed target, not Semantic Kernel alone.

## Quick Gotchas Worth Naming in an Interview

- "Bedrock Agents" as a term is ambiguous in 2026 — clarify whether you mean **Agents Classic** (the original, now maintenance-mode, no-new-customers service) or **AgentCore** (the current platform). Using them interchangeably in an interview signals stale knowledge.
- Denied topics are for *themes*, not entities or exact strings — trying to block "mentions of Competitor X" with a denied topic is the wrong tool; that's a word filter (exact match) or, if it's closer to an entity class, a custom regex in the sensitive-information filter.
- Contextual grounding checks require the model response as input, so they only ever run on output, never on the prompt alone — and they explicitly don't support open-ended conversational/chatbot use cases, only summarization, paraphrasing, and QA.
- PII masking in Guardrails does not touch tool-call arguments, tool results, or tool-spec descriptions in function-calling workloads, and it never touches raw CloudWatch invocation logs — a guardrail can mask PII in the visible response while the unmasked value still sits in your logs unless you separately apply CloudWatch log data protection.
- Safeguard tiers are not purely additive — moving denied topics or content filters to Standard tier requires cross-Region inference, which means your guardrail's inference can now be routed to a different AWS Region than the one you configured it in; that's a data-residency conversation to have before flipping the switch, not after.
- Hierarchical chunking can return fewer results than requested (child chunks get replaced by parent chunks, and duplicates collapse), and it's explicitly discouraged with S3 Vectors as the vector store — a chunking strategy choice is coupled to your vector store choice, not independent of it.
- Bedrock's "LLM-as-a-judge" evaluator model must be a supported judge model (a specific list, not "any model you have access to") — don't assume you can wire an arbitrary fine-tuned model in as the evaluator without checking the supported list first.
- "Return of control" and AgentCore's "inline function tools" solve the same problem (pause for external/human execution) but are configured completely differently — don't describe them as literally the same API surface when explaining a migration.
