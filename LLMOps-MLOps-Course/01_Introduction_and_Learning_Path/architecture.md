# Module 01 — Architecture Deep Dive

This companion document goes deeper on the structural diagrams than the main tutorial: a full end-to-end reference architecture for a mature MLOps/LLMOps platform, a sequence diagram of what happens on a single production request, and a decision tree for a question every reader of this course will face repeatedly — "does this project need classical MLOps, LLMOps, both, or neither?"

Nothing here is invented as a "real company's actual system." These are reasoned, generic reference architectures of the kind commonly described in engineering-blog patterns across the industry, used to teach structure — not a claim about any specific company's internals.

---

## 1. Full Reference Architecture — A Combined MLOps + LLMOps Platform

This is the "big picture" diagram this entire course is teaching you, piece by piece, module by module. Every box below is annotated with the module(s) that teach it.

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                    SOURCE OF TRUTH                                          │
│  ┌───────────────┐   ┌────────────────┐   ┌─────────────────┐   ┌──────────────────────┐   │
│  │  Application   │   │  Prompt/Template│   │  Training/Eval   │   │  Infra-as-Code       │   │
│  │  source code   │   │  repository     │   │  datasets        │   │  (Docker/K8s manifests)│  │
│  │  (git)         │   │  (git, M03/M09) │   │  (DVC/LakeFS,M22)│   │  (M05/M06)            │   │
│  └───────┬───────┘   └───────┬────────┘   └────────┬─────────┘   └──────────┬───────────┘   │
└──────────┼───────────────────┼─────────────────────┼─────────────────────────┼───────────────┘
           │                   │                      │                         │
           ▼                   ▼                      ▼                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                              CI PIPELINE  (Module 02, Module 25)                           │
│  lint/unit tests → data schema validation → build image (M05) → run offline eval (M09-M11) │
│      → compare against release threshold (M12) → push to registry if it passes             │
└───────────────────────────────────────┬────────────────────────────────────────────────────┘
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                    REGISTRY LAYER (Module 03, Module 13)                                    │
│  ┌────────────────────┐   ┌─────────────────────┐   ┌──────────────────────────────────┐   │
│  │  Model registry     │   │  Prompt/template     │   │  Vector index / RAG corpus       │   │
│  │  (MLflow/W&B,        │   │  version registry    │   │  snapshot registry               │   │
│  │   staging→prod tags) │   │  (M03, M09)          │   │  (M15, M16)                      │   │
│  └──────────┬──────────┘   └──────────┬──────────┘   └────────────────┬─────────────────┘   │
└─────────────┼─────────────────────────┼───────────────────────────────┼─────────────────────┘
              ▼                         ▼                                ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                    CD / ORCHESTRATION LAYER (Module 06, Module 20)                          │
│   Kubernetes/KServe rollout  │  Airflow/Prefect/Dagster DAGs for retraining, re-embedding,   │
│   (canary / blue-green)      │  scheduled eval reruns, index refresh jobs                    │
└──────────────────────────────────────────┬──────────────────────────────────────────────────┘
                                            ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                         SERVING LAYER (Module 21)                                           │
│  ┌───────────────────┐   ┌───────────────────────┐   ┌────────────────────────────────┐    │
│  │ Classical model     │   │ LLM inference engine  │   │ Vector database                │    │
│  │ serving (KServe,    │   │ (vLLM / Triton /      │   │ (Qdrant/Pinecone/Milvus/        │    │
│  │ Triton for tabular) │   │  TensorRT-LLM / SGLang│   │  Weaviate — Module 16)          │    │
│  │ (M06, M21)          │   │  or hosted API)       │   │                                 │    │
│  └──────────┬──────────┘   └──────────┬────────────┘   └────────────────┬────────────────┘   │
└─────────────┼─────────────────────────┼──────────────────────────────────┼──────────────────┘
              ▼                         ▼                                  ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER — RAG & AGENTS (Module 15-18)                          │
