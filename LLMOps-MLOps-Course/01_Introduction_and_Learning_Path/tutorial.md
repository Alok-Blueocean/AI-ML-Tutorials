# Module 01 — Introduction and Learning Path

> "In machine learning, the ML code itself is usually a small fraction of a real-world ML system. The required surrounding infrastructure is vast and complex." — Sculley et al., *Hidden Technical Debt in Machine Learning Systems*, NeurIPS 2015

This is the orientation chapter for the entire 26-module course. It does not teach a tool. It builds the mental map you will keep referring back to as you move through Docker, Kubernetes, prompt engineering, evaluation, RAG, agents, serving infrastructure, and the capstone. Read it slowly once, then skim it again after Module 10 and after Module 18 — it will mean more each time.

---

## 1. Learning Objectives, Prerequisites, and Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. State precisely what MLOps is, what problem it solves, and why "just use good DevOps practices" is not sufficient for ML systems.
2. State precisely what LLMOps is, and explain it as an **extension layer** on top of MLOps rather than a separate discipline — and name the specific new artifacts and failure modes it introduces (prompts, embeddings, vector indexes, token economics, hallucination).
3. Distinguish DevOps, MLOps, and LLMOps along five axes: what changes over time, what "testing" means, what gets versioned, what gets monitored, and what a rollback looks like.
4. Run a personal skills self-assessment and identify which prerequisite gaps (if any) to fill before continuing.
5. Read the full 26-module roadmap and explain *why* the modules are sequenced the way they are — which is itself a lesson in how a production LLMOps platform gets built in the real world (foundations first, then release engineering, then application-layer concerns, then infrastructure at scale, then governance/cost, then integration).
6. Plan realistic study time and pacing for the course based on your background and available hours per week.

### Prerequisites

This course assumes, and does not re-teach:

| Area | Expected level |
|---|---|
| Python | Comfortable writing functions, classes, virtual environments, `pip`/`poetry`, reading stack traces, using type hints |
| Basic ML | Know what training/validation/test splits are, what overfitting is, what a model's inputs/outputs look like (you do not need to derive backpropagation) |
| Command line | Comfortable with a shell (bash or PowerShell), navigating directories, running scripts, reading logs |
| Git | `clone`, `commit`, `branch`, `merge`, resolving a conflict, opening a pull request |
| HTTP basics | What a REST API is, what a status code means, what JSON looks like |
| Cloud concepts (helpful, not required) | What a VM, a container, and object storage (S3/GCS) are, at a conceptual level |

You do **not** need prior experience with Docker, Kubernetes, MLflow, LangChain, vector databases, or any LLM provider API — those are taught from first principles starting in Modules 05, 06, 13, 15/16/17, and 07 respectively.

### Skills Self-Assessment Checklist

Before starting Module 02, honestly check off what you already have:

```
[ ] I can write a Python script that reads a config file, calls a function, and writes a log line.
[ ] I can create and activate a virtual environment (venv, conda, or poetry) without looking it up.
[ ] I know the difference between training, validation, and test data, and why you don't test on training data.
[ ] I can write a Dockerfile OR I am willing to learn it in Module 05 (no prior Docker required).
[ ] I can explain what an API endpoint is and call one with curl or Python's `requests`.
[ ] I have used git for at least one real project (solo or team).
[ ] I understand, at least loosely, what an LLM (like GPT-4, Claude, or Llama) is and that you send it text and get text back.
[ ] I know what "overfitting" means intuitively (model memorizes training data, fails on new data).
```

If you checked fewer than 5 of these 8 boxes, spend a few days on introductory Python/Git/HTTP material before continuing — this course moves at a practitioner's pace from Module 02 onward and will not stop to explain what a virtual environment is.

### Key Terminology

Precise definitions you will reuse constantly. Get these right now; sloppy terminology is the single biggest source of confusion for engineers new to this field.

