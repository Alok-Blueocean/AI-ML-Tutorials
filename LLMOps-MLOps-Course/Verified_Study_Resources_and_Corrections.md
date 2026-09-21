# Verified Study Resources & Corrections to Earlier Answers

*This file exists because two earlier resource recommendations in this conversation were given before search access was exhausted, and were flagged as "high-confidence from training knowledge, not freshly verified." This is the follow-up verification pass. Verified via direct fetch to known URLs — the session's open WebSearch capacity is still exhausted, so this is confirmation of specific resources, not a fresh open search (Group A below is still genuinely unresolved for that reason).*

---

## Corrections to earlier answers in this conversation (read this first)

1. **AutoGen is no longer an actively-recommended framework.** I listed "LangGraph / CrewAI / AutoGen" as peer options earlier. Microsoft's AutoGen repo now states it **will not receive new features and is community-managed going forward** — Microsoft has moved on to a successor called **Microsoft Agent Framework**. If you're picking a framework today, treat AutoGen as a maintenance-mode option to know about, not a first choice.
2. **τ-bench (tau-bench) has been superseded by τ²-bench (tau-squared-bench).** Same origin (Sierra), newer repo. Cite τ²-bench going forward.
3. **LangGraph does not own "agent evaluation" itself** — its docs point to a separate product, **LangSmith**, for evaluation. If you want LangGraph *and* a maintained eval framework, you're pairing it with LangSmith (or DeepEval/Ragas as used elsewhere in this course), not expecting LangGraph to do both.
4. **NeMo Guardrails' GitHub org has been renamed** from the plain NVIDIA org to `NVIDIA-NeMo` — same project, new org path.
5. **AgentBench has a newer variant**, "AgentBench FC" (function-calling), added around October 2025 — worth citing instead of/alongside the original.
6. LangGraph's docs URL has moved (old `langchain-ai.github.io/langgraph/` now redirects) — updated link below.

Everything else recommended earlier (Terraform: Up & Running, Designing Machine Learning Systems, Langfuse docs, Evidently AI docs, AWS Skill Builder, AWS Bedrock docs, ReAct, Toolformer, AgentBench, Guardrails AI, CrewAI) checked out as real and current.

---

## Group A — "one unified book/course" question: still unresolved

I could not run an open web search this round either (quota still exhausted for search specifically, though direct fetches to known URLs worked). I have no new evidence for or against a single book covering the full AWS-native LLMOps + agentic stack. Treat Section 0 of `60_Day_AWS_Agentic_LLMOps_Roadmap.md` (no single book exists) as still the best available answer — genuinely re-check this once search quota is confirmed available again, rather than assuming my earlier "no" is fully settled.

---

## Group B — AWS/infra resources (verified)

| Resource | Status | Link / note |
|---|---|---|
| AWS Skill Builder | ✅ Real | `skillbuilder.aws` |
| AWS Bedrock official docs | ✅ Real | `docs.aws.amazon.com/bedrock/latest/userguide/` |
| Terraform AWS provider registry docs | Correct URL, page is JS-rendered so couldn't fully re-fetch content | `registry.terraform.io/providers/hashicorp/aws/latest/docs` |
| Langfuse docs | ✅ Real | `langfuse.com/docs` — covers tracing/observability, prompt management, and evaluation (LLM-as-judge, code evaluators, human feedback) |
| Evidently AI docs | ✅ Real | `docs.evidentlyai.com` — open-source, covers both traditional ML drift/data-quality and LLM eval |
| LangGraph docs | ✅ Real, **URL moved** | now `docs.langchain.com/oss/python/langgraph/overview`; human-in-the-loop guide at `/oss/python/langgraph/interrupts`. **Evaluation is not native to LangGraph** — see correction #3 above |
| *Terraform: Up & Running* (Brikman) | ✅ Real, confirmed | 3rd edition (Sept 2022), adds secrets-management + multi-region chapters; code samples at `github.com/brikis98/terraform-up-and-running-code` |
| *Prometheus: Up & Running* (Brazil) | Uncertain this round | Three fetch attempts (O'Reilly, Robust Perception, Amazon) all failed to load (403/404/500) — not debunked, just couldn't re-confirm live; still the standard reference from general knowledge |
| *Designing Machine Learning Systems* (Chip Huyen) | ✅ Real, confirmed | O'Reilly, 2022, via the author's own site |

---

## Group C — Agent-evaluation / tool-use study resources (verified)

| Resource | Status | Link / note |
|---|---|---|
| ReAct paper (Yao et al.) | ✅ Real | arXiv:2210.03629 — "ReAct: Synergizing Reasoning and Acting in Language Models" |
| Toolformer paper | ✅ Real | arXiv:2302.04761, Schick et al. |
| AgentBench | ✅ Real, **updated** | `github.com/THUDM/AgentBench` (Tsinghua/THUDM), ICLR'24 paper; now has an **AgentBench FC** (function-calling) extension as of ~Oct 2025 |
| τ-bench / τ²-bench (Sierra) | ✅ Real, **superseded** | Original: `github.com/sierra-research/tau-bench` (Yao/Shinn/Razavi/Narasimhan, 2024) — evaluates airline/retail tool-agent-user interactions. **Now evolved into τ²-bench (tau-squared-bench)** in a separate, newer repo — cite that one going forward |
| Guardrails AI | ✅ Real | `github.com/guardrails-ai/guardrails`, ~7.3k stars — input/output risk validation + structured-data generation |
| NeMo Guardrails | ✅ Real, **org renamed** | `github.com/NVIDIA-NeMo/Guardrails` (was under the plain NVIDIA org) — five rail types, Colang DSL |
| CrewAI | ✅ Real, actively maintained | `github.com/crewAIInc/crewAI`, ~56.7k stars |
| AutoGen | ✅ Real, but **maintenance mode** | `github.com/microsoft/autogen` — no new features planned, community-managed; Microsoft's actively-developed successor is **Microsoft Agent Framework** |

---

## Practical takeaway for your 60-day build

- If you were going to pick between LangGraph / CrewAI / AutoGen for the multi-agent project discussed earlier: **LangGraph or CrewAI**, not AutoGen — AutoGen is no longer where new investment is going.
- Pair LangGraph with **LangSmith** (not LangGraph itself) if you want built-in evaluation tooling from the same vendor, or stick with DeepEval/Ragas as already used elsewhere in this course for consistency.
- If you want an academic benchmark to model your own trajectory-evaluation harness on, look at **τ²-bench** or **AgentBench FC** specifically — both are the current, maintained versions, not the ones I mentioned by initial memory.
