# AWS Bedrock for Agentic AI — Scenario-Based Q&A

**Situation:** A client asks you to recommend either Amazon Bedrock's managed Agents/Guardrails/Knowledge Bases stack or a custom-built LangGraph solution for a new internal support-automation agent, and wants the tradeoffs in terms leadership can act on. What would you do and why?

Model answer: Frame it as four axes, not a single yes/no. Cost: Bedrock's managed services are consumption-priced per call/session with no separate orchestration license, but every managed capability (Gateway, Memory, Evaluations) is its own metered line item, while a custom LangGraph build shifts cost from AWS's meter to engineering time and infrastructure the client now owns and operates. Control: a custom build gives full control over orchestration logic, model-provider choice, and latency tuning; Bedrock's managed harness covers the common declarative case well but has real gaps for complex multi-agent supervisor patterns and custom prompt-stage overrides. Vendor lock-in: this is less binary than it used to be — Bedrock AgentCore is explicitly framework-agnostic (it runs LangGraph, CrewAI, and Strands agents underneath its managed infrastructure), so a hybrid is often the actual answer: use LangGraph for orchestration logic worth owning and keep it portable, while using Bedrock Guardrails and Knowledge Bases for the undifferentiated safety/retrieval layer. Time-to-market: managed wins almost always for a first version, since procurement/compliance is faster against an already-approved AWS account. Recommend the managed path by default for a first release with a clear migration story if custom orchestration needs grow, rather than presenting it as a permanent, irreversible choice.

---

**Situation:** Your team stood up a Bedrock Agent 8 months ago (before "Agents Classic" existed as a term) with 5 action groups and a knowledge base, and it's still running fine, but you've just learned the service is now called Bedrock Agents Classic and is in maintenance mode for new customers. Leadership asks if this is an emergency. What would you do and why?

Model answer: Confirm first, calmly, that it isn't an emergency for this specific system: existing agents keep working, all runtime and management APIs remain available to allowlisted accounts (accounts with prior usage), and there's no forced end-of-life date announced. The actual risk is opportunity cost, not breakage — Agents Classic's model catalog is frozen as of the maintenance-mode date, so new model releases will only be reachable through AgentCore, and no new orchestration features will land in Classic. Recommend a non-urgent migration evaluation: use AWS's own migration tooling (the AgentCore CLI import path, or the agent-toolkit skill that inspects the existing agent and produces a component-by-component migration plan) to estimate effort, and schedule the migration as a planned project timed with the next model upgrade the client wants, rather than an emergency response to a naming change.

---

**Situation:** Your Bedrock-based RAG assistant occasionally states a policy detail that isn't actually in the source documents, but only for a narrow set of edge-case questions. What would you do and why?

Model answer: Treat this as a grounding failure to diagnose with the platform's own tools before writing custom code. First, add a contextual grounding check to the guardrail with grounding and relevance thresholds, since this is exactly the failure mode it targets — a response that introduces information not present in the retrieved source. Second, run a retrieve-only RAG evaluation job to check whether the failing questions are a retrieval problem (right chunk never coming back — check context relevance/coverage) versus a generation problem (right chunk retrieved, but the model still added unsupported detail — check faithfulness on a retrieve-and-generate job). Only after isolating which stage is failing would I change chunking strategy (if retrieval) or tighten the generation prompt / lower the grounding threshold (if generation) — fixing the wrong stage first is the classic mistake here.

---

**Situation:** A stakeholder wants the agent to refuse to discuss a competitor's product by name, and separately wants it to never give investment advice. They ask you to implement both as "denied topics." What would you do and why?

Model answer: Push back on lumping these together, because Bedrock Guardrails documentation is explicit that denied topics are for themes evaluated contextually, not for entity/name matching — using a denied topic to catch "mentions of Competitor X" is unreliable and is the documented anti-pattern. Implement "no investment advice" as a genuine denied topic with a precise definition (not an instruction, not a negative definition) and a couple of sample phrases. Implement "never mention Competitor X by name" as a word filter (exact-match custom word list) instead, since that's a literal string-matching problem, not a thematic one. Explain to the stakeholder that using the wrong tool for the competitor-name case would likely both over-block (flagging unrelated content that happens to share context) and under-block (missing a paraphrase that names the competitor differently).

---

**Situation:** Legal asks whether your Bedrock Guardrail's PII masking guarantees that customer emails never end up in application logs. What would you do and why?