| Term | Definition |
|---|---|
| **MLOps** | The discipline and set of practices for taking a machine learning model from a trained artifact to a reliably operating, monitored, continuously-improvable production system — unifying ML development (the "Dev" of model-building) with ML operations (the "Ops" of running it reliably at scale). |
| **LLMOps** | The subset/extension of MLOps practices specific to systems built around large language models — covers prompt versioning, retrieval/embedding pipelines, token-cost management, and LLM-specific evaluation (including LLM-as-judge), layered on top of (not replacing) the MLOps foundation of CI/CD, versioning, and monitoring. |
| **Continuous Integration (CI)** | Automatically building and testing every code change before it merges. In ML, this expands to also validating data schemas and running model-quality checks, not just unit tests. |
| **Continuous Delivery/Deployment (CD)** | Automatically packaging and releasing a validated change to staging/production. In ML, "the artifact" is not just code — it can be a model binary, a prompt template, or a fine-tuned checkpoint. |
| **Continuous Training (CT)** | An ML-specific addition with no DevOps equivalent: automatically retraining and redeploying a model when new data arrives or performance degrades, without a human writing new code. |
| **Model registry** | A versioned, queryable store of trained model artifacts plus their metadata (metrics, lineage, stage: staging/production/archived) — the ML analogue of a container/artifact registry. Covered in depth in Module 03 and Module 13. |
| **Drift** | A change in the statistical properties of input data (data drift) or in the relationship between inputs and the correct output (concept drift) that degrades a deployed model's real-world performance even though the model itself hasn't changed. Covered in Modules 04 and 19. |
| **Prompt engineering** | The practice of designing, structuring, and iterating on the text (and structure) sent to an LLM to reliably elicit a desired behavior — the LLMOps-era analogue of feature engineering. |
| **RAG (Retrieval-Augmented Generation)** | An architecture pattern where an LLM's context is augmented at inference time with documents retrieved from an external knowledge store (typically a vector database), instead of relying solely on the model's trained-in knowledge. |
| **Agent / agentic system** | An LLM-driven system that can decide, at runtime, which tools or actions to invoke and in what sequence, to accomplish a multi-step goal — as opposed to a single prompt-in/response-out call. |
| **LLM-as-judge** | Using a (usually stronger or differently-calibrated) LLM to score or compare the outputs of another LLM against a rubric, as a scalable substitute for exhaustive human evaluation. |
| **Observability** | The ability to ask arbitrary questions about a running system's internal state from its external outputs (logs, metrics, traces) — as opposed to monitoring, which answers a fixed set of pre-defined questions ("is latency > threshold?"). |
| **Governance gap** (2026 industry term) | The widely-discussed observation that LLMOps tooling for *deployment* (serving, scaling, gateways) matured quickly, but tooling for *lineage and governance* — tracing which prompt version, retrieved documents, or vector index state produced a specific bad output — remains roughly where classical MLOps lineage tooling was around 2018: immature and mostly bespoke. |

---

## 2. Why This Topic Matters, and Where It Fits in the Lifecycle

### The core problem MLOps exists to solve

A data scientist can train a good model in a Jupyter notebook in an afternoon. Getting that model to serve millions of correct, monitored, rollback-able predictions in production for two years is a different, much harder engineering problem — and it is the problem this entire course is about.

The seminal argument for why this is hard, and why it deserves its own discipline rather than being "just DevOps," comes from Google's 2015 NeurIPS paper *Hidden Technical Debt in Machine Learning Systems* (full reference in this folder's `references.md`). Its central diagram is famous for a reason: it shows a tiny box labeled "ML Code" surrounded by a much larger ring of boxes — data collection, feature extraction, data verification, process management tools, analysis tools, machine resource management, serving infrastructure, monitoring, configuration. The paper's argument, still true a decade later, is that:

- ML systems have all the technical debt of ordinary software, **plus** ML-specific debt that has no traditional-software analogue.
- The worst of this is **entanglement**: because ML systems learn statistical relationships from data rather than having those relationships hand-written, changing *anything* — a feature, a hyperparameter, an upstream data source — can silently change the behavior of *everything*, a property they nickname **CACE: Changing Anything Changes Everything**.
- Traditional software testing (unit tests, integration tests) verifies that code does what the code says. It cannot verify that a *model* still behaves correctly, because "correct" is defined statistically against data, not defined by the code.

MLOps is the set of engineering practices — versioning, automated evaluation gates, monitoring for drift, staged rollout, automated retraining — built specifically to manage this extra category of risk.

### Where this fits in the lifecycle

```
                         THE ML/LLM SYSTEM LIFECYCLE
   ┌───────────────────────────────────────────────────────────────────┐
   │                                                                     │
   │   DATA           MODEL/PROMPT        RELEASE          OPERATE      │
   │  ┌───────┐       ┌───────────┐      ┌─────────┐      ┌─────────┐   │
   │  │Collect│──────▶│Train/Tune │─────▶│CI/CD +  │─────▶│ Serve + │   │
   │  │Version│       │or Design  │      │Eval Gate│      │ Observe │   │
   │  │Validate│      │Prompt/RAG │      │Registry │      │ Monitor │   │
   │  └───────┘       └───────────┘      └─────────┘      └────┬────┘   │
   │      ▲                                                     │        │
   │      │                 CONTINUOUS TRAINING /                       │
   │      └─────────────  RE-EVALUATION LOOP  ◀────────────────┘        │
   │                    (drift detected → retrain/re-prompt)            │
   └───────────────────────────────────────────────────────────────────┘
      MLOps = engineering the whole loop reliably, not just the middle box
```

Notice the loop at the bottom. This is the feature that makes MLOps/LLMOps fundamentally different from shipping a web app: **the system is expected to degrade on its own, even with zero code changes**, because the world the model or prompt was built against keeps moving. Traditional DevOps has no equivalent concept — a correctly deployed microservice does not spontaneously start returning wrong answers because user behavior shifted. An ML model can, and does.

This course is organized around walking this entire loop once for classical ML (Modules 02–06, 19–22) and once for LLM-based systems (Modules 07–18), before bringing both tracks together for serving-at-scale, security/cost, and the capstone (Modules 20–26).

---

## 3. Main Concepts

### 3.1 DevOps vs. MLOps vs. LLMOps

**Theory.** DevOps solved the problem of unreliable, manual software releases by treating infrastructure as code, automating build/test/deploy, and instrumenting production systems. Its central assumption is that **the artifact is code**, and that code's behavior is deterministic and fully specified by its logic — the same input always produces the same output, and a passing test suite is strong evidence the system behaves correctly.