│   Retriever (hybrid search, re-ranking) → context assembly → LLM call → tool-calling agent   │
│   loop (MCP/function calling, LangGraph/CrewAI) → structured/guarded output                  │
└───────────────────────────────────────┬────────────────────────────────────────────────────┘
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│              GATEWAY / API LAYER  (token metering, auth, guardrails — M08, M23, M24)        │
└───────────────────────────────────────┬────────────────────────────────────────────────────┘
                                         ▼
                                   END USER / CLIENT
                                         │
              ┌──────────────────────────┴───────────────────────────────┐
              ▼                                                          ▼
┌───────────────────────────────────────────┐          ┌───────────────────────────────────────┐
│  OBSERVABILITY (Module 14, Module 18)       │          │  FEEDBACK / DRIFT LOOP (Module 19)      │
│  OpenTelemetry traces, Prometheus metrics,   │          │  Production traffic sampled → compared  │
│  Grafana dashboards, LLM-trace tools         │─────────▶│  against baseline distribution/quality  │
│  (Phoenix/LangSmith/Langfuse)                │  drift   │  → retrain/re-prompt/re-index decision  │
│                                               │  signal  │  matrix → triggers CI pipeline again    │
└───────────────────────────────────────────┘          └───────────────────────────────────────┘
                     ▲                                                    │
                     │                                                    │
                     └────────────────── feeds back into ─────────────────┘
                            CI PIPELINE at top of diagram (the CT loop)
```

**Reading this diagram.** The top half (source of truth → CI → registries → CD/orchestration → serving) is the *release path* — this is what Modules 02–06, 13, and 20–22 teach. The bottom half (application layer → gateway → observability → drift/feedback loop) is the *runtime path* — this is what Modules 07–08, 14–19, 23–24 teach. The loop connecting observability back to the CI pipeline at the top is Continuous Training/Continuous Improvement — the single structural feature that makes this diagram different from a plain DevOps deployment diagram. Security (Module 23) and cost tracking (Module 24) are not drawn as one box because they are cross-cutting — they touch nearly every box above, which is precisely why the course teaches them after the architecture exists, not before.

---

## 2. Sequence Diagram — One Production Request Through a RAG + Agent System

This traces a single user request through a mature system that uses both RAG (Module 15/16) and agentic tool-calling (Module 17), with observability (Module 14/18) instrumented throughout. This is the level of detail the capstone (Module 26) expects you to be able to reproduce.

```
 USER      GATEWAY      AGENT         RETRIEVER      VECTOR DB      TOOL(S)       LLM          OBSERVABILITY
  │           │            │               │              │            │           │                │
  │  request  │            │               │              │            │           │                │
  │──────────▶│            │               │              │            │           │                │
  │           │ auth+quota │               │              │            │           │                │
  │           │ check (M23)│               │              │            │           │                │
  │           │───────────────────────────────────────────────────────────────────────────────────▶│ span: request start
  │           │            │               │              │            │           │                │  (trace_id issued)
  │           │  forward   │               │              │            │           │                │
  │           │───────────▶│               │              │            │           │                │
  │           │            │  plan step 1: │               │            │           │                │
  │           │            │  "need context"│              │            │           │                │
  │           │            │──────────────▶│               │            │           │                │
  │           │            │               │  embed query  │            │           │                │
  │           │            │               │──────────────▶│            │           │                │
  │           │            │               │   top-k docs  │            │           │                │
  │           │            │               │◀──────────────│            │           │                │
  │           │            │  retrieved,   │               │            │           │                │
  │           │            │  re-ranked    │               │            │           │                │
  │           │            │◀──────────────│               │            │           │                │
  │           │            │───────────────────────────────────────────────────────────────────────▶│ span: retrieval
  │           │            │                                                                          │  (docs, scores, latency)
  │           │            │  assemble context + prompt (versioned template, M09)                     │
  │           │            │  call LLM ─────────────────────────────────────────────▶│                │
  │           │            │                                                          │  reasons:      │
  │           │            │                                                          │  "need tool X" │
  │           │            │◀─────────────────────────────────────────────────────────│                │
  │           │            │───────────────────────────────────────────────────────────────────────▶│ span: LLM call #1
  │           │            │                                                                          │  (tokens in/out, TTFT,
  │           │            │  invoke tool (function/MCP call, M17)                                     │   cost — M08/M24)
  │           │            │──────────────────────────────────────────▶│                              │
  │           │            │                              tool result   │                              │
  │           │            │◀──────────────────────────────────────────│                              │
  │           │            │───────────────────────────────────────────────────────────────────────▶│ span: tool call
  │           │            │  call LLM again with tool result ─────────────────────────▶│              │
  │           │            │                                                             │ final answer │
  │           │            │◀────────────────────────────────────────────────────────────│              │
  │           │            │───────────────────────────────────────────────────────────────────────▶│ span: LLM call #2
  │           │            │  (optional) guardrail / schema validation check (M23)                     │
  │           │  response  │               │              │            │           │                │
  │           │◀───────────│               │              │            │           │                │
  │  response │            │               │              │            │           │                │
  │◀───────────│           │               │              │            │           │                │
  │           │            │               │              │            │           │                │
  │           │            │               │              │            │           │                │  ── trace closed;
  │           │            │               │              │            │           │                │     full trace (all
  │           │            │               │              │            │           │                │     spans, one trace_id)
  │           │            │               │              │            │           │                │     shipped to LangSmith/
  │           │            │               │              │            │           │                │     Langfuse/Phoenix (M18)
                                                                                                          │
                                                                                                          ▼
                                                                              sampled + scored asynchronously by
                                                                              LLM-as-judge (M11) and checked
                                                                              against drift baseline (M19)
