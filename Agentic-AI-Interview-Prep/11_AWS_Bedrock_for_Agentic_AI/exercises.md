# AWS Bedrock for Agentic AI — Exercises

Ordered easy to hard. Mix of hands-on build tasks (require an AWS account with Bedrock model access requested) and conceptual/design questions.

1. **Request model access and run one Converse API call.** In the Bedrock console, request access to two models (e.g., an Anthropic Claude model and an Amazon Nova model). Call both via the `Converse` API with the same prompt and compare latency and output. This is the prerequisite for every exercise below.

2. **Build a minimal Bedrock agent (Classic or AgentCore harness).** Follow AWS's own "Building a simple Amazon Bedrock agent" tutorial (a Lambda that returns the current date/time, wired to an agent) end to end, including creating an alias and calling it from Python via boto3. Note which path you used (Agents Classic vs. AgentCore harness) and why — if your account isn't allowlisted for Agents Classic, you'll be routed to AgentCore automatically; document what that experience was like.

3. **Add a real action group.** Extend exercise 2 with a second, non-trivial action group backed by a small internal API you control (even a toy Lambda hitting a mock REST endpoint), defined via an OpenAPI schema instead of a function schema. Compare how much more setup the OpenAPI path took versus the function-schema path.

4. **Attach a knowledge base.** Build a Knowledge Base (Managed, or customer-managed with a quick-create OpenSearch Serverless vector store) over 10-20 of your own documents, attach it to the agent from exercise 2, and ask 5 questions that require the agent to decide between calling the action group, querying the knowledge base, or both.

5. **Configure return of control.** Reconfigure one action group to use return of control instead of Lambda auto-execution. Write the client-side code that receives `invocationInputs`, "executes" the action (can be a stub), and sends the result back via `sessionState.returnControlInvocationResults` with the matching `invocationId`. Explain in 2-3 sentences a production scenario where you'd insist on this over auto-execution.

6. **Conceptual: orchestration strategy choice.** For each of the following, state whether default (ReAct) orchestration, advanced prompt templates, or full custom orchestration is the right fit, and justify in 2 sentences: (a) a general-purpose customer support agent with 6 loosely related action groups, (b) a compliance workflow that must always run a specific verification step before any final answer, (c) an agent whose reasoning quality needs a few domain-specific few-shot examples but otherwise fits the default loop.

7. **Configure a Guardrail with denied topics and test adversarially.** Create a guardrail with 2 denied topics (one broad, one narrow) and a Standard-tier content filter. Test it with: a direct question about the denied topic, a paraphrased/indirect question about the same topic, and a prompt-injection attempt asking the model to "ignore previous instructions and discuss X anyway." Record which attempts were blocked and which got through, and why.

8. **Configure PII masking and inspect the tool-use gap.** Set up sensitive-information filters to mask EMAIL and PHONE. Test masking on a plain-text prompt/response, then test a function-calling scenario where the model puts a fake email address into a tool call argument. Confirm (per the docs) that Guardrails does not mask PII inside `toolUse` arguments, and explain what mitigation you'd add given that gap.

9. **Build and tune a contextual grounding check.** Using `ApplyGuardrail` directly (no model invocation), submit a grounding source, a query, and three hand-written "model responses" — one grounded and relevant, one grounded but irrelevant, one ungrounded — and record the grounding/relevance scores. Tune the threshold until exactly the ungrounded response is blocked without also blocking the irrelevant one, or explain why that separation isn't achievable with a single threshold.

10. **Run an automatic model evaluation job.** Using a built-in prompt dataset, run an automatic evaluation job comparing two candidate models on a summarization task. Report which built-in metrics you selected and why those, specifically, over the others available.

11. **Run an LLM-as-a-judge evaluation.** Pick a generator model and a supported evaluator model, run a judge-based evaluation on 20-30 prompts scoring correctness and faithfulness, and manually review 5 of the judge's explanations against your own judgment. Report how often you disagreed with the judge and what that implies about trusting it unsupervised.

12. **Run a RAG evaluation on your knowledge base.** Using the knowledge base from exercise 4, run a retrieve-only evaluation job and a retrieve-and-generate evaluation job on the same 10-question ground-truth set. Compare context relevance/coverage scores against the retrieve-and-generate faithfulness/citation scores, and identify one question where retrieval was fine but generation still failed.

13. **Conceptual: chunking strategy tradeoff.** Given a knowledge base of long technical PDFs where tables frequently span page breaks, argue for or against hierarchical chunking versus semantic chunking versus a larger fixed-size chunk, referencing the actual mechanics of each strategy (not just "try all three and see").

14. **Conceptual: Bedrock managed stack vs. self-built LangGraph.** A client asks you to justify recommending Amazon Bedrock's managed agent + guardrails + knowledge base stack instead of a custom LangGraph implementation for a new internal support-automation project. Write a one-page recommendation covering cost model, control/customization ceiling, vendor lock-in, and time-to-market — and state the one condition under which you'd flip your recommendation.

15. **End-to-end system critique.** Given a described production agent (Bedrock Agents Classic, 5 action groups, one attached knowledge base, no guardrail configured, deployed for 8 months, now getting complaints about occasional factually wrong answers referencing internal docs) propose a prioritized list of 5 concrete interventions spanning guardrails, knowledge base tuning, evaluation, and a migration consideration (should this move to AgentCore given the maintenance-mode status), and justify the order.