MLOps keeps every DevOps practice (version control, CI/CD, IaC, monitoring) but must add practices for artifacts whose behavior is *learned from data* rather than hand-specified, and whose correctness cannot be fully verified by code review or unit tests alone. Concretely, MLOps must add:

- **Data versioning and validation** — because the "logic" partly lives in the data, not the code.
- **Model versioning and a model registry** — a trained model binary is a build artifact with no source-code equivalent; two runs of the "same" training code with different data or even different random seeds can produce meaningfully different models.
- **Continuous Training (CT)** — a release trigger that doesn't exist in DevOps: "new data arrived, performance has drifted, retrain and consider re-releasing" with no human writing new application code.
- **Model/behavioral evaluation gates** — beyond pass/fail unit tests, ML releases need statistical evaluation against held-out data (accuracy, calibration, fairness metrics) before promotion.
- **Drift monitoring** — production monitoring in DevOps watches latency/error-rate/resource-usage; MLOps must *additionally* watch whether the statistical distribution of inputs or the model's real-world accuracy is silently degrading, even when latency and error rate look perfectly healthy.

LLMOps then extends MLOps again, because large language models introduce artifacts and failure modes that don't fit cleanly into the classical-ML picture:

- **Prompts and prompt templates become versioned, tested, rollback-able artifacts** — as important to govern as the model weights themselves, and much more frequently changed (Module 03, Module 09).
- **Retrieved context and vector indexes become part of "the model's input"** — in RAG systems, a bad answer might trace back to a stale vector index, a bad chunking strategy, or a bad retriever, not the LLM at all (Modules 15, 16, 18).
- **Non-determinism and open-ended output space** — an LLM's output is free-form text, not a fixed label or number, so "did it pass" often cannot be checked with exact-match assertions and instead requires rubric-based or LLM-as-judge evaluation (Modules 09, 10, 11).
- **Hallucination** — a genuinely new failure mode: a fluent, confident, well-formatted, entirely wrong answer, with no equivalent in classical ML (a classifier is wrong or right; it doesn't fabricate a plausible-sounding wrong class with invented supporting "reasoning").
- **Token-based cost and latency economics** — cost and latency now scale with *prompt and output length*, not just request volume, requiring their own monitoring and optimization layer (Module 08, Module 24).
- **Agentic and multi-step failure modes** — when an LLM can call tools and take actions, a single bad reasoning step can cascade into wrong real-world side effects, requiring tracing tools built specifically for this (Modules 17, 18).

A concise industry framing (Red Hat, 2025–2026) states this well: **LLMOps is not a rival discipline to MLOps — it's what MLOps looks like once your "model" is a foundation model plus a prompt plus a retrieval pipeline instead of a model you trained yourself.** Every classical MLOps concern (versioning, CI/CD, monitoring, registries) still applies; it's just applied to new kinds of artifacts.

**Comparison table.**

| Dimension | DevOps | MLOps | LLMOps |
|---|---|---|---|
| Primary artifact | Application code / container image | Code + trained model weights + data | Code + prompt templates + (optionally) fine-tuned weights + vector index/retrieval config |
| What changes over time, unprompted | Nothing (code is static until someone commits) | Real-world data distribution (drift) | Data drift + upstream provider model updates + retrieved-document staleness |
| "Testing" before release | Unit/integration/E2E tests, deterministic pass/fail | Above, plus offline evaluation against held-out data (statistical, not deterministic) | Above, plus rubric/LLM-as-judge evaluation over open-ended text output |
| What gets versioned | Source code | Source code + datasets + model artifacts | Source code + prompts + datasets + vector index snapshots + (if applicable) fine-tuned weights |
| What production monitoring watches | Latency, error rate, throughput, resource usage | All of DevOps, plus prediction distribution, data drift, model accuracy proxy metrics | All of MLOps, plus token cost, TTFT/P95 latency, hallucination rate, retrieval quality, tool-call success rate |
| Rollback unit | Previous container image / commit | Previous model version in registry | Previous prompt version and/or previous model/index version |
| Retraining/re-tuning trigger | N/A (no equivalent) | Continuous Training (CT): drift detected or schedule-based | Prompt iteration, index refresh, or (less often) fine-tune refresh |
| Novel failure mode | Regression bug | Silent accuracy degradation with no code change | Hallucination; fluent, confident, wrong output |

**Architecture — how the three "orbit" each other.**

```
        ┌───────────────────────────────────────────────────────────┐
        │                         LLMOps                            │
        │   prompt/template mgmt · vector DB & retrieval ·           │
        │   token-cost tracking · LLM-as-judge eval ·                │
        │   agent tracing · hallucination monitoring                 │
        │   ┌───────────────────────────────────────────────────┐    │
        │   │                     MLOps                          │   │
        │   │  data/model versioning · model registry ·         │    │
        │   │  offline eval gates · drift detection ·           │    │
        │   │  continuous training · feature stores             │    │
        │   │   ┌───────────────────────────────────────────┐   │    │
        │   │   │                 DevOps                     │   │    │
        │   │   │  CI/CD · IaC · containers/orchestration ·  │   │    │
        │   │   │  logging/metrics/tracing · secrets mgmt     │   │    │
        │   │   └───────────────────────────────────────────┘   │    │
        │   └───────────────────────────────────────────────────┘    │
        └───────────────────────────────────────────────────────────┘
        Each ring assumes and reuses everything inside it. Nothing is thrown away.
```

**When to apply which mindset.** If you are shipping a stateless microservice with no learned/generative component, plain DevOps is sufficient and adding MLOps machinery (registries, drift dashboards) is pure overhead. If you are shipping any system whose behavior is learned from data — a classifier, a recommender, a ranking model — you need MLOps practices even if you never touch an LLM. If any part of your system calls an LLM (even a single prompt-in/response-out call to a hosted API with no fine-tuning, no RAG, and no agentic behavior), you already have an LLMOps surface: you have a prompt to version, token cost to track, and hallucination risk to evaluate — the "LLMOps" ring is not gated behind having a complex agentic architecture.

### 3.2 The overall course arc

The 26 modules are not a flat list — they follow a deliberate build order. Understanding *why* this order was chosen is itself a lesson in how a production LLMOps platform gets assembled by real teams.

```
   RELEASE ENGINEERING           →  the non-negotiable foundation. Nothing else
   (M02–M06)                        matters if you can't safely ship a change.

   PROMPT / CONTEXT ENGINEERING  →  once you can safely ship things, learn what
   (M07)                             you are shipping: how LLMs consume context.

   COST / DEPLOYMENT ECONOMICS   →  before you evaluate anything at scale, you
   (M08)                             must understand what a request costs you.

   EVALUATION METHODOLOGY        →  the hardest and most neglected part of
   (M09–M12)                         LLMOps: how do you know a change is good?

   EXPERIMENT TRACKING           →  formalizes everything above into a system
   (M13)                             of record (this is also classical MLOps).

   PRODUCTION OBSERVABILITY      →  once you can ship + evaluate, you need to
   (M14)                             see what's actually happening in prod.

   RAG / AGENTS                  →  the two dominant production LLM system
   (M15–M18)                        architectures — built on everything above.

   DATA-LAYER MLOPS              →  drift, orchestration, feature stores — the
   (M19–M22)                        classical-ML foundation revisited in depth,
                                     now that you have the LLMOps vocabulary
                                     to contrast it against.

   SERVING AT SCALE              →  M21 sits inside this band: how to serve
   (M20–M22)                        both classical models and LLMs efficiently
                                     under real production load.

   SECURITY / COST / GOVERNANCE  →  cross-cutting concerns that apply to
   (M23–M24)                        everything built so far, addressed last
                                     because they require the full picture.

   CI/CD INTEGRATION             →  ties Modules 02–04's theory into one
   (M25)                             hands-on GitHub Actions pipeline.

   CAPSTONE                      →  integrate all of the above into one
   (M26)                             production-shaped platform.
```

This mirrors how a real platform team actually grows a system: they get release engineering right first (or they get burned repeatedly), then they build the application layer, then they realize evaluation is harder than they thought and invest heavily there, then they add observability once something breaks in prod for the first time, then they build the flagship architectures (RAG/agents), then they harden the data layer and serving infrastructure to handle real load, and only at the end do they formalize security/cost/governance — usually because an audit, an incident, or a cost overrun forces the issue. If you have worked on a real production ML or LLM team, this order will feel less like a syllabus and more like a war story.

### 3.3 The full 26-module roadmap

| # | Module | Focus | Why it sits here |
|---|---|---|---|
| 01 | Introduction and Learning Path | Orientation, MLOps vs LLMOps, prerequisites, full roadmap | You need the map before the terrain — every later module assumes you understand this vocabulary and sequencing rationale. |
| 02 | Foundations of ML/LLM CI/CD | CI/CD gates, DevOps vs MLOps vs LLMOps in pipeline form | Establishes the release-engineering backbone every later module's "how do I ship this safely" question will refer back to. |
| 03 | Versioning, Registries, and Rollback | SemVer for models/prompts/datasets, MLflow/W&B registries, rollback/lineage | You cannot build CI/CD gates (M02) without something to version and roll back — this module supplies the artifact model the pipeline in M02 operates on. |
| 04 | Reproducibility and Environments | Drift, Docker/Conda pinning, dev→staging→prod promotion | Before containerizing (M05) or orchestrating (M06) anything, you need to understand why "works on my machine" fails for ML specifically and what a promotion pipeline must guarantee. |
| 05 | Docker for ML/LLM Systems | Hands-on Docker deep dive | The concrete tool that implements the environment-pinning theory from M04 — taught hands-on now that you know *why* it matters. |
| 06 | Kubernetes for ML/LLM Systems | Hands-on Kubernetes, KServe/Kubeflow | Once a single container (M05) is solid, the natural next question is running many of them reliably at scale — closing out the classical release-engineering block. |
| 07 | Prompt Engineering Fundamentals | Context windows/token budgets, structured prompts, hallucination reduction | With release engineering solid, the course pivots to the LLM application layer — and everything LLM-specific starts with how you construct a prompt. |
| 08 | LLM Latency, Cost, and Deployment | TTFT/P95, batching/caching/streaming, API vs local | Before you can meaningfully evaluate or optimize an LLM system, you need to understand what each request actually costs in time and money. |
| 09 | Prompt Lifecycle and Statistical Evaluation | Prompt versioning, delta tracking, paired t-tests | Prompts (M07) are living artifacts that change constantly — this module formalizes how to test whether a prompt change is actually an improvement, not noise. |
| 10 | Building Evaluation Datasets | Production-sourced gold sets, stratified/edge-case sampling, bias avoidance | Statistical testing (M09) is meaningless without a trustworthy dataset to test against — this supplies that foundation. |
| 11 | LLM-as-Judge Design and Limits | Evaluator prompts, composite scoring, judge bias/calibration | With a gold dataset (M10) in hand, the next problem is scoring open-ended text at scale — LLM-as-judge is the dominant technique, with real caveats you must understand. |
| 12 | Deployment Quality Gates and Dashboards | Eval triggers, release thresholds, readiness dashboards | Closes the evaluation arc (M09–M11) by wiring evaluation results back into the CI/CD gates from M02, now specialized for LLM releases. |
| 13 | Experiment Tracking and MLflow Deep Dive | Full MLflow chapter, W&B/Comet comparison | Formalizes the tracking/registry concepts touched in M03 and used implicitly in M09–M12 into one rigorous, tool-grounded chapter. |
| 14 | Observability, OpenTelemetry, and Monitoring | Inference metrics, OpenTelemetry, Prometheus/Grafana | Once you can ship and evaluate reliably, the next production requirement is seeing what's actually happening live — the natural bridge into the RAG/agent modules that follow. |
| 15 | RAG Systems Deep Dive | Chunking, parent-child retrieval, hybrid search, multi-query, compression, re-ranking | The first of the two flagship production LLM architectures — needs observability (M14) already in place to debug retrieval quality issues. |
| 16 | Vector Databases | Qdrant/Pinecone/Milvus/Weaviate comparison and hands-on | RAG (M15) is impossible without a vector store — this is the infrastructure layer RAG depends on, taught immediately after the concept that motivates it. |
| 17 | Agentic Systems and Tool Calling | Agents, MCP, function calling, memory, planning, LangGraph/CrewAI | The second flagship architecture, and strictly harder than RAG — introduced after RAG because agents often *use* RAG as one of their tools. |
| 18 | Tracing and Debugging Agentic Systems | Phoenix/LangSmith/Langfuse, hallucination root-causing | Agentic systems (M17) are the hardest to debug in the whole course — this module supplies the tracing discipline immediately, before bad habits form. |
| 19 | Drift Detection and Retraining Decisions | Data/behavioral drift, dashboards, retrain decision matrix | Returns to classical MLOps with the full LLMOps vocabulary now available for contrast — drift applies to both classical models and LLM systems (e.g., retrieval drift). |
| 20 | Orchestration: Airflow, Prefect, Dagster | Workflow orchestration for ML/LLM pipelines | Drift detection and retraining (M19) need something to actually schedule and run the retraining/re-embedding jobs — orchestration is that missing piece. |
| 21 | Model Serving at Scale | vLLM, Triton, TensorRT-LLM, SGLang, GPU scheduling/autoscaling | With pipelines orchestrated (M20), the course turns to the highest-leverage infrastructure investment for cost and latency: efficient serving of models at scale. |
| 22 | Data Layer: Feature Stores and Versioning | DVC, LakeFS, Feast | Completes the classical-MLOps data foundation, now informed by the orchestration (M20) and serving (M21) context around it. |
| 23 | Security, Governance, and Responsible AI | Secrets management, guardrails, safety, compliance | A cross-cutting concern deliberately placed after the architectures exist (M15–M22) — you secure and govern a system once you know its full attack surface. |
| 24 | Cost Optimization and FinOps for AI | Token/GPU cost economics | Pairs naturally with M23 as the other major cross-cutting concern — now that serving (M21) and orchestration (M20) costs are concrete, they can be optimized. |
| 25 | CI/CD Pipelines with GitHub Actions | Hands-on pipeline tying Modules 02–04 together practically | A deliberate return to the release-engineering foundation from M02–M04, now built hands-on with everything learned since — the practical payoff of the whole course. |
| 26 | Capstone: Production LLMOps Platform | Final integrated build | Integrates every module into one running system — the single artifact that proves the roadmap actually cohered. |

### 3.4 How the modules build on each other (dependency view)

```
 M01 (you are here)
   │
   ▼
 M02 ──▶ M03 ──▶ M04 ──▶ M05 ──▶ M06        [Release-engineering foundation]
                                   │
                                   ▼
 M07 ──▶ M08 ──▶ M09 ──▶ M10 ──▶ M11 ──▶ M12  [Prompting + evaluation methodology]
                                             │
                                             ▼
                                           M13 ──▶ M14           [Tracking + observability]
                                                     │
                                                     ▼
                                          M15 ──▶ M16            [RAG track]
                                                     │
                                                     ▼
                                          M17 ──▶ M18            [Agent track]
                                                     │
                                                     ▼
                                   M19 ──▶ M20 ──▶ M21 ──▶ M22   [Data/serving infra]
                                                     │
                                                     ▼
                                          M23 ──▶ M24            [Security + cost]
                                                     │
                                                     ▼
                                          M25 ──▶ M26            [Integration + capstone]
```

Each arrow means "assumes you understand the previous node's core concepts, and will feel much harder if you skip it." You can read modules out of order for reference purposes (e.g., jumping straight to Module 16 to look up a vector database comparison), but the *first pass* through the course should follow this order — the exercises and case studies in later modules routinely assume vocabulary and running code from earlier ones.

### 3.5 Study-time budget and pacing

Estimates below assume the self-assessment in Section 1 mostly checks out (5+ of 8 boxes) and roughly 6–10 focused hours per week.

| Block | Modules | Estimated hours | Suggested pacing |
|---|---|---|---|
| Orientation | 01 | 1–2 | Day 1 |
| Release engineering | 02–06 | 18–24 | Weeks 1–3 |
| Prompting + LLM economics | 07–08 | 8–10 | Week 4 |
| Evaluation methodology | 09–12 | 16–20 | Weeks 5–6 |
| Tracking + observability | 13–14 | 10–12 | Week 7 |
| RAG | 15–16 | 12–16 | Weeks 8–9 |
| Agents | 17–18 | 12–16 | Weeks 9–10 |
| Classical-MLOps data layer | 19–22 | 20–24 | Weeks 11–13 |
| Security, governance, cost | 23–24 | 8–10 | Week 14 |
| CI/CD integration | 25 | 6–8 | Week 15 |
| Capstone | 26 | 20–30 | Weeks 16–18 |
| **Total** | **26 modules** | **~130–170 hours** | **~16–18 weeks at 8–10 hrs/week** |

If you already have strong Docker/Kubernetes experience, you can compress Modules 05–06 significantly; if you already have production LLM-application experience, you can compress Modules 07–08. Conversely, engineers with no prior cloud/container exposure should budget roughly 50% more time for the 05–06 and 20–22 blocks. Treat the capstone (Module 26) budget as a floor, not a ceiling — integration work reliably takes longer than any individual component suggests it should, which is itself a lesson in production engineering.

---

## 4. Real-World Case Studies

These are reasoned inferences about how sophisticated engineering organizations *would plausibly* structure MLOps/LLMOps given their publicly known engineering-blog patterns, industry norms, and the scale they operate at — not confirmed internal specifics.

**A company like Netflix** operates at the scale where "just retrain and redeploy manually" breaks almost immediately — their public engineering writing over the years about experimentation platforms and personalization systems suggests they'd need strict separation between the *offline experiment* (does this new recommendation model look better against historical data) and the *online guardrail* (a live A/B test on real traffic with automatic rollback if a metric regresses) — which is exactly the CI/CD-gate-plus-online-monitoring pattern this course builds up across Modules 02, 12, and 19.

**A company like Uber**, given its long-public work on internal ML platforms, would plausibly need a **model/feature registry system** (echoing this course's Modules 03 and 22) simply because thousands of models (pricing, ETA, fraud, matching) sharing overlapping features cannot be managed as one-off scripts — the entanglement problem from the Sculley et al. paper becomes existential at that scale without a shared, versioned feature layer.

**A company like Spotify**, which has spoken publicly about "golden paths" and platform standardization for internal teams, would plausibly formalize an internal MLOps platform less to enable any single model and more to let hundreds of internal teams *not* reinvent CI/CD, registries, and monitoring independently — directly mirroring why this course treats Modules 02–04 as a shared foundation rather than something re-derived per project.

**A company like OpenAI or Anthropic**, operating LLM APIs at massive scale, would plausibly need LLMOps practices pushed to an extreme most companies never reach: prompt/system-message versioning with strict rollback (Module 03/09), heavy investment in automated evaluation including LLM-as-judge at scale (Module 11) because human review cannot scale to their request volume, and deep gateway-level token-cost and latency tracking (Module 08/24) since compute cost is close to their primary cost driver. Their public safety and evaluation research also plausibly motivates why this course treats evaluation (Modules 09–12) as harder and more important than the model-serving mechanics themselves.

**A company like Databricks or NVIDIA**, as infrastructure/tooling vendors rather than pure model operators, would plausibly need their MLOps/LLMOps story to be *generalizable* across arbitrary customer workloads — which is a plausible reason their public tooling (MLflow, NVIDIA's serving stack referenced in Modules 13 and 21) emphasizes open standards and pluggability over vertically-integrated, single-use-case pipelines. This is also why this course teaches MLflow (Module 13) and vLLM/Triton/TensorRT-LLM (Module 21) as the representative, broadly-applicable tools rather than any single company's bespoke internal system.

**A widely discussed 2026 industry framing** — the "governance gap" mentioned in this module's `references.md` — describes LLMOps deployment tooling as mature while lineage/governance tooling lags roughly where classical MLOps was around 2018. Concretely: many real production RAG/agent systems today still cannot cheaply answer "which prompt version, which retrieved documents, and which vector index snapshot produced this specific bad output six weeks ago." This is precisely the gap Modules 03, 18, and 23 exist to close for you before you build production systems that repeat this industry-wide mistake.

---

## 5. Common Mistakes

1. **Treating LLMOps as a total replacement for MLOps, rather than an extension.** Teams that skip classical MLOps fundamentals (versioning, CI/CD gates, monitoring discipline) and go straight to prompt engineering end up with LLM systems that have all the same reliability problems ML systems had in 2016 — just with a chatbot UI on top.
2. **Assuming "no training" means "no MLOps needed."** Calling a hosted LLM API with zero fine-tuning still has a versioned artifact (the prompt), a non-deterministic output space, real cost/latency dynamics, and hallucination risk — skipping MLOps discipline here is the single most common beginner mistake in this field.
3. **Skipping the evaluation modules (09–12) to get to "cooler" topics like agents.** Teams that build elaborate agentic systems (M17) without evaluation discipline (M09–M12) in place cannot tell whether a change made the system better or worse — they are flying blind and will discover this the hard way in production.
4. **Under-investing in observability until after an incident.** This course places Module 14 deliberately before RAG/agents (M15–M18) precisely because most teams do this backwards in the real world, and it is expensive.
5. **Ignoring reproducibility until it causes a production incident.** "Works on my machine" bugs in ML systems are worse than in ordinary software because they can silently affect model accuracy rather than crashing loudly (Module 04).
6. **Confusing a model registry with source control.** Git tracks code; it does not meaningfully version large model binaries, datasets, or prompt-experiment lineage — teams that try to force-fit these into git alone hit scaling and traceability walls (Module 03).
7. **Treating cost optimization (M24) and security/governance (M23) as "later" concerns indefinitely.** These are cheapest to build in from the start and expensive to retrofit — the course sequences them near the end of the *teaching* order, but production systems should not defer them indefinitely in the *build* order.

---

## 6. Best Practices and Production Tips

- **When to apply full MLOps rigor:** any model or LLM-backed feature that affects real users or real money, especially anything that will be iterated on more than once. A one-off analysis notebook does not need a model registry.
- **When NOT to over-engineer:** a prototype or internal proof-of-concept exploring feasibility does not need CI/CD gates, drift dashboards, or a full registry on day one — but design it so those can be added without a rewrite (e.g., don't hardcode prompts inline everywhere if you'll want to version them next month).
- **Alternatives to building your own platform:** managed platforms (e.g., cloud-vendor ML platforms, or LLM-specific observability/eval SaaS products) can replace large parts of Modules 13, 14, and 18's tooling for teams that don't want to operate the infrastructure themselves — this course teaches the underlying concepts and open-source tools so you can evaluate those managed alternatives intelligently, not to argue you must always self-host.
- **Cost:** the biggest early-course cost mistake is conflating "token cost" with "total cost of ownership" — engineering time spent debugging un-versioned prompts or un-observable agent failures usually dwarfs the API bill for small-to-medium systems.
- **Scaling:** the roadmap's ordering (release engineering → evaluation → observability → architecture → infra) is also the order in which most scaling pain actually surfaces — don't invest in Module 21-grade serving infrastructure before Module 09–12-grade evaluation discipline exists to tell you whether your system is even correct.
- **Monitoring:** decide, module by module, whether you are monitoring (a fixed dashboard of known failure modes) or building true observability (the ability to answer a question you didn't anticipate asking) — Module 14 draws this distinction sharply.
- **Security:** don't wait until Module 23 in your own real projects — secrets management and basic guardrails belong in the environment setup from Module 04 onward, even though the course teaches the full depth later.
- **Performance tradeoffs:** almost every tool choice in this course (Module 16's vector databases, Module 21's serving engines) is a latency/cost/accuracy triangle — this module's job is to make sure you know that triangle exists before you get into any single tool's specifics.

---

## 7. Interview Questions

**Q1: What is MLOps, in one sentence, and what specifically does it add on top of standard DevOps?**
A: MLOps is the discipline of reliably taking ML models from trained artifact to monitored production system; on top of DevOps's CI/CD and IaC, it adds data/model versioning, statistical (not just pass/fail) evaluation gates, drift monitoring, and continuous training — because ML system behavior is learned from data, not fully specified by code.

**Q2: Is LLMOps a separate discipline from MLOps? Justify your answer.**
A: No — the well-supported industry framing is that LLMOps is an extension layer on MLOps, not a replacement. It reuses MLOps's versioning/CI-CD/monitoring foundation and adds LLM-specific artifacts and concerns: prompt versioning, vector-index/retrieval management, token-cost tracking, and LLM-as-judge evaluation for open-ended text output.

**Q3: Give a concrete example of "hidden technical debt" (Sculley et al.) that would not show up in a code review.**
A: Entanglement/CACE — e.g., changing an upstream feature's normalization method doesn't break any code, compiles fine, passes unit tests, but silently shifts the input distribution to a downstream model enough to degrade its real-world accuracy. No traditional code review or unit test catches this because the code is "correct"; only evaluation against real data would.

**Q4: Why can't you rely on unit tests alone to validate an ML model before deploying it?**
A: Unit tests verify that code executes the logic it's supposed to; they can't verify that a model's *learned statistical behavior* is still accurate, because correctness for an ML model is defined against a data distribution, not a fixed specification. You need offline evaluation against held-out/gold datasets (and for LLMs, potentially LLM-as-judge scoring), not just "did the function run without throwing."

**Q5: What is Continuous Training (CT), and why doesn't traditional DevOps have an equivalent?**
A: CT is the automated retraining and re-release of a model triggered by new data or detected performance drift, without a human writing new application code. DevOps has no equivalent because ordinary software doesn't spontaneously need a "new version" just because the world changed under it — deployed code doesn't silently start behaving differently due to shifting user behavior, but a static deployed model can.

**Q6: A team says "we don't need MLOps practices, we just call the OpenAI/Anthropic API with a prompt, we don't train anything." Do you agree? Why or why not?**
A: No — even zero-fine-tuning LLM usage has a versioned artifact (the prompt/system message) that changes behavior when edited, real cost/latency dynamics per request, non-deterministic and open-ended output that needs evaluation beyond exact-match testing, and hallucination risk. That's a full LLMOps surface even with no training pipeline at all.

**Q7: What is the "governance gap" some 2026 industry commentary describes in LLMOps, and why does it matter?**
A: It describes LLM deployment/serving tooling maturing quickly while lineage/governance tooling (tracing which prompt version, retrieved documents, or vector-index state produced a specific output) lags roughly where classical-MLOps lineage tooling was around 2018. It matters because without it, teams cannot reliably debug or audit a bad production output after the fact, which is a compliance and trust risk as much as an engineering one.

**Q8: How would you decide whether a new internal project needs full MLOps rigor (registry, CI/CD gates, drift monitoring) from day one, or can defer it?**
A: Base it on blast radius and iteration frequency: if it touches real users/money or will be iterated on repeatedly, build in versioning and evaluation gates early since retrofitting is expensive; if it's a one-off exploratory analysis, defer the heavy machinery but avoid decisions (e.g., prompts hardcoded everywhere) that would make adding it later a rewrite rather than an addition.

---

## 8. Summary, Key Takeaways, and Production Checklist

### Summary

MLOps exists because ML systems accrue a category of technical debt — entanglement, hidden feedback loops, silent data-distribution drift — that traditional DevOps practices were never built to catch, because that debt lives partly in data and learned behavior rather than entirely in code. LLMOps is not a competing discipline but the natural extension of MLOps to systems built around large language models, adding prompt versioning, retrieval/vector-index management, token economics, and open-ended-output evaluation on top of the same versioning/CI-CD/monitoring foundation. This 26-module course is sequenced to mirror how such a platform is actually built in practice: release engineering first, then the LLM application layer (prompting, cost, evaluation), then production observability, then the two flagship architectures (RAG and agents), then the classical-ML data/serving infrastructure at scale, then cross-cutting security/governance/cost concerns, and finally an integrated capstone.

### Key Takeaways

- MLOps = DevOps + (data/model versioning, statistical evaluation gates, drift monitoring, continuous training).
- LLMOps = MLOps + (prompt versioning, vector/retrieval management, token-cost tracking, LLM-as-judge evaluation, hallucination and agent-failure monitoring).
- The entanglement/CACE problem (Sculley et al., 2015) is still the deepest justification for why this whole field exists, a decade later.
- This course's 26-module order is not arbitrary — it follows the real build order of a production platform: foundation → application layer → evaluation → observability → flagship architectures → infrastructure at scale → governance/cost → integration.
- A "governance gap" between mature LLM deployment tooling and immature LLM lineage/governance tooling is a well-recognized 2026 industry problem this course addresses directly (Modules 03, 18, 23) rather than assuming away.

### Production Checklist (for this module — self-check, not a system checklist)

```
[ ] I can state the one-sentence definitions of MLOps and LLMOps without notes.
[ ] I can name at least 3 things MLOps adds on top of DevOps.
[ ] I can name at least 3 things LLMOps adds on top of MLOps.
[ ] I have completed the skills self-assessment and addressed any prerequisite gaps.
[ ] I understand why the 26 modules are sequenced the way they are, not just what each covers.
[ ] I have set a realistic personal pacing plan based on my available hours/week.
[ ] I know where to find deeper source material for this module (see Further Reading below).
```

---

## 9. Further Reading

The full annotated list of official docs, the foundational NeurIPS paper, engineering blog posts, curated GitHub repositories, and recommended videos/books for this module live in this same folder:

- `references.md` — official documentation and foundational papers (Google Cloud MLOps docs, AWS/IBM/Red Hat primers, roadmap.sh, Sculley et al. 2015, CD4ML)
- `videos.md` — recommended video content for this module
- `books.md` — recommended companion books (including Chip Huyen's *AI Engineering* and *Designing Machine Learning Systems*)
- `github.md` — curated repositories (DataTalksClub/mlops-zoomcamp, GokuMohandas/Made-With-ML, chiphuyen/aie-book, SylphAI-Inc/llm-engineer-handbook, kelvins/awesome-mlops, and the roadmap.sh-backing developer-roadmap repo)

Do not skip these — this tutorial deliberately references rather than repeats their content in depth.