```

**Why this matters.** Notice that a *single* user request generates: one retrieval span, at least two LLM-call spans (planning + final answer), one tool-call span, and one trace tying them together with a shared `trace_id`. This is exactly the structure that lets you answer the "governance gap" question from Section 4 of the tutorial — "which prompt version, which retrieved documents, and which index snapshot produced this bad output" — *if and only if* every span is tagged with the prompt version (M09), the vector index snapshot ID (M16), and the model/version identifier (M03) at the time the request ran. Systems that skip this tagging can see that a request happened, but cannot reconstruct why it went wrong. This single insight is worth internalizing now — it is the connective tissue between Modules 03, 09, 14, 16, and 18.

---

## 3. Decision Tree — Which Practices Does This Project Actually Need?

Use this the first time you start any new project during or after this course, to decide how much of the machinery in this course to actually apply.

```
                         ┌───────────────────────────────────────┐
                         │ Does the system make a prediction,     │
                         │ generate content, or make a decision   │
                         │ using a model (trained or pretrained)? │
                         └───────────────────┬───────────────────┘
                                    NO        │        YES
                          ┌───────────────────┘        └───────────────────┐
                          ▼                                                 ▼
              ┌───────────────────────┐                     ┌───────────────────────────────┐
              │ Plain DevOps is       │                     │ Does it call a large language   │
              │ sufficient. Do NOT    │                     │ model (hosted API or self-      │
              │ add MLOps machinery — │                     │ hosted), even without fine-     │
              │ it is pure overhead.  │                     │ tuning?                          │
              └───────────────────────┘                     └───────────────┬────────────────┘
                                                       NO                    │            YES
                                             ┌────────────────────────────────┘             │
                                             ▼                                               ▼
                              ┌───────────────────────────────┐            ┌───────────────────────────────────┐
                              │ Classical MLOps applies.       │            │ LLMOps applies (on top of MLOps     │
                              │ You need: data/model versioning│            │ foundations). At minimum you need:  │
                              │ (M03), a registry (M03/M13),   │            │  - versioned prompts (M03/M09)      │
                              │ offline eval gates (M02/M12),  │            │  - token cost/latency tracking       │
                              │ drift monitoring (M04/M19),    │            │    (M08/M24)                        │
                              │ and CI/CD (M02/M25).           │            │  - output evaluation beyond exact-   │
                              │                                 │            │    match (M09-M11)                  │
                              │ Ask next: is it retrained on a │            │  - basic guardrails (M23)           │
                              │ schedule or trigger?            │            └──────────────────┬───────────────┘
                              └───────────────┬───────────────┘                                │
                                     NO       │       YES                        Ask next: ─────┴──────────────┐
                          ┌────────────────────┘       └────────────────┐        does it augment the LLM's      │
                          ▼                                              ▼        context with retrieved         │
             ┌─────────────────────────┐                ┌─────────────────────────┐  external documents (RAG)?   │
             │ Skip Continuous          │                │ Add Continuous Training  │                            │
             │ Training (M20) for now — │                │ pipeline: orchestration  │        NO         YES       │
             │ manual retrain is fine   │                │ (M20) + retrain decision │  ┌─────────────────┴──┐    │
             │ at small scale.          │                │ matrix (M19).            │  │                     │    │
             └─────────────────────────┘                └─────────────────────────┘  ▼                     ▼    │
                                                                                 (no RAG          Add vector DB   │
                                                                                  needed;          (M16) + chunking/
                                                                                  skip M15/16)      retrieval design
                                                                                                     (M15)         │
                                                                                                                    │
                                                              ┌─────────────────────────────────────────────────┘
                                                              ▼
                                            Ask next: does it decide, at runtime, which
                                            tools/actions to invoke and in what order
                                            (agentic behavior), rather than one prompt-in/
                                            response-out call?
                                                     │
                                    NO               │               YES
                          ┌──────────────────────────┘               └────────────────────────┐
                          ▼                                                                     ▼
             ┌─────────────────────────┐                                     ┌─────────────────────────────────┐
             │ Single-call LLM pattern  │                                     │ Agentic system: add tool-calling  │
             │ is sufficient — skip     │                                     │ framework (M17, e.g. LangGraph/   │
             │ M17/M18's agent-specific │                                     │ CrewAI, or MCP-based tools) AND   │
             │ tracing complexity.      │                                     │ mandatory trace-level debugging   │
             └─────────────────────────┘                                     │ (M18) — agentic failures compound  │
                                                                               │ across steps and are much harder  │
                                                                               │ to root-cause without it.         │
                                                                               └─────────────────────────────────┘

  Regardless of where you land above, ALWAYS add (they are cheap early, expensive late):
    - basic secrets management (M04/M23)
    - structured logging with request/trace IDs (M14)
    - at least a minimal held-out evaluation set (M10), even 20-50 examples
```

**How to use this tree in practice.** Walk it once per project, not once per company — a single organization commonly runs some services that only need plain DevOps, some that need classical MLOps, and some that need full LLMOps, simultaneously. The tree's purpose is to stop two opposite mistakes this course sees constantly: over-engineering a simple internal script with a full model registry and drift dashboard, and under-engineering a production LLM-backed feature by treating it as "just an API call" with none of versioning, cost tracking, or evaluation in place.

---

## 4. How to Read the Rest of This Course Against These Diagrams

Keep a mental (or literal, printed) copy of the Section 1 reference architecture nearby as you progress. Concretely:

- When you reach **Module 03** (Versioning/Registries), you are building the "Registry Layer" band.
- When you reach **Module 06** (Kubernetes), you are building the "CD/Orchestration Layer" band.
- When you reach **Modules 15–18** (RAG/Agents), you are building the entire "Application Layer" band and living inside the sequence diagram in Section 2.
- When you reach **Module 19** (Drift), you are building the "Feedback/Drift Loop" box and closing the loop back to the top of the Section 1 diagram.
- When you reach **Module 26** (Capstone), your job is to have every band in Section 1 represented, even minimally, in one working system — and to be able to draw your own version of the Section 2 sequence diagram for a real request flowing through it.

This is the single most useful "am I actually integrating what I've learned, or just accumulating disconnected tool knowledge" test available in this course — if you cannot point to where your capstone project implements each band of this diagram, that is a signal to go back, not to push forward.