Model answer: Give the precise, not the optimistic, answer: PII masking only applies to the input prompt sent to the model and the response returned from the model — it does not apply to CloudWatch model-invocation logs (which always store the original unmodified request regardless of guardrail action) and it does not apply to PII the model puts inside tool-call arguments or results in function-calling workloads. So no, masking alone does not guarantee emails never reach logs. Recommend a layered answer: keep sensitive-information filters for the user-facing input/output path, separately apply CloudWatch log data protection (or scrub logs at the logging layer) for anything persisted, and audit whether any action group's Lambda receives or returns raw PII in tool arguments, since that's an explicitly documented gap in the guardrail's coverage.

---

**Situation:** You need to decide between Bedrock's default ReAct orchestration, advanced prompt templates, and full custom orchestration for a compliance-heavy workflow that must always run a mandatory verification step before returning any answer. What would you do and why?

Model answer: Default ReAct orchestration lets the model decide the order of operations based on reasoning, which means it can't guarantee a specific step always runs — that's a hard no for a compliance requirement. Advanced prompt templates can bias the model strongly toward running the verification step but still don't guarantee it deterministically, since it's still a probabilistic reasoning process underneath. Custom orchestration, where you replace the orchestration loop with your own Lambda-implemented control flow, is the only option that can hard-code "verification step always executes before final answer" as actual code rather than an instruction the model might skip under distribution shift. Recommend custom orchestration specifically for the compliance-critical path, while noting it's more engineering effort and loses some of the free-form reasoning flexibility elsewhere in the workflow.

---

**Situation:** Two candidate models score similarly on an automatic Bedrock model evaluation job for a summarization task, but the team can't agree on which to ship. What would you do and why?

Model answer: Automatic metrics alone often aren't discriminating enough for genuinely close calls, especially on an open-ended task like summarization. Escalate to an LLM-as-a-judge evaluation on the same prompt set with correctness, completeness, and faithfulness as the metrics, since faithfulness specifically catches subtle hallucination differences that automatic metrics can miss. If the judge scores are still close, pull a sample of 15-20 responses for a quick human-based evaluation pass rather than trusting the judge model unsupervised on a genuinely ambiguous case — LLM-as-a-judge is a fast proxy, not ground truth, and needs occasional human calibration exactly when the automated signal is inconclusive.

---

**Situation:** A knowledge base built over long technical PDFs is returning incomplete answers — the retrieved chunk cuts off mid-explanation right where the user's question needed the next paragraph. What would you do and why?

Model answer: This is very likely a chunking granularity problem, not a retrieval-ranking problem, so I'd check chunk size and boundaries before touching the embedding model or reranker. If chunks are fixed-size and small, a paragraph's continuation may simply be split into a separate, non-retrieved chunk — increasing overlap percentage or moving to hierarchical chunking (retrieve on small, precise child chunks, but return the broader parent chunk for full context) directly addresses this, since hierarchical chunking is designed for exactly the "precise match but need more context" tension. I'd validate the fix with a retrieve-only RAG evaluation job comparing context coverage before and after the chunking change, rather than eyeballing a handful of manually re-tested questions.

---

**Situation:** Your team wants to add a "wire transfer" action to a banking agent, and someone on the team suggests just letting the Lambda execute it automatically like the other action groups, to keep the implementation simple. What would you do and why?

Model answer: Push back specifically on this one action group, not the whole agent design. Auto-executing Lambda actions are fine for read-only or low-consequence actions (status lookups, FAQ retrieval), but an irreversible, high-consequence action like a wire transfer should use return of control instead: the agent proposes the parameters, but a human explicitly approves before the transfer executes, and the result is fed back to the agent afterward using the same invocation ID. Framing this correctly to the team: return of control isn't extra caution for its own sake, it's the mechanism that lets you keep the rest of the agent fully automated while drawing a hard line around the one action category where a hallucinated or misinterpreted parameter has real financial consequences.

---

**Situation:** Leadership asks you to quantify, before a wider rollout, whether the RAG-based support agent is "good enough," and specifically whether it's citing sources correctly.

Model answer: Set up a retrieve-and-generate RAG evaluation job on a held-out, ground-truth-labeled question set, and report the built-in metrics directly rather than an ad hoc sample review: correctness and completeness for answer quality, faithfulness specifically as the hallucination signal, and citation precision/citation coverage specifically for the "citing sources correctly" question leadership asked about. Pair this with a retrieve-only job on the same question set to separate "the knowledge base didn't have the right chunk" failures from "the model had the right chunk but cited or answered incorrectly anyway" failures, since those need different fixes. Recommend treating this evaluation set as a permanent regression suite re-run before every knowledge base or prompt change, not a one-time rollout gate.
