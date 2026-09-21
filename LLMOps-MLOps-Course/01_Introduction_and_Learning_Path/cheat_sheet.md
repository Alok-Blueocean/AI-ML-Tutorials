# Module 01 — Cheat Sheet

One-page reference for MLOps/LLMOps orientation. Pin this next to your desk for the first few weeks of the course.

---

## The One-Sentence Definitions

- **MLOps** = engineering practices for taking a trained model to a reliably operating, monitored, continuously-improvable production system. DevOps + (data/model versioning, statistical eval gates, drift monitoring, Continuous Training).
- **LLMOps** = MLOps + (prompt versioning, vector/retrieval management, token-cost tracking, LLM-as-judge evaluation, hallucination/agent-failure monitoring). **Extension, not a rival discipline.**
- **Golden rule:** if any part of your system calls an LLM — even one prompt-in/response-out call, zero fine-tuning — you already have an LLMOps surface.

---

## DevOps vs. MLOps vs. LLMOps — Condensed Comparison

| Axis | DevOps | MLOps | LLMOps |
|---|---|---|---|
| Artifact | Code / container | Code + model weights + data | Code + prompts + (index/weights) |
| Changes unprompted? | No | Data drift | Data drift + provider model updates + doc staleness |
| "Testing" | Pass/fail unit/E2E | + offline eval vs. held-out data | + LLM-as-judge / rubric eval |
| Versioned | Source code | + datasets + models | + prompts + vector index snapshots |
| Monitored | Latency/error/throughput | + prediction drift, accuracy proxy | + token cost, TTFT/P95, hallucination rate, retrieval quality |
| Rollback unit | Container/commit | Model registry version | Prompt version and/or model/index version |
| Retrain/re-tune trigger | N/A | Continuous Training (CT) | Prompt iteration / index refresh / fine-tune refresh |
| Novel failure | Regression bug | Silent accuracy degradation | **Hallucination** |

---

## Nested-Rings Mental Model

```
LLMOps ⊃ MLOps ⊃ DevOps      (each ring assumes + reuses everything inside it)
```

---

## Decision Rule — Do I Need This Machinery?

```
Model/generation/decision involved?         NO  → plain DevOps, stop here.
      │ YES
Calls an LLM (even zero fine-tune)?         NO  → classical MLOps: versioning,
      │ YES                                       registry, eval gates, drift monitor,
LLMOps: versioned prompts + cost/latency          CI/CD. Retrained on schedule? → add CT
tracking + eval beyond exact-match + guardrails   (orchestration, M20). Else skip.
      │
Retrieves external docs (RAG)?    NO → skip vector DB / M15-16
      │ YES → add vector DB + chunking/retrieval design
Decides tools/actions at runtime (agentic)?  NO → single-call pattern, skip M17/18 tracing
      │ YES → tool-calling framework (M17) + mandatory trace debugging (M18)

ALWAYS regardless of path (cheap early, expensive late):
  secrets mgmt · structured logging w/ trace IDs · a minimal held-out eval set (20-50 examples)
```

---

## Sculley et al. (2015) — Why This Field Exists

- **CACE**: Changing Anything Changes Everything — entanglement from learned (not hand-coded) relationships.
- Unit tests verify code does what code says; they **cannot** verify a model is statistically correct against real data.
- "ML Code" is a small box; the surrounding infra (data collection/verification, serving, monitoring, config, resource mgmt) is the large ring.

---

## The Governance Gap (2026 term)

LLM **deployment** tooling (serving/gateways/scaling) = mature. LLM **lineage/governance** tooling (which prompt version + which retrieved docs + which index snapshot produced output X) = roughly where classical-MLOps lineage was in **2018**. Fix: tag every trace span with `prompt_version`, `retrieved_doc_ids`/`index_snapshot_id`, `model_version`.

---

## Reference Architecture — 8 Bands (memorize the order)

```
Source of Truth (git/DVC/IaC) → CI Pipeline (M02/M25) → Registry Layer (M03/M13)
  → CD/Orchestration (M06/M20) → Serving Layer (M21) → Application Layer:
  RAG & Agents (M15-18) → Gateway/API (M08/M23/M24) → END USER
       ▲                                                    │
       └──── Observability + Feedback/Drift Loop (M14/M18/M19) ◀────┘
                     (this loop = Continuous Training/Improvement)
```

---

## 26-Module Roadmap — Block Order (and why)

| Block | Modules | One-line rationale |
|---|---|---|
| Release engineering | 02–06 | Non-negotiable foundation — can't safely ship anything without it |
| Prompting | 07 | What you're shipping, once you can ship safely |
| Cost/deployment economics | 08 | Know what a request costs before evaluating at scale |
| Evaluation methodology | 09–12 | Hardest, most neglected: how do you know a change is good? |
| Experiment tracking | 13 | Formalizes tracking into a system of record |
| Observability | 14 | See what's happening in prod, before flagship architectures |
| RAG | 15–16 | Flagship architecture #1 |
| Agents | 17–18 | Flagship architecture #2, harder — often *uses* RAG as a tool |
| Data layer / drift | 19–22 | Classical MLOps revisited with full LLMOps vocabulary |
| Serving at scale | 20–22 (21 core) | Highest-leverage cost/latency infra investment |
| Security/cost/governance | 23–24 | Cross-cutting, addressed last because full picture is needed |
| CI/CD integration | 25 | Hands-on payoff tying M02–M04 together |
| Capstone | 26 | Integrate everything into one running system |

**Teaching order ≠ build order.** Security/secrets/logging belong in your *real* projects from day one (M04 onward) even though the course teaches full depth at M23.

---

## Self-Check Before Moving to Module 02

```
[ ] Can state MLOps/LLMOps one-liners without notes
[ ] Can name 3 things MLOps adds over DevOps, and 3 things LLMOps adds over MLOps
[ ] Completed skills self-assessment; gaps have a plan
[ ] Can explain WHY the 26 modules are sequenced this way, not just list them
[ ] Have a realistic personal pacing plan (~130-170 hrs total, 6-10 hrs/wk ⇒ ~16-18 wks)
```
